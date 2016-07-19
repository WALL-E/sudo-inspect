# sudo inspect
探究sudo命令在不同的启动方式下有什么不同的表现？

* 终端命令行
* shell脚本

## 操作系统

```
$ cat /etc/redhat-release
CentOS release 6.7 (Final)
```

## 实验代码
本实验包含3个文件，一个python，代表应用程序，两个shell脚本，代表两种不同的启动方式

* app.py
* start.sh
* sudo_start.sh

```
==> app.py <==
#!/usr/bin/python

import time

time.sleep(3600)

==> start.sh <==
#!/bin/bash

./app.py

==> sudo_start.sh <==
#!/bin/bash

sudo ./app.py
```

## 实验一
使用命令行执行sudo指令

### 启动程序
```
sudo ./start.sh
```

### 查看进程
貌似没有搜索到sudo进程

```
$ ps aux|grep app.py
root      9068  0.0  0.0 117168  3612 pts/1    S+   11:57   0:00 /usr/bin/python ./app.py
root     11372  0.0  0.0 103320   864 pts/5    S+   11:59   0:00 grep app.py
```

### 查看app.py进程在做什么
可以看到，python中的sleep功能是通过select系统调用实现的。

```
$ ps aux|grep app.py
root      5887  0.0  0.0 103320   864 pts/5    S+   11:55   0:00 grep app.py
root     31990  0.0  0.0 117172  3612 pts/1    S+   11:49   0:00 /usr/bin/python ./app.py
 $ strace -p 31990
Process 31990 attached
select(0, NULL, NULL, NULL, {3588, 704690}
```

### 查看进程父子关系
发现sudo进程其实还在运行

```
     ├─sshd─┬─sshd───bash───sudo───start.sh───app.py
     │      ├─5*[sshd───bash]
     │      └─sshd───bash───pstree
```

### 查看sudo进程在做什么
sudo进程没有结束，阻塞在select系统调用

```
$ ps aux|grep sudo
root      2261  0.0  0.0 103320   860 pts/5    S+   11:51   0:00 grep sudo
root     31988  0.0  0.0 189252  2948 pts/1    S+   11:49   0:00 sudo ./start.sh
$ strace -p 31988
Process 31988 attached
select(7, [4 6], [], NULL, NULL
```

## 实验二
在脚本文件里执行sudo指令

### 启动程序
```
./sudo_start.sh
```

### 查看进程
貌似发现有两个进程在运行app.py

```
$ ps aux|grep app.py
root     31961  0.0  0.0 189252  2952 pts/1    S+   14:06   0:00 sudo ./app.py
root     31962  0.1  0.0 117172  3608 pts/1    S+   14:06   0:00 /usr/bin/python ./app.py
root     32056  0.0  0.0 103324   900 pts/5    S+   14:06   0:00 grep app.py
```

### 查看两个app.py进程各在做什么
其实两个进程并不一样，进程号为31961的进程是阻塞在select，这个select系统调用监听了两个文件描述符，并不是我们app.py运行的代码，其实是sudo进程。而进程号为31962的进程虽然也是阻塞在select中，但是这个系统调用表达的确实是sleep的意图，是我们app.py的运行结果。

```
$ strace -p 31961
Process 31961 attached
select(7, [4 6], [], NULL, NULL
$ strace -p 31962
Process 31962 attached
select(0, NULL, NULL, NULL, {3464, 209435}
```

### 查看进程父子关系
可以发现，shell脚本、sudo进程、app.py三个进程依然存在，只是父子关系和实验一有些不同

```
     ├─sshd─┬─sshd───bash───sudo_start.sh───sudo───app.py
     │      ├─5*[sshd───bash]
     │      └─sshd───bash───pstree
```

## 结论
sudo命令在不同的启动方式下，虽然进程的父子关系的表现不一样，但是这并不影响进程正确运行。

```
     ├─sshd─┬─sshd───bash───sudo───start.sh───app.py
     │      ├─5*[sshd───bash]
     │      └─sshd───bash───pstree
     
     ├─sshd─┬─sshd───bash───sudo_start.sh───sudo───app.py
     │      ├─5*[sshd───bash]
     │      └─sshd───bash───pstree

```
## 扩展
本文的核心知识点是linux平台的fork系统调用。

### fork
一个进程，包括代码、数据和分配给进程的资源。fork（）函数通过系统调用创建一个与原来进程几乎完全相同的进程，也就是两个进程可以做完全相同的事，但如果初始参数或者传入的变量不同，两个进程也可以做不同的事。其中，一个进程是父进程，另一个进程为子进程。

### sudo
sudo作为父进程，会使用select系统调用，等待子进程的退出，我们可以从源码中印证一下。
源码版本为sudo-1.8.3p1.tar.gz

* 等待子进程退出
* 使用select系统调用，监听两个文件描述符

```
/*
 * Run the command and wait for it to complete.
 */
int
run_command(struct command_details *details)
{
...
sudo_execve(details, &cstat);
...
}

/*
 * Execute a command, potentially in a pty with I/O loggging.
 * This is a little bit tricky due to how POSIX job control works and
 * we fact that we have two different controlling terminals to deal with.
 */
int
sudo_execve(struct command_details *details, struct command_status *cstat)
{
...
    /*
     * We use a pipe to atomically handle signal notification within
     * the select() loop.
     */
    if (pipe_nonblock(signal_pipe) != 0)
        error(1, _("unable to create pipe"));
...
    /*
     * We communicate with the child over a bi-directional pair of sockets.
     * Parent sends signal info to child and child sends back wait status.
     */
    if (socketpair(PF_UNIX, SOCK_DGRAM, 0, sv) == -1)
        error(1, _("unable to create sockets"));
...

/* Max fd we will be selecting on. */
    maxfd = MAX(sv[0], signal_pipe[0]);
...

nready = select(maxfd + 1, fdsr, fdsw, NULL, NULL);

...
}
...
```

