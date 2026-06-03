---
title: "STL 标准模板库"
---

## 本节解决什么问题

在 C 语言中，你需要自己实现常见的数据结构：动态数组（`realloc`）、链表、哈希表、排序算法等。这不仅开发效率低，而且容易写出 bug（内存泄漏、越界、性能差）。

STL（Standard Template Library，标准模板库）提供了**经过充分测试和优化的通用数据结构和算法**，你只需要包含对应的头文件就能直接使用，不用重复造轮子。

## 什么是 STL

STL 是 C++ 标准库的核心部分，主要由三大组件构成：

| 组件 | 说明 | 示例 |
|:---|:---|:---|
| **容器**（Containers） | 存储数据的数据结构 | `vector`, `map`, `set`, `queue` 等 |
| **算法**（Algorithms） | 操作数据的通用函数 | `sort`, `find`, `count`, `transform` 等 |
| **迭代器**（Iterators） | 连接容器和算法的桥梁 | `begin()`, `end()`, `++`, `*` 等 |

三者的关系可以用一句话概括：**通过迭代器，算法可以操作任何容器中的数据**。

这句话刚开始可能有点抽象，可以先看一个具体例子：

```cpp
std::vector<int> scores = {80, 95, 60, 70};
std::sort(scores.begin(), scores.end());
```

这里发生了三件事：

1. `scores` 是容器，负责保存数据。
2. `scores.begin()` / `scores.end()` 给出一段范围。
3. `std::sort` 是算法，只关心这段范围里的元素，不关心它来自哪个变量。

所以 STL 的学习重点不是背每个函数，而是建立这个模型：

```text
容器保存数据 -> 迭代器描述范围 -> 算法处理范围
```

后面你会反复看到 `[begin, end)` 这种写法。它表示左闭右开范围：包含 `begin` 指向的元素，不包含 `end` 指向的位置。因为 `end` 不是最后一个元素，而是最后一个元素后面的“哨兵位置”。

## C++ 标准版本

STL 从 C++98 起就是 C++ 标准库的一部分，每个新标准都在增强它（C++11 的移动语义、C++17 的并行算法、C++20 的 ranges 等）。

## 学完本节你将掌握

1. 用 `std::vector` 代替 C 数组（自动扩缩容）
2. 用 `std::array` 代替固定大小 C 数组（更安全）
3. 用 `std::string` 代替 `char[]`（自动管理内存）
4. 用 `std::map` / `std::unordered_map` 做键值存储
5. 用 `std::set` / `std::unordered_set` 做去重集合
6. 用 `queue` / `stack` / `deque` 处理 FIFO/LIFO 场景
7. 理解迭代器——统一遍历所有容器的方式
8. 使用 STL 算法——排序、查找、变换一行搞定

## 目录

- [10-1-1 std::vector](ch10-1-1-std-vector) — 动态数组（最常用的容器，首选）
- [10-1-2 std::array](ch10-1-2-std-array) — 固定大小数组（代替 C 数组）
- [10-1-3 std::string](ch10-1-3-std-string) — 字符串（代替 char[]）
- [10-1-4 std::map / std::unordered_map](ch10-1-4-std-map-unordered-map) — 键值对容器
- [10-1-5 std::set / std::unordered_set](ch10-1-5-std-set-unordered-set) — 集合容器（自动去重）
- [10-1-6 queue / stack / deque](ch10-1-6-queue-stack-deque) — 队列、栈、双端队列
- [10-1-7 iterator](ch10-1-7-iterator) — 迭代器（统一遍历方式）
- [10-1-8 algorithm](ch10-1-8-algorithm) — 算法库（排序、查找、变换等）

## 容器选择速查

| 场景 | 推荐容器 | 原因 |
|:---|:---|:---|
| 动态数组 | `std::vector` | 连续内存，随机访问快 |
| 固定大小数组 | `std::array` | 比 C 数组安全 |
| 键值查找（需要有序） | `std::map` | 红黑树，按键排序 |
| 键值查找（追求速度） | `std::unordered_map` | 哈希表，O(1) 查找 |
| 去重集合 | `std::set` / `std::unordered_set` | 自动去重 |
| 先进先出 | `std::queue` | FIFO |
| 先进后出 | `std::stack` | LIFO |
| 双端操作 + 随机访问 | `std::deque` | 两端都快 |

### 选择容器时先问这几个问题

1. **需要按下标快速访问吗？** 需要就优先看 `vector` / `array` / `deque`。
2. **数据个数会变化吗？** 会变化就用 `vector`；固定大小就用 `array`。
3. **需要用 key 找 value 吗？** 需要就看 `map` / `unordered_map`。
4. **只关心元素有没有出现过吗？** 用 `set` / `unordered_set`。
5. **只从一端进、另一端出吗？** 用 `queue`。
6. **只从同一端进出吗？** 用 `stack`。

初学阶段有一个非常实用的默认选择：**不知道选什么时，先考虑 `std::vector`**。它的连续内存、随机访问、缓存友好、生态兼容都很好。只有当你明确需要“自动排序”“按 key 查找”“头部频繁插入删除”“队列语义”时，再换其他容器。

## 从手写循环到 STL 算法

很多 STL 代码看起来很短，初学时反而会觉得“不如自己写循环直观”。下面用同一个需求渐进演示：找出分数大于等于 90 的人数。

### 写法 1：手写下标循环

```cpp
#include <cstddef>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> scores = {80, 95, 60, 100, 88};

    int count = 0;
    for (std::size_t i = 0; i < scores.size(); ++i)
    {
        if (scores[i] >= 90)
        {
            ++count;
        }
    }

    std::cout << count << "\n";

    return 0;
}
```

这个写法适合刚入门理解流程，也适合确实需要下标 `i` 的场景。但它把“遍历”和“统计条件”揉在了一起。

### 写法 2：范围 for

```cpp
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> scores = {80, 95, 60, 100, 88};

    int count = 0;
    for (int score : scores)
    {
        if (score >= 90)
        {
            ++count;
        }
    }

    std::cout << count << "\n";

    return 0;
}
```

范围 for 去掉了下标，意图更清楚：我只是遍历每个分数，不关心它在第几个位置。

### 写法 3：算法 + Lambda

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> scores = {80, 95, 60, 100, 88};

    int count = std::count_if(scores.begin(), scores.end(),
                              [](int score) { return score >= 90; });

    std::cout << count << "\n";

    return 0;
}
```

这一版最像现代 C++ 的写法：`count_if` 表达“统计满足条件的元素”，Lambda 表达“条件是什么”。当需求变复杂时，这种写法更容易维护。

三种写法都能得到同样结果，但它们表达的重点不同：

| 写法 | 重点 | 适合场景 |
|:---|:---|:---|
| 下标循环 | 控制每一步怎么走 | 需要下标、相邻元素比较、调试基础逻辑 |
| 范围 for | 遍历每个元素 | 简单读写每个元素 |
| STL 算法 | 表达“要做什么” | 排序、查找、统计、删除、变换 |

这就是本章后面“先讲用法，再讲例程，再讲区别”的原因：简单数据上看起来都差不多，真实工程里差别会越来越明显。

## 常见 STL 误区

**误区 1：`end()` 是最后一个元素**

`end()` 不是最后一个元素，不能解引用。最后一个元素通常可以用 `back()` 访问，或者用 `std::prev(v.end())` 得到它的迭代器。

**误区 2：算法都会改变容器大小**

大部分算法只移动、重排、修改范围里的元素，不直接改变容器大小。例如 `std::remove_if` 只是把要保留的元素挪到前面，真正删除还要调用容器自己的 `erase`。

**误区 3：`unordered_map` 一定比 `map` 好**

`unordered_map` 平均查找更快，但它不按 key 排序，输出顺序不稳定，还依赖哈希质量。需要稳定有序遍历时，用 `map` 更合适。

**误区 4：容器里保存指针就等于管理了对象生命周期**

`std::vector<T*>` 只保存指针值，不会自动 `delete` 指针指向的对象。需要表达所有权时，后面要学 `std::unique_ptr` / `std::shared_ptr`。

## 阅读顺序建议

**如果你是 STL 新手，建议按顺序阅读**：

1. 先学 `vector` 和 `string` — 最常用的两个容器
2. 再学 `array`、`map`、`set`、`queue/stack/deque`
3. 然后学 `iterator` — 理解前面所有容器的统一遍历方式
4. 最后学 `algorithm` — 用标准算法操作容器

## 小结

STL 是现代 C++ 的基石。掌握了容器 + 迭代器 + 算法，你就拥有了处理数据的"标准工具箱"。后续学习的智能指针、并发编程、以及 ROS2 / Boost.Asio 中的数据传递，都离不开 STL 的基础。

本大节你要重点建立三个观念：

- 容器负责保存数据，不同容器适合不同访问模式。
- 迭代器负责描述范围，`begin()` / `end()` 是 STL 的共同语言。
- 算法负责处理范围，优先用标准算法表达意图。

开始学习第一个容器 → [10-1-1 std::vector](ch10-1-1-std-vector)
