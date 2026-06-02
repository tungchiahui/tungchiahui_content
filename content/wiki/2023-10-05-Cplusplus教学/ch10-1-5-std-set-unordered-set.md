---
title: "std::set / std::unordered_set"
---

## 本节解决什么问题

你需要存储**不重复**的元素，并且能快速判断某个元素是否存在。如果用 vector，每次查找都要遍历，且需要手动去重。

`std::set` 和 `std::unordered_set` 解决了这个问题，自动保证元素唯一性，并提供快速查找。

## 这个特性是什么

- `std::set<T>`：有序集合，元素不重复，底层是红黑树。
- `std::unordered_set<T>`：无序集合，元素不重复，底层是哈希表（查找更快）。

你可以把它理解为"只有 key 没有 value 的 map"。

## C++ 标准版本

- `std::set`：C++98
- `std::unordered_set`：C++11

## 需要的头文件

```cpp
#include <set>              // for std::set
#include <unordered_set>    // for std::unordered_set
```

## 基本语法

```cpp
std::set<int> s;                          // 空集合
std::set<int> s = {1, 2, 3, 4, 5};       // 列表初始化
std::unordered_set<int> us = {1, 2, 3};   // 无序集合

s.insert(6);   // 插入元素
s.erase(3);    // 删除元素
s.count(2);    // 是否存在（返回 0 或 1）
s.find(2);     // 查找（返回迭代器，找不到返回 end()）
```

## 常用用法

| 操作 | 说明 |
|:---|:---|
| `s.insert(x)` | 插入元素（已存在则不做任何事） |
| `s.erase(x)` | 删除元素 |
| `s.find(x)` | 查找（返回迭代器，找不到返回 `s.end()`） |
| `s.count(x)` | 判断是否存在（返回 0 或 1） |
| `s.size()` | 元素个数 |
| `s.empty()` | 判断是否为空 |
| `s.contains(x)` | 判断是否存在（C++20） |

## 示例代码

### 示例 1：创建 set、插入和去重

```cpp
#include <iostream>
#include <set>

int main()
{
    std::set<int> s;

    // 插入元素（重复的会被忽略）
    s.insert(5);
    s.insert(2);
    s.insert(8);
    s.insert(2);  // 重复，不会插入
    s.insert(5);  // 重复，不会插入

    // 遍历（按键升序）
    std::cout << "set elements: ";
    for (int n : s)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";
    std::cout << "size = " << s.size() << "\n";

    return 0;
}
```

**运行结果**：

```
set elements: 2 5 8 
size = 3
```

### 示例 2：在示例 1 基础上，用 find 查找和 erase 删除

```cpp
#include <iostream>
#include <set>

int main()
{
    std::set<int> s = {10, 20, 30, 40, 50};

    // 查找元素
    auto it = s.find(30);
    if (it != s.end())
    {
        std::cout << "Found: " << *it << "\n";
    }

    // 查找不存在的元素
    if (s.find(99) == s.end())
    {
        std::cout << "99 not found\n";
    }

    // 删除元素
    s.erase(20);
    std::cout << "after erase 20: ";
    for (int n : s)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
Found: 30
99 not found
after erase 20: 10 30 40 50 
```

### 示例 3：在示例 2 基础上，用 count 判断存在、unordered_set 无序

```cpp
#include <iostream>
#include <set>
#include <unordered_set>

int main()
{
    std::set<int> ordered = {30, 10, 50, 20, 40};
    std::unordered_set<int> unordered = {30, 10, 50, 20, 40};

    // count() 判断是否存在
    std::cout << "ordered contains 30? " << (ordered.count(30) ? "yes" : "no") << "\n";
    std::cout << "ordered contains 99? " << (ordered.count(99) ? "yes" : "no") << "\n";

    // set：有序
    std::cout << "set (ordered):       ";
    for (int n : ordered)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    // unordered_set：无序
    std::cout << "unordered_set:       ";
    for (int n : unordered)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**（unordered_set 顺序可能不同）：

```
ordered contains 30? yes
ordered contains 99? no
set (ordered):       10 20 30 40 50 
unordered_set:       20 40 10 50 30 
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 基本插入和自动去重 | `insert()`、自动排序 | set 自动保持唯一且有序 | 重复插入不会报错，但不会增加元素 |
| 示例 2 | find 查找和 erase 删除 | `find()`、`erase()` | `find` 返回迭代器，要检查是否为 `end()` | 删除不存在的元素不会出错 |
| 示例 3 | count 判断存在、对比有序和无序 | `count()`、`unordered_set` | `count()` 只能返回 0 或 1，比 find 更简洁 | unordered_set 不排序 |

## 常见错误

**错误 1：想通过下标访问 set 元素**

```cpp
std::set<int> s = {1, 2, 3};
std::cout << s[0];  // ❌ set 不支持下标访问！
```

正确做法：用迭代器或范围 for，set 没有下标。

**错误 2：修改 set 中的元素**

```cpp
std::set<int> s = {1, 2, 3};
auto it = s.find(2);
*it = 5;  // ❌ 编译错误！set 中元素是 const 的
```

正确做法：删除旧的，插入新的。

**错误 3：set 遍历时同时插入删除**

```cpp
for (int n : s)
{
    s.erase(n);  // ❌ 迭代器失效！
}
```

正确做法：使用迭代器版本的 `erase(it++)`，或收集后统一删除。

## 使用建议

1. **需要有序用 `set`**，不需要排序追求速度用 `unordered_set`。
2. **判断存在用 `count()`**（C++20 起更推荐 `contains()`）。
3. **如果要统计频率用 `map`**，如果只需要"出现过没有"用 `set`。
4. **set 不支持下标访问和修改元素**，它和 vector 用法不一样。

## 小结

- `std::set` 自动去重且有序，`std::unordered_set` 自动去重无序但查找更快。
- 用 `insert()` 添加，用 `erase()` 删除，用 `count()` 或 `find()` 判断是否存在。
- set 没有下标，不能修改已有元素，遍历时不能同时删除。
- "去重 + 快速查找"是 set 的经典应用场景。
