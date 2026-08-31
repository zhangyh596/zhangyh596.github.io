---
title: 'C语言中的指针1（超详细带练）'
date: '2025-11-09T10:23:26+08:00'
draft: false
categories: ['C']
tags: ['指针', '地址', '取地址操作符', '解引用']
---
## 指针的定义（类比生活）

指针其实就是地址，能够让代码通过地址访问对应的变量。就比如生活当中，如果我们想要去找人，那他肯定要告诉我地址，这样我们才能找到他。知道了这个，接下来让我们深入了解一下指针吧！

## 1.   指针变量

### 取地址操作符（&）

首先我们要知道取地址操作符（&），这样子才可以取出地址，我们才能用指针

举例

    #include <stdio.h>

    int main()
    {
        int a = 10;
        &a;//取出a的地址
        printf("%p", &a);//打印a的地址
        return 0;
    }

虽然整型变量是四个字节，但只要知道第一个字节，就能获得四个字节。

### 解引用操作符（\*）

我们拿到地址想要存储到变量，就要用到\*，这也就是指针变量，用来存放地址，同时存放在指针变量中的值都会理解成地址

    int main()
    {
        int a = 10;
        printf("%d\n", a);
        int* p = &a;
        *p = 0;//这样就可通过指针更改a的值
        printf("%d\n", a);
        return 0;
    }

p左边写的是int\*，\*说明p是指针变量，而int锁门p指向的是整型类型的对象，比如如果是char类型

    char ch = 'w';
    char * pc = &ch;

同时要记住在x86（32位平台）下指针变量大小是四个字节，x64（64位平台）下指针变量大小是八个字节，和类型无关

## 2.const修饰指针变量

如果你希望变量不被修改就需要用到const


    int main()
    {
        int n = 10;
        int m = 20;
        int const* p = &n;//*p不能修改，p能修改
        printf("n的地址：%p\nm的地址：%p\n", &n, &m);
        printf("%p\n", p);//打印n的地址
        p = &m;//改变了p的指向
        printf("%p\n",p );//打印m的地址
        return 0;
    }

同时const如果放在\*的右边效果就不一样

    int main()
    {
        int n = 10;
        int m = 20;
        int* const p = &n;//*p能修改，p不能修改
        *p = 20;//改变n的值
        printf("%d", n);//n的值为20
        return 0;
    }

const int\* p和int const\*
p效果是一样的，同时如果在\*的左边和右边都加上const，\*p和p就都不能更改了

## 3.指针的运算

数组在内存是连续存放的，因此只要知道第一个元素的地址，就能依次找到后面的元素

![](/images/c/6ac5a92b917b4c798c91a9f209c035ae.png)

指针±整数意味着向后跳过其类型的字节，如果是int类型＋1意味向后跳过四个字节，也就是一个int类型的元素

    int main()
    {
        int arr[10] = { 1,2,3,4,5,6,7,8,9,10 };
        int* p = &arr[0];
        int i = 0;
        int sz = sizeof(arr) / sizeof(arr[0]);
        for (i = 0; i < sz; i++)
        {
            printf("%d ", *(p + i));//这样就可以通过指针打印整个数组
        }
        return 0;
    }

指针-指针就是统计指针之间的个数

下面让我们模拟实现strlen

    int my_strlen(char* s)
    {
        char* p = s;
        while (*p != '\0')
            p++;
        return p - s;
    }

    int main()
    {
        printf("%d", my_strlen("abc"));
        return 0;
    }

## 4.野指针

野指针成因可能是指针未初始化；指针越界访问；指针指向的空间释放

为了规避野指针我们就要将指针初始化；小心指针越界；指针变量不再使用时及时用NULL，指针使用之前检查有效性

## 5.指针的使用

第二种使用strlen的方式

    int my_strlen(const char* str)
    {
        int count = 0;
        while (*str)//str不为\0则一直进行下去
        {
            count++;
            str++;
        }
        return count;
    }

    int main()
    {
        int len = my_strlen("abc");
        printf("%d", len);
        return 0;
    }

### 传值调用

把变量本身传给函数，这就是传值调用

    ​​​​void swap(int x, int y)
    {
        int tmp = x;
        x = y;
        y = tmp;
    }
    int main()
    {
        int a = 10;
        int b = 20;
        printf("交换前：a=%d b=%d\n",a,b);
        swap(a, b);
        printf("交换后：a=%d b=%d\n",a,b);
        return 0;
    }

![](/images/c/e7477222a4a945519869855bc6a64e86.png)

这样却并没有改变a，b的值，这说明实参传递给形参的时候，形参会单独创建一份临时空间来接受实参，对形参的修改不影响实参

### 传址调用 

将地址传递给函数就叫做传址调用，这样可以真正交换a，b，下面我们来实现一下

    ​​void swap(int *x, int *y)
    {
        int tmp = 0;
        tmp = *x;
        *x = *y;
        *y = tmp;
    }
    int main()
    {
        int a = 10;
        int b = 20;
        printf("交换前：a=%d b=%d\n",a,b);
        swap(&a, &b);
        printf("交换后：a=%d b=%d\n",a,b);
        return 0;
    }

![](/images/c/0750fe2bfdae441298c6cc9aca3e3168.png)

这样就真正完成了交换

**接下来还会分享更多的指针知识，关注不迷路哦**
