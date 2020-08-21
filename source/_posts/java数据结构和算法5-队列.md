---
title: java数据结构和算法5-队列
copyright: true
date: 2019-03-06 23:56:59
tags: 队列
categories: 数据结构

---

# 队列的基本概念

队列（queue）是一种特殊的线性表，特殊之处在于它只允许在表的前端（front）进行删除操作，而在表的后端（rear）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为队尾，进行删除操作的端称为队头。队列中没有元素时，称为空队列。

队列的数据元素又称为队列元素。在队列中插入一个队列元素称为入队，从队列中删除一个队列元素称为出队。因为队列只允许在一端插入，在另一端删除，所以只有最早进入队列的元素才能最先从队列中删除，故队列又称为先进先出。

# 队列的分类

队列分为：

①、单向队列（Queue）：只能在一端插入数据，另一端删除数据。

②、双向队列（Deque）：每一端都可以进行插入数据和删除数据操作。

③、这里我们还会介绍一种队列——优先级队列，优先级队列是比栈和队列更专用的数据结构，在优先级队列中，数据项按照关键字进行排序，关键字最小（或者最大）的数据项往往在队列的最前面，而数据项在插入的时候都会插入到合适的位置以确保队列的有序。

# java模拟单向队列

①、与栈不同的是，队列中的数据不总是从数组的0下标开始的，移除一些队头front的数据后，队头指针会指向一个较高的下标位置，如下图：

![队列1](/images/datastructure/队列1.png)

②、我们在设计时，队列中新增一个数据时，队尾的指针rear 会向上移动，也就是向下标大的方向。移除数据项时，队头指针 front 向上移动。那么这样设计好像和现实情况相反，比如排队买电影票，队头的买完票就离开了，然后队伍整体向前移动。在计算机中也可以在队列中删除一个数之后，队列整体向前移动，但是这样做效率很差。我们选择的做法是移动队头和队尾的指针。

③、如果向第②步这样移动指针，相信队尾指针很快就移动到数据的最末端了，这时候可能移除过数据，那么队头会有空着的位置，然后新来了一个数据项，由于队尾不能再向上移动了，为了避免队列不满却不能插入新的数据，我们可以让队尾指针绕回到数组开始的位置，这也称为“循环队列”。


````
package com.chen.dataStructure.queue;

/**
 * @author :  chen weijie
 * @Date: 2019-03-07 12:13 AM
 */
public class MyQueue {

    private Object[] queArray;

    /**
     * 队列总大小
     */
    private int maxSize;

    /**
     * 前端
     */
    private int front;

    /**
     * 后端
     */
    private int fear;

    /**
     * 队列中实际元素个数
     */
    private int nItems;

    public MyQueue(int s) {
        this.maxSize = s;
        this.queArray = new Object[maxSize];
        front = 0;
        fear = -1;
        nItems = 0;
    }


    /**
     * 返回队列的大小
     *
     * @return
     */
    public int getSize() {
        return nItems;
    }

    /**
     * 队列是否为空
     *
     * @return
     */
    public boolean isEmpty() {
        return nItems == 0;
    }


    /**
     * 队列是否满了
     *
     * @return
     */
    public boolean isFull() {
        return maxSize == nItems;
    }

    /**
     * 查看队头元素
     *
     * @return
     */
    public Object peekFront() {
        return queArray[front];
    }


    /**
     * 移除元素
     *
     * @return
     */
    public Object remove() {

        Object removeValue = null;
        if (!isEmpty()) {
            removeValue = peekFront();
            front++;
            if (front == maxSize) {
                front = 0;
            }
            nItems--;
            return removeValue;
        }
        return removeValue;
    }



    /**
     * 队列中新增元素
     *
     * @param value
     */
    public void insert(Object value) {

        if (isFull()) {
            System.out.println("队列已满！！！");
        } else {
            if (fear == maxSize - 1) {
                fear = -1;
            }
            //队尾指针加1，然后在队尾指针处插入新的数据
            queArray[++fear] = value;
            nItems++;
        }
    }


    public static void main(String[] args) {

        MyQueue myQueue = new MyQueue(4);

        myQueue.insert(10);
        myQueue.insert(20);
        myQueue.insert(30);
        myQueue.insert(40);

        //获取队头元素
        System.out.println("myQueue.peekFront():" + myQueue.peekFront());

        //移除元素
        myQueue.remove();

        //插入元素
        myQueue.insert(50);
        myQueue.insert(60);

        for (Object a : myQueue.queArray) {
            System.out.println("a:" + a);
        }
    }




}

````

# 优先级队列

优先级队列（priority queue）是比栈和队列更专用的数据结构，在优先级队列中，数据项按照关键字进行排序，关键字最小（或者最大）的数据项往往在队列的最前面，而数据项在插入的时候都会插入到合适的位置以确保队列的有序。

数组实现优先级队列，声明为int类型的数组，关键字是数组里面的元素，在插入的时候按照从大到小的顺序排列，也就是越小的元素优先级越高。


# 总结之前的数据结构

①、栈、队列（单向队列）、优先级队列通常是用来简化某些程序操作的数据结构，而不是主要作为存储数据的。

②、在这些数据结构中，只有一个数据项可以被访问。

③、栈允许在栈顶压入（插入）数据，在栈顶弹出（移除）数据，但是只能访问最后一个插入的数据项，也就是栈顶元素。

④、队列（单向队列）只能在队尾插入数据，对头删除数据，并且只能访问对头的数据。而且队列还可以实现循环队列，它基于数组，数组下标可以从数组末端绕回到数组的开始位置。

⑤、优先级队列是有序的插入数据，并且只能访问当前元素中优先级别最大（或最小）的元素。

⑥、这些数据结构都能由数组实现，但是可以用别的机制（后面讲的链表、堆等数据结构）实现。