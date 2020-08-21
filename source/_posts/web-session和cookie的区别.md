---
title: web-session和cookie的区别
date: 2017-10-14 00:02:19
tags: redis,session共享
categories: web 

---
## session和cookie时开发中常用到的两个对象。
#### cookie
````
    cookie是一小段文本信息，伴随着用户请求和页面在web服务器和浏览器之间传递。cookie包含每次用户访问站点用户都可以读取的
信息。(cookie会随着每次http请求一起被传到服务端）
    当你在浏览网站的时候，WEB 服务器会先送一小小资料放在你的计算机上，Cookie 会帮你在网站上所打的文字或是一些选择，都纪
录下来。当下次你再光临同一个网站，WEB 服务器会先看看有没有它上次留下的 Cookie 资料，有的话，就会依据 Cookie里的内容来判
断使用者，送出特定的网页内容给你。 Cookie 的使用很普遍，许多有提供个人化服务的网站，都是利用 Cookie来辨认使用者，以方便
送出使用者量身定做的内容，像是 Web 接口的免费 email 网站，都要用到 Cookie。 
    具体来说cookie机制采用的是在客户端保持状态的方案，而session机制采用的是在服务器端保持状态的方案。
    cookie的内容主要包括：名字，值，过期时间，路径和域。路径与域一起构成cookie的作用范围。若不设置过期时间，则表示这个
cookie的生命期为浏览器会话期间，关闭浏览器窗口，cookie就消失。这种生命期为浏览器会话期的cookie被称为会话cookie。会话
cookie一般不存储在硬盘上而是保存在内存里，当然这种行为并不是规范规定的。若设置了过期时间，浏览器就会把cookie保存到硬
盘上，关闭后再次打开浏览器，这些cookie仍然有效直到超过设定的过期时间。存储在硬盘上的cookie可以在不同的浏览器进程间共
享，比如两个IE窗口。而对于保存在内存里的cookie，不同的浏览器有不同的处理方式。
````
#### session
````
    session我们可以使用它来方便地在服务端保存一些与会话相关的信息。比如常见的登录信息。
    session实现原理：http协议是无状态的，对于一个浏览器发出的多次请求，web服务无法区分是不是来自同一个浏览器。所以会
通过一个sessionid来区分。
    当程序需要为某个客户端的请求创建一个session时，服务器首先检查这个客户端的请求里是否已包含了一个session标识(称为
session id），如果已包含则说明以前已经为此客户端创建过session，服务器就按照session id把这个session检索出来使用（检索
不到，会新建一个），如果客户端请求不包含session id，则为此客户端创建一个session并且生成一个与此session相关联的session
 id，session id的值应该是一个既不会重复，又不容易被找到规律以仿造的字符串，这个session id将被在本次响应中返回给客户端保
存。保存这个session id的方式可以采用cookie，这样在交互过程中浏览器可以自动的按照规则把这个标识发送给服务器。一般这个
cookie的名字都是类似于SEEESIONID。但cookie可以被人为的禁止，则必须有其他机制以便在cookie被禁止时仍然能够把session id
传递回服务器。
    经常被使用的一种技术叫做URL重写，就是把session id直接附加在URL路径的后面。还有一种技术叫做表单隐藏字段。就是服务器
会自动修改表单，添加一个隐藏字段，以便在表单提交时能够把session id传递回服务器。

````

#### cookie 和session 的区别：

1. cookie数据存放在客户的浏览器上，session数据放在服务器上。
2. cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗
   考虑到安全应当使用session。
3. session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能
   考虑到减轻服务器性能方面，应当使用COOKIE。
4. 单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。
5.  所以个人建议：
 	将登陆信息等重要信息存放为SESSION；其他信息如果需要保留，可以放在COOKIE中

[session深入探讨](https://maimai.cn/article/detail?fid=971952342&efid=LLj6y1u5UJki64UG-HamIA)


