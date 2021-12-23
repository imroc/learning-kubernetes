---
title: "IO WAIT 高"
type: book
date: "2021-11-12"
weight: 6
---

##  概述

系统如果出现 IO WAIT 高，说明 IO 设备的速度跟不上 CPU 的处理速度，CPU 需要在那里干等，这里的等待实际也占用了 CPU 时间，导致系统负载升高，可能就会影响业务进程的处理速度，导致业务超时。

## 如何判断 ？

使用 `top` 命令看下当前负载：

```text
top - 19:42:06 up 23:59,  2 users,  load average: 34.64, 35.80, 35.76
Tasks: 679 total,   1 running, 678 sleeping,   0 stopped,   0 zombie
Cpu(s): 15.6%us,  1.7%sy,  0.0%ni, 74.7%id,  7.9%wa,  0.0%hi,  0.1%si,  0.0%st
Mem:  32865032k total, 30989168k used,  1875864k free,   370748k buffers
Swap:  8388604k total,     5440k used,  8383164k free,  7982424k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
 9783 mysql     20   0 17.3g  16g 8104 S 186.9 52.3   3752:33 mysqld
 5700 nginx     20   0 1330m  66m 9496 S  8.9  0.2   0:20.82 php-fpm
 6424 nginx     20   0 1330m  65m 8372 S  8.3  0.2   0:04.97 php-fpm
```

`%wa` (wait) 表示 IO WAIT 的 cpu 占用，默认看到的是所有核的平均值，要看每个核的 `%wa` 值需要按下 "1":

```text
top - 19:42:08 up 23:59,  2 users,  load average: 34.64, 35.80, 35.76
Tasks: 679 total,   1 running, 678 sleeping,   0 stopped,   0 zombie
Cpu0  : 29.5%us,  3.7%sy,  0.0%ni, 48.7%id, 17.9%wa,  0.0%hi,  0.1%si,  0.0%st
Cpu1  : 29.3%us,  3.7%sy,  0.0%ni, 48.9%id, 17.9%wa,  0.0%hi,  0.1%si,  0.0%st
Cpu2  : 26.1%us,  3.1%sy,  0.0%ni, 64.4%id,  6.0%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu3  : 25.9%us,  3.1%sy,  0.0%ni, 65.5%id,  5.4%wa,  0.0%hi,  0.1%si,  0.0%st
Cpu4  : 24.9%us,  3.0%sy,  0.0%ni, 66.8%id,  5.0%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu5  : 24.9%us,  2.9%sy,  0.0%ni, 67.0%id,  4.8%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu6  : 24.2%us,  2.7%sy,  0.0%ni, 68.3%id,  4.5%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu7  : 24.3%us,  2.6%sy,  0.0%ni, 68.5%id,  4.2%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu8  : 23.8%us,  2.6%sy,  0.0%ni, 69.2%id,  4.1%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu9  : 23.9%us,  2.5%sy,  0.0%ni, 69.3%id,  4.0%wa,  0.0%hi,  0.3%si,  0.0%st
Cpu10 : 23.3%us,  2.4%sy,  0.0%ni, 68.7%id,  5.6%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu11 : 23.3%us,  2.4%sy,  0.0%ni, 69.2%id,  5.1%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu12 : 21.8%us,  2.4%sy,  0.0%ni, 60.2%id, 15.5%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu13 : 21.9%us,  2.4%sy,  0.0%ni, 60.6%id, 15.2%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu14 : 21.4%us,  2.3%sy,  0.0%ni, 72.6%id,  3.7%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu15 : 21.5%us,  2.2%sy,  0.0%ni, 73.2%id,  3.1%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu16 : 21.2%us,  2.2%sy,  0.0%ni, 73.6%id,  3.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu17 : 21.2%us,  2.1%sy,  0.0%ni, 73.8%id,  2.8%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu18 : 20.9%us,  2.1%sy,  0.0%ni, 74.1%id,  2.9%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu19 : 21.0%us,  2.1%sy,  0.0%ni, 74.4%id,  2.5%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu20 : 20.7%us,  2.0%sy,  0.0%ni, 73.8%id,  3.4%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu21 : 20.8%us,  2.0%sy,  0.0%ni, 73.9%id,  3.2%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu22 : 20.8%us,  2.0%sy,  0.0%ni, 74.4%id,  2.8%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu23 : 20.8%us,  1.9%sy,  0.0%ni, 74.4%id,  2.8%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:  32865032k total, 30209248k used,  2655784k free,   370748k buffers
Swap:  8388604k total,     5440k used,  8383164k free,  7986552k cached
```

`wa` 通常是 0%，如果经常在 1% 之上，说明存储设备的速度已经太慢，无法跟上 cpu 的处理速度。

## 如何排查 ？

参考 [排查 IO 高负载](https://imroc.cc/linux/troubleshooting/io-hang/)