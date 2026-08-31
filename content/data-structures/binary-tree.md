---
title: "二叉树（C语言底层实现）"
date: "2026-03-25T16:39:44+08:00"
draft: false
categories: ["数据结构"]
tags: ["二叉树", "遍历", "递归"]
---
## 前言

树一般是我们所学习的第一个非线性的数据结构，接下来我们就要介绍经典的二叉树，并带大家实现简单的二叉搜索树。

在开始之前，有人会问什么是二叉树？

简单来说，二叉树就像是一棵倒置的树，由一个个“节点（Node）”连接而成。它的核心规则非常简单：**每一个节点最多只能有两个“分支”（即子节点）。**

![](/images/data-structures/0dbba040a21d4dd18bc798718b822b93.png)

同时再来了解几个核心术语

- **根节点（Root）：** 树的最顶端的第一个节点。整棵树从这里开始生长。
- **子节点（Child）：**
  从某个节点延伸出来的节点。在二叉树中，严格区分为**左子节点（Left
  Child）**和**右子节点（Right Child）**。
- **叶子节点（Leaf）：** 没有子节点的节点（也就是树的最末端）。
- **父节点（Parent）：** 连接着下级子节点的节点。

## 1 定义二叉树的结构体

```c
typedef struct TreeNode
{
    int data;//数据域：用来存东西
    struct TreeNode* left;//指针域：指向左边子节点的指针
    struct TreeNOde* right;//指针域：指向右边子节点的指针
}TreeNode;
```


- `int data`（数据域）这就好比树上结的果实。一个节点如果没有装载任何信息，它就毫无意义。

这里我们用 `int data;`
是为了演示方便，表示这个节点存了一个整数。在实际应用中，它可以是任何类型：`char`（存字符）、`float`（存浮点数），甚至可以是另一个复杂的结构体（比如存一个学生的信息）。

- `struct TreeNode* left`和
  `struct TreeNode* right`（指针域）这是二叉树的灵魂所在。为了把孤立的节点连成一棵树，每个节点必须知道它的“左孩子”和“右孩子”在哪里。

**为什么用指针（`*`）？**
在C语言中，指针存储的是**内存地址**。通过保存左子节点和右子节点的内存地址，当前节点顺着这个地址就能找到它们。这就相当于在节点之间牵了一根线。如果没有子节点，我们就把这个指针设为
`NULL`（空）。

**为什么不能直接写 `struct TreeNode left;`（不带星号）？**
这是一个非常常见的初学者疑问。如果你不加指针，C语言编译器会报错。为什么？因为编译器在分配内存时，需要确切知道一个结构体有多大。如果一个结构体里面直接包含它自己，那它的“肚子里”又有一个自己，层层嵌套，无限循环，编译器永远算不出它到底占多少字节。而**指针的大小是固定的**（通常是
4 个字节或 8
个字节），所以结构体里包含指向自身的指针，在C语言中是完全合法且必要的。

## 2 创建新节点

```c
TreeNode* createNode(int value)
{
    TreeNode* newNode = (TreeNode*)malloc(sizeof(TreeNode));
```


```c
    //检查是否malloc成功
    if (newNode == NULL)
    {
        perror("malloc failed");
        exit(1);//退出程序
    }
```


```c
    //正式初始化节点
    newNode->data = value;
    newNode->left = NULL;
    newNode->right = NULL;
```


```c
    return newNode;
}
```


**为什么要用 `malloc`？** 如果我们在函数里直接写
`TreeNode node;`，这个节点会创建在栈（Stack）**内存里。一旦这个
`createNode`
函数运行结束，栈内存就会被系统自动回收，这个节点就“灰飞烟灭”了，你根本没法把它连到树上去。
使用 `malloc`
是在**堆（Heap）内存中申请空间，这里的空间除非你手动销毁（使用
`free`），否则一直都在。这正是我们维持一棵持久存在的树所需要的。

**为什么要强调 `left = NULL` 和 `right = NULL`？**
在C语言中，如果你不给变量赋初值，它里面装的就会是内存里残留的“垃圾数据”。如果你不把左右指针设为
`NULL`（空指针），它们就会随机指向内存里的某个未知地方（这就是可怕的**野指针**）。以后遍历树的时候，程序就会顺着野指针跑到系统禁区，直接崩溃（Segmentation
Fault）。

## 3 插入节点（核心，注意我们实现的是二叉搜索树）

**二叉搜索树有一个非常迷人的核心规则：** 对于树上的任意一个节点，

1.  它的**左子树**上所有节点的值，都**小于**它的值。
2.  它的**右子树**上所有节点的值，都**大于**它的值。

假设我们现在拿着一个新节点准备放进树里，我们就像是在玩一个闯关游戏：

1.  **遇到空地（NULL）：** 太好了，这就是我的位置，直接扎根！
2.  **遇到一个节点：**
```c
比较一下。如果我比它小，我就往它的左边走；如果我比它大，我就往它的右边走。
```


为了实现这个不断往下找空地的过程，最喜欢用的魔法叫做**递归（Recursion）**。

```c
TreeNode* insertNode(TreeNode* root, int value)
{
    if (root == NULL)
    {
        return createNode(value);
    }
```


```c
    // 如果新来的值比当前节点的数据小，说明它该去左边
    if (value < root->data)
    {
        // 让当前节点的左指针，接住从左子树传回来的结果
        root->left = insertNode(root->left, value);
    }
    // 如果新来的值比当前节点的数据大，说明它该去右边
    else if (value > root->data)
    {
        // 让当前节点的右指针，接住从右子树传回来的结果
        root->right = insertNode(root->right, value);
    }
    // 如果新值等于当前节点的值，我们一般什么都不做，因为二叉搜索树通常不存重复数据
    return root;
}
```


`root->left = insertNode(root->left, value);`这一句上。感觉有点绕？我们来重点解答一下

1.  假设当前节点是 `50`，我们要插入 `30`。
2.  程序发现 `30 < 50`。
3.  `50` 就对它的左边喊话：“喂！那个谁（`root->left`），我这儿有个 `30`
```c
给你，你按规矩把它安排在你的地盘里！安排好之后，**把你的新连接状态报告给我**！”
```

4.  `insertNode(root->left, 30)` 就是去执行这个安排任务。
5.  `root->left = ...` 就是 `50`
```c
在接收下方汇报，并重新抓紧连接它左边这根绳子。
```


**为什么必须要有 `return root;`？**
想象一下我们用绳子（指针）串起一串珠子（节点）。每次插入新珠子，我们都是顺着绳子往下摸。当你把新珠子系在最底下之后，整个树的结构并没有散。为了让最上面的根节点依然能抓住整棵树，每个子节点在干完活之后，都必须“向上一层汇报自己现在的位置”。这就是
`return root;` 在做的事情：保持整棵树的连接不断开。

## 4 遍历打印函数（中序遍历）

二叉树有三种常见的遍历方式（前序、中序、后序）。对于二叉搜索树来说，**中序遍历是最神奇的**，因为它的访问顺序是：**左子树
-> 当前节点 -> 右子树**。

前序遍历就是：**当前节点->左子树->右子树**

后序遍历就是：**左子树->右子树->当前节点**

而如果我们用中序遍历，你想想，左边永远比中间小，中间永远比右边小。如果我们按照“左-中-右”的顺序打印，打印出来的数字刚好就是**从小到大完全排好序的！**

```c
void inorderTraversal(TreeNode* root)
{
    if (root != NULL)
    {
        // 先一路向左，去打印比当前节点小的数
        inorderTraversal(root->left);
        // 打印当前节点自己的值
        printf("%d ", root->data);
        // 最后一路向右，去打印比当前节点大的数
        inorderTraversal(root->right);
    }
}
```


## 5 查找某个指定的数字

理解了插入，查找简直就是小菜一碟！二叉搜索树的查找就像**猜数字游戏**（比如猜
1-100 的数字，你说 50，我告诉你大了还是小了，你下一次就会猜 25 或
75）。每次比较，都能排除掉树里一半的节点，效率极高。

```c
TreeNode* searchNode(TreeNode* root, int target)
{
    if (root == NULL || root->data == target)
    {
        return root;
    }
```


```c
    // 如果目标比当前节点小，顺藤摸瓜去左边找
    if (target < root->data)
    {
        return searchNode(root->left, target);
    }
    // 如果目标比当前节点大，顺藤摸瓜去右边找
    else
    {
        return searchNode(root->right, target);
    }
}
```


这就是二叉搜索树能在数据库索引和快速查找中称王称霸的原因：**不用把所有节点看一遍，顺着大小规律一条道走下去就行了。**

## 6 销毁二叉树

释放整棵树，我们需要用到另一种遍历方式：**后序遍历（Post-order
Traversal）**。

**为什么要用后序遍历（左 -> 右 -> 自己）？** 你仔细想一下：如果你先用
`free(root)` 把根节点（比如最上面的 `50`）给销毁了，那你还怎么顺着它的
`left` 和 `right` 指针去找下面的 `30` 和 `70` 呢？线索全断了！
所以，**我们必须像拆楼一样，从最底层的叶子节点开始拆，最后才能拆根节点。**

```c
void freeTree(TreeNode* root)
{
    if (root != NULL)
    {
        freeTree(root->left);
        freeTree(root->right);
        free(root);
    }
}
```


## 7 删除指定数字

- **光杆司令（叶子节点）：** 直接删。

- **单亲带娃（只有一个子节点）：**
  删掉自己，让唯一的孩子顶替自己的位置。

- **儿女双全（有两个子节点）：**
  找右子树里最小的那个“接班人”，把接班人的数据复制到自己肚子里，然后把真正的接班人节点给删掉。

###### 为了实现情况3，我们首先需要一个小小的**辅助函数**：去一棵树里找最小的值。

```c
TreeNode* findMin(TreeNode* root)
{
    if (root == NULL)
    {
        return NULL;
    }
```


```c
    while (root->left != NULL)
    {
        root = root->left;
    }
    return root;
}
```


只要左边还有路，就一直往左走，走到无路可走，那个节点就是最小的！

###### 核心函数：删除节点

```c
TreeNode* deleteNode(TreeNode* root, int target)
{
    // 递归终止条件：如果树是空的，或者一直找没找到这个数字
    if (root == NULL)
    {
        return root;
    }
```


```c
    if (target < root->data)
    {
        // 目标在左边，去左子树里执行删除任务，并重新接好左边的绳子
        root->left = deleteNode(root->left, target);
    }
    else if (target > root->data)
    {
        // 目标在右边，去右子树里执行删除任务，并重新接好右边的绳子
        root->right = deleteNode(root->right, target);
    }
    else
    {
        // 如果没有左孩子（包含了完全没有孩子的情况）
        if (root->left == NULL)
        {
            TreeNode* tmp = root->right;
            free(root);
            return tmp;
        }
        // 如果没有右孩子
        else if (root->right == NULL)
        {
            TreeNode* tmp = root->left;
            free(root);
            return tmp;
        }
        // 走到这里，说明左右孩子都不为空
        // 第一步：去右子树里，找那个最小的“接班人”
        TreeNode* tmp = findMin(root->right);
        // 第二步：把接班人的数据复制到当前节点里
        root->data = tmp->data;
        // 第三步：当前节点的右子树里现在多了一个重复的接班人，去把它删掉
        root->right = deleteNode(root->right, tmp->data);
    }
    return root;
}
```


假设我们要删掉根节点 `50`，它的右子树里有 `60`（接班人）、`70` 等等。

1.  代码一路走到 `else`，发现 `50` 左右都有孩子。
2.  调用 `findMin(root->right)`，顺着 `50` 的右边一直找，找到了 `60`。
3.  `root->data = temp->data;` 这一句，直接把 `50` 改成了
```c
`60`。现在的树根是 `60` 了！
```

4.  `root->right = deleteNode(root->right, 60);`
```c
这一句，命令原本的右子树：“去把你里面个底层的 `60` 给我删了！”
```

5.  因为底层的 `60`
```c
是最小的，它肯定没有左孩子，这就变成了极其简单的**情况1**或**情况2**，直接被
`free` 掉，干净利落！
```


## 8 整合完整代码

BinaryTree.h

```c
#pragma once
```


```c
#include <stdio.h>
#include <stdlib.h>
```


```c
typedef struct TreeNode
{
    int data;//数据域：用来存东西
    struct TreeNode* left;//指针域：指向左边子节点的指针
    struct TreeNode* right;//指针域：指向右边子节点的指针
}TreeNode;
```


```c
//创建新节点
TreeNode* createNode(int value);
//插入节点
TreeNode* insertNode(TreeNode* root, int value);
//中序遍历打印函数
void inorderTraversal(TreeNode* root);
//查找指定的数字
TreeNode* searchNode(TreeNode* root, int target);
//辅助函数（找到一颗树里的最小值）
TreeNode* findMin(TreeNode* root);
//删除指定的数字
TreeNode* deleteNode(TreeNode* root, int target);
//销毁二叉树
void freeTree(TreeNode* root);
```


BinaryTree.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "BinaryTree.h"
```


```c
TreeNode* createNode(int value)
{
    TreeNode* newNode = (TreeNode*)malloc(sizeof(TreeNode));
```


```c
    //检查是否malloc成功
    if (newNode == NULL)
    {
        perror("malloc failed");
        exit(1);//退出程序
    }
```


```c
    //正式初始化节点
    newNode->data = value;
    newNode->left = NULL;
    newNode->right = NULL;
```


```c
    return newNode;
}
```


```c
TreeNode* insertNode(TreeNode* root, int value)
{
    if (root == NULL)
    {
        return createNode(value);
    }
```


```c
    // 如果新来的值比当前节点的数据小，说明它该去左边
    if (value < root->data)
    {
        // 让当前节点的左指针，接住从左子树传回来的结果
        root->left = insertNode(root->left, value);
    }
    // 如果新来的值比当前节点的数据大，说明它该去右边
    else if (value > root->data)
    {
        // 让当前节点的右指针，接住从右子树传回来的结果
        root->right = insertNode(root->right, value);
    }
    // 如果新值等于当前节点的值，我们一般什么都不做，因为二叉搜索树通常不存重复数据
    return root;
}
```


```c
void inorderTraversal(TreeNode* root)
{
    if (root != NULL)
    {
        // 先一路向左，去打印比当前节点小的数
        inorderTraversal(root->left);
        // 打印当前节点自己的值
        printf("%d ", root->data);
        // 最后一路向右，去打印比当前节点大的数
        inorderTraversal(root->right);
    }
}
```


```c
TreeNode* searchNode(TreeNode* root, int target)
{
    if (root == NULL || root->data == target)
    {
        return root;
    }
```


```c
    // 如果目标比当前节点小，顺藤摸瓜去左边找
    if (target < root->data)
    {
        return searchNode(root->left, target);
    }
    // 如果目标比当前节点大，顺藤摸瓜去右边找
    else
    {
        return searchNode(root->right, target);
    }
}
```


```c
void freeTree(TreeNode* root)
{
    if (root != NULL)
    {
        freeTree(root->left);
        freeTree(root->right);
        free(root);
    }
}
```


```c
//辅助函数
TreeNode* findMin(TreeNode* root)
{
    if (root == NULL)
    {
        return NULL;
    }
```


```c
    while (root->left != NULL)
    {
        root = root->left;
    }
    return root;
}
```


```c
TreeNode* deleteNode(TreeNode* root, int target)
{
    // 递归终止条件：如果树是空的，或者一直找没找到这个数字
    if (root == NULL)
    {
        return root;
    }
```


```c
    if (target < root->data)
    {
        // 目标在左边，去左子树里执行删除任务，并重新接好左边的绳子
        root->left = deleteNode(root->left, target);
    }
    else if (target > root->data)
    {
        // 目标在右边，去右子树里执行删除任务，并重新接好右边的绳子
        root->right = deleteNode(root->right, target);
    }
    else
    {
        // 如果没有左孩子（包含了完全没有孩子的情况）
        if (root->left == NULL)
        {
            TreeNode* tmp = root->right;
            free(root);
            return tmp;
        }
        // 如果没有右孩子
        else if (root->right == NULL)
        {
            TreeNode* tmp = root->left;
            free(root);
            return tmp;
        }
        // 走到这里，说明左右孩子都不为空
        // 第一步：去右子树里，找那个最小的“接班人”
        TreeNode* tmp = findMin(root->right);
        // 第二步：把接班人的数据复制到当前节点里
        root->data = tmp->data;
        // 第三步：当前节点的右子树里现在多了一个重复的接班人，去把它删掉
        root->right = deleteNode(root->right, tmp->data);
    }
    return root;
}
```


test.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "BinaryTree.h"
```


```c
int main()
{
    TreeNode* root = NULL;
    printf("开始向二叉树中插入数字\n");
```


```c
    root = insertNode(root, 50);
    root = insertNode(root, 30);
    root = insertNode(root, 70);
    root = insertNode(root, 20);
    root = insertNode(root, 40);
    root = insertNode(root, 60);
    root = insertNode(root, 80);
```


```c
    printf("中序遍历打印的结果是：\n");
    inorderTraversal(root);
    printf("\n");
```


```c
    int target1 = 100;
    TreeNode* result1 = searchNode(root, target1);
    if (result1 != NULL)
    {
        printf("找到了数字%d\n", result1->data);
    }
    else
    {
        printf("没有找到%d这个数字\n", target1);
    }
```


```c
    root = deleteNode(root, 20);
    printf("删除20之后的中序遍历：\n");
    inorderTraversal(root);
    printf("\n");
```


```c
    freeTree(root);
    printf("二叉树成功被释放\n");
```


```c
    return 0;
}
```

