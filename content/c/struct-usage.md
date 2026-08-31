---
title: "结构体的使用（C语言超详细）"
date: "2025-12-14T11:20:09+08:00"
draft: false
categories: ["C"]
tags: ["结构体", "struct", "结构体指针"]
---
## 1.结构体类型的声明

结构体是什么呢？结构体其实就是把一些值存入一个整体中，并且这些值（成员变量）可以是不同类型

### 结构体的声明

声明如下

```c
struct tag
{
     member-list;
}variable-list;
```


例如描述一个学生

```c
struct student
{
    char name[20];//名字
    int age;//年龄
    char sex;//性别
    char id[20];//学号
};
```


### 结构体的访问

为了后续使用，我们还需要知道结构体怎么进行访问，这其实有两种方式

#### （1）直接访问：.操作符

语法格式是结构体变量名.成员名

```c
#include <stdio.h>
```


```c
struct student
{
    char name[20];//名字
    int age;//年龄
    char sex[5];//性别
    char id[20];//学号
};
```


```c
int main()
{
    struct student s = { "zhangsan",20,"男","2025" };
    printf("name:%s\n", s.name);
    printf("age:%d\n", s.age);
    printf("sex:%s\n", s.sex);
    printf("id:%s\n", s.id);
    return 0;
}
```


当然如果不想按照结构体成员的顺序初始化我们也可以用.来按照指定顺序初始化

```c
#include <stdio.h>
```


```c
struct student
{
    char name[20];//名字
    int age;//年龄
    char sex[5];//性别
    char id[20];//学号
};
```


```c
int main()
{
    struct student s = { .age = 20,.name = "zhangsan",.id = "2025",.sex = "男" };
    printf("name:%s\n", s.name);
    printf("age:%d\n", s.age);
    printf("sex:%s\n", s.sex);
    printf("id:%s\n", s.id);
    return 0;
}
```


#### （2）间接访问：->操作符

语法格式：结构体指针名->成员名

其实等价于（\*结构体指针名）.成员名   由于.优先级高于\*故括号不能去除

```c
#include <stdio.h>
```


```c
struct student
{
    char name[20];//名字
    int age;//年龄
    char sex[5];//性别
    char id[20];//学号
};
```


```c
int main()
{
    struct student s = { .age = 20,.name = "zhangsan",.id = "2025",.sex = "男" };
    struct student* ps = &s;
    printf("name:%s\n", ps->name);
    printf("age:%d\n", ps->age);
    printf("sex:%s\n", ps->sex);
    printf("id:%s\n", ps->id);
    return 0;
}
```


## 2.结构体的特殊声明

在声明结构体时可以不完全声明（匿名结构体）

```c
struct 
{
    int a;
    char b;
}x;
struct
{
    int a;
    char b;
}*p;
```


这样的声明只能使用一次，所以如果写p =
&x这是非法的，因为编译器会把上面的两个声明当成不同的类型

匿名的结构体类型如果没有对结构体类型重命名的话，基本只能用一次

## 3.结构体的自引用

比如结构体中包含一个类型为该结构体本身的成员

```c
//错误写法
struct Node
{
    int data;
    struct Node next
};
//正确写法
struct Node
{
    int data;
    struct Node* next;
};
```


第一个错误的原因就是如果计算sizeof（struct
Node），那么将会无穷大，因为会一直访问下去，如果采用指针就可以有效解决这一问题，因为指针的大小是固定的4/8位字节

在结构体自引用的过程中夹杂typedef对匿名结构体类型重命名也容易引入问题

```c
//错误写法
typedef struct
{
      int data;
      Node* next;
}Node;
//正确写法
typedef struct Node
{
      int data;
      struct Node* next;
}Node;
```


虽然Node是typedef对匿名结构体类型重命名产生的，但是在匿名结构体中是不能提前使用Node类型来创建成员变量，解决方法也很简单，就是定义结构体不要使用匿名结构体了

## 4.结构体内存的对齐

掌握了结构体基本使用就可以来计算一下结构体的大小了，结构体内存对齐无疑是一个重要的点

### （1）对齐规则

- 结构体的第一个成员对齐到和结构体变量起始位置偏移量为0的地址
- 其他成员变量要对齐到对齐数的整数倍

对齐数是编译器默认的一个对齐数与该成员变量大小的较小值

VS2022中默认的一个对齐数是8

Linus中gcc没有默认对齐数，对齐数就是成员自身大小

- 结构体总大小为最大对齐数（结构体中每个成员变量都有一个对齐数，所有对齐数中最大的）的整数倍
- 如果嵌套了结构体的情况，嵌套的结构体成员对齐到自己的成员中最大对齐数的整数倍处，结构体的整体大小就是所有最大对齐数（含嵌套结构体中成员的对齐数）的整数倍

例如

```c
#include <stdio.h>
```


```c
struct s1
{
    char c1;//偏移0，占1字节
    int i;//偏移到4，（中间浪费了三个字节），占4个字节
    char c2;//偏移8，占1字节
};//最终需是最大对齐数的整数倍，故结构体大小是12（再次浪费了三个字节）
struct s2
{
    char c1;//偏移0，占1字节
    struct s1 s;//偏移到4，（中间浪费了三个字节），占12个字节
    double d;//偏移16，占8字节
};//24是最大对齐数的整数倍，故结构体大小是24
int main()
{
    printf("%d\n", sizeof(struct s1));
    printf("%d\n", sizeof(struct s2));
    return 0;
}
```


### （2）为什么存在内存对齐

1\. 平台原因 (移植原因)：  
不是所有的硬件平台都能访问任意地址上的任意数据的；某些硬件平台只能在某些地址处取某些特定类型的数据，否则抛出硬件异常。  
2. 性能原因：  
数据结构(尤其是栈)应该尽可能地在⾃然边界上对⻬。原因在于，为了访问未对⻬的内存，处理器需要作两次内存访问；⽽对⻬的内存访问仅需要⼀次访问。假设⼀个处理器总是从内存中取8个字节，则地址必须是8的倍数。如果我们能保证将所有的double类型的数据的地址都对⻬成8的倍数，那么就可以⽤⼀个内存操作来读或者写值了。否则，我们可能需要执⾏两次内存访问，因为对象可能被分放在两个8字节内存块中。  
总体来说：结构体的内存对齐是拿空间来换取时间的说法

所以在设计结构体时要尽量让占空间小的成员集中在一起

### （3）修改默认对齐数

\#pragma这个预处理指令可以改变编译器的默认对齐数

```c
#include <stdio.h>
#pragma pack(1)//设置默认对齐数为1
struct s
{
    char c1;//偏移0，占1字节
    int i;//偏移1，占4字节
    char c2;//偏移5，占1字节
};//结构体总大小6是1的整数倍
int main()
{
    printf("%d", sizeof(struct s));
    return 0;
}
```


如果想取消设置的对齐数，还原成默认，只需要再加上#pragma pack()即可

## 5.结构体传参

联系前面所学的传值调用和传址调用

```c
#include <stdio.h>
```


```c
struct s
{
    int arr[1000];
    int num;
};
struct s s1 = { {1,2,3,4},1000 };//给结构体初始化
//结构体传参
void print1(struct s s1)
{
    printf("%d\n", s1.num);
}
//结构体传址
void print2(struct s* ps)
{
    printf("%d\n", ps->num);
}
int main()
{
    print1(s1);
    print2(&s1);
    return 0;
}
```


两种方法虽然都可以打印出num的值，但是第一张方法却无法更改num的值，第二种方法可以

而且函数传参的时候，参数是需要压栈的，会有时间和空间上的系统开销

如果传递一个结构体对象的时候，结构体过大，参数压栈的系统开销比较大，所以会导致性能的下降

综上，结构体传参的时候要传递结构体的地址

## 6.结构体实现位段

### （1）什么是位段呢

位段的成员通常是int ，unsigned int ，signed
int，C99中位段成员的类型也可以选择其他类型

位段的成员名后边有一个冒号和一个数字

例如

```c
struct A
{
    int a : 2;
    int b : 5;
    int c : 10;
    int d : 30;
};
```


接下来讲位段的大小如何计算（当然不同环境下可能不同，以VS2022为例）

- 确定基础存储单位，是char的话存储单元是1字节（8比特），int的话存储单元是4字节（32比特）
- 逐成员分配比特位，int a：2占用第一个int单元的0-1位，int
  b：5占用第一个int单元的2-6位，int c：10占用第一个int单元的7-16，int
  d：30占用第二个int单元的0-29位（因为第一个int单元剩下的位不足30）
- 统计总存储单元，可以知道结构体共占了2个int单元即8个字节

### （2）位段的内存分配

- 位段的成员可以是int，unsigned int，signed int，char等类型
- 位段的空间上是按照需要以4个字节（int）或1个字节（char）的方式来开辟的
- 位段涉及很多不确定因素，位段是不跨平台的，注重可移植的程序应该避免使用位段

![](/images/c/f2afbe7ec69743c9b16bb6960b4af0c6.png)

![](/images/c/265558d9af1544dfa0b0e32ca77c0ef8.png)

可以看到s的地址最终是62 03 04 01

分析过程

![](/images/c/53d40539d4344b7fba81bdc81f18c0b4.png)

### （3）位段跨平台问题

1.int 位段被当成有符号数还是⽆符号数是不确定的。  
2.
位段中最⼤位的数⽬不能确定。（16位机器最⼤16，32位机器最⼤32，写成27，在16位机器会  
出问题。  
3. 位段中的成员在内存中从左向右分配，还是从右向左分配，标准尚未定义。  
4.
当⼀个结构包含两个位段，第⼆个位段成员⽐较⼤，⽆法容纳于第⼀个位段剩余的位时，是舍弃  
剩余的位还是利⽤，这是不确定的。  
总结：  
跟结构相⽐，位段可以达到同样的效果，并且可以很好的节省空间，但是有跨平台的问题存在。

### （4）位段使用的注意事项

位段的⼏个成员共有同⼀个字节，这样有些成员的起始位置并不是某个字节的起始位置，那么这些位置处是没有地址的。内存中每个字节分配⼀个地址，⼀个字节内部的bit位是没有地址的。  
所以不能对位段的成员使⽤&操作符，这样就不能使⽤scanf直接给位段的成员输⼊值，只能是先输⼊放在⼀个变量中，然后赋值给位段的成员。

```c
#include <stdio.h>
```


```c
struct A
{
    int a : 2;
    int b : 5;
    int c : 10;
    int d : 30;
};
int main()
{
    struct A s = { 0 };
    //错误示范
    //scanf("%d", &s.a);
    //正确写法
    int a = 0;
    scanf("%d", &a);
    s.a = a;
    return 0;
}
```

