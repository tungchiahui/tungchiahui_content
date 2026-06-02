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

- [10-1-1 std::vector](ch10-1-1-std-vector.md) — 动态数组（最常用的容器，首选）
- [10-1-2 std::array](ch10-1-2-std-array.md) — 固定大小数组（代替 C 数组）
- [10-1-3 std::string](ch10-1-3-std-string.md) — 字符串（代替 char[]）
- [10-1-4 std::map / std::unordered_map](ch10-1-4-std-map-unordered-map.md) — 键值对容器
- [10-1-5 std::set / std::unordered_set](ch10-1-5-std-set-unordered-set.md) — 集合容器（自动去重）
- [10-1-6 queue / stack / deque](ch10-1-6-queue-stack-deque.md) — 队列、栈、双端队列
- [10-1-7 iterator](ch10-1-7-iterator.md) — 迭代器（统一遍历方式）
- [10-1-8 algorithm](ch10-1-8-algorithm.md) — 算法库（排序、查找、变换等）

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

## 阅读顺序建议

**如果你是 STL 新手，建议按顺序阅读**：

1. 先学 `vector` 和 `string` — 最常用的两个容器
2. 再学 `array`、`map`、`set`、`queue/stack/deque`
3. 然后学 `iterator` — 理解前面所有容器的统一遍历方式
4. 最后学 `algorithm` — 用标准算法操作容器

## 小结

STL 是现代 C++ 的基石。掌握了容器 + 迭代器 + 算法，你就拥有了处理数据的"标准工具箱"。后续学习的智能指针、并发编程、以及 ROS2 / Boost.Asio 中的数据传递，都离不开 STL 的基础。

开始学习第一个容器 → [10-1-1 std::vector](ch10-1-1-std-vector.md)
