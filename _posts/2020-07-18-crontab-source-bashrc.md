---
layout: post
title: 如何在 crontab 中让 source ~/.bashrc 生效
---

cron 是许多类 Unix 操作系统中都自带的用来调度定时任务的工具，定时任务的配置是写在 crontab 文件中的，但是 crontab 文件不允许直接编辑，一般都是通过命令 crontab -e 来导入配置。配置文件中的每一行定义了一个定时任务，格式如下：

```bash
分钟 小时 天 月份 星期 命令
```
比如，有个需要每天凌晨 2 点执行的任务 `/home/user/task.sh`，那么可以如下配置：

```shell
0 2 * * * /home/user/task.sh > /home/user/log 2>&1
```
注意脚本的路径要写全。

这里一个常见的问题是自己手动在终端执行 `/home/user/task.sh` 可以成功，但是 cron 定时调度执行的时候却失败了，大部分情况是由于 cron 执行时环境变量跟我们在终端执行时的环境变量不同导致的。解决问题的办法有很多，本文要介绍的是用 `source ~/.bashrc` （假设用户需要用的环境变量都定义在 `~/.bashrc` 中）。

先直接给出方案，分如下两步：
- crontab -e 进行配置的时候在前面加上如下配置

```shell
SHELL = /bin/bash # 指定 shell 为 bash
```

- 在 `/home/user/task.sh` 脚本中加入如下命令

```shell
set -i
source ~/.bashrc
```

第一步指定 shell 为 bash，因为大多数情况下 cron 默认的 shell 为 sh，而 `~/.bashrc` 中的一些命令是 sh 不支持的，所以需要将 shell 设置为 bash。

第二步执行 `source ~/.bashrc` 很好理解，就是让 `~/.bashrc` 中配置的环境变量生效。前面的 `set -i` 是设置用交互的方式执行命令，因为在 `~/.bashrc` 的开头有一段脚本为

```shell
# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac
```

说明在非交互式的环境下，不会执行 `~/.bashrc` 中的任何命令。

以上介绍了如何在 crontab 中让 `source ~/.bashrc` 生效，主要目的是解决 cron 调度定时任务时由于环境变量导致的问题。


