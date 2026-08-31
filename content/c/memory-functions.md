---
title: "内存函数（C语言超详细）"
date: "2025-11-29T22:33:45+08:00"
draft: false
categories: ["C"]
tags: ["内存函数", "memcpy", "memmove", "memset", "memcmp"]
---
## 1.memcpy的使用

![](/images/c/c761c675c48a4ffab4db0412ce996d50.png)

- memcpy函数可以从source的位置向后复制num个字节的数据到destination指向的内存位置
- destination是目标内存地址
- source是源内存地址
- num是要复制的字节数
- 确保目标内存足够大，遇到‘\0’不会停下来
- 避免目标内存和源内存重叠的情况
- 返回值是指向destination的指针

复制字符串 （和strcpy类似）

```c
#include <stdio.h>
#include <string.h>
```


```c
int main()
{
    char src[] = "hello,world";
    char dest[20];
    memcpy(dest, src, strlen(src) + 1);
    printf("%s", dest);
    return 0;
}
```


复制整型数组（和strcpy不同，memcpy可以复制任意类型）

```c
#include <stdio.h>
#include <string.h>
```


```c
int main()
{
    int src[5] = { 1,2,3,4,5 };
    int dest[5];
    memcpy(dest, src, sizeof(src));
    for (int i = 0; i < 5; i++)
    {
        printf("%d ", dest[i]);
    }
    return 0;
}
```


复制结构体

```c
#include <stdio.h>
#include <string.h>
```


```c
typedef struct 
{
    int id;
    char name[20];
    int age;
}student;
```


```c
int main()
{
    student s1 = { 2530,"zhangsan",18 };
    student s2;
    memcpy(&s2, &s1, sizeof(student));
    printf("%d %s %d", s2.id, s2.name, s2.age);//学生2复制成功
    return 0;
}
```


### memcpy的模拟实现


```c
#include <stdio.h>
void* my_memcpy(void* dest, const void* src, size_t n)
{
    void* ret = dest;//保留初始位置
    while (n--)
    {
        *(char*)dest = *(char*)src;
        dest = (char*)dest + 1;
        src = (char*)src + 1;
    }
    return ret;
}
```


```c
typedef struct 
{
    int id;
    char name[20];
    int age;
}student;
```


```c
int main()
{
    student s1 = { 2530,"zhangsan",18 };
    student s2;
    my_memcpy(&s2, &s1, sizeof(student));
    printf("%d %s %d", s2.id, s2.name, s2.age);//学生2复制成功
    return 0;
}
```


## 2.memmove的使用

![](/images/c/99a15d23217a4f45b8165222aa4b7267.png)

- memmove是在memcpy的基础上可以处理重叠空间的复制，可以替换memcpy
- 返回值也是指向destination的指针

<!-- -->

```c
#include <stdio.h>
#include <string.h>
```


```c
int main()
{
    int arr1[] = { 1,2,3,4,5,6,7,8,9,10 };
    memmove(arr1 + 2, arr1, 20);//在arr1+2的位置向后复制初始arr1的后20个字节
    for (int i = 0; i < 10; i++)
    {
        printf("%d ", arr1[i]);//得到1 2 1 2 3 4 5 8 9 10
    }
    return 0;
}
```


![](/images/c/e76c0135f31c46c985af3c0674f0c777.png)

而如果是memcpy则结果可能不一样（取决于编译环境）这是因为复制过程可能会覆盖原始数组，从而在复制过程中得到不一样的结果

### memmove的模拟实现

需要分类讨论是从前向后复制还是从后向前复制

比较地址的大小分类讨论


```c
#include <stdio.h>
```


```c
void* my_memmove(void* dest, const void* src, size_t n)
{
    void* ret = dest;
    if (dest <= src)//如果dest的地址在src前面，从前向后复制（重叠时）
    {
        while (n--)
        {
            *(char*)dest = *(char*)src;
            dest = (char*)dest + 1;
            src = (char*)src + 1;
        }
    }
    else//如果dest的地址在src后面，从后面往前面复制（重叠时）由于不重叠时两种方式都可以，所以归类到else里面
    {
        dest = (char*)dest + n - 1;
        src = (char*)src + n - 1;
        while (n--)
        {
            *(char*)dest = *(char*)src;
            dest = (char*)dest - 1;
            src = (char*)src - 1;
        }
    }
    return ret;
}
int main()
{
```


```c
    int arr1[] = { 1,2,3,4,5,6,7,8,9,10 };
    memmove(arr1 + 2, arr1, 20);//在arr1+2的位置向后复制初始arr1的后20个字节
    for (int i = 0; i < 10; i++)
    {
        printf("%d ", arr1[i]);//得到1 2 1 2 3 4 5 8 9 10
    }
    return 0;
}
```


## 3.memset的使用

![](/images/c/e32c1270d59947ffacfaf0e832ad88ca.png)

- memset函数将内存前num个字节设置成特定值
- ptr是指向要填充的内存块的指针
- value是要设置的值
- num是要设置的字节数
- 返回值是指向目标内存块（ptr）的指针

如想要初始化一个全为0的数组

```c
#include <stdio.h>
#include <string.h>
```


```c
int main()
{
    int arr[10];
    memset(arr, 0, sizeof(arr));
    for (int i = 0; i < 10; i++)
    {
        printf("%d ", arr[i]);
    }
    return 0;
}
```


## 4.memcmp的使用

![](/images/c/9e94edf1cfdc4d5aa48251bfc1899fe7.png)

- memcmp可以比较两个内存块的前num个字节
- ptr1是指向第一个内存块的指针
- ptr2是指向第二个内存块的指针
- num是要比较的字节数
- 返回值 <0(ptr1小于ptr2) =0(ptr1等于ptr2) >0(ptr1>ptr2)

<!-- -->

```c
#include <stdio.h>
#include <string.h>
```


```c
int main()
{
    int arr1[] = { 1,2,3,4,5 };
    int arr2[] = { 1,2,3,4,6 };
```


```c
    int ret = memcmp(arr1, arr2, sizeof(arr1));
    printf("%d", ret);
    return 0;
}
```


memcmp可以比较任意类型，而strcmp只可以比较字符串

### memcmp的模拟实现

```c
int my_memcmp(const void* ptr1, const void* ptr2, size_t n)
{
    if (n == 0)
    {
        return 0;
    }
    const char* p1 = (const char*)ptr1;
    const char* p2 = (const char*)ptr2;
    for (int i = 0; i < n; i++)
    {
        if (p1[i] != p2[i])
        {
            return (int)(p1[i]) - (int)(p2[i]);
        }
    }
    return 0;
}
int main()
{
        int arr1[] = { 1,2,3,4,5 };
    int arr2[] = { 1,2,3,4,6 };
```


```c
    int ret = my_memcmp(arr1, arr2, sizeof(arr1));
    printf("%d", ret);
    return 0;
}
```

