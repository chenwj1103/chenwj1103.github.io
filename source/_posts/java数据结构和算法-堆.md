---
title: java数据结构和算法-堆
copyright: true
date: 2019-04-14 02:00:13
tags: 堆
categories: 数据结构和算法

---

树包括二叉树、红黑树、2-3-4树、堆等各种不同的树

这里的堆是一种树，由它实现的优先级队列的插入和删除的时间复杂度都为O(logN)，这样尽管删除的时间变慢了，但是插入的时间快了很多，当速度非常重要，而且有很多插入操作时，可以选择用堆来实现优先级队列。

# 堆的定义

1.它是完全二叉树，除了树的最后一层节点不需要是满的，其它的每一层从左到右都是满的。注意下面两种情况，第二种最后一层从左到右中间有断隔，那么也是不完全二叉树。

![完全二叉树和非完全二叉树](/images/datastructure/完全二叉树和非完全二叉树.png)

2.它通常用数组来实现

![数组来实现二叉树](/images/datastructure/数组来实现二叉树.png)

这种用数组实现的二叉树，假设节点的索引值为index，那么：

- 节点的左子节点是 2*index+1，

- 节点的右子节点是 2*index+2，

- 节点的父节点是 （index-1）/2。

3. 堆中的每一个节点的关键字都大于（或等于）这个节点的子节点的关键字。


## 二叉搜索树和堆的区别

这里要注意堆和前面说的二叉搜索树的区别，二叉搜索树中所有节点的左子节点关键字都小于右子节点关键字，在二叉搜索树中通过一个简单的算法就可以按序遍历节点。但是在堆中，按序遍历节点是很困难的，如上图所示，堆只有沿着从根节点到叶子节点的每一条路径是降序排列的，指定节点的左边节点或者右边节点，以及上层节点或者下层节点由于不在同一条路径上，他们的关键字可能比指定节点大或者小。所以相对于二叉搜索树，堆是弱序的。


## 遍历和查找

　前面我们说了，堆是弱序的，所以想要遍历堆是很困难的，基本上，堆是不支持遍历的。

对于查找，由于堆的特性，在查找的过程中，没有足够的信息来决定选择通过节点的两个子节点中的哪一个来选择走向下一层，所以也很难在堆中查找到某个关键字。

因此，堆这种组织似乎非常接近无序，不过，对于快速的移除最大（或最小）节点，也就是根节点，以及能快速插入新的节点，这两个操作就足够了。

## 移除

移除是指删除关键字最大的节点（或最小），也就是根节点。

　　根节点在数组中的索引总是0，即maxNode = heapArray[0];

　　移除根节点之后，那树就空了一个根节点，也就是数组有了一个空的数据单元，这个空单元我们必须填上。

　　第一种方法：将数组所有数据项都向前移动一个单元，这比较费时。

　　第二种方法：

　　　　①、移走根

　　　　②、把最后一个节点移动到根的位置

　　　　③、一直向下筛选这个节点，直到它在一个大于它的节点之下，小于它的节点之上为止。


![堆移除最大的节点](/images/datastructure/堆移除最大的节点.png)

## 插入

插入节点也很容易，插入时，选择向上筛选，节点初始时插入到数组最后第一个空着的单元，数组容量大小增一。然后进行向上筛选的算法。

注意：向上筛选和向下不同，向上筛选只用和一个父节点进行比较，比父节点小就停止筛选了。

![堆插入和向上筛选](/images/datastructure/堆插入和向上筛选.png)














