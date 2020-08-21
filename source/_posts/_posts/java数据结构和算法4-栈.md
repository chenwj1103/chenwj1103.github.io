---
title: java数据结构和算法4-栈
copyright: true
date: 2019-03-01 01:09:50
tags: 栈
categories: 数据结构

---

栈和队列被用作程序员的工具，它们作为构思算法的辅助工具，而不是完全的数据存储工具。这些数据结构的生命周期比数据库类型的结构要短得多，在程序执行期间它们才被创建，通常用它们去执行某项特殊的业务，执行完成之后，它们就被销毁。这里的它们就是——栈和队列。


# 栈的基本概念

栈（英语：stack）又称为堆栈或堆叠，栈作为一种数据结构，是一种只能在一端进行插入和删除操作的特殊线性表。

它按照先进后出的原则存储数据，先进入的数据被压入栈底，最后的数据在栈顶，需要读数据的时候从栈顶开始弹出数据（最后一个数据被第一个读出来）。栈具有记忆作用，对栈的插入与删除操作中，不需要改变栈底指针。

栈是允许在同一端进行插入和删除操作的特殊线性表。允许进行插入和删除操作的一端称为栈顶(top)，另一端为栈底(bottom)；栈底固定，而栈顶浮动；栈中元素个数为零时称为空栈。插入一般称为进栈（PUSH），删除则称为退栈（POP）。

由于堆叠数据结构只允许在一端进行操作，因而按照后进先出（LIFO, Last In First Out）的原理运作。栈也称为后进先出表

![栈](/images/datastructure/栈.png)


## 代码实现简单的栈

````
package com.chen.dataStructure.stack;

/**
 * @author :  chen weijie
 * @Date: 2019-03-05 11:47 PM
 */
public class MyStack {


    /**
     * 栈的实际元素
     */
    private int[] array;

    /**
     * 当前元素的个数
     */
    private int top;

    /**
     * 栈的容量
     */
    private int maxSize;


    public MyStack(int size) {
        this.maxSize = size;
        array = new int[size];
        top = -1;
    }


    /**
     * 插入元素
     *
     * @param value
     */
    public void push(int value) {
        if (top < maxSize - 1) {
            array[++top] = value;
        }
    }


    /**
     * 弹出元素
     *
     * @return
     */
    public int pop() {
        return array[top--];
    }


    /**
     * 访问栈顶元素
     *
     * @return
     */
    public int peep() {
        return array[top];
    }


    /**
     * 栈是否是空
     *
     * @return
     */
    public boolean isEmpty() {
        return (top == -1);
    }


    /**
     * @return
     */
    public boolean isFull() {
        return (maxSize - 1) == top;
    }


    public static void main(String[] args) {

        MyStack myStack = new MyStack(4);

        myStack.push(10);
        myStack.push(20);
        myStack.push(1);
        myStack.push(-1);

        System.out.println("isFull:" + myStack.isFull());
        System.out.println("peek:" + myStack.peep());

        while (!myStack.isEmpty()) {
            System.out.println("弹出：" + myStack.pop());
            System.out.println("栈内元素个数：" + myStack.top);
        }
    }
}

````
这个栈是用数组实现的，内部定义了一个数组，一个表示最大容量的值以及一个指向栈顶元素的top变量。构造方法根据参数规定的容量创建一个新栈，push()方法是向栈中压入元素，指向栈顶的变量top加一，使它指向原顶端数据项上面的一个位置，并在这个位置上存储一个数据。

pop()方法返回top变量指向的元素，然后将top变量减一，便移除了数据项。要知道 top 变量指向的始终是栈顶的元素。


## 产生的问题

①、上面栈的实现初始化容量之后，后面是不能进行扩容的（虽然栈不是用来存储大量数据的），如果说后期数据量超过初始容量之后怎么办？（自动扩容）

②、我们是用数组实现栈，在定义数组类型的时候，也就规定了存储在栈中的数据类型，那么同一个栈能不能存储不同类型的数据呢？（声明为Object）

③、栈需要初始化容量，而且数组实现的栈元素都是连续存储的，那么能不能不初始化容量呢？（改为由链表实现）

# 增强功能版栈

这个模拟的栈在JDK源码中

````
package com.chen.dataStructure.stack;

import java.util.Arrays;

/**
 * @author :  chen weijie
 * @Date: 2019-03-06 12:31 AM
 */
public class ArrayStack {


    /**
     * object类型的数组可以存储任意类型
     */
    private Object[] elementData;

    /**
     * 指向栈顶的指针
     */
    private int top;

    /**
     * 栈的容量
     */
    private int size;


    public ArrayStack() {

        this.elementData = new Object[10];
        this.top = -1;
        this.size = 10;
    }

    public ArrayStack(int initialCapacity) {

        if (initialCapacity < 0) {
            throw new IllegalArgumentException("栈的初始容量不得小于0：" + initialCapacity);
        }
        this.elementData = new Object[initialCapacity];
        this.top = -1;
        this.size = initialCapacity;
    }


    public Object push(Object value) {
        isGrow(top + 1);
        elementData[++top] = value;
        return value;
    }


    public void remove(int top) {

        //栈顶元素置为null
        elementData[top] = null;
        this.top--;
    }


    public boolean isEmpty() {
        return top == -1;
    }


    public Object peek() {

        if (top == -1) {
            return null;
        }
        return elementData[top];

    }


    public Object pop() {
        Object object = peek();
        remove(top);
        return object;
    }


    public boolean isGrow(int minCapacity) {

        int oldCapacity = size;
        //如果当前元素压入栈之后总容量大于前面定义的容量，则需要扩容
        if (oldCapacity <= minCapacity) {
            //定义扩大之后栈的总容量
            int newCapacity = 0;
            if ((oldCapacity << 1) - Integer.MAX_VALUE > 0) {
                newCapacity = Integer.MAX_VALUE;
            } else {
                newCapacity = (oldCapacity << 1);//左移一位，相当于*2
            }
            this.size = newCapacity;
            int[] newArray = new int[size];
            elementData = Arrays.copyOf(elementData, size);
            return true;
        } else {
            return false;
        }
    }

    public static void main(String[] args) {

        ArrayStack stack = new ArrayStack(3);
        stack.push(1);
        //System.out.println(stack.peek());
        stack.push(2);
        stack.push(3);
        stack.push("abc");
        System.out.println(stack.peek());
        stack.pop();
        stack.pop();
        stack.pop();
        System.out.println(stack.peek());
    }
}



````

# 利用栈反转输出字符串

````

public class RevertString {

    public static void main(String[] args) {


        String str = "how are you ";
        char[] charArray = str.toCharArray();
        ArrayStack myStack = new ArrayStack();
        for (char c : charArray) {
            myStack.push(c);
        }

        while (!myStack.isEmpty()) {
            System.out.print(myStack.pop());
        }

    }

}

````


# 利用栈判断分隔符是否匹配　　

````
package com.chen.dataStructure.stack;

/**
 * 利用栈判断分隔符是否匹配
 * 比如：<abc[123]abc>这是符号相匹配的，如果是 <abc[123>abc] 那就是不匹配的。
 * <p>
 * 　对于 12<a[b{c}]>，我们分析在栈中的数据：遇到匹配正确的就消除
 *
 * @author :  chen weijie
 * @Date: 2019-03-06 1:04 AM
 */
public class TestMatch {


    //遇到左边分隔符了就push进栈，遇到右边分隔符了就pop出栈，看出栈的分隔符是否和这个有分隔符匹配


    public static void main(String[] args) {

        ArrayStack arrayStack = new ArrayStack(3);

        String str = "12<a[b{c}]>";

        char[] chars = str.toCharArray();

        for (char c : chars) {

            switch (c) {

                case '{':
                case '[':
                case '<':
                    arrayStack.push(c);
                    break;
                case '}':
                case ']':
                case '>':
                    if (!arrayStack.isEmpty()) {
                        char ch = arrayStack.pop().toString().toCharArray()[0];
                        if (c == '}' && ch != '{' || c == ']' && ch != '[' ||
                                c == '>' && ch != '<') {
                            System.out.println("Error:" + ch + "-" + c);

                        }
                    }
                    break;
                default:
                    break;

            }
        }


    }

}



````


# 总结

栈要利用它的先进后出的特性处理数据。


