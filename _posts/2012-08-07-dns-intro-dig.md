---
layout: post
title: dns介绍以及dig使用
category: linux
tags: [dns, dig,  linux]
---

上次由于dns服务器坏了导致分析问题半天，虽然dns是最常用的协议之一，但是一直没怎么去研究，就抽了个时间看了看dns协议。

对于dns相关有如下几个关键点:

* dns协议为分层协议，下层的dns服务器需要上层的授权，最顶层的dns服务器为root服务器，为未命名服务器，通常用.表示。
* dns服务器一般通过bind搭建，不过配置相对比较复杂。也可以使用dnsmasq搭建一个轻量级的dns服务器或者用作dns缓存,一般的dns服务器都需要配置root服务器的RR(Resource Record)
* dns的rfc文档为[DOMAIN NAMES - CONCEPTS AND FACILITIES](http://www.ietf.org/rfc/rfc1034.txt) 以及 [DOMAIN NAMES - IMPLEMENTATION AND SPECIFICATION](http://www.ietf.org/rfc/rfc1035.txt)
* dns服务器开启的端口通常为53,同时接收udp/tcp请求，抓包写作：tcpdump -i eth0 -n udp port 53 and host 8.8.8.8
* dns服务器查询一般通过gethostbyname和gethostbyaddr库函数调用，可以通过ltrace进行分析
* dns服务器配置在/etc/resolv.conf，默认先读取/etc/hosts配置的信息，然后在通过dns配置获取

dns协议相对比较简单，这里就不介绍，主要介绍RR的格式

    [domain]     [ttl]     IN     [RR type]     [[RR data]]
    
    常见的正解文件 RR 相关信息
    [domain]    IN  [[RR type]  [RR data]]
    主机名.     IN  A           IPv4 的 IP 地址
    主机名.     IN  AAAA        IPv6 的 IP 地址
    域名.       IN  NS          管理这个域名的服务器主机名字.
    域名.       IN  SOA         管理这个域名的七个重要参数(容后说明)
    域名.       IN  MX          邮件交换记录，顺序数字  接收邮件的服务器主机名字
    主机别名.   IN  CNAME       规范名称，实际代表这个主机别名的主机名字.

主要的标志为A以及NS，A记录主机名称对应的ip地址，NS记录域名对应的域名服务器

dns的测试工具一般有host、nslookup、dig，建议使用dig进行测试

    dig www.baidu.com
    
    ; <<>> DiG 9.8.1-P1 <<>> www.baidu.com
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48376
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0
    
    ;; QUESTION SECTION:
    ;www.baidu.com.     IN  A
    
    ;; ANSWER SECTION:
    www.baidu.com.    19  IN  CNAME www.a.shifen.com.
    www.a.shifen.com. 62  IN  A 61.135.169.105
    www.a.shifen.com. 62  IN  A 61.135.169.125
    
    ;; Query time: 0 msec
    ;; SERVER: 10.0.1.90#53(10.0.1.90)
    ;; WHEN: Tue Aug  7 10:52:01 2012
    ;; MSG SIZE  rcvd: 90

简单分析下输出，前缀没有;为应答的数据,第一行为dig的版本号，第二行为dig的全局选项，第4、5行为dns应答头的解析。

QUESTION SECTION 为请求的格式，ANSWER SECTION 为应答，最后为请求的相关统计数据。

默认查询的资源类型为A，就是查询主机名称对应的ip地址,可以通过-t选项指定查询的资源类型,通过@指定使用dns服务器。

    dig -t ANY www.baidu.com @8.8.8.8
    
    ; <<>> DiG 9.8.1-P1 <<>> -t ANY www.baidu.com @8.8.8.8
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20947
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 4, ADDITIONAL: 4
    
    ;; QUESTION SECTION:
    ;www.baidu.com.     IN  ANY
    
    ;; ANSWER SECTION:
    www.baidu.com.    744 IN  CNAME www.a.shifen.com.
    
    ;; AUTHORITY SECTION:
    baidu.com.    54639 IN  NS  dns.baidu.com.
    baidu.com.    54639 IN  NS  ns4.baidu.com.
    baidu.com.    54639 IN  NS  ns3.baidu.com.
    baidu.com.    54639 IN  NS  ns2.baidu.com.
    
    ;; ADDITIONAL SECTION:
    dns.baidu.com.    61193 IN  A 202.108.22.220
    ns2.baidu.com.    58797 IN  A 61.135.165.235
    ns3.baidu.com.    61195 IN  A 220.181.37.10
    ns4.baidu.com.    67812 IN  A 220.181.38.10
    
    ;; Query time: 1 msec
    ;; SERVER: 8.8.8.8#53(8.8.8.8)
    ;; WHEN: Tue Aug  7 11:03:02 2012
    ;; MSG SIZE  rcvd: 194

其中AUTHORITY SECTION为授权信息，dns服务器信息。ADDITIONAL SECTION 为额外信息。

可以通过dig -x 反向查询，通过ip地址查询主机名称。

-EOF-

{% include references.md %}
