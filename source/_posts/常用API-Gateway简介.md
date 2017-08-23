title: 常用API Gateway简介
author: YoooFeng
date: 2017-08-22 16:38:32
tags:
---
##### 初衷

&emsp;&emsp;API Gateway是与微服务（MicroService）这个概念一同兴起的一种架构，用于解决微服务过于分散，没有统一出入口进行流量管理的问题。当一个大的应用拆分成一个个微服务时，不同的微服务具有不同的职责，但是它们会有一些通用的功能如授权、流量监控和控制、日志统计等，在传统的单体应用中，这些功能一般是内嵌在应用程序中，在微服务框架下，继续使用内嵌的方式会造成非常高的管理和发布成本，因此我们需要一个抽象出一个统一的出入口完成这些功能，这就是API Gateway。我调研了几种市面上常见的API Gateway，简单分析了它们的技术特点和适用的场景。
<!-- more -->
###### 1. Kong
&emsp;&emsp;Kong是一个开源的，通过lua-nginx-module实现，在Nginx中运行的Lua应用程序。KONG没有使用lua-nginx-module直接编译Nginx，而是基于OpenResty，与已经包含了lua-nginx-module的OpenResty一起分发，支持Cassandra和PostgreSQL。从设计之初就考虑到了微服务的特性，对于微服务系统适应性最好。Kong支持用户个性化的定制使用插件来满足不同用户的需求。Kong有以下几个特点：
* nginx常用功能功能rest-api化，降低使用难度，便于二次开发
* 插件较多，有免费授权、限流、日志[syslog\statsd]、黑白名单等常用插件，并且支持用户根据规范开发个性化插件
* 内置支持集群，可以通过简单的连接直接增加机器节点，水平扩展容易
* 即插即用，在云或者内部环境或者容器中，都能直接运行

下面这张图是Kong官网的介绍，简单明了的说明了API Gateway的思想：
![Kong](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/API_Gateway.pic/Kong架构图.png)
<center><font size=2>图1. Kong的API Gateway介绍</center>

###### 2. Zuul
&emsp;&emsp;由Netflix开发的开源API Gateway，由于Netflix的业务特性，每天要处理数量巨大的请求，所以Zuul在大规模请求的处理上性能优秀，当业务规模较大，需要处理的请求数量较多时可以考虑使用Zuul。</br>
&emsp;&emsp;Zuul 的特点之一就是提供了四种过滤器的 API，分别为前置（Pre）、后置（Post）、路由（Route）和错误（Error）四种处理方式，一个请求会先按顺序通过所有的前置过滤器，之后在路由过滤器中转发给后端应用，得到响应后又会通过所有的后置过滤器，最后响应给客户端。在整个流程中如果发生了异常则会跳转到错误过滤器中。一般来说，前置过滤器负责处理到达后端应用前的请求，如请求转发、增加请求参数等；请求完成后需要处理的操作由后置过滤器负责，如统计返回值和调用时间、记录日志、增加跨域头等。</br>
&emsp;&emsp;对于API Gateway来说，稳定性十分重要，要是API Gateway宕机，会导致所有微服务应用的入口流量都被切断，为了避免此类情况的发生，Zuul 设计了隔离机制以及重试机制来保证自身的稳定性。Zuul使用Hystrix对每一个微服务应用(Route)都进行了限流和隔离，防止一个Route抢占过多资源。Hystrix有两种隔离策略，基于线程和基于信号量，默认的是基于线程的隔离机制，即每一个 Route 的请求都会在一个固定大小且独立的线程池中执行，这样即使其中一个 Route 出现了问题，也只会是某一个线程池发生了阻塞，其他 Route 不会受到影响。</br>
目前已知使用Zuul的公司有Cloud、Riot等。Riot公司搭建的Zuul架构如下：
![Zull](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/API_Gateway.pic/Riot的Zuul架构图.png)
<center><font size=2>图2. Riot公司的Zuul架构图</center>

###### 3. Orange
&emsp;&emsp;基于OpenResty开发，支持Mysql，使用前需要对配置文件进行本地化修改，无法即插即用，最近也参考Kong的设计实现了插件功能，用户可以根据规范编写自定义插件扩展Orange的功能。Orange的框架如下：
![Orange](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/API_Gateway.pic/Orange的架构图.png)
<center><font size=2>图3. Orange的架构图</center>

###### 4. Tyk
&emsp;&emsp;基于Go语言开发，开源，有收费的pro版。Tyk主打轻量级的概念，体积较小，安装和部署比较简易，对于需要处理的请求数量级不是很大的场景比较适用。Tyk可以部署多个实例并行，但是这些实例无法即插即用，还需要一个Redis数据库才能运行。Tyk的一些特性如下：
+ 请求配额和速率限制
+ 多种认证方式
+ 数据分析
+ 不停机发布REST API
+ 能够导入Apiary 或者 Swagger接口文档，并Mock
+ 性能监控
+ 报文转换

&emsp;&emsp;Tyk Gateway对于REST API的支持较好，可以直接导入Swagger文档，但是同时，Tyk对于其他的服务支持就比较差了，Tyk不支持SOAP或者RPC等其他服务，可以说是为REST API量身定做的。Tyk API gateway的架构如下：
![Tyk](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/API_Gateway.pic/Tyk的架构图.png)
<center><font size=2>图4. Tyk的架构图</center>

&emsp;&emsp;各类API Gateway在设计理念上都大同小异，只有在具体的应用场景中才会体现出各自的侧重和适用场景，这需要根据具体的需求进行调研。