---
title: "动态内存管理（C语言超详细）"
date: "2025-12-18T14:53:40+08:00"
draft: false
categories: ["C"]
tags: ["动态内存", "malloc", "free", "calloc", "realloc"]
---
## 1.为什么要有动态内存管理

- 空间开辟大小是固定的
- 数组在声明的时候必须指定数组的长度，数组空间一旦确定了大小就不能再调整（C99后引入了变长数组（VS2022不支持），但变长数组是在栈上分配的，且不支持malloc那样的动态扩容，缩容操作）

但是对于空间的需求，有时我们需要的空间大小在程序运行的时候才知道，那么上述开辟方式就不能满足要求了

这样引入动态内存开辟，自己申请和释放空间就比较灵活了（头文件为<stdlib.h>）

## 2.malloc和free

![](/images/c/2336a936309e443a942abc8283408d07.png)

- size是要分配的内存字节数，类型是size_t
- 返回值：分配成功返回指向该内存起始地址的void\*指针，需强制类型转换为对于数据类型指针；分配失败返回NULL

![](/images/c/f2c1de7fbac343b0ab7c84a0d4c2a1f7.png)

- 专门用于释放由动态内存分配开辟的堆内存，避免内存泄露
- ptr指向堆内存的起始地址，类型参数为void\*，无需强制类型转换
- 无返回值

故二者一般是成对出现的，使用free释放后也最后将指针置为NULL避免野指针

```c
#include <stdio.h>
#include <stdlib.h>
```


```c
int main()
{
    int n = 5;
    int* arr = (int*)malloc(n * sizeof(int));
    if (arr == NULL)//检查是否为空指针
    {
        printf("内存分配失败\n");
        return 1;
    }
    for (int i = 0; i < n; i++)
    {
        arr[i] = i + 1;
    }
    for (int i = 0; i < n; i++)
    {
        printf("%d ", arr[i]);
    }
    free(arr);
    arr = NULL;
    return 0;
}
```


## 3.calloc和realloc

![](/images/c/6585223d5c5d43ddae92e2101fdc7139.png)

- num是要分配的元素个数
- size是每个元素的字节大小
- 返回值：分配成功返回指向起始地址的void\*指针，需强制类型转换；分配失败返回NULL
- calloc会将开辟的内存全都初始化为0（与malloc的区别，其他功能基本相同）

<!-- -->

```c
#include <stdio.h>
#include <stdlib.h>
```


```c
int main()
{
    int n = 5;
    int* arr = (int*)calloc(n, sizeof(int));
    if (NULL == arr)
    {
        printf("内存分配失败\n");
        return 1;
    }
    for (int i = 0; i < n; i++)
    {
        printf("%d ", arr[i]);
    }
    free(arr);
    arr = NULL;
    return 0;
}
```


![](/images/c/7108fb9d6ba54d1297cfbb46f6297743.png)

- realloc是用于调整已分配堆内存的大小，可实现扩容和缩容，是实现弹性数据结构的核心函数
- ptr是指向之前由malloc，calloc，realloc分配的内存指针
- size是调整后内存块的新字节大小
- 返回值：成功返回调整后内存起始地址的void\*指针，失败返回NULL，且原内存块不会被释放

这个函数在调整原内存空间大小的基础上还会将原来内存中的数据移动到新的空间

realloc在调整内存空间存在两种情况

情况1：原有空间之后有足够大的内存（扩展内存就直接在原有空间之后追加空间，原来空间的数据不发生变化）

情况2：原有空间之后没有足够大的内存（扩展方法是在堆空间另外找一个合适大小的连续空间来使用，这样函数返回的是一个新的内存地址）

故我们要用一个新的指针来接收

```c
#include <stdio.h>
#include <stdlib.h>
```


```c
int main()
{
    int n = 3;
    int* arr = (int*)malloc(n * sizeof(int));
    if (NULL == arr)
    {
        printf("内存分配失败\n");
        return 1;
    }
    for (int i = 0; i < n; i++)
    {
        arr[i] = i + 1;
    }
    printf("扩容前的数组：");
    for (int i = 0; i < n; i++)
    {
        printf("%d ", arr[i]);
    }
    printf("\n");
    int new_n = 5;
    //错误写法
    //int* arr = (int*)realloc(arr,new_n*sizeof(int));
    
    //正确写法
    int* new_arr = (int*)realloc(arr, new_n * sizeof(int));
    if (NULL == new_arr)
    {
        printf("内存分配失败\n");
        free(arr);//原内存手动释放
        arr = NULL;
        return 1;
    }
    arr = new_arr;
    for (int i = n; i < new_n; i++)
    {
        arr[i] = i + 1;
    }
    printf("扩容后的数组：");
    for (int i = 0; i < new_n; i++)
    {
        printf("%d ", arr[i]);
    }
    free(arr);
    arr = NULL;
    return 0;
}
```


## 4.常见的动态内存分配错误

### （1）对NULL指针的解引用操作

错误示范


```c
int main()
{
    int* p = (int*)malloc(INT_MAX);//动态内存无法开辟过大的空间导致返回空指针
    *p = 20;//无法对空指针解引用
    free(p);
    return 0;
}
```


### （2）对动态开辟空间的越界访问

错误示范

```c
void test()
{
    int i = 0;
    int* p = (int*)malloc(10 * sizeof(int));
    if (NULL == p)
    {
        printf("内存分配失败\n");
    }
    for (int i = 0; i <= 10; i++)//多访问了一个元素
    {
        p[i] = i;
    }
    free(p);
    p = NULL;
}
```


### （3）对非动态开辟内存使用free函数释放

错误示范

```c
void test()
{
    int a = 10;
    int* p = &a;
    free(p);
}
```


### （4）使用free函数释放一块动态开辟内存的一部分

错误示范

```c
void test()
{
    int* p = (int*)malloc(100);
    p++;
    free(p);//p不再指向动态内存的起始地址
}
```


### （5）使用同一块内存多次释放

错误示范

```c
void test()
{
    int* p = (int*)malloc(100);
    free(p);
    free(p);
}
```


### （6）动态开辟内存忘记释放（内存泄露）

错误示范

```c
void test()
{
    int* p = (int*)malloc(100);
    if (NULL != p)
    {
        *p = 20;
    }
}
```

