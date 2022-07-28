---
title: 2-3树与红黑树
date: 2021-08-18 09:28:33
categories: data structure
tags:
    - rb-tree
    - 23-tree
description: 之前一直死记硬背红黑树的特性以及左旋右旋，但实际上红黑树是由23树演变过来的。通过学习23树以及演变规则，我们能更好地了解红黑树的特性
---
# 2-3树
## 定义 
2-3树是一棵平衡搜索树
## 节点类型
- 2-节点 
    存放一个值，两个节点
- 3-节点
    存放两个值，三个节点

![](rbtree-and-23tree/2-3_tree.png)

## api
- search  
    从根节点开始，比节点值小则搜左边，比节点大则搜右边，若是3-节点还可以搜中间  
    ![](rbtree-and-23tree/search.gif)  
    ![](rbtree-and-23tree/search2.gif)  
- insert  
    先搜索，后插入  
    - 若待插入的点是2-节点  
        则升级2-节点为3-节点  
        ![](rbtree-and-23tree/insert1.gif)
    - 若待插入的点是3-节点  
        则升级3-节点为临时4-节点。接着取临时4-节点的中间值，将其合并到临时4-节点的父节点处  
        ![](rbtree-and-23tree/insert2.gif)  
        ![](rbtree-and-23tree/insert3.gif)  

## 效率
1. 查找的时间复杂度：log<sub>3</sub>N ~ log<sub>2</sub>N  
1. 节点的合并与升级是常数级操作，因此插入的时间复杂度也是：log<sub>3</sub>N ~ log<sub>2</sub>N  
3. 由于他新增层数是自底向上的，因此肯定是平衡的

## demo
![](rbtree-and-23tree/23tree-demo.gif)  

# 红黑树
## 定义
红黑树是一棵二叉搜索树  

## 特点
- 节点非黑即红
- 根节点是黑色的
- 叶子节点是黑色的
- 不存在两个连续的红色节点
- 从根节点到叶子节点的黑色数量相同  

## 与2-3树的关系
2-3树与红黑树可以互相转换。其转换方式为：**3节点中较大值作为根节点，较小值为其左子节点，且该边为红边**  
![](rbtree-and-23tree/transform.png)  
边的颜色与节点的颜色关系是：**节点颜色代表该节点到其父节点的边的颜色**  
![](rbtree-and-23tree/color.png)  
完整的转换：
![](rbtree-and-23tree/transform2.png)  

## 基础操作
- 左旋：将一个右侧的红边移到左侧（暂时的）  
    ![](rbtree-and-23tree/left-rotation.gif)
- 右旋：将一个左侧的红边移到右侧（暂时的）  
    ![](rbtree-and-23tree/right-rotation.gif)
- 变色：对应2-3树中，临时4节点的分裂与合并
    ![](rbtree-and-23tree/flip-color.gif)  
  
基础操作的目的：维持搜索树的特性以及黑节点的平衡  

## insert  
**插入的节点必须是红色节点（2-3树的合并特性）**  
基本策略：插入一个节点后，通过上述基础操作，保证红黑树与2-3树能对应转换  
插入类型：
- 2-节点  
    ![](rbtree-and-23tree/insert-2node1.png)  
    ![](rbtree-and-23tree/insert-2node2.png)  
- 3-节点
    ![](rbtree-and-23tree/insert-3node1.png)  
    ![](rbtree-and-23tree/insert-3node2.png)  
    ![](rbtree-and-23tree/insert-3node3.png)  

小诀窍：
- 右节点红，左节点黑，**左旋**
- 左节点红，左左节点红，**右旋**
- 左右节点红，**变色**  

## demo
![](rbtree-and-23tree/rbtree-demo.gif)