---
title: "STL-list类的模拟实现"
date: "2026-08-09T23:03:14+08:00"
draft: false
categories: ["C++"]
tags: ["STL", "list", "模拟实现"]
---
## 1.定义节点的结构

在真实的 SGI STL 中，`std::list` 的底层是一个**双向循环链表**。

我们可以把链表想象成一列火车：

- 每一个车厢就是一个**节点（Node）**。
- 每个车厢不仅要装载货物（**数据**），还要有两只手：一只手拉住前一个车厢（**前驱指针
  prev**），另一只手拉住后一个车厢（**后继指针 next**）。

<!-- -->

```cpp
    template <typename T>
    struct __list_node
    {
        __list_node<T> *_next;
        __list_node<T> *_prev;
        T _data;
    };
```


单向链表如果要做尾删，必须从头遍历寻找“倒数第二个节点”来把它的 `_next`
置空，这就导致时间复杂度从 O(1) 暴跌到了 O(N)。所以真实的 STL
为了保证所有尾部操作都是绝对的 O(1)，必须采用双向循环带头（dummy
node）的结构。

## 2.链表迭代器结构体

节点有了，接下来我们要面临一个棘手的问题：**如何遍历这些节点？**

链表的节点在内存中是**随机散落**的。如果你对一个指向节点的原生指针做
`p++`，它只会跳到一段毫无意义的内存垃圾上。

接下来的迭代器就像是一个“伪装服”**。我们写一个类，把刚才的节点指针
`__list_node*`
偷偷包裹在里面。然后我们通过**运算符重载（Overload），强行改变
`++`、`--` 和 `*`
的行为。让使用者觉得：“这玩意用起来跟普通指针一模一样！”但其实在底层，每次你调用
`++`，我们都在默默地帮你执行 `_node = _node->_next;`。

**这就是 STL 的核心哲学：对使用者屏蔽底层差异，提供统一的指针式接口。**

```cpp
    // 链表迭代器结构体
    template <typename T>
    struct __list_iterator
    {
        // 别名定义，简化代码的书写
        typedef __list_node<T>* link_type;
        typedef __list_iterator<T> self;
        
        link_type _node; // 这里面藏着真正的节点指针
```


```cpp
        // 构造函数
        __list_iterator(link_type x = nullptr) : _node(x) {}
```


```cpp
        // 解引用操作符：当用户写 *it 时，应该暴露出真正的用户数据（注意返回引用，因为用户可能会修改它，例如 *it = 10;）
        T& operator*() const
        {
            return _node->_data;
        }
```


```cpp
        // 前置 ++ 操作符：当用户写 ++it 时，底层节点指针该怎么顺藤摸瓜？
        // 注意：前置++通常返回自身的引用（self&），这样才能支持 ++(++it) 这种连续操作
        self& operator++()
        {
            // 让底层指针走到下一个节点
            _node = _node->_next; 
            return *this;
        }
    };
```


## 3.list容器本体

在工业级 STL（如 SGI STL）中，一个空的 `list`
并不是什么都没有，它在出生时就会自带一个“假车头”，专业术语叫
**“哨兵位节点（Sentinel Node）”** 或 **“头节点（Dummy Node）”**。

为什么一个元素都没有，却非要放一个“假车头”呢？
如果没有这个假车头，当链表为空时，头指针就是
`nullptr`。这时候如果我们要插入第一个元素，就必须写一堆
`if (head == nullptr)` 的特判逻辑。
但如果放了一个假车头，无论链表怎么插入、删除，它永远都在那里。**当链表为空时，这个假车头的
`_next` 和 `_prev` 都指向它自己，形成一个闭环。**
这样就免去了大量的非空判断，代码运行效率极高。

```cpp
    // 真正的 list 容器类
    template <typename T>
    class list
    {
    public:
        // 把我们刚才写的复杂迭代器，对外正式包装成公开的艺名 iterator
        typedef __list_iterator<T> iterator;
        typedef __list_node<T>* link_type;
```


```cpp
    private:
        link_type _head; // 这就是那个“假车头”（哨兵位节点指针）
```


```cpp
    public:
        // 构造函数：创造一辆空火车
        list()
        {
            // 先买一个空车厢作为车头
            _head = new __list_node<T>();
            
            // 空链表状态下，车头的 _next 和 _prev 让它们指向车头自己（_head），形成一个环！
            _head->_next = _head;
            _head->_prev = _head;
        }
    };
```


## 4.尾插函数

由于这是个循环链表，**无论火车有多长，当前最后一节车厢，永远是车头的“前驱”（即
`_head->_prev`）**！我们要把 `new_node` 牢牢地卡在
**原尾节点（`tail`）** 和 **车头（`_head`）** 的中间。这需要调整 **4
根指针的指向**：

```cpp
        // 往链表尾部插入元素
        void push_back(const T& x)
        {
            // 制造一节新节点，把数据装进去
            link_type new_node = new __list_node<T>();
            new_node->_data = x;
```


```cpp
            // 顺藤摸瓜，找到当前链表的尾节点
            link_type tail = _head->_prev;
```


```cpp
            // 开始调整4根指针的指向
            new_node->_next = _head;
            new_node->_prev = tail;
            
            tail->_next     = new_node;
            _head->_prev    = new_node;
        }
```


要想在 `main.cpp` 里用 `for`
循环把火车里的货物打印出来，需要加入三个小零件，然后开始测试一下我们的尾插函数

1.  **`begin()` 函数**：返回指向**第一个真实货物车厢**的遥控器。
2.  **`end()` 函数**：返回指向末尾终点站（假车头）的遥控器。
3.  **`operator!=`
```cpp
运算符**：让两个迭代器能对比，看看是不是走到终点站了。（这个是加入到迭代器的结构体里面）
```


<!-- -->

```cpp
        // 判断两个迭代器是否指向的节点不一样
        bool operator!=(const self &__x) const
        {
            return _node != __x._node;
        }
```


```cpp
        // 返回指向第一个真实货物的迭代器
        iterator begin()
        {
            return iterator(_head->_next);
        }
```


```cpp
        // 返回指向终点（哨兵节点）的迭代器
        iterator end()
        {
            return iterator(_head);
        }
```


测试代码

```cpp
void test1()
{
    zyh::list<int> ls;
```


```cpp
    ls.push_back(1);
    ls.push_back(2);
    ls.push_back(3);
```


```cpp
    zyh::list<int>::iterator it = ls.begin();
```


```cpp
    cout << "zyh::list里面的数据是：";
```


```cpp
    while (it != ls.end())
    {
        cout << *it << " ";
        ++it;
    }
```


```cpp
    cout << endl;
}
```


## 5.插入函数

通用的 `insert` 函数要求我们传入两个参数：

1.  `iterator position`：一个遥控器，代表你想把新车厢插在**哪个位置的面前**。
2.  `const T& x`：要装载的货物。

假设用户给的遥控器 `position` 此时正指向车厢 **`curr`**（通过
`position._node` 拿到）。 我们要把新车厢 **`new_node`** 完美地塞到
`curr` 的**前面**。

既然是双向链表，那 `curr` 原本的前驱节点（左边的车厢），我们可以通过
`curr->_prev` 轻松摸到，我们叫它 **`prev_node`**。

现在的目标，就是把 `new_node` 牢牢地卡在 `prev_node` 和 `curr`
的正中间。依旧是**绑定 4 根绳子**：

1.  `new_node` 的右侧手（`_next`）拉住谁？拉住后面的 `curr`。
2.  `new_node` 的左侧手（`_prev`）拉住谁？拉住前面的 `prev_node`。
3.  前面车厢 `prev_node` 的右侧手（`_next`）改拉谁？改拉 `new_node`。
4.  后面车厢 `curr` 的左侧手（`_prev`）改拉谁？改拉 `new_node`。

<!-- -->

```cpp
        // 在指定的位置前面插入一个节点
        iterator insert(iterator pos, const T &x)
        {
            link_type curr = pos._node;
```


```cpp
            link_type prev = curr->_prev;
```


```cpp
            link_type new_node = new __list_node<T>();
            new_node->_data = x;
```


```cpp
            new_node->_next = curr;
            new_node->_prev = prev;
```


```cpp
            prev->_next = new_node;
            curr->_prev = new_node;
```


```cpp
            // 回指向新节点的迭代器
            return iterator(new_node);
        }
```


当实现完insert函数以后，我们之前的push_back函数便可以通过复用insert函数，也就不再需要长篇大论了，同时我们还可以顺手把push_front也给写了

```cpp
        // 尾插：在终点站（end()）前面插一个车厢
        void push_back(const T& x)
        {
            insert(end(), x);
        }
```


```cpp
        // 头插：在第一节真实车厢（begin()）前面插一个车厢
        void push_front(const T& x)
        {
            insert(begin(), x);
        }
```


## 6.任意位置删除函数

假设用户给了你一个遥控器 `position`，指向了要被销毁的倒霉车厢
**`curr`**。

拆车厢非常简单，我们只需要让它前后的车厢“直接牵手”，跨过它就行了。

1.  找到前面的车厢：`prev_node = curr->_prev;`
2.  找到后面的车厢：`next_node = curr->_next;`
3.  **两根绳子重新绑**：前面车厢的右侧手（`_next`）直接拉住后面的车厢。后面车厢的左侧手（`_prev`）直接拉住前面的车厢。
4.  **销毁证据**：用 `delete` 关键字把 `curr` 这节车厢砸烂（释放内存）。

**STL 铁律**：`erase` 函数执行完毕后，原本的遥控器 `position`
就失效了（因为车厢被砸了）。所以它必须**返回一个指向被删车厢的下一节车厢的新遥控器**。

```cpp
        // 任意位置删除，并返回下一个节点的迭代器
        iterator erase(iterator pos)
        {
            link_type curr = pos._node;
```


```cpp
            link_type prev = curr->_prev;
            link_type next = curr->_next;
```


```cpp
            prev->_next = next;
            next->_prev = prev;
```


```cpp
            delete curr;
```


```cpp
            return iterator(next);
        }
```


接下来测试一下代码

```cpp
// 封装一个打印函数，专门用来测试我们的迭代器
void print_list(zyh::list<int> &l)
{
    cout << "当前火车里的货物: ";
    zyh::list<int>::iterator it = l.begin();
    while (it != l.end())
    {
        cout << *it << " "; // 解引用拿数据
        ++it;               // 迭代器往前走
    }
    cout << endl;
}
```


```cpp
void test2()
{
    zyh::list<int> ls;
```


```cpp
    // 2. 测试尾插 (push_back)
    cout << "\n[测试 1] 尾部插入 10, 20, 30:" << endl;
    ls.push_back(10);
    ls.push_back(20);
    ls.push_back(30);
    print_list(ls);
```


```cpp
    // 3. 测试头插 (push_front)
    cout << "\n[测试 2] 头部插入 5:" << endl;
    ls.push_front(5);
    print_list(ls);
```


```cpp
    // 4. 测试任意位置插入 (insert)
    cout << "\n[测试 3] 在 20 前面插入 15:" << endl;
    zyh::list<int>::iterator it = ls.begin();
    ++it; // 此时它指向 10
    ++it; // 此时它指向 20
    ls.insert(it, 15);
    print_list(ls);
```


```cpp
    // 5. 测试任意位置删除 (erase)
    cout << "\n[测试 4] 删除 10 这个元素:" << endl;
    zyh::list<int>::iterator erase_it = ls.begin();
    ++erase_it; // 让遥控器指向 10
    ls.erase(erase_it);
    print_list(ls);
```


```cpp
    cout << "\n================ 测试圆满结束 ================\n";
}
```


## 7.资源清理工作

为了解决这个问题，我们需要写两个极其重要的清理函数：

1.  **`clear()`
```cpp
函数**：清空火车里的所有货物（销毁所有真实车厢），但**保留假车头**，让火车回到刚出生时的“空车状态”。
```

2.  **`~list()`
```cpp
析构函数**：火车彻底报废，**连假车头一起砸掉**，一寸内存都不留。
```


<!-- -->

```cpp
        // 清空所有真实节点（保留哨兵节点）
        void clear()
        {
            iterator it = begin();
```


```cpp
            while (it != end())
            {
                // erase 会把当前车厢砸掉，并把下一个车厢的遥控器交给我们，我们直接接住！
                it = erase(it);
            }
        }
```


```cpp
        // 析构函数：火车寿命终结，彻底销毁
        ~list()
        {
            clear();
```


```cpp
            delete _head;
            _head = nullptr;
        }
```


## 8.拷贝构造函数

**恐怖的“浅拷贝”炸弹**

假设现在有另外一个程序员，在 `main.cpp` 里写了这样两行代码：

**zyh::list<int> list_A;  
list_A.push_back(10);**

**// 他想用 list_A 完美复制出一个 list_B  
zyh::list<int> list_B = list_A;**

因为我们没有手写“拷贝构造函数”，C++
编译器会自作主张地进行**浅拷贝（Shallow Copy）**。 它会把 `list_A` 的
`_head` 指针，原封不动地复制给 `list_B` 的 `_head`。
结果就是：**`list_A` 和 `list_B` 共享了同一个假车头和同一批车厢！**

- **平时会错乱**：往 B 里加车厢，A 也会跟着变。
- **销毁时必崩（Double Free）**：当程序结束时，A
  调用了我们刚刚写的析构函数 `~list()`，把车厢全砸了；紧接着 B
  也调用析构函数，又对着同一片已经被砸烂的废墟砸第二遍——操作系统会直接抛出段错误（Segmentation
  Fault）强制终止程序。

因此，我们必须手写一个拷贝构造函数，不要共享指针！B
必须自己造一个全新的车头，然后顺着 A
的车厢，一节一节地把里面的货物“复刻”过来。

```cpp
        // 拷贝构造函数
        list(list<T> &other)
        {
            _head = new __list_node<T>();
            _head->_next = _head;
            _head->_prev = _head;
```


```cpp
            // 拿出迭代器，从头到尾遍历另一个链表
            iterator it = other.begin();
            while (it != other.end())
            {
                push_back(*it);
                ++it;
            }
        }
```


## 9.赋值运算符重载

完成了拷贝构造函数，C++
的“三法则”就只剩下最后一块拼图了：**赋值运算符重载（`operator=`）**。也就是处理这种已经出生的两辆火车相互覆盖的情况：

**zyh::list<int> list_A;  
zyh::list<int> list_B;  
// ... 各自加了一些数据 ...**

**list_A = list_B; // 赋值操作**

传统的写法非常痛苦：你要先释放 `list_A` 原有的内存，再把 `list_B`
的东西拷过来，还要小心处理“自己赋值给自己（`list_A = list_A`）”的奇葩情况。

接下来我们要用到现代 C++ 黑魔法：Copy-and-Swap Idiom

先写一个swap函数，把两辆火车的**“假车头指针（`_head`）”**对调一下，整列火车就瞬间互换了！

```cpp
        void swap(list<T> &other)
        {
            std::swap(_head, other._head);
        }
```


接着就可以用短短几行实现赋值运算符重载

```cpp
        // 赋值运算符重载（现代写法：Copy-and-Swap（拷贝并交换））
        list<T> &operator=(list<T> other) // 这里没有用 &，故意用“值传递”！
        {
            swap(other);
```


```cpp
            return *this;
        }
```


**大白话拆解这个黑魔法**：

1.  **自动深拷贝**：因为参数是 `list<T> other`（值传递），用户在调用
```cpp
`list_A = list_B`
时，编译器会自动调用我们刚刚写好的**拷贝构造函数**，在内存里偷偷帮我们造出一辆一模一样的临时火车，叫
`other`。
```

2.  **偷梁换柱**：在函数内部，我们调用 `swap(other)`，把 `list_A`
```cpp
当前的老假车头，和临时火车 `other` 的新假车头**直接对调**！
```

3.  **借刀杀人（完美销毁）**：此时，`list_A`
```cpp
已经拥有了全新的货运车厢。而临时火车 `other` 肚子里装的，正是
`list_A` 替换下来的老旧垃圾车厢。当这个函数结束的大括号 `}`
关闭时，临时变量 `other` 生命周期结束，会自动触发它自己的**析构函数
`~list()`**，顺手把 `list_A` 换下来的老垃圾彻底销毁，不留一丝痕迹！
```


它甚至天然免疫“自己赋值给自己”，因为值传递出来的临时对象绝对是一个独立的个体。

接下来我们测试一下代码

```cpp
// 稍微升级一下打印函数，带上链表的名字方便区分
void print_list(zyh::list<int> &l, const string &name)
{
    cout << name << " 的货物: ";
    zyh::list<int>::iterator it = l.begin();
    while (it != l.end())
    {
        cout << *it << " ";
        ++it;
    }
    cout << endl;
}
```


```cpp
void test3()
{
    cout << "========== 开启深拷贝与赋值测试 ==========\n";
```


```cpp
    // 1. 造一列 A 号链表
    zyh::list<int> list_A;
    list_A.push_back(10);
    list_A.push_back(20);
    list_A.push_back(30);
    print_list(list_A, "list_A");
```


```cpp
    // 2. 测试拷贝构造：照着 A 造一列 B
    cout << "\n[测试 1] 拷贝构造测试 (list_B 照着 list_A 抄):" << endl;
    zyh::list<int> list_B(list_A); // 触发拷贝构造
    print_list(list_B, "list_B");
```


```cpp
    // 验证深拷贝：修改 B 会不会影响 A？
    cout << "\n[验证深拷贝] 往 list_B 尾部加 99，往 list_A 头部加 88:" << endl;
    list_B.push_back(99);
    list_A.push_front(88);
    print_list(list_A, "list_A (预期只有88在头)");
    print_list(list_B, "list_B (预期只有99在尾)");
```


```cpp
    // 3. 测试赋值运算符重载
    cout << "\n[测试 2] 赋值运算符测试 (造一个自带垃圾数据的 list_C):" << endl;
    zyh::list<int> list_C;
    list_C.push_back(666);
    list_C.push_back(777);
    print_list(list_C, "list_C (赋值前)");
```


```cpp
    cout << "\n[执行覆盖] list_C = list_A;" << endl;
    list_C = list_A; // 触发 Copy-and-Swap 黑魔法！
    print_list(list_C, "list_C (赋值后)");
```


```cpp
    cout << "\n================ 测试圆满结束 ================\n";
```


```cpp
    // 当 return 0 执行时：
    // list_C、list_B、list_A 会依次调用我们写的 ~zyh::list() 析构函数。
    // 如果没有引发 Segmentation Fault (段错误)，说明我们的深拷贝大获全胜！
}
```


## 10.进阶内容：常量迭代器（终极迭代器融合形态）

在讲解具体代码之前，我们要先搞清楚一个问题：**为什么我们需要常量迭代器（`const_iterator`）？**

想象一下，如果有一个C++ 程序员写了这样一个打印函数：

**void print_list(const zyh::list<int>& l) // 👈 注意这里加了 const  
{  
    // ...  
}**

因为 `l` 被 `const` 锁死了，它变成了一个“只读火车”。如果你还用普通的
`iterator` 去遍历它，普通的 `iterator` 里的 `T& operator*()`
是会返回一个**可修改的引用**的。这就打破了 `const`
的保护罩，编译器会立刻翻脸报错。

为了保护这辆“只读火车”，我们需要一种**只能看、不能改**的遥控器，也就是
`const_iterator`，它的 `operator*()` 必须返回 `const T&`。

而接下来会有两种做法

第一种做法（不推荐）：代码复制机

把我们刚才写的 `__list_iterator` 结构体从头到尾复制一遍，改名叫
`__list_const_iterator`，然后把里面所有的 `T&` 改成 `const T&`。
*缺点*：产生了大量极其冗余的重复代码。如果以后要改迭代器的某个逻辑，必须同时改两份！

第二种做法（架构师做法）：STL 顶级黑魔法

既然普通的和常量的遥控器，唯一的区别就是**拿货物的返回值不同**，那我们就把“返回值类型”也当成一个模板参数传进去

### 10.1 更改一下\_\_list_iterator的结构

我们要把 `__list_iterator`
的模板参数从一个（`T`），扩充为三个（`T, Ref, Ptr`）：

- `T`：货物原本的类型（比如 `int`）。
- `Ref`：解引用（`*`）时的返回值类型。传入 `T&` 就是普通遥控器，传入
  `const T&` 就是只读遥控器。
- `Ptr`：顺便把 `->` 操作符的返回值（指针类型）也泛型化。这是你按 `->`
  键（箭头）时，想得到的类型。

为什么要把 `Ptr`（箭头 `->`
键）也加进去？

这其实是为了应对一种非常常见的情况：**如果你火车里装的不是简单的 `int`
整数，而是一个复杂的结构体（比如装的是 `Student` 学生实体）**

**struct Student                                                       
                                                                       
  {  
    string name;  
    int age;  
};**

当你的火车里装满了学生（`list<Student>`），你想通过遥控器打印学生的名字。按照
C++ 的习惯，我们通常会直接使用箭头按键 `->`：cout << it->name; 

当你在代码里写下 `it->` 时，编译器底层其实是在调用一个名为 `operator->`
的函数。

**C++ 规定，`operator->` 这个函数，必须返回一个内存地址（指针）。**

**编译器的特殊魔法：消失的“第二个箭头”**

在 C++ 中，`->` 操作符的重载有一条极其霸道且不符合常理的底层铁律：

C++ 规定：你手写的 `operator->`
函数，必须返回一个“原生的指针（内存地址）”。然后，编译器会自动、强行、在后面再补上一个箭头！

**我们来看真实的场景。当你写下 `it->age = 18;`
时，编译器在后台其实把你的代码偷偷重写成了这样：it->age 相当于
(it.operator->()) -> age**

发现了没有？！这里其实出现了**两个箭头**：

1.  **第一个箭头**（`it.operator->()`）：调用你写的函数。你写的函数负责返回货物的**内存地址**（也就是
```cpp
`T*` 指针）。
```

2.  **第二个箭头**（`->age`）：这是**编译器自动赠送的**。它拿着你刚刚返回的那个地址，顺藤摸瓜，去摘取里面的
```cpp
`age`。
```


<!-- -->

```cpp
    // 终极形态迭代器
    template <typename T, typename Ref, typename Ptr>
    struct __list_iterator
    {
        typedef __list_node<T> *link_type;
        typedef __list_iterator<T, Ref, Ptr> self;
```


```cpp
        link_type _node; // 迭代器的芯片，指向节点
```


```cpp
        __list_iterator() : _node(nullptr)
        {
        }
```


```cpp
        __list_iterator(link_type x) : _node(x)
        {
        }
```


```cpp
        // 普通迭代器的解引用：返回货物的引用
        Ref operator*()
        {
            return _node->_data;
        }
```


```cpp
        // 重载箭头操作符
        Ptr operator->()
        {
            return &(_node->_data);
        }
        // 当比如_data是一个自定义的结构体(例如Student)
        // 手写的 operator-> 函数，必须返回一个“原生的指针（内存地址）”
        // 然后，编译器会自动、强行、在后面再补上一个箭头
        // !!! 写下it->age相当于(it.operator->())->age
```


```cpp
        // 前置 ++ 操作符：返回迭代器自己的引用
        self &operator++()
        {
            _node = _node->_next;
            return *this;
        }
```


```cpp
        // 前置 -- 操作符
        self &operator--()
        {
            _node = _node->_prev;
            return *this;
        }
```


```cpp
        // 后置 ++ 操作符
        self operator++(int)
        {
            self tmp(*this);
            _node = _node->_next;
            return tmp;
        }
```


```cpp
        // 后置 -- 操作符
        self operator--(int)
        {
            self tmp(*this);
            _node = _node->_prev;
            return tmp;
        }
```


```cpp
        // 判断两个迭代器是否指向的节点不一样
        bool operator!=(const self &__x) const
        {
            return _node != __x._node;
        }
    };
```


### 10.2 在list内部定义两个迭代器

我们要在 `list` 容器内部，利用 `typedef` 向系统正式注册这两套遥控器。

同时，为了配合 `const` 版本的容器，我们要新增两个带有 `const` 修饰的
`begin()` 和 `end()` 函数。这叫**函数重载**：普通的 `list` 会调用普通的
`begin()`，被 `const` 锁死的 `list` 会自动调用 `const begin()`。

```cpp
    template <typename T>
    class list
    {
    public:
        // 核心魔法：利用不同的模板参数，瞬间裂变出两种遥控器！
        typedef __list_iterator<T, T&, T*>             iterator;
        typedef __list_iterator<T, const T&, const T*> const_iterator;
```


```cpp
        typedef __list_node<T>* link_type;
```


```cpp
    private:
        link_type _head; 
```


```cpp
    public:
        // ===== 原来的代码保持不变 =====
        list() { /* ... */ }
        ~list() { /* ... */ }
        list(list<T>& other) { /* ... */ }
        void swap(list<T>& other) { /* ... */ }
        list<T>& operator=(list<T> other) { /* ... */ }
        void push_back(const T& x) { /* ... */ }
        void push_front(const T& x) { /* ... */ }
        iterator insert(iterator pos, const T& x) { /* ... */ }
        iterator erase(iterator pos) { /* ... */ }
        void clear() { /* ... */ }
```


```cpp
        // ===== 迭代器核心接口 =====
        
        // 1. 普通版本：给普通的 list 用
        iterator begin() { return _head->_next; }
        iterator end()   { return _head; }
```


```cpp
        // 2. 只读版本：专门给 const list 用，返回 const_iterator
        const_iterator begin() const { return _head->_next; }
        const_iterator end() const   { return _head; }
    };
```


- 生成普通遥控器
- typedef \_\_list_iterator<T, T&, T\*> iterator;
- 相当于编译器帮你生成了一个：按 \* 返回 T& 的遥控器（T = T, Ref = T&,
  Ptr = T\*）
- 生成只读遥控器
- typedef \_\_list_iterator<T, const T&, const T\*> const_iterator;
- 相当于编译器帮你生成了一个：按 \* 返回 const T& 的遥控器（T = T, Ref =
  const T&, Ptr = const T\*）

进行测试

```cpp
// 造一个结构体，用来专门测试 -> 操作符
struct Student
{
    string _name;
    int _age;
```


```cpp
    // 构造函数，方便我们初始化装货
    Student(string name = "", int age = 0) : _name(name), _age(age)
    {
    }
};
```


```cpp
// 专门用来测试我们的 const_iterator
void print_const_list(const zyh::list<Student> &l)
{
    cout << "[常量函数内部] 正在使用 const_iterator 打印:\n";
```


```cpp
    // 必须用 const_iterator，否则编译器会直接报错！
    zyh::list<Student>::const_iterator it = l.begin();
    while (it != l.end())
    {
        // 🔥 见证奇迹的时刻！直接用 -> 访问结构体零件
        // 底层偷偷执行了: ( it.operator->() )->_name
        cout << "学生姓名: " << it->_name << ", 年龄: " << it->_age << endl;
```


```cpp
        // 🧪 恶作剧测试：如果解开下面这行的注释，编译器会立刻拦截，证明 const 锁生效了！
        // it->_age = 100;
```


```cpp
        ++it;
    }
}
```


```cpp
void test4()
{
    cout << "========== 开启 zyh::list 终极测试 ==========\n\n";
```


```cpp
    // 第一步：造一列普通火车，装入 3 个学生实体
    zyh::list<Student> school_bus;
    school_bus.push_back(Student("张三", 18));
    school_bus.push_back(Student("李四", 19));
    school_bus.push_back(Student("王五", 20));
```


```cpp
    // 第二步：使用普通迭代器（可读可改）
    cout << "[测试 1] 使用普通迭代器遍历，并将李四的年龄改成 99:\n";
    zyh::list<Student>::iterator it = school_bus.begin();
    while (it != school_bus.end())
    {
        if (it->_name == "李四")
        {
            it->_age = 99; // 普通迭代器，允许修改！
        }
        cout << "学生: " << it->_name << ", 修改后年龄: " << it->_age << endl;
        ++it;
    }
    cout << "\n--------------------------------------------------\n\n";
```


```cpp
    // 第三步：把火车送进只读打印函数（触发 const 机制）
    // 此时 school_bus 会被以 const& 的身份传进去
    print_const_list(school_bus);
```


```cpp
    cout << "\n========== 所有核心工程全部完美通过！ ==========\n";
}
```


## 11.添加一些常用的高频函数

- `empty()`：判断火车是不是空的。
- `size()`：目前我们没有记录火车的长度（可以在类里加一个 `size_t _size`
  变量，在 `insert` 时 `++_size`，`erase` 时 `--_size`）。
- `pop_back()` 和 `pop_front()`：尾删和头删

<!-- -->

```cpp
        // 辅助函数：判断链表是否为空
        bool empty() const
        {
            return _head->_next == _head;
        }
```


```cpp
        // 头删
        void pop_front()
        {
            erase(begin());
        }
```


```cpp
        // 尾删
        void pop_back()
        {
            // 要删最后一个，就是 end() 的前一个
            erase(--end());
        }
```


加上 `size()` 功能，为了让 `size()` 调用速度极快（O(1)），我们要给
`list` 类增加一个成员变量 `_size` 来实时记录长度。

1.  在 `list` 的 `private` 区域添加：`size_t _size;`
2.  在 `list()` 构造函数中初始化：`_size = 0;`
3.  在 `insert` 函数里：`++_size;`
4.  在 `erase` 函数里：`--_size;`
5.  添加 `size()` 函数

<!-- -->

```cpp
    size_t size() const
    {
        return _size;
    }
```


最终测试

```cpp
void test5()
{
    cout << "========== 开启 zyh::list 最终测试 ==========\n\n";
```


```cpp
    zyh::list<int> my_list;
```


```cpp
    // ---------------------------------------------------------
    // 测试 1：仪表盘测试 (empty 和 size)
    // ---------------------------------------------------------
    cout << "[测试 1]\n";
    cout << "是否为空? " << (my_list.empty() ? "是 (Empty)" : "否") << "\n";
    cout << "当前数量: " << my_list.size() << "\n\n";
```


```cpp
    // 装入 4 节车厢
    my_list.push_back(10);
    my_list.push_back(20);
    my_list.push_back(30);
    my_list.push_back(40);
```


```cpp
    cout << "[测试 2]\n";
    cout << "是否为空? " << (my_list.empty() ? "是" : "否 (Not Empty)") << "\n";
    cout << "当前数量: " << my_list.size() << "\n\n";
```


```cpp
    // ---------------------------------------------------------
    // 测试 3：遍历测试 (包含后置 ++ 和 前置后置 --)
    // ---------------------------------------------------------
    cout << "[测试 3] 使用后置 it++ 顺向遍历:\n";
    zyh::list<int>::iterator it = my_list.begin();
    while (it != my_list.end())
    {
        cout << *it << " ";
        it++; // 专门测试你的后置 ++
    }
    cout << "\n\n";
```


```cpp
    cout << "[测试 4] 使用倒车档逆向遍历 (从尾开到头):\n";
    zyh::list<int>::iterator rit = my_list.end();
    rit--; // 先测试后置 --，把迭代器从哨兵节点退到真实的最后一节车厢 (40)
```


```cpp
    // 得益于我们的双向循环结构，一直倒车直到再次撞到哨兵节点(end)为止
    while (rit != my_list.end())
    {
        cout << *rit << " ";
        --rit; // 专门测试你的前置 --
    }
    cout << "\n\n";
```


```cpp
    // ---------------------------------------------------------
    // 测试 5：弹射器测试 (pop_front 和 pop_back)
    // ---------------------------------------------------------
    cout << "[测试 5] 启动弹射器:\n";
```


```cpp
    my_list.pop_front(); // 干掉车头 (10)
    cout << "执行 pop_front() 后，头部被删，剩余大小: " << my_list.size() << "\n";
```


```cpp
    my_list.pop_back(); // 干掉车尾 (40)
    cout << "执行 pop_back() 后，尾部被删，剩余大小: " << my_list.size() << "\n";
```


```cpp
    cout << "现在的剩余车厢是: ";
    for (zyh::list<int>::iterator cur = my_list.begin(); cur != my_list.end(); ++cur)
    {
        cout << *cur << " "; // 应该只剩下 20 和 30
    }
    cout << "\n\n";
```


```cpp
    cout << "========== 恭喜！所有测试完美通过！ ==========\n";
}
```


## 12.完整版代码

MyList.h

```cpp
#ifndef MY_STL_LIST_HPP
#define MY_STL_LIST_HPP
```


```cpp
#include <cstddef>
#include <utility>
#include <cassert>
```


```cpp
namespace zyh
{
    template <typename T>
    struct __list_node
    {
        __list_node<T> *_next;
        __list_node<T> *_prev;
        T _data;
    };
```


```cpp
    // 终极形态迭代器
    template <typename T, typename Ref, typename Ptr>
    struct __list_iterator
    {
        typedef __list_node<T> *link_type;
        typedef __list_iterator<T, Ref, Ptr> self;
```


```cpp
        link_type _node; // 迭代器的芯片，指向节点
```


```cpp
        __list_iterator() : _node(nullptr)
        {
        }
```


```cpp
        __list_iterator(link_type x) : _node(x)
        {
        }
```


```cpp
        // 支持用普通 iterator 隐式构造 const_iterator
        __list_iterator(const __list_iterator<T, T &, T *> &it)
            : _node(it._node)
        {
        }
```


```cpp
        // 普通迭代器的解引用：返回货物的引用
        Ref operator*() const
        {
            return _node->_data;
        }
```


```cpp
        // 重载箭头操作符
        // 当比如_data是一个自定义的结构体(例如Student)
        // 手写的 operator-> 函数，必须返回一个“原生的指针（内存地址）”
        // 然后，编译器会自动、强行、在后面再补上一个箭头
        // !!! 写下it->age相当于(it.operator->())->age
        Ptr operator->() const
        {
            return &(_node->_data);
        }
```


```cpp
        // 前置 ++ 操作符：返回迭代器自己的引用
        self &operator++()
        {
            _node = _node->_next;
            return *this;
        }
```


```cpp
        // 前置 -- 操作符
        self &operator--()
        {
            _node = _node->_prev;
            return *this;
        }
```


```cpp
        // 后置 ++ 操作符
        self operator++(int)
        {
            self tmp(*this);
            _node = _node->_next;
            return tmp;
        }
```


```cpp
        // 后置 -- 操作符
        self operator--(int)
        {
            self tmp(*this);
            _node = _node->_prev;
            return tmp;
        }
```


```cpp
        // 判断两个迭代器是否指向的节点不一样
        bool operator!=(const self &__x) const
        {
            return _node != __x._node;
        }
```


```cpp
        // 判断两个迭代器是否指向的节点一样
        bool operator==(const self &__x) const
        {
            return _node == __x._node;
        }
    };
```


```cpp
    template <typename T>
    class list
    {
    public:
        typedef __list_iterator<T, T &, T *> iterator;
        typedef __list_iterator<T, const T &, const T *> const_iterator;
```


```cpp
        typedef __list_node<T> *link_type;
```


```cpp
    private:
        link_type _head; // 哨兵节点
        size_t _size;
```


```cpp
    public:
        // 构造函数
        list()
        {
            _size = 0;
            _head = new __list_node<T>();
```


```cpp
            _head->_next = _head;
            _head->_prev = _head;
        }
```


```cpp
        // 拷贝构造函数
        list(const list<T> &other) : _size(0)
        {
            _head = new __list_node<T>();
            _head->_next = _head;
            _head->_prev = _head;
```


```cpp
            // 拿出迭代器，从头到尾遍历另一个链表
            const_iterator it = other.begin();
            while (it != other.end())
            {
                push_back(*it);
                ++it;
            }
        }
```


```cpp
        void swap(list<T> &other)
        {
            std::swap(_head, other._head);
            std::swap(_size, other._size);
        }
```


```cpp
        // 赋值运算符重载（现代写法：Copy-and-Swap（拷贝并交换））
        list<T> &operator=(list<T> other) // 这里没有用 &，故意用“值传递”！
        {
            swap(other);
```


```cpp
            return *this;
        }
```


```cpp
        // // 尾插函数
        // void push_back(const T &x)
        // {
        //     link_type new_node = new __list_node<T>();
        //     new_node->_data = x;
```


```cpp
        //     link_type tail = _head->_prev;
```


```cpp
        //     new_node->_next = _head;
        //     new_node->_prev = tail;
```


```cpp
        //     tail->_next = new_node;
        //     _head->_prev = new_node;
        // }
```


```cpp
        // 返回指向第一个真实货物的迭代器
        iterator begin()
        {
            return iterator(_head->_next);
        }
```


```cpp
        // 只读版本
        const_iterator begin() const
        {
            return _head->_next;
        }
```


```cpp
        // 返回指向终点（哨兵节点）的迭代器
        iterator end()
        {
            return iterator(_head);
        }
```


```cpp
        // 只读版本
        const_iterator end() const
        {
            return _head;
        }
```


```cpp
        // 在指定的位置前面插入一个节点
        iterator insert(const_iterator pos, const T &x)
        {
            link_type curr = pos._node;
```


```cpp
            link_type prev = curr->_prev;
```


```cpp
            link_type new_node = new __list_node<T>();
            new_node->_data = x;
```


```cpp
            new_node->_next = curr;
            new_node->_prev = prev;
```


```cpp
            prev->_next = new_node;
            curr->_prev = new_node;
```


```cpp
            ++_size;
```


```cpp
            // 回指向新节点的迭代器
            return iterator(new_node);
        }
```


```cpp
        // 更改之前的尾插函数，复用insert函数
        void push_back(const T &x)
        {
            insert(end(), x);
        }
```


```cpp
        // 头插函数，复用insert
        void push_front(const T &x)
        {
            insert(begin(), x);
        }
```


```cpp
        // 任意位置删除，并返回下一个节点的迭代器
        iterator erase(const_iterator pos)
        {
            assert(!empty());
```


```cpp
            link_type curr = pos._node;
```


```cpp
            link_type prev = curr->_prev;
            link_type next = curr->_next;
```


```cpp
            prev->_next = next;
            next->_prev = prev;
```


```cpp
            delete curr;
```


```cpp
            --_size;
```


```cpp
            return iterator(next);
        }
```


```cpp
        // 清空所有真实节点（保留哨兵节点）
        void clear()
        {
            iterator it = begin();
```


```cpp
            while (it != end())
            {
                // erase 会把当前车厢砸掉，并把下一个车厢的遥控器交给我们，我们直接接住！
                it = erase(it);
            }
        }
```


```cpp
        // 辅助函数：判断链表是否为空
        bool empty() const
        {
            return _head->_next == _head;
        }
```


```cpp
        // 头删
        void pop_front()
        {
            assert(!empty());
```


```cpp
            erase(begin());
        }
```


```cpp
        // 尾删
        void pop_back()
        {
            assert(!empty());
```


```cpp
            // 要删最后一个，就是 end() 的前一个
            erase(--end());
        }
```


```cpp
        size_t size() const
        {
            return _size;
        }
```


```cpp
        ~list()
        {
            clear();
```


```cpp
            delete _head;
            _head = nullptr;
        }
    };
}
```


```cpp
#endif // MY_STL_LIST_HPP
```


main.cpp

```cpp
#include <iostream>
#include "MyList.h"
using namespace std;
```


```cpp
void test1()
{
    zyh::list<int> ls;
```


```cpp
    ls.push_back(1);
    ls.push_back(2);
    ls.push_back(3);
```


```cpp
    zyh::list<int>::iterator it = ls.begin();
```


```cpp
    cout << "zyh::list里面的数据是：";
```


```cpp
    while (it != ls.end())
    {
        cout << *it << " ";
        ++it;
    }
```


```cpp
    cout << endl;
}
```


```cpp
// // 封装一个打印函数，专门用来测试我们的迭代器
// void print_list(zyh::list<int> &l)
// {
//     cout << "当前火车里的货物: ";
//     zyh::list<int>::iterator it = l.begin();
//     while (it != l.end())
//     {
//         cout << *it << " "; // 解引用拿数据
//         ++it;               // 迭代器往前走
//     }
//     cout << endl;
// }
```


```cpp
// void test2()
// {
//     zyh::list<int> ls;
```


```cpp
//     // 2. 测试尾插 (push_back)
//     cout << "\n[测试 1] 尾部插入 10, 20, 30:" << endl;
//     ls.push_back(10);
//     ls.push_back(20);
//     ls.push_back(30);
//     print_list(ls);
```


```cpp
//     // 3. 测试头插 (push_front)
//     cout << "\n[测试 2] 头部插入 5:" << endl;
//     ls.push_front(5);
//     print_list(ls);
```


```cpp
//     // 4. 测试任意位置插入 (insert)
//     cout << "\n[测试 3] 在 20 前面插入 15:" << endl;
//     zyh::list<int>::iterator it = ls.begin();
//     ++it; // 此时它指向 10
//     ++it; // 此时它指向 20
//     ls.insert(it, 15);
//     print_list(ls);
```


```cpp
//     // 5. 测试任意位置删除 (erase)
//     cout << "\n[测试 4] 删除 10 这个元素:" << endl;
//     zyh::list<int>::iterator erase_it = ls.begin();
//     ++erase_it; // 让遥控器指向 10
//     ls.erase(erase_it);
//     print_list(ls);
```


```cpp
//     cout << "\n================ 测试圆满结束 ================\n";
// }
```


```cpp
// 稍微升级一下打印函数，带上链表的名字方便区分
void print_list(zyh::list<int> &l, const string &name)
{
    cout << name << " 的货物: ";
    zyh::list<int>::iterator it = l.begin();
    while (it != l.end())
    {
        cout << *it << " ";
        ++it;
    }
    cout << endl;
}
```


```cpp
void test3()
{
    cout << "========== 开启深拷贝与赋值测试 ==========\n";
```


```cpp
    // 1. 造一列 A 号链表
    zyh::list<int> list_A;
    list_A.push_back(10);
    list_A.push_back(20);
    list_A.push_back(30);
    print_list(list_A, "list_A");
```


```cpp
    // 2. 测试拷贝构造：照着 A 造一列 B
    cout << "\n[测试 1] 拷贝构造测试 (list_B 照着 list_A 抄):" << endl;
    zyh::list<int> list_B(list_A); // 触发拷贝构造
    print_list(list_B, "list_B");
```


```cpp
    // 验证深拷贝：修改 B 会不会影响 A？
    cout << "\n[验证深拷贝] 往 list_B 尾部加 99，往 list_A 头部加 88:" << endl;
    list_B.push_back(99);
    list_A.push_front(88);
    print_list(list_A, "list_A (预期只有88在头)");
    print_list(list_B, "list_B (预期只有99在尾)");
```


```cpp
    // 3. 测试赋值运算符重载
    cout << "\n[测试 2] 赋值运算符测试 (造一个自带垃圾数据的 list_C):" << endl;
    zyh::list<int> list_C;
    list_C.push_back(666);
    list_C.push_back(777);
    print_list(list_C, "list_C (赋值前)");
```


```cpp
    cout << "\n[执行覆盖] list_C = list_A;" << endl;
    list_C = list_A; // 触发 Copy-and-Swap 黑魔法！
    print_list(list_C, "list_C (赋值后)");
```


```cpp
    cout << "\n================ 测试圆满结束 ================\n";
```


```cpp
    // 当 return 0 执行时：
    // list_C、list_B、list_A 会依次调用我们写的 ~zyh::list() 析构函数。
    // 如果没有引发 Segmentation Fault (段错误)，说明我们的深拷贝大获全胜！
}
```


```cpp
// 造一个结构体，用来专门测试 -> 操作符
struct Student
{
    string _name;
    int _age;
```


```cpp
    // 构造函数，方便我们初始化装货
    Student(string name = "", int age = 0) : _name(name), _age(age)
    {
    }
};
```


```cpp
// 专门用来测试我们的 const_iterator
void print_const_list(const zyh::list<Student> &l)
{
    cout << "[常量函数内部] 正在使用 const_iterator 打印:\n";
```


```cpp
    // 必须用 const_iterator，否则编译器会直接报错！
    zyh::list<Student>::const_iterator it = l.begin();
    while (it != l.end())
    {
        // 见证奇迹的时刻！直接用 -> 访问结构体零件
        // 底层偷偷执行了: ( it.operator->() )->_name
        cout << "学生姓名: " << it->_name << ", 年龄: " << it->_age << endl;
```


```cpp
        // 恶作剧测试：如果解开下面这行的注释，编译器会立刻拦截，证明 const 锁生效了！
        // it->_age = 100;
```


```cpp
        ++it;
    }
}
```


```cpp
void test4()
{
    cout << "========== 开启 zyh::list 终极测试 ==========\n\n";
```


```cpp
    // 第一步：造一列普通火车，装入 3 个学生实体
    zyh::list<Student> school_bus;
    school_bus.push_back(Student("张三", 18));
    school_bus.push_back(Student("李四", 19));
    school_bus.push_back(Student("王五", 20));
```


```cpp
    // 第二步：使用普通迭代器（可读可改）
    cout << "[测试 1] 使用普通迭代器遍历，并将李四的年龄改成 99:\n";
    zyh::list<Student>::iterator it = school_bus.begin();
    while (it != school_bus.end())
    {
        if (it->_name == "李四")
        {
            it->_age = 99; // 普通迭代器，允许修改！
        }
        cout << "学生: " << it->_name << ", 修改后年龄: " << it->_age << endl;
        ++it;
    }
    cout << "\n--------------------------------------------------\n\n";
```


```cpp
    // 第三步：把火车送进只读打印函数（触发 const 机制）
    // 此时 school_bus 会被以 const& 的身份传进去
    print_const_list(school_bus);
```


```cpp
    cout << "\n========== 所有核心工程全部完美通过！ ==========\n";
}
```


```cpp
void test5()
{
    cout << "========== 开启 zyh::list 最终测试 ==========\n\n";
```


```cpp
    zyh::list<int> my_list;
```


```cpp
    // ---------------------------------------------------------
    // 测试 1：仪表盘测试 (empty 和 size)
    // ---------------------------------------------------------
    cout << "[测试 1]\n";
    cout << "是否为空? " << (my_list.empty() ? "是 (Empty)" : "否") << "\n";
    cout << "当前数量: " << my_list.size() << "\n\n";
```


```cpp
    // 装入 4 节车厢
    my_list.push_back(10);
    my_list.push_back(20);
    my_list.push_back(30);
    my_list.push_back(40);
```


```cpp
    cout << "[测试 2]\n";
    cout << "是否为空? " << (my_list.empty() ? "是" : "否 (Not Empty)") << "\n";
    cout << "当前数量: " << my_list.size() << "\n\n";
```


```cpp
    // ---------------------------------------------------------
    // 测试 3：遍历测试 (包含后置 ++ 和 前置后置 --)
    // ---------------------------------------------------------
    cout << "[测试 3] 使用后置 it++ 顺向遍历:\n";
    zyh::list<int>::iterator it = my_list.begin();
    while (it != my_list.end())
    {
        cout << *it << " ";
        it++; // 专门测试你的后置 ++
    }
    cout << "\n\n";
```


```cpp
    cout << "[测试 4] 使用倒车档逆向遍历 (从尾开到头):\n";
    zyh::list<int>::iterator rit = my_list.end();
    rit--; // 先测试后置 --，把迭代器从哨兵节点退到真实的最后一节车厢 (40)
```


```cpp
    // 得益于我们的双向循环结构，一直倒车直到再次撞到哨兵节点(end)为止
    while (rit != my_list.end())
    {
        cout << *rit << " ";
        --rit; // 专门测试你的前置 --
    }
    cout << "\n\n";
```


```cpp
    // ---------------------------------------------------------
    // 测试 5：弹射器测试 (pop_front 和 pop_back)
    // ---------------------------------------------------------
    cout << "[测试 5] 启动弹射器:\n";
```


```cpp
    my_list.pop_front(); // 干掉车头 (10)
    cout << "执行 pop_front() 后，头部被删，剩余大小: " << my_list.size() << "\n";
```


```cpp
    my_list.pop_back(); // 干掉车尾 (40)
    cout << "执行 pop_back() 后，尾部被删，剩余大小: " << my_list.size() << "\n";
```


```cpp
    cout << "现在的剩余车厢是: ";
    for (zyh::list<int>::iterator cur = my_list.begin(); cur != my_list.end(); ++cur)
    {
        cout << *cur << " "; // 应该只剩下 20 和 30
    }
    cout << "\n\n";
```


```cpp
    cout << "========== 恭喜！所有测试完美通过！ ==========\n";
}
```


```cpp
int main()
{
    // test1();
    // test2();
    // test3();
    // test4();
    test5();
    return 0;
}
```


## 13.补充说明const_iterator和const iterator的区别

**很多初学者容易把 `const iterator` 和
`const_iterator` 搞混：**

1.  **`const iterator it`**（等价于
```cpp
`int* const p`）：代表**迭代器变量本身是常量**。`it`
被锁死了，**不能执行 `++it`**。一旦编译器走到 `++it`
就会报错：无法修改 `const` 对象。
```


2.  **`const_iterator it`**（等价于
```cpp
`const int* p`）：代表**迭代器指向的内容是常量**。`it`
本身是自由的（可以 `++it`），但不能通过 `*it`
去修改链表里的节点数据。
```


在遍历链表时，迭代器自身必须能够向前走（`++it`），所以必须使用
`const_iterator`，而不是在 `iterator` 前面加 `const`。

**`const_iterator`（类型只读）与
`const iterator`（变量只读）**

- **`const_iterator`（只读迭代器类型）**： 指的是**指向的数据不能改**。
- **`const iterator`（被 `const` 修饰的迭代器变量）**：
  指的是**这个迭代器变量自己不能改**（不能做
  `++it`，不能指向别的节点）。

这是最容易混淆的概念，可以用指针做对比：

| **类型** | **相当于原生指针** | **迭代器自身（指针本身）** | **指向的数据（内存里的值）** |
|----|----|----|----|
| `iterator` | `int* p` | 可变（可以 `++it`） | 可变（可以 `*it = 10`） |
| `const_iterator` | `const int* p` | 可变（可以 `++it`） | **不可变**（不能 `*it = 10`） |
| `const iterator` | `int* const p` | **不可变**（不能 `++it`） | 可变（可以 `*it = 10`） |
