---
title: "STL-vector的模拟实现"
date: "2026-07-05T18:18:08+08:00"
draft: false
categories: ["C++"]
tags: ["STL", "vector", "模拟实现"]
---
## 1.定义vector的基本结构

在 STL 源码中，`vector` 根本没有什么 `int size` 或 `int capacity`
成员变量，而是通过迭代器来实现的

想象你租了一个仓库（连续的内存空间）来堆放快递。
在这个仓库里，你需要三个“标记牌”来管理空间：

1.  **`start`（起点）**：挂在仓库最门口，代表空间的起始位置。
2.  **`finish`（终点）**：挂在你目前堆放的最后一个快递的**下一个空位**上。代表当前物品的结尾。
3.  **`end_of_storage`（容量边界）**：挂在仓库的最深处那堵墙上。代表你租的这块地盘的极限。

**仅仅靠这三个指针**，就能推算出一整套逻辑！

**为什么 `vector`
的迭代器就是普通的指针**

因为 `vector` 的底层是连续内存。对于连续内存，原生指针（`T*`）天然就支持
`++`、`--`、`+n` 等操作，完全符合随机访问迭代器（Random Access
Iterator）的要求。所以在工业级实现中，`vector` 的迭代器直接使用模板类型
`T*` 即可。

```cpp
namespace zyh
{
template <class T>
class vector {
public:
    // vector的迭代器其实就是原生指针
    typedef T* iterator; 
    typedef const T* const_iterator;
```


```cpp
private:
    iterator _start;          // 空间头（仓库门）
    iterator _finish;         // 有效元素尾（当前快递结尾）
    iterator _end_of_storage; // 可用空间尾（仓库后墙）
```


```cpp
public:
    // 构造函数先简单初始化为空
    vector() : _start(nullptr), _finish(nullptr), _end_of_storage(nullptr) {}
```


```cpp
    // 获取当前元素个数
    size_t size() const
    {
         return _finish - _start;
    }
```


```cpp
    // 获取当前总容量
    size_t capacity() const
    {
         return _end_of_storage - _start;
    }
```


```cpp
    iterator begin()
    {
            return _start;
    }
```


```cpp
    iterator end()
    {
            return _finish;
    }
};
```


```cpp
}
```


**有人可能就会想为什么要用迭代器而不是直接用指针呢？**

**指针 `T*` 就能干活，为什么不直接写
`T*`，非要用 `typedef` 给它起个名叫 `iterator` 呢？**

**这是因为 STL 的算法（比如 `sort`, `find`）全都是“盲人”！
算法根本不知道你到底是 `vector` 还是
`list`。算法的潜规则是：“我不管你底层是什么牛鬼蛇神，只要你向我提供一个名叫
`iterator` 的内部工作牌，我就能指挥你干活。”而迭代器就是 STL
容器对外的“统一通用接口”。**

他们让 `list` 和 `map` 的作者，自己写一个复杂的类，把那些 `->next`
和在树里爬来爬去的逻辑，全部**偷偷隐藏在 `++`
符号的背后**。然后给这个类统一贴上一个标签：`iterator`。

然后写

auto it = 容器的起点;  
it++; // 算法：我不管你底层怎么变魔术，反正 ++ 就能给我下一个元素！

## 2.扩容机制（push_back 和 reserve）

1.  假如现在我们要塞入一个新的数据，我们要看一下现在的仓库是不是满了
2.  如果满了，我们就得先去租一个两倍大（或1.5倍）的新仓库
3.  然后把旧仓库的数据全搬到新仓库里面
4.  最后退掉旧仓库（释放内存）
5.  重新把 `_start`、`_finish`、`_end_of_storage`
```cpp
这三个牌子挂到新仓库对应的地方
```

6.  最后，把新的数据，安安稳稳地放在 `_finish` 指向的位置，然后把
```cpp
`_finish` 往后挪一格
```


**为什么扩容是 2 倍或 1.5 倍？**

频繁搬家是非常耗时（耗性能）的。如果满一次只多租 1
个空位，那每来一个新快递就要搬一次家，系统直接卡死。

- **GCC（Linux下常用的编译器）**：简单粗暴，满了一律按 **2 倍**扩容。
- **MSVC（Windows下的 VS）**：稍微精打细算，按 **1.5 倍**扩容。
  咱们这里为了好算，采用 GCC 的 **2 倍扩容大法**！

<!-- -->

```cpp
        // 请求开辟新容量
        void reserve(size_t n)
        {
            if (n > capacity())
            {
                T *tmp = new T[n];
                // 提前记下旧仓库里有多少件快递（极其关键！因为等下 _start 换成新仓库后，就没法用旧距离算了）
                size_t old_size = size();
```


```cpp
                if (_start != nullptr)
                {
                    for (size_t i = 0; i < old_size; ++i)
                    {
                        tmp[i] = _start[i];
                    }
                    delete[] _start;
                }
```


```cpp
                _start = tmp;
                _finish = _start + old_size;
                _end_of_storage = _start + n;
            }
        }
```


```cpp
        // 尾部插入新元素
        void push_back(const T &x)
        {
            // 检查仓库是不是满了 (当前结尾已经顶到了可用空间尾)
            if (_finish == _end_of_storage)
            {
                size_t new_capacity = capacity() == 0 ? 4 : capacity() * 2;
                reserve(new_capacity);
            }
```


```cpp
            *_finish = x;
            ++_finish;
        }
```


然后我们测试一下我们写的代码

```cpp
#include <iostream>
#include "MyVector.h"
```


```cpp
int main() {
    zyh::vector<int> v;
```


```cpp
    // 阶段 1：初始状态
    std::cout << "--- 阶段 1 ---" << std::endl;
    std::cout << "Size: " << v.size() << ", Capacity: " << v.capacity() << std::endl;
```


```cpp
    // 阶段 2：连续插入 4 个元素
    v.push_back(10);
    v.push_back(20);
    v.push_back(30);
    v.push_back(40);
    std::cout << "--- 阶段 2 ---" << std::endl;
    std::cout << "Size: " << v.size() << ", Capacity: " << v.capacity() << std::endl;
```


```cpp
    // 阶段 3：插入第 5 个元素（触发扩容！）
    v.push_back(50);
    std::cout << "--- 阶段 3 ---" << std::endl;
    std::cout << "Size: " << v.size() << ", Capacity: " << v.capacity() << std::endl;
```


```cpp
    // 阶段 4：遍历打印
    std::cout << "--- 阶段 4 元素内容 ---" << std::endl;
    // 利用我们刚学过的：iterator 就是原生指针，直接用它来遍历！
    for (zyh::vector<int>::iterator it = v.begin(); it != v.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
```


```cpp
    return 0;
}
```


##### 拓展内容

在刚才的 `reserve` 里，我们用了这行代码：T\* tmp = new T[n];

这行代码在 C++ 底层实际上偷偷干了两件事：

1.  **分配内存**：在堆上开辟能容纳 `n` 个 `T`
```cpp
类型对象的连续空间（这没问题）。
```


2.  **强制构造**：编译器会急不可耐地**自动调用这 `n`
```cpp
个位置的默认构造函数（Default Constructor）**！
```


这就带来了两个在工业界无法忍受的致命问题：

- 性能大浪费（无效劳动）

假设你 `reserve(1000)`，你只是想预留 1000 个位置，现在里面还是空的。 但
`new T[1000]` 会强行把这 1000
个位置全部塞满“垃圾默认值”（如果是自定义的复杂类，就会白白调用 1000
次构造函数）。 接着你调用
`push_back`，又把原本默认构造好的对象给**覆盖赋值**了一遍。
**先构造、再覆盖，前一次的构造完全是无用功！**

- 直接编译崩溃（致命缺陷）

如果用户定义的类，**根本没有默认构造函数**呢？比如下面这个：

```cpp
class Person
{
public:
    Person(int age) {} // 只有带参构造函数，没有默认构造函数
};
```


如果你尝试创建
`zyh::vector<Person> v; v.reserve(10);`，你的代码会**直接报错编译失败**！因为
`new Person[10]` 找不到默认构造函数，它罢工了。但真正的 `std::vector`
是绝对允许存这种类的。

**【真实的 STL 哲学：内存管理与对象构造彻底解耦】**

SGI STL
的核心设计哲学是：**把“开辟内存”和“构造对象”这两件事，死死地拆开！**

- **要空间时**：只给我最纯净的、没有经过任何污染的“原始裸内存（Raw
  Memory）”，**绝对不准调用构造函数**。

- **塞物品时（`push_back`）**：在指定的、干净的裸内存位置上，精准地**把对象构造出来**。

为此，我们需要引入 C++
底层最硬核的武器：**`::operator new`（只管申请内存）** 和
**`placement new`（定位放置构造）**。

升级版代码

```cpp
        // 请求开辟新容量
        void reserve(size_t n)
        {
            if (n > capacity())
            {
                // 【工业级规范】只申请裸内存，不调用任何 T 的构造函数！
                T *tmp = (T *)::operator new(n * sizeof(T));
                // 提前记下旧仓库里有多少件快递（极其关键！因为等下 _start 换成新仓库后，就没法用旧距离算了）
                size_t old_size = size();
```


```cpp
                if (_start != nullptr)
                {
                    for (size_t i = 0; i < old_size; ++i)
                    {
                        tmp[i] = _start[i];
                        // 【工业级规范】旧仓库的快递搬走了，必须手动调用析构函数把它销毁
                        _start[i].~T();
                    }
```


```cpp
                    // 【工业级规范】因为是裸内存，退租时必须用 ::operator delete
                    ::operator delete(_start);
                }
```


```cpp
                _start = tmp;
                _finish = _start + old_size;
                _end_of_storage = _start + n;
            }
        }
```


```cpp
        // 尾部插入新元素
        void push_back(const T &x)
        {
            // 检查仓库是不是满了 (当前结尾已经顶到了可用空间尾)
            if (_finish == _end_of_storage)
            {
                size_t new_capacity = capacity() == 0 ? 4 : capacity() * 2;
                reserve(new_capacity);
            }
```


```cpp
            new (_finish) T(x); // 含义：就在 _finish 这个地址上，调用 T 的拷贝构造函数，原地生出一个新对象
            ++_finish;
        }
```


尾插的时候也不能再写\*\_finish = x;

现在的 `_finish` 指向的是通过 `::operator new`
刚申请出来的**裸内存荒地**！上面根本没有老电视，只有地底下埋着的随机垃圾二进制（乱码）。

当你对一头指针用 `=` 赋值时

`=` 的内部逻辑是：

- **第一步（拆迁）**：先把旧的数据砸了，把它的内部申请的堆内存、文件句柄等资源全部销毁释放。

- **第二步（入住）**：把新数据的东西复制进来。

现在的 `_finish` 指向的是通过 `::operator new`
刚申请出来的**裸内存荒地**！上面根本没有老电视，只有地底下埋着的随机垃圾二进制（乱码）。

这时候拆迁队（`=`）冲过去，对着一堆空气和乱码一通乱砸，试图去释放堆内存、文件句柄等资源——**轰！程序当场内存非法访问，直接崩溃！**

## 3.带参构造函数与析构函数

```cpp
        // 带参构造函数：创建 n 个大小的容器，并用 value 初始化
        vector(size_t n, const T &value = T())
        {
            _start = (T *)::operator new(n * sizeof(T));
```


```cpp
            for (size_t i = 0; i < n; ++i)
            {
                new (_start + i) T(value);
            }
```


```cpp
            _finish = _start + n;
            _end_of_storage = _start + n;
        }
```


```cpp
        // 析构函数
        ~vector()
        {
            if (_start != nullptr)
            {
                for (iterator it = _start; it != _finish; ++it)
                {
                    it->~T();
                }
```


```cpp
                ::operator delete(_start);
                _start = _finish = _end_of_storage = nullptr;
            }
        }
```


**为什么析构函数中间还要手动析构呢
？**

如果仓库里装的是 `int`、`double`
这种没有灵魂的普通数字，你说的完全对！就算不写这个 `for`
循环手动析构，也绝对不会有任何问题。

但是，**如果仓库里装的是 `std::string` 或者另一个 `vector`
这种有“私产”的复杂对象呢？**

`std::string` 内部藏了一个指针，指向了**另外一片**属于它自己的堆内存。

如果我们不写中间的 `for` 循环，直接执行
`::operator delete(_start);`，底层会发生这样的惨剧：

1.  **`::operator delete(_start)` 执行：** 操作系统收回了 `vector`
```cpp
的连续地皮。
```


2.  **后果：** 存放在这块地皮上的 `string` 对象本身确实消失了。

3.  **灾难发生：** 但是！这些 `string`
```cpp
对象在临死前**没有调用它们自己的析构函数**，导致它们自己偷偷在外面申请的那片堆内存（真正存字符串文本的地方）**彻底变成了无人认领的孤魂野鬼**！
```


这就是典型的**内存泄漏（Memory
Leak）**。你虽然把房子的地基拆了，但由于没有走正常的“断水断电”毁灭程序，房子里连着的远程服务器和外部资源永远被挂起了。

| **阶段** | **动作** | **负责的事情** |
|----|----|----|
| **生** | 1\. `::operator new` | 仅仅向系统申请裸内存（建毛坯房） |
| **生** | 2\. `placement new` | 在裸内存上调用构造函数（精准注入灵魂，让对象活过来） |
| **死** | **3. `it->~T()`** | **手动调用析构函数（让对象交代后事，释放自己的私产资源）** |
| **死** | 4\. `::operator delete` | 仅仅把裸内存还给系统（把毛坯房推倒） |

- **`new` 关键字** = 自动调用 `::operator new` 开辟内存 + 自动调用
  `T(x)` 构造函数。
- **`delete` 关键字** = **自动调用 `~T()` 析构函数** + 自动调用
  `::operator delete` 释放内存。

## 4.拷贝构造函数

```cpp
zyh::vector<std::string> v1(5, "hello"); // 正常构造，v1 的 _start 指向地址 0x10
zyh::vector<std::string> v2(v1);         // 触发了编译器的默认浅拷贝，v2 的 _start 也指向了 0x10
```


当我们这样子写了以后，操作系统会发生浅拷贝，在释放内存的时候就会释放两次，引发重复释放的报错，也就是段错误

我们也就需要手动实现深拷贝

编译器默认的“浅拷贝”只会傻傻地复制指针（导致两人共享同一块地皮），那我们就必须亲自接管，写一个工业级的**拷贝构造函数（Copy
Constructor）**。

核心思想就三个字：**“另起炉灶”**。 当 `v2` 想要拷贝 `v1` 时：

1.  `v2` 必须自己去向操作系统申请一块**全新的、大小一样的裸内存地皮**。
2.  然后把 `v1` 地皮上的物品，一个一个地**复制构造**到自己的新地皮上。
3.  这样，两人各自拥有一块完全独立的地皮，死的时候各回各家，各删各找，完美避开
```cpp
Double Free！
```


<!-- -->

```cpp
        // 拷贝构造函数：深拷贝
        vector(const vector<T> &v)
        {
            _start = (T *)::operator new(v.capacity() * sizeof(T));
```


```cpp
            for (size_t i = 0; i < v.size(); ++i)
            {
                new (_start + i) T(v._start[i]);
            }
```


```cpp
            _finish = _start + v.size();
            _end_of_storage = _start + v.capacity();
        }
```


接下来我们进行测试一下

```cpp
#include <iostream>
#include <string>
#include "MyVector.h"
```


```cpp
int main()
{
    std::cout << "=== 阶段 1：构建 v1 ===" << std::endl;
    // 创建一个装了 3 个 "Hello" 的容器
    zyh::vector<std::string> v1(3, "Hello"); 
    
    std::cout << "=== 阶段 2：触发深拷贝构建 v2 ===" << std::endl;
    // 这里会精准调用你刚刚手写的【拷贝构造函数】！
    zyh::vector<std::string> v2(v1); 
    
    std::cout << "=== 阶段 3：验证物理隔离 ===" << std::endl;
    // 给 v2 单独尾插一个新字符串，看看会不会影响 v1
    v2.push_back("World");
```


```cpp
    // 打印 v1
    std::cout << "v1 (size: " << v1.size() << ") 元素: ";
    for(zyh::vector<std::string>::iterator it = v1.begin(); it != v1.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
```


```cpp
    // 打印 v2
    std::cout << "v2 (size: " << v2.size() << ") 元素: ";
    for(zyh::vector<std::string>::iterator it = v2.begin(); it != v2.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
```


```cpp
    std::cout << "=== 阶段 4：程序结束，见证奇迹 ===" << std::endl;
    // 当 main 函数结束（遇到 return 0）时：
    // v2 会先调用析构函数，销毁自己的地皮和字符串。
    // v1 会后调用析构函数，销毁自己的地皮和字符串。
    // 因为你写了深拷贝，这里绝对不会出现 Double Free（段错误）！程序将完美退出。
    return 0;
}
```


## 5.拷贝赋值运算符

```cpp
zyh::vector<std::string> v3; // 已经存在的空容器
v3 = v1;                     // 赋值操作
```


如果这样子写，我们将又会面临浅拷贝构造的风险

- 如果对象**还没诞生**，正在创建中（比如 `vector v2(v1);` 或
  `vector v2 = v1;`），调用的是**拷贝构造函数**。
- 如果对象**已经存在**，是一个老油条了（比如 `v3`
  早就被创建出来了），此时写 `v3 = v1;`，调用的是**拷贝赋值运算符（Copy
  Assignment Operator，也就是 `operator=`）**。

如果此时编译器执行默认的浅拷贝（直接把 `v1` 的 `_start` 赋值给 `v3` 的
`_start`），会引发极其惨烈的**双重灾难**：

1.  **内存泄漏**：`v3`
```cpp
原本手里那块地皮的指针被覆盖了，导致那块地皮永远无法被回收，变成了赛博空间里的垃圾！
```

2.  **Double Free（段错误）**：`v3` 的指针现在指向了 `v1`
```cpp
的地皮。等程序结束时，两人又会对同一块地皮释放两次，系统直接崩溃！
```


实现的具体步骤如下：

1.  **防范于未然**：判断是不是自己给自己赋值（比如写了
```cpp
`v1 = v1;`）。如果是，千万别动，否则会把自己提前砸没了。
```

2.  **清理门户**：把 `v3`
```cpp
自己原来的物品全杀光，地皮全退掉（和析构函数干的活一模一样）。
```

3.  **另起炉灶**：根据 `v1` 的大小，申请一块全新的地皮。
4.  **搬运货物**：把 `v1` 里的东西用 placement new 一个个深拷贝过来。

<!-- -->

```cpp
        // 拷贝赋值运算符重载
        vector<T> &operator=(const vector<T> &v)
        {
            if (this != &v)
            {
                if (_start != nullptr)
                {
                    for (iterator it = _start; it != _finish; ++it)
                    {
                        it->~T();
                    }
```


```cpp
                    ::operator delete(_start);
                }
```


```cpp
                // 另起炉灶：根据别人的容量申请新地皮
                _start = (T *)::operator new(v.capacity() * sizeof(T));
```


```cpp
                for (size_t i = 0; i < v.size(); ++i)
                {
                    new (_start + i) T(v._start[i]);
                }
```


```cpp
                _finish = _start + v.size();
                _end_of_storage = _start + v.capacity();
            }
```


```cpp
            // 返回自己的引用，支持连续赋值 (比如 v3 = v2 = v1;)
            return *this;
        }
```


##### 现代 STL 架构师的降维打击：现代写法（Copy-and-Swap）

现代 C++ 架构师发明了一种极其优雅的招式，叫做
**Copy-and-Swap（拷贝并交换）技术**。

```cpp
        // 【现代级神仙写法】利用编译器和现有的拷贝构造函数
        vector<T> &operator=(vector<T> v) // 注意：这里故意【不用引用】，而是【传值】！！！
        {
            // 利用 swap 交换两者的三个门牌号指针
            swap(_start, v._start);
            swap(_finish, v._finish);
            swap(_end_of_storage, v._end_of_storage);
```


```cpp
            return *this;
        }
```


**底层逻辑**

第一次看到这个写法的人，肯定会有疑惑🤨。没有任何的 `delete`，没有任何的
`new`，甚至连自赋值检查都没写，它凭什么能完成“深拷贝”和“释放旧内存”这两大重任？

1.  **借尸还魂（触发拷贝构造）**： 注意看函数的形参
```cpp
`vector<T> v`，这里**没有写 `&`
符号**！这意味着在进入函数体之前，编译器会强行调用你写好的【拷贝构造函数】，在栈上临时深拷贝构造出一个名为
`v` 的临时小老弟。此时 `v` 手里拿着一份和 `v1`
一模一样的独立新资产。
```

2.  **李代桃僵（指针交换 swap）**： 进入函数后，我们用 `std::swap` 把
```cpp
`v3` 现在的三个指针和临时小老弟 `v`
的三个指针**进行了大风吹式的对调**。 对调之后：`v3` 成功拿到了 `v`
手里的新资产（也就是 `v1` 的深拷贝）。而临时小老弟 `v` 倒霉地接盘了
`v3` 原来的那一堆旧烂摊子资产。
```

3.  **借刀杀人（自动触发析构）**： 当 `operator=`
```cpp
执行到最后大括号，函数结束时，临时变量 `v`
的生命周期到头了。编译器会**自动调用 `v` 的析构函数 `~vector()`**。
因为指针已经交换了，此时 `v` 的析构函数会顺理成章地、开开心心地把你
`v3` 原本不要的旧资产给**彻底销毁、夷为平地**！
```


好处

- **绝对的代码复用**：你只要写好了“拷贝构造”和“析构”，赋值运算符就直接白嫖它们，不需要写任何内存分配的代码。
- **异常安全（Exception
  Safety）**：如果在深拷贝时系统内存不足崩溃了，它会死在形参拷贝阶段，绝对不会污染和破坏
  `v3` 原本的数据。

接下来我们再进行 测试一下

```cpp
#include <iostream>
#include <string>
#include "MyVector"
```


```cpp
int main()
{
    std::cout << "=== 阶段 1：准备测试数据 ===" << std::endl;
    // v1 是我们要被抄袭的“原件”
    zyh::vector<std::string> v1(2, "Apple"); 
    
    // v3 是一个“老油条”，里面已经占用了地皮，存了 3 个 "Banana"
    zyh::vector<std::string> v3(3, "Banana"); 
    
    std::cout << "=== 阶段 2：触发赋值运算符 ===" << std::endl;
    // 关键时刻！这里会触发 operator=
    // v3 必须先销毁自己的 3 个 "Banana"，退掉旧地皮，再去深拷贝 v1 的 2 个 "Apple"
    v3 = v1; 
    
    std::cout << "=== 阶段 3：测试连续赋值 (Chain Assignment) ===" << std::endl;
    zyh::vector<std::string> v4;
    // 这里先执行 v3 = v1，然后返回 v3 的引用，再执行 v4 = v3
    v4 = v3 = v1; 
    
    std::cout << "=== 阶段 4：验证物理隔离 ===" << std::endl;
    // 我们往 v3 里加点新东西，看看 v1 和 v4 会不会跟着变
    v3.push_back("Orange");
```


```cpp
    // --- 打印结果 ---
    std::cout << "v1 (size: " << v1.size() << ") 元素: ";
    for(zyh::vector<std::string>::iterator it = v1.begin(); it != v1.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
```


```cpp
    std::cout << "v3 (size: " << v3.size() << ") 元素: ";
    for(zyh::vector<std::string>::iterator it = v3.begin(); it != v3.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
```


```cpp
    std::cout << "v4 (size: " << v4.size() << ") 元素: ";
    for(zyh::vector<std::string>::iterator it = v4.begin(); it != v4.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
```


```cpp
    std::cout << "=== 阶段 5：程序结束，见证奇迹 ===" << std::endl;
    // 程序退出时，v4, v3, v1 将依次调用析构函数。
    // 如果没有段错误（Core Dump），说明我们不仅没有 Double Free，也没有内存泄漏！
    return 0;
}
```


## 6.任意位置**插入**

我们先补充两个下标访问的函数

```cpp
        // 下标访问（普通版：可读可写）
        T& operator[](size_t i)
        {
            return _start[i];
        }
```


```cpp
        // 下标访问（const版：只读）
        const T& operator[](size_t i) const
        {
            return _start[i];
        }
```


既然 `push_back`
是在尾部追加，那真正的通用容器必须支持在**任意位置插入（`insert`）。**

`insert` 的底层逻辑听起来很简单：

1.  看看仓库满了没有，满了就扩容（`reserve`）。
2.  从插入位置 `pos` 开始，把后面的元素全部往后挪一个格子，腾出空间。
3.  在 `pos` 位置上把新元素放进去。

然后我们看一个常见的错误代码

```cpp
        // 任意位置插入
        iterator insert(iterator pos, const T& x)
        {
            // 1. 检查扩容
            if (_finish == _end_of_storage)
            {
                size_t new_capacity = capacity() == 0 ? 4 : capacity() * 2;
                reserve(new_capacity); 
            }
```


```cpp
            // 2. 挪动数据：把 pos 及它后面的元素全部往后挪一格
            iterator end_it = _finish - 1;
            while (end_it >= pos)
            {
                // 注意：这里最严谨的做法是判断是不是纯裸内存。
                // 为了简单说明逻辑，暂且先用 placement new 和赋值来处理
                if (end_it == _finish - 1)
                {
                    new (end_it + 1) T(*end_it); // 最后一个元素挪到裸内存上
                }
                else
                {
                    *(end_it + 1) = *end_it;     // 其他元素在已有对象的内存上挪动
                }
                --end_it;
            }
```


```cpp
            // 3. 填入新元素
            *pos = x; 
            ++_finish;
```


```cpp
            return pos;
        }
```


这里简单一看可能好像没什么错误，但其实藏着一个 C++
领域臭名昭著、足以让你在腾讯/字节跳动面试中直接被挂掉的致命大
Bug（迭代器失效问题）

1.  `pos` 作为一个指针，最初指向的是旧地皮上的某个位置（比如 `0x104`）。
2.  `reserve` 偷偷在底层把整个仓库搬到了新地皮（比如
```cpp
`0x500`），并且**直接把旧地皮 `0x100` 给 `delete` 夷为平地了**。
```

3.  但 `pos`
```cpp
这个指针是个“铁憨憨”，它并不知道家已经搬了，它依然死死指着已经被释放的
`0x104`！
```

4.  接下来你往 `*pos` 写入数据，就是经典的 **野指针访问 /
```cpp
Use-After-Free**，程序当场段错误崩溃！
```


**解决方法：记住相对距离（Offset）**

很简单：在搬家之前，我们先拿个小本本记下 `pos`
距离仓库大门（`_start`）到底有几个格子的**相对距离**。搬完家之后，在新仓库的大门基础上，重新把这个距离加上去，就能找到
`pos` 的新位置了！

修复过后的代码

```cpp
        // 任意位置插入
        iterator insert(iterator pos, const T &x)
        {
            if (_finish == _end_of_storage)
            {
                // 搬家前，先算好 pos 距离 _start 的偏移量
                size_t offset = pos - _start;
```


```cpp
                size_t new_capacity = capacity() == 0 ? 4 : capacity() * 2;
                reserve(new_capacity);
```


```cpp
                // 搬家后，刷新 pos，让它指向新地皮上的正确位置！
                pos = _start + offset;
            }
```


```cpp
            // 挪动数据：把 pos 及它后面的元素全部往后挪一格
            iterator end_it = _finish - 1;
            while (end_it >= pos)
            {
                // 注意：这里最严谨的做法是判断是不是纯裸内存。
                // 为了简单说明逻辑，暂且先用 placement new 和赋值来处理
                if (end_it == _finish - 1)
                {
                    new (end_it + 1) T(*end_it);
                }
                else
                {
                    *(end_it + 1) = *end_it;
                }
                --end_it;
            }
            *pos = x;
            ++_finish;
```


```cpp
            return pos;
        }
```


接下来进入测试

```cpp
#include <iostream>
#include <string>
#include "MyVector.h"
```


```cpp
int main()
{
    std::cout << "=== 阶段 1：测试 push_back 和 operator[] ===" << std::endl;
    zyh::vector<std::string> v;
    
    // 连续放入 4 个元素，假设此时容量刚好达到了 4
    v.push_back("A");
    v.push_back("B");
    v.push_back("C");
    v.push_back("D"); 
    
    std::cout << "原始元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        std::cout << v[i] << " "; // 测试 operator[]
    }
    std::cout << "\n\n";
```


```cpp
    std::cout << "=== 阶段 2：触发致命危机 (扩容 + 插入) ===" << std::endl;
    // 此时 v 的 size 是 4，capacity 也是 4，仓库已满！
    // 我们获取一个指向第二个元素 "B" 的迭代器（此时它指向旧地皮）
    zyh::vector<std::string>::iterator it = v.begin() + 1; 
    
    std::cout << "准备在 'B' 的位置前面插入一个 'X'..." << std::endl;
    // 关键时刻！
    // insert 内部发现满载，会调用 reserve 申请新地皮并销毁旧地皮。
    // 如果没有我们的 offset 修复逻辑，这里继续使用 it 就会直接发生段错误崩溃！
    v.insert(it, "X"); 
    std::cout << "插入成功！没有发生崩溃！\n\n";
```


```cpp
    std::cout << "=== 阶段 3：验证挪动结果 ===" << std::endl;
    std::cout << "插入后 vector 大小: " << v.size() << "，容量: " << v.capacity() << std::endl;
    std::cout << "最终元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        std::cout << v[i] << " ";
    }
    std::cout << "\n";
    
    return 0;
}
```


## 7.任意位置删除

`erase` 的逻辑听起来和 `insert` 刚好相反：

1.  从 `pos`
```cpp
位置的下一个元素开始，把后面的元素全部**往前挪一格**，直接覆盖掉要删除的那个元素。
```

2.  整个队伍往前缩了一格，那最后空出来的那一个位置怎么办？必须让它“安乐死”（调用析构函数）。
3.  最后把 `_finish` 指针往回退一格。

这里有一个极其绝妙的底层细节！当你把后面的元素往前挪（执行
`*(it - 1) = *it;`）时，由于用的是赋值运算符，它会自动释放前一个位置（旧元素）的深拷贝资源。所以被删除的元素其实在挪动过程中就已经“自然死亡”了。
**我们真正需要手动释放的，反而是往前挪动之后，留在原地的那个“尾巴分身”**

```cpp
        // 任意位置删除
        iterator erase(iterator pos)
        {
            if (pos >= _start && pos < _finish)
            {
                // 挪动数据：从 pos 的下一个位置开始，全部往前挪一格
                iterator it = pos + 1;
                while (it != _finish)
                {
                    *(it - 1) = *it;
                    ++it;
                }
```


```cpp
                --_finish;
```


```cpp
                _finish->~T();
            }
            // C++ 官方标准规定：erase 必须返回被删除元素的【下一个元素】的新位置
            // 因为后面的元素全往前挪了，所以原本的 pos 指向的正好就是新的下一个元素
            return pos;
        }
```


## 8.尾删和清空和判空

```cpp
        // 尾删函数
        void pop_back()
        {
            if (_start != _finish)
            {
                erase(_finish - 1);
            }
        }
```


```cpp
        // 清空所有元素
        void clear()
        {
            while (_start != _finish)
            {
                --_finish;
                _finish->~T();
            }
        }
```


```cpp
        // 判断容器是否为空
        bool empty() const
        {
            return _start == _finish;
        }
```


测试函数

```cpp
#include <iostream>
#include <string>
#include "MyVector"
```


```cpp
int main()
{
    zyh::vector<std::string> v;
    v.push_back("Apple");
    v.push_back("Banana");
    v.push_back("Cherry");
    v.push_back("Date");
```


```cpp
    std::cout << "=== 阶段 1：原始数据 ===" << std::endl;
    std::cout << "v (size: " << v.size() << ") 元素: ";
    for (size_t i = 0; i < v.size(); ++i) { std::cout << v[i] << " "; }
    std::cout << "\n\n";
```


```cpp
    std::cout << "=== 阶段 2：测试 erase (删除 Banana) ===" << std::endl;
    // 获取指向 "Banana" 的迭代器 (begin + 1)
    zyh::vector<std::string>::iterator it = v.begin() + 1;
    
    // 删除它，并接收返回的新迭代器
    it = v.erase(it); 
    
    std::cout << "删除后 v (size: " << v.size() << ") 元素: ";
    for (size_t i = 0; i < v.size(); ++i) { std::cout << v[i] << " "; }
    std::cout << "\n";
    // 验证返回值是不是指向了原本 Banana 后面的 Cherry
    std::cout << "erase 返回的迭代器现在指向: " << *it << "\n\n";
```


```cpp
    std::cout << "=== 阶段 3：测试 pop_back 和 clear ===" << std::endl;
    v.pop_back(); // 删掉 Date
    std::cout << "pop_back 后元素: ";
    for (size_t i = 0; i < v.size(); ++i) { std::cout << v[i] << " "; }
    std::cout << "\n";
```


```cpp
    v.clear(); // 全部清空
    std::cout << "clear 后，size: " << v.size() << "，capacity: " << v.capacity() << std::endl;
```


```cpp
    return 0;
}
```


## 9.对比reserve函数和resize函数

- **`reserve(n)`（扩容）**：纯纯的“房地产开发商”。只负责向系统买一块能容纳
  `n` 个元素的**裸地皮**，绝对不在上面建任何房子。它只改变
  `capacity`，**绝不改变 `size`**。
- **`resize(n, val)`（改变大小）**：真正的“包工头”。它不仅要保证地皮够大，还要真刀真枪地在地皮上**建房子或者拆房子**！它会直接改变
  `size`。

resize的两种情况

1.  **情况 1：缩小规模（`n < size()`）** 比如原本有 5 个元素，用户要求
```cpp
`resize(3)`。 此时包工头根本不需要管地皮，他只需要把多出来的 2
个房子“拆除（调用析构函数 `~T()`）”，然后把 `_finish` 指针往回退 2
格就行了。
```

2.  **情况 2：扩大规模（`n > size()`）** 比如原本有 2 个元素，用户要求
```cpp
`resize(5, "Hello")`。包工头先看地皮够不够大，如果 `capacity` 小于
5，就先呼叫开发商 `reserve(5)` 买地。        然后，包工头拿出图纸
`"Hello"`，在空出来的 3 块地皮上，使用**拷贝构造函数（Placement
New）**，一口气建起 3 座一模一样的房子！
```


<!-- -->

```cpp
        // 调整有效元素个数
        void resize(size_t n, const T &val = T())
        {
            if (n < size())
            {
                // 情况 1：缩小。多出来的元素我们要让它们“安乐死”
                while (size() > n)
                {
                    --_finish;
                    _finish->~T();
                }
            }
            else if (n > size())
            {
                // 情况 2：放大。先看地皮够不够
                if (n > capacity())
                {
                    reserve(n);
                }
```


```cpp
                while (size() < n)
                {
                    push_back(val);
                }
            }
        }
```


最后我们再进行测试一下

```cpp
#include <iostream>
#include <string>
#include "MyVector.h"
```


```cpp
int main()
{
    zyh::vector<std::string> v;
```


```cpp
    std::cout << "=== 阶段 1：测试 empty ===" << std::endl;
    if (v.empty())
    {
        std::cout << "刚初始化的 vector 是空的！(验证成功)\n\n";
    }
```


```cpp
    std::cout << "=== 阶段 2：填入初始数据 ===" << std::endl;
    v.push_back("Apple");
    v.push_back("Banana");
    std::cout << "当前 size: " << v.size() << "，capacity: " << v.capacity() << std::endl;
    std::cout << "元素: ";
    for (size_t i = 0; i < v.size(); ++i) { std::cout << v[i] << " "; }
    std::cout << "\n\n";
```


```cpp
    std::cout << "=== 阶段 3：测试 resize 放大 (带默认参数) ===" << std::endl;
    // 原本只有 2 个元素，要求扩到 5 个。会补充 3 个 "Hello"
    v.resize(5, "Hello");
    std::cout << "resize(5, 'Hello') 之后 size: " << v.size() << "，capacity: " << v.capacity() << std::endl;
    std::cout << "元素: ";
    for (size_t i = 0; i < v.size(); ++i) { std::cout << v[i] << " "; }
    std::cout << "\n\n";
```


```cpp
    std::cout << "=== 阶段 4：测试 resize 缩小 ===" << std::endl;
    // 从 5 个缩减到 1 个。尾部的 4 个元素必须被执行析构函数（安乐死）
    v.resize(1);
    std::cout << "resize(1) 之后 size: " << v.size() << "，capacity: " << v.capacity() << std::endl;
    std::cout << "元素: ";
    for (size_t i = 0; i < v.size(); ++i) { std::cout << v[i] << " "; }
    std::cout << "\n\n";
```


```cpp
    std::cout << "=== 阶段 5：清理门户 ===" << std::endl;
    v.clear();
    std::cout << "clear() 之后，empty() 返回: " << (v.empty() ? "true" : "false") << std::endl;
    std::cout << "此时 capacity (地皮) 依然是: " << v.capacity() << std::endl;
    
    std::cout << "\n=== 毕业典礼：程序完美结束，没有段错误！ ===" << std::endl;
```


```cpp
    return 0;
}
```


## 10.总结

#### 铁律一：分清“买地”与“建房”（底层的双剑客）

在 C++ 的裸内存操作中，千万不能把“分配内存”和“构造对象”混为一谈

| **行为** | **内存操作（买卖地皮）** | **对象操作（建房/拆房）** |
|----|----|----|
| **生** | `::operator new`（只向系统批裸地皮，绝不调构造） | `placement new`（在指定的裸地皮上，调用构造函数建房） |
| **死** | `::operator delete`（把整块裸地皮还给系统） | `~T()`（手动调用析构函数，拔电源拆房） |

#### 铁律二：守护内存安全的护城河

只要你的类里管理了堆区资源（写了析构函数释放内存），你就必须防范编译器默认的“浅拷贝”带来的两大死神：**Double
Free（段错误）** 和 **内存泄漏**

1.  **拷贝构造函数 `vector(const vector& v)`**：**场景：**
```cpp
`vector v2(v1);`**核心：** 另起炉灶，申请新地皮，用 `placement new`
把老物件一个个**深拷贝**过来。
```

2.  **赋值运算符重载 `operator=`**：**场景：** `v2 = v1;`**核心：**
```cpp
先破后立。先销毁自己原有的旧房子并退掉旧地皮，再去深拷贝别人的。**进阶大招：**
熟练使用 **Copy-and-Swap**
技巧，利用形参的临时深拷贝对象，交换指针后借尸还魂，让临时对象替你销毁旧资产（4
行代码搞定）。
```


#### 铁律三：“迭代器失效”大魔王（极其危险的 `insert`）

在执行 `insert` 时，如果容量满了，底层会触发 `reserve` 扩容。

- **死亡过程：** `reserve` 会去买新地皮，搬家，并把旧地皮 `delete`
  掉。此时你手里拿着的那个指向插入位置的 `pos`
  迭代器，就变成了一个指向已被释放内存的**野指针**。继续使用直接崩溃。

- **破解之法：** **记录 Offset（相对距离）**。在搬家前，算好
  `pos - _start`。搬完家后，在新地皮上加上这个偏移量，让 `pos`
  重新定位。

#### 铁律四：只杀对象，不退地皮（删除操作的潜规则）

当实现 `erase`、`pop_back` 或 `clear`
时，我们仅仅是在缩减“队伍的有效长度”。

- **操作：** 只需要把 `_finish`
  往回退，并对多出来的那些房子**手动调用析构函数 `_finish->~T()`**。
- **禁忌：** **绝对不能调用 `delete`！**
  因为底层的地皮是一整块连续的，不能只退中间的一小部分给操作系统，否则会导致系统堆区账本损坏（Heap
  Corruption）。留着那块空地（维持 `capacity`），等下次 `push_back`
  继续用。

#### 铁律五：认清包工头与开发商（`resize` vs `reserve`）

- **开发商 `reserve(n)`**：只管买地皮。如果
  `n > capacity`，就申请一块能容纳 n
  个元素的裸内存，搬家过去。**它绝不改变 `size`，也就是不建房子**。
- **包工头 `resize(n, val)`**：实打实地改变容器里房子的数量。如果缩小
  (`n < size`)：疯狂调用 `~T()` 拆房子，直到剩余房子数量为 n。如果放大
  (`n > size`)：先看地皮够不够（不够就叫开发商 `reserve`），然后疯狂调用
  `push_back(val)` 盖新房子，直到房子总数达到 n。

## 完整代码

MyVector.h

```cpp
#pragma once
```


```cpp
namespace zyh
{
    template <class T>
    class vector
    {
    public:
        typedef T *iterator;
        typedef const T *const_iterator;
```


```cpp
    private:
        iterator _start;
        iterator _finish;
        iterator _end_of_storage; // 可用空间尾（仓库后墙）
```


```cpp
    public:
        vector() : _start(nullptr), _finish(nullptr), _end_of_storage(nullptr)
        {
        }
        // 带参构造函数：创建 n 个大小的容器，并用 value 初始化
        vector(size_t n, const T &value = T())
        {
            _start = (T *)::operator new(n * sizeof(T));
```


```cpp
            for (size_t i = 0; i < n; ++i)
            {
                new (_start + i) T(value);
            }
```


```cpp
            _finish = _start + n;
            _end_of_storage = _start + n;
        }
```


```cpp
        // 析构函数
        ~vector()
        {
            if (_start != nullptr)
            {
                for (iterator it = _start; it != _finish; ++it)
                {
                    it->~T();
                }
```


```cpp
                ::operator delete(_start);
                _start = _finish = _end_of_storage = nullptr;
            }
        }
```


```cpp
        // 拷贝构造函数：深拷贝
        vector(const vector<T> &v)
        {
            _start = (T *)::operator new(v.capacity() * sizeof(T));
```


```cpp
            for (size_t i = 0; i < v.size(); ++i)
            {
                new (_start + i) T(v._start[i]);
            }
```


```cpp
            _finish = _start + v.size();
            _end_of_storage = _start + v.capacity();
        }
```


```cpp
        // // 拷贝赋值运算符重载
        // vector<T> &operator=(const vector<T> &v)
        // {
        //     if (this != &v)
        //     {
        //         if (_start != nullptr)
        //         {
        //             for (iterator it = _start; it != _finish; ++it)
        //             {
        //                 it->~T();
        //             }
```


```cpp
        //             ::operator delete(_start);
        //         }
```


```cpp
        //         // 另起炉灶：根据别人的容量申请新地皮
        //         _start = (T *)::operator new(v.capacity() * sizeof(T));
```


```cpp
        //         for (size_t i = 0; i < v.size(); ++i)
        //         {
        //             new (_start + i) T(v._start[i]);
        //         }
```


```cpp
        //         _finish = _start + v.size();
        //         _end_of_storage = _start + v.capacity();
        //     }
```


```cpp
        //     // 返回自己的引用，支持连续赋值 (比如 v3 = v2 = v1;)
        //     return *this;
        // }
```


```cpp
        // 【现代级神仙写法】利用编译器和现有的拷贝构造函数
        vector<T> &operator=(vector<T> v) // 注意：这里故意【不用引用】，而是【传值】！！！
        {
            // 利用 swap 交换两者的三个门牌号指针
            swap(_start, v._start);
            swap(_finish, v._finish);
            swap(_end_of_storage, v._end_of_storage);
```


```cpp
            return *this;
        }
```


```cpp
        // 下标访问
        T &operator[](size_t i)
        {
            return _start[i];
        }
```


```cpp
        const T &operator[](size_t i) const
        {
            return _start[i];
        }
```


```cpp
        size_t size() const
        {
            return _finish - _start;
        }
```


```cpp
        size_t capacity() const
        {
            return _end_of_storage - _start;
        }
```


```cpp
        iterator begin()
        {
            return _start;
        }
```


```cpp
        iterator end()
        {
            return _finish;
        }
```


```cpp
        // 请求开辟新容量
        void reserve(size_t n)
        {
            if (n > capacity())
            {
                // 【工业级规范】只申请裸内存，不调用任何 T 的构造函数！
                T *tmp = (T *)::operator new(n * sizeof(T));
                // 提前记下旧仓库里有多少件快递（极其关键！因为等下 _start 换成新仓库后，就没法用旧距离算了）
                size_t old_size = size();
```


```cpp
                if (_start != nullptr)
                {
                    for (size_t i = 0; i < old_size; ++i)
                    {
                        tmp[i] = _start[i];
                        // 【工业级规范】旧仓库的快递搬走了，必须手动调用析构函数把它销毁
                        _start[i].~T();
                    }
```


```cpp
                    // 【工业级规范】因为是裸内存，退租时必须用 ::operator delete
                    ::operator delete(_start);
                }
```


```cpp
                _start = tmp;
                _finish = _start + old_size;
                _end_of_storage = _start + n;
            }
        }
```


```cpp
        // 尾部插入新元素
        void push_back(const T &x)
        {
            // 检查仓库是不是满了 (当前结尾已经顶到了可用空间尾)
            if (_finish == _end_of_storage)
            {
                size_t new_capacity = capacity() == 0 ? 4 : capacity() * 2;
                reserve(new_capacity);
            }
```


```cpp
            new (_finish) T(x); // 含义：就在 _finish 这个地址上，调用 T 的拷贝构造函数，原地生出一个新对象
            ++_finish;
        }
```


```cpp
        // 任意位置插入
        iterator insert(iterator pos, const T &x)
        {
            if (_finish == _end_of_storage)
            {
                // 搬家前，先算好 pos 距离 _start 的偏移量
                size_t offset = pos - _start;
```


```cpp
                size_t new_capacity = capacity() == 0 ? 4 : capacity() * 2;
                reserve(new_capacity);
```


```cpp
                // 搬家后，刷新 pos，让它指向新地皮上的正确位置！
                pos = _start + offset;
            }
```


```cpp
            // 挪动数据：把 pos 及它后面的元素全部往后挪一格
            iterator end_it = _finish - 1;
            while (end_it >= pos)
            {
                // 注意：这里最严谨的做法是判断是不是纯裸内存。
                // 为了简单说明逻辑，暂且先用 placement new 和赋值来处理
                if (end_it == _finish - 1)
                {
                    new (end_it + 1) T(*end_it);
                }
                else
                {
                    *(end_it + 1) = *end_it;
                }
                --end_it;
            }
            *pos = x;
            ++_finish;
```


```cpp
            return pos;
        }
```


```cpp
        // 任意位置删除
        iterator erase(iterator pos)
        {
            if (pos >= _start && pos < _finish)
            {
                // 挪动数据：从 pos 的下一个位置开始，全部往前挪一格
                iterator it = pos + 1;
                while (it != _finish)
                {
                    *(it - 1) = *it;
                    ++it;
                }
```


```cpp
                --_finish;
```


```cpp
                _finish->~T();
            }
            // C++ 官方标准规定：erase 必须返回被删除元素的【下一个元素】的新位置
            // 因为后面的元素全往前挪了，所以原本的 pos 指向的正好就是新的下一个元素
            return pos;
        }
```


```cpp
        // 尾删函数
        void pop_back()
        {
            if (_start != _finish)
            {
                erase(_finish - 1);
            }
        }
```


```cpp
        // 清空所有元素
        void clear()
        {
            while (_start != _finish)
            {
                --_finish;
                _finish->~T();
            }
        }
```


```cpp
        // 调整有效元素个数
        void resize(size_t n, const T &val = T())
        {
            if (n < size())
            {
                // 情况 1：缩小。多出来的元素我们要让它们“安乐死”
                while (size() > n)
                {
                    --_finish;
                    _finish->~T();
                }
            }
            else if (n > size())
            {
                // 情况 2：放大。先看地皮够不够
                if (n > capacity())
                {
                    reserve(n);
                }
```


```cpp
                while (size() < n)
                {
                    push_back(val);
                }
            }
        }
```


```cpp
        // 判断容器是否为空
        bool empty() const
        {
            return _start == _finish;
        }
    };
}
```


main.cpp

```cpp
#include <iostream>
#include <string>
#include "MyVector.h"
using namespace std;
```


```cpp
void test1()
{
    zyh::vector<int> v;
    // 阶段 1：初始状态
    cout << "--- 阶段 1 ---" << endl;
    cout << "Size: " << v.size() << ", Capacity: " << v.capacity() << endl;
```


```cpp
    // 阶段 2：连续插入 4 个元素
    v.push_back(10);
    v.push_back(20);
    v.push_back(30);
    v.push_back(40);
    cout << "--- 阶段 2 ---" << endl;
    cout << "Size: " << v.size() << ", Capacity: " << v.capacity() << endl;
```


```cpp
    // 阶段 3：插入第 5 个元素（触发扩容！）
    v.push_back(50);
    cout << "--- 阶段 3 ---" << endl;
    cout << "Size: " << v.size() << ", Capacity: " << v.capacity() << endl;
```


```cpp
    // 阶段 4：遍历打印
    cout << "--- 阶段 4 元素内容 ---" << endl;
    // 利用我们刚学过的：iterator 就是原生指针，直接用它来遍历！
    for (zyh::vector<int>::iterator it = v.begin(); it != v.end(); ++it)
    {
        cout << *it << " ";
    }
    cout << endl;
}
```


```cpp
void test2()
{
    cout << "=== 阶段 1：构建 v1 ===" << endl;
    // 创建一个装了 3 个 "Hello" 的容器
    zyh::vector<string> v1(3, "Hello");
```


```cpp
    cout << "=== 阶段 2：触发深拷贝构建 v2 ===" << endl;
    // 这里会精准调用你刚刚手写的【拷贝构造函数】！
    zyh::vector<string> v2(v1);
```


```cpp
    cout << "=== 阶段 3：验证物理隔离 ===" << endl;
    // 给 v2 单独尾插一个新字符串，看看会不会影响 v1
    v2.push_back("World");
```


```cpp
    // 打印 v1
    cout << "v1 (size: " << v1.size() << ") 元素: ";
    for (zyh::vector<string>::iterator it = v1.begin(); it != v1.end(); ++it)
    {
        cout << *it << " ";
    }
    cout << endl;
```


```cpp
    // 打印 v2
    cout << "v2 (size: " << v2.size() << ") 元素: ";
    for (zyh::vector<string>::iterator it = v2.begin(); it != v2.end(); ++it)
    {
        cout << *it << " ";
    }
    cout << endl;
```


```cpp
    cout << "=== 阶段 4：程序结束，见证奇迹 ===" << endl;
    // v2 会先调用析构函数，销毁自己的地皮和字符串。
    // v1 会后调用析构函数，销毁自己的地皮和字符串。
    // 因为你写了深拷贝，这里绝对不会出现 Double Free（段错误）！程序将完美退出。
}
```


```cpp
void test3()
{
    cout << "=== 阶段 1：准备测试数据 ===" << endl;
    // v1 是我们要被抄袭的“原件”
    zyh::vector<string> v1(2, "Apple");
```


```cpp
    // v3 是一个“老油条”，里面已经占用了地皮，存了 3 个 "Banana"
    zyh::vector<string> v3(3, "Banana");
```


```cpp
    cout << "=== 阶段 2：触发赋值运算符 ===" << endl;
    // 关键时刻！这里会触发 operator=
    // v3 必须先销毁自己的 3 个 "Banana"，退掉旧地皮，再去深拷贝 v1 的 2 个 "Apple"
    v3 = v1;
```


```cpp
    cout << "=== 阶段 3：测试连续赋值 (Chain Assignment) ===" << endl;
    zyh::vector<string> v4;
    // 这里先执行 v3 = v1，然后返回 v3 的引用，再执行 v4 = v3
    v4 = v3 = v1;
```


```cpp
    cout << "=== 阶段 4：验证物理隔离 ===" << endl;
    // 我们往 v3 里加点新东西，看看 v1 和 v4 会不会跟着变
    v3.push_back("Orange");
```


```cpp
    // --- 打印结果 ---
    cout << "v1 (size: " << v1.size() << ") 元素: ";
    for (zyh::vector<string>::iterator it = v1.begin(); it != v1.end(); ++it)
    {
        cout << *it << " ";
    }
    cout << endl;
```


```cpp
    cout << "v3 (size: " << v3.size() << ") 元素: ";
    for (zyh::vector<string>::iterator it = v3.begin(); it != v3.end(); ++it)
    {
        cout << *it << " ";
    }
    cout << endl;
```


```cpp
    cout << "v4 (size: " << v4.size() << ") 元素: ";
    for (zyh::vector<string>::iterator it = v4.begin(); it != v4.end(); ++it)
    {
        cout << *it << " ";
    }
    cout << endl;
```


```cpp
    cout << "=== 阶段 5：程序结束，见证奇迹 ===" << endl;
    // 程序退出时，v4, v3, v1 将依次调用析构函数。
    // 如果没有段错误（Core Dump），说明我们不仅没有 Double Free，也没有内存泄漏！
}
```


```cpp
void test4()
{
    cout << "=== 阶段 1：测试 push_back 和 operator[] ===" << endl;
    zyh::vector<string> v;
```


```cpp
    // 连续放入 4 个元素，假设此时容量刚好达到了 4
    v.push_back("A");
    v.push_back("B");
    v.push_back("C");
    v.push_back("D");
```


```cpp
    cout << "原始元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        std::cout << v[i] << " "; // 测试 operator[]
    }
    cout << endl
         << endl;
```


```cpp
    cout << "=== 阶段 2：触发致命危机 (扩容 + 插入) ===" << endl;
    // 此时 v 的 size 是 4，capacity 也是 4，仓库已满！
    // 我们获取一个指向第二个元素 "B" 的迭代器（此时它指向旧地皮）
    zyh::vector<string>::iterator it = v.begin() + 1;
```


```cpp
    cout << "准备在 'B' 的位置前面插入一个 'X'..." << endl;
    // 关键时刻！
    // insert 内部发现满载，会调用 reserve 申请新地皮并销毁旧地皮。
    // 如果没有我们的 offset 修复逻辑，这里继续使用 it 就会直接发生段错误崩溃！
    v.insert(it, "X");
    cout << "插入成功！没有发生崩溃！\n\n";
```


```cpp
    cout << "=== 阶段 3：验证挪动结果 ===" << endl;
    cout << "插入后 zyh::vector 大小: " << v.size() << "，容量: " << v.capacity() << endl;
    cout << "最终元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        cout << v[i] << " ";
    }
    cout << "\n";
}
```


```cpp
void test5()
{
    zyh::vector<string> v;
    v.push_back("Apple");
    v.push_back("Banana");
    v.push_back("Cherry");
    v.push_back("Date");
```


```cpp
    cout << "=== 阶段 1：原始数据 ===" << endl;
    cout << "v (size: " << v.size() << ") 元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        cout << v[i] << " ";
    }
    cout << "\n\n";
```


```cpp
    cout << "=== 阶段 2：测试 erase (删除 Banana) ===" << endl;
    // 获取指向 "Banana" 的迭代器 (begin + 1)
    zyh::vector<string>::iterator it = v.begin() + 1;
```


```cpp
    // 删除它，并接收返回的新迭代器
    it = v.erase(it);
```


```cpp
    cout << "删除后 v (size: " << v.size() << ") 元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        cout << v[i] << " ";
    }
    cout << "\n";
    // 验证返回值是不是指向了原本 Banana 后面的 Cherry
    cout << "erase 返回的迭代器现在指向: " << *it << "\n\n";
```


```cpp
    cout << "=== 阶段 3：测试 pop_back 和 clear ===" << endl;
    v.pop_back(); // 删掉 Date
    cout << "pop_back 后元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        cout << v[i] << " ";
    }
    cout << "\n";
```


```cpp
    v.clear(); // 全部清空
    cout << "clear 后，size: " << v.size() << "，capacity: " << v.capacity() << endl;
}
```


```cpp
void test6()
{
    zyh::vector<string> v;
```


```cpp
    cout << "=== 阶段 1：测试 empty ===" << endl;
    if (v.empty())
    {
        cout << "刚初始化的 zyh::vector 是空的！(验证成功)\n\n";
    }
```


```cpp
    cout << "=== 阶段 2：填入初始数据 ===" << endl;
    v.push_back("Apple");
    v.push_back("Banana");
    cout << "当前 size: " << v.size() << "，capacity: " << v.capacity() << endl;
    cout << "元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        cout << v[i] << " ";
    }
    cout << "\n\n";
```


```cpp
    cout << "=== 阶段 3：测试 resize 放大 (带默认参数) ===" << endl;
    // 原本只有 2 个元素，要求扩到 5 个。会补充 3 个 "Hello"
    v.resize(5, "Hello");
    cout << "resize(5, 'Hello') 之后 size: " << v.size() << "，capacity: " << v.capacity() << endl;
    cout << "元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        cout << v[i] << " ";
    }
    cout << "\n\n";
```


```cpp
    cout << "=== 阶段 4：测试 resize 缩小 ===" << endl;
    // 从 5 个缩减到 1 个。尾部的 4 个元素必须被执行析构函数（安乐死）
    v.resize(1);
    cout << "resize(1) 之后 size: " << v.size() << "，capacity: " << v.capacity() << endl;
    cout << "元素: ";
    for (size_t i = 0; i < v.size(); ++i)
    {
        cout << v[i] << " ";
    }
    cout << "\n\n";
```


```cpp
    cout << "=== 阶段 5：清理门户 ===" << endl;
    v.clear();
    cout << "clear() 之后，empty() 返回: " << (v.empty() ? "true" : "false") << endl;
    cout << "此时 capacity (地皮) 依然是: " << v.capacity() << endl;
}
```


```cpp
int main()
{
    // test1();
    // test2();
    // test3();
    // test4();
    // test5();
    test6();
    return 0;
}
```

