---
title: "栈和队列（C语言底层实现环形队列）"
date: "2026-03-17T10:00:55+08:00"
draft: false
categories: ["数据结构"]
tags: ["环形队列", "循环队列", "队列"]
---
- **链表队列**
  就像是“排队买奶茶”：队伍长度不固定，人多我就排长一点，人少我就排短一点，灵活多变，适合高级业务逻辑。
- **环形队列**
  就像是“机场的行李转盘”（或者旋转门）：转盘大小是固定的，不需要临时扩建。放行李的工人在后面放（入队），旅客在前面拿（出队）。只要拿的速度跟得上放的速度，这个固定大小的转盘就能无限地处理几万件行李！

## 1.定义环形队列的结构体

```c
typedef struct CircularQueue
{
    int* data;//待会用malloc获取一块连续的数字空间
    int capacity;//记录这个队列的总容量
    int front;//队头游标，排在最前面的数据
    int rear;//队尾游标，永远指向下一个空位
}CircularQueue;
```


## 2.初始化环形队列

我们要体现环形队列最核心的“心法”：**为了区分“空”和“满”，必须故意浪费一个座位！**

```c
void queueInit(CircularQueue* q, int maxSize)
{
    //【核心心法】：你想装 maxSize 个人，我底层必须开辟 maxSize + 1 个位置！
    //那个多出来的位置，就是为了防止 front 追上 rear 时分不清是空还是满。
    q->capacity = maxSize + 1;
```


```c
    //此时malloc一块空间之后不会再用malloc
    q->data = (int*)malloc(sizeof(int) * q->capacity);
    if (q->data == NULL)
    {
        perror("malloc failed");
        return;
    }
```


```c
    q->front = 0;
    q->rear = 0;
```


```c
    printf("环形队列初始化成功，真实容量为%d，可装载人数为%d\n", q->capacity, maxSize);
}
```


- 为什么 `capacity = maxSize + 1`？

假设顾客说：“我要一个能装 4 个人的队列。”

1.  如果我们底层只申请 4 个位置的数组。当 4 个人全站满时，`rear`
```c
绕了一圈回到了起点，刚好和 `front` 重叠（`rear == front`）。
```

2.  但是，当队列空无一人时，`rear` 和 `front`
```c
也是重叠的（`rear == front`）。
```

3.  这样一来，程序就彻底抓瞎了！

所以，我们极其狡猾地向系统申请 **5** 个位置（`capacity = 5`）：

1.  我们依然只允许最多站 4 个人。
2.  这样，当站满 4 个人时，`rear` 和 `front`
```c
之间**永远会隔着一个空位**！它们永远不会重叠！
```

3.  只有当真正的空无一人时，它们才会重叠。

这就是计算机科学里极其经典的“牺牲空间换取逻辑清晰”的做法。

## 3.判空函数

```c
bool queueIsEmpty(CircularQueue* q)
{
    //当front和rear重合即一个人也没有为空
    if (q->front == q->rear)
    {
        return true;
    }
    else
    {
        return false;
    }
}
```


## 4.判满函数

```c
bool queueIsFull(CircularQueue* q)
{
    //【核心心法】：让rear在脑海里往前“试探性”地走一步。
    //如果往前走一步刚好撞上了front，说明转盘只剩最后那个“必须浪费的空位”了，队列已满！
    int nextRear = (q->rear + 1) % q->capacity;
```


```c
    if (nextRear == q->front)
    {
        return true;
    }
    else
    {
        return false;
    }
}
```


**为什么要进行取模运算？**

假设我们申请了一个 `capacity = 5` 的数组（下标是 0, 1, 2, 3, 4）。
我们的目标是：让游标从 0 走到 4 之后，下一步**瞬间穿越回 0**！

- **正常情况（没走到头）：** 假设现在
  `rear = 2`。往前走一步：`(2 + 1) % 5` ，3 除以 5，商是 0，余数是
  **3**。所以下一步去了下标 3。非常正常。
- **走到尽头的情况（见证奇迹的时刻）：** 假设现在
  `rear = 4`（已经站在物理数组的最后一个位置了）。往下走一步：`(4 + 1) % 5` ，5
  除以 5，商是 1，余数是 **0**！ **看！它完美地绕回了起点 0！**
  物理上是一条直线的数组，在逻辑上被这个 `%`
  号硬生生“掰弯”成了一个圆环！

所以，**在 `rear` 距离 `front`
还有一格距离的时候，我们就必须强行拉响警报：“满了满了！不能再放了！**

**rear指向那个是即将放入人的那个位置，而非最后一个人**

## 5.入列函数

把人放进空位，然后让游标往前挪一步。

```c
void queuePush(CircularQueue* q, int value)
{
    if (queueIsFull(q))
    {
        printf("队列已满，入列失败\n");
        return;
    }
```


```c
    //把value装进下标为q->value的位置
    q->data[q->rear] = value;
    //【核心】：工人 (rear) 往前走一步，寻找下一个空位
    //如果走到了数组的尽头，% 取模运算会把他瞬间传送回下标 0
    q->rear = (q->rear + 1) % q->capacity;
```


```c
    printf("%d入队成功\n", value);
}
```


## 6.出列函数

出队的逻辑几乎和入队是对称的，只是操作的游标换成了 `front`

```c
int queuePop(CircularQueue* q)
{
    if (queueIsEmpty(q))
    {
        printf("出列失败，队列为空\n");
        return -1;
    }
```


```c
    //拿到即将删除的那个值
    int popValue = q->data[q->front];
```


```c
    //核心：旅客(front) 拿完行李，往前走一步，准备接下一个。
    //同样，走到要 % 传送回起点。
    q->front = (q->front + 1) % q->capacity;
```


```c
    printf("%d出列成功\n", popValue);
    return popValue;
}
```


## 7.销毁队列

```c
void queueDestroy(CircularQueue* q)
{
    if (q->data != NULL)
    {
        free(q->data);
        q->data = NULL;
    }
```


```c
    q->front = 0;
    q->rear = 0;
    q->capacity = 0;
    printf("队列成功被销毁\n");
}
```


我们一开始用 `malloc`
申请了那个巨大的底层数组，关机的时候当然要还给操作系统。

## 8.不同文件代码的整合

CircularQueue.h

```c
#pragma once
```


```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>
```


```c
typedef struct CircularQueue
{
    int* data;//待会用malloc获取一块连续的数字空间
    int capacity;//记录这个队列的总容量
    int front;//队头游标，排在最前面的数据
    int rear;//队尾游标，永远指向下一个空位
}CircularQueue;
```


```c
//初始化环形队列
void queueInit(CircularQueue* q, int maxSize);
//判空函数
bool queueIsEmpty(CircularQueue* q);
//判满函数
bool queueIsFull(CircularQueue* q);
//入列函数
void queuePush(CircularQueue* q, int value);
//出列函数
int queuePop(CircularQueue* q);
//销毁队列
void queueDestroy(CircularQueue* q);
```


CircularQueue.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "CircularQueue.h"
```


```c
void queueInit(CircularQueue* q, int maxSize)
{
    //【核心心法】：你想装 maxSize 个人，我底层必须开辟 maxSize + 1 个位置！
    //那个多出来的位置，就是为了防止 front 追上 rear 时分不清是空还是满。
    q->capacity = maxSize + 1;
```


```c
    //此时malloc一块空间之后不会再用malloc
    q->data = (int*)malloc(sizeof(int) * q->capacity);
    if (q->data == NULL)
    {
        perror("malloc failed");
        return;
    }
```


```c
    q->front = 0;
    q->rear = 0;
```


```c
    printf("环形队列初始化成功，真实容量为%d，可装载人数为%d\n", q->capacity, maxSize);
}
```


```c
bool queueIsEmpty(CircularQueue* q)
{
    //当front和rear重合即一个人也没有为空
    if (q->front == q->rear)
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
bool queueIsFull(CircularQueue* q)
{
    //【核心心法】：让rear在脑海里往前“试探性”地走一步。
    //如果往前走一步刚好撞上了front，说明转盘只剩最后那个“必须浪费的空位”了，队列已满！
    int nextRear = (q->rear + 1) % q->capacity;
```


```c
    if (nextRear == q->front)
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
void queuePush(CircularQueue* q, int value)
{
    if (queueIsFull(q))
    {
        printf("队列已满，入列失败\n");
        return;
    }
```


```c
    //把value装进下标为q->value的位置
    q->data[q->rear] = value;
    //【核心】：工人 (rear) 往前走一步，寻找下一个空位
    //如果走到了数组的尽头，% 取模运算会把他瞬间传送回下标 0
    q->rear = (q->rear + 1) % q->capacity;
```


```c
    printf("%d入队成功\n", value);
}
```


```c
int queuePop(CircularQueue* q)
{
    if (queueIsEmpty(q))
    {
        printf("出列失败，队列为空\n");
        return -1;
    }
```


```c
    //拿到即将删除的那个值
    int popValue = q->data[q->front];
```


```c
    //核心：旅客(front) 拿完行李，往前走一步，准备接下一个。
    //同样，走到要 % 传送回起点。
    q->front = (q->front + 1) % q->capacity;
```


```c
    printf("%d出列成功\n", popValue);
    return popValue;
}
```


```c
void queueDestroy(CircularQueue* q)
{
    if (q->data != NULL)
    {
        free(q->data);
        q->data = NULL;
    }
```


```c
    q->front = 0;
    q->rear = 0;
    q->capacity = 0;
    printf("队列成功被销毁\n");
}
```


test.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "CircularQueue.h"
```


```c
int main()
{
    CircularQueue MyQueue;
    queueInit(&MyQueue, 5);
```


```c
    queuePush(&MyQueue, 1);
    queuePush(&MyQueue, 2);
    queuePush(&MyQueue, 3);
    queuePush(&MyQueue, 4);
    queuePush(&MyQueue, 5);
```


```c
    queuePush(&MyQueue, 1);
```


```c
    /*queuePop(&MyQueue);
    queuePop(&MyQueue);
    queuePop(&MyQueue);
    queuePop(&MyQueue);
    queuePop(&MyQueue);
```


```c
    queuePop(&MyQueue);*/
```


```c
    queueDestroy(&MyQueue);
    return 0;
}
```

