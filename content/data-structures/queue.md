---
title: "栈和队列（C语言底层实现队列）"
date: "2026-03-15T00:20:56+08:00"
draft: false
categories: ["数据结构"]
tags: ["队列", "循环队列", "栈和队列"]
---
如果说“栈”是竖着的薯片桶（后进先出
LIFO），那么“队列”就是**排队买奶茶**（先进先出 FIFO，First In First
Out）。

- **入队（Push / Enqueue）**：新来的人永远排在队伍的**最后面（队尾
  Rear）**。
- **出队（Pop / Dequeue）**：买完奶茶离开的人，永远是队伍最前面（队头
  Front）的那个人。

为什么不用数组，而要用“链表”？

你想想，如果用数组排队，最前面的人（下标为
0）买完走了，后面所有的人是不是都要往前挪一步？在代码里，让几万个数据全部往前移动一格，效率极其低下！所以，我们引入**链表**。你可以把链表想象成一列**火车**：

- 每存一个数据，我们就去内存里临时造一节“车厢”。
- 车厢里装两样东西：**数据本身**，和一条**指向下一节车厢的绳子（指针）**。
- 这样，如果最前面的人走了，我们只需要把指向“队头”的标志换给第二个人，完全不需要挪动其他人！

## 1.定义队列的结构体

```c
// 第 1 个结构体：链表的节点 (相当于“火车车厢”)
typedef struct Node
{
    int data;//存放真正的数据 (乘客)
    struct Node* next;//指向下一节车厢的绳子 (钩子)
}Node;
```



```c
// 第 2 个结构体：队列管理器 (相当于“火车站长”)
typedef struct Queue
{
    Node* front;//队头指针：永远指向排在最前面的人（出队的地方）
    Node* rear;//队尾指针：永远指向排在最后面的人（入队的地方）
}Queue;
```


为什么 `Queue` 里面需要 front 和 rear

 两个指针？

- **front (队头)
  的作用**：出队的时候，我们要知道谁在最前面，才能让他走。
- **`rear` (队尾) 的作用**：想象一下，如果只有
  `front`，当来了一个新人要排队时，我们不得不顺着队头一个人一个人地往后找：“你是最后一个吗？不是？那是你吗？……”，这就太慢了！有了
  `rear`，站长直接看着队尾，新人来了直接用绳子连在 `rear`
  后面，一步到位，极其高效！

## 2.初始化队列

```c
void queueInit(Queue* q)
{
    // 刚开始的时候，站台是空的，一节车厢（乘客）都没有
    // 所以队头 (front) 和队尾 (rear) 都指向 NULL（空）
    q->front = NULL;
    q->rear = NULL;
```


```c
    printf("初始化成功，队列目前为空\n");
}
```


**注意到我们的队列是不需要malloc的**

**之前的栈（基于数组）：**
就像是建一栋楼。你在打地基（初始化）的时候，就必须告诉包工头（操作系统）：“我要建一栋
10 层的楼（`capacity = 10`）”。包工头会一次性把 10 层的空间全都给你用
`malloc` 批下来。哪怕你现在只住 1 个人，那 9 层的空间也是空着浪费的。

**现在的队列（基于链表）：**
就像是造小火车。火车站长（`queueInit`）刚上任的时候，**一节车厢都不造，一丁点额外的内存都不借**。他只准备了两个空闲的钩子（`front`
和 `rear` 都指向 `NULL`）。

**那什么时候才用 `malloc` 呢？** 只有当第一个真正排队的人（入队
Push）来到站台时，站长才会去临时找包工头（`malloc`），**现做一节车厢**给他。来一个人，就现造一节车厢；走一个人，就把那节车厢销毁（`free`）。这叫做**按需分配**，绝不浪费一滴内存！

## 3.判空函数

```c
bool queueIsEmpty(Queue* q)
{
    //只要队头没有指向任何车厢，说明队列是空的
    if (q->front == NULL)
    {
        return true;
    }
    else
    {
        return false;
    }
}
```


队列为空的特征是 `front == NULL`

## 4.入队

### 在入队之前我们先写一个辅助函数，用来创建新节点

```c
//辅助函数（创建新节点）
Node* createNode(int value)
{
    Node* newNode = (Node*)malloc(sizeof(Node));
```


```c
    //检测是否malloc成功
    if (newNode == NULL)
    {
        perror("malloc failed");
        return NULL;
    }
```


```c
    newNode->data = value;
    newNode->next = NULL;
```


```c
    return newNode;
}
```


### 之后编写我们的入队函数

```c
void queuePush(Queue* q, int value)
{
    Node* newNode = createNode(value);
    if (newNode == NULL)
    {
        return;
    }
```


```c
    //入队情况要分类讨论
    if (queueIsEmpty(q))
    {
        q->front = newNode;
        q->rear = newNode;
    }
    else
    {
        q->rear->next = newNode;
        q->rear = newNode;
    }
}
```


为什么分情况 A 和情况 B？

很多初学者在这里会摔跟头，他们会问：*“干嘛这么麻烦？不管是第几个来的，直接
`q->rear->next = newNode;` 然后 `q->rear = newNode;` 不就行了吗？”*

**绝对不行！这就是 C 语言里最容易引发“程序崩溃（段错误 Segmentation
Fault）”的雷区。**

你想想，如果站台是空的（情况 A），那时候的 `q->rear` 是什么？我们在
`Init` 初始化的时候把它设成了 `NULL`，对吧？ 如果你不分情况，直接写
`q->rear->next = newNode;`，翻译成大白话就是：**“让那个不存在的幽灵，伸出它的绳子去拉新车厢。”**
电脑一听直接宕机，因为 `NULL` 是没有 `next` 属性的！

所以，我们必须严谨地区分：

- **如果是第一个来的**：直接把 `front` 和 `rear`
  这两个大标志牌都挂在他身上。
- **如果不是第一个来的**：才是让真正的“最后一个人”去拉他，然后再更新队尾标志。

## 5.出队

在链表队列中，出队（Dequeue）的操作永远发生在**队头（Front）**。

```c
int queuePop(Queue* q)
{
    if (queueIsEmpty(q))
    {
        printf("出队失败，队列是空的\n");
        return -1;
    }
```


```c
    // 我们必须用一个临时指针 temp 抓住现在的队头车厢，不然等会儿改变指针后就找不到它了！
    Node* tmp = q->front;
    int popValue = tmp->data;
```


```c
    //让队列向前走一步
    q->front = q->front->next;
```


```c
    //【极其致命的隐藏雷区！】检查队伍是不是空了
    // 如果刚才走的那个人，刚好是队伍里的最后一个人
    // 那么现在 q->front 已经变成了 NULL。
    // 此时，q->rear（队尾）还指着刚才那个马上要被销毁的车厢，变成了危险的“野指针”！
    // 所以我们必须手动把队尾也清空：
    if (q->front == NULL)
    {
        q->rear = NULL;
    }
```


```c
    free(tmp);
```


```c
    printf("%d出队成功\n", popValue);
    return popValue;
}
```


**为什么要用 `temp` 指针？为什么不直接 `free(q->front)`？**

很多新手会这样写（**这是错的！**）：

```c
int value = q->front->data;
free(q->front);              // 先把队头销毁了
q->front = q->front->next;   // 然后再去拿下一个人的地址
```


**错在哪？** 你都已经把这节车厢炸毁了（`free`），你再去这节废墟里找
`next`（通向下一节车厢的绳子），电脑瞬间就会崩溃（段错误）！
**正确做法：** 必须先找个临时工 `temp` 抓住它，然后让 `q->front`
安全地转移到下一个人身上，最后再回头把 `temp`
抓住的那节车厢炸掉。这就是“先过桥，再拆桥”。

**为什么要检查 `if (q->front == NULL)`？**

假设队伍里原本只有 **1** 个人。 此时，`front` 和 `rear` 都指着他。
他出队了，`q->front = q->front->next;` 执行完后，`front` 乖乖变成了
`NULL`。 **但是 `rear` 呢？** 刚才没有任何代码去管
`rear`！如果你不管它，它依然指着刚才那块已经被 `free` 掉的内存。
等下一次有人来入队（Push）时，程序一看：哎哟，`rear`
不是空呀！然后它就会傻傻地执行
`q->rear->next = newNode;`。去一块已经被销毁的废墟里连绳子，**程序再次崩溃！**
所以，**当最后一个元素出队时，必须把 `rear` 也同步重置为
`NULL`**，让站台彻底恢复刚初始化时的干净状态。

## 6.销毁队列

```c
void queueDestroy(Queue* q)
{
    //借助我们写的queuePop函数
    while (!queueIsEmpty(q))
    {
        queuePop(q);
    }
```


```c
    printf("队列成功销毁\n");
}
```


如果你不复用 `queuePop`，你就得在 `queueDestroy`
里重新写一遍找临时指针、断开绳子、释放内存的循环逻辑。但是既然 `Pop`
已经能完美做到“拿数据 + 销毁节点”，我们直接用一个 `while`
循环调用它，简直优雅到了极致！

## 7.整合成完整代码

Queue.h

```c
#pragma once
```


```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
```


```c
// 第 1 个结构体：链表的节点 (相当于“火车车厢”)
typedef struct Node
{
    int data;//存放真正的数据 (乘客)
    struct Node* next;//指向下一节车厢的绳子 (钩子)
}Node;
```



```c
// 第 2 个结构体：队列管理器 (相当于“火车站长”)
typedef struct Queue
{
    Node* front;//队头指针：永远指向排在最前面的人（出队的地方）
    Node* rear;//队尾指针：永远指向排在最后面的人（入队的地方）
}Queue;
```


```c
//初始化队列
void queueInit(Queue* q);
//判空
bool queueIsEmpty(Queue* q);
//辅助函数（创建新节点）
Node* createNode(int value);
//入队
void queuePush(Queue* q, int value);
//出队
int queuePop(Queue* q);
//销毁队列
void queueDestroy(Queue* q);
```


Queue.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "Queue.h"
```


```c
void queueInit(Queue* q)
{
    // 刚开始的时候，站台是空的，一节车厢（乘客）都没有
    // 所以队头 (front) 和队尾 (rear) 都指向 NULL（空）
    q->front = NULL;
    q->rear = NULL;
```


```c
    printf("初始化成功，队列目前为空\n");
}
```


```c
bool queueIsEmpty(Queue* q)
{
    //只要队头没有指向任何车厢，说明队列是空的
    if (q->front == NULL)
    {
        return true;
    }
    else
    {
        return false;
    }
}
```


```c
//辅助函数（创建新节点）
Node* createNode(int value)
{
    Node* newNode = (Node*)malloc(sizeof(Node));
```


```c
    //检测是否malloc成功
    if (newNode == NULL)
    {
        perror("malloc failed");
        return NULL;
    }
```


```c
    newNode->data = value;
    newNode->next = NULL;
```


```c
    return newNode;
}
```


```c
void queuePush(Queue* q, int value)
{
    Node* newNode = createNode(value);
    if (newNode == NULL)
    {
        return;
    }
```


```c
    //入队情况要分类讨论
    if (queueIsEmpty(q))
    {
        // 【情况 A：站台里空无一人，他是第一个来的！】
        // 既然只有他一个人，那他既是排在最前面的 (队头)，也是排在最后面的 (队尾)
        q->front = newNode;
        q->rear = newNode;
    }
    else
    {
        // 【情况 B：队伍里已经有其他人了】
        // 让原来排在队伍最后的那个人 (q->rear)，向后扔出绳子 (next)，挂住这个新车厢
        q->rear->next = newNode;
        q->rear = newNode;
    }
```


```c
    printf("%d入队成功\n", value);
}
```


```c
int queuePop(Queue* q)
{
    if (queueIsEmpty(q))
    {
        printf("出队失败，队列是空的\n");
        return -1;
    }
```


```c
    // 我们必须用一个临时指针 temp 抓住现在的队头车厢，不然等会儿改变指针后就找不到它了！
    Node* tmp = q->front;
    int popValue = tmp->data;
```


```c
    //让队列向前走一步
    q->front = q->front->next;
```


```c
    //【极其致命的隐藏雷区！】检查队伍是不是空了
    // 如果刚才走的那个人，刚好是队伍里的最后一个人
    // 那么现在 q->front 已经变成了 NULL。
    // 此时，q->rear（队尾）还指着刚才那个马上要被销毁的车厢，变成了危险的“野指针”！
    // 所以我们必须手动把队尾也清空：
    if (q->front == NULL)
    {
        q->rear = NULL;
    }
```


```c
    free(tmp);
```


```c
    printf("%d出队成功\n", popValue);
    return popValue;
}
```


```c
void queueDestroy(Queue* q)
{
    //借助我们写的queuePop函数
    while (!queueIsEmpty(q))
    {
        queuePop(q);
    }
```


```c
    printf("队列成功销毁\n");
}
```


test.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "Queue.h"
```


```c
int main()
{
    Queue myQueue;
    
    queueInit(&myQueue);
```


```c
    if (queueIsEmpty)
    {
        printf("队列为空\n");
    }
    else
    {
        printf("队列不为空\n");
    }
```


```c
    queuePush(&myQueue, 1);
```


```c
    queuePop(&myQueue);
```


```c
    queueDestroy(&myQueue);
    return 0;
}
```

