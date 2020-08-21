---
title: 深入理解java虚拟机-虚拟机类加载机制
copyright: true
date: 2018-11-02 10:22:04
tags: 虚拟机类加载机制
categories: JVM

---

虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制。

# 类加载的时机

类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）和卸载（Unloading）7个阶段。其中验证、准备、解析3个部分统称为连接（Linking）

![类的生命周期](/images/jvm/类的生命周期.png)

以下情况才会触发初始化：

1. 使用new关键字实例化对象的时候、读取或设置一个类的静态字段；
2. 使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。
3. 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含main（）方法的那个类），虚拟机会先初始化这个主类
5. 动态语言支持的，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化

## 被动引用

通过子类引用父类的静态字段，不会导致子类初始化；

通过数组定义来引用类，不会触发此类的初始化；

常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。

## 接口的加载过程的不同

当一个类在初始化时，要求其父类全部都已经初始化过了，但是一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化

# 类加载的过程

## 加载

加载阶段需要完成三件事情：

1. 通过一个类的全限定名来获取定义此类的二进制字节流;
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构;
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口

一个非数组类的加载阶段（准确地说，是加载阶段中获取类的二进制字节流的动作）是开发人员可控性最强的，因为加载阶段既可以使用系统提供的引导类加载器来完成，也可以由用户自定义的类加载器去完成，

对于数组类而言，情况就有所不同，数组类本身不通过类加载器创建，它是由Java虚拟机直接创建的。但数组类与类加载器仍然有很密切的关系，因为数组类的元素类型最终需要类加载器组创建。

数组类的可见性与它的组件类型的可见性一致，如果组件类型不是引用类型，那数组类的可见性将默认为public。

## 验证

验证是连接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

1. 文件格式验证
2. 元数据验证
3. 字节码验证
4. 符号引用验证

## 准备 

准备阶段是正式为类变量分配内存并设置变量初始值的阶段。这些变量所使用的内存都将在方法区中进行分配。

这个阶段中有两个容易产生混淆的概念需要强调一下，首先，这时候进行内存分配的仅包括类变量（被static修饰的变量），而不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在Java堆中。

![基本数据类型的零值](/images/jvm/基本数据类型的零值.png)

## 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。

直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。

## 初始化

类初始化阶段是类加载过程的最后一步，前面的类加载过程中，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码（或者说是字节码）。

在准备阶段，变量已经赋过一次系统要求的初始值，而在初始化阶段，则根据程序员通过程序制定的主观计划去初始化类变量和其他资源，或者可以从另外一个角度来表达：初始化阶段是执行类构造器＜clinit＞（）方法的过程。

＜clinit＞（）方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量


# 类加载器

通过一个类的全限定名来获取描述此类的二进制字节流”这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。实现这个动作的代码模块称为“类加载器”。

## 类与类加载器

类加载器虽然只用于实现类的加载动作，但它在Java程序中起到的作用却远远不限于类加载阶段。

对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，
每一个类加载器，都拥有一个独立的类名称空间。这句话可以表达得更通俗一些：比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，
否则，即使这两个类来源于同一个Class文件，被同一个虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。

````
package cn.chen.exercise.chapter7;

import java.io.IOException;
import java.io.InputStream;

/**
 * 不同的类加载器对instanceof关键字运算的结果的影响
 *
 * @author Chen WeiJie
 * @date 2018-11-05 22:49:24
 **/
public class ClassLoaderTest {

    public static void main(String[]args)throws Exception{
        ClassLoader myLoader=new ClassLoader(){
            @Override
            public Class<?>loadClass(String name)throws ClassNotFoundException{
                try{
                    String fileName=name.substring(name.lastIndexOf(".")+1)+".class";
                    InputStream is=getClass().getResourceAsStream(fileName);
                    if(is==null){
                        return super.loadClass(name);
                    }
                    byte[]b=new byte[is.available()];
                    is.read(b);
                    return defineClass(name,b,0,b.length);
                }catch(IOException e){
                    throw new ClassNotFoundException(name);
                }
            }
        };
        Object obj=myLoader.loadClass("cn.chen.exercise.chapter7.ClassLoaderTest").newInstance();
        System.out.println(obj.getClass());
        System.out.println(obj instanceof cn.chen.exercise.chapter7.ClassLoaderTest);
    }
    
}

````
它可以加载与自己在同一路径下的Class文件。我们使用这个类加载器去加载了一个名为“org.fenixsoft.classloading.ClassLoaderTest”的类，并实例化了这个类的对象。两行输出结
果中，从第一句可以看出，这个对象确实是类org.fenixsoft.classloading.ClassLoaderTest实例化出来的对象，但从第二句可以发现，这个对象与类org.fenixsoft.classloading.ClassLoaderTest做
所属类型检查的时候却返回了false，这是因为虚拟机中存在了两个ClassLoaderTest类，一个是由系统应用程序类加载器加载的，另外一个是由我们自定义的类加载器加载的，虽然都来
自同一个Class文件，但依然是两个独立的类，做对象所属类型检查时结果自然为false

## 双亲委派模型

## 从Java虚拟机的角度来讲，只存在两种不同的类加载器：

一种是启动类加载器（Bootstrap  ClassLoader），这个类加载器使用C++语言实现  ，是虚拟机自身的一部分；另

一种就是所有其他的类加载器，这些类加载器都由Java语言实现，独立于虚拟机外部，并且全都继承自抽象类java.lang.ClassLoader

## 从Java开发人员的角度来看，类加载器还可以划分得更细致一些，绝大部分Java程序都会使用到以下3种系统提供的类加载器。

- 启动类加载器（Bootstrap ClassLoader）：

这个类将器负责将存放在＜JAVA_HOME＞\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机
识别的（仅按照文件名识别，如rt.jar，名字不符合的类库即使放在lib目录中也不会被加载）类库加载到虚拟机内存中。启动类加载器无法被Java程序直接引用，用户在编写自定义类加
载器时，如果需要把加载请求委派给引导类加载器，那直接使用null代替即可。

````
/**
Returns the class loader for the class.Some implementations may use null to represent the bootstrap class loader.This method will return null in such
implementations if this class was loaded by the bootstrap class loader.
*/
public ClassLoader getClassLoader（）{
ClassLoader cl=getClassLoader0（）；
if（cl==null）
return null；
SecurityManager sm=System.getSecurityManager（）；
if（sm！=null）{
ClassLoader ccl=ClassLoader.getCallerClassLoader（）；
if（ccl！=null＆＆ccl！=cl＆＆！cl.isAncestor（ccl））{
sm.checkPermission（SecurityConstants.GET_CLASSLOADER_PERMISSION）；
}
}
return cl；
}

````

## 扩展类加载器（Extension  ClassLoader）

这个加载器由sun.misc.Launcher $ExtClassLoader实现，它负责加载＜JAVA_HOME＞\lib\ext目录中的，或者被java.ext.dirs系
统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器。

## 应用程序类加载器（Application ClassLoader）

这个类加载器由sun.misc.Launcher $App-ClassLoader实现。由于这个类加载器是ClassLoader中的getSystemClassLoader（）方法的返回
值，所以一般也称它为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

![类加载器的双亲委派模型](/images/jvm/类加载器的双亲委派模型.png)


上图展示的类加载器之间的这种层次关系，称为类加载器的双亲委派模型（Parents Delegation Model）。双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当
有自己的父类加载器。这里类加载器之间的父子关系一般不会以继承（Inheritance）的关系来实现，而是都使用组合（Composition）关系来复用父加载器的代码。


双亲委派模型不是一个强制性的约束模型，而是java开发者推荐的一种类加载器实现方式。

### 双亲委派模型的工作过程是

如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是
如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

### 双亲委派模型的好处

使用双亲委派模型来组织类加载器之间的关系，有一个显而易见的好处就是Java类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存放在
rt.jar之中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。
相反，如果没有使用双亲委派模型，由各个类加载器自行去加载的话，

如果用户自己编写了一个称为java.lang.Object的类，并放在程序的ClassPath中，那系统中将会出现多个不同的Object
类，Java类型体系中最基础的行为也就无法保证，应用程序也将会变得一片混乱。

如果读者有兴趣的话，可以尝试去编写一个与rt.jar类库中已有类重名的Java类，将会发现可以正常编译，但永远无法被加载运行

### 双亲委派模型的实现逻辑

先检查是否已经被加载过，若没有加载则调用父加载器的loadClass（）方法，若父加载器为空则默认使用启动类加载器作为父加载器。如果父类加载失败，抛出

ClassNotFoundException异常后，再调用自己的findClass（）方法进行加载。

- 双亲委派模型的实现

````
protected synchronized Class＜?＞loadClass（String name,boolean resolve）throws ClassNotFoundException
{
//首先，检查请求的类是否已经被加载过了
Class c=findLoadedClass（name）；
if（c==null）{
try{
if（parent！=null）{
c=parent.loadClass（name,false）；
}else{
c=findBootstrapClassOrNull（name）；
}
}catch（ClassNotFoundException e）{
//如果父类加载器抛出ClassNotFoundException
//说明父类加载器无法完成加载请求
}
if（c==null）{
//在父类加载器无法加载的时候
//再调用本身的findClass方法来进行类加载
c=findClass（name）；
}
}
if（resolve）{
resolveClass（c）；
}
return c；
}


````






































