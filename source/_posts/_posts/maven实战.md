---
title: maven实战
copyright: true
date: 2018-06-09 03:03:03
tags: maven实战
categories: maven
---

# 简介

构建: 清理 编译 测试生成报告 打包 部署 就是自动化构建的过程;

maven不仅是构建工具,还是一个依赖管理工具和项目信息管理工具(包括获取项目文档 测试报告 静态分析报告 源码版本日志报告等项目的信息).

## 安装maven

安装maven需要首先安装jdk环境,配置java环境变量.然后安装maven,这里不在赘述.

## /.m2

mvn help:system 命令执行后会打印出所有的java系统属性和环境变量.

包括System Properties和Environment Variables两大部分

## 设置http代理

确认是否可以访问公共的maven仓库, ping repol.maven.org.如果不通,则需要telnet需要设置的代理ip和port,连接正确,则输入ctrl+] 绕后q回车即可.

检查完毕之后,编辑~/.m2/settings.xml文件.添加代理配置 在maven的conf目录下是有proxies节点配置代理的,copy到.m2下的settings文件中.

## 设置MAVEN_OPTS环境变量

执行mvn命令实际上执行的时java命令,那么运行的时候需要执行虚拟机参数.默认的MAVEN_OPTS="-Xms128m -Xmx512m"是不能满足需求的,可以自行修改,和设置maven_home时一样的设置方式.

export MAVEN_OPTS="-Xms256m -Xmx512m"

## 不要使用IDE内嵌的maven

原因:

1.内嵌的maven通常会比较新,通常不稳定;

2.命令行的和内嵌的版本不一致,我们有时候需要使用命令行运行maven,这样会产生一些奇怪的问题.


# 使用入门

maven项目的核心是pom.xml文件(project object model,项目对象模型).

## 编写pom文件

````
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.juvenxu.mvnbook</groupId>
    <artifactId>hello-world</artifactId>
    <version>1.0-SNAPSHOT</version>
    <name>Maven Hello World Project</name>

</project>         

````

pom文件的demo.SNAPSHOT意思是快照,说明该项目还在开发中,是不稳定的版本.

## 编写主代码

1.主代码和测试代码不同,项目的主代码会被打包到最终的构建如jar包中,而测试代码只在运行测试时用到.

2.默认情况下,maven假设主代码位于src/main/java目录.

````
package com.juvenxu.mvnbook.helloworld;

public class HelloWorld{

public String sayHello(){

 return "hello maven";
 }

public static void main(String [] args){

System.out.println(new HelloWorld().sayHello());
}

}

````

java类中的包要基于项目的groupId和artifactId,这样逻辑清晰;

3.在项目的根目录下编译 mvn clean compile

````
zhuningning@ubuntu:~/IdeaProjects/hello-world$ mvn clean compile
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------< com.juvenxu.mvnbook:hello-world >-------------------
[INFO] Building Maven Hello World Project 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ hello-world ---
[INFO] Deleting /home/zhuningning/IdeaProjects/hello-world/target
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ hello-world ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/zhuningning/IdeaProjects/hello-world/src/main/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ hello-world ---
[INFO] Changes detected - recompiling the module!
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 1 source file to /home/zhuningning/IdeaProjects/hello-world/target/classes
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 1.869 s
[INFO] Finished at: 2018-06-09T15:13:33+08:00
[INFO] ------------------------------------------------------------------------
zhuningning@ubuntu:~/IdeaProjects/hello-world$ 

````
clean告诉maven清理输出目录target/,compile告诉maven编译项目主代码.

从输出中看到maven首先执行了clean任务,删除target/目录,默认构建的所有输出都会在target目录下,紧接着执行resources任务,最后执行compile命令,将项目主代码编译到target/classes目录下.

## 编写测试代码

1.测试代码的默认目录是src/test/java/,而测试使用的是junit单元测试标准.

<scope>test</scope> 的意思是,在import junit代码中引入是没有问题的,但是在主代码中引入就会编译错误.

````
package com.juvenxu.mvnbook.helloworld

import static org.junit.Assert.assertEquals;
import org.junit.Test;

public class HelloWorldTest{

@Test
public void testSayHello(){

  HelloWorld helloWorld =new HelloWorld();
  String result = helloWorld.sayHello();
  assertEquals("hello maven",result);

}

}

````
2.典型的单元测试分为三个步骤:准备测试类和数据;执行要测试的行为;检查结果;


3.执行mvn clean test命令.实际执行的命令包括:clean:clean / resources:resources /compiler:compile / resources:testResources /compiler:testCompile

 过程中出现surefire:test,它是maven中负责执行测试的插件,并输出测试报告.

````
zhuningning@ubuntu:~/IdeaProjects/hello-world$ mvn clean test
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------< com.juvenxu.mvnbook:hello-world >-------------------
[INFO] Building Maven Hello World Project 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ hello-world ---
[INFO] Deleting /home/zhuningning/IdeaProjects/hello-world/target
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ hello-world ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/zhuningning/IdeaProjects/hello-world/src/main/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ hello-world ---
[INFO] Changes detected - recompiling the module!
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 1 source file to /home/zhuningning/IdeaProjects/hello-world/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ hello-world ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/zhuningning/IdeaProjects/hello-world/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ hello-world ---
[INFO] Changes detected - recompiling the module!
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 1 source file to /home/zhuningning/IdeaProjects/hello-world/target/test-classes
[INFO] 
[INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ hello-world ---
[INFO] Surefire report directory: /home/zhuningning/IdeaProjects/hello-world/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running com.juvenxu.mvnbook.helloworld.HelloWorldTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.055 sec

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 2.951 s
[INFO] Finished at: 2018-06-09T15:37:21+08:00
[INFO] ------------------------------------------------------------------------
zhuningning@ubuntu:~/IdeaProjects/hello-world$ 


````

## 打包和运行

1.打包是下一个进行的步骤,由于pom中没有指定打包的类型,所以默认的打包类型时jar包.

````
zhuningning@ubuntu:~/IdeaProjects/hello-world$ mvn clean package 
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------< com.juvenxu.mvnbook:hello-world >-------------------
[INFO] Building Maven Hello World Project 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ hello-world ---
[INFO] Deleting /home/zhuningning/IdeaProjects/hello-world/target
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ hello-world ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/zhuningning/IdeaProjects/hello-world/src/main/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ hello-world ---
[INFO] Changes detected - recompiling the module!
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 1 source file to /home/zhuningning/IdeaProjects/hello-world/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ hello-world ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/zhuningning/IdeaProjects/hello-world/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ hello-world ---
[INFO] Changes detected - recompiling the module!
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 1 source file to /home/zhuningning/IdeaProjects/hello-world/target/test-classes
[INFO] 
[INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ hello-world ---
[INFO] Surefire report directory: /home/zhuningning/IdeaProjects/hello-world/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running com.juvenxu.mvnbook.helloworld.HelloWorldTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.05 sec

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] 
[INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ hello-world ---
[INFO] Building jar: /home/zhuningning/IdeaProjects/hello-world/target/hello-world-1.0-SNAPSHOT.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.030 s
[INFO] Finished at: 2018-06-09T17:46:57+08:00
[INFO] ------------------------------------------------------------------------


````

2.jar:jar 操作就是负责打包. 如何让其它的项目直接引用到这个jar包呢?

运行mvn clean install ,安装任务 install:install.将项目输出的jar安装到了maven本地仓库中.

````
zhuningning@ubuntu:~/IdeaProjects/hello-world$ mvn clean install
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------< com.juvenxu.mvnbook:hello-world >-------------------
[INFO] Building Maven Hello World Project 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ hello-world ---
[INFO] Deleting /home/zhuningning/IdeaProjects/hello-world/target
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ hello-world ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/zhuningning/IdeaProjects/hello-world/src/main/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ hello-world ---
[INFO] Changes detected - recompiling the module!
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 1 source file to /home/zhuningning/IdeaProjects/hello-world/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ hello-world ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/zhuningning/IdeaProjects/hello-world/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ hello-world ---
[INFO] Changes detected - recompiling the module!
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 1 source file to /home/zhuningning/IdeaProjects/hello-world/target/test-classes
[INFO] 
[INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ hello-world ---
[INFO] Surefire report directory: /home/zhuningning/IdeaProjects/hello-world/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running com.juvenxu.mvnbook.helloworld.HelloWorldTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.063 sec

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] 
[INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ hello-world ---
[INFO] Building jar: /home/zhuningning/IdeaProjects/hello-world/target/hello-world-1.0-SNAPSHOT.jar
[INFO] 
[INFO] --- maven-install-plugin:2.4:install (default-install) @ hello-world ---
[INFO] Installing /home/zhuningning/IdeaProjects/hello-world/target/hello-world-1.0-SNAPSHOT.jar to /home/zhuningning/.m2/repository/com/juvenxu/mvnbook/hello-world/1.0-SNAPSHOT/hello-world-1.0-SNAPSHOT.jar
[INFO] Installing /home/zhuningning/IdeaProjects/hello-world/pom.xml to /home/zhuningning/.m2/repository/com/juvenxu/mvnbook/hello-world/1.0-SNAPSHOT/hello-world-1.0-SNAPSHOT.pom
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 2.913 s
[INFO] Finished at: 2018-06-09T17:52:27+08:00
[INFO] ------------------------------------------------------------------------


````
install命令执行完之后,本地仓库/.m2 中有项目的pom和jar

3. main方法的运行

到目前为止还是没有运行Hello World项目的,但是helloWorld类是有main方法的.默认打包生成的jar包是不能够直接运行的.那是因为带有main方法的类信息是不会添加到mainfest中(打开jar文件中的META-INF/MANIFEST.MF文件,无法查看到Main-Class-mainifest),为了生成,需要借助maven-shade-plugin

配置插件如下:

````
<build>
<plugins>
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-shade-plugin</artifactId>
  <version>1.2.1</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <transformers>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                  <mainClass>com.juvenxu.mvnbook.helloworld.HelloWorld</mainClass>
                </transformer>
              </transformers>
            </configuration>
          </execution>
        </executions>
      </plugin>
</plugins>
</build>

````

然后再次运行mvn clean install

在target目录下有jar包,然后执行 java -jar

````
zhuningning@ubuntu:~/IdeaProjects/hello-world/target$ java -jar hello-world-1.0-SNAPSHOT.jar 
hello maven
zhuningning@ubuntu:~/IdeaProjects/hello-world/target$ 

````

## 使用Archetype生成项目骨架

项目的默认约定：在项目的根目录中防止pom.xml 在/src/main/java中放置项目的主代码，在/src/test/java中放置项目的测试代码。

执行以下命令后需要输入groupId、artifactId、version、packaging。创建默认结构的maven项目。

````
zhuningning@ubuntu:~/IdeaProjects/test$ mvn archetype:generate
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------< org.apache.maven:standalone-pom >-------------------
[INFO] Building Maven Stub Project (No POM) 1
[INFO] --------------------------------[ pom ]---------------------------------
[INFO] 
[INFO] >>> maven-archetype-plugin:3.0.1:generate (default-cli) > generate-sources @ standalone-pom >>>
[INFO] 
[INFO] <<< maven-archetype-plugin:3.0.1:generate (default-cli) < generate-sources @ standalone-pom <<<
[INFO] 
[INFO] 
[INFO] --- maven-archetype-plugin:3.0.1:generate (default-cli) @ standalone-pom ---
[INFO] Generating project in Interactive mode




````





## maven坐标

如下是引入spring的context包。

````
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>4.3.5.RELEASE</version>
</dependency>

````

groupId：定义当前maven项目隶属的实际项目，groupId不应该对应项目隶属的公司或者组织，因为一个公司有可能有多个项目。

artifactId： 该元素定义实际项目中的一个maven项目（模块），推荐的是使用实际项目名称作为artifactId的前缀。

version:项目的版本，

packaging：该元素定义的maven项目的打包方式。默认为jar



# 依赖的配置


````  
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>4.3.5.RELEASE</version>
    <type>jar</type>
    <scope>runtime</scope>
    <optional>false</optional>
    <exclusions>
        <exclusion>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>

````
## 依赖范围。

依赖范围就是用来控制依赖与三种classpath（编译classpath、测试classpath、运行classpath）的关系

scope范围包括：

1. compile 编译依赖范围，默认的。对于编译、测试、运行三种classpath都有用。

2. test 测试有效。在编译和运行主代码时无效。如 junit

3. provided 编译和测试的时候有效，运行的时候无效。如 servlet-api

4. runtime 对于测试和运行有效，对编译主代码无效。

5. system 系统依赖范围，和compile一致，但是必须通过systemPath元素显式的指定依赖文件的路径。

6. import dependencyManagement中才可以使用import，表示从其它的pom文件中导入依赖设置。

## 传递性依赖

如A项目依赖 spring-core包，而spring-core包依赖一个common-logging包，common-logging包就是A项目的传递依赖。

假设A依赖于B，B依赖于C，我们说A对于B是第一直接依赖，B对于C是第二直接依赖，A对于C是传递性依赖。

## 依赖调解

maven引入的传递性依赖机制。一方面大大简化和方便了依赖声明；另一方面，大部分情况我们只需要关心项目的直接依赖，而不用考虑直接依赖会引入什么传递性依赖。

例子： 项目A有这样的依赖关系：A->B->C->X(1.0),A->D->X(2.0),X是传递性依赖，但是有2个版本，要选择一个去依赖。**调解的第一原则：路径最近者优先，会选择2.0；当路径相同的情况下，第二原则：第一声明者优先。**

## 可选依赖

项目B实现了2个特性，其中一个特性依赖于X，另一个依赖于Y，而且这两个特性是互斥的。用户不可能同时使用两个特性。比如支持多种数据库，在构建的时候需要两种数据库的驱动程序，但在使用这个工具包的时候指挥依赖一种数据库。

<optional> true </optional>

## 项目依赖实践技巧

1.排除依赖

exclusions 排除依赖。声明exclusion的时候，只需要groupId和artifactId。不需要version

2.依赖归类

<properties> 声明一个属性子元素，多个版本相同的项目的依赖使用同一个版本。

3.优化依赖


①查看当前项目的已解析依赖  mvn:dependency:list

②查看当前项目的依赖树  mvn:dependency:tree

③查看使用未声明的依赖和声明未使用的依赖： mvn dependency:analyze 。used undeclared dependencies 和Unused declared undeclared.由于该命令只会分析编译主代码和测试主代码需要的依赖，一些执行测试和运行时需要的依赖发现不了。所以不要随便根据此提示来删除一些声明未使用的依赖。

# 仓库

## 仓库分类

maven仓库是为共享依赖的jar包而创建的，分为本地仓库和远程仓库。

当maven寻找组件的时候，首先从本地仓库查找，如果本地没有或者需要更新的时候从远程仓库下载到本地仓库。

两个特殊的远程仓库 1.中央仓库：maven核心自带的仓库； 2.私服：特殊的局域网假设的仓库；


## 本地仓库

.m2/repository/的仓库目录。可以在maven的settings配置中设置本地仓库的位置。

一个构件只有在本地仓库中之后，才能由其它maven项目使用，那么构件如何进入本地仓库呢？最常见的是从远程下载到本地，还有一种是将本地的项目的构件安装到maven仓库中。

mvn clean install命令就是，执行install插件的install的目标项目的构建输出文件安装到本地仓库。

## 远程仓库


## 中央仓库

默认的maven仓库是中央仓库，在maven的lib/maven-model-builder-3.0.jar，然后访问路径org/apache/maven/model/pom-4.0.0.xml可以看到

````

<repositories>     
  <repository>     
    <id> central</id>     
    <name> Maven Repository Switchboard</name>     
    <layout> default</layout>     
    <url> http://repo1.maven.org/maven2</url>     
    <snapshots>     
      <enabled> false</enabled>     
    </snapshots>     
  </repository>     
</repositories>    

````

这是所有maven项目都会继承的超级pom。

## 私服

私服是一种特殊的远程仓库，它时架设在局域网内的仓库服务。当maven需要下载构件的时候，先从私服请求，如果私服上不存在该构件，则从外部的远程仓库下载，缓存在私服上之后，再为maven的下载请求提供服务。

优点：

1. 节省自己的外网带宽；
2. 加速maven构建；
3. 部署第三方构件；
4. 提高稳定性，增强控制；
5. 降低中央仓库的负荷；

## 远程仓库的配置

### 在pom中配置远程仓库

在pom中添加repositories/repository 两个节点，可以配置多个远程仓库。其中id是唯一的，中央仓库的id时central。

````
 <repositories>  
    <repository>  
      <id>jboss</id>  
      <name>JBoss Repository</name>  
      <url>http://repository.jboss.com/maven2/</url>  
      <releases>  
        <enabled>true</enabled>  
      </releases>  
    </repository>  
    <snapshots>  
      <enabled>false</enabled>  
    </snapshots>  
    <layout>default</layout>  
  </repositories>  

````
其中关键的两个节点releases 表示发布版本的下载为true，snapshots表示快照版本的下载为false。

snapshots比较重要的节点

````
<snapshots>  
  <enabled>true</enabled>  
  <updatePolicy>daily</updatePolicy>  
  <checksumPolicy>ignore</checksumPolicy>  
</snapshots>  

````

updatePolicy daily，表示Maven每天检查一次。其他可用的值包括：never---从不检查更新；always---每次构建都检查更新；interval:X---每隔X分钟检查一次更新(X为任意整数)

checksumPolicy节点表示检查更新的频率和下载失败时的处理策略。

值为默认的warn时，Maven会在执行构建时输出警告信息，其他可用的值包括：fail---Maven遇到校验和错误就让构建失败；ignore---使用Maven完全忽略校验和错误。


### 在setting.xml中配置远程仓库

````
1.需要在profiles标签中添加远程仓库配置

<profile>  
        <id>myProfiel</id>    
    <repositories>    
        <repository>    
            <id>me</id>    
            <name>me Repository</name>    
            <url>http://192.168.106.58:57770/nexus/</url>    
            <releases>    
                <updatePolicy>daily</updatePolicy>never,always,interval n    
                <enabled>true</enabled>    
                <checksumPolicy>warn</checksumPolicy>fail,ignore    
            </releases>    
            <snapshots>    
                <enabled>false</enabled>    
            </snapshots>    
            <layout>default</layout>    
        </repository>    
    </repositories>    
</profile>  

2.在settings标签中添加activeProfiles标签，用于激活配置的profile标签

<activeProfiles>       
    <activeProfile>myProfiel</activeProfile>       
</activeProfiles> 
 
````



### 远程仓库的认证

大部分仓库时无需认证的，但是有的时需要的。这个是必须在settings.xml中配置。

````
<settings>  
  ...  
  <servers>  
    <server>  
      <id>my-proj</id>  
      <username>repo-user</username>  
      <password>repo-pwd</password>  
    </server>  
  </servers>  
  ...  
</settings>  

````

### 部署至远程仓库

私服的一大作用是部署第三方构件，包括组织内部生成的构件以及一些无法从外部仓库直接获取的构件。无论是日常开发中生成的构件，还是正式版本发布的构件，都需要部署到仓库中，供其他团队成员使用。

Maven除了能对项目进行编译、测试、打包之外，还能将项目生成的构建部署到仓库中。首先，需要编辑项目的pom.xml文件。配置distributionManagement元素

````
<project>  
  ...  
  <destributionManagement>  
    <repository>  
      <id>proj-releases</id>  
      <name>Proj Release Repository</name>  
      <url>http://192.168.1.100/content/repositories/proj-releases</url>  
    </repository>  
    <snapshotRepository>  
      <id>proj-snapshots</id>  
      <name>Proj Snapshot Repository</name>  
      <url>http://192.168.1.100/content/repositories/proj-snapshots</url>  
    </snapshotRepository>  
  </destributionManagement>  
  ...  
</project> 


````
distributionManagement包含repository和snapshotRepository子元素，前者表示发布版本构件的仓库，后者表示快照版本的仓库。这两个元素下都需要配置id、name和url，id为该远程仓库的唯一标识，name是为了方便人阅读，关键的url表示该仓库的地址。

一般都需要认证。配置正确后，在命令行运行mvn clean deploy，Maven就会将项目构建输出的构件部署到配置对应的远程仓库。

## 快照版本

快照版本时为了解决A依赖B模块，B模块一直在不停的更新的情况。

在开发过程中只需将B模块改为2.1-SNAPSHOT，然后发布到私服中，在发布的过程中，maven会自动为构件打时间戳，比如2.1-2009-1214.221414-13表示这个时间点的第13次构建。有了时间戳maven就能随时找到仓库中该构件的最新的文件，这个更新策略是在之前的私服配置中已经介绍，或者使用mvn clean install-U。如果已经稳定，则将2.1-SNAPSHOT改为2.1 .

组织内部的模块或者项目的依赖,由于具有完全的理解和控制权可以使用快照版本，外部依赖如果使用快照版本则会存在潜在的危险。

## 仓库解析依赖的机制

当本地仓库没有依赖构件的时候，maven会自动从远程仓库下载；当依赖版本为快照版本的时候maven会自动找到最新的快照。这背后的依赖解析机制可以概括如下:

1.当依赖的范围是system的时候maven直接从本地文件系统解析构件。

2.根据依赖坐标计算仓库路径后，尝试直接从本地仓库寻找构件，如果发现相应构件则解析成功。

3.在本地仓库不存在相应构件的情况下，如果依赖的版本显示的发布版本构件，如1.2,2.1-beta-1等，则遍历所有的远程仓库，发现后下载并解析使用。

4.如果依赖的版本是RELEASE或者LATEST，则基于更新策略读取所有远程仓库的元数据groupId/artifactId/maven-metadata.xml,将其与本地仓库对应元数据合并后计算出RELEASE或者LATEST真实的值，然后基于这个真实的值检查本地和远程仓库如步骤2和3。

5.如果依赖的版本是SNAPSHOT，则基于更新策略读取所有远程仓库的元数据groupId/artifactId/version/maven-metadata.xml，将其与本地仓库对应元数据合并后得到最新快照版本的值，然后基于该值检查本地仓库或者从远程仓库下载。

6.如果最后解析得到的构件版本时间是时间戳格式的快照，如1.4.1-20091104.121450-121，则复制其时间戳格式的文件至非时间戳格式，如SNAPSHOT，并使用该非时间戳格式的构件。

## 镜像

如果仓库X可以提供仓库Y存储的所有内容，那么就可以认为X是Y的一个镜像

settings.xml 中配置镜像，以下是中央仓库在中国的一个镜像

````
<settings>  
  ...  
  <mirrors>  
    <mirror>  
      <id>maven.net.cn</id>  
      <name>one of the central mirrors in china</name>  
      <url>http://maven.net.cn/content/groups/public/</url>  
      <mirrorOf>central</mirrorOf>  
    </mirror>  
  </mirrors>  
  ...  
</settings>

````

其中一般私服和镜像一起使用。

## 仓库搜索服务

一般使用仓库搜索服务来根据关键字得到maven坐标。


主流的仓库搜索服务

````
1. sonatype
https://repository.sonatype.org

2. MVN Browser
http://www.mvnbrowser.com

3. MVNrepository
http://mvnrepository.com/

````

# 生命周期和插件


maven的生命周期是抽象的，其实际行为都是由插件来完成，如package阶段的任务就会由maven-jar-plugin完成。生命周期和插件两者协同工作，密不可分。

## 生命周期

maven的生命周期就是为了对所有的构建进行抽象和统一。总结了一套包含项目**清理、初始化、编译、测试、打包、集成测试、验证、部署和站点生成等**几乎所有的构建步骤。

maven的生命周期时抽象的，生命周期本身不做工作，主要由插件来完成。类似于设计模式中的模板方法。

maven在生命周期的各个阶段绑定了一个或者多个插件，而且maven为生命周期绑定了默认的插件。

### 优点

maven定义生命周期和插件机制一方面保证了所有maven项目有一致的构建标准，另一方面又通过默认插件简化和稳定了实际项目的构建。

## 生命周期详解

maven有三套相互独立的生命周期，他们分别为clean default site。clean生命周期主要是清理项目，default生命周期主要是构建项目，site生命周期主要时建立项目站点。

### clean生命周期：清理项目，包含三个phase。

1）pre-clean：执行清理前需要完成的工作

2）clean：清理上一次构建生成的文件

3）post-clean：执行清理后需要完成的工作

### default生命周期：构建项目，重要的phase如下。

1）validate：验证工程是否正确，所有需要的资源是否可用。

2）initialize 初始化操作

2）compile：编译项目的源代码。  

3）test：使用合适的单元测试框架来测试已编译的源代码。这些测试不需要已打包和布署。

4）Package：把已编译的代码打包成可发布的格式，比如jar。

5）integration-test：如有需要，将包处理和发布到一个能够进行集成测试的环境。

6）verify：运行所有检查，验证包是否有效且达到质量标准。

7）install：把包安装到maven本地仓库，可以被其他工程作为依赖来使用。

8）Deploy：在集成或者发布环境下执行，将最终版本的包拷贝到远程的repository，使得其他的开发者或者工程可以共享。

### site生命周期：建立和发布项目站点，phase如下

1）pre-site：生成项目站点之前需要完成的工作

2）site：生成项目站点文档

3）post-site：生成项目站点之后需要完成的工作

4）site-deploy：将项目站点发布到服务器

### 命令行与生命周期

1.mvn clean 执行clean生命周期的clean阶段，包括pre-clean、clean

2.mvn test 执行default生命周期之前的所有阶段，包括validate initialize compile test

3.mvn clean install 执行包括clean和default生命周期从validate到install的所有阶段

4.mvn clean deploy site-deploy ,执行包括clean 和default所有阶段以及site周期的所有阶段

## 插件目标

一个maven插件maven-dependency-plugin可以做很多事情，包括分析项目依赖，列出依赖树，列出所有已经解析的依赖等等。这些功能聚集在一个插件里，每个功能就是一个目标。

maven-dependency-plugin有10多个插件目标如：dependency：analyze；dependency：tree；dependency：list等。冒号前是插件前缀，冒号后面是插件的目标。

## 插件绑定

生命周期的阶段与插件的目标项目绑定，以完成某个具体的构建任务。如项目编译这一任务，它对应了default生命周期的compile这一阶段，而maven-compile-plugin这一插件的compile目标能够完成。

### 内置绑定

![default生命周期的内置插件绑定关系和具体任务](/images/maven/default生命周期的内置插件绑定关系和具体任务.png)

### 自定义绑定

除了内置绑定以外，用户还能够自己选择将某个插件目标绑定到生命周期的某个阶段。自定义绑定允许我们自己掌控插件目标与生命周期的结合


````
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>2.1.1</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <phase>verify</phase>
                        <goals>
                            <goal>jar-no-fork</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

````

上述配置有插件的坐标声明、还有excutions下面每个excution子元素配置的执行的一个个任务、通过phase指定与生命周期的那个阶段绑定、在通过goals指定执行绑定插件的哪些目标。

输出对应插件的详细信息：

### 使用maven-help-plugin获取插件描述

以下为显示目标插件的描述命令：

````
mvn help:describe -Dplugin=org.apache.maven.plugins:maven-source-plugin -Dgoal=jar-no-fork -Ddetail=true

````

可以使用插件目标前缀来替换坐标:

````
mvn help:describe -Dplugin = compiler

````
如果仅仅是需要描述某个插件的目标信息，则需要加上goal目标 更详细的信息添加-Ddeta
````
mvn help:describe -Dplugin = compiler -Dgoal = compile

````


### 插件配置

#### 从命令行调用插件

````
root@ubuntu:/home/zhuningning/IdeaProjects/hello-world# mvn -h
usage: mvn [options] [<goal(s)>] [<phase(s)>]

````
我们可以从命令行激活生命周期阶段如：

````
mvn clean
````

还可以支持从命令行调用插件目标如：

````
mvn dependency:tree(插件前缀：插件目标)

````



### Maven变量及常见插件配置详解

1.自定义变量
````
    <properties>  
        <project.build.name>tools</project.build.name>  
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>  
    </properties>  
````
2.内置变量

````
${basedir} 项目根目录
${project.build.directory} 构建目录，缺省为target
${project.build.outputDirectory} 构建过程输出目录，缺省为target/classes
${project.build.finalName} 产出物名称，缺省为${project.artifactId}-${project.version}
${project.packaging} 打包类型，缺省为jar
${project.xxx} 当前pom文件的任意节点的内容

````

3.编译插件

````
<plugin>  
   <groupId>org.apache.maven.plugins</groupId>  
   <artifactId>maven-compiler-plugin</artifactId>  
   <configuration>  
       <source>1.6</source>  
       <target>1.6</target>  
       <encoding>${project.build.sourceEncoding}</encoding>  
   </configuration>  
</plugin>

   source： 源代码编译版本；  
   target： 目标平台编译版本；  
   encoding： 字符集编码

````

4.设置资源文件的编码方式

````

<plugin>  
  <groupId>org.apache.maven.plugins</groupId>  
  <artifactId>maven-resources-plugin</artifactId>  
  <version>2.4.3</version>  
  <executions>  
      <execution>  
          <phase>compile</phase>  
      </execution>  
  </executions>  
  <configuration>  
      <encoding>${project.build.sourceEncoding}</encoding>  
  </configuration>  
</plugin> 

````

5.自动拷贝jar包到target目录 [参考](http://liugang594.iteye.com/blog/2093082)

```` 

<plugin>  
  <groupId>org.apache.maven.plugins</groupId>  
  <artifactId>maven-dependency-plugin</artifactId>  
  <version>2.6</version>  
  <executions>  
      <execution>  
          <id>copy-dependencies</id>  
          <phase>compile</phase>  
          <goals>  
              <goal>copy-dependencies</goal>  
          </goals>  
          <configuration>  
              <!-- ${project.build.directory}为Maven内置变量，缺省为target -->  
              <outputDirectory>${project.build.directory}/lib</outputDirectory>  
              <!-- 表示是否不包含间接依赖的包 -->  
              <excludeTransitive>false</excludeTransitive>  
              <!-- 表示复制的jar文件去掉版本信息 -->  
              <stripVersion>true</stripVersion>  
          </configuration>  
      </execution>  
  </executions>  
</plugin>  


````

6.生成源代码jar包

````

<plugin>  
  <artifactId>maven-source-plugin</artifactId>  
  <version>2.1</version>  
  <configuration>  
      <!-- <finalName>${project.build.name}</finalName> -->  
      <attach>true</attach>  
      <encoding>${project.build.sourceEncoding}</encoding>  
  </configuration>  
  <executions>  
      <execution>  
          <phase>compile</phase>  
          <goals>  
              <goal>jar</goal>  
          </goals>  
      </execution>  
  </executions>  
</plugin>
````

7. 将项目打成jar包

````

<plugin>  
  <groupId>org.apache.maven.plugins</groupId>  
  <artifactId>maven-jar-plugin</artifactId>  
  <version>2.4</version>  
  <configuration>  
      <archive>  
          <manifest>  
              <!-- 告知 maven-jar-plugin添加一个 Class-Path元素到 MANIFEST.MF文件，以及在Class-Path元素中包括所有依赖项 -->  
              <addClasspath>true</addClasspath>  
              <!-- 所有的依赖项应该位于 lib文件夹 -->  
              <classpathPrefix>lib/</classpathPrefix>  
              <!-- 当用户使用 lib命令执行JAR文件时，使用该元素定义将要执行的类名 -->  
              <mainClass>com.zhengtian.tools.service.phone.MobilePhoneTool</mainClass>  
          </manifest>  
      </archive>  
  </configuration>  
</plugin>  

````

### maven的模块聚合(多模块)

为了将2个模块聚合到一个模块下，必须新建一个聚合模块，它有自己的pom文件，packaging为pom。modules下配置多个模块

聚合项目可以使用父子结构和平行结构，一般建议使用父子结构。

构建各个模块的顺序是采用反应堆构建顺序


````
<project
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
        http://maven.apache.org/maven-v4_0_0.xsd>
        <modelVersion>4.0.0</modelVersion>
        <groupId>com.juvenxu.mvnbook.account</groupId>
        <artifact>account-aggregator</artifact>
        <version>1.0.0-SNAPSHOT</version>
        <packaging>pom</packaging>
        <name>Account Aggregator</name>
        <modules>
            <module>account-email</module>
            <module>account-persist</module>
        </modules>
</project>


````


### maven的模块继承 

父模块的packaging为pom，这一点与聚合模块一样。父模块是为了消除配置的重复。

父模块：

````
<project
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:shemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.juvenxu.mvnbook.account</groupId>
    <artifactId>account-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    <name>Account Parent</name>
</project>

````

继承父模块的子模块

````
<project
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:shemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/maven-v4_0_0.xsd">
    <parent>
        <groupId>com.juvenxu.mvnbook.account<groupId>
        <artifactId>account-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../account-parent/pom.xml</relativePath>
    </parent
    
    <artifactId>account-email</artifactId>
    <name>Account Email</name>
    
    <dependencies>
        ....
    </dependencies>
    <build>
        <plugins>
            ....
        </plugins>
    </build>
</project>

````

父模块加入到聚合模块


````
<project
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
        http://maven.apache.org/maven-v4_0_0.xsd>
        <modelVersion>4.0.0</modelVersion>
        <groupId>com.juvenxu.mvnbook.account</groupId>
        <artifact>account-aggregator</artifact>
        <version>1.0.0-SNAPSHOT</version>
        <packaging>pom</packaging>
        <name>Account Aggregator</name>
        <modules>
            <module>account-email</module>
            <module>account-persist</module>
            <module>account-parent</module>
        </modules>
</project>

````

可以继承的pom元素


````
groupId:项目组ID,项目坐标的核心元素
version:项目版本,项目坐标的核心元素
description:项目的描述信息
organnization:项目的组织信息
inceptionYear:项目的创始年份
url:项目的URL地址
developers:项目的开发者信息
contributors:项目的贡献者信息
distributionManagement:项目的部署配置
issueManagement:项目的缺陷跟踪系统信息
ciManagement:项目的集成信息
scm:项目的版本控制系统信息
mailingLists:项目的邮件列表信息
properties:自定义的Maven属性
dependencies:项目的依赖配置
dependencyManagement:项目的依赖管理配置
repositories:项目的仓库配置
build:包括项目的源码目录配置、输出目录配置、插件配置、插件管理配置等
reporting:包括项目的报告输出目录配置，报告插件配置等。


````
### 模块的依赖管理

Maven提供的dependentcyManagement元素既能让子模块继承到父模块的依赖配置，又能保证子模块依赖使用的灵活度。在dependentcyManagement元素下的依赖声明不会引入实际的依赖，不过他能够约束denpendencies下的依赖使用

整理后的父模块的pom

````
<project
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:shemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.juvenxu.mvnbook.account</groupId>
    <artifactId>account-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    <name>Account Parent</name
    <properties>
        <springframework.version>2.5.6</springframework.version>
        <junit.version>4.7</junit.version>
    </properties>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-core</artifactId>
                <version>${springframework.version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-beans</artifactId>
                <version>${springframework.version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-context</artifactId>
                <version>${springframework.version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-context-support</artifactId>
                <version>${springframework.version}</version>
            </dependency>
            <dependency>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
                <version>${springframework.version}</version>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>

````

整理后的继承父模块的子模块

````
<properties>
    <javax.mail.version>1.4.1</javax.mail.version>
    <greenmail.version>1.3.1b</greenmail.version>
</properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context-support</artifactId>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
        </dependency>
        <dependency>
            <groupId>javax.mail</groupId>
            <artifactId>mail</artifactId>
            <version>${javax.mail.version}</version>
        </dependency>
        <dependency>
            <groupId>javax.icegreen</groupId>
            <artifactId>greenmail</artifactId>
            <version>${greenmail.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

````

子模块只需要配置简单的groupId和artifactId就能获得对应的依赖信息，从而引入正确的依赖。

如果在子模块不声明依赖的使用，即使该依赖已经在父POM的dependencyManagement中声明了，也不会产生任何实际的效果。

依赖范围有个import范围，import依赖范围只有在dependencyManagement下才有效果，使用该范围的依赖通常指向一个POM，作用是将目标中的dependencyManagement配置导入到当前POM的dependencyManagement中配置，除了复制配置或者继承这两种方式之外，还可以使用import范围依赖将这一配置导入。

### 插件管理

Maven提供了dependencyManagement元素帮忙管理依赖，类似地，Maven也提供了pluginManagement元素帮忙管理插件。该元素中配置的依赖不会造成实际的插件调用行为，当POM中配置了真正的plugin元素，并且其groupId和artifactId与pluginManagement中配置的插件匹配时，pluginManagement的配置才会影响实际的插件行为。

### 聚合和继承的关系

多模块中的聚合与继承其实是两个概念，其目的是完全不同的，前者主要是为了方便快速构建项目，后者主要是为了消除重复配置。

在现有的实际项目中，往往会发现一个POM即是聚合POM，又是父POM，这么做主要是为了方便。

### 约定优于配置

Maven默认的源码目录是：src/main/java但是用户也可以自己指定源码目录，如下：

````
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.juvenxu.mvnbook<groupId>
    <artifactId>my-project</artifactId>
    <version>1.0</version>
    <build>
        <sourceDirectory>src/java</sourceDirectory>
    </build>
</project>
````

maven会默认隐身的继承于超级pom，超级pom内容如下：

````
<repositories>
    <repository>
        <id>central</id>
        <name>Maven Repository Switchboard</name>
        <url>http://repo1.maven.org/maven2</url>
        <layout>default</layout>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
<pluginRepositories>
    <pluginRepository>
        <id>central</id>
        <name>Maven Plugin Repository</name>
        <url>http://repo1.maven.org/maven2</url>
        <layout>default</layout>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
        <releases>
            <updatePolicy>never</updatePolicy>
        </releases>
    </pluginRepository>
</pluginRepositories>

````
超级POM中关于项目结构的定义：

````
<build>
    <directory>${project.basedir}/target<directory>
    <outputDirectory>${project.build.directory}/classes</outputDirectory>
    <finalName>${project.artifactId}-${project.version}</finalName>
    <testOutputDirectory>${project.build.directory}/test-classes</testOutputDirectory>
    <sourceDirectory>${project.basedir}/src/main/java</sourceDirectory>
    <scriptSourceDirectory>src/main/script</scriptSourceDirectory>
    <testSourceDirectory>${project.basedir}/src/test/java</testSourceDirectory>
    <resources>
        <resource>
            <directory>${project.basedir}/src/main/resources</directory>
        </resource>
    </resources>
    <testResources>
        <testResource>
            <directory>${project.basedir}/src/test/resources</directory>
        </testResource>
    </testResources>


````
以下是超级pom关于插件版本的设定

````
<pluginManagement>
    <plugins>
        <plugin>
            <artifactId>maven-antrun-plugin</artifactId>
            <version>1.3</version>
        </plugin>

        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <version>2.2-bete-4</version>
        </plugin>
        
        <plugin>
            <artifactId>maven-clean-plugin</artifactId>
            <version>2.3</version>
        </plugin>
        
        <plugin>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>2.0.2</version>
        </plugin>
        ........
    </plugins>
</pluginManagement>

</build>

````
超级pom的位置

对于Maven3,超级POM在文件$MAVEN_HOME/lib/maven-model-builder-x.x.x.jar中的org/apache/maven/model/pom-4.0.0.xml路径下。

对于Maven2,超级POM在文件$MAVEN_HOME/lib/maven-x.x.x-uber.jar中的org/apache/maven/project/pom-4.0.0.xml目录下。

### 反应堆

对于一个多模块的项目，反应堆就是所有模块构成的一个构建结构。反应堆就包含了各模块之间继承与依赖的关系，从而能够自动计算出合理的模块构建顺序。
````
<modules>
    <module>account-email</module>
    <module>account-persist</module>
    <module>account-parent</module>
</modules>
````

实际项目时这样的，maven按照顺序读取pom，如果pom没有依赖模块，那么就构建该模块。否则就构建其它依赖模块。如果其它依赖模块还依赖别的其它模块，则先构建别的模块。

以上，构建顺序不一定是顺序去读取POM的顺序，当一个模块依赖于另外一个模块，Maven会先去构建被依赖模块，
并且Maven中不允许出现循环依赖的情况，就是，当出现模块A依赖于B,而B又依赖于A的情况时，Maven就会报错。

### 裁剪反应堆

一般来说，用户会选择构建整个项目或者选择构建单个的模块，但有些时候，用户会想要仅仅构建完整反应堆中的某些个模块。换句话说，用户需要实时地剪裁反应堆。

Maven提供很多命令行选项支持裁剪反应堆，输入mvn -h可以看到这些选项：

````
-am, --also-make：同时构建所列模块的依赖模块
-amd -also-make-dependents:同时构建依赖于所列模块的模块
-p,--project 构建指定的模块，模块之间使用逗号分割
-rf -resume-from 从指定的模块回复反应堆

````
例子：

默认情况从account-aggregator执行mvn clean install会得到如下完整的反应堆：
````
[INFO]-----------------------------------------------------
[INFO]Reactor Build Order:
[INFO]
[INFO]Account Aggregator
[INFO]Account Parent
[INFO]Account Email
[INFO]Account Persist
[INFO]
[INFO]-----------------------------------------------------
````

可以使用-pl选项指定构建某几个模块，如运行如下命令：

$ mvn clean install -p account-email,account-persist
得到的反应堆为:

````
[INFO]-----------------------------------------------------
[INFO]Reactor Build Order:
[INFO]
[INFO]Account Email
[INFO]Account Persist
[INFO]
[INFO]-----------------------------------------------------
````

使用-am选项可以同时构建所列模块的依赖模块，例如：

$ mvn clean install -pl account-parent -am
由于account-email和account-persist都依赖于account-parent，因此会得到如下反应堆：

````
[INFO]-----------------------------------------------------
[INFO]Reactor Build Order:
[INFO]
[INFO]Account Parent
[INFO]Account Email
[INFO]Account Persist
[INFO]
[INFO]-----------------------------------------------------
````

使用-rf选项可以在完整的反应堆构建顺序基础上指定从哪个模块开始构建。例如：

$mvn clean install -rf account-email
完整的反应堆构建顺序中，account-email位于第三，它之后只有account-persist因此会得到如下的剪裁反应堆：
````
[INFO]-----------------------------------------------------
[INFO]Reactor Build Order:
[INFO]
[INFO]Account Email
[INFO]Account Persist
[INFO]
[INFO]-----------------------------------------------------
````
最后在-pl -am或者-pl -amd的基础上，还能应用-rf参数，以对裁剪后的反应堆再次剪裁,例如：

$ mvn clean install -pl account-parent -amd -rf account-email
该命令中的-pl和-amd参数会裁剪出一个account-parent、account-email和account-persist的反应堆，在此基础上，-rf参数指定从account-email参数构建。因此会得到如下的反应堆：

````
[INFO]-----------------------------------------------------
[INFO]Reactor Build Order:
[INFO]
[INFO]Account Email
[INFO]Account Persist
[INFO]
[INFO]-----------------------------------------------------
````

在开发过程中，灵活应用上述四个参数，可以帮助我们跳过无须构建的模块，从而加速构建。在项目庞大、规模特别多的时候，这种效果就会异常明显。


## 私服

### 私服的优点

1.节省外网宽带
2.加速Maven构建
3.部署第三方构件
4.提高稳定性、增强控制：原因是外网不稳定
5.降低中央仓库的负荷：原因是中央仓库访问量太大

### 镜像库
mirror相当于一个拦截器，它会拦截maven对remote repository的相关请求，把请求里的remote repository地址，重定向到mirror里配置的地址。

### nexus私服的安装方式 

Nexus提供了两种安装方式，一种是内嵌Jetty的bundle，只要你有JRE就能直接运行。第二种方式是WAR，你只须简单的将其发布到web容器中即可使用。

我们采用第一种：

1.下载解压后的文件 [bundle文件下载](http://download.sonatype.com/nexus/oss/nexus-2.10.0-02-bundle.tar.gz)

````
zhuningning@ubuntu:/usr/local/nexus-2.10.0-02$ ls
bin  conf  lib  LICENSE.txt  logs  nexus  NOTICE.txt  tmp

````
2.在bin目录下启动文件

````
zhuningning@ubuntu:/usr/local/nexus-2.10.0-02/bin$ sudo ./nexus start
****************************************
WARNING - NOT RECOMMENDED TO RUN AS ROOT
****************************************
If you insist running as root, then set the environment variable RUN_AS_USER=root before running this script.


````

/nexus-2.12.0-01/bin/nexus 在该文件中增加一行配置：

````
RUN_AS_USER=root

````

3.修改nexus.properties配置文件(可选) 在config目录下

4.启动

````
zhuningning@ubuntu:/usr/local/nexus-2.10.0-02/bin$ sudo ./nexus start
****************************************
WARNING - NOT RECOMMENDED TO RUN AS ROOT
****************************************
Starting Nexus OSS...
Started Nexus OSS.

````

5.报错 nexus启动报错(Unable to start JVM: No such file or directory (2))

````
通过 service nexus start 启动报错，查看 /usr/local/nexus/nexus-2.10.0-02/logs 下的日志得出是找不到jdk路径

解决:
修改[NEXUS_HOME]/bin/jsw/conf/wrapper.conf（/usr/local/nexus/nexus-2.10.0-02/bin/jsw/conf） 文件中的 wrapper.java.command=java目录  就可以了，将它改成java的安装位置。
    wrapper.java.command=/usr/java/jdk1.8.0_151/bin/java
````

6.重新启动 成功

````
jvm 1    | 2018-09-17 00:25:10,375+0800 INFO  [jetty-main-1] *SYSTEM org.eclipse.jetty.server.AbstractConnector - Started InstrumentedSelectChannelConnector@0.0.0.0:8081

表示启动成功
````

7.访问 http://localhost:8081/nexus/#welcome 使用默认密码登录

````
默认管理员

账号：admin

密码：admin123

 

默认发布者

账号：deployment

密码：deployment123

````

8.登录后如图所示：

![nexus登录界面](/images/maven/nexus登录界面.png)

### nexus仓库与仓库组

有四种类型:group(仓库组),hosted(宿主),proxy(代理),virtual(虚拟)。仓库还有一个属性为Pilicy(策略),表示该仓库为发布(Release)版本还是快照(Snapshot)版本库。最后两列为仓库的状态和路径

下面解释一下各个仓库的用途，maven1格式的仓库不会介绍。此外由于虚拟类型仓库的作用实际上是动态地将仓库内容格式转换，换言之也是为了服务maven1格式，因此也被省略。

- Central:该仓库代理maven中央仓库，其策略为Release，因此只会下载和缓存中央仓库中的发布版本构件。

- Releases:这是一个策略为Release的宿主类型仓库，用来部署组织内部的发布版本构件。

- Snapshots:这是一个策略为Snapshots的宿主类型仓库，用来部署组织内部的快照版本构件。

- 3rd party:这是一个策略为Release的宿主类型仓库，用来部署无法从公共仓库获得的第三方发布版本构件。

- Apache Snapshots:这是一个策略为Snapshot的代理仓库，用来代理Apache Maven仓库的快照版本构件。

- Codehaus Snapshots:这是一个策略为Snapshot的代理仓库，用来代理Codehaus Maven仓库的快照版本构件。

- Public Repositories:该仓库组将上述所有策略为Release的仓库聚合并通过一致的地址提供服务。

### nexus仓库分类的概念

![各种类型的nexus仓库](/images/maven/各种类型的nexus仓库.png)

maven可以直接从宿主仓库下载构建，也可以直接从代理仓库下载构建，而代理仓库会直接从远程仓库下载并缓存构建。最后为了方便可以从仓库组下载构建。

### 从nexus私服下载jar包

在maven的settings的文件中配置

````
<profile>    
    <!--profile 的 id--> 
   <id>dev</id>    
    <repositories>    
      <repository>   
        <!--仓库 id，repositories 可以配置多个仓库，保证 id 不重复--> 
        <id>nexus</id>    
        <!--仓库地址，即 nexus 仓库组的地址--> 
        <url>http://localhost:8081/nexus/content/groups/public/</url>    
        <!--是否下载 releases 构件--> 
        <releases>    
          <enabled>true</enabled>    
        </releases>    
        <!--是否下载 snapshots 构件--> 
        <snapshots>    
          <enabled>true</enabled>    
        </snapshots>    
      </repository>    
    </repositories>   
     <pluginRepositories>   
     <!-- 插件仓库，maven 的运行依赖插件，也需要从私服下载插件 --> 
        <pluginRepository>   
         <!-- 插件仓库的 id 不允许重复，如果重复后边配置会覆盖前边 --> 
            <id>public</id>   
            <name>Public Repositories</name>   
            <url>http://localhost:8081/nexus/content/groups/public/</url>   
        </pluginRepository>   
    </pluginRepositories>   
  </profile>   

````

使用 profile 定义仓库需要激活才可生效。

````
  <activeProfiles> 
    <activeProfile>dev</activeProfile> 
  </activeProfiles> 
````


#### 比较常用的maven镜像

````
<mirror>
    <id>alimaven</id>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
<mirror>
    <id>ui</id>
    <mirrorOf>central</mirrorOf>
    <name>Human Readable Name for this Mirror.</name>
    <url>http://uk.maven.org/maven2/</url>
</mirror>
<mirror>
    <id>jboss-public-repository-group</id>
    <mirrorOf>central</mirrorOf>
    <name>JBoss Public Repository Group</name>
    <url>http://repository.jboss.org/nexus/content/groups/public</url>
</mirror>
<mirror>
    <id>repo2</id>
    <mirrorOf>central</mirrorOf>
    <name>Human Readable Name for this Mirror.</name>
    <url>http://repo2.maven.org/maven2/</url>
</mirror>
<mirror>
    <id>OSChina</id>
    <name>OSChina Central</name>
    <url>http://maven.oschina.net/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
<mirror>
    <id>nexus-osc-thirdparty</id>
    <mirrorOf>thirdparty</mirrorOf>
    <name>Nexus osc thirdparty</name>
    <url>http://maven.oschina.net/content/repositories/thirdparty/</url>
</mirror>

````

### maven镜像的使用


编辑settings.xml配置中央仓库镜像：

````
<settings>
  ...
  <mirrors>
    <mirror>
      <id>maven.net.cn</id>
      <name>one of the central mirrors in china</name>
      <url>http://maven.net.cn/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
  ...
</settings>

````

该例中，<mirrorOf>的值为central，表示该配置为中央仓库的镜像，任何对于中央仓库的请求都会转至该镜像，用户也可以使用同样的方法配置其他仓库的镜像。另外三个元素id,name,url与一般仓库配置无异，表示该镜像仓库的唯一标识符、名称以及地址。类似地，如果该镜像需认证，也可以基于该id配置仓库认证。
任何需要的构件都可以从私服获得，私服就是所有仓库的镜像。这时，可以配置这样的一个镜像，如例： 

````
<settings>
  ...
  <mirrors>
    <mirror>
      <id>internal-repository</id>
      <name>Internal Repository Manager</name>
      <url>http://192.168.1.100/maven2</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
  ...
</settings>

````
该例中<mirrorOf>的值为星号，表示该配置是所有Maven仓库的镜像，任何对于远程仓库的请求都会被转至http://192.168.1.100/maven2/。如果该镜像仓库需要认证，则配置一个Id为internal-repository的<server>即可。为了满足一些复杂的需求，Maven还支持更高级的镜像配置：

1.<mirrorOf>*</mirrorOf>

匹配所有远程仓库。

2.<mirrorOf>external:*</mirrorOf>

匹配所有远程仓库，使用localhost的除外，使用file://协议的除外。也就是说，匹配所有不在本机上的远程仓库。

3.<mirrorOf>repo1,repo2</mirrorOf>

匹配仓库repo1和repo2，使用逗号分隔多个远程仓库。

4.<mirrorOf>*,!repo1</miiroOf>

匹配所有远程仓库，repo1除外，使用感叹号将仓库从匹配中排除。

### 部署构件到nexus

如果只为代理外部公共仓库，那么nexus的代理仓库就已经完全能够满足需求了。对于宿主仓库来说，主要作用是存储组织内部的或者一些无法从公共仓库中获得的第三方构件。

使用maven部署构件至nexus，具体配置如下：

````
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
      <name>Nexus Release Repository</name>
      <url>http://42.121.113.40:8981/nexus/content/repositories/releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <name>Nexus Snapshot Repository</name>
    <url>http://42.121.113.40:8981/nexus/content/repositories/snapshots/</url>
  </snapshotRepository>
</distributionManagement>
    
 所有的release构件部署到Nexus的Releases仓库中。由于部署需要登陆，因为我们在settings.xml中配置对应Repository id的用户名和密码。

<servers>  
      <server>  
        <id>nexus-releases</id>  
        <username>admin</username>  
        <password>admin123</password>  
      </server>  
      <server>  
        <id>nexus-snapshots</id>  
        <username>admin</username>  
       <password>admin123</password>  
      </server>     
 </servers>  


````
