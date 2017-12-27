---
title: 记一次tomcat cpu占用率过高问题的分析
tags: ["java", "问题分析"]
categories: ["java", "问题分析"]
icon: fa-handshake-o
---

# 问题描述
linux系统下，一个tomcat web服务的cpu占用率非常高，top显示结果超过200%。请求无法响应。反复重启依然同一个现象。

# 问题排查

## 1、获取进程信息
通过jdk提供的jps命令可以快速查出jvm进程，
>jps pid

## 2、查看jstack信息

>jstack pid

发现存在大量log4j线程block，处于waiting lock状态

    org.apache.log4j.Category.callAppenders(org.apache.log4j.spi.LoggingEvent) @bci=12, line=201 (Compiled frame)

搜索相关信息，发现log4j 1.x版本存在死锁问题。

发现问题，于是调整log4j配置，仅打开error级别日志，重启tomcat。此时stack中block线程消失，但进程cpu占用率依然高涨。

## 3、进一步排查
分析每个线程的cpu占用量，此处需要引入一个大神贡献的脚本，计算java进程中，每个线程的cpu使用量。

    #!/bin/bash

    typeset top=${1:-10}
    typeset pid=${2:-$(pgrep -u $USER java)}
    typeset tmp_file=/tmp/java_${pid}_$$.trace

    $JAVA_HOME/bin/jstack $pid > $tmp_file
    ps H -eo user,pid,ppid,tid,time,%cpu --sort=%cpu --no-headers\
            | tail -$top\
            | awk -v "pid=$pid" '$2==pid{print $4"\t"$6}'\
            | while read line;
    do
            typeset nid=$(echo "$line"|awk '{printf("0x%x",$1)}')
            typeset cpu=$(echo "$line"|awk '{print $2}')
            awk -v "cpu=$cpu" '/nid='"$nid"'/,/^$/{print $0"\t"(isF++?"":"cpu="cpu"%");}' $tmp_file
    done

    rm -f $tmp_file

脚本适用范围

因为ps中的%CPU数据统计来自于/proc/stat，这个份数据并非实时的，而是取决于OS对其更新的频率，一般为1S。所以你看到的数据统计会和jstack出来的信息不一致也就是这个原因～但这份信息对持续LOAD由少数几个线程导致的问题排查还是非常给力的，因为这些固定少数几个线程会持续消耗CPU的资源，即使存在时间差，反正也都是这几个线程所导致。

除了这个脚本，简单点儿的方法则是，查出进程id后，通过如下命令查看该进程中每个线程的资源使用情况
>top -H -p pid

从这里获取pid（线程id），转换为16进制，然后去stack信息中查找对象的线程信息。

通过上述方法，查出tomcat进程对应的线程cpu占用率累积之和约80%，远小于top给出的200%+

说明并不存在长期占用cpu的线程，应该是属于有许多短暂性的cpu密集计算。进而怀疑是不是jvm内存不足，频繁gc导致。

>jstat -gc pid

发现jvm内存使用并未出现异常，gc次数明显暴涨

查完内存，由于本身是一个网络程序，进一步排查网络连接。

## 4、问题定位

查询tomcat对应端口的tcp链接，发现存在大量EASTABLISH的链接，还有部分其它状态的连接，总计400+。
>netstat -anp | grep port

进一步查看这些连接的来源，发现是该tomcat服务的应用端，存在大量后台线程，在频繁轮询该服务，导致该服务tomcat 连接数被打满，无法继续接收请求。

## 5、根源分析
直接触发原因是客户端轮询，请求异常，继续轮序；客户端不断有新的后台线程加入轮询队伍，最终导致服务端tomcat连接被打满。

但是，为什么会出现客户端请求异常，根本原因就在于服务端log4j的导致线程block所致。
