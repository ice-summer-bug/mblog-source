---
layout: post
title: 记一次线上端口排查过程
categories: linux
tags: [linux指令]
date: 2018-01-18 21:00:00
description: netstat 指令排查端口占用
---

### 前言

今天在查看线上应用的时候发现一个服务启动之后报错断开已经被占用，但是`tomcat` 容器的日志中只说了端口已经被占用，但是没说是哪个端口被占用了，我们只能看看这个服务在其他机器上占用了什么接口，这时候我们就想到 `netstat` 指令

## `netstat` 查看端口占用情况

### 常用 `netstat` 指令

```bash
$netstat -tlp | grep 99573
tcp        0      0 0.0.0.0:24013           0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:17134           0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:sieve-filter    0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:18134           0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 localhost:19134         0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:commplex-main   0.0.0.0:*               LISTEN      99573/java          
```

`sieve-filter` 这个是什么？`commplex-main` 又是什么？我们怎么才能看到这两个端口的具体数值呢？

### `netstat` 指令常用参数

    netstat: 查看网络端口使用情况
        -a: 打印索引的套接字连接
        -t: 只打印TCP 连接信息
        -u: 只打印UDP 连接信息
        -p: 打印进程名称及进程ID
        -l: 打印所有在监听的服务端口
        -n: 用数字的形式打印端口信息，不对端口进行解析

到这里我们就知道如何去打印具体的端口信息，去解决上面遇到的问题了

```
$netstat -tnlp | grep 99573
tcp        0      0 0.0.0.0:24013           0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:17134           0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:2000    0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:18134           0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 localhost:19134         0.0.0.0:*               LISTEN      99573/java          
tcp        0      0 0.0.0.0:5000   0.0.0.0:*               LISTEN      99573/java         
```
### 数据列说明
1. Proto: 通信协议，tcp, udp, raw 等
2. Rece-Q: 接收队列中上午读取的数据，单位字节，一般是 0
3. Send-Q: 发送到远端尚未收到 ACK 的数据，单位字节，一般是0
4. Local Address: 通信连接中的本机地址及端口，一般情况下是 FQDN（全限定域名） 格式显示的地址，端口的显示格式默认是端口对应的服务名称，只有指定 `--numersic` 的时候才会按照数字形式打印
5. Foreign Address: 套接字的远端信息，地址加端口
6. State: 套接字连接状态
这里我们可以穿插讲解一下套接字连接状态

6.1 ESTABLISHED 传输中断状态
6.2 SYN_SENT 客户端发起连接发送连接请求后的状态
6.3 SYN_RECV 套接字服务端接收到连接请求后的状态
6.4 FIN_WAIT1 客户端发起断开连接请求后的状态
6.5 FIN_WAIT2 断开连接请求方接收到对方回复之后，等待对方发起断开连接之后的状态
6.6 TIME_WAIT 客户端接收到服务端的断开连接的请求之后，向服务端发送ACK 信息后，进入此状态，是为了防止网络故障，进入 `TIME_WAIT` 后，如果服务端一直没有收到 ACK，会重新发送断开连接的请求
6.7 CLOSED 连接关闭的状态
6.8 CLOSED_WAIT 服务端接收到断开连接请求之后，发送ACK 之后
6.9 LAST_ACK 服务端确认数据发送完毕后，发送断开连接请求之后，进入此状态   
6.10 LISTEN 暴露接口等待连接 
6.11 CLOSING 连接双方同时发起断开连接的时候，出现的一个短暂的状态
6.12 UNKNOWN 未知的状态

### `netstat` 指令参数说明

```bash
$netstat -h
usage: netstat [-vWeenNcCF] [<Af>] -r         netstat {-V|--version|-h|--help}
       netstat [-vWnNcaeol] [<Socket> ...]
       netstat { [-vWeenNac] -I[<Iface>] | [-veenNac] -i | [-cnNe] -M | -s [-6tuw] } [delay]

        -r, --route              display routing table
        -I, --interfaces=<Iface> display interface table for <Iface>
        -i, --interfaces         display interface table
        -g, --groups             display multicast group memberships
        -s, --statistics         display networking statistics (like SNMP)
        -M, --masquerade         display masqueraded connections

        -v, --verbose            be verbose
        -W, --wide               don't truncate IP addresses
        -n, --numeric            don't resolve names
        --numeric-hosts          don't resolve host names
        --numeric-ports          don't resolve port names
        --numeric-users          don't resolve user names
        -N, --symbolic           resolve hardware names
        -e, --extend             display other/more information
        -p, --programs           display PID/Program name for sockets
        -o, --timers             display timers
        -c, --continuous         continuous listing

        -l, --listening          display listening server sockets
        -a, --all                display all sockets (default: connected)
        -F, --fib                display Forwarding Information Base (default)
        -C, --cache              display routing cache instead of FIB
        -Z, --context            display SELinux security context for sockets

  <Socket>={-t|--tcp} {-u|--udp} {-U|--udplite} {-w|--raw} {-x|--unix}
           --ax25 --ipx --netrom
  <AF>=Use '-6|-4' or '-A <af>' or '--<af>'; default: inet
  List of possible address families (which support routing):
    inet (DARPA Internet) inet6 (IPv6) ax25 (AMPR AX.25)
    netrom (AMPR NET/ROM) ipx (Novell IPX) ddp (Appletalk DDP)
    x25 (CCITT X.25)
```


### 端口占用具体结论

| 端口 | 使用协议 | 服务 | 说明 |
|-|-|-|-|
| 5000 | TCP/UDP | commplex-main | |
| 2000 | TCP/UDP | sieve-filter | |


结论：
        这里使用 `commplex-main` 和 `sieve-filter` 端口的是 `crashub`
