title: Pipeline的优化与定制
author: YoooFeng
categories: 技术相关
tags:
  - jenkins pipeline
  - jenkins shared library
  - jenkins
date: 2017-10-30 10:15:00
---
### 写在前面

项目需求要求对Jenkins Pipeline的功能尝试进行一些改进。对Jenkins进行了一些调研，将资料和过程记录下来。

### Jenkins的Pipeline

#### Pipeline的源代码
Jenkins的pipeline原先是一个插件，独立于jenkins核心代码，作为一个单独的java类文件在启动时被加载。现在jenkins官方将pipeline作为jenkins的核心功能之一进行维护，加入到了jenkins的核心代码中。目前pipeline在github上叫workflow，并且被拆分成了几个不同插件模块：
+ workflow-step-api-plugin
+ workflow-basic-steps-plugin
+ workflow-api-plugin
+ workflow-support-plugin
+ workflow-job-plugin
+ workflow-durable-task-step-plugin
+ workflow-scm-step-plugin
+ pipeline-build-step-plugin + pipeline-input-step-plugin + pipeline-stage-step-plugin（功能相近）
+ pipeline-stage-view-plugin
+ workflow-cps-plugin
+ workflow-cps-global-lib-plugin
+ workflow-aggregator-plugin。

具体每个plugin的功能可以在pipeline文档中找到：
https://github.com/jenkinsci/pipeline-plugin/blob/master/CONTRIBUTING.md

我想实现的功能是在pipeline的执行流程中插入一个标记，使得原先只能从上到下线性执行的pipeline流程可以实现回溯、跳转、重复执行stage等功能。~~因此，需要先明确上面拆分的这么多插件中，哪一个是我们需要改动的。~~如果对jenkins源码进行了修改，那么我们可能需要跟随jenkins的版本更迭维护修改的部分，所以理想的做法是在jenkins之上再添加一层我们个性化修改之后的东西，这样便于维护和使用。
<!-- more -->
#### 什么是shared library
官方提供一个名叫shared library的机制，允许我们对pipeline流程进行一些优化和定制。通过这个机制，我们可以定义一些变量、函数，以供在pipeline流程之中使用。Shared library的目录结构如下：

(root)  
+- src                     # Groovy source files   
|&emsp;&emsp;  +- org  
|&emsp;&emsp;&emsp;&emsp;   +- foo  
|&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;+- Bar.groovy  # for org.foo.Bar class  
+- vars  
|&emsp;&emsp;+- foo.groovy          # for global 'foo' variable  
|&emsp;&emsp;+- foo.txt             # help for 'foo' variable  
+- resources               # resource files (external libraries only)  
|&emsp;&emsp;+- org  
|&emsp;&emsp;&emsp;&emsp;+- foo  
|&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;+- bar.json    # static helper data for org.foo.Bar  

src目录是存放主要代码的目录，通过@Library标记和import语句，在jenkinsfile中引入我们定义的Groovy函数；在vars目录下，定义全局函数和变量，这些函数和变量在pipeline流程中可以被直接调用。这些我们定义的.groovy文件有很高的权限，可以调用Java、Groovy、Jenkins内部API、Jenkins插件或者第三方库。

#### Shared library - 从jenkins的机制说起
这部分内容主要介绍jenkins的运行机制，以及Shared library是如何集成到jenkins中产生作用的。

Jenkins的运作主要由stapler和jelly两大核心模块构成。简单来说Stapler通过反射机制，将 URL 绑定到应用程序对象。Stapler的工作机制很好理解，比如说Hudson实例被映射到了路由的根路径，对于二级路径/project（Project是一个Java类），即会被映射为Hudson对象的一个getProject方法；如果Project下还有下一级的路由比如/project/status（status同样也是java类），那么getProject就会返回一个带有getStatus方法的对象，依次类推。而Jelly则是一种约定大于配置的页面渲染机制，不展开说了。

Jenkins中的插件实际上是一个一个的Java class，Jenkins在启动时通过单独的类加载器加载每个插件。Jenkins提供了一些basic building block，我们开发插件需要继承相应的类，然后来实现我们的功能。

所以从设计理念上说，Jenkins的core核心代码部分其实只有Stapler、Jelly、还有一些定义的抽象的model、security规则，其余的功能可以都看作是plugin，比如说pipeline，最开始也是一个plugin，后来官方觉得这个功能不错，官方来维护，现在这个插件嵌入了核心代码中（安装时不可选不安装）。

Shared Libraray实际上也是Jenkins的一个插件，名叫workflow-cps-global-lib-plugin。既然是插件，那么必然是java类，也继承了Jenkins提供的basic building block。我们查看shared library的源代码：

```
//workflow-cps-global-lib-plugin/.../GroovyShellDecoratorImpl.java

import groovy.lang.GroovyClassLoader;
import groovy.lang.GroovyShell;
import hudson.Extension;
import org.jenkinsci.plugins.workflow.cps.CpsFlowExecution;
import org.jenkinsci.plugins.workflow.cps.GroovyShellDecorator;
...
@Extension
public class GroovyShellDecoratorImpl extends GroovyShellDecorator {
......
shell.getClassLoader().addURL(new File(repo.workspace,"src").toURI().toURL());

shell.getClassLoader().addURL(new File(repo.workspace, UserDefinedGlobalVariable.PREFIX).toURI().toURL());
...
```
Shared Library实质就是用户编写的Groovy function，插件做的事情就是在将这些.groovy文件作为class，load到Jenkins的Groovy class库中，然后在jenkinsfile中用户可以通过@Libraray标记import和使用自己定义的函数或者变量。在方法的定义之前有一个@Extension标识，功能类似于微服务中的服务发现和服务注册，Jenkins遇到这个标识就会自动生成一个该class的实例，然后把这个class放到Extensionlist中。


#### 在jenkinsfile中使用shared library
在jenkinsfile中，要调用这些shared library，需要标注@Library('Your-Library-name')。在Jenkins job的构建过程中，Jenkins首先检查和下载shared library的更新，然后再执行jenkinsfile。在.groovy文件中我们可以定义多个pipeline，然后根据执行情况和设定的条件选择执行哪一个pipeline，但是一次build只能执行一个pipeline，如果尝试执行第二个，会提示build失败。

### 开始实践

有了以上的背景介绍，接下来我们开始实现我们需要的功能。目标项目是一个利用ant脚本执行简单junit测试的java程序，我预期想要实现的功能包括：
- 重复进行某个stage
- 根据stage的执行结果，跳转执行指定的stage

我用Groovy语言实现了一个简单的函数，当抛出异常时，用户选择可以重试该函数，这个函数包含一个代码块，里面可以写所有steps支持的操作，然后函数在一个stage中被执行。也就是说，我们把本来应该直接写在stage中的steps操作，用一个可以自动重试的函数包裹了一层，将要执行的操作作为函数的一个参数，这样来实现我们就可以通过重复执行这个函数来达到自动重试stage的功能。同样，要想实现执行指定的stage的功能，只需要调用指定的函数即可。

在将这个函数应用到jenkinsfile的过程中，遇到了一些问题，记录下来：
+ @Library、import的标注可以统一写在pipeline之外，整个jenkinsfile的最前面，如果import写在pipeline里面会导致报错，IMPORT type unknown
+ 我们在stage->steps中调用我们定义的函数时，要将代码块放在script{}中
+ 我在按以往的流程使用命令操作时提示文件不存在，查阅资料之后发现，因为函数中的steps是在node代码块中执行的，在jenkins机制下这种方式可以指定一个特定的slave执行，如果没指定，则默认在本机另外新建一个沙箱，在项目名为Your-Project-Name@xxx的目录下对项目进行操作，所以在写node代码的时候要加入check scm命令，把git上的源代码pull下来。
+ Jenkins考虑到安全问题，默认禁止shared library中用户定义的方法再调用和访问jenkins内部的方法，因此我们要想获取一些结果的时候会遇到scripts not permitted to use method.....的报错，我们需要在jenkins中进行设置，点击系统管理，在插件管理选项的下面有一项In-process Script Approval，进入进行设置

我们定义的groovy函数主要代码如下：
```
//org/iscas/yf/PipelineHelper.groovy

package org.iscas.yf
import hudson.tasks.test.AbstractTestResultAction

def class PipelineHelper {
    def steps
    //这个steps对象可以应用所有jenkins支持的step操作，通过steps.sh、steps.echo等方式调用
    
    PipelineHelper(steps) {
        this.steps = steps
    }
	/*主要的函数，retryOrAbort，具体的思路是if语句判断测试是否通过，测试不通过就throw一个异常，这个函数catch到异常之后，由用户选择retry或者retry*/
	
    void retryOrAbort(final Closure<?> action, int maxAttempts, int timeoutSeconds, final int count = 0) {
    if (something wrong) then 
    	return retryOrAbort(...)
```

通过递归调用函数，实现重试。

完整的代码可以在：https://github.com/YoooFeng/PipelineHelper  
用于测试的项目：https://github.com/YoooFeng/junit-sample/


之后，我们点击jenkins的“立即构建”按钮，查看输出的控制台日志，可以发现我们的函数成功的重复执行了三次，实现了我们预期的功能。