---
title: "现代C++"
---

"现代 C++"（Modern C++）指的是自 C++11 标准以来，C++ 语言的一系列重要更新和改进。现代 C++ 的目标是让代码更加**简洁、安全、高效**，同时增强语言的表达能力。

C++11、C++14、C++17、C++20、C++23 以及未来的 C++26 标准，每个版本都加入了大量新特性。本章按照"由浅入深、从常用到进阶"的顺序，帮你系统地学习现代 C++ 的核心内容，为后续学习 ROS2、Boost.Asio、机器人项目打下扎实的 C++ 基础。

## 目录

### [10-1 STL 标准模板库](ch10-1-STL标准模板库.md)（大节概览）

- [10-1-1 std::vector](ch10-1-1-std-vector.md) — 动态数组
- [10-1-2 std::array](ch10-1-2-std-array.md) — 固定大小数组
- [10-1-3 std::string](ch10-1-3-std-string.md) — 字符串
- [10-1-4 std::map / std::unordered_map](ch10-1-4-std-map-unordered-map.md) — 键值对容器
- [10-1-5 std::set / std::unordered_set](ch10-1-5-std-set-unordered-set.md) — 集合容器
- [10-1-6 queue / stack / deque](ch10-1-6-queue-stack-deque.md) — 队列和栈
- [10-1-7 iterator](ch10-1-7-iterator.md) — 迭代器
- [10-1-8 algorithm](ch10-1-8-algorithm.md) — 算法库

### 10-2 ~ 10-22 现代 C++ 语法

- [10-2 auto](ch10-2-auto.md) — 自动类型推导
- [10-3 nullptr](ch10-3-nullptr.md) — 安全空指针
- [10-4 using](ch10-4-using.md) — 类型别名
- [10-5 enum class](ch10-5-enum-class.md) — 枚举类
- [10-6 范围 for 循环](ch10-6-范围for循环.md) — 简化容器遍历
- [10-7 结构化绑定](ch10-7-结构化绑定.md) — 解包元组和结构体
- [10-8 constexpr](ch10-8-constexpr.md) — 编译期常量与函数
- [10-9 RAII](ch10-9-RAII.md) — 资源获取即初始化
- [10-10 智能指针](ch10-10-智能指针.md) — unique_ptr / shared_ptr / weak_ptr
- [10-11 右值引用和移动语义](ch10-11-右值引用和移动语义.md) — 移动语义
- [10-12 Lambda 表达式](ch10-12-Lambda表达式.md) — 匿名函数
- [10-13 std::function](ch10-13-std-function.md) — 可调用对象包装器
- [10-14 std::bind](ch10-14-std-bind.md) — 参数绑定
- [10-15 std::optional](ch10-15-std-optional.md) — 可选值
- [10-16 std::variant](ch10-16-std-variant.md) — 多类型安全联合体
- [10-17 std::span](ch10-17-std-span.md) — 视图
- [10-18 std::chrono](ch10-18-std-chrono.md) — 时间库
- [10-19 std::format / std::print](ch10-19-std-format-print.md) — 格式化输出
- [10-20 并发编程](ch10-20-并发编程.md) — 线程与互斥量
- [10-21 std::filesystem](ch10-21-std-filesystem.md) — 文件系统
- [10-22 modules 简介](ch10-22-modules简介.md) — 模块

## 阅读建议

1. **按顺序阅读**：章节按由浅入深排列，建议按顺序学习。
2. **动手实践**：每个示例都可以直接复制编译运行，动手敲一遍效果最好。
3. **重点关注**：STL 容器、智能指针、Lambda、并发编程是后续学习 ROS2 和 Boost.Asio 的基础，请务必掌握。
4. **不要跳过大节**：STL 和并发编程是独立的"大节"，内容较多，但它们是现代 C++ 的核心。
