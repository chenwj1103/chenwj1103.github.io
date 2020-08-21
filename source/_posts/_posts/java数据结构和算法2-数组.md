---
title: java数据结构和算法2-数组
copyright: true
date: 2019-02-26 22:47:00
tags: 数组
categories: 数据结构和算法

---

# java数组介绍

在Java中，数组是用来存放同一种数据类型的集合，注意只能存放同一种数据类型(Object类型数组除外)

## 数组的声明

数据类型[] 数组名称 = new 数据类型[数组长度];

数据类型[] 数组名称 = {数组元素1，数组元素2，数组元素3.....}

````
//声明数组1,声明一个长度为3，只能存放int类型的数据
int [] myArray = new int[3];
//声明数组2,声明一个数组元素为 1,2,3的int类型数组
int [] myArray2 = {1,2,3};
````

数组的下标从0开始，但是length属性记录的是数组的长度。

````
//声明数组2,声明一个数组元素为 1,2,3的int类型数组，记着不是myArray2.length-1
int [] myArray2 = {1,2,3};
for(int i = 0 ; i < myArray2.length ; i++){
    System.out.println(myArray2[i]);
}

````

# 例子
````
package com.chen.dataStructure;

/**
 * @author :  chen weijie
 * @Date: 2019-02-26 10:57 PM
 */
public class MyArray {



    //实现 增、删、查、迭代功能

    private int [] intArray;

    /**
     * 数组的元素的个数
     */
    private int elems;

    private int length;

    /**
     * 构造一个长度为50的数组
     */
    public MyArray() {
        elems = 0;
        length = 50;
        intArray = new int[length];
    }


    /**
     * 获取数组的有效长度
     * @return
     */
    public int getSize(){
        return elems;
    }


    /**
     * 遍历数组
     */
    public void display() {
        for (int i = 0; i < elems; i++) {
            System.out.println(intArray[i] + " ");
        }
    }


    /**
     * 数组中添加元素
     *
     * @param value
     * @return
     */
    public boolean add(int value) {

        if (length == elems) {
            return false;
        } else {
            intArray[elems] = value;
            elems++;
        }
        return true;
    }


    /**
     * 根据下标获取元素
     * @param i
     * @return
     */
    public int getElems(int i) {

        if (i < 0 || i > elems) {
            System.out.println("数组下标越界");
        }
        return intArray[i];
    }


    /**
     * 查找元素 查找的元素如果存在则返回下标值，如果不存在，返回 -1
     *
     * @return
     */
    public int find(int value) {

        for (int i = 0; i < elems; i++) {
            if (value == intArray[i]) {
                return i;
            }
        }
        return -1;
    }


    /**
     * 如果要删除的值不存在，直接返回 false;否则返回true，删除成功
     *
     * @param value
     * @return
     */
    public boolean delete(int value) {

        int k = find(value);
        if (k == -1) {
            return false;
        } else {
            if (k == elems - 1) {
                elems--;
            } else {
                for (int i = 0; i < elems - 1; i++) {
                    intArray[elems] = intArray[elems + 1];
                }
                elems--;
            }
            return true;
        }
    }


    /**
     * 修改成功返回true，修改失败返回false
     *
     * @param oldValue
     * @param newValue
     * @return
     */
    public boolean modify(int oldValue, int newValue) {
        int i = find(oldValue);
        if (i == -1) {
            System.out.println("需要修改的数据不存在");
            return false;
        } else {
            intArray[i] = newValue;
            return true;
        }

    }

    public static void main(String[] args) {

        MyArray myArray = new MyArray();
        myArray.add(1);
        myArray.add(12);
        myArray.add(13);
        myArray.add(14);
        myArray.add(15);

        myArray.display();
        System.out.println("find 12:" + myArray.find(12));
        System.out.println("find 10:" + myArray.find(10));
        myArray.delete(10);
        System.out.println("get i=2 :" + myArray.getElems(2));
        System.out.println("size:" + myArray.getSize());
    }

}

````

# 数组的局限性

1.插入快，对于无序数组，上面我们实现的数组就是无序的，即元素没有按照从大到小或者某个特定的顺序排列，只是按照插入的顺序排列。无序数组增加一个元素很简单，只需要在数组末尾添加元素即可，但是有序数组却不一定了，它需要在指定的位置插入
2.查找慢，当然如果根据下标来查找是很快的。但是通常我们都是根据元素值来查找，给定一个元素值，对于无序数组，我们需要从数组第一个元素开始遍历，直到找到那个元素。有序数组通过特定的算法查找的速度会比无需数组快，后面我们会讲各种排序算法。
3.删除慢，根据元素值删除，我们要先找到该元素所处的位置，然后将元素后面的值整体向前面移动一个位置。也需要比较多的时间。
4.数组一旦创建后，大小就固定了，不能动态扩展数组的元素个数。如果初始化你给一个很大的数组大小，那会白白浪费内存空间，如果给小了，后面数据个数增加了又添加不进去了。

