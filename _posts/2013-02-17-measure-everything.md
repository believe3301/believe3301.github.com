---
layout: post
title: measure everything , monitor everything
category: linux
tags: [statsd, graphite, monitor, linux]
---

一直在思考在linux下的性能计数器/监控以及报警如何去做，在这方面有很多成熟的解决方案，例如nagios/ganglia/zabbix/cacti。觉得这些解决方案太重，或者在某个方面做的不好。
有次在github发现了statsd项目，受到了启发，可以使用collectd+statsd+graphite+nagios进行搭建。

系统架构
-------------

主要分为如下三个部分：

####指标收集

指标分为系统指标以及应用指标. 在linux系统指标收集的程序挺多的，中意collectd，主要是由于其支持java plugin以及write_graphite插件，这样通过java plugin写java代码监控java进程。而且数据可以直接发送到graphite。 

应用指标的收集通过进程将指标喂给statsd，由其进行聚集之后发送到graphite,具体可以参考etsy写的文章。对于nginx或者apache这样的进程，可以通过etsy出品的logster进行分析处理，然后发送给graphite。

####指标存储以及展现

graphite是一个可伸缩的实时的绘图系统，主要分为3个部分。

* whisper类似于rrd为定长的数据库，用于储存指标。
* carbon为后端存储应用，复杂处理请求以及调用whisper存储数据。
* graphite-web是一个展现数据的web app，其提供了render api，可以方便的将图形嵌入到其他的系统中。  

graphite专注与绘图，实现的非常棒，使用其提供的graph function(timeShift)，可以非常容易的实现指标的环比或者同比。
其提供的dashboard功能太简单，不过有很多第三方的实时的dashboard实现，我比较喜欢<https://github.com/jondot/graphene>。

####报警

报警可以使用nagios来完成，etsy提供了nagios插件用于分析graphite的数据进行报警。

具体可以查看<https://github.com/etsy/nagios_tools/blob/master/check_graphite_data>。


Reference
-----------

<http://codeascraft.etsy.com/2011/02/15/measure-anything-measure-everything/>  
<https://graphite.readthedocs.org/en/latest/>  
<http://blog.dccmx.com/2012/12/build-perfect-monitoring-system/>

-EOF-

{% include references.md %}
