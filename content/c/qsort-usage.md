---
title: 'qsort的运用（C语言指针扩展）'
date: '2025-11-23T10:40:20+08:00'
draft: false
categories: ['C']
tags: ['qsort', '排序', '函数指针']
---
为了引入qsort这一排序，我会先带大家逐步深入，同时内容有关指针，希望大家没看我之前写的指针内容的可以先去看一下才能达到更好的效果

## 1.冒泡排序（引入排序内容）

首先什么是冒泡排序呢？

核心思想其实就是将两两相邻的元素进行比较

一共就需要进行元素个数-1趟，接下来看一下代码实现（附有注释）

```c
#include <stdio.h>

void bubble_sort(int arr[], int sz)
{
    int i = 0;
    for (int i = 0; i < sz - 1; i++)//只需要进行sz-1趟即可完成
    {
        for (int j = 0; j < sz - 1 - i; j++)//j只需要后面的sz-1-i个数比,前面的数已经有序
        {
            if (arr[j] > arr[j + 1])
            {
                int tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
}
int main()
{
    int arr[10] = { 2,5,9,7,8,6,4,3,1,10 };
    int sz = sizeof(arr) / sizeof(arr[0]);
    printf("交换前:");
    for (int i = 0; i < sz; i++)
    {
        printf("%d ", arr[i]);
    }
    printf("\n");
    bubble_sort(arr, sz);//冒泡排序
    printf("交换后:");
    for (int i = 0; i < sz; i++)
    {
        printf("%d ", arr[i]);
    }
}
```

![](/images/c/d5e91a43dda148d7ac92aec723a00aae.png)

这样就实现了我们的冒泡排序，那么有没有什么办法可以优化呢

比如某一趟过后已经有序，我们没必要再去进行剩余元素的比较，这时我们可以引入flag来标记一下

```c
#include <stdio.h>

void bubble_sort(int arr[], int sz)
{
    int i = 0;
    for (int i = 0; i < sz - 1; i++)//只需要进行sz-1趟即可完成
    {
        int flag = 1;//假设已经有序
        for (int j = 0; j < sz - 1 - i; j++)//j只需要后面的sz-1-i个数比,前面的数已经有序
        {
            if (arr[j] > arr[j + 1])
            {
                flag = 0;//发送交换就把flag值赋成0，无序
                int tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
        if (flag)//如果flag是1，即flag为真表示已经有序
            break;
    }
}
int main()
{
    int arr[10] = { 2,5,9,7,8,6,4,3,1,10 };
    int sz = sizeof(arr) / sizeof(arr[0]);
    printf("交换前:");
    for (int i = 0; i < sz; i++)
    {
        printf("%d ", arr[i]);
    }
    printf("\n");
    bubble_sort(arr, sz);//冒泡排序
    printf("交换后:");
    for (int i = 0; i < sz; i++)
    {
        printf("%d ", arr[i]);
    }
}
```

## 2.回调函数

有了冒泡排序的知识，接下来我想给大家讲一下回调函数

回调函数又是什么呢？

回调函数其实就是一个通过函数指针调用的函数

就是你把函数的地址作为参数传递给另一个函数，当这个地址（指针）被用来调用其所指向的函数时，被调用的函数就是回调函数

回顾一下我们之前所学的转移表就可以写成回调函数的形式

```c
int add(int x, int y)
{
    return x + y;
}
int sub(int x, int y)
{
    return x - y;
}
int mul(int x, int y)
{
    return x * y;
}
int div(int x, int y)
{
    return x / y;
}
//回调函数
void calc(int(*pf)(int x, int y))
{
    int ret = 0;
    int x, y;
    printf("输入操作数：");
    scanf("%d %d", &x, &y);
    ret = (*pf)(x, y);
    printf("ret = %d\n", ret);
}
int main()
{
    int x, y;//操作数
    int input = 1;//进行选择
    int ret = 0;//存放结果
    do
    {
        //打印菜单
        printf("***************************\n");
        printf("**1.add   2.sub  **********\n");
        printf("**3.mul   4.div  **********\n");
        printf("**0.exit         **********\n");
        printf("***************************\n");
        printf("请选择:");
        scanf("%d", &input);
        switch (input)
        {
        case 1:
            calc(add);
            break;
        case 2:
            calc(sub);
            break;
        case 3:
            calc(mul);
            break;
        case 4:
            calc(div);
            break;
        case 0:
            printf("退出\n");
            break;
        default:
            printf("请重新选择\n");
            break;
        }
    } while (input);
    return 0;
}
```

![](/images/c/bae869115dd64d4f80e3f48749a285d1.png)

![](/images/c/51013c2b37ce4e6a99ccd632f654fe88.png))

base：要排序的数组的首地址

num：数组的元素个数

size：单个元素的字节大小（用sizeof获取）

compar：比较函数的指针（这其实就是一个回调函数）

由于我们还不知道接受的元素是什么类型，所以用void来接收，而实际我们会知道元素是什么类型，然后我们就在compar中就需要使用强制类型转换

#### 排序整型数据

接下来我们就用qsort实现排序一个整型数组

```c
#include <stdio.h>

int compar(const void* p1, const void* p2)
{
    return (*(int*)p1) - (*((int*)p2));
}
int main()
{
    int arr[] = { 5,4,6,2,3,7,8,9,1,10 };
    int sz = sizeof(arr) / sizeof(arr[0]);
    qsort(arr, sz, sizeof(arr[0]),compar);
    for (int i = 0; i < 10; i++)
    {
        printf("%d ", arr[i]);
    }
    return 0;
}
```

#### 排序结构体的数据

##### 排序结构体中的整型

```c
#include <stdio.h>
//创建结构体
struct stu
{
    char name[20];
    int age;
};
//创建通过age比较的回调函数
int compar_by_age(const void* p1, const void* p2)
{
    return ((struct stu*)p1)->age - ((struct stu*)p2)->age;
}
int main()
{
    struct stu s[] = { {"张三",20},{"李四",30},{"王五",18} };
    int sz = sizeof(s) / sizeof(s[0]);
    qsort(s, sz, sizeof(s[0]), compar_by_age);
    return 0;
}
```

排序前![](/images/c/c7f0494df6de47789c5c9e381b520e68.png)

排序后![](/images/c/48aaa42d7df549f7b5fb04fc60bf24e3.png)

##### 排序结构体中的字符

```c
#include <stdio.h>
//创建结构体
struct stu
{
    char name[20];
    int age;
};
//创建通过name比较的回调函数
int compar_by_name(const void* p1,const void* p2)
{
    return strcmp(((struct stu*)p1)->name, ((struct stu*)p2)->name);
}
int main()
{
    struct stu s[] = { {"张三",20},{"李四",30},{"王五",18} };
    int sz = sizeof(s) / sizeof(s[0]);
    qsort(s, sz, sizeof(s[0]), compar_by_name);
    return 0;
}
```

排序前![](/images/c/39987b1e89954667af369033f8e945ff.png)

排序后![](/images/c/2b2aff3a11cc45268067b6455cfd1f96.png)

## 4.模拟实现qsort（采用冒泡排序思想）

因为qsort可以对任意类型排序，而我们预先又不知道排序的是什么类型，于是我们选择强制转换为char类型按逐字节偏移

```c
#include <stdio.h>

//回调函数
int compar(const void* p1, const void* p2)
{
    //将void*强制转换为比较的类型
    return (*(int*)p1 - *(int*)p2);
}
//交换函数（任意类型都可以，所以用char类型逐字节交换）
//size用于接收每个元素的大小
void swap(void* p1, void* p2, size_t size)
{
    for (int i = 0; i < size; i++)
    {
        char tmp = *((char*)p1 + i);
        *((char*)p1 + i) = *((char*)p2 + i);
        *((char*)p2 + i) = tmp;
    }
}
//参数类型和qsort一样
void my_qsort(void* base, size_t count, size_t size, int(*cmp)(void*, void*))
{
    //排序轮数
    for (int i = 0; i < count - 1; i++)
    {
        //比较相邻元素
        for(int j = 0;j<count-1-i;j++)
        {
            //(char *)base + j*size 是第j个元素的地址
            //(char *)base + (j + 1)*size 是第j+1个元素的地址

            //使用比较函数判断是否需要交换
            if (cmp((char*)base + j * size, (char*)base + (j + 1) * size) > 0)
            {
                //需要交换则手动交换
                swap((char*)base + j * size, (char*)base + (j + 1) * size, size);
            }
        }
    }
}
int main()
{
    //排序一个整型数组
    int arr[] = { 4,5,3,2,8,9,6,1,10,7 };
    int sz = sizeof(arr) / sizeof(arr[0]);
    my_qsort(arr, sz, sizeof(arr[0]), compar);
    for (int i = 0; i < sz; i++)
    {
        printf("%d ", arr[i]);
    }
    return 0;
}
```

解析关键点

void\* 是通用指针，可以指向任何类型数据

地址计算技巧：通过基地址+索引\*元素大小来访问任意类型的元素

如（char\*）base + j\*size

（1）将base转换为char\*

（2）j\*size计算偏移量

（3）得到第j个元素的地址
