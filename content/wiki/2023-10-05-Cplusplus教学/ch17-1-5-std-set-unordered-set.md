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

## set 和 vector 去重的区别

`set` 适合“边插入边保持唯一”，`vector + sort + unique` 适合“先收集一批数据，最后统一去重”。

| 场景 | 推荐写法 | 原因 |
|:---|:---|:---|
| 每来一个 ID 就要判断是否见过 | `std::unordered_set` | 在线查询快 |
| 需要一直保持有序唯一 | `std::set` | 插入后自动有序 |
| 已经收集了一批数据，最后去重 | `vector + sort + unique` | 批量处理通常更直接 |
| 需要保留原始出现顺序并去重 | `vector + unordered_set` 辅助 | set 会改变顺序 |

### 示例 4：批量去重时，vector + sort + unique 也很合适

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> ids = {3, 1, 2, 3, 2, 5, 1};

    std::sort(ids.begin(), ids.end());
    ids.erase(std::unique(ids.begin(), ids.end()), ids.end());

    for (int id : ids)
    {
        std::cout << id << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
1 2 3 5
```

这个例子最后得到的是有序唯一结果。如果你只是想把一批数据去重后输出，`vector + sort + unique` 很清楚；如果你在数据到来的过程中就要不断判断“之前是否出现过”，用 `set` / `unordered_set` 更自然。

### 示例 5：保留首次出现顺序的去重

```cpp
#include <iostream>
#include <unordered_set>
#include <vector>

int main()
{
    std::vector<int> input = {3, 1, 2, 3, 2, 5, 1};
    std::vector<int> result;
    std::unordered_set<int> seen;

    for (int id : input)
    {
        if (seen.insert(id).second)
        {
            result.push_back(id);
        }
    }

    for (int id : result)
    {
        std::cout << id << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
3 1 2 5
```

这里 `unordered_set` 只负责判断是否见过，`vector` 负责保存原始出现顺序。容器不是只能单独用，很多工程写法都是组合使用。

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

**错误 4：以为 set 会保留插入顺序**

```cpp
std::set<int> s;
s.insert(3);
s.insert(1);
s.insert(2);
// 遍历结果是 1 2 3，不是 3 1 2
```

正确做法：需要保留插入顺序时，用 `vector` 保存顺序，再用 `unordered_set` 辅助去重。

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
