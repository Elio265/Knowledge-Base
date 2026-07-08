---
tags: [linux, troubleshooting, tools]
created: 2026-07-07
updated: 2026-07-07
status: 面试重点
---

# Linux 常用排查工具

## 一句话理解

Linux 排查不是背命令，而是先判断问题属于 CPU、内存、磁盘、网络、文件描述符、系统调用还是日志，再选对应工具逐层缩小范围。

## 排查总流程

```text
先判断是哪类资源异常
再定位到进程 / 线程 / 文件 / 端口
再结合调用栈、系统调用、日志确认原因
最后看最近发布、流量变化和配置变更
```

## CPU 飙高

常用步骤：

```bash
top
top -H -p <pid>
gdb -p <pid>
thread apply all bt
strace -p <pid>
```

解释：

- `top`：先找 CPU 高的进程。
- `top -H -p <pid>`：看进程内哪个线程 CPU 高。
- `gdb`：查看线程调用栈，判断是否死循环、热点函数、锁竞争。
- `strace`：看是否频繁系统调用。

面试表达：

> 我会先用 `top` 找到 CPU 高的进程，再用 `top -H -p <pid>` 定位高 CPU 线程。然后用 `gdb` attach 查看线程调用栈，确认是在死循环、热点计算、锁竞争还是业务逻辑里。如果怀疑系统调用异常，再用 `strace -p <pid>` 辅助观察。

## 内存持续上涨

常用步骤：

```bash
top
free -h
ps aux --sort=-%mem
pmap -x <pid>
cat /proc/<pid>/status
cat /proc/<pid>/smaps
```

C/C++ 程序怀疑泄漏时：

```bash
asan
valgrind
heaptrack
gperftools
```

注意：

- 内存问题通常先看进程级别，线程没有独立堆。
- `gdb` 可以辅助看调用栈和对象状态，但不是排查泄漏的首选工具。

面试表达：

> 我会先用 `top`、`free -h`、`ps aux --sort=-%mem` 判断是系统整体内存压力，还是某个进程 RSS 持续上涨。定位进程后，用 `pmap`、`/proc/<pid>/status`、`smaps` 看内存区域增长。C/C++ 程序如果怀疑泄漏，会用 ASan、Valgrind、heaptrack 等工具定位分配栈。

## 端口被占用

常用命令：

```bash
ss -lntp
netstat -lntp
lsof -i :8080
ps -fp <pid>
```

解释：

- `ss -lntp`：看 TCP 监听端口和进程，现代 Linux 更推荐。
- `netstat -lntp`：传统方式。
- `lsof -i :端口`：直接查某个端口被谁占用。

面试表达：

> 我会用 `ss -lntp` 或 `netstat -lntp` 查看监听端口和进程。如果只关心某个端口，用 `lsof -i :端口号`。拿到 PID 后，再用 `ps -fp <pid>` 确认启动命令。

## 文件删除后空间没释放

原因：

```text
rm 删除目录项
进程仍然持有 fd
inode 和数据块还不能释放
最后一个 fd 关闭后，空间才真正回收
```

排查：

```bash
lsof | grep deleted
lsof +L1
lsof -p <pid>
```

处理：

- 最稳妥是重启或优雅 reload 占用文件的进程。
- 如果是日志文件，优先让服务重新打开日志。
- 不要随便操作 `/proc/<pid>/fd/*`。

面试表达：

> 文件删除后空间没释放，通常是仍有进程打开着这个文件。`rm` 只是删除目录项，inode 和数据块要等链接数为 0 且没有任何 fd 引用时才释放。我会用 `lsof +L1` 或 `lsof | grep deleted` 找到占用进程，再让进程 reload、重启或关闭对应 fd。

## 进程卡在系统调用

常用命令：

```bash
strace -p <pid>
strace -T -p <pid>
strace -c -p <pid>
```

常见判断：

| 现象 | 可能原因 |
|------|----------|
| 卡在 `read` / `recvfrom` | 等文件或网络数据 |
| 卡在 `epoll_wait` | 可能只是事件循环空闲 |
| 大量 `open` / `stat` | 频繁访问文件 |
| 大量 `futex` | 锁竞争或线程同步等待 |
| `connect` 超时 | 网络不通或对端不可达 |

面试表达：

> 如果怀疑进程卡在系统调用上，我会用 `strace -p <pid>` attach 观察 syscall。配合 `-T` 看耗时，配合 `-c` 看统计。比如频繁 `futex` 可能是锁竞争，卡在 `read` 可能是在等 IO，卡在 `epoll_wait` 可能只是服务空闲。

## 查看进程打开了什么

常用命令：

```bash
lsof -p <pid>
lsof -i
lsof -i :8080
lsof +L1
```

`lsof` 可以查看进程打开的普通文件、socket、pipe、动态库等资源。

## 磁盘满

常用步骤：

```bash
df -h
du -sh *
du -sh /var/*
du -sh /var/log/*
find /var -type f -size +1G
lsof +L1
```

解释：

- `df -h`：先看哪个挂载点满了。
- `du -sh *`：逐层看哪个目录大。
- `find`：按大小找大文件。
- `lsof +L1`：排查删除文件仍被进程占用。

面试表达：

> 我会先用 `df -h` 看哪个文件系统满了，再用 `du -sh 目录/*` 逐层定位大目录和大文件。如果 `df` 显示空间没释放，但 `du` 找不到大文件，会怀疑 deleted 文件仍被进程打开，用 `lsof +L1` 排查。

## 负载高但 CPU 不高

Linux load 不只统计正在跑 CPU 的任务，也会统计不可中断睡眠任务。

常见原因：

- 磁盘 IO 慢。
- 网络文件系统卡住。
- 块设备异常。
- 很多进程处于 D 状态。

常用命令：

```bash
top
uptime
vmstat 1
iostat -x 1
ps -eo state,pid,ppid,cmd
```

重点看：

- `vmstat` 的 `r`：等待 CPU 的任务数。
- `vmstat` 的 `b`：不可中断睡眠任务数。
- `iostat -x` 的 `%util`、`await`、队列长度。
- `ps` 里是否有大量 D 状态进程。

面试表达：

> load 高但 CPU 不高，通常说明任务不是在消耗 CPU，而是在等待 IO 或处于不可中断睡眠。Linux load 会统计运行态和 D 状态任务。我会用 `vmstat 1` 看 `r` 和 `b`，用 `iostat -x 1` 看磁盘 await 和 util，用 `ps` 找 D 状态进程。

## 系统日志和服务失败

常用命令：

```bash
systemctl status <service>
journalctl -u <service>
journalctl -u <service> -f
journalctl -xe
dmesg
dmesg -T
tail -f /var/log/messages
tail -f /var/log/syslog
grep "error" /var/log/xxx.log
```

使用场景：

- `systemctl status`：看服务状态、退出码、最近日志。
- `journalctl -u`：看 systemd 服务日志。
- `dmesg`：看内核日志，如 OOM、磁盘错误、网卡错误。
- `tail` / `grep`：看传统日志文件和搜索关键字。

面试表达：

> 排查服务启动失败，我会先用 `systemctl status 服务名` 看状态、退出码和最近日志，再用 `journalctl -u 服务名` 查看完整服务日志。内核相关问题，比如 OOM、磁盘或网卡错误，可以看 `dmesg`。传统日志文件则用 `tail -f` 和 `grep`。

## 容易踩坑的地方

1. CPU 高只停留在进程层面，没有继续定位到线程和调用栈。
2. 内存上涨时误用 `top -H` 查线程；内存主要先看进程级别和内存区域。
3. 把 `free` 当成磁盘排查命令；磁盘空间主要看 `df` 和 `du`。
4. 不知道删除文件后空间不释放可能是 fd 仍被进程持有。
5. 不知道 `strace` 用来观察系统调用。
6. 不知道 `lsof` 可以查看进程打开的文件、socket、pipe。
7. load 高但 CPU 不高时，只盯 CPU，忽略 D 状态和 IO 等待。
8. 看系统日志时只想到 `vim` / `grep`，没有先用 `systemctl`、`journalctl`、`dmesg` 找入口。

## 我的薄弱点

- CPU 排查能想到 `top` 和 `ps`，但要补牢 `top -H -p <pid>`、`gdb` 调用栈和 `strace`。
- 内存排查容易往线程方向想，需要记住线程共享进程堆，优先看进程 RSS、`pmap`、`smaps` 和泄漏工具。
- 端口排查能说出 `netstat -natp`，后续要优先掌握 `ss -lntp` 和 `lsof -i :port`。
- 删除文件空间不释放、`strace`、`lsof`、load 高 CPU 不高、系统日志入口都是这轮新知识点，需要后续复测。
- 磁盘排查曾把 `free` 和磁盘混在一起，要明确：内存看 `free`，磁盘看 `df` / `du`。

## 成长记录

- 已能从 CPU 飙高场景想到先用 `top` 定位进程。
- 已能说出端口排查可以用 `netstat -natp`。
- 已补齐 CPU 排查链路：进程、线程、调用栈、系统调用。
- 已建立工具分类意识：CPU、内存、磁盘、端口、fd、系统调用、日志分别用不同入口。

## 面试高频问题

1. 线上服务 CPU 飙高，你怎么排查？
2. 服务内存持续上涨，你怎么排查？
3. 怎么查某个端口被哪个进程占用？
4. 文件删除后磁盘空间没释放，可能是什么原因？
5. `strace` 是做什么的？常见输出怎么判断？
6. `lsof` 能查什么？
7. 磁盘满了，怎么定位哪个目录或文件占用最多？
8. 系统 load 很高但 CPU 不高，可能是什么原因？
9. 怎么查看服务启动失败原因？
10. `systemctl status`、`journalctl`、`dmesg` 分别看什么？

## 关联知识

- [[文件描述符与重定向]]
- [[inode与软硬链接]]
- [[进程状态与回收]]
- [[IO多路复用]]
- [[TCP服务端连接建立]]
