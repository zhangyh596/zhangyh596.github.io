---
title: 'C语言中的指针2（超详细带练）'
date: '2025-11-13T14:26:44+08:00'
draft: false
categories: ['C']
tags: ['指针', '数组', '指针与数组']
---
## 1.数组的地址

我们知道&arr\[0\],可以拿到数组第一个元素的地址，那arr的地址是什么呢，让我们运行来看一下

```c
#include <stdio.h>

int main()
{
    int arr[10] = { 1,2,3,4,5,6,7,8,9,10 };
    printf("%p\n", &arr[0]);
    printf("%p\n", arr);
    return 0;
}
```

![](/images/c/cb8be9970e7244989da13e2e66ab238d.png)

我们可以看到是一样的地址，也就是说数组名就是数组首元素的地址，但是我们要记住两个特例

第一个是sizeof（数组名），sizeof中单独放数组名，这里的数组名表示整个数组，计算的是整个数组的大小，单位是字节

第二个是&数组名，这里的数组名表示整个数组，取出的是整个数组的地址（虽然打印的地址都是数组首元素的地址，但如果是&arr+1跳过整个数组，而arr+1跳过的是一个元素，这里是有区别的）

## 2.指针访问数组

接下来我们可以用指针来打印我们的整个数组

```c
int main()
{
    int arr[10] = { 1,2,3,4,5,6,7,8,9,10 };
    int sz = sizeof(arr) / sizeof(arr[0]);
    int* p = arr;
    for (int i = 0; i < sz; i++)
    {
        scanf("%d", p + i);
    }
    for (int i = 0; i < sz; i++)
    {
        printf("%d ", *(p + i));
    }
    return 0;
}
```

实际上\*（p + i）等价于\* （arr +
i）等价于arr\[i\]等价于p\[i\]，数组名p和arr在这里是等价的

## 3.一维数组传参本质

数组名是数组首元素的地址，那么在数组传参的时候，传递的是数组名，也就是说本质上数组传参传递的是数组首元素的地址，接下来我们来看一段代码

```c
​​void test1(int arr[])
{
    printf("%d\n", sizeof(arr));
}
void test2(int* arr)
{
    printf("%d\n", sizeof(arr));
}
int main()
{
    int arr[10] = { 1,2,3,4,5,6,7,8,9,10 };
    printf("%d\n", sizeof(arr));
    test1(arr);
    test2(arr);
    return 0;
}
```

不妨猜一下这段代码的输出是什么（x86环境）

![](/images/c/d03c7e4907c3430ab447f602c4f90d35.png)

所以说函数形参部分理论上一个使用指针变量来接收首元素的地址，那么在函数内部我们写sizeof（arr）计算的是一个地址的大小（单位字节）而不是数组的大小（单位字节），真是一位函数的参数部分本质是指针，所以在函数的内部是没办法求数组元素个数的

## 4.二级指针

指针变量也是变量，变量就有地址，那指针变量的地址存放的地方其实就是二级指针

```c
int main()
{
    int a = 10;
    int* pa = &a;
    int** pa = &pa;
    return 0;
}
```

![](/images/c/1d73b6f675f14441b51304a4a1c05ca9.png))

指针数组的每个元素是地址，又可以指向一块区域

## 6.指针数组模拟二维数组


```c
int main()
{
    int arr1[] = { 1,2,3,4,5 };
    int arr2[] = { 6,7,8,9,10 };
    int arr3[] = { 11,12,13,14,15 };
    int* parr[3] = { arr1,arr2,arr3 };
    for (int i = 0; i < 3; i++)
    {
        for (int j = 0; j < 5; j++)
        {
            printf("%d ", parr[i][j]);
        }
        printf("\n");
    }
    return 0;
}
```

![](/images/c/ad3c9e2692c84977b6c0e5dc0c7e8296.png)

parr\[i\]访问的是parr数组的元素，parr\[i\]找到的数组元素指向整型一维数组，parr\[i\]\[j\]就是整型一维数组的元素

这只是模拟出了二维数组的效果并非真正的二维数组，因为每一行并非连续的

今天的指针内容到此为止，之后依旧会介绍剩下的内容
