title: Jenkins插件作用机制
author: YoooFeng
categories: 技术相关
tags:
  - jenkins插件
  - jenkins
date: 2017-09-14 19:34:25
---
# Jenkins插件作用机制调研
Jenkins的插件实际上就是一个一个的Java class，jenkins 通过单独的类加载器加载每个插件以避免插件之间产生冲突。插件就像 Jenkins 内置的其他类一样参与到系统的活动中。另外，插件可以通过 XStream 持久化，可以由 Jelly 提供视图技术，可以提供图片等静态资源，插件中所有的功能可以无缝的加入到 Jenkins 内置的功能上。

我们开发的插件要想集成到jenkins中，在编写的时候就需要继承对应的jenkins提供的模板类，比方说我们要想编写一个参数类的插件，就需要（必须）继承jenkins官方提供的ParameterDefinition模板类，所有参数相关的插件都需要继承这个类。
<!-- more -->

ParameterDefinition这个类中提前写好了配置job时输入参数的文本框，只有继承这个类，才能在job配置页面中的Add Parameter下拉框中看到我们编写的插件。换句话说，Jenkins使用继承模板类的方式将Jenkins本身与插件的接口提供给我们，我们也只能通过继承模板类的方式将我们的插件集成到jenkins中。这些模板中有一些基本的方法，例如文本框的名称，一些图标的定义等，我们可以通过override这些方法来进行个性化设计。

Jenkins提供了一系列的basic building block，以供我们进行插件开发，这些blocks其实就是之前作者写的hudson包中的抽象类，比如说Action、Plugin、Api、RSS等。

然后是我们要考虑的Jenkins插件是否可以进行远端服务器通信。在默认的情况下，Jenkins只为使用了Content-Security-Policy header的有限制的静态文件提供服务，来保证服务器的安全，CSP实际上就是一个白名单制度，开发者指定来自哪些站点的资源可以接收、加载。我们有这样的需求，需要jenkins接受来自第三方网站的测试报告（HTML页面或者别的什么），这样的报告来自我们部署该服务的站点，但是我们把插件做成一个服务，不可能要求每一个用户在使用插件之前先修改自己本地的jenkins CSP规则，所以我们需要想一个折中的方法，绕过CSP的机制，建立远程连接，展示我们的测试报告。

Jenkins官方提供了Remote Access API相关功能及扩展，使得大部分jenkins的项目资源都可以远程进行访问。要想知道特定对象有哪些api接口，只需要在路径之后加入/api/即可，举个例子，访问我的jenkins服务器地址：localhost:8888/api/，可以看到jenkins系统层面暴露到外界的api接口，可以重启jenkens服务器、拷贝jenkins中的项目资源等；访问localhost:8888/job/myproject/api/，可以看到在项目层面jenkins提供的api接口，可以远程进行构建、删除该项目等。

Jenkins内部维护的数据模型可以想象成一棵树，当用户向jenkins发送远程API请求时，相当于只向jenkins请求整个数据树的某棵子树，这棵子树的根节点即为用户请求的某一个job，这棵子树从根节点切断，这样做可以避免jenkins向用户发送过多数据，而只专注于某一个项目的数据。要是有特殊的需求，也可以进行调节，将这个数据子树从根节点向下扩展，调节节点的深度，获取更多的数据。具体的应用还需要进一步调研。

Jenkins目前提供三种方式使用API接口，XML、JSON还有python。官方文档中演示了使用curl http的方式使用API接口。

```
curl -X POST JENKINS_URL/job/JOB_NAME/build \
  --user USER:TOKEN \
  --data-urlencode json='{"parameter": [{"name":"id", "value":"123"}, {"name":"verbosity", "value":"high"}]}'
```

Jenkins支持使用USER_TOKEN和USER_NAME&USER_PASSWORD的方式进行权限判定。但是目前jenkins还不支持以这种方式从第三方网站接收数据。

# 插件源码分析 – 以cucumber为例
为了弄清楚jenkins插件的具体实现机制，我以Cucumber-reports-Plugin这个插件为例对Jenkins插件的源代码进行分析。

下面是cucumber插件的项目结构：
![structure](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/jenkins-plugin.pic/cucumber_pi_structure.png)

整个项目的组织结构如上图所示，其中主要部分由CucumberReportBaseAction、CucumberReportDescriptor、CucumberReportProjectAction、CucumberReportPulisher、SafeArchiveServingAction、SafeArchiveServingRunAction等六个Java类组成。

## CucumberReportBaseAction
这个java类是继承自jenkins的hudson.model.Action类，Action类是在jenkins的某个界面上新建一个图标和链接，指向一个子空间，这个子空间显示一些插件本身提供的东西与用户交互。

因此，这个类的主要作用就是，定义插件的名称、指向子空间的URL链接，还有插件显示的图标。

## CucumberReportDescriptor
继承自hudson.tasks. BuildStepDescriptor类。类中的声明如下：

	public class CucumberReportDescriptor extends BuildStepDescriptor<Publisher> {

这个类继承自BuildStepDescriptor，使用的是Publisher模板。BuildStepDescriptor有两个template，一个是Builder，一个是Publisher，当指定模板为Builder时，说明这个插件是作用在构建过程中的，这个插件出现在jenkins项目设置中的 构建 一栏；当指定模板为Publisher时，说明这个插件是项目构建结束之后发挥作用的，出现在 构建后操作 一栏。

Cucumber插件的测试报告显然是需要等项目成功构建之后才能发挥作用的，所以是指定为Publisher。在 构建 或者 构建后操作 选择使用此插件后，需要用户输入的一些参数也是在这里指定的，比如下面的几行代码：

```
   public ListBoxModel doFillBuildStatusItems() {
      return new ListBoxModel(
         // default option should be listed first
         new ListBoxModel.Option(Messages.BuildStatus_unchanged(), null),
         new ListBoxModel.Option(Messages.BuildStatus_FAILURE(),Result.FAILURE.toString()),
         new ListBoxModel.Option(Messages.BuildStatus_UNSTABLE(),
Result.UNSTABLE.toString()));
    }
```

这几行代码作用就是新建了一些ListBox，供用户对插件进行配置。所以，这个类的主要作用就是在jenkins的项目配置界面中嵌入插件，使之对用户可用。

## CucumberReportProjectAction
类的声明如下：

	public class CucumberReportProjectAction extends CucumberReportBaseAction implements ProminentProjectAction {

继承自之前的CucumberReportBaseAction类，同时实现了ProminentProjectAction接口。之前的CucumberReportBaseAction类只是设置了插件的名称、图标等，这个类就具体指定了这个插件的子界面链接出现在项目的主页面，下面这张图指的就是项目的主界面：
![project-page](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/jenkins-plugin.pic/project_page.png)


## *CucumberReportPulisher
这个类就是该插件的主要实现类了，它的声明如下：

	public class CucumberReportPublisher extends Publisher implements SimpleBuildStep {

继承了Publisher类，还有实现了jenkins.tasks.SimpleBuildStep类。这个类中实现了SimpleBuildStep中提供的perform方法，这个方法中是插件方法的具体实现与执行。

关于Cucumber的功能，作者只需要直接import所有cucumber的java实现包，这些包往往都提供了一系列的接口，因此只要将这些接口应用到代码中即可。

 
从机制上来看，如果我们需要的功能已经有了现成的java实现，我们只需要进行一些简单的jenkins页面的编辑和设定，然后引入这些java实现包作为API使用即可。对于那些没有java实现的功能，可能就需要我们用java自己实现了。

类中还有一个关键字：@DataBoundSetter。这个来自包org.kohsuke.stapler.DataBoundSetter，是stapler的一部分。Stapler的运行机制是，首先调用DataBoundConstructor方法，如果里面定义的属性在JSON文件中存在，那么就去找这个属性的DataBoundSetter，将JSON文件中的属性改为代码指定的值。setter中属性的名字必须与JSON中的名字一致。

## SafeArchiveServingAction&SafeArchiveServingRunAction
这两个类是用来生成和展示其他格式的cucumber报告的。如上文所说，jenkins是不运行加载和运行没有CSP header的HTML页面的，cucumber生成的报告是HTML格式，而且显然没有CSP Header。插件作者提出了一个目前很多Plugin都在使用的一个替代解决方法，使用hudson.model.DirectoryBrowserSupport类为生成的报告提供服务。DirectoryBrowserSupport的工作原理是，当项目进行构建的时候，首先扫描一次项目目录，记录所有文件的checksum，当要求为这些文件提供服务时，检查文件的checksum是否一致，只为一致的文件提供服务。


# Jenkins插件开发
## 环境配置
+ Jenkins本身是使用java编写的，所以要想开发jenkins插件，需要安装JDK8开发环境。
+ 安装Maven开发环境。
+ 配置Maven。在目录~/.m2下新建文件settings.xml，输入以下内容：

```
<settings>
  <pluginGroups>
    <pluginGroup>org.jenkins-ci.tools</pluginGroup>
  </pluginGroups>

  <profiles>
    <profile>
      <id>jenkins</id>
      <activation>
        <activeByDefault>true</activeByDefault> 
      </activation>
      <repositories> 
        <repository>
          <id>repo.jenkins-ci.org</id>
          <url>https://repo.jenkins-ci.org/public/</url>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>repo.jenkins-ci.org</id>
          <url>https://repo.jenkins-ci.org/public/</url>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
```
+ pluginGroup ：允许在命令行中使用缩写hpi代替org.jenkins-ci.tools:maven-hpi-plugin；
+ activeByDefault：如果同时有其他开发项目在使用maven，设置为false；
+ repositories：执行hpi:create命令需要。

在IDE中进行配置，以IDEA为例：

![idea-config](https://raw.githubusercontent.com/YoooFeng/YoooFeng.github.io/hexo/source/_posts/jenkins-plugin.pic/idea_config.png)


## 创建插件开发项目
输入命令mvn -u hpi:create。然后命令台会出现提示：

```
Enter the groupId of your plugin [org.jenkins-ci.plugins]: 
Enter the artifactId of your plugin (normally without '-plugin' suffix): demo 
[INFO] Defaulting package to group ID + artifact ID: org.jenkins-ci.plugins.demo 
[WARNING] Package name contains invalid characters. Replacing it with ‘org.jenkinsci.plugins.demo'
```

+ groupId：开发者信息，可以使用默认值；
+ artifactId：插件名，也会作为项目的文件夹名。

新建项目之后，使用以下命令查看构建是否成功：

	$ cd demo 
	$ mvn verify

随后就可以进入正常开发，过程中使用如下命令进行运行，此命令会打开一个本地的带当前开发插件的jenkins服务器，地址为http://localhost:8080/jenkins：

	$ mvn hpi:run

如果需要断点调试，可以运行如下命令，此命令会在8000端口建立监听，然后可以在IDE中配置对应的Run/Debug Configuration，配置完后运行Debug就会自动触发插件编译和运行流程。

	$ mvnDebug hpi:run

## 打包插件
完成开发后，只需要使用命令：

	$ mvn clean install

就可以将编写的插件代码打包成一个.hpi文件（相当于一个jar包），然后上传到jenkins的服务器，这样就可以安装使用插件了。

# 总结&计划 
目前已经明确的是，第三方站点从jenkins获取项目文件、进行测试、生成测试报告是可行的，具体的实现方法还需要调研。但是生成了测试报告后，将这个报告集成到jenkins中，更远一点说，用这个报告的结果影响jenkins的pipeline过程，目前来看还比较困难。

接下来几周准备继续熟悉jenkins的插件机制和官方提供的API接口，并且自己动手开发一个简单的jenkins插件，主要目的是实现jenkins与远端服务器的通信和数据交互。