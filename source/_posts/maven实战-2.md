---
title: maven实战-2
copyright: true
date: 2018-09-19 01:25:57
tags: maven
categories: maven

---

# 使用maven进行测试

maven的重要职责就是自动运行单元测试，它通过maven-surefire-plugin与主流的单元测试框架junit4 以及testNG继承，并且能够自动生成丰富的结果报告。

## maven-surefire-plugin 插件简介

maven-surefire-plugin可以被称作测试运行器。生命周期阶段需要绑定到某个插件的目标才可以完成真正的工作。test阶段正是和maven-surefire-plugin的test目标绑定的。

在默认情况下，maven-surefire-plugin插件的test目标会自动执行测试源码路径（src/test/java）下符合一组命名模式的测试类。

1.**/Test*.java 任何子目录下所有命名以Test开头的java类。
2.**/*Test.java 任何子目录下所有命名以Test结尾的Java类
3.**/*TestCase.java 任何子目录下所有命名以testCase结尾的java类。

只要按照以上模式命名，maven就可以自动运行他们。当然可以自定义测试类的模式。

当然为了运行测试，maven需要自己引入测试框架的依赖。

## 跳过测试
以下配置可以跳过测试类

````
maven package -Dmaven.test.skip =true
````

也可以采用配置的形式。

````

<!-- 编译插件 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>1.7</source>
        <target>1.7</target>
        <!-- 略过测试代码的编译，不推荐 -->
        <skip>true</skip>
    </configuration>
</plugin>
<!-- 单元测试插件 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <!-- 略过整个单元测试的执行，不推荐 -->
        <skip>true</skip>
    </configuration>
</plugin>

````

## 动态指定要运行的测试用例

1.指定单个测试用例

mvn test -Dtest =Randm*Test

2.指定多个测试用例

mvn test -Dtest = ATest,BTest

## 包含与排除测试用例

即使不符合测试类命名模式，即便如此，maven-surefire-plugin插件还是允许用户通过额外的配置自定义包含一些其它测试类，或者排除一些复合默认命名模式的测试类。

````
< plugin >   
    < groupId > org.apache.maven.plugins </ groupId >   
    < artifactId > maven-surefire-plugin </ artifactId >   
    < version > 2.5 </ version >   
    < configuration >   
        < includes >   
            < include > **/*Tests.java </ include >   
        </ includes >   
        < excludes >   
            < exclude > **/*ServiceTest.java </ exclude >   
            < exclude > **/TempDaoTest.java </ exclude >   
        </ excludes >   
    </ configuration >   
</ plugin >  
````
## 生成测试报告

### 基本测试报告

默认情况下，maven-surefire-plugin会在项目的target/surefire-reports目录下生成两种格式的错误报告。

-- 简单文本格式——内容十分简单，可以看出哪个测试项出错。
-- 与JUnit兼容的XML格式——XML格式已经成为了Java单元测试报告的事实标准，这个文件可以用其他的工具如IDE来查看。

### 测试覆盖率报告

测试覆盖率是衡量项目代码质量的一个重要的参考指标。Cobertura是一个优秀的开源 [测试覆盖率统计工具]( http://cobertura.sourceforge.net) ，Maven通过cobertura-maven-plugin与之集成，用户可 以使用简单的命令为Maven项目生成测试覆盖率报告。运行下面命令生成报告：

mvn cobertura:cobertura 

在target/site/cobertura下的index.xml文件可以看到覆盖率报告。

### 重用测试代码

当命令行运行mvn package的时候，Maven只会打包主代码及资源文件，并不会对测试代码打包。如果测试代码中有需要重用的代码，这时候就需要对测试代码打包了。
这时候需要配置maven-jar-plugin将测试类打包，如：

````
< plugin >   
    < groupId > org.apache.maven.plugins </ groupId >   
    < artifactId > maven-jar-plugin </ artifactId >   
    < version > 2.2 </ version >   
    < executions >   
        < execution >   
            < goals >   
                < goal > test-jar </ goal >   
            </ goals >   
        </ execution >   
    </ executions >   
</ plugin >  

````

maven-jar-plugin有两个目标，分别为jar和test-jar。这两个目标都默认绑定到default生命周期的package阶段运行，只是test-jar并没有在超级POM中配置，因此需要我们另外在pom中配置

# 使用maven构建web应用

之前我们讨论的都是JAR或者POM的Maven项目，但在现今的互联网时代，我们创建的大部分应用程序都是Web应用，基于Java的Web应用，其标准的打开方式是war。
WAR与JAR相似，只不过他可以包含更多的内容，如JSP文件、Servlet、Java类、web.xml配置文件、依赖JAR包、静态web资源（如HTML、CSS、JavaScript文件）等。一个典型的WAR文件会有如下目录结构：

![war文件的目录结构](/images/maven/war文件的目录结构.png)

![maven的web目录结构](/images/maven/maven的web目录结构.png)

1.需要注意的时war文件的目录下有一个lib文件夹，而maven项目下没有，这是因为maven项目的所有依赖都在pom文件中，只有在打包的时候才会被下载到lib文件夹中。

2.web应用全部都是需要依赖servlet-api和jsp-api这两个几乎所有的web应用都要依赖的包。因为他们胡server和api的编写提供支持，但是他们的依赖范围是provided，表示他们不会被打包到war文件中，因为几乎所有的web容器都会提供这两个类库。

3.在一些web项目中，读者可能胡看到finalName元素的配置，该元素用来标识项目生成主构件的名称，该元素默认值已经在超级pom中配置，值为${project.artifactId}-${project.version}.例如<finalName>course</finalName>



## 使用jetty-maven-plugin 进行测试

传统的web测试要求我们编译/测试/打包及部署，这会消耗大量的时间，jetty-maven-plugin能够节约我们的时间，它可以周期性的减产项目内容，便跟后更新到内置的jetty web容器中，
jetty-maven-plugin默认就很好地支持了Maven的项目目录，通常情况下插件发现编译后的变化后，自动将其更新到Jetty容器，就可以直接测试Web页面了

````
<build>  
    <finalName>rop-sample</finalName>  
    <plugins>  
        <!-- jetty插件 -->  
        <plugin>  
            <groupId>org.mortbay.jetty</groupId>  
            <artifactId>maven-jetty-plugin</artifactId>  
            <version>6.1.5</version>  
            <configuration>  
                <webAppSourceDirectory>src/main/webapp</webAppSourceDirectory>  
                <scanIntervalSeconds>3</scanIntervalSeconds>  
                <contextPath>/</contextPath>  
                <connectors>  
                    <connector implementation="org.mortbay.jetty.nio.SelectChannelConnector">  
                        <port>8088</port>  
                    </connector>  
                </connectors>  
            </configuration>  
        </plugin>  
    </plugins>  
</build> 

其中scanIntervalSecond表示扫描项目变更的时间间隔，默认为0，表示不扫描。
ContextPath表示项目的访问路径，比如此：http://localhost:8787/test/

port表示绑定的端口号，默认监听的端口是8080。

````
[intellij idea使用配置jetty maven 插件](https://blog.csdn.net/machao0903/article/details/73481095)

[解决maven热部署webapp下的文件不生效的问题](http://stamen.iteye.com/blog/1933452)

下一步是启动jetty-maven-plugin，不过在此之前需要对settings.xml做个微小的改动，默认的只有org.apache.maven.plugins和org.codehaus.mojo两个groupId下的插件支持简化的命令行调用。

mvn help:system 可以运行是因为groupId是org.apache.maven.plugins。而jetty-maven-plugin的插件不是，为了可以运行，需要在settings.xml中配置

````
<settings>
  <pluginGroups>
  <pluginGroup>
    org.mortbay.jetty
  </pluginGroup>
   <pluginGroups>
</settings>

````
然后可以使用 mvn:jetty:run命令，默认监听的是8080端口。

## 使用cargo实现自动化部署

为了能在命令行中使用cargo，需要修改maven的settings.xml文件，修改如下所示：

````
<pluginGroup>org.codehaus.cargo</pluginGroup>
````

Cargo支持两种本地部署的方式：standalone模式和existing模式。

### standalone模式

Cargo会从web容器的安装目录中复制一份配置到用户指定的目录，然后在此基础上部署应用，每次重新构建的时候，这个目录会被清空，所有配置被重新 生成。

````
<plugin>
    <groupId>org.codehaus.cargo</groupId>
    <artifactId>cargo-maven2-plugin</artifactId>
    <version>1.4.0</version>
    <configuration>
        <container>
            <containerId>tomcat6x</containerId>
            <home>D:\Program Files\apache-tomcat-6.0.18</home>
        </container>
        <configuration>
            <type>standalone</type>
            <home>${project.build.directory}/tomcat6x</home>
             <properties> <cargo.servlet.port>8181</cargo.servlet.port> </properties>
        </configuration>
    </configuration>
</plugin>

````

对参数做如下说明：

containerId表示容器的类型。

home表示容器的安装目录。

configuration子元素type表示部署模式。

configuration子元素home表示复制容器配置到什么位置，其中${project.build.directory}表示target目录。

Cargo.servlet.port表示绑定的端口号，默认为8080。

打开命令提示符，执行如下命令：

mvn clean package 和 mvn cargo:run 

或者 mvn clean package cargo:run

### existing模式

在该模式中，指定现有的web容器配置目录，cargo会直接使用这些配置并将应用部署到对应的位置。

````

<plugin>
    <groupId>org.codehaus.cargo</groupId>
    <artifactId>cargo-maven2-plugin</artifactId>
    <version>1.4.0</version>
    <configuration>
        <container>
                <containerId>tomcat6x</containerId>
                <home>D:\Program Files\apache-tomcat-6.0.18</home>
        </container>
        <configuration>
            <type>existing</type>
            <home>D:\Program Files\apache-tomcat-6.0.18</home>
        </configuration>	
    </configuration>
</plugin>

````
执行如下命令：
````
mvn clean compile

mvn clean package

mvn cargo:run
````
之后cargo会自动把war包部署到D:\Program Files\apache-tomcat-6.0.18\webapps中。

### 部署至远程web容器

````
<plugin>
    <groupId>org.codehaus.cargo</groupId>
    <artifactId>cargo-maven2-plugin</artifactId>
    <version>1.4.0</version>
    <configuration>
        <container>
            <containerId>tomcat6x</containerId>
            <type>remote</type>
        </container>
        <configuration>
            <type>runtime</type>
            <properties>
                <cargo.remote.uri>http://10.64.204.188:8080/manager</cargo.remote.uri>
                <cargo.remote.username>admin</cargo.remote.username>
                <cargo.remote.password>admin</cargo.remote.password>
            </properties>
        </configuration>
    </configuration>
</plugin>

 mvn clean compile
 mvn clean package
 mvn cargo:redeploy

执行上述命令后可以从远程tomcat的控制台中就可以看到部署的项目信息。
````

# 版本管理

版本管理是指项目整体版本的演变过程管理，版本控制是指借助版本控制工具追踪代码的每一个变更。

快照版本和发布版本。快照版本对应了项目的开发过程，往往对应了很长的时间，而正式版本对应了项目的发布，因此仅仅代表某个时刻项目的状态。

![快照版本和发布版本之间的转换](/images/maven/快照版本和发布版本之间的转换.png)

满足发布版本的条件：

1.所有自动化测试应当全部通过；
2.项目没有配置任何快照版本的依赖；
3.项目没有配置任何快照版本的插件；
4.项目所包含的代码已经全部提交到版本控制系统中。


## 版本号的定义

<主版本>.<次版本>.<增量版本>-<里程碑版本>

主版本表示项目的重大架构变更；

次版本表示较大范围的功能增加和变化以及bug修复；

增量版本表示重大bug的修复；（非必须）

里程碑版本 与正式版本相比表示不是非常稳定，需要很多测试；（非必须）

## 主干 标签和分支

主干表示项目开发代码的主体；分支表示从主干的某个点分离出来的代码拷贝；tag表示主干或者分支的某个点的状态，表示项目的稳定状态，通常时发布状态。

## 自动化版本发布

流程自动化。maven release plugin提供了这样的功能。主要有三个目标

1.release：prepare 准备版本发布，一次执行下列操作：

- 检查项目是否有未提交的代码；
- 检查项目是否有快照版本的依赖；
- 根据用户的输入将快照版本升级为发布版；
- 将pom中的scm信息更新为标签地址；
- 基于修改后的pom执行maven构建；
- 提交pom变更；
- 基于用户输入人为代码打标签；
- 将代码从发布版升级为新的快照版本；
- 提交pom变更；

2.release：rollback 回退release：prepare所执行的操作，将pom回退至该操作之前的状态，并提交。需要注意的是不会删除标签，需要用户手动删除。

3.release: perform 执行版本发布，签出 release:prepare生成的标签中的源代码，并在此基础上执行mvn deploy 命令打包并部署构件至仓库；

要为项目发布版本，首先需要为其添加正确的版本控制信息，这是因为maven release plugin需要知道版本控制系统的主干，标签等地址信息才能执行相关的操作。
一般的scm信息如代码所示

````
<scm>
 <connection>scm:svn:http://192.168.1.103/app/trunk</connection> --表示只读的scm地址
 <developerConnection>scm:svn:https://192.168.1.103/app/trunk</developerConnection>--表示一个可写的scm地址
 <url>http://192.168.1.103/account/trunk</url>--表示可在浏览器访问的url地址
</scm>

<plugin>
 <groupId>org.apache.maven.plugins</>
 <artifactId>maven-release-plugin</artifactId>
 <version>2.0</version>
 <configuration>
  <tagBase>https://192.168.1.103/app/tags/</tagBase>
 </configuration>
</plugin>
注意：
1、本机必须安装命令行可用的svn 
2、pom必须配置了可用的部署仓库

````
一切就绪之后，在项目的根目录下运行如下命令：
mvn release:prepare 之后，maven release plugin开始准备发布版本，如果检查到项目有未提交的代码，或者有依赖快照版本的会提示出错；
如果没有问题，则会提示用户输入想要发布的版本号/标签的名称以及新的快照版本号。

如果标签等信息配置错误，则可以输入release：rollback命令回退发布，maven release plugin会将pom的配置回退到release：prepare之前的状态，但需要注意的是版本控制系统的标签并不会被删除。

如果是多模块的项目，则会提示分别输入每个模块的发布版本号。

运行如下的命令后，maven-release-plugin就会自动为所有子模块使用与父模块相同的发布版本和新的SNAPSHOT版本：

mvn release：prepare -DautoVersionSubModules=true

如果所有的没有问题执行 mvn release:perform。该命令将标签中的代码签出，执行mvn deploy 命令构建刚才准备的1.0.0版本，并部署到仓库中。至此版本1.0.0发布完成。

# 灵活的构建

Maven为了支持构建的灵活性，内置三大属性，即：属性   Profile  资源过滤

## 六大属性

1.内置属性 ${basedir} 项目根目录， ${version} 表示项目版本。
2.pom属性

````
${project.build.sourceDirectory}：项目的主源码目录，默认为src/main/java/。
${project.build.testSourceDirectory}：项目的测试源码目录，默认为src/test/java/。
${project.build.directory}：项目构建输出目录，默认为target/。
${project.outputDirectory}：项目的主代码编译输出目录，默认为target/classes/。
${project.testOutputDirectory}：项目测试代码编译输出目录，默认为target/test-classes/。
${project.groupId}：项目的groupId。
${project.artifactId}：项目的artifactId。
${project.version}：项目的version。
${project.build.finalName}：项目打包输出文件的名称，默认为${project.artifactId}-${project.version}

````
3.自定义属性

<properties> 
 <sys.version> test </sys.version>
</properties>


4.settings属性

用户以settings来头的属性引用settings文件中的xml元素的值。如常使用的${settings.localRepository} 执行本地仓库的地址

5.java系统属性

所有java系统属性都可以使用maven属性引用，${user.home} 指向了用户目录。

6.环境变量属性

所有环境变量属性都可以使用env.开头的maven属性引用。例如：${env.JAVA_HOME}指代了JAVA_HOME环境变量。

## 构建环境的差异

使用不同的profile

````
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <env>dev</env>
        </properties>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
    <profile>
        <id>prd</id>
        <properties>
            <env>prd</env>
        </properties>
    </profile>
</profiles>

````

# 生成项目站点

使用 maven-site-plugin


````
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.chenwj</groupId>
    <artifactId>spring-config</artifactId>
    <version>1.0-SNAPSHOT</version>
    <!-- 问题解决信息 -->
    <issueManagement>
        <system>Linux</system>
        <url>http://www.baidu.com/</url>
    </issueManagement>
    <!-- 持续集成信息 -->
    <ciManagement>
        <url>http://127.0.0.1:8080/hudson</url>
        <system>windows</system>
    </ciManagement>
    <!-- 开发人员信息 -->
    <developers>
        <developer>
            <id>liuyan</id>
            <email>suhuanzheng7784877@163.com</email>
            <name>liuyan</name>
            <organization>uxian99</organization>
            <roles>
                <role>softwareengineer</role>
            </roles>
            <timezone>8</timezone>
        </developer>
    </developers>
    <!--许可证 -->
    <licenses>
        <license>
            <url>http://127.0.0.1:8080</url>
            <comments>评论</comments>
            <name>完全开源</name>
        </license>
    </licenses>
    <scm>
        <connection>
            scm:svn:https://liuyan:111111@127.0.0.1:8443/svn/mysvn/mysrc/01-OpenSource/maven/MavenAccount-aggregator
        </connection>
        <developerConnection>
            scm:svn:https://liuyan:111111@127.0.0.1:8443/svn/mysvn/mysrc/01-OpenSource/maven/MavenAccount-aggregator
        </developerConnection>
        <url>https://127.0.0.1:8443/svn/mysvn/mysrc/01-OpenSource/maven/MavenAccount-aggregator
        </url>
    </scm>
    <reporting>
        <plugins>
            <!-- 构建项目站点报告插件-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-site-plugin</artifactId>
                <version>3.0-beta-3</version>
                <configuration>
                    <!-- 配置站点国际化 -->
                    <locales>zh_CN</locales>
                    <!-- 输出编码 -->
                    <outputEncoding>GBK</outputEncoding>
                </configuration>
            </plugin>
            <!-- 构建项目站点报告插件-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-site-plugin</artifactId>
                <version>3.0-beta-3</version>
                <configuration>
                    <!-- 配置报告信息 -->
                    <reportPlugins>
                        <!-- 检查代码规范报告 -->
                        <plugin>
                            <groupId>org.apache.maven.plugins</groupId>
                            <artifactId>maven-checkstyle-plugin</artifactId>
                        </plugin>
                        <!-- 测试报告 -->
                        <plugin>
                            <groupId>org.apache.maven.plugins</groupId>
                            <artifactId>maven-surefire-report-plugin</artifactId>
                        </plugin>
                        <!-- 项目基本信息报告 -->
                        <plugin>
                            <groupId>org.apache.maven.plugins</groupId>
                            <artifactId>maven-project-info-reports-plugin</artifactId>
                            <version>2.2</version>
                            <configuration>
                                <dependencyDetailsEnabled>true</dependencyDetailsEnabled>
                                <dependencyLocationsEnabled>false</dependencyLocationsEnabled>
                            </configuration>
                        </plugin>
                        <!-- 项目API doc报告 -->
                        <plugin>
                            <groupId>org.apache.maven.plugins</groupId>
                            <artifactId>maven-javadoc-plugin</artifactId>
                            <version>2.7</version>
                        </plugin>
                        <!-- 项目源代码报告 -->
                        <plugin>
                            <groupId>org.codehaus.mojo</groupId>
                            <artifactId>jxr-maven-plugin</artifactId>
                        </plugin>
                        <!-- 项目还需要做的TODO报告 -->
                        <plugin>
                            <groupId>org.codehaus.mojo</groupId>
                            <artifactId>taglist-maven-plugin</artifactId>
                        </plugin>
                        <!-- 项目源代码分析报告 -->
                        <plugin>
                            <groupId>org.apache.maven.plugins</groupId>
                            <artifactId>maven-pmd-plugin</artifactId>
                            <version>2.5</version>
                            <configuration>
                                <linkXref>true</linkXref>
                                <sourceEncoding>GBK</sourceEncoding>
                                <minimumTokens>100</minimumTokens>
                                <targetJdk>1.5</targetJdk>
                            </configuration>
                        </plugin>

                        <plugin>
                            <groupId>org.apache.maven.plugins</groupId>
                            <artifactId>maven-dependency-plugin</artifactId>
                            <version>3.1.1</version>
                            <executions>
                                <execution>
                                    <id>copy</id>
                                    <phase>package</phase>
                                    <goals>
                                        <goal>copy</goal>
                                    </goals>
                                    <configuration>
                                        <artifactItems>
                                            <artifactItem>
                                                <groupId>[ groupId ]</groupId>
                                                <artifactId>[ artifactId ]</artifactId>
                                                <version>[ version ]</version>
                                                <type>[ packaging ]</type>
                                                <classifier>[classifier - optional]</classifier>
                                                <overWrite>[ true or false ]</overWrite>
                                                <outputDirectory>[ output directory ]</outputDirectory>
                                                <destFileName>[ filename ]</destFileName>
                                            </artifactItem>
                                        </artifactItems>
                                        <!-- other configurations here -->
                                    </configuration>
                                </execution>
                            </executions>
                        </plugin>
                        <!-- 生成站点文件具体信息报告 -->
                        <plugin>
                            <groupId>org.apache.maven.plugins</groupId>
                            <artifactId>maven-linkcheck-plugin</artifactId>
                            <version>1.1</version>
                            <configuration>
                            </configuration>
                        </plugin>
                        <!-- 单元测试覆盖率报告 -->
                        <plugin>
                            <groupId>org.codehaus.mojo</groupId>
                            <artifactId>cobertura-maven-plugin</artifactId>
                        </plugin>
                    </reportPlugins>
                </configuration>
            </plugin>
        </plugins>
    </reporting>
</project>

````

运行maven site命令后，可以生成站点文件target/site/

![site站点文件](/images/maven/site站点文件.png)

要在浏览器中预览结果，你可以运行mvn site:run，Maven会构建站点并启动一个内嵌的Jetty容器。一旦Jetty启动并开始监听8080端口（默认情况下），你就可以通过在浏览器中输入http://localhost:8080/查看项目站点了。

![丰富的项目信息](/images/maven/丰富的项目信息.png)

可以通过丰富的项目报告插件配置，生成内容丰富的报告。报告插件不同于普通的插件（<project><build><plugins>）它是配置在<project><reporting><plugins>节点下

# pom元素 settings元素以及常用插件

![pom元素1](/images/maven/pom元素1.png)
![pom元素2](/images/maven/pom元素2.png)
![setting元素](/images/maven/setting元素.png)
![常用插件1](/images/maven/常用插件1.png)
![常用插件2](/images/maven/常用插件2.png)


# Archetype的使用

archetype 可以快速生产项目骨架，可以理解archetype是maven项目的模板，例如maven-archetype-quickstart就是最简单的maven项目模板。

archetype是通过maven-archetype-plugin插件来实现的。

## 使用archetype的一般步骤

1.执行命令
````
maven3 :mvn archetype:generate

maven2 :mvn org.apache.maven.plugins:maven-archetype-plugin:2.0-alpha-5:generate
````
2.输入基本的参数

groupId artifactId version package 


## 常用archetype介绍

1. maven-archetype-quickstart 最简单的maven项目

2. maven-archetype-webapp 最简单的web项目









