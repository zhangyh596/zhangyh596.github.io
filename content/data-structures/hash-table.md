---
title: "哈希表（C语言底层实现）"
date: "2026-03-11T23:51:26+08:00"
draft: false
categories: ["数据结构"]
tags: ["哈希表", "哈希函数", "链地址法"]
---
众所周知，哈希表是数据结构经常用到的，但是哈希表是如何实现的呢

哈希表的核心思想是通过一个**哈希函数（Hash
Function）将键（Key）映射到数组的索引上。但因为不同的键可能会计算出相同的索引（这就是哈希冲突**），我们需要一种方法来解决它。最常用且最容易实现的方法是**链地址法（Separate
Chaining）**，即在每个数组位置上挂一个链表。

下面我将带你一步一步用 C
语言实现一个键为字符串（String）、值为整数（Int）的哈希表。

## 1.定义数据结构

你可以把哈希表想象成一个“带挂钩的储物柜”。

- **键（Key）**：比如你的名字 "apple"。
- **值（Value）**：比如你的存款 100 块钱。
- **哈希函数（Hash Function）**：柜台的魔法前台。你告诉她你的名字
  "apple"，她算了一下，告诉你：“去 3 号柜子”。
- **冲突（Collision）**：有时候，前台算出来 "banana" 也应该放在 3
  号柜子。但是 3 号柜子已经有 "apple" 了怎么办？我们就在 3
  号柜子下面打个孔，**挂一条链子**，把 "banana" 挂在 "apple" 下面。

这就需要我们造两个部件袋子（Node）和储物柜本体（HashTable）

### 先造袋子：struct Node

```c
typedef struct Node
{
    char* key;//你的名字（比如“apple”）
    int value;//你的存款（比如100）
    struct Node* next;//重点！是一个铁钩子，用来挂下一个袋子
}Node;
```


这是一个包裹。里面装了名字、钱，还有一个 `next` 指针。`next`
的作用就是，如果有人跟你分到了同一个柜子，就把他的袋子挂在你的 `next`
钩子上。如果下面没袋子了，这个钩子就是空的（在 C 语言里叫 `NULL`）。

### 再造储物柜本体：struct HashTable

```c
typedef struct HashTable
{
    int size;//记录柜子一共有多少个抽屉
    Node** enties;//重点！这是抽屉的集合
}HashTable;
```


有同学可能会想问为什么要写成二级指针

如果我们要存一个普通的数字，我们用 `int`。如果是一排数字（数组），我们用
`int *`。但现在，我们每个抽屉里放的是什么？放的是**袋子的钩子（指针）**，也就是
`Node *`。所以，我们需要“一排钩子”（一个存放指针的数组）。在 C
语言里，指向“指针数组”的指针，就要加两个星号，写成 `Node **`。

`entries`
就是这一排抽屉。每个抽屉里不直接放东西，而是伸出一个钩子（`Node *`），准备挂我们刚才造好的“袋子”。

## 2.哈希函数

这个函数是把我们的名字（“apple”）变成一个具体的柜子编号。这里使用到了djb2算法这个经典配方

```c
unsigned int hash(const char* key, int table_size)
{
    unsigned long int value = 5381;//魔法初始数字
    unsigned int i = 0;
    unsigned int key_len = strlen(key);
```


```c
    //核心
    for (int i = 0; i < key_len; i++)
    {
        value = value * 33 + key[i];
    }
```


```c
    //挤进柜子里面
    return value % table_size;
}
```


- **`unsigned long int value = 5381：`**

**`相当于`**前台拿出了计算器，先在上面敲了一个神奇的初始数字
`5381`。为什么是 5381？别深究，这是一个叫 Dan Bernstein
的大神经过大量数学测试后发现的“黄金数字”，用它算出来的结果最不容易让大家撞在同一个柜子里（减少冲突）。

- **`unsigned int key_len = strlen(key)：`**

前台数了数你的名字有几个字母。"apple" 是 5 个字母。

- **`for (i = 0; i < key_len; ++i) { value = value * 33 + key[i]; }`**

**这里是核心搅拌机**！电脑其实不认识英文字母，在它眼里，字母都是数字（比如
'a' 是 97，'p' 是 112）。前台开始循环操作：拿起计算器上的数字，乘以
`33`，然后加上名字里第一个字母代表的数字。接着再乘
33，加上第二个字母代表的数字……就这样把 "apple"
里的每个字母都“搅拌”进去。经过这一顿猛如虎的操作，原本的 "apple"
变成了一个巨大无比、看起来毫无规律的数字，比如 `123456789`。

- **`return value % table_size;`**

现在遇到了一个现实问题，计算器上算出了 `123456789`，但我们的柜子一共只有
`10` 个（`table_size` 为 10），去哪找第 1 亿号柜子呢？这时候就需要用到
`%`（取模运算，也就是求除法的**余数**）。`123456789 除以 10`，余数是
`9`。前台大喊一声：“你去 **9 号柜子**！”

这就完成了一次神奇的转换！下次你再拿 "apple"
来，经过一模一样的乘法和加法，算出来的结果肯定还是 9
号柜子，绝对不会找错地方。

**最后呢有人就会想为什么要设计这么复杂的一个函数，直接把所有字母代表的数字相加不就可以了吗，然而我要告诉你这是万万不行的**

比如：'a' 是 97，'b' 是 98，'c' 是
99，算一下 "abc"：`97 + 98 + 99 = 294`。去 294
号柜子。听起来很完美？但致命漏洞来了！如果有人叫 "cba" 呢？算出来也是
`99 + 98 + 97 = 294`。如果有人叫 "bac" 呢？也是
294！如果仅仅是相加，所有**字母相同、顺序不同**的词，全都会挤进同一个柜子！哈希表瞬间就变成了“垃圾堆”，前台找东西依然要翻半天。

for循环里面先让value\*33，然后再加上字母的数字，这样每一次加入新字母前，**前面所有的累积值都会被放大
33 倍**。这就导致排在前面的字母权重非常高！这样一来，"ab" 和 "ba"
算出来的结果就天差地别了，完美解决了刚才“位置不同但结果相同”的漏洞。

**为什么非要是 `33`，为什么初始值非要是 `5381`？**这两个数字，是由一位叫
Dan Bernstein
的计算机大牛，用极其无聊的穷举法，在电脑上试了成千上万个数字后找出来的“黄金搭档”。如果乘数选
32（偶数），在二进制计算里很容易造成数据丢失。如果选
31，虽然也是奇数，但在英文单词的测试里，撞柜子的概率就是比 33 高。`5381`
也是他测试出来的，作为起点，能让最终结果分布得最均匀。所以，面对这段代码，我们不需要去推导它的数学公式，你只需要把它当成**计算机界公认的祖传秘方**。这个秘方（djb2
算法）只有几行代码，跑得飞快，而且能把英文单词极度均匀地分散到各个柜子里。

## 3.初始化哈希表

C语言里面，我们想要一个柜子，就得向“仓库管理员”（操作系统）打申请，要一块指定大小的木头（内存），这个申请就是malloc（Memory
Alllocation，内存分配）

```c
HashTable* create_table(int size)
{
    //打造柜子的外框
    HashTable* ht = (HashTable*)malloc(sizeof(HashTable));
    ht->size = size;
```


```c
    //打造柜子里的一排抽屉（钩子）
    ht->entries = (Node**)malloc(sizeof(Node*) * size);
```


```c
    //把所有钩子置空
    for (int i = 0; i < size; i++)
    {
        ht->entries[i] = NULL;
    }
```


```c
    //结束
    return ht;
}
```


- ** 打造柜子的外框**

`HashTable *ht = malloc(sizeof(HashTable))；`我们跟系统说：“嘿，给我一块足够大能装下
`HashTable` 这个结构的木头（`sizeof`
就是用来量尺寸的）！”系统划好一块地，把这块地的地址（也就是指针
`ht`）交给你。现在，柜子的空壳子做好了。

`ht->size = size；`我们在柜子外面贴个标签，写上：“这个柜子一共有
`size`（比如 10）个抽屉”。（`->`
这个符号，就是用来操作指针指向的结构体里面的东西的，你可以把它读作“的”：`ht` 的size）。

- **打招柜子里的那一排抽屉（核心重点！）**

`ht->entries = malloc(sizeof(Node*) * size);`注意看这里的尺寸，是
`sizeof(Node*)` 乘以抽屉的数量。为什么是
`Node*`（加了星号的钩子），而不是
`Node`（袋子本身）？因为在初始化的时候，**我们只装一排空挂钩，不直接造袋子**！造一个挂钩（指针）只需要一点点金属（8
个字节），但造一个大袋子（结构体）要费很多布料。等以后真的有人来存东西了，我们再去造“袋子”。

- **把所有的钩子擦干净**

`for (int i = 0; i < size; i++) { ht->entries[i] = NULL; }`这是一个循环，把
0 到 9
号抽屉里的钩子全部设置为空（`NULL`）。 在现实中，新柜子肯定是空的。但在
C
语言的内存世界里，你刚向系统要来的“木头”，可能之前被别的程序用过（比如刚跑过一个游戏，或者看过一张图片），上面会残留着**乱七八糟的垃圾数据**。如果你不把它们强行清零（设为
`NULL`），电脑在找东西的时候，可能会摸到一个“幽灵钩子”（垃圾数据），误以为那里挂着一个袋子，顺着找过去就会导致**程序直接崩溃**。这在
C 语言里叫“野指针”，是新手容易踩的坑。

- **交工**

`return ht;`柜子造好了，抽屉安上了，钩子也都擦亮了。现在把这个柜子的钥匙（指针
`ht`）交还给前台，准备开始营业！

## 4.插入（或更新）键值对

假设迎来了第一位顾客！顾客说：“我叫 **'apple'**，我要存 **100** 块钱。”

### 缝制一个新袋子（辅助函数）

```c
Node* create_node(const char* key, int value)
{
    //做一个空袋子
    Node* new_node = (Node*)malloc(sizeof(Node));
```


```c
    //准备写上名字
    new_node->key = (char*)malloc(strlen(key) + 1);
    strcpy(new_node->key, key);
```


```c
    //把钱放进去
    new_node->value = value;
```


```c
    //袋子下面的钩子暂时没有东西
    new_node->next = NULL;
```


```c
    return new_node;
}
```


- **第 1 步**：用 `malloc` 向系统要一块正好能装下 `Node`（袋子）的内存。

- **第 2 步（超级大坑 ⚠️）**：为什么存个名字 `key` 还要再用一次
  `malloc`？

  - 因为顾客报的名字 "apple"
```c
只是飘在空中的一句话，如果你不自己准备一张纸条（分配内存）把它抄下来（`strcpy`，复制字符串），顾客一走，这名字就丢了！
```


  - **为什么长度要 `+1`？** C 语言里的字符串它不知道自己有多长。所以 C
```c
语言有个死规定：每个单词的屁股后面必须跟着一个隐形的结束符号
`\0`（相当于句号）。"apple" 是 5 个字母，加上句号就是 6
个空间，所以必须 `+ 1`。
```


- **第 3、4 步**：把 100 块钱装进去，并且保证袋子底部的钩子 `next`
  是空的，别乱挂东西。

### 开始插入（挂进柜子里）

```c
void insert(HashTable* ht, const char* key, int value)
{
    //询问前台去几号抽屉
    unsigned int slot = hash(key, ht->size);
```


```c
    //走到那个抽屉看看现在的挂钩挂着什么
    Node* entry = ht->entries[slot];
```


```c
    //顺着挂钩往下翻看看有没有同一个名字的（处理冲突/更新）
    while (entry)
    {
        if (strcmp(entry->key, key) == 0)
        {
            //找到同名的袋子
            entry->value = value;//直接更新值
            return;
        }
        entry = entry->next;
    }
```


```c
    //翻到底都没有同名的，那就拿一个新袋子
    Node* new_node = create_node(key, value);
```


```c
    //采用头插法，尾插还得遍历到尾节点和分类讨论
    new_node->next = ht->entries[slot];
    ht->entries[slot] = new_node;
}
```


- `strcmp` 是 C 语言用来对比两个单词是否一模一样的工具。如果它等于
  0，说明找到了 "apple" 的袋子。这种情况说明顾客以前存过钱（比如昨天存了
  50），今天又带了 100
  来，我们要**更新**他的存款，而不是再傻乎乎地给他挂个新袋子。

- 假设 3 号抽屉现在已经挂了 "banana" 的袋子。现在我们要把新做好的
  "apple" 挂上去。

  - **普通人的思维**：顺着 "banana" 往下爬，把 "apple"
```c
挂在最底下。但如果抽屉里已经挂了 1000 个袋子呢？爬到最下面太累了！
```


  - **程序员的聪明思维（头插法）**：

```c
1.  `new_node->next = ht->entries[slot];` -> 把新 "apple"
    袋子底部的钩子，一把勾住原来挂在抽屉上的 "banana" 袋子。
```


```c
2.  `ht->entries[slot] = new_node;` -> 然后，把 "apple"
    袋子直接挂在抽屉最上面的主挂钩上！
```


  - **结果**："apple" 成功插队到了第一名，原来的那一长串顺理成章地挂在了
```c
"apple" 的下面。动作干净利落，瞬间完成！
```


## 5.查找和删除

### 查找

```c
int search(HashTable* ht, const char* key, int* out_value)
{
    //问前台要去几号柜子找
    unsigned int slot = hash(key, ht->size);
```


```c
    //走到抽屉前，抓住最上面的第一个袋子
    Node* entry = ht->entries[slot];
```


```c
    //开始顺着钩子往下找
    while (entry)
    {
        if (strcmp(entry->key, key) == 0)
        {
            *out_value = entry->value;
            return 1;//1代表找到了
        }
        entry = entry->next;
    }
    return 0;//0代表没找到
}
```


为什么参数里有个奇怪的 `int *out_value`？因为 C
语言的函数只能返回一个值。我们用 `return 1` 或 `0`
来表示“找没找到”，那找到的“钱”怎么拿出来呢？我们就让顾客自己递过来一个空信封的地址（指针
`out_value`），我们直接把钱塞进这个信封里（`*out_value = entry->value;`）

### 删除

删除操作是链表里最容易弄错的地方。想象一下，一根链子上挂了 A -> B -> C
三个袋子。如果你想把中间的 B 摘掉，你不能直接把 B 扔了，否则 C
也就跟着掉到地上找不到了！ **你必须先解开 A 的下摆挂钩，越过 B，直接挂在
C 身上，然后再把 B 扔掉。**

```c
void delete_key(HashTable* ht, const char* key)
{
    unsigned int slot = hash(key, ht->size);
    Node* entry = ht->entries[slot];//当前正在看的袋子
    Node* prev = NULL;//上一个看过的袋子（关键）
```


```c
    while (entry)
    {
        if (strcmp(entry->key, key) == 0)
        {
            //分类讨论删除情况
            if (prev == NULL)
            {
                // 情况一：它是抽屉上的第一个袋子！
                // 直接把抽屉的主挂钩，挂到它的下一个袋子上。
                ht->entries[slot] = entry->next;
            }
            else
            {
                // 情况二：它挂在半中间！
                // 让上一个袋子的钩子，越过它，直接勾住它的下一个袋子。
                prev->next = entry->next;
            }
            //顺序不能反
            free(entry->key);
            free(entry);
            return;
        }
```


```c
        //没找到的话继续往下找
        prev = entry;
        entry = entry->next;
    }
}
```


**工具人
`prev`（上一个袋子）**：在单向链表里，我们只能往下看，不能回头。所以我们必须引入一个
`prev` 指针。每次 `entry` 往下走一步之前，`prev` 都要记住 `entry`
刚才的位置。

**清理现场 (`free`)**：在 C 语言里，用 `malloc` 借来的东西，必须用
`free`
原封不动地还回去。而且**顺序不能错**！我们要先释放名牌（`free(entry->key)`），再去释放袋子本体（`free(entry)`）。如果你先把袋子烧了，名牌就成了孤魂野鬼（内存泄漏）。

## 6.清理和释放内存

```c
void free_table(HashTable* ht)
{
    //挨个拉开所有抽屉
    for (int i = 0; i < ht->size; i++)
    {
        Node* entry = ht->entries[i];//抓住当前抽屉上的第一个袋子
```


```c
        while (entry)
        {
            //先记录entry的下一个袋子，防止找不到了
            Node* next_entry = entry->next;
```


```c
            free(entry->key);
            free(entry);
```


```c
            entry = next_entry;
        }
    }
```


```c
    //所有的袋子和名字都被清空了再来清空抽屉
    free(ht->entries);
    //清空柜子,最后要在函数里面自己手动置空，写ht = NULL;
    free(ht);
}
```


## 7.测试函数

```c
int main()
{
    //造一个有10个抽屉的柜子
    HashTable* ht = create_table(10);
```


```c
    //插入内容
    insert(ht, "apple", 100);
    insert(ht, "banana", 200);
    insert(ht, "pear", 300);
```


```c
    //苹果涨价了
    insert(ht, "apple", 150);
```


```c
    //查找不同的水果的价钱（检测查找函数）
    int money = 0;
    if (search(ht, "apple", &money))
    {
        printf("找到了，apple的价格是%d\n", money);
    }
    else
    {
        printf("没找到apple\n");
    }
```


```c
    //删除
    printf("正在删除banana...\n");
    delete_key(ht, "banana");
    if (search(ht, "banana", &money))
    {
        printf("找到了，banana的价格是%d\n", money);
    }
    else
    {
        printf("没找到banana，删除成功\n");
    }
```


```c
    free_table(ht);
    ht = NULL;
    printf("柜子已释放，无内存泄露\n");
    return 0;
}
```


**关于我们上述所写的哈希表的时间复杂度的探讨**

一般我们认为哈希表的插入、查找、删除的时间复杂度都是O(1）指的是平均复杂度

但是最坏情况下，我们发现其实哈希表里面是存在while循环的，有没有可能所有袋子都装到了一起，这样时间复杂度就是O(n)了

然而这样就充分体现了我们哈希函数的强大，也就是djb算法，这个函数能尽可能的把所有袋子尽量分到不同的抽屉里面

但是在c++中存在保住哈希表O(1)的武器

- **负载因子（Load Factor）**：哈希表里有一个监控指标，等于
  `当前袋子总数 / 抽屉总数`。

- **自动扩容**：如果在营业过程中，系统发现抽屉快挂满了（比如负载因子超过了
  0.75，即 10 个抽屉挂了 7、8 个袋子），系统会触发警报！

- **大搬家**：系统会在后台偷偷买一个**大两倍**的新柜子（比如 20
  个抽屉），然后把旧柜子里的所有袋子，重新算一遍号码，均匀地挂到新柜子里。抽屉多了，链条自然就变短了。

接下来我们继续优化一下我们的哈希表，再写一个动态扩容函数

## 8.优化代码（实现动态扩容）

### 8.1先给柜子装上“检测器”

重新修改一下我们所定义的哈希表（修改一下我们的柜子）

```c
typedef struct HashTable
{
    Node** entries;//重点！这是抽屉的集合
    int size;//记录柜子一共有多少个抽屉
    int count;//新增：检测当前用了多少个抽屉
}HashTable;
```


在 `create_table` 的时候，记得加一句 `ht->count = 0;` 初始化一下。

### 8.2实现动态扩容函数（resize函数）

它的核心奥义是：**环保！绝对的环保！** 老袋子里的名字和钱完全不需要重新
`malloc`（造新房子），我们只需要**把它们从旧抽屉上摘下来，重新算个号，挂到新抽屉上就行了。**

```c
void resize(HashTable* ht)
{
    int old_size = ht->size;
    int new_size = old_size * 2;
```


```c
    //打造一排全新的抽屉
    Node** new_entries = (Node**)malloc(sizeof(Node*) * new_size);
    for (int i = 0; i < new_size; i++)
    {
        new_entries[i] = NULL;//先对每个新抽屉置空
    }
```


```c
    Node** old_entries = ht->entries;//拿到旧抽屉
```


```c
    //搬家：挨个翻旧抽屉
    for (int i = 0; i < old_size; i++)
    {
        Node* entry = old_entries[i];//抓住旧抽屉上的袋子
```


```c
        while (entry)
        {
            // 极其致命的拆迁铁律又来了：拔下来之前，先看下一个袋子是谁！
            Node* next_entry = entry->next;
```


```c
            // 重新算号 (Rehash)：
            // 以前是除以 10 取余数，现在是除以 20 取余数。
            // 比如算出来原本是 23，以前在 3 号抽屉，现在应该去 3 号抽屉；
            // 如果原本是 33，以前在 3 号抽屉，现在就要去 13 号抽屉了！
            unsigned int new_slot = hash(entry->key, new_size);
```


```c
            //头插法插入
            entry->next = new_entries[new_slot];
            new_entries[new_slot] = entry;
```


```c
            entry = entry->next;
        }
    }
```


```c
    //销毁旧抽屉
    free(old_entries);
```


```c
    // 更新系统：让大柜子换上新的核心
    ht->entries = new_entries;
    ht->size = new_size;
    // ht->count 不变！因为袋子的总数还是那么多，只是住得更宽敞了。
}
```


**为什么没有 `free(entry)`？**
因为在搬家过程中，我们**一个袋子都没扔**！所有的 `Node`
都在，它们只是换了个新老板（`new_entries`）。如果这里 `free`
了，顾客存的钱和名字就全化为灰烬了。

有人会问为什么不用realloc扩容而要用malloc

我只能说如果用了，数据不仅会逻辑丢失，还会全乱套！

我们回到“魔法算号器”的逻辑：抽屉号 =
`hash(名字) % 抽屉总数`。假设我们原来有 **10 个抽屉**。顾客 "apple"
算出来的原始哈希值是 13。**原来住哪？** `13 % 10 = 3`。所以 "apple"
被我们挂在了 **3 号抽屉**。现在，你用 `realloc`
把抽屉数组的长度直接拉长到了 **20 个**。`realloc`
的特性是“原封不动地保留原有数据”。所以 "apple" 依然安静地挂在 **3
号抽屉**下面。

**灾难发生了：** 第二天，顾客 "apple" 来取钱。前台一看，现在的抽屉总数是
20 了！**现在去哪找？** `13 % 20 = 13`。前台指挥顾客去 **13
号抽屉**找。**结果：** 13 号抽屉空空如也！

**结论：** 数据在物理内存上虽然没丢（还在 3
号抽屉），但在业务逻辑上已经**彻底失联**了！因为抽屉总数变了，取余数的规则变了，所有老顾客的“门牌号”全都得重新算！

### 8.3把扩容函数装进insert函数

负载因子 = 哈希表中存储的元素总数 (所有袋子) / 哈希表的槽位总数
(所有抽屉)

```c
void insert(HashTable *ht, const char *key, int value) 
{
    // 警报器：如果袋子数量达到了抽屉数量的 75%
    // 比如 10 个抽屉挂了 7 个以上的袋子，说明快挤爆了！
    if (ht->count >= ht->size * 0.75) 
    {
        // 瞬间触发大搬家！在后台偷偷扩容成 20 个抽屉
        resize(ht); 
    }
```


```c
    // ... 下面就是原来一模一样的查重、找抽屉、挂袋子逻辑 ...
    
    // 记得在最后，如果是挂了新袋子（不是更新老顾客），让监控器 +1
    // ht->count++;
}
```


同时，在 `delete_key`
成功删掉一个袋子时，别忘了 `ht->count--;`

## 9.最终不同文件整合

### HashTable.h

```c
#pragma once
```


```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
```


```c
typedef struct Node
{
    char* key;//你的名字（比如“apple”）
    int value;//你的存款（比如100）
    struct Node* next;//重点！用来挂下一个袋子
}Node;
```


```c
typedef struct HashTable
{
    Node** entries;//重点！这是抽屉的集合
    int size;//记录柜子一共有多少个抽屉
    int count;//新增：检测当前挂了多少个袋子
}HashTable;
```


```c
//辅助函数创建一个新节点
Node* create_node(const char* key, int value);
//动态扩容函数
void resize(HashTable* ht);
//哈希函数
unsigned int hash(const char* key, int table_size);
//初始化哈希表
HashTable* create_table(int size);
//插入（或更新）键值对
void insert(HashTable* ht, const char* key, int value);
//查找
int search(HashTable* ht, const char* key, int* out_value);
//删除
void delete_key(HashTable* ht, const char* key);
//清理和释放内存
void free_table(HashTable* ht);
```


### HashTable.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "HashTable.h"
```


```c
//辅助函数（创建一个新节点）
Node* create_node(const char* key, int value)
{
    //做一个空袋子
    Node* new_node = (Node*)malloc(sizeof(Node));
```


```c
    //准备写上名字
    new_node->key = (char*)malloc(strlen(key) + 1);
    strcpy(new_node->key, key);
```


```c
    //把钱放进去
    new_node->value = value;
```


```c
    //袋子下面的钩子暂时没有东西
    new_node->next = NULL;
```


```c
    return new_node;
}
```


```c
void resize(HashTable* ht)
{
    int old_size = ht->size;
    int new_size = old_size * 2;
```


```c
    //打造一排全新的抽屉
    Node** new_entries = (Node**)malloc(sizeof(Node*) * new_size);
    for (int i = 0; i < new_size; i++)
    {
        new_entries[i] = NULL;//先对每个新抽屉置空
    }
```


```c
    Node** old_entries = ht->entries;//拿到旧抽屉
```


```c
    //搬家：挨个翻旧抽屉
    for (int i = 0; i < old_size; i++)
    {
        Node* entry = old_entries[i];//抓住旧抽屉上的袋子
```


```c
        while (entry)
        {
            // 极其致命的拆迁铁律又来了：拔下来之前，先看下一个袋子是谁！
            Node* next_entry = entry->next;
```


```c
            // 重新算号 (Rehash)：
            // 以前是除以 10 取余数，现在是除以 20 取余数。
            // 比如算出来原本是 23，以前在 3 号抽屉，现在应该去 3 号抽屉；
            // 如果原本是 33，以前在 3 号抽屉，现在就要去 13 号抽屉了！
            unsigned int new_slot = hash(entry->key, new_size);
```


```c
            //头插法插入
            entry->next = new_entries[new_slot];
            new_entries[new_slot] = entry;
```


```c
            entry = next_entry;
        }
    }
```


```c
    //销毁旧抽屉
    free(old_entries);
```


```c
    // 更新系统：让大柜子换上新的核心
    ht->entries = new_entries;
    ht->size = new_size;
    // ht->count 不变！因为袋子的总数还是那么多，只是住得更宽敞了。
}
```


```c
unsigned int hash(const char* key, int table_size)
{
    unsigned long int value = 5381;//魔法初始数字
    unsigned int key_len = strlen(key);
```


```c
    //核心
    for (int i = 0; i < key_len; i++)
    {
        value = value * 33 + key[i];
    }
```


```c
    //挤进柜子里面
    return value % table_size;
}
```


```c
HashTable* create_table(int size)
{
    //打造柜子的外框
    HashTable* ht = (HashTable*)malloc(sizeof(HashTable));
    ht->size = size;
```


```c
    //当前还没挂入袋子
    ht->count = 0;
```


```c
    //打造柜子里的一排抽屉（钩子）
    ht->entries = (Node**)malloc(sizeof(Node*) * size);
```


```c
    //把所有钩子置空
    for (int i = 0; i < size; i++)
    {
        ht->entries[i] = NULL;
    }
```


```c
    //结束
    return ht;
}
```


```c
void insert(HashTable* ht, const char* key, int value)
{
    // 警报器：如果袋子数量达到了抽屉数量的 75%
    // 比如 10 个抽屉挂了 7 个以上的袋子，说明快挤爆了！
    if (ht->count >= ht->size * 0.75) {
        // 瞬间触发大搬家！在后台偷偷扩容成 20 个抽屉
        resize(ht);
    }
```


```c
    //询问前台去几号抽屉
    unsigned int slot = hash(key, ht->size);
```


```c
    //走到那个抽屉看看现在的挂钩挂着什么
    Node* entry = ht->entries[slot];
```


```c
    //顺着挂钩往下翻看看有没有同一个名字的（处理冲突/更新）
    while (entry)
    {
        if (strcmp(entry->key, key) == 0)
        {
            //找到同名的袋子
            entry->value = value;//直接更新值
            return;
        }
        entry = entry->next;
    }
```


```c
    //翻到底都没有同名的，那就拿一个新袋子
    Node* new_node = create_node(key, value);
```


```c
    //采用头插法，尾插还得遍历到尾节点和分类讨论
    new_node->next = ht->entries[slot];
    ht->entries[slot] = new_node;
```


```c
    //袋子数量+1
    ht->count++;
}
```


```c
int search(HashTable* ht, const char* key, int* out_value)
{
    //问前台要去几号柜子找
    unsigned int slot = hash(key, ht->size);
```


```c
    //走到抽屉前，抓住最上面的第一个袋子
    Node* entry = ht->entries[slot];
```


```c
    //开始顺着钩子往下找
    while (entry)
    {
        if (strcmp(entry->key, key) == 0)
        {
            *out_value = entry->value;
            return 1;//1代表找到了
        }
        entry = entry->next;
    }
    return 0;//0代表没找到
}
```


```c
void delete_key(HashTable* ht, const char* key)
{
    unsigned int slot = hash(key, ht->size);
    Node* entry = ht->entries[slot];//当前正在看的袋子
    Node* prev = NULL;//上一个看过的袋子（关键）
```


```c
    while (entry)
    {
        if (strcmp(entry->key, key) == 0)
        {
            //分类讨论删除情况
            if (prev == NULL)
            {
                // 情况一：它是抽屉上的第一个袋子！
                // 直接把抽屉的主挂钩，挂到它的下一个袋子上。
                ht->entries[slot] = entry->next;
            }
            else
            {
                // 情况二：它挂在半中间！
                // 让上一个袋子的钩子，越过它，直接勾住它的下一个袋子。
                prev->next = entry->next;
            }
            //顺序不能反
            free(entry->key);
            free(entry);
            ht->count--;//删除后袋子数量-1
            return;
        }
```


```c
        //没找到的话继续往下找
        prev = entry;
        entry = entry->next;
    }
}
```


```c
void free_table(HashTable* ht)
{
    //挨个拉开所有抽屉
    for (int i = 0; i < ht->size; i++)
    {
        Node* entry = ht->entries[i];//抓住当前抽屉上的第一个袋子
```


```c
        while (entry)
        {
            //先记录entry的下一个袋子，防止找不到了
            Node* next_entry = entry->next;
```


```c
            free(entry->key);
            free(entry);
```


```c
            entry = next_entry;
        }
    }
```


```c
    //所有的袋子和名字都被清空了再来清空抽屉
    free(ht->entries);
    //清空柜子,最后要在函数里面自己手动置空，写ht = NULL;
    free(ht);
}
```


### test.c

```c
#define _CRT_SECURE_NO_WARNINGS 1
```


```c
#include "HashTable.h"
```


```c
int main()
{
    //造一个有10个抽屉的柜子
    HashTable* ht = create_table(10);
```


```c
    //插入内容
    insert(ht, "apple", 100);
    insert(ht, "banana", 200);
    insert(ht, "pear", 300);
```


```c
    //苹果涨价了
    insert(ht, "apple", 150);
```


```c
    //查找不同的水果的价钱（检测查找函数）
    int money = 0;
    if (search(ht, "apple", &money))
    {
        printf("找到了，apple的价格是%d\n", money);
    }
    else
    {
        printf("没找到apple\n");
    }
```


```c
    //删除
    printf("正在删除banana...\n");
    delete_key(ht, "banana");
    if (search(ht, "banana", &money))
    {
        printf("找到了，banana的价格是%d\n", money);
    }
    else
    {
        printf("没找到banana，删除成功\n");
    }
```


```c
    free_table(ht);
    ht = NULL;
    printf("柜子已释放，无内存泄露\n");
    return 0;
}
```

