---
title: "iterator"
---

## 本节解决什么问题

遍历 vector 可以用下标 `v[i]`，遍历 map 呢？map 没有下标（key 不是整数）。而且每种容器的内部结构不同，有没有一种统一的方式来遍历所有容器？

**迭代器**（iterator）就是答案。它提供了一种**统一的方式**访问容器中的元素，不管容器内部是什么结构——你可以用同样的 `begin()` / `end()` / `++` 语法遍历 vector、map、set、list 等等。

## 这个特性是什么

迭代器是一个对象，**指向容器中的某个位置**，行为类似指针：
- `*it`：获取迭代器指向的元素
- `++it`：移动到下一个元素
- `it == end`：判断是否到达末尾

## C++ 标准版本

C++98（C++11 起增强了 `begin`/`end` 自由函数，C++20 引入了 ranges）。

## 需要的头文件

```cpp
#include <iterator>  // 高级迭代器功能（基本使用不需要单独包含）
```

迭代器定义在各个容器的头文件中，使用容器时自动获得，不需要单独 `#include <iterator>`。

## 基本语法

```cpp
// 正向迭代器
容器::iterator it;           // 可修改元素的迭代器
容器::const_iterator it;     // 只读迭代器

// 获取迭代器
auto it = c.begin();         // 指向第一个元素
auto end = c.end();          // 指向最后一个元素之后（不是最后一个！）

// 反向迭代器
auto rit = c.rbegin();       // 指向最后一个元素
auto rend = c.rend();        // 指向第一个元素之前

// 遍历
for (auto it = c.begin(); it != c.end(); ++it) { ... }
```

## 常用用法

| 操作 | 说明 |
|:---|:---|
| `c.begin()` / `c.end()` | 正向迭代器范围 |
| `c.rbegin()` / `c.rend()` | 反向迭代器范围 |
| `*it` | 解引用，获取元素 |
| `++it` / `--it` | 移动到下一个 / 上一个元素 |
| `it->member` | 访问元素的成员（等价于 `(*it).member`） |
| `std::advance(it, n)` | 前进 n 步 |
| `std::next(it, n)` | 返回前进 n 步后的迭代器（不修改原迭代器） |
| `std::prev(it, n)` | 返回后退 n 步后的迭代器 |

## 示例代码

### 示例 1：用迭代器遍历 vector

```cpp
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> v = {10, 20, 30, 40, 50};

    // 用迭代器遍历
    std::cout << "forward: ";
    for (auto it = v.begin(); it != v.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
forward: 10 20 30 40 50 
```

### 示例 2：在示例 1 基础上，反向遍历和 const 迭代器

```cpp
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> v = {10, 20, 30, 40, 50};

    // 反向遍历
    std::cout << "reverse: ";
    for (auto rit = v.rbegin(); rit != v.rend(); ++rit)
    {
        std::cout << *rit << " ";
    }
    std::cout << "\n";

    // const 迭代器：只读
    std::vector<int>::const_iterator it = v.cbegin();
    // *it = 100;   // ❌ 编译错误！const 迭代器不能修改元素

    std::cout << "first element via const_iterator: " << *it << "\n";

    return 0;
}
```

**运行结果**：

```
reverse: 50 40 30 20 10 
first element via const_iterator: 10
```

### 示例 3：在示例 2 基础上，不同容器的迭代器使用方式相同

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <set>

int main()
{
    std::vector<int> v = {10, 20, 30};
    std::map<std::string, int> m = {{"Alice", 85}, {"Bob", 92}};
    std::set<int> s = {5, 2, 8};

    // 三种容器，同样的迭代器语法
    std::cout << "vector: ";
    for (auto it = v.begin(); it != v.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << "\n";

    std::cout << "map:    ";
    for (auto it = m.begin(); it != m.end(); ++it)
    {
        std::cout << it->first << ":" << it->second << " ";
    }
    std::cout << "\n";

    std::cout << "set:    ";
    for (auto it = s.begin(); it != s.end(); ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
vector: 10 20 30 
map:    Alice:85 Bob:92 
set:    2 5 8 
```

### 示例 4：用 `std::next` 和 `std::advance` 移动迭代器

```cpp
#include <iostream>
#include <vector>
#include <iterator>

int main()
{
    std::vector<int> v = {10, 20, 30, 40, 50, 60};

    auto it = v.begin();

    // std::next：返回新迭代器，不修改原迭代器
    auto it2 = std::next(it, 2);
    std::cout << "begin + 2 = " << *it2 << "\n";
    std::cout << "original it still at: " << *it << "\n";

    // std::advance：原地移动迭代器
    std::advance(it, 3);
    std::cout << "after advance 3: " << *it << "\n";

    // std::prev：返回前一个位置
    auto it3 = std::prev(v.end(), 1);
    std::cout << "last element via prev: " << *it3 << "\n";

    return 0;
}
```

**运行结果**：

```
begin + 2 = 30
original it still at: 10
after advance 3: 40
last element via prev: 60
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 基础正向遍历 | `begin()`、`end()`、`auto`、`++it` | 统一语法遍历所有容器 | `end()` 不是最后一个元素，是最后一个之后 |
| 示例 2 | 反向遍历和 const | `rbegin()`、`rend()`、`const_iterator` | 反向遍历用反向迭代器更方便 | 用 `const_iterator` 或 `cbegin()` 保证不修改元素 |
| 示例 3 | 不同容器统一语法 | `it->first`、`it->second` | map 的迭代器解引用是 `pair`，必须用 `->` 访问 | map 迭代器的 `*it` 是 `std::pair` |
| 示例 4 | 移动迭代器 | `std::next()`、`std::advance()`、`std::prev()` | `next` 不修改原迭代器，`advance` 会修改 | `advance` 对某些容器可能是 O(n) 操作 |

## 常见错误

**错误 1：解引用 end() 迭代器**

```cpp
auto it = v.end();
std::cout << *it;  // ❌ 未定义行为！
```

正确做法：`end()` 指向最后一个元素之后，不能解引用。

**错误 2：在遍历时增删元素导致迭代器失效**

```cpp
for (auto it = v.begin(); it != v.end(); ++it)
{
    v.erase(it);  // ❌ it 失效！++it 行为未定义
}
```

正确做法：用 `it = v.erase(it)`（erase 返回下一个有效迭代器）。

**错误 3：不同类型容器的迭代器互相赋值**

```cpp
std::vector<int>::iterator it = my_set.begin();  // ❌ 类型不兼容！
```

正确做法：每类容器的迭代器类型不同，用 `auto` 可以自动推导。

## 使用建议

1. **遍历优先用范围 for**：它底层就是迭代器，但语法更简洁。
2. **需要 "当前位置" 信息时用迭代器**：比如条件删除、插入指定位置。
3. **泛型代码中用迭代器**：写模板函数时，迭代器让代码与容器类型解耦。
4. **C++20 ranges 更强大**：`std::ranges::for_each(v, ...)` 等，但迭代器是基础。

## 小结

- 迭代器是对容器元素的抽象指针，提供统一的访问方式。
- `begin()` → 首元素，`end()` → 尾元素后，`rbegin()` → 尾元素，`rend()` → 首元素前。
- 用 `*it` 解引用，`++it` 前进，`it->member` 访问成员。
- `std::next/prev/advance` 移动迭代器，注意 `advance` 修改原迭代器。
- 理解迭代器是理解 STL 算法的基础（下一节）。
