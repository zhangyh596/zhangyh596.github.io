---
title: "二分查找三种模板（C语言&amp;C++）最新最详细，带你拿下二分查找"
date: "2026-03-09T15:47:59+08:00"
draft: false
categories: ["算法"]
tags: ["二分查找", "算法模板", "二分"]
---
相信大家每次在写二分查找的时候，都会有不少问题。循环条件到底是 <还是
`<=`？边界更新是 `mid` 还是
`mid + 1`？这些细节经常让人抓狂，哪怕是经验丰富的程序员也会偶尔掉进死循环的坑里。

接下来为了帮助你解决困惑和理清思路，我总结了三大模板帮助你

## 前言：注意事项

首先二分查找针对的是有序数组，所以最重要的第一步当然是给数组进行排序qsort和sort都可以

在二分查找过程中，如果是向下取整，我们采用（left + right）/
2;如果是向上取整，我们采用（left + right + 1）/
2，这是因为mid是int类型，会自动约掉后面的小数

同时我们可能会遇到溢出的情况（就是left +
right可能会超出int类型的范围），我们采用先算差值，把加法拆分，于是可以写成left
+（right - left）/ 2，很好解决了溢出的情况，接下来开始正言

![](/images/practice/15978cd856df4871b9e5b5cba9bf249c.png)

找左端点和找右端点都可以把区间拆分成两部分，利用这种性质来查找

## 模板一：左闭右闭[left, right]

**初始状态：**left = 0， right = numsSize - 1；

**循环条件：**while(left <= right)（这是因为如果left ==
right时，此时刚好还剩下最后一个元素需要检查）

**收缩逻辑：**一旦检查过mid，mid就会被排除出下一步的搜索区间，所以一定会有+1和-1

### 找左端点

```c
int binarySearchLeft1(int* nums, int numsSize, int target)
{
    int left = 0, right = numsSize - 1;
    while (left <= right)
    {
        int mid = left + (right - left) / 2;
        if (nums[mid] >= target)
        {
            right = mid - 1;//注意这一步，待会会解释
        }
        else
        {
            left = mid + 1;
        }
    }
```


```c
    if (left < numsSize && nums[left] == target)//这一步也很重要，因为数组里面可能根本就没有target
        return left;
    return -1;
}
```


深度剖析right = mid -
1哪一步，其实就是要知道循环不变量（就是left永远会比right大1，即最终left
== right + 1）

首先，如果是nums[mid] >
target的话，很好理解，因为mid不是我们要找的点，所以right = mid -
1；但是如果nums[mid] == target的话，为什么也要让right = mid - 1

想象一个数组[1, 2, 2, 2,
3]，你要找2，mid刚好指向了中间的3，但是你是要找左端点，所以你必须要让right
= mid - 1

你可能又会疑惑万一中间的那个mid刚好就是我们要找的第一个2呢（这其实没关系，因为left会给我们兜底）

- 执行完right = mid  - 1的步骤，我们就进入了mid左边的区域查找
- 如果在左边再也找不到target了，也就是接下来的数都会< target
- 这时候程序就会不断的进行left = mid + 1这一步
- 也就是left一路向右走，当走到right这个位置的时候，进入最后一次循环，left依旧向右走，最终刚好走到我们最开始抛弃的那个mid的位置
- 然后left > right，循环到此结束，返回left就拿到了正确答案

为什么不能写成right =
mid，这是因为在闭区间的模板，每次缩小区间必须要有+1或-1，否则极易陷入死循环

假设最终left 和 right指向同一个元素，如果此时这个元素满足nums[mid] >=
target，就会执行right =
mid这一步，而right又没有改变，于是就卡在了这一步陷入了死循环

所以，既然 `while (left <= right)` 意味着 `mid`
已经被我们“检查过”了，不管它是大、是小、还是等于，我们都要干净利落地把它从下一次搜索的区间里踢出去（`mid - 1`
或 `mid + 1`）。

或者采用这种写法

```c
int searchLeft_T1(int* nums, int numsSize, int target) {
    int left = 0, right = numsSize - 1;
    int ans = -1; // 用来记录最新找到的 target 位置
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) {
            ans = mid;       // 记录下来
            right = mid - 1; // 既然找左端点，那就继续去左半边找找看
        } else if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return ans;
}
```


### 找右端点

```c
int binarySearchRight1(int* nums, int numsSize, int target)
{
    int left = 0, right = numsSize - 1;
    while (left <= right)
    {
        int mid = left + (right - left) / 2;
        if (nums[mid] <= target)
        {
            left = mid + 1;//关键的一步（和找左端点的思想同理）
        }
        else
        {
            right = mid - 1;
        }
    }
```


```c
    if (right >= 0 && nums[right] == target)
        return right;
    return -1;
}
```


让left = mid + 1其实是有right给我们兜底

- 实战一下，假设有一个数组[1, 2, 2, 2, 3]，我们要找2
- 第一轮left = 0， right = 4，找到下标为2的中间那个2，此时执行left =
  mid + 1（搜索区间变成了[3, 4]）
- 第二轮left = 3， right = 4，计算出mid =
  3，而nums[3]刚好是最后一个2，但是我们依旧执行left = mid +
  1这一步（搜索区间变成了[4, 4]）
- 之后数组里面的元素都会＞target，于是只会进入else分支，也就是right一直向右走，最终刚好right
  = 3，left = 4跳出循环

为什么找左端点要return left，找右端点要return
right是因为在循环结束之后，left和right一定会交错，left == right +
1（循环不变量）

- 找左端点因为遇到 >= 时我们一直在推 right，所以
  right 最终会停在最后一个小于** target 的元素**上。而 left 被推到了
  right 的右边一个位置，所以 left 永远指向第一个大于等于** target
  的元素**
- 找右端点因为遇到 <`=` 时我们一直在推 `left`，所以 `left`
  最终会停在**第一个大于 target 的元素**上。而 `right` 被推到了 `left`
  的左边一个位置，所以 `right` 永远指向**最后一个小于等于 target
  的元素**

**知道了这个我们也可以利用二分查找找到第一个大于target的数和最后一个小于target的数**

或者采用这种写法

```c
int searchRight_T1(int* nums, int numsSize, int target) {
    int left = 0, right = numsSize - 1;
    int ans = -1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) {
            ans = mid;       // 记录下来
            left = mid + 1;  // 既然找右端点，那就继续去右半边找找看
        } else if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return ans;
}
```


### 适用于没有重复元素的写法

```c
int binarySearch(int* nums, int numsSize, int target)
{
    if (nums == NULL || numsSize == 0)
        return -1;
```


```c
    int left = 0, right = numsSize - 1;
    while (left <= right)
    {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target)
        {
            return mid;//找到目标
        }
        else if (nums[mid] < target)
        {
            left = mid + 1;//目标在右侧
        }
        else
        {
            right = mid - 1;//目标在左侧
        }
    }
    return -1;
}
```


这其实才是正宗的左闭右闭的写法，但是却找不到左端点和右端点，如果数组没有重复元素的话可以采用这种方法，但如果有重复元素，也就是要找左端点和右端点，其实我更喜欢模板二的写法

##  模板二：非对称挤压（竞赛常用）

![](/images/practice/19aafcf02b97406b884142cee07f3db0.png)

**初始条件：**left = 0, right = numsSize - 1

**循环条件：**while（left < right）,但left == right时循环终止

**判断与收缩：此时重点看一下我们上面画的图像，拆分成了两个区间**

找左端点（拆分成了>=target的区间和＜target的区间）

- 如果 `nums[mid] < target`，说明目标在 `mid` 的右侧，更新
  left` = mid + 1`。
- 如果 `nums[mid] >= target`，说明目标可能是 `mid` 或者在 `mid`
  的左侧。此时**不能** `right = mid - 1`（否则会漏掉刚好是目标的
  `mid`），必须更新为 `right = mid`。

找右端点（拆分成了<=target的区间和>target的区间）

- 如果 `nums[mid] > target`，说明目标在 `mid` 的左侧，更新
  right` = mid - 1`。
- 如果 `nums[mid] <= target`，说明目标可能是 `mid` 或者在 `mid`
  的右侧。此时**不能** left` = mid + 1`（否则会漏掉刚好是目标的
  `mid`），必须更新为 left` = mid`。

**结束状态：**循环结束后left ==
right，我们需要额外检查一下nums[left]是否真的等于target，如果不是说明目标不存在

### 找左端点

```c
int binarySearchLeft2(int* nums, int numsSize, int target)
{
    int left = 0, right = numsSize;//这里right可以=numsSize（类似左闭右开的写法）也可以=numsSize - 1
    while (left < right)
    {
        int mid = left + (right - left) / 2;//向下取整
        if (nums[mid] >= target)
        {
            right = mid;
        }
        else
        {
            left = mid + 1;
        }
    }
    if (left < numsSize && nums[left] == target)
        return left;
    return -1;
}
```


为什么要向下取整

- 假设到了最后一步，区间只剩下两个元素nums = [3,
  3],而我们刚好要找3的左端点，此时left = 0，right = 1
- 此时向下取整mid = 0，nums[mid]刚好=3
- 走right= mid 的分支，left = right = 0
- 此时跳出循环，left就是答案

如果写了向上取整，那么第二步就会算出mid = 1，然后还是走right = mid
的分支，right依旧是1，然后就陷入死循环了

**结论： 因为你要执行
`right = mid`，为了防止 `right` 原地踏步，`mid` 就绝不能偏向
`right`。所以必须向下取整（偏向 left）。**

### 找右端点

```c
int binarySearchRight2(int* nums, int numsSize, int target)
{
    int left = 0, right = numsSize - 1;
    while (left < right)
    {
        int mid = left + (right - left + 1) / 2;//向上取整
        if (nums[mid] <= target)
        {
            left = mid;
        }
        else
        {
            right = mid - 1;
        }
    }
```


```c
    if (left < numsSize && nums[left] == target)
        return left;
    return -1;
}
```


为什么要向上取整

- 依旧假设剩下最后两个元素nums[3, 3]，我们要找3的右端点，left = 0，
  right = 1
- 此时mid = 1，nums[mid]刚好等于3
- 走left = mid 的分支
- 此时left = right = 1，成功跳出循环

如果写了向下取整，那么第二步就会算出mid = 0，然后还是走left = mid
的分支，left还是等于0，最终陷入死循环

**结论： 因为你要执行 `left = mid`，为了防止
`left` 原地踏步，`mid` 绝不能偏向 `left`。所以必须向上取整（偏向
right）。**

## 模板三：left + 1 < right(万能绝对安全模板）（这种写法比较少见）

这是一种更灵活的模板，每次区间收缩时保证至少保留两个元素。常用于需要访问
`mid` 左右相邻元素的情况。

**初始条件：**left = 0， right = numsSize - 1；

**循环条件：**while(left + 1 <
right)。这意味着只要区间内还有至少三个元素，循环继续

**判断与收缩：**由于循环结束时至少还剩下两个元素，因此在收缩的时候不需要纠结mid+1还是mid-1，直接把mid赋值给left或者right其中一个就可以，这样绝对不会越界也不会死循环

**结束状态：**循环结束后，剩下left和right两个候选位置，需要在循环外分别对这两个位置进行处理后判断

### 找左端点

```c
int binarySearchLeft3(int* nums, int numsSize, int target)
{
    if (nums == NULL || numsSize == 0) return -1; // 边界防御,否则最后查看nums[left]和nums[right]可能会出现越界访问
```


```c
    int left = 0, right = numsSize - 1;
    //至少有三个元素才循环
    while (left + 1 < right)
    {
        int mid = left + (right - left) / 2;
        if (nums[mid] >= target)
        {
            right = mid;// 目标在左边或者就是 mid，把右边界收缩过来
        }
        else
        {
            left = mid;// nums[mid] < target，目标在右边，把左边界收缩过来，可以写成left = mid + 1，但是没必要，我们多找一次也无妨，减去记忆负担
        }
    }
    // 退出循环时，区间必定只剩下 left 和 right 两个相邻的元素
    // 因为找的是【左】端点，所以理所当然优先检查【左】边的 left
    if (nums[left] == target) return left;
    if (nums[right] == target) return right;// 左边不是，兜底查一下右边
    return -1;
}
```


**安全阀门 `while (left + 1 < right)`**

**`这是这个模板的灵魂，只有但left和right之间隔着至少一个元素循环才能够执行`**

**核心数学原理： 只要能进循环，`left` 和 `right` 的距离>=2。
此时计算出来的 `mid = left + (right - left) / 2`，绝对不可能等于
`left`，也绝对不可能等于 `right`！`mid`
永远是一个位于它们中间的全新下标。**

**好处：因为 `mid` 绝不等于 `left` 和 `right`，所以无论你接下来执行
`left = mid` 还是 `right = mid`，区间都必定会实质性地缩小（至少缩小 1
个单位）。这从根本上杜绝了死循环的可能性！**

**注意事项：找左端点一定要先看left，因为如果你先看right的话，万一下标为left和right的元素都是target，你先看right就返回right了，就出现了错误**

### 找右端点

```c
int binarySearchRight1(int* nums, int numsSize, int target)
{
    int left = 0, right = numsSize - 1;
    //区间至少有三个元素才循环
    while (left + 1 < right)
    {
        int mid = left + (right - left) / 2;
        if (nums[mid] <= target)
        {
            left = mid;// 目标在右边或者就是 mid，把左边界收缩过来
        }
        else
        {
            right = mid;// nums[mid] > target，目标在左边，把右边界收缩过来
        }
    }
    // 退出循环时，区间必定只剩下 left 和 right 两个相邻的元素
    // 因为找的是【右】端点，所以理所当然优先检查【右】边的 right
    if (nums[right] == target) return right;
    if (nums[left] == target) return left; // 右边不是，兜底查一下左边
    return -1;
}
```


**注意事项：找右端点一定要先看right，因为如果你先看left的话，万一下标为left和right的元素都是target，你先看left就返回left了，就出现了错误**

## 总结

总的来说三种模板各有各的优点

1.  **模板一（扫描仪）：while(left <= right)**
2.  **特点：有 +1 和 -1，最后区间为空。**
3.  **适用：最基础的找某个数是否存在（不需要找边界）**

<!-- -->

1.  **模板二（老虎钳）：while(left < right)**
2.  **特点：要区分取整方向，只有一边 +1或一边 -1，最后剩下一个元素。**
3.  **适用：C++ STL 标准库的底层逻辑，代码最紧凑，但需要极高的熟练度。**

<!-- -->

1.  **模板三（绝对安全）：while(left + 1 < right)**
2.  **特点：无脑向下取整，无脑 left=mid 和
```c
right=mid，最后剩下两个元素，靠 if 顺序定乾坤。**
```

3.  **适用：面试实战的防翻车神器。当你脑子糊涂、面对极其复杂的边界条件时，直接掏出这套模板，写出来绝对一次过！**

我们其实没必要说掌握三种模板，掌握一种就足以应对了，你可以选择一个你喜欢的模板记下来，本期内容到此结束了，对你有帮助的可以收藏，制作不易！
