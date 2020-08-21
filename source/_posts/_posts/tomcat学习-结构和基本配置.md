---
title: tomcat学习-结构和基本配置
date: 2017-11-06 23:34:04
tags: tomcat
categories: web容器

---

- Tomcat的安装目录结构

```
bin: 启动和关闭Tomcat脚本文件。
conf: Tomcat服务器的各种配置文件，包括：server.xml、web.xml、catalina.policy等。
lib: Tomcat服务器和所有web应用可以访问的jar包。
logs: Tomcat的日志文件。
webapps: Tomcat自带的两个web应用：admin和manager，用来管理Tomcat的Web服务。
work: JSP经过Tomcat编译后生成的Servlet。
temp: Tomcat运行时的临时文件。

```

- Tomcat常用配置文件

```
server.xml：Tomcat中最重要的配置文件，定义了tomcat的体系结构，包括连接器端口、连接数、集群、虚拟目录、访问日志等的设置。
context.xml：全局context的配置文件，包括JNDI等信息的配置。
tocmat-users.xml：Tocmat管理员身份的配置文件，关键是设置管理员账号的密码。
logging.properties：Tocmat日志配置文件，可以修改默认的Tocmat日志路径和名称。

```

- Tomcat JVM参数调整

```
根据系统物理内存大小合理设置下列五个参数catalina.sh/catalina.bat

-server 
-Xms512m 
-Xmx512m
-XX:PermSize=128m
-XX:MaxPermSize=128m

   一般情况下，设置-Xms=-Xmx、-XX:PermSize=-XX:MaxPermSize，正式服务器必须设置以上参数，以尽可能压榨服务器性能。
相关参数取值需要根据实际情况考虑，不要超过(物理内存-其他程序内存)的80%即可。

没有特殊理由，尽量不要对上述五个参数外的其他JVM参数进行设置;
无法保证各种操作系统平台的可移植性;
过度干涉JVM内存管理会导致无法预料的后果;
如果在Windows平台上将解压版的Tomcat安装为服务，可以通过修改批处理文件$CATALINA_HOME/bin/service.bat对JVM参数进行调整。

```

- Tomcat日志配置

```
Tomcat日志信息包括访问日志和运行日志。

访问日志用于对用户访问系统的行为进行跟踪记录，主要记录用户访问的时间、对应的IP
地址、访问的资料等信息。记录访问日志主要是基于对系统安全的考虑，对系统中一些重
要、敏感信息的资料访问历史进行记录，便于对资源的访问历史进行追踪，对于敏感信息
未经授权访问等进行事后追查有一定帮助。但记录访问日志会对服务器性能产生一定的影
响，在生产系统中需要慎用。

运行日志主要记录程序运行的一些信息，其中的异常错误信息可以为我们定位错误。
从6.0版本开始，Tomcat的日志接口采用是对Apache Commons Logging日志接口进行独
立封装，缺省配置下，该日志接口采用硬编码使用java.util.logging日志框架。

由于Tomcat发布版本中独立封装的Apache Commons Logging接口并没有对接口完全实
现，如果要选择不同的日志框架就需要将该日志接口替换为完全实现的版本。

缺省配置下，Tomcat是不记录访问日志的，可以通过如下配置允许Tomcat记录访问日志：

修改$CATALINA_HOME/server.xml，在Host标签下，找到如下配置信息，去掉两端的注释
就会启用访问日志记录功能：

<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"

prefix="localhost_access_log." suffix=".txt" pattern="common" resolveHosts="false"/>

通过对pattern项的修改，可以改变日志输出的内容。该项值可以为： common 与 combined，对应的日志输出内容如下所示：

common: %h %l %u %t %r %s %b

combined: %h %l %u %t %r %s %b %{Referer}i %{User-Agent}i

pattern 也可以根据需要自由组合, 例如 pattern="%h %l"，对 于各 fields 字段的含义请参照Tomcat官方文档。

在不同的环境下，需要设置不同的日志级别，在生产环境中，为了提高效率和稳定性，一般
会将日志级别设置为相对较高的级别，而开发环境中为了跟踪程序流程，可以将日志级别
调整为较低的级别。不同日志框架有不同的日志级别，常用的日志框架对应级别如下：

Java.util.logging对应的日志级别由高到低分别为：

severe > warning > info > config > fine > finer > finest

org.apache.log4j对应的日志级别由高到低分别为：

fatal > error > warn > info > debug > trace

在缺省配置下，Tomcat采用Java.util.logging日志框架，对应的配置文件
为$CATALINA_HOME/ logging.properties，常用的日志级别设定方法如下：

设置catalina日志的级别为：FINE

1catalina.org.apache.juli.FileHandler.level = FINE

禁用catalina日志的输出：

1catalina.org.apache.juli.FileHandler.level = OFF

设置catalina所有的日志消息均输出：

1catalina.org.apache.juli.FileHandler.level = ALL

Log4j是目前应用最广的日志框架，可以使用Log4j替换Tomcat缺省采用的
java.util.logging
日志框架，步骤如下：

创建log4j配置文件log4j.properties ，保存在$CATALINA_HOME/lib 下。

从Apache官网Log4J项目下载Log4J（1.2版本以后）。

从Apache官网Tomcat项目下载tomcat-juli.jar和tomcat-juli-adapters.jar。

复制log4j.jar、tomcat-juli-adapters.jar到$CATALINA_HOME/lib下。

用tomcat-juli.jar覆盖$CATALINA_HOME/bin下的同名文件。

删除Tomcat的缺省日志配置文件$CATALINA_HOME/conf/ logging.properties，以避免生成一些冗余的空日志文件。


```


- Tomcat URL编码格式设置

```
缺省情况下，如果URL当中包含有非英文字符，需要通过在程序对URL进行转码处理，否
则URL中的非英文字符无法保证正确解析。在无特殊要求的情况下，需要将URL编码设置为
和项目统一的编码格式，目前公司大部分项目都统一采用UTF-8字符编码方式，示例如下：

<Connector port="8087" protocol="HTTP/1.1" connectionTimeout="20000"
redirectPort="8443" URIEncoding="utf-8" />

```

- Tomcat 常见问题总结

```
JVM内存溢出(OOM)，分为堆内存溢出和PermGen区内存溢出：

java.lang.OutOfMemoryError: PermGen space

PermGen space(Permanent Generation space)，是指内存的永久保存区域，主要用于存
放Class和Meta信息的，Class在被Loader时就会被放到PermGen space中, 它和存放类实
例(Instance)的Heap区域不同，GC(Garbage Collection)不会在主程序运行期对其进行清
理，所以如果应用中有很多CLASS的话，就很可能出现PermGen space错误。如果加载的Class超过MaxPermSize，就会抛出该异常，可以通过调整MaxPermSize进行解决。

java.lang.OutOfMemoryError: Java heap space

JVM堆是指java程序运行过程中JVM可以调配使用的内存空间。JVM在启动的时候会自动
设置Heap size的值，其初始空间(-Xms)是物理内存的1/64，最大空间(-Xmx)是物理内存
的1/4。可以利用JVM提供的-Xmn -Xms -Xmx等选项可进行设置。Heap size 的大小是
Young Generation 和Tenured Generaion 之和。在JVM中如果98％的时间是用于GC且
可用的Heap size 不足2％的时候将抛出此异常信息。

大量用户访问时浏览器没有响应
并发线程数设置太小，调整$CATALINA/conf/server.xml中连接器对应的请求处理线程数。

```

- tomcat的并发介绍

```
对web应用开发者来说，我们很关心应用可同时处理的请求数，以及响应时间。

对tomcat来说，每一个进来的请求(request)都需要一个线程，直到该请求结束。如果同时
进来的请求多于当前可用的请求处理线程数，额外的线程就会被创建，直到到达配置的最大
线程数(maxThreads属性值)。如果仍就同时接收到更多请求，这些来不及处理的请求就会
在Connector创建的ServerSocket中堆积起来，直到到达最大的配置值(acceptCount属性
值)。至此，任何再来的请求将会收到connection refused错误，直到有可用的资源来处理
它们。

这里我们关心的是tomcat能同时处理的请求数和请求响应时间，显然Connector元素
的maxThreads和acceptCount属性对其有直接的影响。无论acceptCount值为多
少，maxThreads直接决定了实际可同时处理的请求数。而不管maxThreads如何acceptCount
则决定了有多少请求可等待处理。然而，不管是可立即处理请求还是需要放入等待区，都
需要tomcat先接受该请求(即接受client的连接请求，建立socketchannel)，那么tomcat
同时可建立的连接数(maxConnections属性值)也会影响可同时处理的请求数。

```



- tomcat并发配置１

	
```
tomcat的http connector有三种：bio、nio、apr。从上面的属性描述中可以看出对于不同
的connector实现，相同的属性可能会有不同的默认值和不同的处理策略，所以在调整配
置前，要先弄清楚各种实现之间的不同，以及当前部署容器使用的是哪种connector。
查阅Tomcat7 http connector 配置文档Connector Comparison部分便可获知各种connector实现间的差异。
怎样才能知道容器使用的是何种connector实现？启动tomcat后，访问Server Status Page，看到如下信息即可知道使用的是何种connector：

我的OS是windows，所以tomcat默认使用的是aprconnector。在linux上，默认使用的是bio connector。与nio相比，bio性能较低。将<TOMCAT_HOME>/conf/server.xml中的如下配置片段：

<Connectorport="8080"protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
修改为：

<Connectorport="8080"protocol="org.apache.coyote.http11.Http11NioProtocol"
               connectionTimeout="20000"
               redirectPort="8443" />
就可将http connector切换至nio了。

　tomcat的最大线程数设置：

Tomcat的server.xml中连接器设置如下

   <Connectorport="8080"  maxThreads="150"minSpareThreads="25" maxSpareThreads="75"  enableLookups="false"redirectPort="8443" acceptCount="100"  debug="0"connectionTimeout="20000"  disableUploadTimeout="true"/> 
 tomcat在配置时设置最大线程数，当前线程数超过这个数值时会出错

minProcessors：最小空闲连接线程数，用于提高系统处理性能，默认值为10
maxProcessors：最大连接线程数，即：并发处理的最大请求数，默认值为75
acceptCount：允许的最大连接数，应大于等于maxProcessors，默认值为100
enableLookups：是否反查域名，取值为：true或false。为了提高处理能力，
应设置为false  
connectionTimeout：网络连接超时，单位：毫秒。设置为0表示永不超时，这样设置有隐患的。通常可设置为30000毫秒。
其中和最大连接数相关的参数为maxProcessors和acceptCount。如果要加大并发连接数，应同时加大这两个参数。

```

- 如何加大tomcat可以使用的内存

tomcat默认可以使用的内存为128MB，在较大型的应用项目中，这点内存是不够的，需要调大。
Unix下，在文件{tomcat_home}/bin/catalina.sh的前面，增加如下设置：
JAVA_OPTS='-Xms【初始化内存大小】 -Xmx【可以使用的最大内存】'
需要把这个两个参数值调大。例如：
JAVA_OPTS='-Xms256m -Xmx512m'
表示初始化内存为256MB，可以使用的最大内存为512MB



- Tomcat中的并发配置２

[官网关于线程数的配置](http://tomcat.apache.org/tomcat-7.0-doc/config/http.html)

[修改Tomcat Connector运行模式](http://www.365mini.com/page/tomcat-connector-mode.htm)

tomcat的线程配置有两种：可以在connector中配置，也可以配置executor。

连接器中的线程配置是私有的，连接器自己的配置只能自己使用；而线程池可以被共享，
多个连接器通过executor属性关联到连接池的name完成配置（见下图）；
一旦连接器中配置了一个存在的executor，那么线程池的配置将会覆盖连接器的线程配置。
![连接器和线程池](/images/tomcat/tomcat3.png)

线程池覆盖连接器中的线程配置，主要是覆盖maxThreads和minSpareThreads两个参数，这两个参数在线程池标签和连接器标签中都存在。
    a. 对于线程池而言：最主要的配置项是maxThreads和minSpareThreads（参看3.1表格，官网全部配置项参看4.1链接），前者代表线程池中可以创建的最大线程数，后者代表当线程池空闲的时候保留的线程数量，当线程池覆盖连接器的线程配置的时候，最主要的是覆盖这两个参数，连接器如何关联线程池，参看上文截图。
    b. 对于连接器而言：线程相关的主要参数有四个——maxThreads、minSpareThreads、maxConnections、acceptCount，如果连接器关联了线程池，那么maxThreads和minSpareThreads会被覆盖（重要的事情反复说），acceptCount和maxConnections继续生效；
    maxThreads：一个请求分配一个线程处理，其实就是serversocket.accept之后，新建了一个线程去处理这个接受的socket，然后这个连接器可以建立的最大线程数量即为maxThreads配置的值，如果是多个连接器关联了同一个线程池，那么多个连接器允许建立的线程数量之和即为maxThreads的值；
    minSpareThreads：如果请求少或者没有请求，那么tomcat就会主动关闭一些不用的线程，但是至少会保留minSpareThreads配置的线程数；
    maxConnections：表示可以同时被当前连接器处理的请求数量，这个配置项需要和maxThreads进行对比，一个请求一旦被接受处理就会分配给一个线程，这时候能够同时处理多少个请求取决于maxConnections和maxThreads两个配置项中的较小者，如果连接器是BIO则maxConnections默认等于maxThreads（连接器的maxThreads，如果关联了线程池，则为线程池中的maxThreads），NIO2默认是10000，APR默认是8192（BIO/NIO/NIO2/APR主要是根据connector的protocol属性，这个属性可以配置小表中的类）；
    举个例子，连接器中maxThreads=200，maxConnections=100，这时候因为线程池最多只允许创建100个线程，因此能被同时处理的请求数量是100，取决于两者中的较小者；但是如果连接器关联了线程池maxThreads=200，连接器A的maxConnections=120，连接器B的maxConnections=100，这时候对于连接器A而言，处理请求的最大的线程数是120，不会超过120，但是能不能达到120还需要看线程池中的可用的线程数，如果这时候连接器B已经拿了100个线程，那么此时A只能拿到100个线程；

```
Java Blocking Connector BIO	Java Nio Connector NIO	 Java Nio2 Connector NIO2	APRnative Connector
APR
Classname	Http11Protocol	Http11NioProtocol	Http11Nio2Protocol	Http11AprProtocol

```

    acceptCount：相当于建立ServerSocket的时候构造函数中传入的backlog参数，表
示serversocket中完成三次握手后（成功建立TCP连接）允许等待被accept的socket数量，
已经建立好了TCP连接，但是socket还没有被accept取出来处理；



-  Tomcat结构
Tomcat是一个基于组件的服务器，它的构成组件都是可配置的，其中最外层的组件是Catalina Servlet容器，其他的组件按照一定的格式要求配置在这个顶层容器中。Tomcat的各个组件是在<TOMCAT_HOME>\conf\server.xml文件中配置的，Tomcat服务器默认情况下对各种组件都有默认的实现，下面通过分析server.xml文件来理解Tomcat的各个组件是如何组织的。server.xml文件的基本组成结构如下。

 ![](/images/tomcat/tomcat1.jpg)


```
XML配置文件结构
<Server>                     顶层类元素：一个配置文件中只能有一个<Server>元素，可包含多个Service。
    <Service>                顶层类元素：本身不是容器，可包含一个Engine，多个Connector。
        <Connector/>         连接器类元素：代表通信接口。
           <Engine>   容器类元素：为特定的Service组件处理所有客户请求，可包含多个Host。
              <Host>    容器类元素：为特定的虚拟主机处理所有客户请求，可包含多个Context。
                 <Context>   容器类元素：为特定的Web应用处理所有客户请求。
                 </Context>
               </Host>
              </Engine>
     </Service>
</Server>

1)Service
Service组件是一些Connector组件的集合，它本身不是一个容器，所以在这里不能定义日志
等组件。一个Service组件中只能有一个Engine组件，可以包含多个Connector组件。

2)Connector组件
Connector组件表示一个接口，通过这个接口接收客户的请求，然户发送给其他的容器组件，最后再把服务器的响应结果传递给客户。

3) Engine, Host和context
上面介绍的3个组件本身并不能处理客户请求，也不能生成响应。在Tomcat中只有3个组件是
可以处理客户请求并生成响应的，这3个组件分别是 Engine、Host和Context组件。这3个组
件分别代表了不同的服务范围，通过嵌套关系可以知道3个组件的范围有如下的关系：Engine>Host>Context。
a.Engine组件下可以包含多个Host组件，它为特定的Service组件处理所有客户请求。
b.一个Host组件代表一个虚拟主机，一个虚拟主机中可以包含多个Web应用（Context组件）。
c.Context组件代表一个Web应用。
Tomcat的各个组件关系，可以用下图描述。
```
 ![](/images/tomcat/tomcat2.jpg)

```
2. Tomcat处理一个HTTP请求的过程
假设来自客户的请求为： http://localhost:8080/wsota/wsota_index.jsp 
1) 请求被发送到本机端口8080，被在那里侦听的Coyote HTTP/1.1 Connector获得 
2) Connector把该请求交给它所在的Service的Engine来处理，并等待来自Engine的回应 
3) Engine获得请求localhost/wsota/wsota_index.jsp，匹配它所拥有的所有虚拟主机Host 
4) Engine匹配到名为localhost的Host（即使匹配不到也把请求交给该Host处理，因为该Host被定义为该Engine的默认主机） 
5) localhost Host获得请求/wsota/wsota_index.jsp，匹配它所拥有的所有Context 
6) Host匹配到路径为/wsota的Context（如果匹配不到就把该请求交给路径名为""的Context去处理） 
7) path="/wsota"的Context获得请求/wsota_index.jsp，在它的mapping table中寻找对应的servlet 
8) Context匹配到URL PATTERN为*.jsp的servlet，对应于JspServlet类 
9) 构造HttpServletRequest对象和HttpServletResponse对象，作为参数调用JspServlet的doGet或doPost方法 
10)Context把执行完了之后的HttpServletResponse对象返回给Host 
11)Host把HttpServletResponse对象返回给Engine 
12)Engine把HttpServletResponse对象返回给Connector 
13)Connector把HttpServletResponse对象返回给客户browser



```

[tomcat的结构](http://www.cnblogs.com/xzpp/archive/2012/06/15/2550473.html)

[官网connector配置](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html)

[官网executor配置](http://tomcat.apache.org/tomcat-8.0-doc/config/executor.html)

[参考资料１](http://www.cnblogs.com/doit8791/archive/2012/10/27/2742768.html)

[参考资料２](http://www.cnblogs.com/softidea/p/5750791.html)

[参考资料３](http://www.cnblogs.com/softidea/p/5750791.html)

[tomcat源码解析](https://blog.csdn.net/w1992wishes/article/details/79242797)







