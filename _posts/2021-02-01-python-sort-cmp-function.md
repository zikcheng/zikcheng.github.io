---
layout: post
title: Python 中 sorted 如何自定义比较逻辑
---

在 Python 中对一个可迭代对象进行排序是很常见的一个操作，一般会用到 sorted() 函数

``` python
num_list = [4, 2, 8, -9, 1, -3]
sorted_num_list = sorted(num_list)
print(sorted_num_list)
```

上面的代码是对整数列表 num_list 按从小到大的顺序进行排序，得到的结果如下

```
[-9, -3, 1, 2, 4, 8]
```

有时候不仅仅是对元素本身进行排序，而是在元素值的基础上进行一些计算之后再进行比较，比如将 num_list 中的元素按照其平方值的大小进行排序。

在 Python 2 中，可以通过 sorted() 函数中的 cmp 或 key 参数来实现这种自定义的比较逻辑。cmp 比较函数接收两个参数 x 和 y（x 和 y 都是列表中元素）并且返回一个数字，如果返回正数表示 x > y，返回 0 表示 x == y，返回负数表示 x < y。key 函数接收一个参数，重新计算出一个结果，然后用计算出的结果参与排序比较。因此在 Python 2 中按平方值大小排序可以有下面两种实现方式

```python
num_list = [4, 2, 8, -9, 1, -3]
# cmp 参数只在 Python 2 中存在，Python 3 及之后的版本移除了 cmp 参数
sorted_num_list = sorted(num_list, cmp=lambda x, y: x ** 2 - y ** 2)
sorted_num_list = sorted(num_list, key=lambda x: x ** 2)
```

但是随着 Python 3.0 的发布，cmp 参数也随之被移除了，也就是说在 Python 3 中自定义比较逻辑就只能通过 key 参数来实现。至于为什么将 cmp 参数移除，在 Python 的 [Issue tracker](https://bugs.python.org/issue1771) 中有一段很长的讨论，主要有以下两点原因

- cmp 是一个冗余参数，所有使用 cmp 的场景都可以用 key 来代替
- 使用 key 比使用 cmp 的性能更快，对于有 N 个元素的列表，在排序过程中如果调用 cmp 进行比较，那么 cmp 的调用次数为 Nlog(N) 量级（基于比较的排序的最快时间复杂度），如果使用 key 参数，那么只需要在每个元素上调用一次 key 函数，只有 N 次调用，虽然使用 key 参数也要进行 O(Nlog(N)) 量级比较次数，但这些比较是在 C 语言层，比调用用户自定义的函数快。

关于上面性能的问题，我做了一个实验，分别随机生成 1000、10000、100000 和 1000000 个整数，然后用 key 和 cmp 的方式分别进行排序并记录排序的时间消耗

``` python
import random
import time

counts = (1000, 10000, 100000, 1000000)

def custom_cmp(x, y):
    return x ** 2 - y ** 2

def custom_key(x):
    return x ** 2

print('%7s%20s%20s' % ('count', 'cmp_duration', 'key_duration'))
for count in counts:
    min_num = -count // 2
    max_num = count // 2
    nums = [random.randint(min_num, max_num) for _ in range(count)]
    start = time.time()
    sorted(nums, cmp=custom_cmp)
    cmp_duration = time.time() - start
    start = time.time()
    sorted(nums, key=custom_key)
    key_duration = time.time() - start
    print('%7d%20.2f%20.2f' % (count, cmp_duration, key_duration))
```

在我的笔记本上一次运行结果如下

```
  count        cmp_duration        key_duration
   1000                0.00                0.00
  10000                0.02                0.01
 100000                0.34                0.11
1000000                4.75                1.85
```

可以看到，当列表中数字的数量超过 100000 的时候，使用 key 函数的性能优势就非常明显了，比 cmp 快了 2~3 倍。

对于熟悉 Java 或 C++ 等其他编程语言的同学来说，可能更熟悉 cmp 的比较方式。其实 Python 3 中也可以通过 functools 工具包中的 cmp_to_key() 函数来将 cmp 转换成 key，从而使用接收两个参数的自定义比较函数 cmp。

```python
import functools

num_list = [4, 2, 8, -9, 1, -3]

def custom_cmp(x, y):
    return x ** 2 - y ** 2

sorted_num_list = sorted(num_list, key=functools.cmp_to_key(custom_cmp))
print(sorted_num_list)
```

那么，cmp_to_key() 函数是如何将 cmp 转换成 key 的呢，我们可以通过[源码](https://github.com/python/cpython/blob/3.9/Lib/functools.py#L202)一探究竟

``` python
def cmp_to_key(mycmp):
    """Convert a cmp= function into a key= function"""
    class K(object):
        __slots__ = ['obj']
        def __init__(self, obj):
            self.obj = obj
        def __lt__(self, other):
            return mycmp(self.obj, other.obj) < 0
        def __gt__(self, other):
            return mycmp(self.obj, other.obj) > 0
        def __eq__(self, other):
            return mycmp(self.obj, other.obj) == 0
        def __le__(self, other):
            return mycmp(self.obj, other.obj) <= 0
        def __ge__(self, other):
            return mycmp(self.obj, other.obj) >= 0
        __hash__ = None
    return K
```

其实 cmp_to_key() 返回的是一个类 K，只不过在类 K 中重载了各种比较运算符，重载的过程中使用到了自定义的比较函数 mycmp，使得 K 的大小比较逻辑与 mycmp 一致。这样，对于 num_list 中的每个元素 num 都会执行一次 K(num) 生成一个类 K 的实例，然后通过比较不同 K 的实例的大小进行排序。

虽然通过 cmp_to_key() 可以调用自定义的 cmp 函数，但是还是要优先使用 key 函数，因为通过 cmp_to_key() 方式会在排序过程中创建很多类 K 的实例，对性能有很大影响，下面是 cmp_to_key() 和 key 的性能比较

```
  count          cmp_to_key        key_duration
   1000                0.01                0.00
  10000                0.10                0.01
 100000                1.36                0.09
1000000               16.89                1.13
```

当 num_list 中的数量为 1000000 的时候 key 比 cmp_to_key 快了将近 15 倍。

本文主要介绍了如何在 sorted 函数中自定义比较逻辑，Python 2 中可以通过 cmp 或 key 来实现，cmp 接收 2 个参数，通过返回的数值来判断两个参数的大小，key 重新计算一个新的结果参与比较。在 Python 3 中，考虑到 cmp 的性能和冗余的原因，将其移除了。在 Python 3.2 中提供了 functools.cmp_to_key 这个函数来使用自定义的比较函数 cmp，但是出于性能的考虑，我们还是要优先使用 key 来进行排序。

