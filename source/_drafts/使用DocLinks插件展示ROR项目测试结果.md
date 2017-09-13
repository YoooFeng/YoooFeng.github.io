title: 使用DocLinks插件展示ROR项目测试结果
author: YoooFeng
date: 2017-09-13 14:11:28
tags:
---
# Jenkins中ROR插件调研

## Ruby Plugin
这个插件的工作原理类似于Shell builder，在下图中输入你想要运行的ruby脚本，系统默认的ruby编译器就会去执行文本框中的ruby脚本代码。

 
## *RubyMetrics plugin
一个分析ruby代码的工具，支持分析Rcov -- Code Coverage for Ruby，并且可以将分析的结果自动生成图表展示。举例来说，只需要使用命令Rspec,所有的测试就会自动进行，测试的覆盖率也一并计算，最后生成的是一些图片和表格，对整个测试的情况进行总结。

我们首先在jenkins插件中心安装RubyMetrcs Plugin插件，要使用插件，我们在项目配置中进行设置。
进入“构建后操作”这一栏，点击“增加构建后操作步骤”，选择“Publish Rcov report”，出现如下界面：
（图add_Rcov）

然后我们只需要设置好生成Rcov报告的目录，以及各项指标的的阈值，在项目完成构建后，就会在我们指定的目录生成Rcov Report。除此之外，我们还需要修改一下项目的Gemfile和spec/spec_helper.rb。
我们在Gemfile中加入以下代码：

	# Gemfile
	gem 'simplecov', :require => false, :group => :test
	gem 'simplecov-rcov'

在spec_helper.rb中在文件开头加入以下代码：

```
# spec_helper.rb
require 'simplecov'
require 'simplecov-rcov'
SimpleCov.formatter = SimpleCov::Formatter::RcovFormatter
SimpleCov.start 'rails'
```

RottenPotatoes似乎没有写单元测试的测试用例，rspec无法进行，因此当jenkins执行构建时会显示rcov report index file wasn’t found. Google了一下没有找到解决方案，后面可能需要手动加入几个测试用例。

## Cucumber-report-plugin
一个测试项目代码是否符合Behavioral Driven Design的插件，可以自动生成HTML形式的测试结果，但是目前该插件集成的是cucumber的java实现版本cucumber-jvm。

## Junit Plugin
测试失败，项目构建不成功。

## Brakeman Plugin
对ruby on rails项目进行安全检测的插件，目前无此需求。

# 解决方案
通过调研，最后我选择的解决方法是在命令行中利用shell代码进行rspec测试、生成Rcov报告，然后利用一个通用的jenkins插件DocLinks，将生成的报告集成到项目管理界面上。之后对rails项目进行分析和测试也可以自己选择想进行的测试工具（如cucumber等），然后在命令行中执行测试，最后只需要将结果集成到jenkins中即可。具体的配置过程如下：
首先在jenkins中安装Doclinks插件，这里不再详细描述。

然后编辑项目目录下的Gemfile文件和spec_helper文件：


	#Gemfile
	#Add lines
	gem ‘simplecov’, :require=>false, group: :test
	gem ‘simplecov-rcov’, :require=>false, group: :test

```
#spec_helper.rb
#加在所有测试代码运行之前，即文件最开头
require ‘simplecov’
require ‘simplecov-rcov’

SimpleCov.formatter = SimpleCov::Formatter::HTMLFormatter
SimpleCov.start
```
进入jenkins的项目配置页面，在构建->Execute shell中最后加入运行rspec测试的命令：

	rspec spec

项目配置->构建后操作->增加构建后操作步骤，选择选项publish documents，进行如下配置：

 

参数说明：
Title：显示的文件名；
Description：对文件的说明；
Dictionary to archive：文件所在目录（simplecov默认为项目目录下的coverage文件夹）
Index file：显示的主页面

保存设置，对项目进行构建，查看控制台输出，发现SimpleCov报告已经生成了：
 

回到jenkins项目主页，可以直接点击查看生成的测试报告：

 


测试报告的详情：
 

由于rottenpotato网站的代码中没有写rspec测试代码，所以没有example成功或失败，默认显示的覆盖率为100%。

# 关于Blue Ocean：
Blue Ocean目前支持的jenkins job类型只有Multibranch Pipeline job，因此我们需要新建一个此类型的项目。在创建项目时，Blue Ocean提供了一个线性的创建流程，如果我们在远程仓库中预先写好jenkinsfile，那么当项目创建时，jenkins会根据jenkinsfile自动生成pipeline流程。






	
