---
layout:     post
title:     Spark with IDEA
subtitle:   大数据处理
date:       2019-10-18
author:     Yi Xia
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - Java
---

# IDEA with maven
IDEA 全称 IntelliJ IDEA，是java语言开发的集成环境，IntelliJ在业界被公认为最好的Java开发工具之一, IDEA是JetBrains公司的产品,现在有逐步取代老牌Java开发工具Eclipse的趋势.那本人也是从Eclipse 转到IDEA.那刚转换过来时,确实很不适应,不过好在坚持使用了几天后,确实感觉IntelliJ IDEA比Eclipse更加智能.
Maven项目对象模型(POM)，是一个项目管理工具可以通过一小段描述信息来管理项目的构建，报告和文档的软件。那我们想要在IDEA中使用Maven得进行一些配置,那接下来
我们具体看一下是如何配置使用的?
## Maven
maven是一款优秀的服务构建工具，基于约定优于配置原则，提供标准的服务构建流程。maven的优点不仅限于服务构建，使用maven能够做到高效的依赖管理，并且提供有中央仓库可以完成绝大多数依赖的下载使用。


![-w680](/img/blog_img/15724225884585.jpg)

maven自身提供有丰富的插件，可以在不使用额外插件的条件下完成服务的编译、测试、打包、部署等服务构建流程，即maven对服务的构建过程是通过多个插件完成的，且maven已经自定义了插件的行为。可以理解为每一个插件都是对接口的实现，可以自定义插件，以完成自定义功能，例如完成对不同编程语言的服务构建过程。不过相对于gradle的自定义插件行为，maven的实现过程略微复杂。



### profile
真实项目中，每一个项目都会有多套环境，包括开发环境，测试环境，灰度机环境以及最终的生产环境，每一套环境对应着不同的配置参数，比如JDBC连接信息肯定会有所差别，如果发布到某一环境中就需要改写一次配置文件，只有一个jdbc.properties还可以接受，想象一下真实项目中的配置文件的数量头就大，更重要的是如果写错了某参数后果将不堪设想！此时利用Maven管理的的另一个长处变显现出来了，利用Maven可以为每一个环境配置一个Profile，编译的时候指定Profile的名字即可达到编译文件按需产生。

### introduce
日常开发中，我们用到的maven相关功能大概以下几种： 
1、 管理jar依赖 
2、 构建项目（打包、编译等） 
3、 发布项目（共享、上传至服务器，供他人使用）


- **settings.xml**：</br>
settings.xml文件用于记录本地仓库、远程仓库以及认证信息等maven工程使用的元素，该文件有两种级别，用户级别和全局级别，存放位置一般为${maven.home}/conf/settings.xml和${user.home}/.m2/settings.xml。Local repository本地仓库用于存放自动下载后的依赖文件和安装到本地的服务。
- **依赖名称**</br>
maven配置文件路径就在maven主程序目录\conf\下，默认名为setting.xml；
这里要注意，在我们自己发布项目时尽量遵守以上规范，否则当别人搜索依赖时会写的很冗余很混乱，这也是很多私服中不断重复上传相同jar会导致项目出错的原因。 
正常来说，一个项目（如spring的jar）应该是机构、名称和版本唯一的，当我们引用时，可以通过这三个参数唯一标识出一个项目

```java
<dependency>
     <groupId>org.springframework</groupId>
     <artifactId> org.springframework .spring-core</artifactId>
     <version>4.1.1.RELEASE</version>
</dependency>

```
- maven 项目执行过程
当我们编译项目时一般会有如下过程： 
1）编译 
2）找到pom文件，并扫描相关依赖 
3）根据pom中依赖配置信息，到本地maven仓库去查找jar包 
4）如果本地没有jar，maven会到它的中心仓库下载jar（ 
 http://www.sonatype.org/nexus/ 
http://mvnrepository.com/） 
5）如果你公司创建了私服，并且你的maven中做了配置，那么在本地没有找到jar包时，会根据maven的配置文件(conf/setting.xml)所配置的地址，到maven的私服仓库去寻找jar，私服也会根据它的配置规则在本地或中央仓库查找jar，如果本地没有就从服务端下载到本地。（私服的优点见下文）
在步骤1 中，编译会将本地*.java编译为字节码文件，其中很多jar依赖于第三方类库，这时候IDE会根据已配置的 maven\conf\下的setting.xml文件中的配置（或IDE的maven仓库路径）去查找相应jar包。如果有jar包，则会拷贝到项目的编译目录下（myEclipse默认为“tomcat\webapp\lib\”目录下；Intellij IDEA 默认在“项目target\webapps\lib”目录下）, 
如果本地没有，那么会先去仓库下载到本地，再从本地拷贝到项目编译目录。 
- 生命周期
maven将项目的生命周期大致分为9个，分别为：clean、validate、compile、test、package、verify、install、site、deploy 
我经常用的也就是clean、compile、package、install、deploy，而且deploy相对也较少，因为很少发布公共的项目供别人依赖使用，基本也就是项目打包为war时候会打包到私服，运维人员可以到私服上直接下载对应版本。 
其中clean即清除项目中编译文件和本地仓库中已打包的文件（即本地install的文件，install后面讲到） 
compile即编译项目中的java文件，并存放在项目的编译目录（根据不同的配置，编译目录也不一样） 
test 即运行项目中的测试用例文件，如果测试用例未通过，也会打包失败，另，这里的test过程可以在pom中通过配置跳过。（想想也是，我项目都好了，其实不是非要跑测试用例的） 
package 即将本地编译好的文件打包为war 或者jar(这是最常见的两种，其他相关自行了解) 
verify 我很少用到，没怎么了解过 
install 将打包的代码存放到本地maven仓库，可供本地其它项目依赖使用 
site生成项目报告，站点，发布站点，这个也很少用到，不是很清楚 
deploy 将打包在本地仓库中的项目发不到服务器，供他人依赖使用 