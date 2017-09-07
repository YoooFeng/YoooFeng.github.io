title: 将Jenkins部署在Docker中
author: YoooFeng
date: 2017-08-24 16:10:37
tags:
---
# 动机
Docker，Github 社区最火的项目之一，Docker是一个容器，主要的作用就是在一台机器上搭建出不同的环境。考虑以下的需求，想要在一台机器上既运行Java6的程序，又运行Java8的程序，如果安装虚机，一个环境动辄需要2-3G内存的开销，而Docker仅仅需要几百MB就能搭建出一个完全独立运行的环境。

这几天研究了一下Jenkins的工作原理和流程，Jenkins的slave节点完全可以放在Docker中，既便于管理，也节省了硬件开销，在网上查了一些资料，借鉴了别人的一些想法，准备按照下面的思路自己动手试验一下：用 Jenkins 来做节点控制、版本管理、流程设置、触发 Job，用 Docker 来搭建编译部署环境。把这一切连接起来，我们要做的就是流程脚本和 Dockerfile 的编写。
<!-- more -->

# 环境安装及配置
从头开始详细记录一下整个环境安装和配置的过程。下面是我选择的系统、软件版本，以供参考：

+ 操作系统：Ubuntu 16.04 LTS
+ Jenkins：2.60.3
+ Maven: 3.3.9
+ Docker: 17.06.1-ce

提一下，以前常用的系统其实是CentOS，不过Docker不支持CentOS6，最近才开始支持CentOS7，而且不是很稳定，所以说对Docker这类新潮的玩意儿还是选择Ubuntu，毕竟Docker也是基于Ubuntu优化的。</br>

首先安装Docker，我们使用如下命令进行安装：

	curl -sSL https://get.docker.com/ | sh

如果网络环境不是很好，你可以考虑使用阿里云或者github的镜像安装Docker，此处不再赘述。
安装成功后，使用命令

	docker -v
	Docker version 17.06.1-ce, build 874xxx

即表示安装成功了。

接下来构建我们的Jenkins image，这里使用的是改良版的jenkins镜像

	https://hub.docker.com/r/lw96/java8-jenkins-maven-git-vim/
    
从名字可以看出，除了jenkins外，该镜像还集成了maven、git等常用的工具。我们在后面的构建过程中会用到这些工具，目前的方案是使用安装了这些环境的jenkins镜像，并且与宿主机使用volume进行安装位置的映射，这样我们可以将maven、jenkins的插件缓存保存在主机上，当镜像关闭时缓存得以保留。因此，我们需要在主机上同样安装java、maven、jenkins等环境。

使用以下命令可以从官方的镜像仓库将我们想要的镜像下载下来，如果下载速度不理想可以使用阿里云提供的加速服务器。

	docker pull /lw96/java8-jenkins-maven-git-vim 

接下来启动容器，同时将宿主机上面的maven、jenkins文件目录挂载到镜像中，让jenkins image可以使用宿主机中的插件缓存：

	docker run -d -p 8080:8080 -v /var/lib/jenkins:/jenkins 
	-v /etc/localtime:/etc/localtime:ro 
	-v /usr/share/maven:/opt/maven 
	-v /var/run/docker.sock:/var/run/docker.sock 
	--name jenkins lw96/java8-jenkins-maven-git-vim

+ -d: 指明镜像在后台运行(Daemonized)；
+ -p: 将容器8080端口映射给主机的8080端口,这样我们可以通过宿主机的浏览器输入地址http://localhost:8080 来访问该容器；
+ -v: 挂载数据卷，把引号前面的绝对路径换成系统中对应的路径；
+ --name: 给容器一个名字。

