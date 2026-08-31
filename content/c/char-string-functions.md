---
title: "字符函数和字符串函数（C语言详细）"
date: "2025-11-26T20:30:39+08:00"
draft: false
categories: ["C"]
tags: ["字符分类函数", "字符串函数", "ctype.h"]
---
## 1.字符分类函数

接下来所说函数在头文件\<ctype.h\>里面

- iscntrl 任何控制字符
- isspace 空白字符：空格'
  '，换页'\f'，换行'\n'，回车'\r'，制表符'\t'，垂直制表符'\v'
- isdigit 十进制数字'0'~'9'字符
- isxdigit
  十六进制数字字符，包括所有十进制数字字符，小写字母a~f，大写字母A~F
- islower 小写字母a~z
- isupper 大写字母A~Z
- isalpha 小写字母+大写字母
- isalnum 字母或者数字a~z，A~Z，0~9
- ispunct 标点符号，任何不属于数字或者字母的图形字符（可打印）
- isgraph 任何图形字符
- isprint 任何可打印字符，包括图形字符和空白字符

如果函数里面的参数符合上述条件就返回真

例如islower的用法

    #include <stdio.h>
    #include <ctype.h>

    int main()
    {
        int i = 0;
        char str[] = "hello,world";
        while (str[i])
        {
            if (islower(str[i]))
                str[i] -= 32;
            putchar(str[i]);
            i++;
        }

        return 0;
    }

![](/images/c/f747ba7a8d5d44d1b7417046f2129365.png)

## 2.字符转换函数

- tolower 将参数传进去的大写字母转小写
- toupper 将参数传进去的小写字母转大写

这样我们就可以更改上述代码

    #include <stdio.h>
    #include <ctype.h>

    int main()
    {
        int i = 0;
        char str[] = "hello,world";
        while (str[i])
        {
            str[i] = toupper(str[i]);//使用转换函数
            putchar(str[i]);
            i++;
        }

        return 0;
    }

![](/images/c/7437c008ea1849c5b98db31305efad64.png)

## 3.字符串函数（包含头文件\<string.h\>）

### strlen的使用

###   ![](/images/c/2ec6166c277e47b2b5ffd3d33b0d9f9e.png)

- strlen函数会返回在字符串中‘\0’之前出现的字符个数（不包含‘\0’）
- 参数指向的字符串必须以‘\0’结束
- 返回值是size_t，无符号整数

<!-- -->

    #include <stdio.h>
    #include <string.h>

    int main()
    {
        char str[] = "hello,world";
        printf("%zu", strlen(str));
        return 0;
    }

![](/images/c/20e68d5e4bc94816bd59bf21193dd008.png)

#### 模拟实现strlen

方式一（常规）

    #include <stdio.h>
    #include <string.h>

    size_t my_strlen(const char* str)
    {
        int count = 0;
        while (*str)
        {
            count++;
            str++;
        }
        return count;
    }
    int main()
    {
        char str[] = "hello,world";
        printf("%zu", my_strlen(str));
        return 0;
    }

方式二（递归）

    #include <stdio.h>
    #include <string.h>

    size_t my_strlen(const char* str)
    {
        if (*str == '\0')
            return 0;
        else
            return 1 + my_strlen(str + 1);
    }
    int main()
    {
        char str[] = "hello,world";
        printf("%zu", my_strlen(str));
        return 0;
    }

方式三（指针）

    #include <stdio.h>
    #include <string.h>

    size_t my_strlen(const char* str)
    {
        char* p = str;
        while (*p != '\0')
            p++;
        return p - str;
    }
    int main()
    {
        char str[] = "hello,world";
        printf("%zu", my_strlen(str));
        return 0;
    }

### strcpy的使用

![](/images/c/9839df85e1654bf491c041d03557caac.png)

- strcpy函数会将源字符串复制到目标字符串
- 源字符串必须以‘\0’结束
- 会将源字符串的‘\0’拷贝到目标字符串
- 目标字符串必须足够大
- 目标字符串可以修改
- 返回值是目标字符串地址

<!-- -->

    #include <stdio.h>
    #include <string.h>

    int main()
    {
        char dest[] = "hello";
        char src[] = "world";
        printf("%s", strcpy(dest, src));

        return 0;
    }

#### 模拟实现strcpy

    #include <stdio.h>
    #include <string.h>

    char* my_strcpy(char* dest, const char* src)
    {
        char* ret = dest;
        int i = 0;
        while (src[i] != '\0')
        {
            dest[i] = src[i];
            i++;
        }
        dest[i] = '\0';//最后手动给destkaobei'\0'
        return ret;
    }
    int main()
    {
        char dest[] = "hello";
        char src[] = "world";
        printf("%s", my_strcpy(dest, src));

        return 0;
    }

优化（无需最后手动给dest拷贝‘\0’）

    #include <stdio.h>
    #include <string.h>

    char* my_strcpy(char* dest, const char* src)
    {
        char* ret = dest;
        while (*dest++ = *src++)
        {

        }
        return ret;
    }
    int main()
    {
        char dest[] = "hello";
        char src[] = "world";
        printf("%s", my_strcpy(dest, src));

        return 0;
    }

### strcat的使用

![](/images/c/196bf0dc817244838e42440629c98ab0.png)

- strcat函数会将源字符串拼接到目标字符串末尾，自动覆盖目标字符串的‘\0’，最终在拼接结果加‘\0’

- 目标字符串必须足够大

- 源字符串和目标字符串均以‘\0’结束

- 目标字符串可以修改

- 两个字符串不能重叠

- 返回值是目标字符串的首地址

      #include <stdio.h>
      #include <string.h>

      int main()
      {
          char dest[20] = "hello,";
          char src[] = "world";
          printf("%s", strcat(dest, src));
          return 0;
      }

  #### 模拟实现strcat

<!-- -->

    #include <stdio.h>
    #include <string.h>

    char* my_strcat(char* dest, const char* src)
    {
        char* ret = dest;
        while (*dest != '\0')//找到dest为\0的位置
        {
            dest++;
        }
        while (*src != '\0')//给dest追加src
        {
            *dest = *src;
            dest++;
            src++;
        }
        *dest = '\0';
        return ret;
    }
    int main()
    {
        char dest[20] = "hello,";
        char src[] = "world";
        printf("%s", my_strcat(dest, src));
        return 0;
    }

### strcmp的使用

![](/images/c/1520160cdd0e49a298cc04836832aecd.png)

- strcmp可以比较两个字符串（基于ASCII码）
- 从两个字符串的第一个字符开始逐个比较，直到遇到不同的字符或\0
- 字符串必须以‘\0’结束
- 返回值 \>0(str1\>str2);=0(str1=str2);\<0(str1\<str2)

<!-- -->

    #include <stdio.h>
    #include <string.h>

    int main()
    {
        char str1[] = "apple";
        char str2[] = "banana";
        printf("%d", strcmp(str1, str2));
        return 0;
    }

#### 模拟实现strcmp

    #include <stdio.h>
    #include <string.h>

    int my_strcmp(const char* str1, const char* str2)
    {
        while (str1 != '\0'&&str2 != '\0'&&(*str1 == *str2))
        {
            str1++;
            str2++;
        }
        return *str1 - *str2;
    }
    int main()
    {
        char str1[] = "apple";
        char str2[] = "banana";
        printf("%d", my_strcmp(str1, str2));
        return 0;
    }

### strncpy的使用

![](/images/c/fd022968cea646bf8c7205cee2c89554.png)

- strncpy可以将源字符串的最多前n个字符复制到目标字符串中（当拷贝了n个字符或者遇到了源字符串的‘\0’停止）
- 目标空间足够大
- src长度大于等于n，今复制前n个字符，不会在dest末尾补‘\0’
- src长度小于n，复制完src后剩位置补‘\0’，直到写入n个字符
- 返回值是目标字符串的其实地址

<!-- -->

    #include <stdio.h>
    #include <string.h>

    int main()
    {
        char dest[10] = "hello";
        char src[] = "world";
        printf("%s", strncpy(dest, src, 5));
        return 0;
    }

#### 模拟实现strncpy

    #include <stdio.h>
    #include <string.h>
    #include <assert.h>

    char* my_strncpy(char* dest, const char* src,size_t n)
    {
        char* ret = dest;
        assert(dest && src);
        while (n > 0 && *src != '\0')
        {
            *dest++ = *src++;
            n--;
        }
        while (n > 0)
        {
            *dest++ = '\0';
            n--;
        }
        return ret;


    }
    int main()
    {
        char dest[20] = "hello";
        char src[] = "world";
        printf("%s", my_strncpy(dest, src, 5));
        return 0;
    }

### strncat的使用

![](/images/c/f0d3cbd9be6d4541bc195df92ad3cdf1.png)

- strncat可以将源字符串的前n个字符追加到目标字符串的末尾
- 目标字符串足够大
- 会在目标字符串最后添加‘\0’
- 返回值是目标字符串的地址

<!-- -->

    #include <stdio.h>
    #include <string.h>

    int main()
    {
        char dest[20] = "hello,";
        char src[] = "world";
        printf("%s", strncat(dest, src, 5));
        return 0;
    }

#### 模拟实现strncat

（和strcat区别不大）

    #include <stdio.h>
    #include <string.h>
    char* my_strncat(char* dest, const char* src, size_t n)
    {
        char* ret = dest;
        while (*dest != '\0')
        {
            dest++;
        }
        while (n > 0 && *src != '\0')
        {
            *dest = *src;
            dest++;
            src++;
            n--;
        }
        *dest = '\0';
        return ret;
    }
    int main()
    {
        char dest[20] = "hello,";
        char src[] = "world";
        printf("%s", my_strncat(dest, src, 5));
        return 0;
    }

### strncmp的使用

![](/images/c/c1ece720f94d4afa9b79807cc07e9cef.png)

- strncmp就是在strcmp的基础上只比较两个字符串的前n个字符
- 遇到‘\0’会停止
- 返回值和strcmp一样

<!-- -->

    #include <stdio.h>
    #include <string.h>

    int main()
    {
        char str1[] = "apple";
        char str2[] = "banana";
        printf("%d", strncmp(str1, str2, 5));
        return 0;
    }

#### 模拟实现strncmp

    #include <stdio.h>
    #include <string.h>

    int my_strncmp(const char* str1, const char* str2, size_t n)
    {
        if (n == 0)
            return 0;
        while (n > 0&&str1 != '\0' && str2 != '\0' && (*str1 == *str2))
        {
            str1++;
            str2++;
            n--;
        }
        return *str1 - *str2;
    }
    int main()
    {
        char str1[] = "apple";
        char str2[] = "banana";
        printf("%d", my_strncmp(str1, str2, 5));
        return 0;
    }

### strstr的使用

![](/images/c/9d14e15985c84c4e9e15edc005bda2f6.png)

- strstr函数可以查找str2在str1第一次出现的位置
- 找到子字符串返回指向第一次出现位置的指针
- 如果没找到，返回NULL
- 如果子字符串是空字符串，返回str1的地址

<!-- -->

    #include <stdio.h>
    #include <string.h>

    int main()
    {
        char str1[] = "abcdeabcd";
        char str2[] = "bcd";
        printf("%s", strstr(str1, str2));
        return 0;
    }

#### 模拟实现strstr

    #include <stdio.h>
    #include <string.h>

    char* my_strstr(const char* str1, const char* str2)
    {
        char* cp = (char*)str1;//需要强制类型转换，遍历str1的字符
        char* s1;//当前str1的匹配指针
        char* s2;//当前str2的匹配指针
        if (*str2 == '\0')
            return ((char*)str1);
        while (*cp != '\0')//*cp为真表示str1还未走到末尾
        {
            s1 = cp;//不匹配s1返回到cp的位置
            s2 = (char*)str2;
            while (*s1 && *s2 && (*s1 == *s2))//对s1和s2匹配
            {
                s1++;
                s2++;
            }
            if (*s2 = '\0')
                return cp;
            cp++;
        }
        return (NULL);
    }
    int main()
    {
        char str1[] = "abcdeabcd";
        char str2[] = "bcd";
        printf("%s", my_strstr(str1, str2));
        return 0;
    }

### strtok的使用

![](/images/c/75475701134f432cac1aeb279d685274.png)

- strtok可以用于分割字符串
- delimiters是分隔符字符串，包含所有可能的分隔字符串
- 返回值 第一次调用：返回第一个标记的指针 后续调用：返回下一个标记的指针
- 如果没有更多标记返回
- strtok会修改原字符串，将分隔符替换成‘\0’
- 会跳过连续的分隔符
- 后续调用传NULL继续分割同一个字符串

<!-- -->

    #include <stdio.h>
    #include <string.h>

    int main()
    {
        char arr[] = "111.222.333.444";
        char* sep = '.';
        char* str = NULL;
        for (str = strtok(arr, sep); str != NULL; str = strtok(NULL, sep))
        {
            printf("%s\n", str);
        }
        return 0;
    }

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 
