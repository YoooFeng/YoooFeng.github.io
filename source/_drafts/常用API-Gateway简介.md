title: 常用API Gateway简介
author: YoooFeng
date: 2017-08-22 16:38:32
tags:
---
##### 初衷

&emsp;&emsp;API Gateway是与微服务（MicroService）这个概念一同兴起的一种架构，用于解决微服务过于分散，没有统一出入口进行流量管理的问题。当一个大的应用拆分成一个个微服务时，不同的微服务具有不同的职责，但是它们会有一些通用的功能如授权、流量监控和控制、日志统计等，在传统的单体应用中，这些功能一般是内嵌在应用程序中，在微服务框架下，继续使用内嵌的方式会造成非常高的管理和发布成本，因此我们需要一个抽象出一个统一的出入口完成这些功能，这就是API Gateway。我调研了几种市面上常见的API Gateway，简单分析了它们的技术特点和适用的场景。

###### 1. Kong
&emsp;&emsp;Kong是一个开源的，通过lua-nginx-module实现，在Nginx中运行的Lua应用程序。KONG没有使用lua-nginx-module直接编译Nginx，而是基于OpenResty，与已经包含了lua-nginx-module的OpenResty一起分发，支持Cassandra和PostgreSQL。从设计之初就考虑到了微服务的特性，对于微服务系统适应性最好。Kong支持用户个性化的定制使用插件来满足不同用户的需求。Kong有以下几个特点：
* nginx常用功能功能rest-api化，降低使用难度，便于二次开发
* 插件较多，有免费授权、限流、日志[syslog\statsd]、黑白名单等常用插件，并且支持用户根据规范开发个性化插件
* 内置支持集群，可以通过简单的连接直接增加机器节点，水平扩展容易
* 即插即用，在云或者内部环境或者容器中，都能直接运行

下面这张图是Kong官网的介绍，简单明了的说明了API Gateway的思想：
