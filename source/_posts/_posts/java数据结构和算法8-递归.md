---
title: java数据结构和算法8-递归
copyright: true
date: 2019-03-14 00:14:41
tags: 算法
categories: 递归

---

# 递归的定义

递归就是在运行的过程中调用自己，必须具备三个要素：

1.边界条件；
2.递归前进段；
3.递归返回段；

当边界条件不满足时，递归前进；当边界条件满足时，递归返回。

# 第一个数的阶乘

````
package com.chen.algorithm.recursion;

/**
 * 求一个数的阶乘
 *
 * @author :  chen weijie
 * @Date: 2019-03-14 11:53 PM
 */
public class Recursion {


    /**
     * for循环处理阶乘
     * @param n
     * @return
     */
    public static int getFactorialFor(int n) {

        int temp = 1;
        if (n < 0) {
            return -1;
        } else {
            for (int i = 1; i <= n; i++) {
                temp = temp * i;
                System.out.println("i===" + i + ",temp==" + temp);
            }
        }
        return temp;
    }


    public static int getFactorial(int n) {
        if (n >= 0) {
            if (n == 0) {
                return 1;
            } else {
                return n * getFactorial(n - 1);
            }
        }

        return -1;
    }

    public static void main(String[] args) {
        System.out.println(getFactorialFor(3));
        System.out.println(getFactorial(0));
    }
}

````

# 递归的二分查找

二分查找的数组一定是有序的

## 思路

在有序数组array[]中，不断将数组的中间值（mid）和被查找的值比较，如果被查找的值等于array[mid],就返回下标mid; 否则，就将查找范围缩小一半。如果被查找的值小于array[mid], 就继续在左半边查找;如果被查找的值大于array[mid],  就继续在右半边查找。 直到查找到该值或者查找范围为空时， 查找结束。


````

/**
 * 找到目标值返回数组下标，找不到返回-1
 *
 * @param array
 * @param key
 * @return
 */
public static int findTwoPoint(int[] array, int key) {

    if (array == null || array.length == 0) {
        return -1;
    }

    int min = 0;
    int max = array.length - 1;

    while (max >= min) {

        int mid = (max + min) / 2;

        if (key == array[mid]) {
            return mid;
        }

        if (key > array[mid]) {

            min = mid + 1;
        }

        if (key < array[mid]) {
            max = mid - 1;
        }
    }
    return -1;
}


/**
 * 递归二分查找
 *
 * @param low   低位
 * @param high  高位
 * @param array 有序数组
 * @param key   要查找的值
 * @return
 */
public static int sort(int low, int high, int[] array, int key) {

    while (low < high) {
        int mid = (low + high) / 2;

        if (key == array[mid]) {
            return mid;
        }

        if (key > array[mid]) {
            low = mid + 1;
            sort(low, high, array, key);
        }

        if (key < array[mid]) {
            high = mid - 1;
            sort(low, high, array, key);
        }
    }
    return -1;
}


````

# 汉诺塔问题

我们都将其看做只有两个盘子。假设有 N 个盘子在塔座A上，我们将其看为两个盘子，其中(N-1)~1个盘子看成是一个盘子，最下面第N个盘子看成是一个盘子，那么解决办法为：

　　①、先将A塔座的第(N-1)~1个盘子看成是一个盘子，放到中介塔座B上，然后将第N个盘子放到目标塔座C上。

　　②、然后A塔座为空，看成是中介塔座，B塔座这时候有N-1个盘子，将第(N-2)~1个盘子看成是一个盘子，放到中介塔座A上，然后将B塔座的第(N-1)号盘子放到目标塔座C上。

　　③、这时候A塔座上有(N-2)个盘子，B塔座为空，又将B塔座视为中介塔座，重复①，②步骤，直到所有盘子都放到目标塔座C上结束。

简单来说，跟把大象放进冰箱的步骤一样，递归算法为：

　　①、从初始塔座A上移动包含n-1个盘子到中介塔座B上。

　　②、将初始塔座A上剩余的一个盘子（最大的一个盘子）放到目标塔座C上。

　　③、将中介塔座B上n-1个盘子移动到目标塔座C上。

````
public static void move(int dish, String from, String temp, String to) {

    if (dish == 1) {
        System.out.println("将盘子" + dish + "从塔座" + from + "移动到目标塔座" + to);
    } else {
        //A为初始塔，B为目标塔，C为中介塔
        move(dish - 1, from, to, temp);
        System.out.println("将盘子" + dish + "从塔座" + from + "移动到目标塔座" + to);
        //B为初始塔，C为目标塔，A是中介塔
        move(dish - 1, temp, from, to);
    }
}

````

## 归并排序

````
package com.chen.algorithm.recursion;

/**
 * 归并排序
 * <p>
 * 　归并算法的中心是归并两个已经有序的数组。归并两个有序数组A和B，就生成了第三个有序数组C。数组C包含数组A和B的所有数据项。
 *
 * @author :  chen weijie
 * @Date: 2019-03-25 11:31 PM
 */
public class MergeSort {


    public static int[] sort(int[] a, int[] b) {


        int aNum = 0, bNum = 0, cNum = 0;

        int[] c = new int[a.length + b.length];

        while (aNum < a.length && bNum < b.length) {
            //将更小的复制给c数组
            if (a[aNum] > b[bNum]) {
                c[cNum++] = b[bNum++];
            } else {
                c[cNum++] = a[aNum++];
            }

            //如果a数组全部赋值到c数组了，但是b数组还有元素，则将b数组剩余元素按顺序全部复制到c数组
            while (aNum == a.length && bNum < b.length) {
                c[cNum++] = b[bNum++];
            }
            //如果b数组全部赋值到c数组了，但是a数组还有元素，则将a数组剩余元素按顺序全部复制到c数组
            while (bNum == b.length && aNum < a.length) {
                c[cNum++] = a[aNum++];
            }
        }
        return c;
    }


    public static void main(String[] args) {

        int[] a = {2, 5, 7, 8, 9, 10};
        int[] b = {1, 2, 3, 5, 6, 10, 29};


        int[] c = sort(a, b);

        for (int i = 0; i < c.length - 1; i++) {
            System.out.println(c[i]);
        }
    }
}


````















