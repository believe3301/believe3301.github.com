---
layout: post
title: 通过strace分析zookeeper服务器端连接超时
category: tool
tags: [strace, tcpdump, linux]
---

上周六出现了一次故障，然后半夜机叫，说java服务启动的时候连接不了zookeeper。
由于zookeeper服务器端是集群而且zookeeper的客户端实现是通过shuffle选择一台服务器端进行连接，本以为是其中某台zookeeper服务器挂了，所以告知晚上在场的兄弟先改为连接其中的一台，谁知道启动成功了。

周一来了后进行问题的排查，首先检查服务器端，服务器端运行正常。然后在其他机器通过zk客户端连接，发觉特别的慢。通过tcpdump抓包发现启动客户端到发送sync包建立连接耗费了10s左右的时间。
决定通过strace查看在connect系统调用之前有啥系统调用特别耗时间。

    strace -f -t -econnect -o2.log ./zkCli.sh -server 192.168.110.231:8998

结果如下:
  
    23647 15:49:10 connect(13, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("10.0.1.93")}, 16) = 0
    23649 15:49:10 --- SIGCHLD (Child exited) @ 0 (0) ---
    23634 15:49:10 --- SIGCHLD (Child exited) @ 0 (0) ---
    23652 15:49:10 --- SIGCHLD (Child exited) @ 0 (0) ---
    23634 15:49:10 --- SIGCHLD (Child exited) @ 0 (0) ---
    23654 15:49:10 --- SIGCHLD (Child exited) @ 0 (0) ---
    23634 15:49:10 --- SIGCHLD (Child exited) @ 0 (0) ---
    23647 15:49:15 connect(14, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("10.0.1.95")}, 16) = 0
    23647 15:49:18 connect(15, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("10.0.1.92")}, 16) = 0
    23647 15:49:38 connect(13, {sa_family=AF_FILE, path="/var/run/avahi-daemon/socket"}, 110) = 0
    23647 15:49:43 connect(11, {sa_family=AF_INET6, sin6_port=htons(8998), inet_pton(AF_INET6, "::ffff:192.168.110.231", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, 28) = -1 EINPROGRESS (Operation now in progress)

简单说下参数含义
  
* -f  跟踪fork出来子进程
* -t  记录系统调用的时间
* -e  跟踪指定的系统调用
* -o  将结果输出到指定的文件

还有个-c选项很有用，可以对系统调用进行统计。

输出的含义为

* 第一列23647为系统调用所在进程的pid，在linux的线程其实也为一个轻量级的进程
* 第二列为系统调用的时间
* 第三列为系统调用的名称,例如上面的例子只跟踪了connect
* = 后为系统调用的返回指, 如果发生错误会perror

通过输出发现会反复的连接10.0.1.95:53, 10.0.1.92:53，怀疑是dns配置错误。

查看/etc/resolv.conf文件，发觉确实配置了一个坏掉的dns，导致查询dns非常慢。

ps:

实际排错走了很多弯路，就不一一详细描述了。其实早就知道dns坏了，一直认为是通过ip去访问的，没想到会是dns的问题。

最后发觉是zookeeper客户端会调用getHostName，通过ip查找主机名称然后设置线程名

    setName(getName().replaceAll("\\(.*\\)", "(" + addr.getHostName() + ":" + addr.getPort() + ")")); 

-EOF-

{% include references.md %}
