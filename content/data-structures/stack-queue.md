---
title: "栈和队列（C语言底层实现栈）"
date: "2026-03-13T23:40:27+08:00"
draft: false
categories: ["数据结构"]
tags: ["栈", "队列", "栈和队列"]
---
栈的核心思想就是“后进先出” (LIFO - Last In First
Out)，就像我们在桌子上叠盘子一样：最后放上去的盘子，必须最先拿下来。

## 1.定义栈的结构体

我们可以把“栈”想象成一个**竖着的薯片桶**。
你只能从最上面放薯片进去（入栈），也只能从最上面拿薯片出来（出栈）。

```c
typedef struct
{
    int capacity;//薯片桶的长度（最大容量）
    int top;//记录当前位置
    int* data;//真正放薯片的空间（指针）
}Stack;
```


- ##### **`int capacity;`（容量）**

很好理解，你总得知道这个薯片桶**最多能装多少片薯片**吧？ 如果
`capacity = 10`，那就说明这个桶最多装 10
个数据。超过了就会溢出（报错）。所以我们需要这个变量来记住桶的大小。

- ##### `int top;`（栈顶游标）

这是一个非常关键的变量。你可以把它想象成**一根悬在薯片桶外面的手指**。桶是空的时候，这根手指指着最底下，也就是
`-1` 的位置（因为数组的第一层是 0，-1 代表连第 0 层都没东西）。放进 1
片薯片，手指向上移一格，指向 `0`。再放 1 片薯片，手指向上移一格，指向
`1`。

如果你要拿走一片薯片，你不需要去销毁那片薯片，你只需要**把手指向下移一格**。下次再放新薯片的时候，直接把原来的位置覆盖掉就行了。
`top`
存在的意义，就是永远告诉你：**现在最新放进去的那个数据，在第几层？**

- ##### `int *data;`（指向数据的指针）

你可能会问：既然用来存数据，为什么不直接写成一个数组，比如
`int data[100];` 呢？如果写成
`int data[100];`，这就叫**写死了**。这意味着你造出来的每一个薯片桶，不管里面装了几片薯片，它永远占用
100 个位置的空间。太浪费内存了！而写成 `int *data;`（带个星号 `*`
的指针），它就像是一把**万能钥匙**。它暂时不占用巨大空间，只是准备好去指向一块未来的空间。等到程序运行的时候，用户说“我要一个装
3 个数据的栈”，我们就去内存里临时租一个大小为 3 的空间，把钥匙交给
`data`；用户说“我要装 10000 个”，我们就去租大小为 10000
的空间。这就叫**动态分配**，极其灵活！

## 2.初始化栈

```c
void stackInit(Stack* s, int capacity)
{
    s->data = (int*)malloc(capacity * sizeof(int));
```


```c
    if (s->data == NULL)
    {
        perror("malloc failed");
        exit(1);//强行退出
    }
```


```c
    s->top = -1;//说明现在是空的
    s->capacity = capacity;//记住最大容量
}
```


`top` 代表最新放进去的数据在第几层。 现在桶是空的，连第 0
层（数组的第一个位置）都还没有放东西，所以我们把指示手指（`top`）指向
`-1`。这明确告诉所有人：**这个栈现在是空的！**

## 3.判断栈是否已满（辅助函数）

```c
bool stackIsFull(Stack* s)
{
    // 逻辑：如果游标 top 已经到了数组的最后一个合法下标，就是满了。
    // 因为数组下标从 0 开始，所以最大容量为 capacity 时，最后一个下标是 capacity - 1。
    if (s->top == s->capacity - 1)
    {
        return true;
    }
    else
    {
        return false;
    }
}
```


## 4.压栈（入栈）

当我们要往“薯片桶”里塞数据时，逻辑顺序必须是这样的：

1.  **看一眼桶满了没**。满了就不能硬塞，否则数组越界（内存溢出）。
2.  **把“记号笔”（`top` 游标）往上移动一格**，指向一个新的空位置。
3.  **把数据放进这个新位置**。

<!-- -->

```c
void stackPush(Stack* s, int value)
{
    if (stackIsFull(s))
    {
        printf("入栈失败，栈已经满了\n");
        return;
    }
```


```c
    //移动游标并赋值
    s->top++;
    s->data[s->top] = value;
}
```


最容易犯的错误是先赋值，再移动 `top`（写成
`s->data[s->top] = value; s->top++;`）。 为什么那是错的？
回忆一下，我们初始化的时候，`s->top` 是 `-1`。数组是没有 `-1`
这个下标的！ 所以，当我们要放第一个元素时，必须**先让
`top++`**，也就是让 `top` 变成 `0`，然后再把数据放到 `s->data[0]`
的位置。这个逻辑顺序是雷打不动的。

## 5.判断栈是否为空（辅助函数）

```c
bool stackIsEmpty(Stack* s)
{
    // 逻辑：如果游标 top 还是当初初始化的 -1，说明里面一个数据都没有
    if (s->top == -1)
    {
        return true;
    }
    else
    {
        return false;
    }
}
```


## 6.弹栈（出栈）

拿数据的逻辑刚好反过来：

1.  **看一眼桶是不是空的**。空的就没东西可拿，不能硬拿，否则也是数组越界。
2.  **把当前 `top` 指向的那个数据拿出来，记在心里**。
3.  **把“记号笔”（`top` 游标）往下移动一格**。
4.  **把刚才记在心里的数据交出去**。

<!-- -->

```c
int stackPop(Stack* s)
{
    if (stackIsEmpty(s))
    {
        printf("出栈失败，栈是空的\n");
        return -1;
    }
```


```c
    //先把栈顶的数据拿出来存到一个临时变量里
    int popValue = s->data[s->top];
    s->top--;//游标往下走一格，代表原来的栈顶元素被“抛弃”了
    return popValue;//把取出的数据返回给调用者
}
```


**注意看第三步 `s->top--;`。我们有没有去写类似 `s->data[s->top] = 0;`
这样的代码去清空内存？**没有！**
在底层的数组里，那个被弹出的数据其实**还在原来的位置待着。但是，对于我们这个栈来说，它已经“不存在”了。
因为栈的所有操作都只认 `top`
这个游标。游标降下去了，就算那个内存里还有旧数据，下次我们执行 `Push`
的时候，`top++` 会再次指向上来，用新的数据**无情地覆盖掉**它。
这种做法极大地提升了程序的运行效率，因为我们省去了清理内存的无用功。

## 7.销毁栈

```c
void stackDestroy(Stack* s)
{
    if (s->data != NULL)
    {
        free(s->data);
        s->data = NULL;
    }
```


```c
    //把游标和容量都恢复到初始状态
    s->top = -1;
    s->capacity = 0;
    printf("栈成功销毁\n");
}
```


## 8.拓展（实现扩容版本）

有人会想当满了以后我们其实可以对栈扩容，这样就需要用到realloc

`可以把realloc` 想象成一家提供“无缝换大房”服务的搬家公司。
它的用法是：`realloc(旧房子的钥匙, 新房子需要的总大小)`。

它在底层会做三件事：去内存里找一块你要求的**新大小**的空间。自动把你旧空间里的数据**原封不动地搬过去**。自动把旧空间**销毁（free）**，然后把新空间的钥匙交给你。

我们也仅仅需要更改一下我们的入栈函数

```c
void stackPush(Stack* s, int value)
{
    if (stackIsFull(s))
    {
        int newCapacity = s->capacity * 2;
        //我们要先用一个临时钥匙 (newData) 来接新房子的钥匙
        int* newData = (int*)realloc(s->data, newCapacity * sizeof(int));
```


```c
        if (newData == NULL)
        {
            perror("realloc failed");
            return;//放弃这次入栈
        }
```


```c
        s->data = newData;
        s->capacity = newCapacity;
        printf("栈已扩容，新容量为:%d\n", s->capacity);
    }
```


```c
    //移动游标并赋值
    s->top++;
    s->data[s->top] = value;
    printf("入栈成功:%d\n", value);
}
```


## 9.完整代码

Stack.h

```c
#pragma once
```


```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
```


```c
typedef struct
{
    int capacity;//薯片桶的长度（最大容量）
    int top;//记录当前位置
    int* data;//真正放薯片的空间（指针）
}Stack;
```


```c
//初始化栈
void stackInit(Stack* s, int capacity);
//判断栈是否已满（辅助函数）
bool stackIsFull(Stack* s);
//入栈
void stackPush(Stack* s, int value);
//判断栈是否为空（辅助函数）
bool stackIsEmpty(Stack* s);
//出栈
int stackPop(Stack* s);
//销毁栈
void stackDestroy(Stack* s);
```


Stack.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "Stack.h"
```


```c
// s 代表我们要操作的那个栈（通过指针传进来）
// capacity 是用户告诉我们，这个栈要多大
void stackInit(Stack* s, int capacity)
{
    s->data = (int*)malloc(capacity * sizeof(int));
```


```c
    if (s->data == NULL)
    {
        perror("malloc failed");
        exit(1);//强行退出
    }
```


```c
    s->top = -1;//说明现在是空的
    s->capacity = capacity;//记住最大容量
    printf("成功初始化栈\n");
}
```


```c
bool stackIsFull(Stack* s)
{
    // 逻辑：如果游标 top 已经到了数组的最后一个合法下标，就是满了。
    // 因为数组下标从 0 开始，所以最大容量为 capacity 时，最后一个下标是 capacity - 1。
    if (s->top == s->capacity - 1)
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
void stackPush(Stack* s, int value)
{
    if (stackIsFull(s))
    {
        int newCapacity = s->capacity * 2;
        //我们要先用一个临时钥匙 (newData) 来接新房子的钥匙
        int* newData = (int*)realloc(s->data, newCapacity * sizeof(int));
```


```c
        if (newData == NULL)
        {
            perror("realloc failed");
            return;//放弃这次入栈
        }
```


```c
        s->data = newData;
        s->capacity = newCapacity;
        printf("栈已扩容，新容量为:%d\n", s->capacity);
    }
```


```c
    //移动游标并赋值
    s->top++;
    s->data[s->top] = value;
    printf("入栈成功:%d\n", value);
}
```


```c
bool stackIsEmpty(Stack* s)
{
    // 逻辑：如果游标 top 还是当初初始化的 -1，说明里面一个数据都没有
    if (s->top == -1)
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
int stackPop(Stack* s)
{
    if (stackIsEmpty(s))
    {
        printf("出栈失败，栈是空的\n");
        return -1;
    }
```


```c
    //先把栈顶的数据拿出来存到一个临时变量里
    int popValue = s->data[s->top];
    s->top--;//游标往下走一格，代表原来的栈顶元素被“抛弃”了
    return popValue;//把取出的数据返回给调用者
}
```


```c
void stackDestroy(Stack* s)
{
    if (s->data != NULL)
    {
        free(s->data);
        s->data = NULL;
    }
```


```c
    //把游标和容量都恢复到初始状态
    s->top = -1;
    s->capacity = 0;
    printf("栈成功销毁\n");
}
```


test.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "Stack.h"
```


```c
int main()
{
    Stack myStack;
    stackInit(&myStack, 3);
```


```c
    stackPush(&myStack, 10);
    stackPush(&myStack, 20);
    stackPush(&myStack, 30);
    stackPush(&myStack, 40);
```


```c
    while (!stackIsEmpty(&myStack))
    {
        printf("出栈数据：%d\n", stackPop(&myStack));
    }
    stackPop(&myStack);
```


```c
    stackDestroy(&myStack);
    return 0;
}
```

