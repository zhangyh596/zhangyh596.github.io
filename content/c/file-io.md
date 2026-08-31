---
title: "文件操作（C语言超详细讲解）"
date: "2025-12-22T17:14:57+08:00"
draft: false
categories: ["C"]
tags: ["文件操作", "fopen", "fread", "fwrite", "fclose"]
---
## 1.使用文件的原因

如果没有文件，我们写的程序的数据是存储在电脑的内存中，如果程序退出，内存回收，数据就丢失了，等再次运行程序，是看不到上次程序的数据的，如果要将数据进行持久化的保存，我们可以使用文件。

## 2.什么是文件

磁盘（硬盘）上的文件是文件。
但是在程序设计中，我们⼀般谈的文件有两种：程序文件、数据文件（从文件功能的角度来分类的）。

### （1）程序文件

程序⽂件包括源程序⽂件（后缀为.c）,⽬标⽂件（windows环境后缀为.obj）,可执⾏程序（windows
环境后缀为.exe）。

### （2）数据文件

⽂件的内容不⼀定是程序，⽽是程序运⾏时读写的数据，⽐如程序运⾏需要从中读取数据的⽂件，或者输出内容的⽂件。

### （3）文件名

一个⽂件要有⼀个唯⼀的⽂件标识，以便⽤户识别和引⽤。
⽂件名包含3部分：⽂件路径+⽂件名主⼲+⽂件后缀

例如： c:\code\test.txt 为了⽅便起⻅，⽂件标识常被称为⽂件名。

## 3.二进制文件和文本文件（数据文件）

数据在内存中以⼆进制的形式存储，如果不加转换的输出到外存的⽂件中，就是⼆进制⽂件

如果要求在外存上以ASCII码的形式存储，则需要在存储前转换。以ASCII字符的形式存储的⽂件就是⽂本⽂件（内容可以直接阅读）

字符⼀律以ASCII形式存储，数值型数据既可以⽤ASCII形式存储，也可以使⽤⼆进制形式存储。
如有整数10000，如果以ASCII码的形式输出到磁盘，则磁盘中占⽤5个字节（每个字符⼀个字节），⽽⼆进制形式输出，则在磁盘上只占4个字节。

![](/images/c/e8df7ad9d6c141fa9fd61440f85efccd.png)

## 4.文件的打开和关闭

### （1）流和标准流

我们程序的数据需要输出到各种外部设备，也需要从外部设备获取数据，不同的外部设备的输⼊输出
操作各不相同，为了⽅便程序员对各种设备进⾏⽅便的操作，我们抽象出了流的概念，我们可以把流
想象成流淌着字符的河。

⼀般情况下，我们要想向流⾥写数据，或者从流中读取数据，都是要打开流，然后操作。

C语言程序启动的时候默认打开了三个流

- stdin-标准输入流，大多数的环境中从键盘输入，scanf函数就是从标准输入流中读取数据
- stdout-标准输出流，大多数的环境中输出至显示器界面，printf函数就是将信息输出到标准输出流中
- stderr-标准错误流，大多数环境中输出到显示器界面

三个流的类型是FILE\*，通常称为文件指针

### （2）文件指针

缓冲文件系统中，关键的概念是“⽂件类型指针”，简称“⽂件指针”。

每个被使⽤的⽂件都在内存中开辟了⼀个相应的⽂件信息区，⽤来存放⽂件的相关信息（如⽂件的名字，文件状态及⽂件当前的位置等）。这些信息是保存在⼀个结构体变量中的。该结构体类型是由系统声明的，取名FILE.

```c
//VS2013编译环境提供的stdio.h头文件中有以下的文件类型声明
struct _iobuf
{
       char *_ptr;
       int   _cnt;
       char *_base;
       int   _flag;
       int   _file;
       int   _charbuf;
       int   _bufsiz;
       char *_tmpfname;
};
```


```c
typedef struct _iobuf FILE;
```


不同C语言编译器 的FILE类型包含的内容不完全相同但大同小异

每当打开⼀个⽂件的时候，系统会根据⽂件的情况⾃动创建⼀个FILE结构的变量，并填充其中的信
息，使⽤者不必关⼼细节。

⼀般都是通过⼀个FILE的指针来维护这个FILE结构的变量，这样使⽤起来更加⽅便。

```c
FILE* pf；//文件指针变量
```


定义pf是⼀个指向FILE类型数据的指针变量。可以使pf指向某个⽂件的⽂件信息区（是⼀个结构体变
量）。通过该⽂件信息区中的信息就能够访问该⽂件。也就是说，通过⽂件指针变量能够间接找到与
它关联的⽂件。

### （3）文件的打开和关闭

文件在读写之前应该先打开文件，在使用结束后应该关闭文件

fopen函数就是用来打开文件，fclose函数是用来关闭文件

![](/images/c/2f15f6e86f0c4940b41ff1e39f1eb628.png)

filename是字符串，指定要打开文件的路径名称（如“test.txt”,“D:/data.bin”）

mode表示文件的打开方式

成功返回FILE\*指针，失败返回NULL

    文件使用方式                含义                                   
                       如果指定文件不存在

- “r”（只读）    为了输入数据，打开一个已经存在的文本文件         出错
- “w”（只写）  为了输出数据，打开一个文本文件                           
  建立一个新的文件
- “a”（追加）   向文本文件末尾添加数据                                 
          建立一个新的文件
- “rb”（只读）  为了输入数据，打开一个二进制文件                       
  出错
- “wb”（只写） 为了输出数据，打开一个二进制文件                       
  建立一个新的文件
- “ab”（追加）  向一个二进制文件末尾添加数据                           
     建立一个新的文件
- “r+”（读写）   为了读和写，打开一个文本文件                           
    出错
- “w+”（读写）  为了读和写，建立一个新的文件                           
    建立一个新的文件
- “a+”（读写）  打开一个文件，在文件末尾读写                           
    建立一个新的文件
- “rb+”（读写） 为了读和写打开一个二进制文件                           
    出错
- “wb+”（读写）为了读和写，新建一个二进制文件                         
  建立一个新的文件
- “ab+”（读写）打开一个二进制文件，在文件末尾读写                 
   建立一个新的文件

![](/images/c/78d1a8cd43c147109abbdd719bbcba6d.png)

stream指向已打开文件的FILE\*指针（由fopen返回）

成功关闭返回0；失败返回EOF

**二者综合使用**

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("data.txt", "w");
    if (NULL == pf)
    {
        perror("fopen failed");//打印错误原因
    }
    //后续操作...
    fclose(pf);
    pf = NULL;//避免野指针
    return 0;
}
```


## 5.！！！文件的顺序读写！！！

函数介绍

### （1）fgetc（字符输入函数）

![](/images/c/10856350b19143829bdd0b878a26dd86.png)

- stream指向已打开文件的FILE\*指针
- 返回值：成功读取返回读取到字符的ASCII码值（int类型）；失败或读取到文件末尾返回EOF

<!-- -->

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("test.txt", "r");
    if (pf == NULL)
    {
        perror("fopen failed");
    }
    int ch;//用int类型接收而非char
    while ((ch = fgetc(pf)) != EOF)
    {
        putchar(ch);//输出读取到的字符
    }
    return 0;
}
```


### （2）fputc（字符输出函数）

![](/images/c/66cc202dd7c5413fb603a0b12d4ef3ad.png)

- character是要写入的字符（虽然是int类型，但实际只取低八位的字符ASCII码值）
- stream指向已打开文件的FILE\*指针
- 返回值：成功写入返回写入的字符ASCII码值；失败返回EOF

fputc
 会将指定字符写入文件的当前读写位置，写入后文件内部的位置指针会自动后移一个字节，指向下一个待写入的位置。

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("test.txt", "w");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    //写入字符
    fputc('h', pf);
    fputc('e', pf);
    fputc('l', pf);
    fputc('l', pf);
    fputc('o', pf);
    fclose(pf);
    pf = NULL;
    return 0;
}
```


打开test.txt会出现hello

### （3）fgets（文本行输入函数）

![](/images/c/89b49b59a053457ca6a54502e4f0df67.png)

- str指向用于存储读取内容的字符数组（缓冲区）
- n是缓冲区的最大容量（最多读取n-1个字符，末尾自动添加\0形成字符串）
- stream指向已打开文件的FILE\*指针
- 返回值：成功返回str指针；失败或读取到文件末尾返回NULL；

从文件当前位置开始读取字符，直到遇到换行符\n，文件末尾EOF，读满n-1个字符，停止读取

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("test.txt", "r");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    char buf[1024];
    while (fgets(buf, sizeof(buf), pf) != NULL)
    {
        printf("%s", buf);
    }
    fclose(pf);
    pf = NULL;
    return 0;
}
```


### （4）fputs（文本行输出函数）

![](/images/c/eb1bf08f910f4a44a254d1dc88e4f988.png)

- str是待写入的字符串指针（必须以\0结尾）
- stream指向已打开文件的FILE\*指针
- 返回值：成功返回一个非负的值；失败返回EOF

<!-- -->


```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("test.txt", "w");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    const char* str1 = "hello";
    const char* str2 = "world";
    fputs(str1, pf);
    fputs("\n", pf);
    fputs(str2, pf);
    fclose(pf);
    pf = NULL;
```


```c
    return 0;
}
```


![](/images/c/c6b077cd1ca84d91b53422c96cb000ca.png)

### （5）fscanf（格式化输入函数）

![](/images/c/563e62ebce66434e8308212b88a3fde6.png)

- stream指向已打开文件的FILE\*指针
- format是格式控制字符串，和scanf的格式符一致（%d，%s，%f）
- ...可变参数列表，传入存储读取数据的变量地址
- 返回值：成功读取的变量个数；读取到EOF或失败返回EOF

<!-- -->

```c
#include <stdio.h>
```


```c
int main()
{
    //data.txt的内容是10 3.14 hello
    FILE* pf = fopen("data.txt", "r");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    int a;
    float b;
    char c[20];
    int count = fscanf(pf, "%d %f %s", &a, &b, c);
    printf("读取到的个数：%d\n", count);
    printf("a=%d,b=%.2f,c=%s", a, b, c);
    fclose(pf);
    pf = NULL;
    return 0;
}
```


### （6）fprintf（格式化输出函数）

![](/images/c/866e31210a5043158b76c27bedddeef0.png)

- stream指向已打开文件的FILE\*指针
- format是格式控制字符串
- ...可变参数列表，传入要写入的具体数据
- 返回值：成功写入的字符总数；失败时返回负数

<!-- -->

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("data.txt", "w");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    int id = 1001;
    char name[] = "zhangsan";
    float score = 90.5;
    fprintf(pf, "%d %s %.1f", id, name, score);
    fclose(pf);
    pf = NULL;
    return 0;
}
```


### （7）fread（二进制输入）

![](/images/c/fee1f05eaa7748eebb83e473b4791032.png)

- ptr指向内存缓冲区的指针，用于存储读取到的数据
- size是单个数据块的字节大小（如sizeof（int））
- count是要读取的数据块数量
- stream指向以打开文件的FILE\*指针，需用二进制模式打开（rb）
- 返回值：成功读取的数据块数量；若返回值小于count可能是读取到文件末尾或发生错误，需用feof/ferror判断

<!-- -->

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("data.bin", "rb");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    int arr[5];
    size_t count = fread(arr, sizeof(int), 5, pf);
    printf("读取到的个数：%zd", count);
    for (int i = 0; i < count; i++)
    {
        printf("%d ", arr[i]);
    }
    fclose(pf);
    pf = NULL;
    return 0;
}
```


### （8）fwrite（二进制输出）

![](/images/c/6c59d3fc355847f0adfc436844a11f5e.png)

- ptr指向待写入数据内存缓冲区指针
- size是单个数据块的字节大小
- count是要写入的数据块数量
- stream指向已打开文件的FILE\*指针，需要二进制模式打开（wb）
- 返回值：成功写入的数据块数量；若返回值小于count，说明写入失败（没有专门失败返回值）

<!-- -->

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("data.bin", "wb");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    int arr[5] = { 1,2,3,4,5 };
    size_t count = fwrite(&arr, sizeof(int), 5, pf);
    if (count == 5)
    {
        printf("数据写入成功 \n");
    }
    fclose(pf);
    pf = NULL;
    return 0;
}
```


### （9）对比函数scanf / fscanf / sscanf

![](/images/c/9040f9c2e8c944408954e4f7ede8904a.png)

scanf（数据源：标准输入（键盘）），从控制台读取数据

![](/images/c/98c2b4cc42464f5e9db1dd571a907c5d.png)

fscanf（数据源：文件（FILE指针）），从文件读取数据

![](/images/c/aacbef54c5fd45d9b48357c88638dbb5.png)

sscanf（数据源：字符串（char数组）），从内存中的字符串提取数据

- s待解析的源字符串
- format是格式控制符
- ...是接收数据的变量地址

三者格式控制符规则完全一致，都返回成功读取的变量个数，下面介绍sscanf的使用

```c
#include <stdio.h>
```


```c
int main()
{
    char s[] = "zhangsan 20 90.5";
    char name[20];
    int age;
    float score;
    int ret = sscanf(s, "%s %d %f", name, &age, &score);
    printf("成功读取变量个数：%d\n", ret);
    printf("name=%s age=%d score=%.1f", name, age, score);
    return 0;
}
```


### （10）对比函数printf / fprintf / sprintf

![](/images/c/539307079ede4a088a0c5959c330d2dd.png)

printf（输出目标：标准输出（屏幕/控制台）），向控制台打印数据

![](/images/c/92f6c607d5054acc884de0f3c97d9559.png)

fprintf（输出目标：文件（FILE指针）），向文件中写入数据

![](/images/c/fb8bc4cb914f44fc87638799ee4318af.png)

sprintf（输出目标：字符串（char数组）），将格式化数据写入内存字符串

- str指目标字符串，用于 存储格式化后的字符串，需要提前分配足够内存
- format是格式控制符
- ...是可变参数，即需要格式化的若干数据

三者返回值成功时返回写入的字符总数，失败时返回负数，下面介绍sprintf的使用


```c
#include <stdio.h>
```


```c
int main()
{
    char buf[100];
    int id = 1001;
    float price = 29.9;
    char name[] = "book";
    int len = sprintf(buf, "商品：%s,编号：%d,价格：%.1f", name, id, price);
    printf("成功写入字符数%d\n", len);
    printf("拼接结果：%s", buf);
    return 0;
}
```


## 6.文件的随机读写

### （1）fseek

![](/images/c/d36d5c7f579b45a3bd9149d0ebbb7cc6.png)

- fseek可以用于移动文件流的文件指针到指定位置，为后续的读写操作确定起始点
- stream指向已打开文件的文件指针
- offset是偏移量，单位是字节，正（向后移动）负（向前移动）也可为0
- origin偏移的参考起点有三个宏定义                                     
                                                       
   SEEK_SET：参考文件开头（文件起始位置）                               
                                           
   SEEK_CUR：参考当前文件指针的位置                                     
                                               
   SEEK_END：参考文件末尾位置
- 返回值：成功返回0，失败返回非0值

<!-- -->


```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("test.txt", "r");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    fseek(pf, 5, SEEK_SET);
    int ch = fgetc(pf);
    if (ch == EOF)
    {
        printf("读取失败或已到文件末尾\n");
    }
    else
    {
        printf("读取的字符：%c\n", ch);
    }
    fclose(pf);
    pf = NULL;
    return 0;
}
```


### （2）ftell

![](/images/c/0e14ca8d03dc44458e5909ea33ef725f.png)

- ftell用于获取当前文件指针相对于文件开头的偏移字节数
- stream指向已打开文件的文件指针
- 返回值：成功返回当前文件指针的便宜字节数；失败返回-1L，并设置errno标识错误原因

<!-- -->

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("test.txt", "rb");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    fseek(pf, 0, SEEK_END);
    long size = ftell(pf);
    printf("文件大小：%ld字节\n", size);
    fclose(pf);
    pf = NULL;
    return 0;
}
```


### （3）rewind

![](/images/c/a10489a682be41af871b2e01023e1ef0.png)

- rewind用于将文件指针重置到文件的起始位置，同时清除文件流的错误标志和EOF标志
- stream指向已打开的文件指针
- 无返回值

rewind（pf）等价于fseek（pf，0L，SEEK_SET），但fseek会返回int类型的状态值，可以判断定位是否成功，rewind没有返回值就无法判断

```c
#include <stdio.h>
```



```c
int main()
{
    FILE* pf = fopen("test.txt", "r");
    if (pf == NULL)
    {
        printf("fopen failed");
    }
    char buf[100];
    fgets(buf, sizeof(buf), pf);
    printf("第一次读取%s\n", buf);
```


```c
    rewind(pf);
    fgets(buf, sizeof(buf), pf);
    printf("第二次读取%s", buf);
    fclose(pf);
    pf = NULL;
    return 0;
}
```


## 7.文件读取结束的判断

### （1）feof

![](/images/c/b5e000ed963b4e83a54ca46daf0ea809.png)

- feof是用于检测文件指针是否到达文件末尾（EOF）
- stream指向已打开文件的文件指针
- 返回值：若文件指针到达末尾返回非0值；否则返回0

⽂本⽂件读取是否结束，判断返回值是否为 EOF （ fgetc ），或者 NULL （
fgets ）

例如：fgetc 判断是否为 EOF .

           fgets 判断返回值是否为 NULL .

          ⼆进制⽂件的读取结束判断，判断返回值是否⼩于实际要读的个数。

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("test.txt", "r");
    if (pf == NULL)
    {
        perror("fopen failed");
    }
    char buf[100];
    while (fgets(buf, sizeof(buf), pf) != NULL)
    {
        printf("%s", buf);
    }
    if (feof(pf))
    {
        printf("\n已读取到文件末尾\n");
    }
    else
    {
        printf("\n读取过程中发生错误\n");
    }
    fclose(pf);
    pf = NULL;
    return 0;
}
```


### （2）ferror

![](/images/c/e1d75c67ef4a4b8b8beea59566133376.png)

- ferror用于检测文件流是否发生读写错误（可以与feof配合使用）
- stream指向已打开文件的文件指针
- 返回值：若文件流发生读写错误，返回非0值，否则返回0

<!-- -->

```c
#include <stdio.h>
```


```c
int main()
{
    FILE* pf = fopen("test.txt", "r");
    if (NULL == pf)
    {
        perror("fopen failed");
    }
    char buf[100];
    while (fgets(buf, sizeof(buf), pf) != NULL)
    {
        printf("%s", buf);
    }
    if (feof(pf))
    {
        printf("\n已到达文件末尾\n");
    }
    else if (ferror(pf))
    {
        printf("\0发生文件读写错误\n");
    }
    fclose(pf);
    pf = NULL;
    return 0;
}
```


## 8.文件缓冲区

ANSIC标准采⽤“缓冲⽂件系统”处理的数据⽂件的，所谓缓冲⽂件系统是指系统⾃动地在内存中为
程序中每⼀个正在使⽤的⽂件开辟⼀块“⽂件缓冲区”。从内存向磁盘输出数据会先送到内存中的缓
冲区，装满缓冲区后才⼀起送到磁盘上。如果从磁盘向计算机读⼊数据，则从磁盘⽂件中读取数据输
⼊到内存缓冲区（充满缓冲区），然后再从缓冲区逐个地将数据送到程序数据区（程序变量等）。缓
冲区的⼤⼩根据C编译系统决定的

![](/images/c/dd9c9a5ed42748a484eb6bfb542d2b1b.png)

这⾥可以得出⼀个结论：
因为有缓冲区的存在，C语⾔在操作⽂件的时候，需要做刷新缓冲区或者在⽂件操作结束的时候关闭⽂
件。

如果不做，可能导致读写⽂件的问题。
