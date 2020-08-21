---
title: python2.7高级教程入门
copyright: true
date: 2018-07-11 01:57:51
tags: python2.7高级教程
categories: python
---

# 面向对象

## 类

实例：

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

class Employee:
    ' 所有员工的基类 '
    empCount = 0

    def __init__(self, name, salary):
        self.name = name
        self.salary = salary
        Employee.empCount += 1

    def displayCount(self):
        print "total employee %d" % Employee.empCount

    def displayEmployee(self):
        print "name:", self.name, ",salary:", self.salary


# empCount 变量是一个类变量，它的值将在这个类的所有实例之间共享。你可以在内部类或外部类使用 Employee.empCount 访问。
# 第一种方法__init__()方法是一种特殊的方法，被称为类的构造函数或初始化方法，当创建了这个类的实例时就会调用该方法
# self 代表类的实例，self 在定义类的方法时是必须有的，虽然在调用时不必传入相应的参数。

````

## 创建实例对象和访问属性与调用方法


````
emp1 = Employee("lee", 2000)
emp2 = Employee("zhang", 4000)

emp1.displayCount()
emp2.displayEmployee()

print "total employee %d" % Employee.empCount

````

## 对属性的操作

````
emp1.age = 7  # 添加一个 'age' 属性
emp1.age = 8  # 修改 'age' 属性
del emp1.age  # 删除 'age' 属性

hasattr(emp1, 'age')    # 如果存在 'age' 属性返回 True。
getattr(emp1, 'age')    # 返回 'age' 属性的值
setattr(emp1, 'age', 8) # 添加属性 'age' 值为 8
delattr(emp1, 'age')    # 删除属性 'age'


````

## python内置类属性


- __dict__ : 类的属性（包含一个字典，由类的数据属性组成）
- __doc__ :类的文档字符串
- __name__: 类名
- __module__: 类定义所在的模块（类的全名是'__main__.className'，如果类位于一个导入模块mymod中，那么className.__module__ 等于 mymod）
- __bases__ : 类的所有父类构成元素（包含了一个由所有父类组成的元组）

## python对象销毁(垃圾回收)

Python 使用了引用计数这一简单技术来跟踪和回收垃圾。在 Python 内部记录着所有使用中的对象各有多少引用。
一个内部跟踪变量，称为一个引用计数器。
当对象被创建时， 就创建了一个引用计数， 当这个对象不再需要时， 也就是说， 这个对象的引用计数变为0 时， 它被垃圾回收。但是回收不是"立即"的， 由解释器在适当的时机，将垃圾对象占用的内存空间回收。


````
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
class Point:
   def __init__( self, x=0, y=0):
      self.x = x
      self.y = y
   def __del__(self):
      class_name = self.__class__.__name__
      print class_name, "销毁"
 
pt1 = Point()
pt2 = pt1
pt3 = pt1
print id(pt1), id(pt2), id(pt3) # 打印对象的id
del pt1
del pt2
del pt3

````

## 类的继承

通过继承创建的新类称为子类或派生类，被继承的类称为基类、父类或超类。

````
class SubClassName (ParentClass1[, ParentClass2, ...]):

````

## 方法重写

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
class Parent:        # 定义父类
   def myMethod(self):
      print '调用父类方法'
 
class Child(Parent): # 定义子类
   def myMethod(self):
      print '调用子类方法'
 
c = Child()          # 子类实例
c.myMethod()         # 子类调用重写方法

````
## 类属性与方法

### 类的私有属性

__private_attrs：两个下划线开头，声明该属性为私有，不能在类的外部被使用或直接访问。在类内部的方法中使用时 self.__private_attrs。

### 类的方法

在类的内部，使用 def 关键字可以为类定义一个方法，与一般函数定义不同，类方法必须包含参数 self,且为第一个参数

### 类的私有方法

__private_method：两个下划线开头，声明该方法为私有方法，不能在类的外部调用。在类的内部调用 self.__private_methods

Python不允许实例化的类访问私有数据，但你可以使用 object._className__attrName（ 对象名._类名__私有属性名 ）访问属性

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

class Runoob:
    __site = "www.runoob.com"

runoob = Runoob()
print runoob._Runoob__site

````

## 单下划线、双下划线、头尾双下划线说明：

- __foo__: 定义的是特殊方法，一般是系统定义名字 ，类似 __init__() 之类的。
- _foo: 以单下划线开头的表示的是 protected 类型的变量，即保护类型只能允许其本身与子类进行访问，不能用于 from module import *
- __foo: 双下划线的表示的是私有类型(private)的变量, 只能是允许这个类本身进行访问了。


# python正则表达式

Python 自1.5版本起增加了re 模块，它提供 Perl 风格的正则表达式模式。

## re.match函数

re.match 尝试从字符串的起始位置匹配一个模式，如果不是起始位置匹配成功的话，match()就返回none。

re.match(pattern, string, flags=0)
- pattern	匹配的正则表达式

- string	要匹配的字符串。

- flags	标志位，用于控制正则表达式的匹配方式，如：是否区分大小写，多行匹配等等。

## group

我们可以使用group(num) 或 groups() 匹配对象函数来获取匹配表达式。

- group(num=0)	匹配的整个表达式的字符串，group() 可以一次输入多个组号，在这种情况下它将返回一个包含那些组所对应值的元组。
- groups()	返回一个包含所有小组字符串的元组，从 1 到 所含的小组号。

## re.search方法

re.search  扫描整个字符串并返回第一个成功的匹配。

## re.match与re.search的区别

re.match只匹配字符串的开始，如果字符串开始不符合正则表达式，则匹配失败，函数返回None；而re.search匹配整个字符串，直到找到一个匹配。


## 检索和替换 re.sub

````
re.sub(pattern, repl, string, count=0, flags=0)

````
- pattern : 正则中的模式字符串。
- repl : 替换的字符串，也可为一个函数。
- string : 要被查找替换的原始字符串。
- count : 模式匹配后替换的最大次数，默认 0 表示替换所有的匹配。

## re.compile 函数

compile 函数用于编译正则表达式，生成一个正则表达式（ Pattern ）对象，供 match() 和 search() 这两个函数使用。
语法格式为：

````
re.compile(pattern[, flags])

pattern : 一个字符串形式的正则表达式
flags : 可选，表示匹配模式，比如忽略大小写，多行模式等，具体参数为：
re.I 忽略大小写
re.L 表示特殊字符集 \w, \W, \b, \B, \s, \S 依赖于当前环境
re.M 多行模式
re.S 即为 . 并且包括换行符在内的任意字符（. 不包括换行符）
re.U 表示特殊字符集 \w, \W, \b, \B, \d, \D, \s, \S 依赖于 Unicode 字符属性数据库
re.X 为了增加可读性，忽略空格和 # 后面的注释

````

例子：

````
pattern = re.compile(r'\d+')                    # 用于匹配至少一个数字
m = pattern.match('one12twothree34four')        # 查找头部，没有匹配
print m
m = pattern.match('one12twothree34four', 2, 10) # 从'e'的位置开始匹配，没有匹配
print m
m = pattern.match('one12twothree34four', 3, 10) # 从'1'的位置开始匹配，正好匹配
print m                                         # 返回一个 Match 对象
print m.group(0)   # 可省略 0
print m.start(0)   # 可省略 0
print m.end(0)     # 可省略 0
print m.span(0)    # 可省略 0

````
在上面，当匹配成功时返回一个 Match 对象，其中：

- group([group1, …]) 方法用于获得一个或多个分组匹配的字符串，当要获得整个匹配的子串时，可直接使用 group() 或 group(0)；
- start([group]) 方法用于获取分组匹配的子串在整个字符串中的起始位置（子串第一个字符的索引），参数默认值为 0；
- end([group]) 方法用于获取分组匹配的子串在整个字符串中的结束位置（子串最后一个字符的索引+1），参数默认值为 0；
- span([group]) 方法返回 (start(group), end(group))。


## findall 

在字符串中找到正则表达式所匹配的所有子串，并返回一个列表，如果没有找到匹配的，则返回空列表。

````
findall(string[, pos[, endpos]])

````

- string : 待匹配的字符串。
- pos : 可选参数，指定字符串的起始位置，默认为 0。
- endpos : 可选参数，指定字符串的结束位置，默认为字符串的长度。

## re.finditer

和 findall 类似，在字符串中找到正则表达式所匹配的所有子串，并把它们作为一个迭代器返回。

re.finditer(pattern, string, flags=0)

````
it = re.finditer(r"\d+","12a32bc43jf3")
for match in it:
    print (match.group() )

````

## re.split 

split 方法按照能够匹配的子串将字符串分割后返回列表

## 正则表达式对象


### re.RegexObject

re.compile() 返回 RegexObject 对象。

### re.MatchObject

- group() 返回被 RE 匹配的字符串。
- start() 返回匹配开始的位置
- end() 返回匹配结束的位置
- span() 返回一个元组包含匹配 (开始,结束) 的位置

## 正则表达式模式

- 字母和数字表示他们自身。一个正则表达式模式中的字母和数字匹配同样的字符串。
- 多数字母和数字前加一个反斜杠时会拥有不同的含义。
- 标点符号只有被转义时才匹配自身，否则它们表示特殊的含义。
- 反斜杠本身需要使用反斜杠转义。
- 由于正则表达式通常都包含反斜杠，所以你最好使用原始字符串来表示它们。模式元素(如 r'\t'，等价于 '\\t')匹配相应的特殊字符。


# CGI编程

Python 提供了两个级别访问的网络服务。：
1. 低级别的网络服务支持基本的 Socket，它提供了标准的 BSD Sockets API，可以访问底层操作系统Socket接口的全部方法。
2. 高级别的网络服务模块 SocketServer， 它提供了服务器中心类，可以简化网络服务器的开发。

## socket()函数

socket.socket([family[, type[, proto]]])

family: 套接字家族可以使AF_UNIX或者AF_INET

type: 套接字类型可以根据是面向连接的还是非连接分为SOCK_STREAM或SOCK_DGRAM

protocol: 一般不填默认为0.

## Socket 对象(内建)方法

### 服务器端套接字

s.bind() 绑定地址（host,port）到套接字， 在AF_INET下,以元组（host,port）的形式表示地址。

s.listen() 开始TCP监听

s.accept() 被动接受TCP客户端连接,(阻塞式)等待连接的到来

### 客户端套接字 

s.connect()	主动初始化TCP服务器连接，。一般address的格式为元组（hostname,port），如果连接出错，返回socket.error错误。

### 公共用途的套接字函数

s.recv() 接收TCP数据，数据以字符串形式返回，bufsize指定要接收的最大数据量。flag提供有关消息的其他信息，通常可以忽略。

s.send() 发送TCP数据，将string中的数据发送到连接的套接字。返回值是要发送的字节数量，该数量可能小于string的字节大小。

s.close() 关闭套接字

客户端示例：

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-
# 文件名：client.py

import socket               # 导入 socket 模块

s = socket.socket()         # 创建 socket 对象
host = socket.gethostname() # 获取本地主机名
port = 12345                # 设置端口号

s.connect((host, port))
print s.recv(1024)
s.close() 

````



### Python Internet 模块

![Python Internet 模块](/images/python/2.7/PythonInternet模块.png)

# python多线程

````
thread.start_new_thread ( function, args[, kwargs] 

function - 线程函数。

args - 传递给线程函数的参数,他必须是个tuple类型。

kwargs - 可选参数。

````

## python多线程的例子

````
#!/usr/bin/python
# -*- coding: utf-8 -*-

import thread
import time


# 为线程定义一个函数
def print_time(threadName, delay):
    count = 0
    while (count < 5):
        time.sleep(delay)
        count = +1
        print "%s: %s" % (threadName, time.ctime(time.time()))


# 创建两个线程
try:
    thread.start_new_thread(print_time, ("Thread-1", 2), )
    thread.start_new_thread(print_time, ("Thread-2", 4), )
except:
    print "Error: unable to start thread"

while 1:
    pass

````

## 线程模块

Python通过两个标准库thread和threading提供对线程的支持。thread提供了低级别的、原始的线程以及一个简单的锁

threading.currentThread(): 返回当前的线程变量。

threading.enumerate(): 返回一个包含正在运行的线程的list。正在运行指线程启动后、结束前，不包括启动前和终止后的线程。

threading.activeCount(): 返回正在运行的线程数量，与len(threading.enumerate())有相同的结果。

线程模块同样提供了Thread类来处理线程，Thread类提供了以下方法:

run(): 用以表示线程活动的方法。

start():启动线程活动。

join([time]): 等待至线程中止。这阻塞调用线程直至线程的join() 方法被调用中止-正常退出或者抛出未处理的异常-或者是可选的超时发生。

isAlive(): 返回线程是否活动的。

getName(): 返回线程名。

setName(): 设置线程名。

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
import threading
import time
 
exitFlag = 0
 
class myThread (threading.Thread):   #继承父类threading.Thread
    def __init__(self, threadID, name, counter):
        threading.Thread.__init__(self)
        self.threadID = threadID
        self.name = name
        self.counter = counter
    def run(self):                   #把要执行的代码写到run函数里面 线程在创建后会直接运行run函数 
        print "Starting " + self.name
        print_time(self.name, self.counter, 5)
        print "Exiting " + self.name
 
def print_time(threadName, delay, counter):
    while counter:
        if exitFlag:
            (threading.Thread).exit()
        time.sleep(delay)
        print "%s: %s" % (threadName, time.ctime(time.time()))
        counter -= 1
 
# 创建新线程
thread1 = myThread(1, "Thread-1", 1)
thread2 = myThread(2, "Thread-2", 2)
 
# 开启线程
thread1.start()
thread2.start()
 
print "Exiting Main Thread"
````

##  线程同步

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
import threading
import time
 
class myThread (threading.Thread):
    def __init__(self, threadID, name, counter):
        threading.Thread.__init__(self)
        self.threadID = threadID
        self.name = name
        self.counter = counter
    def run(self):
        print "Starting " + self.name
       # 获得锁，成功获得锁定后返回True
       # 可选的timeout参数不填时将一直阻塞直到获得锁定
       # 否则超时后将返回False
        threadLock.acquire()
        print_time(self.name, self.counter, 3)
        # 释放锁
        threadLock.release()
 
def print_time(threadName, delay, counter):
    while counter:
        time.sleep(delay)
        print "%s: %s" % (threadName, time.ctime(time.time()))
        counter -= 1
 
threadLock = threading.Lock()
threads = []
 
# 创建新线程
thread1 = myThread(1, "Thread-1", 1)
thread2 = myThread(2, "Thread-2", 2)
 
# 开启新线程
thread1.start()
thread2.start()
 
# 添加线程到线程列表
threads.append(thread1)
threads.append(thread2)
 
# 等待所有线程完成
for t in threads:
    t.join()
print "Exiting Main Thread"

````

# python JSON

使用 Python 语言来编码和解码 JSON 对象。

1. json.dumps	将 Python 对象编码成 JSON 字符串

2. json.loads	将已编码的 JSON 字符串解码为 Python 对象

**json类型转换到python的类型对照表**

![json类型转换到python的类型对照表](/images/python/2.7/json类型转换到python的类型对照表.png)


[Demjson](https://github.com/dmeranda/demjson) 是python的第三方JSON类库。

1.encode	将 Python 对象编码成 JSON 字符串

2.decode	将已编码的 JSON 字符串解码为 Python 对象


# python2.0和python3的区别

1. print语句没有了，取而代之的是print()函数
2. Python3.X 源码文件默认使用utf-8编码
3. 除法运算：

   在python 2.x中/除法就跟我们熟悉的大多数语言，比如Java啊C啊差不多，整数相除的结果是一个整数，把小数部分完全忽略掉，浮点数除法会保留小数点的部分得到一个浮点数的结果。
   
   在python 3.x中/除法不再这么做了，对于整数之间的相除，结果也会是浮点数

等等。。






















