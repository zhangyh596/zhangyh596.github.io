---
title: 'C语言中的指针3（超详细带练）'
date: '2025-11-20T16:02:40+08:00'
draft: false
categories: ['C']
tags: ['指针', '函数指针', '回调函数']
---
## 1.字符指针变量

字符指针就是char\*，使用方法如下，和前面所讲的整型指针没有什么区别

```c
int main()
{
    char ch = 'a';
    char* pc = &ch;
    *pc = 'b';
    printf("%c", ch);
    return 0;
}
```

下面这种使用方法就有所区别，叫做常量字符串


```c
int main()
{
    const char* pstr = "hello,world";
    printf("%s\n", pstr);
    return 0;
}
```

有人可能会认为是把一个字符串存放到了pstr指针变量里面，而事实并非如此，本质上把字符串hello,world首字符的地址放到了pstr中

**！！！如果创建两个相同的常量字符串并不会开辟多个内存，而是存放到一个内存区域，当几个指针指向同一个字符串的时候，他们实际会指向同一块内存**

## **2.数组指针变量**

前面我们讲过指针数组，指针数组是一种数组，数组里面存放的是地址，而数组指针就是指针

**数组指针变量是存放数组的地址，能够指向数组的指针变量（这么说有点绕不妨看一下代码）**

```c
int *p1[10]//数组指针
int(*p2)[10]//指针数组
```

### 初始化数组指针变量

想要初始化数组指针变量，得运用我们之前学的&和数组地址的相关知识

```c
int arr[10] = {0};
int (*p)[10] = &arr;
```

接下来解释一下

![](/images/c/90a75d7f136f4c239bc57bcb3cf184bd.png))
{
    for (int i = 0; i < row; i++)
    {
        for (int j = 0; j < col; j++)
        {
            printf("%d ", arr[i][j]);
        }
        printf("\n");
    }
}
int main()
{
    int arr[5][5] = { {1,2,3,4,5},{2,3,4,5,6},{3,4,5,6,7},{4,5,6,7,8},{5,6,7,8,9} };
    test(arr, 5, 5);
    return 0;
}
```

回顾二维数组，二维数组可以看作每个元素是一维数组的数组，所以二维数组的首元素就是第一行，是一个一维数组

![](/images/c/ef7159fb41c4462d9ec3f3a0938a73fe.png))[5], int row, int col)
{
    for (int i = 0; i < row; i++)
    {
        for (int j = 0; j < col; j++)
        {
            printf("%d ", *(*(p + i) + j));
        }
        printf("\n");
    }
}
int main()
{
    int arr[5][5] = { {1,2,3,4,5},{2,3,4,5,6},{3,4,5,6,7},{4,5,6,7,8},{5,6,7,8,9} };
    test(arr, 5, 5);
    return 0;
}
```

总结：二维数组传参，形参的部分可以写成数组，也可以写成指针

## 4.函数指针变量

我们根据前面的内容不难猜出函数指针变量是用来存放函数地址的，未来能够通过地址调用函数

下面我们看一下函数的地址是什么

```c
void test()
{
    printf("hello");
}
int main()
{
    printf("%p\n", test);
    printf("%p\n", &test);
    return 0;
}
```

![](/images/c/26d1fcd62643450ca76ceee2e9255db7.png)

可以看到函数有地址并且函数名就是函数的地址，用&函数名也可以获得函数的地址

有了这些知识接下来我们看一下函数指针变量究竟要怎么写（其实和数组指针非常类似）

```c
int Add(int x, int y)
{
    return x + y;
}
int main()
{
    int(*pf1)(int x, int y) = Add;//x y 也可以不用写进去 
    printf("%d\n", (*pf1)(2, 3));
    printf("%d\n", pf1(2, 3));
    return 0;
}
```

![](/images/c/308c7c351c894bc88a0ef382c3e07889.png))();

## 6.转移表

最后让我们运用以上知识实现一个简单的计算器（转移表）

首先先看不用函数指针怎么写

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
            printf("输入操作数：");
            scanf("%d %d", &x, &y);
            ret = add(x, y);
            printf("ret = %d\n", ret);
            break;
        case 2:
            printf("输入操作数：");
            scanf("%d %d", &x, &y);
            ret = sub(x, y);
            printf("ret = %d\n", ret);
            break;
        case 3:
            printf("输入操作数：");
            scanf("%d %d", &x, &y);
            ret = mul(x, y);
            printf("ret = %d\n", ret);
            break;
        case 4:
            printf("输入操作数：");
            scanf("%d %d", &x, &y);
            ret = div(x, y);
            printf("ret = %d\n", ret);
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

这样子写稍显复杂因为存在创建的四个函数有相似之处，所以接下来看用函数指针怎么实现转移表


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
int main()
{
    int x, y;//操作数
    int input = 1;//进行选择
    int ret = 0;//存放结果
    int (*p[5])(int x, int y) = { 0,add,sub,mul,div };//转移表
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
        if (input <= 4 && input >= 1)
        {
            printf("请输入操作数：");
            scanf("%d %d", &x, &y);
            ret = (*p[input])(x, y);
            printf("ret = %d\n", ret);
        }
        else if (input == 0)
        {
            printf("退出\n");
        }
        else
        {
            printf("请重新输入\n");
        }
    } while (input);
    return 0;
}
```

这样我们就用函数指针稍微简化了代码

## 7.指针加法的数学本质

在 C
语言中，内存是一维的、连续的字节数组。指针本质上就是一个存储了某个内存地址的整数。但是，指针比普通的整数多了一个极其重要的属性：**步长（Type
Size）**。

**万能推导公式**：  
对于任意指针 p，p +
1 在内存中实际跨越的字节数，严格等于 **sizeof(\*p)**。

也就是说：**指针加
1，跳过的是它所指向的那个“数据类型”在内存中占用的总字节数。**

假设我们有一段代码

```c
int a[5] = {1, 2, 3, 4, 5};
```

在内存中，这开辟了连续的 5 \* 4 = 20个字节

**为什么 a + 1 是跳过一个元素？**

- **逻辑推导**：在 C
  语言中，直接使用数组名 a，它会“退化”为指向首元素的指针。首元素是 int 类型，所以 a 的类型是 int
  \*。

- **套用公式**：\*(a) 的类型是 int。所以 a +
  1 跨越的内存大小是 sizeof(int)，也就是 4 个字节。

- **结论**：它精确地跳过了一个元素，指向了 a[1]。

**为什么 &a + 1 是跳过整个数组？**

- **逻辑推导**：&a 是对整个数组取地址。它的类型是“指向含有 5 个 int
  的数组的指针”，记作 int (\*)[5]。

- **套用公式**：\*(&a) 的类型是 int[5]（一个完整的包含 5
  个元素的数组）。所以 &a + 1 跨越的内存大小是 sizeof(int[5])，也就是5
  \* 4 = 20个字节

- **结论**：它精确地跳过了整个数组，指向了数组 a 最后一个元素之后的内存位置。
