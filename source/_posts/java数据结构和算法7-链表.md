---
title: java数据结构和算法7-链表
copyright: true
date: 2019-02-22 00:07:05
tags: 链表
categories: 数据结构与算法

---


我们知道数组是一种通用的数据结构，能用来实现栈、队列等很多数据结构。而链表也是一种使用广泛的通用数据结构，它也可以用来作为实现栈、队列等数据结构的基础，基本上除非需要频繁的通过下标来随机访问各个数据，否则很多使用数组的地方都可以用链表来代替。

但是我们需要说明的是，链表是不能解决数据存储的所有问题的，它也有它的优点和缺点。本篇博客我们介绍几种常见的链表，分别是单向链表、双端链表、有序链表、双向链表以及有迭代器的链表。并且会讲解一下抽象数据类型（ADT）的思想，如何用 ADT 描述栈和队列，如何用链表代替数组来实现栈和队列。

# 链表

链表通常由一连串节点组成，每个节点包含任意的实例数据（data fields）和一或两个用来指向上一个/或下一个节点的位置的链接（"links"）

链表（Linked list）是一种常见的基础数据结构，是一种线性表，但是并不会按线性的顺序存储数据，而是在每一个节点里存到下一个节点的指针(Pointer)。

# 单向链表

单链表是链表中结构最简单的。一个单链表的节点(Node)分为两个部分，第一个部分(data)保存或者显示关于节点的信息，另一个部分存储下一个节点的地址。最后一个节点存储地址的部分指向空值。

单向链表只可向一个方向遍历，一般查找一个节点的时候需要从第一个节点开始每次访问下一个节点，一直访问到需要的位置。而插入一个节点，对于单向链表，我们只提供在链表头插入，只需要将当前插入的节点设置为头节点，next指向原头节点即可。删除一个节点，我们将该节点的上一个节点的next指向该节点的下一个节点。

![单向链表结构](/images/datastructure/单向链表结构.png)

![在表头增加节点](/images/datastructure/在表头增加节点.png)

![删除节点](/images/datastructure/删除节点.png)

## 单向链表的实现

````
package com.chen.dataStructure.linklistnode;

/**
 * 单链表的具体实现
 *
 * @author :  chen weijie
 * @Date: 2019-02-21 11:35 PM
 */
public class SingleLinkedList {

    //链表节点的个数
    private int size;

    //头节点
    private Node head;


    private class Node {

        //每个节点的数据
        private Object data;

        //每个节点指向下一个节点的连接
        private Node next;

        public Node(Object data) {
            this.data = data;
        }
    }


    /**
     * 单链表的表头添加元素
     *
     * @param data
     * @return
     */
    public Object addHead(Object data) {

        Node newHead = new Node(data);
        if (size == 0) {
            head = newHead;
        } else {
            newHead.next = head;
            head = newHead;
        }
        size++;

        return head;
    }


    /**
     * 删除链表头节点
     *
     * @return
     */
    public Object deleteHead() {
        Object data = head.data;
        head = head.next;
        size--;
        return data;
    }

    /**
     * 找到指定元素返回节点Node，找不到返回null
     *
     * @param data
     * @return
     */
    public Node find(Object data) {

        Node current = head;

        int tempSize = size;
        while (tempSize > 0) {
            if (data.equals(current.data)) {
                return current;
            } else {
                current = current.next;
                tempSize--;
            }

        }
        return null;

    }

    /**
     * 删除指定元素，删除成功返回true
     *
     * @param data
     * @return
     */
    public boolean delete(Object data) {

        if (size == 0) {
            return true;
        }
        Node current = head;
        Node previous = head;

        while (current.data != data) {
            if (current.next == null) {
                return false;
            } else {
                previous = current;
                current = current.next;
            }
        }

        //如果删除的节点是第一个节点
        if (current == head) {
            head = current.next;
            size--;
        } else {
            //删除的节点不是第一个节点
            previous.next = current.next;
            size--;
        }
        return true;
    }


    /**
     * 判断链表是不是空
     *
     * @return
     */
    public boolean isEmpty() {
        return size == 0;
    }


    //显示节点信息
    public void display() {
        if (size > 0) {
            Node node = head;
            int tempSize = size;
            //当前链表只有一个节点
            if (tempSize == 1) {
                System.out.println("[" + node.data + "]");
                return;
            }
            while (tempSize > 0) {
                if (node.equals(head)) {
                    System.out.print("[" + node.data + "->");
                } else if (node.next == null) {
                    System.out.print(node.data + "]");
                } else {
                    System.out.print(node.data + "->");
                }
                node = node.next;
                tempSize--;
            }
            System.out.println();
        } else {
            //如果链表一个节点都没有，直接打印[]
            System.out.println("[]");
        }

    }


    public static void main(String[] args) {

        SingleLinkedList singleLinkedList = new SingleLinkedList();
        singleLinkedList.addHead("A");
        singleLinkedList.addHead("B");
        singleLinkedList.addHead("C");
        singleLinkedList.addHead("D");

        singleLinkedList.display();
        singleLinkedList.delete("B");
        singleLinkedList.display();
        System.out.println("find:" + singleLinkedList.find("D"));

    }
}

````

## 用单向链表实现栈

````
package com.chen.dataStructure.linklistnode;

/**
 * 用单向链表实现栈
 * 栈的pop()方法和push()方法，对应于链表的在头部删除元素deleteHead()以及在头部增加元素addHead()。
 *
 * @author :  chen weijie
 * @Date: 2019-03-08 12:57 AM
 */
public class StackSingleLink {


    private SingleLinkedList linkedList;


    public StackSingleLink(SingleLinkedList linkedList) {
        this.linkedList = linkedList;
    }

    public void push(Object data) {
        linkedList.addHead(data);
    }


    public Object pop() {
        return linkedList.deleteHead();
    }


    public boolean isEmpty() {
        return linkedList.isEmpty();
    }


}

````

# 双端链表

对于单项链表，我们如果想在尾部添加一个节点，那么必须从头部一直遍历到尾部，找到尾节点，然后在尾节点后面插入一个节点。这样操作很麻烦，如果我们在设计链表的时候多个对尾节点的引用，那么会简单很多。

![双端链表](/images/datastructure/双端链表.png)

````
package com.chen.dataStructure.linknode;

/**
 * 双端队列
 *
 * @author :  chen weijie
 * @Date: 2019-03-13 11:12 PM
 */
public class DoublePointLinkedList {


    private Node head;

    private Node tail;

    private int size;


    private class Node {

        private Object data;

        private Node next;

        public Node(Object data) {
            this.data = data;
        }
    }


    public DoublePointLinkedList() {
        this.head = null;
        this.tail = null;
        this.size = 0;
    }


    /**
     * 头部添加节点
     *
     * @param data
     */
    public void addHead(Object data) {

        Node node = new Node(data);

        //如果链表为空，那么头节点和尾节点都是该新增节点
        if (size == 0) {
            size = 0;
            head = node;
            tail = node;
        } else {
            node.next = head;
            head = node;
            size++;
        }
    }


    public void addTail(Object data) {

        Node node = new Node(data);

        //如果链表为空，那么头节点和尾节点都是该新增节点
        if (size == 0) {
            size = 0;
            head = node;
            tail = node;
        } else {
            node.next = tail;
            tail = node;
            size++;
        }

    }


    /**
     * 删除头节点
     *
     * @return
     */
    public boolean deleteHead() {

        if (size == 0) {
            return false;
        }

        if (head.next == null) {
            head = null;
            tail = null;
        } else {
            head = head.next;
        }
        size--;
        return true;
    }


    /**
     * 判断是否为空
     *
     * @return
     */
    public boolean isEmpty() {
        return (size == 0);
    }

    /**
     * 获得链表的节点个数
     *
     * @return
     */
    public int getSize() {
        return size;
    }

    /**
     * 显示节点信息
     */

    public void display() {
        if (size > 0) {
            Node node = head;
            int tempSize = size;
            if (tempSize == 1) {//当前链表只有一个节点
                System.out.println("[" + node.data + "]");
                return;
            }
            while (tempSize > 0) {
                if (node.equals(head)) {
                    System.out.print("[" + node.data + "->");
                } else if (node.next == null) {
                    System.out.print(node.data + "]");
                } else {
                    System.out.print(node.data + "->");
                }
                node = node.next;
                tempSize--;
            }
            System.out.println();
        } else {//如果链表一个节点都没有，直接打印[]
            System.out.println("[]");
        }
    }
}

````

# 双向链表

我们知道单向链表只能从一个方向遍历，那么双向链表它可以从两个方向遍历。

![双向链表](/images/datastructure/双向链表.png)

````
package com.chen.dataStructure.linknode;

/**
 * 双向链表
 *
 * @author :  chen weijie
 * @Date: 2019-03-13 11:49 PM
 */
public class TwoWayLinkedList {


    private int size;

    private Node head;

    private Node tail;


    public class Node {

        private Object data;

        private Node next;

        private Node prew;


        public Node(Object data) {
            this.data = data;
        }
    }


    public TwoWayLinkedList() {
        this.size = 0;
        this.head = null;
        this.tail = null;
    }


    public void addHead(Object data) {

        Node node = new Node(data);
        if (size == 0) {
            head = node;
            tail = node;
        } else {
            head.prew = node;
            node.next = head;
            head = node;
        }

        size++;
    }


    public void addTail(Object data) {

        Node node = new Node(data);
        if (size == 0) {
            head = node;
            tail = node;
        } else {
            node.prew = tail;
            tail.next = node;
            tail = node;
        }
        size++;
    }


    public Node deleteHead() {

        Node temp = head;
        if (size > 0) {
            head = head.next;
            head.prew = null;
            size--;
        }
        return temp;
    }

    public Node deleteTail() {

        Node temp = tail;
        if (size > 0) {
            tail = head.prew;
            tail.next = null;
            size--;
        }
        return temp;
    }

}

````