---
layout: post
title: Python 中 lru_cache 的使用和实现
---

在计算机软件领域，缓存（Cache）指的是将部分数据存储在内存中，以便下次能够更快地访问这些数据，这也是一个典型的用空间换时间的例子。一般用于缓存的内存空间是固定的，当有更多的数据需要缓存的时候，需要将已缓存的部分数据清除后再将新的缓存数据放进去。需要清除哪些数据，就涉及到了缓存置换的策略，LRU（Least Recently Used，最近最少使用）是很常见的一个，也是 Python 中提供的缓存置换策略。

下面我们通过一个简单的示例来看 Python 中的 lru_cache 是如何使用的。

``` python
def factorial(n):
    print(f"计算 {n} 的阶乘")
    return 1 if n <= 1 else n * factorial(n - 1)

a = factorial(5)
print(f'5! = {a}')
b = factorial(3)
print(f'3! = {b}')
```

上面的代码中定义了函数 factorial，通过递归的方式计算 n 的阶乘，并且在函数调用的时候打印出 n 的值。然后分别计算 5 和 3 的阶乘，并打印结果。运行上面的代码，输出如下

```
计算 5 的阶乘
计算 4 的阶乘
计算 3 的阶乘
计算 2 的阶乘
计算 1 的阶乘
5! = 120
计算 3 的阶乘
计算 2 的阶乘
计算 1 的阶乘
3! = 6
```

可以看到，`factorial(3)` 的结果在计算 `factorial(5)` 的时候已经被计算过了，但是后面又被重复计算了。为了避免这种重复计算，我们可以在定义函数 factorial 的时候加上 lru_cache 装饰器，如下所示

```python
import functools
# 注意 lru_cache 后的一对括号，证明这是带参数的装饰器
@functools.lru_cache()
def factorial(n):
    print(f"计算 {n} 的阶乘")
    return 1 if n <= 1 else n * factorial(n - 1)
```

重新运行代码，输出如下

```
计算 5 的阶乘
计算 4 的阶乘
计算 3 的阶乘
计算 2 的阶乘
计算 1 的阶乘
5! = 120
3! = 6
```

可以看到，这次在调用 `factorial(3)` 的时候没有打印相应的输出，也就是说 `factorial(3)` 是直接从缓存读取的结果，证明缓存生效了。

被 lru_cache 修饰的函数在被相同参数调用的时候，后续的调用都是直接从缓存读结果，而不用真正执行函数。下面我们深入源码，看看 Python 内部是怎么实现 lru_cache 的。写作时 Python 最新发行版是 3.9，所以这里使用的是 [Python 3.9 的源码](https://github.com/python/cpython/blob/00e24cdca422f792b80016287562b6b3bccab239/Lib/functools.py#L478)，并且保留了源码中的注释。

{% highlight python linenos %}
def lru_cache(maxsize=128, typed=False):
    """Least-recently-used cache decorator.
    If *maxsize* is set to None, the LRU features are disabled and the cache
    can grow without bound.
    If *typed* is True, arguments of different types will be cached separately.
    For example, f(3.0) and f(3) will be treated as distinct calls with
    distinct results.
    Arguments to the cached function must be hashable.
    View the cache statistics named tuple (hits, misses, maxsize, currsize)
    with f.cache_info().  Clear the cache and statistics with f.cache_clear().
    Access the underlying function with f.__wrapped__.
    See:  http://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)
    """

    # Users should only access the lru_cache through its public API:
    #       cache_info, cache_clear, and f.__wrapped__
    # The internals of the lru_cache are encapsulated for thread safety and
    # to allow the implementation to change (including a possible C version).
    
    if isinstance(maxsize, int):
        # Negative maxsize is treated as 0
        if maxsize < 0:
            maxsize = 0
    elif callable(maxsize) and isinstance(typed, bool):
        # The user_function was passed in directly via the maxsize argument
        user_function, maxsize = maxsize, 128
        wrapper = _lru_cache_wrapper(user_function, maxsize, typed, _CacheInfo)
        wrapper.cache_parameters = lambda : {'maxsize': maxsize, 'typed': typed}
        return update_wrapper(wrapper, user_function)
    elif maxsize is not None:
        raise TypeError(
            'Expected first argument to be an integer, a callable, or None')
    
    def decorating_function(user_function):
        wrapper = _lru_cache_wrapper(user_function, maxsize, typed, _CacheInfo)
        wrapper.cache_parameters = lambda : {'maxsize': maxsize, 'typed': typed}
        return update_wrapper(wrapper, user_function)
    
    return decorating_function
{% endhighlight %}

这段代码中有如下几个关键点

- 关键字参数

  `maxsize` 表示缓存容量，如果为 `None` 表示容量不设限， `typed` 表示是否区分参数类型，注释中也给出了解释，如果 `typed == True`，那么 `f(3)` 和 `f(3.0)` 会被认为是不同的函数调用。

- 第 24 行的条件分支

  **如果 lru_cache 的第一个参数是可调用的，直接返回 wrapper，也就是把 lru_cache 当做不带参数的装饰器，这是 Python 3.8 才有的特性**，也就是说在 Python 3.8 及之后的版本中我们可以用下面的方式使用 lru_cache，可能是为了防止程序员在使用 lru_cache 的时候忘记加括号。

  ```python
  import functools
  # 注意 lru_cache 后面没有括号，
  # 证明这是将其当做不带参数的装饰器
  @functools.lru_cache
  def factorial(n):
      print(f"计算 {n} 的阶乘")
      return 1 if n <= 1 else n * factorial(n - 1)
  ```

  **注意**，Python 3.8 之前的版本运行上面代码会报错：TypeError: Expected maxsize to be an integer or None。

lru_cache 的具体逻辑是在 [`_lru_cache_wrapper`](https://github.com/python/cpython/blob/3.9/Lib/functools.py#L524) 函数中实现的，还是一样，列出源码，保留注释。

{% highlight python linenos %}
def _lru_cache_wrapper(user_function, maxsize, typed, _CacheInfo):
    # Constants shared by all lru cache instances:
    sentinel = object()          # unique object used to signal cache misses
    make_key = _make_key         # build a key from the function arguments
    PREV, NEXT, KEY, RESULT = 0, 1, 2, 3   # names for the link fields

    cache = {}
    hits = misses = 0
    full = False
    cache_get = cache.get    # bound method to lookup a key or return None
    cache_len = cache.__len__  # get cache size without calling len()
    lock = RLock()           # because linkedlist updates aren't threadsafe
    root = []                # root of the circular doubly linked list
    root[:] = [root, root, None, None]     # initialize by pointing to self

    if maxsize == 0:

        def wrapper(*args, **kwds):
            # No caching -- just a statistics update
            nonlocal misses
            misses += 1
            result = user_function(*args, **kwds)
            return result

    elif maxsize is None:

        def wrapper(*args, **kwds):
            # Simple caching without ordering or size limit
            nonlocal hits, misses
            key = make_key(args, kwds, typed)
            result = cache_get(key, sentinel)
            if result is not sentinel:
                hits += 1
                return result
            misses += 1
            result = user_function(*args, **kwds)
            cache[key] = result
            return result

    else:

        def wrapper(*args, **kwds):
            # Size limited caching that tracks accesses by recency
            nonlocal root, hits, misses, full
            key = make_key(args, kwds, typed)
            with lock:
                link = cache_get(key)
                if link is not None:
                    # Move the link to the front of the circular queue
                    link_prev, link_next, _key, result = link
                    link_prev[NEXT] = link_next
                    link_next[PREV] = link_prev
                    last = root[PREV]
                    last[NEXT] = root[PREV] = link
                    link[PREV] = last
                    link[NEXT] = root
                    hits += 1
                    return result
                misses += 1
            result = user_function(*args, **kwds)
            with lock:
                if key in cache:
                    # Getting here means that this same key was added to the
                    # cache while the lock was released.  Since the link
                    # update is already done, we need only return the
                    # computed result and update the count of misses.
                    pass
                elif full:
                    # Use the old root to store the new key and result.
                    oldroot = root
                    oldroot[KEY] = key
                    oldroot[RESULT] = result
                    # Empty the oldest link and make it the new root.
                    # Keep a reference to the old key and old result to
                    # prevent their ref counts from going to zero during the
                    # update. That will prevent potentially arbitrary object
                    # clean-up code (i.e. __del__) from running while we're
                    # still adjusting the links.
                    root = oldroot[NEXT]
                    oldkey = root[KEY]
                    oldresult = root[RESULT]
                    root[KEY] = root[RESULT] = None
                    # Now update the cache dictionary.
                    del cache[oldkey]
                    # Save the potentially reentrant cache[key] assignment
                    # for last, after the root and links have been put in
                    # a consistent state.
                    cache[key] = oldroot
                else:
                    # Put result in a new link at the front of the queue.
                    last = root[PREV]
                    link = [last, root, key, result]
                    last[NEXT] = root[PREV] = cache[key] = link
                    # Use the cache_len bound method instead of the len() function
                    # which could potentially be wrapped in an lru_cache itself.
                    full = (cache_len() >= maxsize)
            return result

    def cache_info():
        """Report cache statistics"""
        with lock:
            return _CacheInfo(hits, misses, maxsize, cache_len())

    def cache_clear():
        """Clear the cache and cache statistics"""
        nonlocal hits, misses, full
        with lock:
            cache.clear()
            root[:] = [root, root, None, None]
            hits = misses = 0
            full = False

    wrapper.cache_info = cache_info
    wrapper.cache_clear = cache_clear
    return wrapper
{% endhighlight %}

函数开始的地方 2~14 行定义了一些关键变量，

- `hits` 和 `misses` 分别表示缓存命中和没有命中的次数
- `root` 双向循环链表的头结点，每个节点保存前向指针、后向指针、key 和 key 对应的 result，其中 key 为 `_make_key` 函数根据参数结算出来的字符串，result 为被修饰的函数在给定的参数下返回的结果。**注意**，root 是不保存数据 key 和 result 的。
- `cache` 是真正保存缓存数据的地方，类型为 dict。`cache` 中的 key 也是 `_make_key` 函数根据参数结算出来的字符串，value 保存的是 key 对应的双向循环链表中的节点。

接下来根据 `maxsize` 不同，定义不同的 `wrapper`。

- `maxsize == 0`，其实也就是没有缓存，那么每次函数调用都不会命中，并且没有命中的次数 `misses` 加 1。

- `maxsize is None`，不限制缓存大小，如果函数调用不命中，将没有命中次数 `misses` 加 1，否则将命中次数 `hits` 加 1。

- 限制缓存的大小，那么需要根据 LRU 算法来更新 `cache`，也就是 42~97 行的代码。

  - 如果缓存命中 key，那么将命中节点移到双向循环链表的结尾，并且返回结果（47~58 行）

    **这里通过字典加双向循环链表的组合数据结构，实现了用 O(1) 的时间复杂度删除给定的节点。**

  - 如果没有命中，并且缓存满了，那么需要将最久没有使用的节点（root 的下一个节点）删除，并且将新的节点添加到链表结尾。在实现中有一个优化，直接将当前的 root 的 key 和 result 替换成新的值，将 root 的下一个节点置为新的 root，这样得到的双向循环链表结构跟删除 root 的下一个节点并且将新节点加到链表结尾是一样的，但是避免了删除和添加节点的操作（68~88 行）

  - 如果没有命中，并且缓存没满，那么直接将新节点添加到双向循环链表的结尾（`root[PREV]`，这里我认为是结尾，但是代码注释中写的是开头）（89~96 行）

最后给 `wrapper` 添加两个属性函数 `cache_info` 和 `cache_clear`，`cache_info` 显示当前缓存的命中情况的统计数据，`cache_clear` 用于清空缓存。对于上面阶乘相关的代码，如果在最后执行 `factorial.cache_info()`，会输出

```
CacheInfo(hits=1, misses=5, maxsize=128, currsize=5)
```

第一次执行 `factorial(5)` 的时候都没命中，所以 misses = 5，第二次执行 `factorial(3)` 的时候，缓存命中，所以 hits = 1。

最后需要说明的是，**对于有多个关键字参数的函数，如果两次调用函数关键字参数传入的顺序不同，会被认为是不同的调用，不会命中缓存。另外，被 lru_cache 装饰的函数不能包含可变类型参数如 list，因为它们不支持 hash。**

总结一下，这篇文章首先简介了一下缓存的概念，然后展示了在 Python 中 `lru_cache` 的使用方法，最后通过源码分析了 Python 中 `lru_cache` 的实现细节。



