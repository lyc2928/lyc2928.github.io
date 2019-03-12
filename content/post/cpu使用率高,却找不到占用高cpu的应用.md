---
title: "Cpu使用率高,却找不到占用高cpu的应用"
date: 2019-03-13T03:18:09+08:00
draft: false
categories: ["linux"]
tags: ["linux", "监控"]
---


### **top** 中 cpu (us)使用率过高， 但top/ps 中却无法发现高 cpu 占用的应用
```bash
...
%Cpu(s): 80.8 us, 15.1 sy, 0.0 ni, 2.8 id, 0.0 wa, 0.0 hi, 1.3 si, 0.0 st
...
PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
6882 root 20 0 8456 5052 3884 S 2.7 0.1 0:04.78 docker-containe
6947 systemd+ 20 0 33104 3716 2340 S 2.7 0.0 0:04.92 nginx
7494 daemon 20 0 336696 15012 7332 S 2.0 0.2 0:03.55 php-fpm
7495 daemon 20 0 336696 15160 7480 S 2.0 0.2 0:03.55 php-fpm
10547 daemon 20 0 336696 16200 8520 S 2.0 0.2 0:03.13 php-fpm
10155 daemon 20 0 336696 16200 8520 S 1.7 0.2 0:03.12 php-fpm
10552 daemon 20 0 336696 16200 8520 S 1.7 0.2 0:03.12 php-fpm
15006 root 20 0 1168608 66264 37536 S 1.0 0.8 9:39.51 dockerd
4323 root 20 0 0 0 0 I 0.3 0.0 0:00.87 kworker/u4:1
```
> mpstat -P ALL 5 1

> pidstat -u 5 1

### 常用 pidstat 来发现异常占用资源的进程 
```bash
pidstat -u 5 1
pidstat -wt 5

cpu使用情况统计(-u)
内存使用情况统计(-r)
IO情况统计(-d)
针对特定进程统计(-p)
线程(-t)
```

### 最终都无法发现, 使用利器perf
```
perf record -g
稍等后 ctrl+c
perf report
```
> 发现原来有进程， 不断启动，启动不成功，反复启动
