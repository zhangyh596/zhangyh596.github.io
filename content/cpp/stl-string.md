---
title: "STL-string类的模拟实现"
date: "2026-06-14T16:58:14+08:00"
draft: false
categories: ["C++"]
tags: ["STL", "string", "模拟实现"]
---
## 1.定义类的基本结构

因为字符串的长度是可变的（今天存 "Hi"，明天可能存 "Hello
World"），我们不能在类里面写死一个固定大小的数组（比如
`char data[100]`，这样太浪费空间或者根本不够用）。所以，我们必须使用**动态内存分配**。

在 C++ 中，管理动态内存的黄金搭档就是：**一个指针** +
**一个长度变量**。（空间容量可选）

```c
class MyString
{
private:
    char *_data;
    size_t _length;
    size_t _capacity;
```


```c
public:
    // 构造与析构
    MyString();
    MyString(const char *str);
    ~MyString();
```


```c
    // 拷贝构造与移动构造
    MyString(const MyString &other);
    MyString(MyString &&other) noexcept;
```


```c
    // 赋值运算符
    MyString &operator=(const MyString &other);
    MyString &operator=(MyString &&other) noexcept;
```


```c
    // 常用功能
    const char *c_str() const;
    size_t size() const;
    void set_char(size_t index, char ch);
```


```c
    // 运算符重载 (成员函数)
    char &operator[](size_t index);
    const char &operator[](size_t index) const;
    MyString &operator+=(const MyString &other);
};
```


## 2.构造函数

分成无参构造函数和带参构造函数（也可以直接写成带默认值的构造函数）

1.  **无参构造函数**：创建一个空字符串（比如 `MyString s1;`）。         
```c
                                       关键点来了！`new` 是 C++
里的“圈地”命令。它会向操作系统申请一块内存。虽然是空字符串，但我们在底层也要申请
1 个字节的空间，用来放 `'\0'`。
```

2.  **带参构造函数**：用一个 C 语言风格的字符串来初始化（比如
```c
`MyString s2("Hello");`）。
```


<!-- -->

```c
// 1. 默认构造函数
MyString::MyString() : _length(0), _capacity(0)
{
    _data = new char[1];
    _data[0] = '\0';
}
```


```c
// 2. 带参构造函数
MyString::MyString(const char *str)
{
    if (str != nullptr)
    {
        _length = strlen(str);
        _capacity = _length;
        _data = new char[_capacity + 1];
        strcpy(_data, str);
    }
    else
    {
        _length = 0;
        _capacity = 0;
        _data = new char[1];
        _data[0] = '\0';
    }
}
```


**疑惑解答：带参构造函数里面为什么不直接
\_`data = str;`？**

- 如果直接等于，那是“浅拷贝”，两个指针指向了同一块地方。一旦外面的 `str`
  销毁了，我们的 \_`data`就变成无家可归的“野指针”了。

- 所以必须用 `new` 重新开辟一块属于 `MyString` 自己的专属空间（5个字符 +
  1个结束符 = 6个字节）。

## 3.析构函数和c_str()函数

```c
// 3. 析构函数
MyString::~MyString()
{
    delete[] _data;
}
```


```c
// 辅助函数
const char *MyString::c_str() const
{
    return _data ? _data : "";
}
```


### 辅助函数：`const char* c_str() const`

1.  **干嘛用的**：现在的 `MyString` 是个自定义类，`std::cout`
```c
还不认识它，直接打印会报错。这个函数把内部的底层指针
\_`data` 借给外面用一下，方便配合 `std::cout` 打印。
```

2.  **后面的 `const`
```c
是干嘛的**：这叫“常成员函数”。意思是承诺在这个函数内部，绝对不会悄悄修改类里的任何变量（比如不修改
\_`length`），只是纯粹地读数据。
```


接下来开始测试一下我们写的函数

```c
void test1()
{
    MyString s1;
    cout << s1.c_str() << endl;
```


```c
    MyString s2("hello c++");
    cout << s2.c_str() << endl;
}
```


## 4.拷贝构造函数（防止浅拷贝）

如果你自己不写，C++
编译器会非常热心地帮你自动生成一个默认的拷贝构造函数。但这个默认的函数是个“睁眼瞎”，它只会搞**浅拷贝（Shallow
Copy）**。

**浅拷贝的灾难**：它只是简单地把 `s1` 内部的 \_`length` 和
\_`data` 指针的值复制给 `s2`。结果就是 `s1._data` 和
`s2._data` **指向了堆内存中的同一块地方**

**后果**：

1.  **互相牵连**：你改 `s2` 的字符，`s1`
```c
的字符也跟着变了，它们没有独立主权。
```


2.  **内存崩溃（Double Free）**：当函数结束，`s2`
```c
先死，调用析构函数把这块内存 `delete[]` 了。接着 `s1` 死，它又去
`delete[]` 这块已经被释放的内存！程序瞬间崩溃。
```


<!-- -->

```c
// 4. 拷贝构造函数
MyString::MyString(const MyString &other)
{
    _length = other._length;
    _capacity = _length;
    _data = new char[_capacity + 1];
    strcpy(_data, other._data);
}
```


- **为什么参数是
  `const`**：因为我们只是去“抄作业”的，我们保证在复制的过程中，绝对不会不小心修改了被抄的对象（`other`）。

- **为什么一定要加引用符号 `&`？**：

  - 如果不加 `&`，参数就会变成“值传递”。

  - 值传递在传参的一瞬间，又会去调用拷贝构造函数。

  - 这样就会陷入：*为了调用拷贝构造函数 -> 需要传参 ->
```c
传参又触发拷贝构造函数 -> 再次传参...*
的**死循环（无限递归）**，直到电脑内存耗尽程序崩溃。
```


  - 加上 `&` 变成引用，就只是给旧对象起个别名叫
```c
`other`，直接拿来用，不会触发新的拷贝。
```


接下来我们测试一下，同时补充两个辅助函数

```c
size_t MyString::size() const
{
    return _length;
}
```


```c
void MyString::set_char(size_t index, char ch)
{
    if (index < _length)
    {
        _data[index] = ch;
    }
}
```


```c
void test2()
{
    MyString s1("hello");
    cout << s1.c_str() << endl;
```


```c
    MyString s2 = s1;
    cout << s2.c_str() << endl;
```


```c
    s2.set_char(0, 'B');
    cout << "修改后s2为：" << s2.c_str() << endl;
}
```


## 5.拷贝赋值运算符

很多人经常把这一步和上一步的“拷贝构造”搞混。我们先用一句话区分它们：

- **拷贝构造**：对象刚诞生，用别人来初始化自己（比如
  `MyString s2 = s1;`）。

- **拷贝赋值**：对象**已经出生很久了**，中途想换个新值（比如
  `MyString s1("Hello"); MyString s2("World"); s2 = s1;`）。

<!-- -->

```c
// 5. 拷贝赋值运算符
MyString &MyString::operator=(const MyString &other)
{
    if (this == &other)
    {
        return *this;
    }
```


```c
    delete[] _data;
```


```c
    _length = other._length;
    _capacity = _length;
    _data = new char[_capacity + 1];
    strcpy(_data, other._data);
```


```c
    return *this;
}
```


##### 为什么要检查 `if (this == &other)`？（天坑一：自我毁灭）

- `this` 是当前对象的指针，`&other` 是传进来的对象的地址。

- 想象一下，如果用户不小心写了
  `s1 = s1;`。如果你不加这行检查，程序会直接执行第二步：`delete[] _data;`。

- 结果就是：**你把自己的数据删了，紧接着第三步你试图去复制
  `other._data`，但 `other` 就是你自己，它的数据已经被你删干净了！**
  这就叫自我毁灭。所以必须先检查是不是同一个人。

##### 为什么要 `delete[] _data;`？（天坑二：内存泄漏）

- 因为 `s2` 在被赋值之前，它自己肚子里已经有一块用 `new`
  申请的内存了（比如装了 `"World"`）。

- 如果你不先 `delete[]` 掉它，就直接让 \_`data = new ...`
  指向新房子，那么 `"World"`
  那块内存就彻底断联了，变成了无法回收的僵尸内存。

##### 为什么返回值是 `MyString&`，最后要 `return *this;`？

- C++ 允许你写这样的代码：`s3 = s2 = s1;`（把 s1 赋给 s2，再把结果赋给
  s3）。

- 为了支持这种连等操作，`operator=`
  必须把**赋值完成后的自己**当作结果返回回去。`*this`
  代表当前对象本身，返回它的引用 `MyString&` 效率最高。

接下来我们再测试一下

```c
void test3()
{
    MyString s1("Apple");
    MyString s2("Banana");
    MyString s3("Orange");
```


```c
    cout << "赋值前:" << endl;
    cout << "s1: " << s1.c_str() << ", s2: " << s2.c_str() << ", s3: " << s3.c_str() << endl;
```


```c
    // 测试点 1：连续赋值 (s3 = s2 = s1)
    s3 = s2 = s1;
```


```c
    cout << "\n连续赋值后:" << endl;
    cout << "s1: " << s1.c_str() << ", s2: " << s2.c_str() << ", s3: " << s3.c_str() << endl;
```


```c
    // 测试点 2：修改 s2，看看 s1 和 s3 会不会被影响（验证深拷贝）
    s2.set_char(0, 'G'); // 把 s2 改成 "Gpple"
```


```c
    cout << "\n修改 s2 后的独立性测试:" << endl;
    cout << "s1 (应该还是 Apple): " << s1.c_str() << endl;
    cout << "s2 (应该变成了 Gpple): " << s2.c_str() << endl;
    cout << "s3 (应该还是 Apple): " << s3.c_str() << endl;
```


```c
    // 测试点 3：自我赋值测试，确保不崩溃
    s1 = s1;
    cout << "\n自我赋值后 s1 依然完好: " << s1.c_str() << endl;
}
```


## 6.移动语义（现代写法）

- **拷贝语义**：买一个新笔记本，把 1
  万字**抄一遍**，然后把旧笔记本烧了。这叫深拷贝，巨慢！

- **移动语义**：直接把旧笔记本上的**名字标签撕了，贴上你的新名字**。文字根本没动。这叫移动，瞬间完成！

移动语义的核心思想就是：**既然你这个临时对象马上就要死了，那与其让我拷贝一遍，不如把你的内存指针直接“偷”过来据为己有！**

```c
// 6. 移动构造函数
MyString::MyString(MyString &&other) noexcept
    : _data(other._data), _length(other._length), _capacity(other._capacity)
{
    other._data = nullptr;
    other._length = 0;
    other._capacity = 0;
    cout << "【触发移动构造函数】" << endl;
}
```


```c
// 7. 移动赋值运算符
MyString &MyString::operator=(MyString &&other) noexcept
{
    cout << "【触发移动赋值运算符】" << endl;
```


```c
    if (this != &other)
    {
        delete[] _data;
        _data = other._data;
        _length = other._length;
        _capacity = other._capacity;
```


```c
        other._data = nullptr;
        other._length = 0;
        other._capacity = 0;
    }
    return *this;
}
```


注意那个
`&&`！它读作“右值引用”。只要传进来的是临时对象，编译器就会自动优先调用这个函数，而不是拷贝构造。

`noexcept` 是一个关键字，它的字面意思是 **“No
Exception”（不抛出异常）**。把它写在函数后面，就是向编译器和系统发出的一个**硬核保证**

##### 为什么要执行 `other._data = nullptr;`？

- **这是移动语义中最关键的一步！** \* 此时，我们的
  \_`data` 指针已经指到了 `other._data` 的地盘。如果我们不把
  `other._data` 改为 `nullptr`，那么一会儿 `other`
  生命周期结束调用析构函数时，就会执行 `delete[] _data;`。

- 结果就是：**把我们刚刚偷过来的精美大宅子，直接给炸飞了！**

- 把它变成 `nullptr` 之后，`other` 析构时执行 `delete[] nullptr;`，这在
  C++ 里是完全安全且合法的，啥也不会发生。

在 `main` 函数中进行测试

因为平时我们写代码时很难手动制造一个“完美的临时对象”，所以 C++ 提供了
`std::move()`
函数，它可以把一个普通的左值**强制伪装成一个“快要死掉的临时对象”**，从而强行触发移动语义。

```c
void test3()
{
    MyString s1("very long string");
    cout << "s1初始内容：" << s1.c_str() << endl;
```


```c
    MyString s2 = move(s1);
```


```c
    cout << "移动后" << endl;
    cout << "s2的内容：" << s2.c_str() << endl;
    cout << "s1的内容：" << s1.c_str() << endl;
}
```


## 7.重载下标操作符

##### 为什么要重载两次？

在 C++ 中，当我们想要通过 `s[index]`
去访问字符串里的字符时，通常有两种情况：

1.  **读写模式**：比如 `s[0] = 'A';`，我们要去修改里面的字符。

2.  **只读模式**：比如把字符串传给一个只读函数
```c
`void print(const MyString& s)` 时，我们只想打印 `s[0]`，不能修改。
```


为了应对这两种情况，我们必须重载两个版本的
`operator[]`：一个是普通版本（返回引用，可修改），一个是常数版本（返回常量引用，只读）。

简单起见，这里不进行越界检查。 在真实的 std::string 中，[]
通常也不检查越界（为了极致性能）， 另一个函数 at()
才会检查越界并抛出异常

```c
// 8. 下标操作符 (普通)
char &MyString::operator[](size_t index)
{
    return _data[index];
}
```


```c
// 9. 下标操作符 (常量)
const char &MyString::operator[](size_t index) const
{
    return _data[index];
}
```


在 `main` 函数中进行测试

```c
void print_first_char(const MyString &str)
{
    if (str.size() > 0)
    {
        cout << "函数内部读取的首字母: " << str[0] << endl;
    }
}
```


```c
void test4()
{
    MyString s1("hello");
    cout << "初始内容：" << s1.c_str() << endl;
```


```c
    s1[0] = 'H';
    cout << "修改s1[0]后：" << s1.c_str() << endl;
```


```c
    for (size_t i = 0; i < s1.size(); ++i)
    {
        cout << s1[i] << " ";
    }
    cout << endl;
```


```c
    print_first_char(s1);
}
```


## 8.字符串拼接

**`operator+=`（追加）**：比如 `s1 += s2`。这会**改变 `s1`
自己**的内容。我们需要在堆内存里开辟一块更大的新房子，把 `s1`
原来的字搬进去，再把 `s2` 的字接着搬进去，最后把 `s1` 的旧房子炸掉。

**`operator+`（相加）**：比如 `s3 = s1 + s2`。注意，**`s1` 和 `s2`
都不能被改变！** 我们需要凭空创造一个全新的字符串 `s3`
来装它们俩的结果。

```c
// 10. 追加操作符 (+=)
MyString &MyString::operator+=(const MyString &other)
{
    if (other._length == 0)
        return *this;
```


```c
    size_t new_length = _length + other._length;
    char *new_data = new char[new_length + 1];
```


```c
    if (_data)
    {
        strcpy(new_data, _data);
    }
    else
    {
        new_data[0] = '\0';
    }
```


```c
    if (other._data)
    {
        strcat(new_data, other._data);
    }
```


```c
    delete[] _data;
```


```c
    _data = new_data;
    _length = new_length;
    _capacity = new_length;
```


```c
    return *this;
}
```


### 在 `MyString` 类**外部**添加 `operator+`

之所以不把它写进类里面，不是因为不能，而是为了追求 C++
中至高无上的“对称性（Symmetry）”。我们用一个非常现实的例子来解释。因为写在类里面的函数默认第一个参数其实是this指针。

- 正常运行

MyString s1("World");  
MyString s2 = s1 + "Hello"; // 相当于 s1.operator+("Hello")

- 直接报错崩溃

MyString s2("World");  
MyString s3 = "Hello" + s2; // 相当于 "Hello".operator+(s2) ？？？

你可能会问：“它既然在类外面，那如果它需要读取类里面的 `private`
变量（比如 `data_`），它不是没权限吗？”

有两种解决办法：

1.  **高级写法（我们用的这种）**：我们在全局 `+`
```c
里面，只调用了类内部公开的 `+=` 运算符。因为 `+=`
是类的内部成员，它可以合法访问所有私有变量。这种不依赖特权、只用公开接口的写法，在设计模式里叫**低耦合**，是非常高级的代码风格。
```


2.  **特殊通行证（`friend`
```c
友元）**：如果它真的非要直接读私有变量，我们可以在类内部给它开个后门，写一行
`friend MyString operator+(...);`。这样它就成了这个类的“好基友”，可以随意翻看私有隐私了。
```


<!-- -->

```c
// 11. 相加操作符 (+)
MyString operator+(const MyString &lhs, const MyString &rhs)
{
    MyString result(lhs);
    result += rhs;
    return result;
}
```


还记得我们第六步学的**移动语义**吗？ 这里的 `operator+` 在
`return result;` 的时候，返回的是一个**临时对象**。当你写
`MyString s3 = s1 + s2;` 时，编译器会发现 `s1 + s2`
产生的结果是个即将死去的临时工，于是**完美触发了我们之前写的移动构造函数（Move
Constructor）**！`s3` 会直接把 `result`
的底层内存“偷”走，完全不需要再发生一次深拷贝！性能直接拉满！

在 `main` 函数中进行测试

```c
void test5()
{
    MyString s1("hello");
    MyString s2(" world");
```


```c
    cout << "测试 + 号" << endl;
```


```c
    MyString s3 = s1 + s2;
```


```c
    cout << s1.c_str() << endl;
    cout << s2.c_str() << endl;
    cout << s3.c_str() << endl;
```


```c
    cout << "测试 += 号" << endl;
    s1 += s2;
    cout << s1.c_str() << endl;
}
```


## 9.重载输出流操作符

##### 为什么它也必须写在类外面？

- std::cout << s1;在这个式子里，**左边**的操作数是 `std::cout`（它属于
  `std::ostream` 类），**右边**的操作数才是我们的 `MyString`。
- 如果我们要把 `<<` 写成 `MyString`
  的成员函数，那我们的字符串就必须放在左边，代码就会变成极其反人类的：s1
  << std::cout; 
- 既然左边是被 C++ 标准库霸占的 `std::ostream`，我们又没有权限去修改 C++
  原生的 `std::ostream`
  源码，那唯一的办法就是：**在外面写一个全局函数，把 `std::cout` 和
  `MyString` 当作两个平等的参数传进去。**

<!-- -->

```c
// 12. 重载输出流操作符 (<<)
ostream &operator<<(ostream &os, const MyString &str)
{
    os << str.c_str();
    return os;
}
```


这里的参数和返回值大有学问：

- **第一个参数 `std::ostream& os`**： 这里**必须**加 `&`（引用）。因为
  C++
  严禁拷贝输入输出流对象（你想想，屏幕和键盘怎么能被复制呢？）。所以我们只能借用原本的那个
  `cout`。

- **为什么返回值是 `std::ostream&`，且最后要 `return os;`？**
  这是为了支持**连续打印（链式调用）**！ 当你写
  `std::cout << s1 << " " << s2;` 时：

  1.  系统先执行 `std::cout << s1`。

  2.  我们写好的函数打印完 `s1` 后，把 `std::cout` 又返回了回来。

  3.  于是式子变成了 `std::cout << " "`，继续打印空格。

  4.  空格打印完，又返回 `cout`，接着打印 `s2`。 一环扣一环，完美衔接！

在 `main`
函数中进行测试 我们可以把之前所有臃肿的 `.c_str()` 全都删掉了

```c
void test6()
{
    MyString s1("hello");
    MyString s2("C++");
    MyString s3("world");
```


```c
    cout << s1 << " " << s2 << " " << s3 << endl;
}
```


## 10.让字符串可以被比较

还记得我们在写 `operator+` 时学到的“对称性（Symmetry）”吗？

如果我们要判断 `"Hello" == s1`，为了让原生的 C
风格字符串也能在左边公平地参与比较，所有的比较运算符**都必须写在类外面，做成全局函数**。

在实现比较时，我们不用自己去傻乎乎地用 `for` 循环一个一个字符对比，C
语言的标准库里早就为我们准备好了神器：`std::strcmp()`。

- `strcmp(a, b) == 0`：说明两者完全相等。

- `strcmp(a, b) < 0`：说明 a 的字典序排在 b 前面（比如 "apple" <
  "banana"）。

- `strcmp(a, b) > 0`：说明 a 的字典序排在 b 后面。

<!-- -->

```c
// 13. 比较操作符
bool operator==(const MyString &lhs, const MyString &rhs)
{
    return strcmp(lhs.c_str(), rhs.c_str()) == 0;
}
```


```c
bool operator!=(const MyString &lhs, const MyString &rhs)
{
    return !(lhs == rhs);
}
```


```c
bool operator<(const MyString &lhs, const MyString &rhs)
{
    return strcmp(lhs.c_str(), rhs.c_str()) < 0;
}
```


```c
bool operator>(const MyString &lhs, const MyString &rhs)
{
    return rhs < lhs;
}
```


```c
bool operator<=(const MyString &lhs, const MyString &rhs)
{
    return !(lhs > rhs);
}
```


```c
bool operator>=(const MyString &lhs, const MyString &rhs)
{
    return !(lhs < rhs);
}
```


在 `main` 函数中进行测试

```c
void test7()
{
    MyString s1("apple");
    MyString s2("banana");
    MyString s3("apple");
```


```c
    cout << (s1 == s3 ? "Yes" : "No") << endl;
    cout << (s1 != s2 ? "Yes" : "No") << endl;
}
```


## 11.重载输入流操作符和实现全局的getline函数

标准 C++ 的 `cin >>`
有一个雷打不动的规矩：**跳过前面的空格/回车，遇到货就装，一旦再次遇到空格/回车，就立刻停手。**

`operator>>` 的实现代码拆成三步

1\. 清空旧仓库 2. 扔掉传送带前段的“垃圾”（空格、回车）3.
用“临时水桶”装货

```c
// 14. 重载流提取运算符 (>>)
istream &operator>>(istream &is, MyString &str)
{
    // 先清空原本字符串里的旧数据（直接赋一个空字符串对象）
    str = MyString();
```


```c
    char ch;
    // 跳过前导空白字符（空格、回车、制表符等）
    while (is.get(ch) && isspace(ch))
    {
    }
```


```c
    // 如果读到文件尾(EOF)或者流出错，直接返回
    if (!is)
        return is;
```


```c
    // 此时 ch 已经是第一个有效字符了
    char buf[1024];
    size_t index = 0;
    buf[index++] = ch;
```


```c
    // 循环读取，直到遇到下一个空白字符
    while (is.get(ch) && !isspace(ch))
    {
        buf[index++] = ch;
        if (index >= 1023)
        {
            buf[index] = '\0';
            str += MyString(buf);
            index = 0;
        }
    }
```


```c
    // 把最后残留在缓冲区的数据追加到 str
    if (index > 0)
    {
        buf[index] = '\0';
        str += MyString(buf);
    }
```


```c
    return is;
}
```


工人开始从传送带上拿东西（`is.get(ch)` 表示拿一个字符）

`isspace(ch)`
是个判断函数：如果这个字符是空格、制表符（Tab）或者回车，它就返回“真”。

因为我们不知道用户到底会输入多长的字符串，如果每拿一个字符，就去堆内存申请一次空间（`new`
一次），电脑会累死。所以我们准备了一个能装 1024 个字符的**大水桶**
`char buf[1024]`。

##### 为什么不用 `cin >> ch` 而是用 `is.get(ch)`？

- **因为 `cin >>` 实在太傲娇了，它一旦看到空格，连碰都不碰。**
  如果我们用 `is >> ch`，那第二个 `while`
  循环（判断什么时候结束的那个）就**永远没办法读到单词后面的那个空格**！它会误以为后面一直有货，导致把所有单词连在一起读进来。
- 而 `is.get(ch)`
  就像一个毫无感情的机器人，不管是字母、数字、还是空格、换行，只要在传送带上，它通通都能拿起来。这样我们才能通过
  `isspace(ch)` 亲手去掌控“什么时候该停手”。

### getline函数

`getline`
的核心目标只有一个：**“不管你里面有几个空格、几个Tab，只要没遇到那堵‘叹息之墙’（换行符），我就全给你铲进箱子里！”**

```c
// 15. 实现 getline 函数
// 核心逻辑：不跳过前导空白，一直读取直到遇到指定的结束符（默认是 '\n'）
istream &getline(istream &is, MyString &str, char delim)
{
    // 清空原本的数据
    str = MyString();
```


```c
    char buf[1024];
    size_t index = 0;
    char ch;
```


```c
    // 循环读取，直到读到分隔符或者流结束
    while (is.get(ch) && ch != delim)
    {
        buf[index++] = ch;
        if (index >= 1023) // 缓冲区满了，先倒进 str
        {
            buf[index] = '\0';
            str += MyString(buf);
            index = 0;
        }
    }
```


```c
    // 倒出最后残留的数据
    if (index > 0)
    {
        buf[index] = '\0';
        str += MyString(buf);
    }
```


```c
    return is;
}
```


你仔细看 `getline` 的代码，相比之前的 `cin >>`，它**少了一个
`while (isspace(ch))` 的循环**。

`ch != delim`：看看这个字符是不是 `delim`。我们前面设定了 `delim` 默认是
`'\n'`（回车键）。所以这就相当于问：**“这是回车键吗？”**
如果不是，就继续装！

在 `main.cpp` 里面测试

```c
void test8()
{
    MyString s1;
    MyString s2;
```


```c
    cout << "========= 测试 cin >> =========" << endl;
    cout << "请输入两个单词（用空格隔开）: ";
    cin >> s1 >> s2;
    cout << "读取到的单词 1: " << s1 << endl;
    cout << "读取到的单词 2: " << s2 << endl;
```


```c
    // 吸收掉残留的回车符，防止影响接下来的 getline
    char flush_char;
    cin.get(flush_char);
```


```c
    cout << "\n========= 测试 getline =========" << endl;
    cout << "请输入一整行带空格的句子: ";
    getline(cin, s1);
    cout << "读取到的整行内容为: " << s1 << endl;
}
```


## 12.完整代码

### MyString.h

```c
#ifndef MYSTRING_H
#define MYSTRING_H
```


```c
#include <iostream>
#include <cstring>
#include <utility> // 为了 move
```


```c
class MyString
{
private:
    char *_data;
    size_t _length;
    size_t _capacity;
```


```c
public:
    // 构造与析构
    MyString();
    MyString(const char *str);
    ~MyString();
```


```c
    // 拷贝构造与移动构造
    MyString(const MyString &other);
    MyString(MyString &&other) noexcept;
```


```c
    // 赋值运算符
    MyString &operator=(const MyString &other);
    MyString &operator=(MyString &&other) noexcept;
```


```c
    // 常用功能
    const char *c_str() const;
    size_t size() const;
    void set_char(size_t index, char ch);
```


```c
    // 运算符重载 (成员函数)
    char &operator[](size_t index);
    const char &operator[](size_t index) const;
    MyString &operator+=(const MyString &other);
};
```


```c
// 运算符重载 (非成员函数声明)
MyString operator+(const MyString &lhs, const MyString &rhs);
std::ostream &operator<<(std::ostream &os, const MyString &str);
std::istream &operator>>(std::istream &is, MyString &str);
std::istream &getline(std::istream &is, MyString &str, char delim = '\n');
```


```c
bool operator==(const MyString &lhs, const MyString &rhs);
bool operator!=(const MyString &lhs, const MyString &rhs);
bool operator<(const MyString &lhs, const MyString &rhs);
bool operator>(const MyString &lhs, const MyString &rhs);
bool operator<=(const MyString &lhs, const MyString &rhs);
bool operator>=(const MyString &lhs, const MyString &rhs);
```


```c
#endif // MYSTRING_H
```


### MyString.cpp

```c
#include "MyString.h"
```


```c
using namespace std; // 在 .cpp 文件中可以使用这个，不影响其他文件
```


```c
// 1. 默认构造函数
MyString::MyString() : _length(0), _capacity(0)
{
    _data = new char[1];
    _data[0] = '\0';
}
```


```c
// 2. 带参构造函数
MyString::MyString(const char *str)
{
    if (str != nullptr)
    {
        _length = strlen(str);
        _capacity = _length;
        _data = new char[_capacity + 1];
        strcpy(_data, str);
    }
    else
    {
        _length = 0;
        _capacity = 0;
        _data = new char[1];
        _data[0] = '\0';
    }
}
```


```c
// 3. 析构函数
MyString::~MyString()
{
    delete[] _data;
}
```


```c
// 辅助函数
const char *MyString::c_str() const
{
    return _data ? _data : "";
}
```


```c
size_t MyString::size() const
{
    return _length;
}
```


```c
void MyString::set_char(size_t index, char ch)
{
    if (index < _length)
    {
        _data[index] = ch;
    }
}
```


```c
// 4. 拷贝构造函数
MyString::MyString(const MyString &other)
{
    _length = other._length;
    _capacity = _length;
    _data = new char[_capacity + 1];
    strcpy(_data, other._data);
}
```


```c
// 5. 拷贝赋值运算符
MyString &MyString::operator=(const MyString &other)
{
    if (this == &other)
    {
        return *this;
    }
```


```c
    delete[] _data;
```


```c
    _length = other._length;
    _capacity = _length;
    _data = new char[_capacity + 1];
    strcpy(_data, other._data);
```


```c
    return *this;
}
```


```c
// 6. 移动构造函数
MyString::MyString(MyString &&other) noexcept
    : _data(other._data), _length(other._length), _capacity(other._capacity)
{
    other._data = nullptr;
    other._length = 0;
    other._capacity = 0;
    cout << "【触发移动构造函数】" << endl;
}
```


```c
// 7. 移动赋值运算符
MyString &MyString::operator=(MyString &&other) noexcept
{
    cout << "【触发移动赋值运算符】" << endl;
```


```c
    if (this != &other)
    {
        delete[] _data;
        _data = other._data;
        _length = other._length;
        _capacity = other._capacity;
```


```c
        other._data = nullptr;
        other._length = 0;
        other._capacity = 0;
    }
    return *this;
}
```


```c
// 8. 下标操作符 (普通)
char &MyString::operator[](size_t index)
{
    return _data[index];
}
```


```c
// 9. 下标操作符 (常量)
const char &MyString::operator[](size_t index) const
{
    return _data[index];
}
```


```c
// 10. 追加操作符 (+=)
MyString &MyString::operator+=(const MyString &other)
{
    if (other._length == 0)
        return *this;
```


```c
    size_t new_length = _length + other._length;
    char *new_data = new char[new_length + 1];
```


```c
    if (_data)
    {
        strcpy(new_data, _data);
    }
    else
    {
        new_data[0] = '\0';
    }
```


```c
    if (other._data)
    {
        strcat(new_data, other._data);
    }
```


```c
    delete[] _data;
```


```c
    _data = new_data;
    _length = new_length;
    _capacity = new_length;
```


```c
    return *this;
}
```


```c
// --- 下面是非成员函数的实现 ---
```


```c
// 11. 相加操作符 (+)
MyString operator+(const MyString &lhs, const MyString &rhs)
{
    MyString result(lhs);
    result += rhs;
    return result;
}
```


```c
// 12. 重载输出流操作符 (<<)
ostream &operator<<(ostream &os, const MyString &str)
{
    os << str.c_str();
    return os;
}
```


```c
// 14. 重载流提取运算符 (>>)
istream &operator>>(istream &is, MyString &str)
{
    // 先清空原本字符串里的旧数据（直接赋一个空字符串对象）
    str = MyString();
```


```c
    char ch;
    // 跳过前导空白字符（空格、回车、制表符等）
    while (is.get(ch) && isspace(ch))
    {
    }
```


```c
    // 如果读到文件尾(EOF)或者流出错，直接返回
    if (!is)
        return is;
```


```c
    // 此时 ch 已经是第一个有效字符了
    char buf[1024];
    size_t index = 0;
    buf[index++] = ch;
```


```c
    // 循环读取，直到遇到下一个空白字符
    while (is.get(ch) && !isspace(ch))
    {
        buf[index++] = ch;
        if (index >= 1023)
        {
            buf[index] = '\0';
            str += MyString(buf);
            index = 0;
        }
    }
```


```c
    // 把最后残留在缓冲区的数据追加到 str
    if (index > 0)
    {
        buf[index] = '\0';
        str += MyString(buf);
    }
```


```c
    return is;
}
```


```c
// 15. 实现 getline 函数
// 核心逻辑：不跳过前导空白，一直读取直到遇到指定的结束符（默认是 '\n'）
istream &getline(istream &is, MyString &str, char delim)
{
    // 清空原本的数据
    str = MyString();
```


```c
    char buf[1024];
    size_t index = 0;
    char ch;
```


```c
    // 循环读取，直到读到分隔符或者流结束
    while (is.get(ch) && ch != delim)
    {
        buf[index++] = ch;
        if (index >= 1023) // 缓冲区满了，先倒进 str
        {
            buf[index] = '\0';
            str += MyString(buf);
            index = 0;
        }
    }
```


```c
    // 倒出最后残留的数据
    if (index > 0)
    {
        buf[index] = '\0';
        str += MyString(buf);
    }
```


```c
    return is;
}
```


```c
// 13. 比较操作符
bool operator==(const MyString &lhs, const MyString &rhs)
{
    return strcmp(lhs.c_str(), rhs.c_str()) == 0;
}
```


```c
bool operator!=(const MyString &lhs, const MyString &rhs)
{
    return !(lhs == rhs);
}
```


```c
bool operator<(const MyString &lhs, const MyString &rhs)
{
    return strcmp(lhs.c_str(), rhs.c_str()) < 0;
}
```


```c
bool operator>(const MyString &lhs, const MyString &rhs)
{
    return rhs < lhs;
}
```


```c
bool operator<=(const MyString &lhs, const MyString &rhs)
{
    return !(lhs > rhs);
}
```


```c
bool operator>=(const MyString &lhs, const MyString &rhs)
{
    return !(lhs < rhs);
}
```


### main.cpp

```c
#include "MyString.h"
```


```c
using namespace std;
```


```c
void test1()
{
    MyString s1;
    cout << s1.c_str() << endl;
```


```c
    MyString s2("hello c++");
    cout << s2.c_str() << endl;
}
```


```c
void test2()
{
    MyString s1("hello");
    cout << s1.c_str() << endl;
```


```c
    MyString s2 = s1;
    cout << s2.c_str() << endl;
```


```c
    s2.set_char(0, 'B');
    cout << "修改后s2为：" << s2.c_str() << endl;
}
```


```c
void test3_1()
{
    MyString s1("Apple");
    MyString s2("Banana");
    MyString s3("Orange");
```


```c
    cout << "赋值前:" << endl;
    cout << "s1: " << s1.c_str() << ", s2: " << s2.c_str() << ", s3: " << s3.c_str() << endl;
```


```c
    // 测试点 1：连续赋值 (s3 = s2 = s1)
    s3 = s2 = s1;
```


```c
    cout << "\n连续赋值后:" << endl;
    cout << "s1: " << s1.c_str() << ", s2: " << s2.c_str() << ", s3: " << s3.c_str() << endl;
```


```c
    // 测试点 2：修改 s2，看看 s1 和 s3 会不会被影响（验证深拷贝）
    s2.set_char(0, 'G'); // 把 s2 改成 "Gpple"
```


```c
    cout << "\n修改 s2 后的独立性测试:" << endl;
    cout << "s1 (应该还是 Apple): " << s1.c_str() << endl;
    cout << "s2 (应该变成了 Gpple): " << s2.c_str() << endl;
    cout << "s3 (应该还是 Apple): " << s3.c_str() << endl;
```


```c
    // 测试点 3：自我赋值测试，确保不崩溃
    s1 = s1;
    cout << "\n自我赋值后 s1 依然完好: " << s1.c_str() << endl;
}
```


```c
void test3_2()
{
    MyString s1("very long string");
    cout << "s1初始内容：" << s1.c_str() << endl;
```


```c
    MyString s2 = move(s1);
```


```c
    cout << "移动后" << endl;
    cout << "s2的内容：" << s2.c_str() << endl;
    cout << "s1的内容：" << s1.c_str() << endl;
}
```


```c
void print_first_char(const MyString &str)
{
    if (str.size() > 0)
    {
        cout << "函数内部读取的首字母: " << str[0] << endl;
    }
}
```


```c
void test4()
{
    MyString s1("hello");
    cout << "初始内容：" << s1.c_str() << endl;
```


```c
    s1[0] = 'H';
    cout << "修改s1[0]后：" << s1.c_str() << endl;
```


```c
    for (size_t i = 0; i < s1.size(); ++i)
    {
        cout << s1[i] << " ";
    }
    cout << endl;
```


```c
    print_first_char(s1);
}
```


```c
void test5()
{
    MyString s1("hello");
    MyString s2(" world");
```


```c
    cout << "测试 + 号" << endl;
```


```c
    MyString s3 = s1 + s2;
```


```c
    cout << s1.c_str() << endl;
    cout << s2.c_str() << endl;
    cout << s3.c_str() << endl;
```


```c
    cout << "测试 += 号" << endl;
    s1 += s2;
    cout << s1.c_str() << endl;
}
```


```c
void test6()
{
    MyString s1("hello");
    MyString s2("C++");
    MyString s3("world");
```


```c
    cout << s1 << " " << s2 << " " << s3 << endl;
}
```


```c
void test7()
{
    MyString s1("apple");
    MyString s2("banana");
    MyString s3("apple");
```


```c
    cout << (s1 == s3 ? "Yes" : "No") << endl;
    cout << (s1 != s2 ? "Yes" : "No") << endl;
}
```


```c
void test8()
{
    MyString s1;
    MyString s2;
```


```c
    cout << "========= 测试 cin >> =========" << endl;
    cout << "请输入两个单词（用空格隔开）: ";
    cin >> s1 >> s2;
    cout << "读取到的单词 1: " << s1 << endl;
    cout << "读取到的单词 2: " << s2 << endl;
```


```c
    // 吸收掉残留的回车符，防止影响接下来的 getline
    char flush_char;
    cin.get(flush_char);
```


```c
    cout << "\n========= 测试 getline =========" << endl;
    cout << "请输入一整行带空格的句子: ";
    getline(cin, s1);
    cout << "读取到的整行内容为: " << s1 << endl;
}
```


```c
int main()
{
    cout << "--- 测试 ---" << endl;
    // test1();
    // test2();
    // test3();
    // test4();
    // test5();
    // test6();
    // test7();
    test8();
    return 0;
}
```

