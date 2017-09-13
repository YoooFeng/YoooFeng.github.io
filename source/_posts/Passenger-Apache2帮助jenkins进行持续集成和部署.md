title: Passenger + Apache2帮助jenkins进行持续集成和部署
author: YoooFeng
date: 2017-09-13 17:05:39
categories: 技术相关
tags: [CI&CD, jenkins, passenger, apache]
---
# 简述

上周提到使用jenkins部署rottenpotatoes项目，直接运用jenkins运行网站导致构建过程无法结束。为了解决这个问题，我在网上调研了一些解决方案，一种解决方案是，利用warbler工具，将烂土豆网站打包成war包，再放入tomcat中执行。但是warbler需要用JRuby环境，而且需要配置数据库连接等，配置比较复杂，另一种解决方案是使用phusion-passenger，下文我们使用的就是这种方案。
PhusionPassenger是一个开源的轻量化部署工具，可以跟nginx、apache等集成，使用passenger后，只需要进行一些配置，网站就可以直接使用nginx或者apache部署。
<!--more-->

# Passenger + Apache2的安装和配置

系统中已经有了ruby、rails等环境，接下来主要讲一下passenger的安装：
```
# 安装依赖软件
sudo apt-get install -y dirmngr gnupg
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
sudo apt-get install -y apt-transport-https ca-certificates

# 添加APT repository
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger xenial main > /etc/apt/sources.list.d/passenger.list'

sudo apt-get update

# 安装 Passenger + Apache2
sudo apt-get install -y apache2-dev
sudo apt-get install -y libapache2-mod-passenger
```
安装完成后启用passenger插件：

	sudo a2enmod passenger
	sudo apache2ctl restart
 
重启apache服务，使用命令查看安装是否成功：

	sudo service apache restart
	sudo /usr/bin/passenger-config validate-install
	sudo /usr/sbin/passenger-memory-stats

安装成功后，利用passenger对网站进行部署。假设电脑中安装了多个版本的ruby，先要查看passenger使用的是哪个版本的ruby:

	passenger-config about ruby-command

如果ruby版本不符，使用 rvm use x.x.x 指定使用的ruby版本。

接下来在/etc/apache2/sites-enabled/中新建一个项目的配置文件yourapp.conf，其中yourapp用项目名代替：

```
<VirtualHost *:80>
    ServerName yourserver.com

    # Tell Apache and Passenger where your app's 'public' directory is
    DocumentRoot /var/www/myapp/code/public

    PassengerRuby /path-to-ruby

    # Relax Apache security settings
    <Directory /var/www/myapp/code/public>
      Allow from all
      Options -MultiViews
      # Uncomment this if you're on Apache >= 2.4:
      Require all granted
      Options FollowSymLinks
      AllowOverride None
    </Directory>
</VirtualHost>
```
依次将文件中的ServerName改为本机的IP地址、DocumentRoot指向项目的public文件夹、PassengerRuby指向jenkins下的ruby-2.3.0文件夹、Directory也改成项目的public文件夹。配置完成后重启apache服务，访问http://localhost:80 出现网站界面后，说明部署成功。

# 网站有更新时自动应用更新

我们搭建的jenkins+gitlab CI&CD平台，当jenkins检测到gitlab仓库中的项目有更新时，会自动将更新后的代码下载到本地workspace文件夹中，我们用apache运行的就是workspace文件夹中的网站。当jenkins将更新后的代码下载下来后，passenger和apache运行的还是旧版本的网站，那么如何自动更新呢？

我们直接使用Passenger提供的app-restart命令：

	passenger-config restart-app $(pwd)
    
每次jenkins从gitlab上下载最新的代码时，在jenkins的构建后执行的shell脚本中加入passenger的重启命令即可。

我们还需要对原先的ci_database.yml文件进行一些修改，在文件末尾将production环境加入：

```
#~/ci_database.yml

.......

production: 
  <<: *default
  database: jenkins_pro
  username: postgres
  password: password
```
 原本jenkins中设置的构建脚本更新为：
 
 ```
 #!/bin/bash
export RAILS_ENV=development
source $HOME/.bashrc
cd .  
# Force RVM to load the correct Ruby version

bundle install

cp ~/ci_database.yml ./config/database.yml     
# 将之前写好的数据库配置yml文件拷贝到当前项目中
bundle exec rake db:setup

bundle exec rake assets:precompile db:migrate RAILS_ENV=production
# 初始化数据库

passenger-config restart-app $(pwd)
#passenger提供的更新网站的命令

# bundle exec rake test           
# 运行测试，可以选择使用的minitest或者RSpec
# rails s 
#启动服务器，默认地址为http://localhost:3000
```
这样配置之后，网站的运行就不需要jenkins的命令行后台持续运行，不会导致jenkins任务构建不成功了。部署后的实际效果如下图所示：
![log](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/passenger_apache.pic/passenger_jenkins_log.png)

![rp](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/passenger_apache.pic/passenger_jenkins_rp.png)


