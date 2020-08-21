---
title: java数据结构和算法10-树
copyright: true
date: 2019-03-26 23:25:02
tags: 树
categories: 数据结构和算法

---

前面我们介绍数组的数据结构，我们知道对于有序数组，查找很快，并介绍可以通过二分法查找，但是想要在有序数组中插入一个数据项，就必须先找到插入数据项的位置，然后将所有插入位置后面的数据项全部向后移动一位，来给新数据腾出空间，平均来讲要移动N/2次，这是很费时的。同理，删除数据也是。

然后我们介绍了另外一种数据结构——链表，链表的插入和删除很快，我们只需要改变一些引用值就行了，但是查找数据却很慢了，因为不管我们查找什么数据，都需要从链表的第一个数据项开始，遍历到找到所需数据项为止，这个查找也是平均需要比较N/2次。

那么我们就希望一种数据结构能同时具备数组查找快的优点以及链表插入和删除快的优点，于是 树 诞生了。


# 树的概念


![树](/images/datastructure/树.png)

①、路径：顺着节点的边从一个节点走到另一个节点，所经过的节点的顺序排列就称为“路径”。

②、根：树顶端的节点称为根。一棵树只有一个根，如果要把一个节点和边的集合称为树，那么从根到其他任何一个节点都必须有且只有一条路径。A是根节点。

③、父节点：若一个节点含有子节点，则这个节点称为其子节点的父节点；B是D的父节点。

④、子节点：一个节点含有的子树的根节点称为该节点的子节点；D是B的子节点。

⑤、兄弟节点：具有相同父节点的节点互称为兄弟节点；比如上图的D和E就互称为兄弟节点。

⑥、叶节点：没有子节点的节点称为叶节点，也叫叶子节点，比如上图的H、E、F、G都是叶子节点。

⑦、子树：每个节点都可以作为子树的根，它和它所有的子节点、子节点的子节点等都包含在子树中。

⑧、节点的层次：从根开始定义，根为第一层，根的子节点为第二层，以此类推。

⑨、深度：对于任意节点n,n的深度为从根到n的唯一路径长，根的深度为0；

⑩、高度：对于任意节点n,n的高度为从n到一片树叶的最长路径长，所有树叶的高度为0；

# 二叉树

二叉树：树的每个节点最多只能有两个子节点。

二叉搜索树(binary search tree)：若它的左子树不空，则左子树上所有结点的值均小于它的根结点的值； 若它的右子树不空，则右子树上所有结点的值均大于它的根结点的值； 它的左、右子树也分别为二叉排序树。

树的效率：查找节点的时间取决于这个节点所在的层数，每一层最多有2n-1个节点，总共N层共有2n-1个节点，那么时间复杂度为O(logn),底数为2。



## 查找节点

查找某个节点，我们必须从根节点开始遍历。

①、查找值比当前节点值大，则搜索右子树；

②、查找值等于当前节点值，停止搜索（终止条件）；

③、查找值小于当前节点值，则搜索左子树；

## 插入节点

 要插入节点，必须先找到插入的位置。与查找操作相似，由于二叉搜索树的特殊性，待插入的节点也需要从根节点开始进行比较，小于根节点则与根节点左子树比较，反之则与右子树比较，直到左子树为空或右子树为空，则插入到相应为空的位置，在比较的过程中要注意保存父节点的信息 
 及 待插入的位置是父节点的左子树还是右子树，才能插入到正确的位置。


## 遍历树

````
①、中序遍历:左子树——》根节点——》右子树 （根节点在中间）
②、前序遍历:根节点——》左子树——》右子树 （根节点在前边）
③、后序遍历:左子树——》右子树——》根节点 （根节点在后边）

````

![遍历树](/images/datastructure/遍历树.png)

## 用数组表示树

用数组表示树，那么节点是存在数组中的，节点在数组中的位置对应于它在树中的位置。下标为 0 的节点是根，下标为 1 的节点是根的左子节点，以此类推，按照从左到右的顺序存储树的每一层。

![用数组表示树](/images/datastructure/用数组表示树.png)

在大多数情况下，使用数组表示树效率是很低的，不满的节点和删除掉的节点都会在数组中留下洞，浪费存储空间。更坏的是，删除节点如果要移动子树的话，子树中的每个节点都要移到数组中新的位置，这是很费时的。

# 总结

树是由边和节点构成，根节点是树最顶端的节点，它没有父节点；二叉树中，最多有两个子节点；某个节点的左子树每个节点都比该节点的关键字值小，右子树的每个节点都比该节点的关键字值大，那么这种树称为二叉搜索树，其查找、插入、删除的时间复杂度都为logN；可以通过前序遍历、中序遍历、后序遍历来遍历树，前序是根节点-左子树-右子树，中序是左子树-根节点-右子树，后序是左子树-右子树-根节点；删除一个节点只需要断开指向它的引用即可；哈夫曼树是二叉树，用于数据压缩算法，最经常出现的字符编码位数最少，很少出现的字符编码位数多一些。

````
package com.chen.algorithm.tree;

/**
 * 二叉树搜索树
 *
 * @author :  chen weijie
 * @Date: 2019-04-08 11:22 AM
 */
public class BinaryTree implements Tree {


    private Node root;


    @Override
    public Node find(int key) {

        Node current = root;

        while (current != null) {
            //当前值比查找值大，搜索左子树
            if (current.data > key) {
                current = current.leftNode;
                //当前值比查找值小，搜索右子树
            } else if (current.data < key) {
                current = current.rightNode;
            } else {
                return current;
            }
        }
        return null;
    }

    @Override
    public boolean insert(int data) {

        Node newNode = new Node(data);
        //当前树为空树，没有任何节点
        if (root == null) {
            root = newNode;
            return true;
        } else {
            Node current = root;
            Node parentNode = null;

            while (current != null) {
                parentNode = current;
                //当前值比插入值大，搜索左子节点
                if (current.data > data) {
                    current = current.leftNode;
                    //左子节点为空，直接将新值插入到该节点
                    if (current == null) {
                        parentNode.leftNode = newNode;
                        return true;
                    }
                } else {
                    current = current.rightNode;
                    if (current == null) {
                        parentNode.rightNode = newNode;
                        return true;
                    }
                }
            }
        }
        return false;
    }

    @Override
    public boolean delete(int key) {
        return false;
    }


    /**
     * 中序遍历
     *
     * @param current
     */
    public void infixOrder(Node current) {

        if (current != null) {
            infixOrder(current.leftNode);
            System.out.println(current.data + " ");
            infixOrder(current.rightNode);
        }
    }


    /**
     * 前序遍历
     *
     * @param current
     */
    public void preOrder(Node current) {
        if (current != null) {
            System.out.println(current.data + " ");
            preOrder(current.leftNode);
            preOrder(current.rightNode);
        }
    }

    /**
     * 后序遍历
     *
     * @param current
     */
    public void postOrder(Node current) {

        if (current != null) {
            postOrder(current.leftNode);
            postOrder(current.rightNode);
            System.out.println(current.data + " ");
        }
    }


    /**
     * 找到最大的节点
     *
     * @return
     */
    public Node findMax() {
        Node current = root;
        Node maxNode = current;
        while (current != null) {
            maxNode = current;
            current = current.rightNode;
        }
        return maxNode;
    }

    /**
     * 查找最小节点
     *
     * @return
     */
    public Node finMin() {
        Node current = root;
        Node minNode = current;
        while (current != null) {
            minNode = current;
            current = current.leftNode;
        }
        return minNode;
    }

    public Node getRoot() {
        return root;
    }

    public void setRoot(Node root) {
        this.root = root;
    }

    public static void main(String[] args) {

        BinaryTree bt = new BinaryTree();
        bt.insert(50);
        bt.insert(20);
        bt.insert(80);
        bt.insert(10);
        bt.insert(30);
        bt.insert(60);
        bt.insert(90);
        bt.insert(25);
        bt.insert(85);
        bt.insert(100);

        System.out.println("infixOrder:");
        bt.infixOrder(bt.root);
        System.out.println("preOrder:");
        bt.preOrder(bt.root);
        System.out.println("postOrder:");
        bt.postOrder(bt.root);


        System.out.println("max:" + bt.findMax().data);
        System.out.println("min:" + bt.finMin().data);
        System.out.println(bt.find(100));
        System.out.println(bt.find(200));
    }

}

````

[参考资料](https://www.cnblogs.com/ysocean/p/8032642.html)