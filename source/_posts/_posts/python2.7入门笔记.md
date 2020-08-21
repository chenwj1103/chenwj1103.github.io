---
title: python2.7入门笔记
copyright: true
date: 2018-06-29 00:22:00
tags: python2.7入门
categories: python

---

# python介绍

python是一种解释型、面向对象、动态数据类型高级程序设计语言、脚本语言。由荷兰人发明。

# 基础语法

## Python 标识符

1. 在 Python 里，标识符由字母、数字、下划线组成。
2. 在 Python 中，所有标识符可以包括英文、数字以及下划线(_)，**但不能以数字开头**。
3. Python 中的标识符是区分大小写的。
4. 以**下划线开头**的标识符是有特殊意义的。以单下划线开头 _foo 的代表**不能直接访问的类属性**，需通过类提供的接口进行访问，不能用 **from xxx import * 而导入**
5. 以**双下划线开头**的 __foo 代表类的私有成员；以**双下划线开头和结尾**的 __foo__ 代表 Python 里特殊方法专用的标识，如 __init__() 代表类的构造函数
6. Python 可以同一行显示多条语句，方法是用分号 ; 分开

## Python保留字

包括 and **exec** **not** assert finally or break for **pass** class **from** print continue **global** **raise** def if return **del**

import try **elif** in while else **is** **with** **except** **lambda** yield

## 行和缩进

Python 的代码块不使用大括号 {} 来控制类，函数以及其他逻辑判断。python 最具特色的就是用缩进来写模块。

**常见的错误：** IndentationError: unindent does not match any outer indentation level错误表明，你使用的缩进方式不一致，有的是 tab 键缩进，有的是空格缩进，改为一致即可。

## 多行语句

Python语句中一般以新行作为语句的结束符。

但是我们可以使用斜杠（ \）将一行的语句分为多行显示。

````
total =item_one + \
       item_two + \
       item_three

````
语句中包含 [], {} 或 () 括号就不需要使用多行连接符。如下实例：

````
days = ['Monday', 'Tuesday', 'Wednesday',
        'Thursday', 'Friday']

````

## Python 引号

Python 可以使用引号( ' )、双引号( " )、三引号( ''' 或 """ ) 来表示字符串，引号的开始与结束必须的相同类型的。

其中三引号可以由多行组成，编写多行文本的快捷语法，常用于文档字符串，在文件的特定地点，被当做注释。

## Python 注释

python中单行注释采用 # 开头。可以在语句或者表达式行末；

多行使用三引号表示

````
'''
这是多行注释，使用单引号。
这是多行注释，使用单引号。
这是多行注释，使用单引号。
'''

"""
这是多行注释，使用双引号。
这是多行注释，使用双引号。
这是多行注释，使用双引号。
"""

````

## Python空行

函数之间或类的方法之间用空行分隔，表示一段新的代码的开始。类和函数入口之间也用一行空行分隔，以突出函数入口的开始。

空行与代码缩进不同，空行并不是Python语法的一部分。书写时不插入空行，Python解释器运行也不会出错。但是空行的作用在于分隔两段不同功能或含义的代码，便于日后代码的维护或重构。

记住：空行也是程序代码的一部分。

## 等待用户输入

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

raw_input("按下 enter键退出，其他任意键显示。。。\n")

````

## 同一行显示多条语句

同一行显示多条语句使用;分割

## Print输出

print 默认输出是换行的，如果要实现不换行需要在变量末尾加上逗号 ,

# 变量类型

有五个标准的数据类型：Numbers（数字）/  String（字符串） /List（列表）/Tuple（元组）/Dictionary（字典）
           
python中变量类型不需要声明，每个变量在内存中创建都包括变量的标识，名称和数据这些信息。

每个变量在使用前都会被赋值，变量赋值后才会被创建。

## 多个变量赋值

a =b =c =1

或者 a, b ,c =1,2,"hohn"

## python数字

当你指定一个值时，Number对象就会被创建，使用del语句删除单个或多个对象的引用

````
var1 = 1

var2 =10

del var1,var2

````

python支持4种不同的数据类型 int long float complex（复数）

### Python math 模块、cmath 模块

Python 中数学运算常用的函数基本都在 math 模块、cmath 模块中。

Python math 模块提供了许多对浮点数的数学运算函数。

Python cmath 模块包含了一些用于复数运算的函数。

cmath 模块的函数跟 math 模块函数基本一致，区别是 cmath 模块运算的是复数，math 模块运算的是数学运算。


### python数学函数

abs(x)	返回数字的绝对值，如abs(-10) 返回 10

ceil(x)	返回数字的上入整数，如math.ceil(4.1) 返回 5

cmp(x, y)	如果 x < y 返回 -1, 如果 x == y 返回 0, 如果 x > y 返回 1

exp(x)	返回e的x次幂(ex),如math.exp(1) 返回2.718281828459045

fabs(x)	返回数字的绝对值，如math.fabs(-10) 返回10.0

floor(x)	返回数字的下舍整数，如math.floor(4.9)返回 4

log(x)	如math.log(math.e)返回1.0,math.log(100,10)返回2.0

log10(x)	返回以10为基数的x的对数，如math.log10(100)返回 2.0

max(x1, x2,...)	返回给定参数的最大值，参数可以为序列。

min(x1, x2,...)	返回给定参数的最小值，参数可以为序列。

modf(x)	返回x的整数部分与小数部分，两部分的数值符号与x相同，整数部分以浮点型表示。

pow(x, y)	x**y 运算后的值。

round(x [,n])	返回浮点数x的四舍五入值，如给出n值，则代表舍入到小数点后的位数。

sqrt(x)	返回数字x的平方根

## python字符串

字符串的取值顺序： 从左到右索引默认0开始的，最大范围是字符串长度少1。

如果你要实现从字符串中获取一段子字符串的话，可以使用变量 [头下标:尾下标]，就可以截取相应的字符串(包括头下标，不包括尾下标)

加号（+）是字符串连接运算符，星号（*）是重复操作

````
print str  # 输出完整字符串
print str[0]  # 输出字符串中的第一个字符
print str[2:5]  # 输出字符串中第三个至第五个之间的字符串
print str[2:]  # 输出从第三个字符开始的字符串
print str * 2  # 输出字符串两次
print str + "TEST"  # 输出连接的字符串

````
### python字符串操作符


````
a值为：hello  b值为：world

+	字符串连接   
>>>a + b
'HelloPython'

*	重复输出字符串
>>>a * 2
'HelloHello'

[]	通过索引获取字符串中字符
>>>a[1]
'e'

[ : ]	截取字符串中的一部分
>>>a[1:4]
'ell'

in	成员运算符 - 如果字符串中包含给定的字符返回 True
>>>"H" in a
True


````

实例

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
a = "Hello"
b = "Python"
 
print "a + b 输出结果：", a + b 
print "a * 2 输出结果：", a * 2 
print "a[1] 输出结果：", a[1] 
print "a[1:4] 输出结果：", a[1:4] 
 
if( "H" in a) :
    print "H 在变量 a 中" 
else :
    print "H 不在变量 a 中" 
 
if( "M" not in a) :
    print "M 不在变量 a 中" 
else :
    print "M 在变量 a 中"
 
print r'\n'
print R'\n'

````
### Python 字符串格式化

%s是字符串占位符，%d是数字占位符。  注意字符串和值之间的连接符%

````
print "My name is %s and weight is %d kg!" % ('Zara', 21) 

````

### Python三引号

python三引号允许一个字符串跨多行，字符串中可以包含换行符、制表符以及其他特殊字符。三引号的语法是一对连续的单引号或者双引号。

### python的字符串内建函数





## 列表

列表中值的切割也可以用到变量 [头下标:尾下标] ，就可以截取相应的列表，从左到右索引默认 0 开始，从右到左索引默认 -1 开始，下标可以为空表示取到头或尾。

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

list = ['runoob', 786, 2.23, 'john', 70.2]
tinylist = [123, 'john']

print list  # 输出完整列表
print list[0]  # 输出列表的第一个元素
print list[1:3]  # 输出第二个至第三个元素
print list[2:]  # 输出从第三个开始至列表末尾的所有元素
print tinylist * 2  # 输出列表两次
print list + tinylist  # 打印组合的列表

list.append(5)  # 添加元素
list.append('test')

del list[4]  # 删除元素

````


### list更新列表

添加元素：

list.append(5)

删除元素:

del list[4] 


### 列表脚本操作符

````
print '列表长度：',len(list)  # 长度
print '列表组合：',list +tinylist  # 组合
print '列表重复：', list * 3
print '某个元素是否在列表中：',5 in list
for x in list :
    print '循环元素：',x

````

## 列表的函数


````
lenList = [4, 6, 7, 8, 9]
# 获取列表的长度
print len(lenList)
# 获取列表元素的最大值
print max(lenList)
# 将元祖转化为列表
tup = (5, 7, 9, 0)
print tup
tupList = list(tup)
print tupList
# 添加元素
tupList.append(5)
# 统计某个元素在列表中出现的次数
n = tupList.count(5)
print '元素在列表中出现的次数：', n

# 一次性扩展多个值
aList = [123, 'xyz', 'zara', 'abc', 123]
bList = [2009, 'manni']
aList.extend(bList)

# 列出某个元素在列表中的位置
bList.index(123)

# 在某个位置插入某个元素
list.insert(3, 'dddd')

# 移除列表中的一个元素（默认最后一个元素），并且返回该元素的值
bList.pop(-1)

# 移除列表中某个值的第一个匹配项
bList.remove(5)

'''
cmp -- 可选参数, 如果指定了该参数会使用该参数的方法进行排序。
key -- 主要是用来进行比较的元素，只有一个参数，具体的函数的参数就是取自于可迭代对象中，指定可迭代对象中的一个元素来进行排序。
reverse -- 排序规则，reverse = True 降序， reverse = False 升序（默认）。
'''
# 排序
list.sort(cmp=None, key=None, reverse=False)

````



## Python元组

元组是另一个数据类型，类似于List（列表）。**元组用"()"标识**。内部元素用逗号隔开。但是元组不**能二次赋值**，**相当于只读列表**。

其它和list一样

## Python 字典

字典的每个键值 key=>value 对用冒号 : 分割，每个键值对之间用逗号 , 分割，整个字典包括在花括号 {} 中 ,格式如下所示：



````python
#!/usr/bin/python

dict ={}
dict['one'] = "this is one"
dict[2] = "this is two"

tinydict ={'name':'john','code':'123','dept':'sales'}

print dict['one']
print dict[2]
print tinydict
print dict
print tinydict.viewkeys()
print tinydict.viewvalues()

````

字典值可以没有限制地取任何python对象，既可以是标准的对象，也可以是用户定义的，但键不行。键必须不可变，所以可以用数字，字符串或元组充当，所以用列表就不行。


````
tinydict = {'name': 'john', 'code': '123', 'dept': 'sales'}
tinySet = {'a', 6, 7}
tinyList = ['f', 7, 9]
tinyTup = (6, 8, 9)
print 'tinydict type:', type(tinydict)
print 'tinySet type:', type(tinySet)
print 'tinyList type:', type(tinyList)
print 'tinyTup type:', type(tinyTup) # 类型

print "after del:", dict
dict.clear() # 删除字典的所有元素
print "after clear:", dict

# 直接引用
tinydict1 = tinydict

# 浅copy 深拷贝父对象（一级目录），子对象（二级目录）不拷贝，还是引用
tinydict2 = tinydict.copy()
tinydict['name'] = '张三'
del tinydict['name']

print 'tinydict1：', tinydict1
print 'tinydict2：', tinydict2

````

不存在则返回默认值

````
# 如果键在字典dict里返回true，否则返回false
dict.has_key(key)

# 以列表返回可遍历的(键, 值) 元组数组
dict.items()

# 以列表返回一个字典所有的键
dict.keys()

# 和get()类似, 但如果键不存在于字典中，将会添加键并将值设为default
dict.setdefault(key,"")

# 把字典dict2的键/值对更新到dict里
dict.update(dict2)
# 以列表返回字典中的所有值
dict.values()
# 删除字典给定键 key 所对应的值，返回值为被删除的值。key值必须给出。 否则，返回default值。
pop(key[,default])

# 随机返回并删除字典中的一对键和值。
popitem()

````

## 日期和时间

````

#!/usr/bin/python
# -*- coding: UTF-8 -*-

# 引入time模块
import time

# 时间戳
ticks = time.time()
print 'ticks:', ticks

# 获取当前时间,元组形式
localTime = time.localtime(time.time())
print 'localTime:', localTime

# 获取格式化日期
# 格式化成2016-03-20 11:45:39形式
print time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())

# 格式化成Sat Mar 28 22:24:24 2016形式
print time.strftime("%a %b %d %H:%M:%S %Y", time.localtime())

# 将格式字符串转换为时间戳
a = "Sat Mar 28 22:24:24 2016"
print time.mktime(time.strptime(a, "%a %b %d %H:%M:%S %Y"))

# 输出日历
cal = calendar.month(2016, 1)
print "以下输出2016年1月份的日历:"
print cal

# 用以浮点数计算的秒数返回当前的CPU时间。用来衡量不同程序的耗时，比time.time()更有用。
time.clock( )

i = datetime.datetime.now()
print ("当前的日期和时间是 %s" % i)
print ("ISO格式的日期和时间是 %s" % i.isoformat())
print ("当前的年份是 %s" % i.year)
print ("当前的月份是 %s" % i.month)
print ("当前的日期是  %s" % i.day)
print ("dd/mm/yyyy 格式是  %s/%s/%s" % (i.day, i.month, i.year))
print ("当前小时是 %s" % i.hour)
print ("当前分钟是 %s" % i.minute)
print ("当前秒是  %s" % i.second)

````

# 运算符

## Python算术运算符

````
+ - * / % 之外 

````

** 幂-返回x的y次幂  4**2 =16

//	取整除 - 返回商的整数部分   9//2 =4

Python2.x 里，整数除整数，只能得出整数。如果要得到小数部分，把其中一个数改成浮点数即可。

````
a = 10
b = 20
c = a + b
print "1---c的值为：", c

c = b - a
print "2---c的值为：",c

c = a * b
print "3---c的值为：",c

c = b / a
print '4----c的值为：',c

c = b ** a
print '5----c的值为：',c

c = b // a
print '6----c的值为：',c

c = b % a
print '7----c的值为：',c


````
## Python比较运算符

==  !=  <>  >  <  >=  <=

## Python赋值运算符

= += -= *= 等等

## Python位运算符

按位运算符是把数字看作二进制来进行计算的

& 与运算符：参与运算的两个值,如果两个相应位都为1,则该位的结果为1,否则为0

| 或运算符：只要对应的二个二进位有一个为1时，结果位就为1。

^ 异或运算符：当两对应的二进位相异时，结果为1; 

~ 取反运算符：对数据的每个二进制位取反,即把1变为0,把0变为1 。~x 类似于 -x-1

<< 左移动运算符：运算数的各二进位全部左移若干位，由 << 右边的数字指定了移动的位数，高位丢弃，低位补0。

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

a = 60 # 60 = 0011 1100
b = 13 # 13 = 0000 1101
c = 0

c = a & b
print 'c===',c

c = a | b
print 'c====',c

c = a ^ b
print 'c====',c

c = - a
print 'c====',c

c = ~a
print "4 - c 的值为：", c # ~x 类似于 -x-1

c = a << 2
print "5 - c 的值为：", c

c = a >> 2
print "6 - c 的值为：", c

````

## Python逻辑运算符

and or not



# python条件语句

Python程序语言指定任何非0和非空（null）值为true，0 或者 null为false。

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

flag = False
name = 'luren'
if name == 'python':
    flag = True
    print 'welcome python'
elif name == 'luren':
    print 'welcome luren'
else:
    print name

````

# 循环语句

包括 while for 嵌套循环

## while循环 break和continue

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

i = 1
while i < 10:
    i += 1
    print i
else:
    print "end！"

print "----------"

i = 2
while i < 20:
    i += 1
    if i % 2 == 0:
        break # continue
    print "i==",i
else:
    print 'end'

````
break只能跳出一层循环，如果你的循环是嵌套循环，那么你需要按照你嵌套的层次，逐步使用break来跳出。

continue 语句用来告诉Python跳过当前循环的剩余语句，然后继续进行下一轮循环。

## for 循环语句

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

# 遍历字符串
for letter in "python":
    print '当前字母：',letter


# 遍历list
fruits = ['apple', 'banana', 'peer']
for fruit in fruits:
    print '当前水果为：', fruit

# 通过下标遍历list
for index in range(len(fruits)):
    print '通过下标获取的水果的名字为：', fruits[index]

````

## 循环嵌套

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

i = 2
while i < 100:
    j = 2
    while j <= (i / j):
        if not (i % j): break
        j += 1
    if j > (i / j): print i, "是素数"
    i += 1
print 'good bye'

````
## pass 语句

Python pass是空语句，是为了保持程序结构的完整性。

pass 不做任何事情，一般用做占位语句。

````
#!/usr/bin/python
# -*- coding: UTF-8 -*- 

# 输出 Python 的每个字母
for letter in 'Python':
   if letter == 'h':
      pass
      print '这是 pass 块'
   print '当前字母 :', letter

print "Good bye!"

````

执行结果

````
当前字母 : P
当前字母 : y
当前字母 : t
这是 pass 块
当前字母 : h
当前字母 : o
当前字母 : n
Good bye!

````
## python函数

### 定义一个函数

你可以定义一个由自己想要功能的函数，以下是简单的规则：

````
函数代码块以 def 关键词开头，后接函数标识符名称和圆括号()。

任何传入参数和自变量必须放在圆括号中间。圆括号之间可以用于定义参数。

函数的第一行语句可以选择性地使用文档字符串—用于存放函数说明。

函数内容以冒号起始，并且缩进。

return [表达式] 结束函数，选择性地返回一个值给调用方。不带表达式的return相当于返回 None。

````

### 在python中，类型属于对象，变量是没有对象的。

````
a=[1,2,3]

a="Runoob"

````
以上代码中，[1,2,3] 是 List 类型，"Runoob" 是 String 类型，而变量 a 是没有类型，她仅仅是一个对象的引用（一个指针），可以是 List 类型对象，也可以指向 String 类型对象。

### 可更改(mutable)与不可更改(immutable)对象

**不可变类型**：变量赋值 a=5 后再赋值 a=10，这里实际是新生成一个 int 值对象 10，再让 a 指向它，而 5 被丢弃，不是改变a的值，相当于新生成了a。

**可变类型**：变量赋值 la=[1,2,3,4] 后再赋值 la[2]=5 则是将 list la 的第三个元素值更改，本身la没有动，只是其内部的一部分值被修改了。

#### python 函数的参数传递：

**不可变类型**：类似 c++ 的值传递，如 整数、字符串、元组。如fun（a），传递的只是a的值，没有影响a对象本身。比如在 fun（a）内部修改 a 的值，只是修改另一个复制的对象，不会影响 a 本身。

**可变类型**：类似 c++ 的引用传递，如 列表，字典。如 fun（la），则是将 la 真正的传过去，修改后fun外部的la也会受影响

````
# 传递不可变对象
def changeInt(a):
    b = a
    a = 10
    print a
    print b
    return


b = 120
changeInt(b)
print b


# 传可变对象实例
def changeMe(myList):
    '''修改传入的列表：'''
    myList.append([1, 4, 6, 8])
    return


myList = [10, 40, 60]
changeMe(myList)
print 'myList', myList

````
### 参数

#### 必备参数

必备参数须以正确的顺序传入函数。调用时的数量必须和声明时的一样。无需指定参数的名字

#### 关键字参数

关键字参数和函数调用关系紧密，函数调用使用关键字参数来确定传入的参数值。

使用关键字参数允许函数调用时参数的顺序与声明时不一致，因为 Python 解释器能够用参数名匹配参数值。

````
def printinfo( name, age ):
   "打印任何传入的字符串"
   print "Name: ", name;
   print "Age ", age;
   return;
 
#调用printinfo函数
printinfo( age=50, name="miki" );

````

#### 缺省参数

调用函数时，缺省参数的值如果没有传入，则被认为是默认值。

````
#可写函数说明
def printinfo( name, age = 35 ):
   "打印任何传入的字符串"
   print "Name: ", name;
   print "Age ", age;
   return;
 
#调用printinfo函数
printinfo( age=50, name="miki" );
printinfo( name="miki" );

````

#### 不定长参数

你可能需要一个函数能处理比当初声明时更多的参数。这些参数叫做不定长参数，和上述2种参数不同，声明时不会命名

加了星号（*）的变量名会存放所有未命名的变量参数

````
def printinfo(arg1, *vartuple):
    "打印任何传入的参数"
    print "输出: "
    print arg1
    for var in vartuple:
        print 'var:', var
    return


# 调用printinfo 函数
printinfo(10)
printinfo(70, 60, 50)

````

### 匿名函数

python 使用 lambda 来创建匿名函数。

1.lambda只是一个表达式，函数体比def简单很多。

2.lambda的主体是一个表达式，而不是一个代码块。仅仅能在lambda表达式中封装有限的逻辑进去。

3.lambda函数拥有自己的命名空间，且不能访问自有参数列表之外或全局命名空间里的参数。

4.虽然lambda函数看起来只能写一行，却不等同于C或C++的内联函数，后者的目的是调用小函数时不占用栈内存从而增加运行效率。

语法：

lambda [arg1 [,arg2,.....argn]]:expression

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
# 可写函数说明
sum = lambda arg1, arg2: arg1 + arg2;
 
# 调用sum函数
print "相加后的值为 : ", sum( 10, 20 )
print "相加后的值为 : ", sum( 20, 20 )

````

## 全局变量和局部变量

局部变量只能在其被声明的函数内部访问，而全局变量可以在整个程序范围内访问。调用函数时，所有在函数内声明的变量名称都将被加入到作用域中。

````
#!/usr/bin/python
# -*-coding: UTF-8-*-

total = 0  # 这是一个全局变量


# 可写函数说明
def sum(arg1, arg2):
    # 返回2个参数的和."
    total = arg1 + arg2  # total在这里是局部变量.
    print "函数内是局部变量 : ", total
    return total;


# 调用sum函数
sum(10, 20)
# total = sum(30, 50)
print "函数外是全局变量 : ", total

````

# python 模块

Python 模块(Module)，是一个 Python 文件，以 .py 结尾，包含了 Python 对象定义和Python语句

import语句 引入模块

from…import语句 Python 的 from 语句让你从模块中导入一个指定的部分到当前命名空间中

## 搜索路径

当你导入一个模块，Python 解析器对模块位置的搜索顺序是：

1. 当前目录
2. 如果不在当前目录，Python 则搜索在 shell 变量 PYTHONPATH 下的每个目录。
3. 如果都找不到，Python会察看默认路径。UNIX下，默认路径一般为/usr/local/lib/python/。

模块搜索路径存储在 system 模块的 sys.path 变量中。变量里包含当前目录，PYTHONPATH和由安装过程决定的默认目录。

## 命名空间和作用域

变量是拥有匹配对象的名字（标识符）。命名空间是一个包含了变量名称们（键）和它们各自相应的对象们（值）的字典。

如果一个局部变量和一个全局变量重名，则局部变量会覆盖全局变量。

Python 会智能地猜测一个变量是局部的还是全局的，它假设任何在函数内赋值的变量都是局部的。因此，如果要给函数内的全局变量赋值，必须使用 global 语句。

global VarName 的表达式会告诉 Python， VarName 是一个全局变量，这样 Python 就不会在局部命名空间里寻找这个变量了。

````
#!/usr/bin/python
# -*-coding: UTF-8 -*-


money = 3000


def AddMoney():
    # 全局变量的使用
    global money
    money = money + 1


print 'money:', money
AddMoney()
print 'money:', money

````

### dir()函数

dir() 函数一个排好序的字符串列表，内容是一个模块里定义过的名字。

````
#!/usr/bin/python
# -*-coding: UTF-8 -*-
import math
from math import floor


# dir函数
content = dir(math)
print 'content:', content

print floor(5.8)


````

### globals() 和 locals() 函数

根据调用地方的不同，globals() 和 locals() 函数可被用来返回全局和局部命名空间里的名字。

如果在函数内部调用 locals()，返回的是所有能在该函数里访问的命名。

如果在函数内部调用 globals()，返回的是所有在该函数里能访问的全局名字。


### Python中的包

简单来说，包就是文件夹，但该文件夹下必须存在 __init__.py 文件, 该文件的内容可以为空。__init__.py 用于标识当前文件夹是一个包。

## Python 文件I/O

raw_input函数

````
str = raw_input("按下 enter键退出，其他任意键显示。。。\n")
print 'str:', str

````

input函数

input([prompt]) 函数和 raw_input([prompt]) 函数基本类似，但是 input 可以接收一个Python表达式作为输入，并将运算结果返回。

````
str = input("请输入：")  # 输入[x*5 for x in range(2,10,2)]
print "你输入的内容是: ", str

````

# 文件IO

## open 函数

````
file object = open(file_name [, access_mode][, buffering])

````
各个参数的细节如下：

file_name：file_name变量是一个包含了你要访问的文件名称的字符串值。

access_mode：access_mode决定了打开文件的模式：只读，写入，追加等。所有可取值见如下的完全列表。这个参数是非强制的，默认文件访问模式为只读(r)。

buffering:如果buffering的值被设为0，就不会有寄存。如果buffering的值取1，访问文件时会寄存行。如果将buffering的值设为大于1的整数，表明了这就是的寄存区的缓冲大小。如果取负值，寄存区的缓冲大小则为系统默认。


![python文件访问权限](/images/python/2.7/python文件访问权限.png)

## 文件定位

tell() 该方法告诉你文件内的当前位置, 换句话说，下一次的读写会发生在文件开头这么多字节之后。

seek（offset [,from]）方法改变当前文件的位置。Offset变量表示要移动的字节数。From变量指定开始移动字节的参考位置。

## 重命名和删除文件

Python的os模块提供了帮你执行文件处理操作的方法，比如重命名和删除文件。

````
os.rename(current_file_name, new_file_name)

````

## 文件和目录操作的例子

````
#!/usr/bin/python
# -*-coding: UTF-8 -*-

import os

fo = open("foo.txt", "w+")
print "文件名字：", fo.name
print "是否已经关闭1：", fo.closed
print "打开文件的访问模式：", fo.mode
# 文件写入内容 write()方法不会在字符串的结尾添加换行符('\n')：
fo.write("i'm writing something.....\n 第二行文本  \n \n 第四行文本")
# 关闭打开的文件
fo.close()
print "是否已经关闭2：", fo.closed

fo1 = open("foo.txt", "r+")
# read（）方法从一个打开的文件中读取一个字符串。需要重点注意的是，Python字符串可以是二进制数据，而不是仅仅是文字。  fileObject.read([count])
str = fo1.read(10)
print "str:", str

print "分隔符1：", "--------------------------------"

# 打开一个文件
fo2 = open("foo.txt", "r+")
str = fo2.read(10)
print "读取的字符串是 : ", str

# 查找当前位置
position = fo2.tell()
print "当前文件位置 : ", position

# 把指针再次重新定位到文件开头
position = fo2.seek(0, 0)
str = fo2.read(10)
print "重新读取字符串 : ", str
# 关闭打开的文件
fo2.close()

print "分隔符2：", "--------------------------------"

# 重命名和删除文件
os.rename("foo.txt", "fool2.txt")

# 删除一个已经存在的文件
# os.remove("fool2.txt")

# 创建目录
# os.mkdir("test")

# 改变当前目录
# 将当前目录改为"/home/newdir"
# os.chdir("/home/zhuningning/PycharmProjects/pythonBase/base/test")

# 显示当前目录
print os.getcwd()

# 删除目录
# os.rmdir("/home/zhuningning/PycharmProjects/pythonBase/base/test")

print "分隔符3：", "--------------------------------"

# 每行读取文件
fo = open("fool2.txt", "rw+")
print "文件名为: ", fo.name

for index in range(4):
    line = fo.next()
    print "第 %d 行 - %s" % (index+1, line)

# 关闭文件
fo.close()

````

# 异常处理

python提供了两个非常重要的功能来处理python程序在运行中出现的异常和错误：异常处理 和 断言(Assertions)


捕捉异常可以使用try/except语句。

try/except语句用来检测try语句块中的错误，从而让except语句捕获异常信息并处理。

语法如下：

````
try:
<语句>        #运行别的代码
except <名字>：
<语句>        #如果在try部份引发了'name'异常
except <名字>，<数据>:
<语句>        #如果引发了'name'异常，获得附加的数据
else:
<语句>        #如果没有异常发生

````

````
try的工作原理是，当开始一个try语句后，python就在当前程序的上下文中作标记，这样当异常出现时就可以回到这里，try子句先执行，接下来会发生什么依赖于执行时是否出现异常。

如果当try后的语句执行时发生异常，python就跳回到try并执行第一个匹配该异常的except子句，异常处理完毕，控制流就通过整个try语句（除非在处理异常时又引发新的异常）。

如果在try后的语句里发生了异常，却没有匹配的except子句，异常将被递交到上层的try，或者到程序的最上层（这样将结束程序，并打印缺省的出错信息）。

如果在try子句执行时没有发生异常，python将执行else语句后的语句（如果有else的话），然后控制流通过整个try语句。

````


异常的demo

````
#!/usr/bin/python
# -*- coding: UTF-8 -*-

try:
    fh = open("testfile.txt", "w")
    fh.write("write someting...")
except IOError:
    print "error: 没有找到文件或者读取文件失败"
else:
    print "内容写入成功！"
    fh.close()

# try finally语句

try:
    fh = open("testfile", "w")
    fh.write("这是一个测试文件1，用于测试异常!")
finally:
    print "Error: 没有找到文件或读取文件失败"


# 用户自定义异常

class Networkerror(RuntimeError):
    def __init__(self, arg):
        self.args = arg


try:
    raise Networkerror("Bad hostname")
except Networkerror, e:
    print e.args

# 使用except而带多种异常类型

try:
    print "正常执行！"
except(IOError, BaseException, IndentationError):
    print "发生异常！"
else:
    print "如果没有异常执行这块代码"

````


**print语句默认的会在后面加上 换行  加了逗号之后 换行 就变成了 空格**  






















