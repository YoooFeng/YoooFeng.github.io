title: 利用Jenkins对Daytrader进行自动化CI/CD
author: YoooFeng
date: 2017-08-30 17:22:23
tags:
---
# 写在前面

最近在探索jenkins的使用和部署，准备在自己的机器上部署一个开源的网站，熟悉熟悉jenkins的使用。

这次选择的项目是Daytrader，一个在线的股票交易系统，同时也是作为一个benchmark app开发的。我这次使用的是将Daytrader划分为微服务之后的版本，作者取名叫做μDaytrader。部署的系统为Ubuntu 16.04 LTS。

首先安装和配置tomcat环境：

为了系统安全，tomcat不应该在root用户组下运行，所以我们首先创建新的tomcat组：

	sudo groudadd tomcat
    
然后创建一个叫tomcat的用户：

	sudo useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat
  
tomcat用户属于tomcat组，home目录是/opt/tomcat，我们把tomcat安装在这个目录。/bin/false代表这个用户是不能登录的。

接下来创建/opt/tomcat文件夹，将下载的tomcatxxx.tar.gz包解压到该目录。

赋予tomcat用户管理tomcat的权限，进入/opt/tomcat文件夹，执行命令：

	sudo chgrp -R tomcat conf
    sudo chmod g+rwx conf
    sudo g+r conf/*
    sudo chown -R tomcat webapps/work/temp/logs
    
c
