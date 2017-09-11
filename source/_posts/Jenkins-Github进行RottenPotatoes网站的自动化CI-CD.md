title: Jenkins+Github进行RottenPotatoes网站的自动化CI/CD
author: YoooFeng
date: 2017-09-06 15:57:58
categories: 技术相关
tags: [jenkins, CI&CD]
---
# 前言
前段时间研究了一些关于jenkins的安装、部署和配置，这次结合gitlab将SaaS教科书中的RottenPotatoes示例网站进行自动化CI/CD的部署。

# 部署环境
+ 操作系统：Ubuntu 16.04
+ Jenkins：2.60.3
+ Gitlab：9.5.2
+ Git：2.7.4
+ Tomcat：8.55
+ Ruby：2.3.0

<!-- more -->
首先在ubuntu上配置ruby on rails环境，先执行如下命令安装：

	apt install -y curl bison build-essential zlib1g-dev libssl-dev libreadline5-dev libxml2-dev git-core nodejs

接着进入jenkins用户下安装ruby：
	
    su - jenkins
    curl -L https://get.rvm.io | bash -s stable --ruby=2.3.0
    echo 'source "/var/lib/jenkins/.rvm/scripts/rvm"' >>        /var/lib/jenkins/.bashrc
    
    source ~/.bashrc
    rvm install 2.3.0
    gem install bundler
    
这样我们需要的ruby、rvm、bundler都安装好了。安装过程中如果有提示jenkins不在sudoers中，需要去/etc/sudoers中修改jenkins的权限为ALL。

接下来安装postgre数据库：

	apt-get install -y libpq-dev postgresql postgresql-client
    
设置用户名、密码：

	 sudo -u postgres psql -c "ALTER USER postgres PASSWORD'password';"


关于如何安装及配置jenkins，之前的博客已经写过，此处不再赘述。下面重点讲一下jenkins与gitlab的webhook的配置。

## 配置SSH
首先需要在jenkins和gitlab上配置ssh。在机器上进入jenkins账号
	
    su - jenkins
    
然后进入jenkins的.ssh目录，ubuntu16.04下是 /var/lib/jenkins/.ssh(若.ssh文件夹不存在则用mkdir新建)，然后运行命令ssh-keygen,一路空格下去就行了。接着使用命令：
	
    cat id_rsa.pub
    
复制公钥，接着在Gitlab -> User settings -> SSH keys中添加公钥。

## 配置webhook  
首先新建一个job，选择构建一个free-style项目，然后对项目进行配置，设置好gitlab仓库的地址与ssh。
	
![Job](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/Jenkins_rp.pic/jenkins_jobs.png)

    
配置jenkins与gitlab的webhook，勾选如下选项，并复制jenkins项目的项目地址：
	
![webhook_jenkins](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/Jenkins_rp.pic/webhook_jenkins.png)
    
接着在Gitlab上添加一个webhook，进入项目的settings -> integration -> add webhook, 然后输入刚才在jenkins得到的地址链接：

![webhook_gitlab](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/Jenkins_rp.pic/webhook_gitlab.png)
    
点击test进行测试，查看返回结果是否正常,若返回HTTP状态码为200，表示webhook配置成功。
    
配置过程中也发现了一些问题，目前jenkins更新很快，有些插件很久没有更新，导致发生一些错误。比如说Gitlab hook plugin给的hook url为
	
    http://youip:yourport/project/yourproject

但是实际上新版本的jenkins已将项目路径更名为

	http://youip:yourport/job/yourproject
    
如果还使用原先的url会导致404错误。

还有，webhook可能会返回403错误，查看错误日志，发现这是由于jenkins拒绝匿名（anonymous）访问，没有授权成功导致的。我们需要将gitlab中webhook url的格式改为

	http://username:user-api-token@gitlab-path/job/yourproject
    
其中username与user-api-token可以在jenkins->系统设置->管理用户->设置中找到，如图：
	
![jenkins_api](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/Jenkins_rp.pic/jenkins-user-apitoken.png)
  
这样webhook就配置好了，当开发者向gitlab push代码时，jenkins会将更新代码fetch到本地的workspace目录下。

在实际的应用中，我们使用每次push都让jenkins构建一次照成的成本太高，因此我在jenkins中设置的是周期性地检查代码是否有更新，如果有更新就将最新代码下载下来并进行构建和部署。我们在jenkins的项目设置中，将poll scm一栏打勾，并在下面的文本框中写上 

	H/5 * * * *
    
表示每五分钟进行一次更新检查，这个数值可以根据项目需要更新的频率自行设置。

# 项目配置
接下来为项目配置ruby的运行环境。首先在jenkins的根目录下维护一个文件ci_dababase.yml，每次配置时将数据库的配置文件都写入项目目录，如果对于数据库有修改，只需要修改配置文件即可，如果没有修改，cp操作花费的成本也比较低。

```
# ~/ci_database.yml

dafault: &default
  host: localhost
  adapter: postresql
  encoding: unicode
  pool: 5
  
test:
  <<: *default
  database: jenkins_test
  username: postgres
  password: password
  
development: 
  <<: *default
  database: jenkins_dev
  username: postgres
  password：password
```

接着进入项目配置界面，在 构建 一栏中，增加一个构建步骤，选择Execute Shell，输入以下shell脚本：

```
#!/bin/bash
export RAILS_ENV=development
source $HOME/.bashrc

# cd .  
# Force RVM to load the correct Ruby version


cd rottenpotatoes-rails-intro
#进入项目目录

bundle install

cp ~/ci_database.yml ./config/database.yml     
# 将之前写好的数据库配置yml文件拷贝到当前项目中

bundle exec rake db:setup
# 初始化数据库

bundle exec rake test           
# 运行测试，可以选择使用的minitest或者RSpec

rails s
#启动服务器，默认地址为http://localhost:3000
```

上面这段shell脚本可以让jenkins检测到更新的时候将最新代码自动下载、部署到本地服务器上，项目的构建脚本可以根据实际需求灵活编写，更推荐的方式是在项目中维护一个脚本文件，这里只要写两行shell命令让jenkins去执行项目中的脚本就可以了。查看jenkins的部署日志，打开rails默认端口，可以看到网站已经成功部署到本地服务器上了:

![jenkins_log](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/Jenkins_rp.pic/jenkins_log.png)

![jenkins_rp](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/Jenkins_rp.pic/rottenporatoes.png)
    
但是目前还有一个小问题，就是构建之后要保持rottenpotatoes网站的运行需要jenkins的控制台在后台持续运行，导致jenkins的构建过程无法结束，理想的情况应该是jenkins指定tomcat或者其他什么代理进行网站的运行，后面还需要调研一下方法。



部署成功之后，我准备研究一下jenkins插件中对于ruby的代码风格、程序框架以及一些测试的功能如rspec等有没有一些后续支持的插件。