---
layout: post
title: 使用TCPDUMP分析Linux网路数据包
category: funny
---

用简单的话来定义tcpdump，就是—— `dump the traffice on a network`，根据使用者的定义对网络上的数据包进行截获的包分析工具。tcpdump存在于基本的FreeBSD系统中，但是普通用户不能正常使用，具备root权限的用户可以直接执行这个命令。

### 初识tcpdump  

先看看manual的选项：  

>NAME
       tcpdump - dump traffic on a network
>
SYNOPSIS
       tcpdump [ -AdDeflLnNOpqRStuUvxX ] [ -c count ]
               [ -C file_size ] [ -F file ]
               [ -i interface ] [ -m module ] [ -M secret ]
               [ -r file ] [ -s snaplen ] [ -T type ] [ -w file ]
               [ -W filecount ]
               [ -E spi@ipaddr algo:secret,...  ]
               [ -y datalinktype ] [ -Z user ]
               [ expression ]

对tcpdump的功能简介如下：  

>DESCRIPTION
       Tcpdump  prints  out the headers of packets on a network interface that match the boolean expression.  It can also be run with the -w flag, which causes
       it to save the packet data to a file for later analysis, and/or with the -r flag, which causes it to read from a saved packet file rather than  to  read
       packets from a network interface.  In all cases, only packets that match expression will be processed by tcpdump.
>
       Tcpdump will, if not run with the -c flag, continue capturing packets until it is interrupted by a SIGINT signal (generated, for example, by typing your
       interrupt character, typically control-C) or a SIGTERM signal (typically generated with the kill(1) command); if run with the -c flag, it  will  capture
       packets until it is interrupted by a SIGINT or SIGTERM signal or the specified number of packets have been processed.
>
       When tcpdump finishes capturing packets, it will report counts of:
>
              packets ‘‘captured’’ (this is the number of packets that tcpdump has received and processed);
>
              packets ‘‘received by filter’’ (the meaning of this depends on the OS on which you’re running tcpdump, and possibly on the way the OS was config-
              ured - if a filter was specified on the command line, on some OSes it counts packets regardless of  whether  they  were  matched  by  the  filter
              expression  and, even if they were matched by the filter expression, regardless of whether tcpdump has read and processed them yet, on other OSes
              it counts only packets that were matched by the filter expression regardless of whether tcpdump has read and processed them  yet,  and  on  other
              OSes it counts only packets that were matched by the filter expression and were processed by tcpdump);
>
              packets ‘‘dropped by kernel’’ (this is the number of packets that were dropped, due to a lack of buffer space, by the packet capture mechanism in
              the OS on which tcpdump is running, if the OS reports that information to applications; if not, it will be reported as 0).

默认情况下，tcpdump会监视第一个网路接口上所有流过的数据包，即包含接收的和发送的。  

~~~~  
[root@localhost ~]# tcpdump
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 96 bytes
18:55:21.359018 IP XXX.XXX.XXX.XXX.ssh > XXX.17.XXX.147.55840: P 1393521XXX:1393521XXX(196) ack 2467111378 win 72
18:55:21.359269 IP XXX.XXX.XXX.XXX.50354 > XXX.XXX.XXX.XX.domain:  22008+ PTR? XXX.XX.XX.XX.in-addr.arpa. (45)
18:55:21.360273 IP XXXX.XXXX.XXX.XXX.XXX > XXX.XXX.XXX.XXX.XX:  22008 NXDomain 0/1/0 (105)
~~~~  

