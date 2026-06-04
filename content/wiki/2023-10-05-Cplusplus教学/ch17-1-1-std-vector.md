---
title: "std::vector"
---

## 本节解决什么问题

在 C 语言中，数组的大小在编译时就必须确定，无法在运行时动态改变。如果你不知道数据量有多少，只能定义一个"足够大"的数组，浪费内存，或者用 `malloc/free` 手动管理动态内存，容易出错。

`std::vector` 解决了这个问题：它是一个**能自动扩缩容的动态数组**，连续存储，支持随机访问，是你在 C++ 中应该首选使用的容器。

## 这个特性是什么

`std::vector` 是 C++ STL 中的动态数组容器。它把元素放在**连续**的内存中（和普通数组一样），但大小可以在运行时动态增长或缩小。你不需要手动 `new/delete`，`vector` 会在离开作用域时自动释放内存。

## C++ 标准版本

C++98（C++11 起增加了 `emplace_back`、移动语义等增强）。

## 需要的头文件

```cpp
#include <vector>
```

## 基本语法

```cpp
std::vector<元素类型> 变量名;                     // 空容器
std::vector<元素类型> 变量名(大小);                // 指定大小的容器
std::vector<元素类型> 变量名(大小, 初始值);         // 指定大小和初始值
std::vector<元素类型> 变量名 = {值1, 值2, ...};    // 列表初始化
```

## 常用用法

先记住一句话：`vector` 像一个会自动变长的数组，但它不只是 `push_back` 那么简单。学 `vector` 时要同时理解四件事：

1. **元素个数**：现在真正存了多少个元素，用 `size()` 看。
2. **容量**：底层已经申请了多大空间，用 `capacity()` 看。
3. **访问方式**：下标、迭代器、范围 for、原始指针各有适用场景。
4. **修改后果**：插入、删除、扩容可能导致迭代器和引用失效。

### 创建和容量

| 操作 | 说明 | 常见场景 |
|:---|:---|:---|
| `std::vector<int> v;` | 创建空 vector | 数据量运行时才知道 |
| `std::vector<int> v(n);` | 创建 `n` 个元素，默认初始化 | 你已经确定需要 `n` 个位置 |
| `std::vector<int> v(n, value);` | 创建 `n` 个元素，每个都是 `value` | 初始化固定数量的默认值 |
| `std::vector<int> v = {1, 2, 3};` | 列表初始化 | 写测试数据、配置表 |
| `v.size()` | 当前元素个数 | 遍历、判断范围 |
| `v.capacity()` | 当前底层容量 | 观察是否可能扩容 |
| `v.reserve(n)` | 预留容量，不创建元素 | 提前知道最多会放多少个元素 |
| `v.resize(n)` | 改变元素个数，多出来的元素会被创建 | 真的需要多出 `n` 个可访问元素 |
| `v.empty()` | 判断是否为空 | `front()` / `back()` / `pop_back()` 前先判断 |

### 访问元素

| 操作 | 说明 | 注意 |
|:---|:---|:---|
| `v[i]` | 下标访问 | 不检查越界，速度快但危险 |
| `v.at(i)` | 下标访问 | 检查越界，越界抛 `std::out_of_range` |
| `v.front()` | 第一个元素 | 空 vector 上调用是未定义行为 |
| `v.back()` | 最后一个元素 | 空 vector 上调用是未定义行为 |
| `v.data()` | 返回底层连续内存指针 | 常用于和 C API / 底层库交互 |

### 遍历元素

| 操作 | 说明 | 常见场景 |
|:---|:---|:---|
| `for (int x : v)` | 范围 for，按值遍历 | 元素很小，只读即可 |
| `for (int& x : v)` | 范围 for，按引用遍历 | 需要修改元素 |
| `for (const int& x : v)` | 范围 for，只读引用 | 元素较大，避免拷贝 |
| `v.begin()` / `v.end()` | 正向迭代器范围 | 配合 STL 算法、手动遍历 |
| `v.cbegin()` / `v.cend()` | 只读正向迭代器范围 | 明确不允许修改元素 |
| `v.rbegin()` / `v.rend()` | 反向迭代器范围 | 倒序遍历 |

`begin()` 指向第一个元素，`end()` 指向**最后一个元素的下一个位置**。所以 `[begin, end)` 是左闭右开的范围，这也是 STL 里最常见的范围表示。

### 修改元素

| 操作 | 说明 | 注意 |
|:---|:---|:---|
| `v.push_back(x)` | 在末尾添加一个元素 | 可能触发扩容 |
| `v.emplace_back(args...)` | 在末尾原地构造元素 | 适合自定义类型 |
| `v.pop_back()` | 删除最后一个元素 | 不返回被删元素；空 vector 不能删 |
| `v.insert(pos, x)` | 在 `pos` 前插入元素 | 插入点及后面的迭代器可能失效 |
| `v.erase(pos)` | 删除 `pos` 指向的元素 | 返回删除后下一个有效迭代器 |
| `v.erase(first, last)` | 删除一个范围 | 常配合算法使用 |
| `v.clear()` | 清空所有元素 | 通常不释放容量 |
| `v.assign(n, value)` | 重新赋值为 `n` 个 `value` | 原来的元素会被替换 |
| `v.swap(other)` | 和另一个 vector 交换内容 | 常用于高效交换两个数组 |

### reserve 和 resize 的区别

`reserve` 是**只准备空间，不创建元素**；`resize` 是**改变元素个数，会真的创建或删除元素**。

```cpp
std::vector<int> a;
a.reserve(10);
// a.size() == 0，此时 a[0] 仍然不能用

std::vector<int> b;
b.resize(10);
// b.size() == 10，此时 b[0] 到 b[9] 都能用
```

这个区别非常重要：很多初学者以为 `reserve(10)` 之后就可以写 `v[0] = 1`，这是错的。

## 示例代码

### 示例 1：从空 vector 开始，添加、访问和遍历

```cpp
#include <cstddef>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> nums;

    nums.push_back(10);
    nums.push_back(20);
    nums.push_back(30);

    std::cout << "size = " << nums.size() << "\n";
    std::cout << "first = " << nums.front() << "\n";
    std::cout << "last = " << nums.back() << "\n";

    std::cout << "by index: ";
    for (std::size_t i = 0; i < nums.size(); ++i)
    {
        std::cout << nums[i] << " ";
    }
    std::cout << "\n";

    std::cout << "by range-for: ";
    for (int n : nums)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
size = 3
first = 10
last = 30
by index: 10 20 30
by range-for: 10 20 30
```

这个例子里，下标循环和范围 for 的结果一样。区别在于：

- 下标循环适合你需要元素位置 `i` 的场景。
- 范围 for 适合单纯遍历所有元素，代码更短，也更不容易写出越界。

### 示例 2：begin/end 是算法和容器之间的桥梁

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> nums = {10, 20, 30, 40, 50};

    std::cout << "forward: ";
    for (auto it = nums.begin(); it != nums.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << "\n";

    std::cout << "reverse: ";
    for (auto it = nums.rbegin(); it != nums.rend(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << "\n";

    auto it = std::find(nums.begin(), nums.end(), 30);
    if (it != nums.end())
    {
        std::cout << "found: " << *it << "\n";
    }

    return 0;
}
```

**运行结果**：

```
forward: 10 20 30 40 50
reverse: 50 40 30 20 10
found: 30
```

这里的关键点不是“手写迭代器比范围 for 更好”，而是 `begin()` / `end()` 是 STL 的通用接口。`std::find` 不需要知道 `nums` 是什么容器，它只需要一段 `[begin, end)` 范围。

### 示例 3：reserve 和 resize 看起来像，实际完全不同

```cpp
#include <cstddef>
#include <iostream>
#include <vector>

void print_info(const std::vector<int>& v, const char* name)
{
    std::cout << name
              << " size=" << v.size()
              << " capacity=" << v.capacity()
              << "\n";
}

int main()
{
    std::vector<int> a;
    a.reserve(3);
    print_info(a, "after reserve");

    a.push_back(10);
    a.push_back(20);
    print_info(a, "after push_back twice");

    std::vector<int> b;
    b.resize(3);
    print_info(b, "after resize");

    b[0] = 100;
    b[1] = 200;
    b[2] = 300;

    std::cout << "b: ";
    for (int n : b)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**可能的运行结果**：

```
after reserve size=0 capacity=3
after push_back twice size=2 capacity=3
after resize size=3 capacity=3
b: 100 200 300
```

`capacity` 的具体数值由标准库实现决定，可能和这里略有不同，但规律不变：

- `reserve(3)` 后 `size` 仍然是 0，不能访问 `a[0]`。
- `resize(3)` 后 `size` 是 3，`b[0]` 到 `b[2]` 都已经存在。

这个例子就像你提到的异步定时器：只放一两个元素时，`reserve` 和不 `reserve` 结果可能看不出差别；一旦大量插入，是否提前预留空间就会影响扩容次数、指针稳定性和性能。

### 示例 4：大量插入时，reserve 的区别才明显

```cpp
#include <cstddef>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> without_reserve;
    std::vector<int> with_reserve;

    with_reserve.reserve(1000);

    int grow1 = 0;
    int grow2 = 0;

    std::size_t last_cap1 = without_reserve.capacity();
    std::size_t last_cap2 = with_reserve.capacity();

    for (int i = 0; i < 1000; ++i)
    {
        without_reserve.push_back(i);
        if (without_reserve.capacity() != last_cap1)
        {
            ++grow1;
            last_cap1 = without_reserve.capacity();
        }

        with_reserve.push_back(i);
        if (with_reserve.capacity() != last_cap2)
        {
            ++grow2;
            last_cap2 = with_reserve.capacity();
        }
    }

    std::cout << "without reserve grow times: " << grow1 << "\n";
    std::cout << "with reserve grow times: " << grow2 << "\n";

    return 0;
}
```

**可能的运行结果**：

```
without reserve grow times: 11
with reserve grow times: 0
```

扩容并不是“多申请一点内存”这么简单。`vector` 扩容时通常会申请一块更大的连续内存，然后把旧元素移动或拷贝过去，所以：

- 如果提前知道大概会插入多少元素，用 `reserve` 可以减少扩容次数。
- 如果你保存了元素引用、指针、迭代器，扩容后它们可能失效。
- 如果元素很大，扩容时移动/拷贝成本会更明显。

### 示例 5：insert / erase 要用迭代器定位

```cpp
#include <iostream>
#include <vector>

void print(const std::vector<int>& v)
{
    for (int n : v)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";
}

int main()
{
    std::vector<int> nums = {10, 20, 30, 40};

    auto pos = nums.begin() + 2;  // 指向 30
    nums.insert(pos, 25);         // 在 30 前插入
    print(nums);

    for (auto it = nums.begin(); it != nums.end();)
    {
        if (*it < 30)
        {
            it = nums.erase(it);  // erase 返回下一个有效位置
        }
        else
        {
            ++it;
        }
    }
    print(nums);

    return 0;
}
```

**运行结果**：

```
10 20 25 30 40
30 40
```

删除元素时不要在 `erase` 之后继续使用旧迭代器。`erase` 会返回“删除位置后面的下一个元素”，循环里要接住它。

### 示例 6：存储自定义类型，理解 push_back 和 emplace_back 的区别

```cpp
#include <iostream>
#include <string>
#include <utility>
#include <vector>

struct Student
{
    std::string name;
    int score;

    Student(std::string name_, int score_)
        : name(std::move(name_)), score(score_)
    {
        std::cout << "construct " << name << "\n";
    }
};

int main()
{
    std::vector<Student> students;
    students.reserve(3);

    Student alice("Alice", 85);
    students.push_back(alice);             // 把已有对象拷贝进 vector
    students.push_back(Student("Bob", 92)); // 先创建临时对象，再移动进 vector
    students.emplace_back("Charlie", 78);  // 直接在 vector 内部构造

    for (const auto& s : students)
    {
        std::cout << s.name << ": " << s.score << "\n";
    }

    return 0;
}
```

**可能的运行结果**：

```
construct Alice
construct Bob
construct Charlie
Alice: 85
Bob: 92
Charlie: 78
```

这个例子里输出的构造次数不一定能完整展示拷贝/移动次数，因为编译器可能做优化。你要抓住用法差异：

- 已经有一个对象时，用 `push_back(obj)` 很自然。
- 临时创建一个对象再放进去时，可以用 `push_back(Student(...))`。
- 想直接把构造参数交给 vector，让元素在容器内部创建，用 `emplace_back(...)`。

### 示例 7：data() 用于和 C 风格接口配合

```cpp
#include <cstddef>
#include <iostream>
#include <vector>

void fill_from_c_api(int* data, std::size_t size)
{
    for (std::size_t i = 0; i < size; ++i)
    {
        data[i] = static_cast<int>(i * 10);
    }
}

int main()
{
    std::vector<int> buffer(5);

    fill_from_c_api(buffer.data(), buffer.size());

    for (int n : buffer)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
0 10 20 30 40
```

`vector` 的元素是连续存储的，所以 `data()` 可以把它交给一些需要 `T*` 的底层接口，比如 C 库、串口收发缓冲区、图像数据缓冲区等。但要注意：如果后面 `push_back` 触发扩容，之前拿到的 `data()` 指针也可能失效。

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 最基础的创建、添加、访问、遍历 | `push_back()`、`front()`、`back()`、范围 for | 先建立 vector 的数组直觉 | 空 vector 上不能调用 `front()` / `back()` |
| 示例 2 | 迭代器和算法 | `begin()`、`end()`、`rbegin()`、`rend()`、`std::find` | STL 算法通过迭代器操作范围 | `end()` 不是最后一个元素，不能解引用 |
| 示例 3 | 容量和元素个数 | `reserve()`、`resize()`、`capacity()` | 分清“预留空间”和“创建元素” | `reserve` 后不能用下标访问不存在的元素 |
| 示例 4 | 扩容成本 | 容量变化统计 | 让 `reserve` 的意义在大量插入时显现出来 | 容量增长策略由标准库实现决定 |
| 示例 5 | 插入和删除 | `insert()`、`erase()` | 用迭代器定位位置，并正确接住删除后的迭代器 | `erase` 后旧迭代器可能失效 |
| 示例 6 | 自定义类型 | `push_back()`、`emplace_back()`、`const auto&` | 区分已有对象、临时对象、原地构造 | `emplace_back` 不是永远更好，简单类型差别很小 |
| 示例 7 | 连续内存 | `data()` | 和 C API、底层缓冲区接口配合 | 扩容后旧指针可能失效 |

## 常用写法对比

### 下标遍历 vs 范围 for vs 迭代器

| 写法 | 优点 | 缺点 | 推荐场景 |
|:---|:---|:---|:---|
| `for (size_t i = 0; i < v.size(); ++i)` | 能拿到下标 | 容易写出越界或类型警告 | 需要位置编号 |
| `for (auto& x : v)` | 简洁，不容易越界 | 不直接知道下标 | 遍历和修改元素 |
| `for (auto it = v.begin(); it != v.end(); ++it)` | 能配合插入、删除、算法概念 | 代码稍长 | 需要迭代器位置 |

### push_back vs emplace_back

| 写法 | 含义 | 推荐场景 |
|:---|:---|:---|
| `v.push_back(obj)` | 把已有对象放进 vector | 对象已经存在 |
| `v.push_back(T(args...))` | 先创建临时对象，再放进 vector | 写法直观，适合简单场景 |
| `v.emplace_back(args...)` | 直接在 vector 内部构造对象 | 对象构造参数已经在手上，尤其是自定义类型 |

对于 `int`、`double` 这种简单类型，`push_back(1)` 和 `emplace_back(1)` 差别几乎可以忽略。不要为了“看起来高级”把所有 `push_back` 都改成 `emplace_back`。

### reserve vs resize

| 写法 | 改变 size 吗 | 改变 capacity 吗 | 元素能访问吗 |
|:---|:---|:---|:---|
| `reserve(n)` | 不改变 | 可能增加 | 新预留的位置不能访问 |
| `resize(n)` | 改变 | 可能增加 | 新创建的元素可以访问 |

## 迭代器失效规则

`vector` 的底层是连续内存，所以它的高性能来自连续存储，也因此会有迭代器失效问题。你不需要一开始背完整标准，先记住这张表：

| 操作 | 什么时候会失效 | 哪些东西会失效 |
|:---|:---|:---|
| `push_back` / `emplace_back` | 如果触发扩容 | 所有迭代器、引用、指针 |
| `push_back` / `emplace_back` | 如果没有触发扩容 | 通常只有 `end()` 失效 |
| `insert` | 如果触发扩容 | 所有迭代器、引用、指针 |
| `insert` | 如果没有触发扩容 | 插入点及其后面的迭代器、引用、指针 |
| `erase` | 删除元素后 | 被删位置及其后面的迭代器、引用、指针 |
| `clear` | 清空后 | 所有迭代器、引用、指针 |
| `reserve` | 如果真的重新分配了内存 | 所有迭代器、引用、指针 |

通俗一点说：只要 `vector` 可能搬家，你手里保存的“旧地址”就可能不可信；只要中间插入或删除，后面的元素位置就可能变化。

## 常见错误

**错误 1：访问空 vector 的元素**

```cpp
std::vector<int> v;
v[0] = 10;  // ❌ 未定义行为！v 是空的，没有第 0 个元素
```

正确做法：先用 `push_back()` 或 `resize()` 分配元素。

**错误 2：把 reserve 当 resize 用**

```cpp
std::vector<int> v;
v.reserve(10);
v[0] = 10;  // ❌ reserve 只是预留空间，没有创建第 0 个元素
```

正确做法：

```cpp
std::vector<int> v;
v.resize(10);
v[0] = 10;  // ✅ 第 0 个元素已经存在
```

或者：

```cpp
std::vector<int> v;
v.reserve(10);
v.push_back(10);  // ✅ 添加一个真实元素
```

**错误 3：扩容后继续使用旧迭代器、引用或指针**

```cpp
std::vector<int> v = {1, 2, 3};
auto it = v.begin();
v.push_back(4);  // 可能导致扩容，it 失效！
std::cout << *it; // ❌ 危险！
```

正确做法：添加元素后重新获取迭代器。

**错误 4：删除元素时没有接住 erase 返回值**

```cpp
std::vector<int> v = {1, 2, 3, 4};
for (auto it = v.begin(); it != v.end(); ++it)
{
    if (*it % 2 == 0)
    {
        v.erase(it);  // ❌ it 已经失效，for 循环还会 ++it
    }
}
```

正确做法：

```cpp
for (auto it = v.begin(); it != v.end();)
{
    if (*it % 2 == 0)
    {
        it = v.erase(it);
    }
    else
    {
        ++it;
    }
}
```

**错误 5：用 `v[i]` 添加元素**

```cpp
std::vector<int> v;
v[0] = 10;  // ❌ v 是空的，v[0] 不会自动创建元素
```

正确做法：用 `push_back()` 或先 `resize()`。

## 使用建议

1. 默认优先选 `std::vector`，除非你明确需要有序键值、频繁在头部插入、自动去重等能力。
2. 只读遍历大对象时，用 `const auto&`；需要修改时，用 `auto&`；简单数字按值也可以。
3. 提前知道元素数量时，用 `reserve` 减少扩容；需要真正创建固定个数元素时，用 `resize`。
4. 删除多个满足条件的元素时，优先考虑 `erase + remove_if`，这个会在 `algorithm` 小节详细讲。
5. 不要长期保存 `vector` 元素的指针、引用或迭代器，除非你能保证期间不会触发扩容或删除。

## 小结

`std::vector` 是现代 C++ 中最常用的容器。它适合存放数量会变化、又希望像数组一样快速随机访问的数据。

本节最重要的不是背 API，而是理解几组区别：

- `size` 是已有元素个数，`capacity` 是已申请空间。
- `reserve` 是预留空间，`resize` 是改变元素个数。
- `begin/end` 表示范围，STL 算法通过它们操作容器。
- `push_back` 放入对象，`emplace_back` 在容器内部构造对象。
- 插入、删除、扩容后，旧迭代器、引用、指针可能失效。
