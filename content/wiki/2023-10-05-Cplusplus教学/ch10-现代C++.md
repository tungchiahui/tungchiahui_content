---
title: "现代C++"
---

"现代 C++"（Modern C++）指的是自 C++11 标准以来，C++ 语言的一系列重要更新和改进。现代 C++ 的目标是让代码更加**简洁、安全、高效**，同时增强语言的表达能力。

C++11、C++14、C++17、C++20、C++23 以及未来的 C++26 标准，每个版本都加入了大量新特性。本章按照"由浅入深、从常用到进阶"的顺序，帮你系统地学习现代 C++ 的核心内容，为后续学习 ROS2、Boost.Asio、机器人项目打下扎实的 C++ 基础。

## 本章怎么学

现代 C++ 不应该按"背语法点"来学，而应该按"解决问题"来学。

这一章可以分成三条主线：

| 主线 | 你会遇到的问题 | 对应章节 |
|:---|:---|:---|
| 数据怎么放 | 动态数组、字符串、键值表、集合、队列、栈 | `vector`、`array`、`string`、`map`、`set`、`queue` |
| 代码怎么写得安全清晰 | 自动类型推导、空指针、别名、枚举、范围循环、资源管理 | `auto`、`nullptr`、`using`、`enum class`、范围 for、RAII、智能指针 |
| 复杂工程怎么组织 | 回调、异步、时间、文件、多线程、模块 | Lambda、`function`、`chrono`、`filesystem`、并发、modules |

学习时建议先看"用法"，再看"例程"，最后看"常见错误"。因为很多现代 C++ 特性单独看很简单，但放到工程里差别才明显：

- 一个 `vector` 只 `push_back` 几个 `int` 时，`reserve` 和不 `reserve` 看不出区别；大量插入或保存迭代器时，差别就很明显。
- 一个 Lambda 只做加法时，按值捕获和按引用捕获都能运行；放进异步回调或线程后，生命周期问题就会暴露。
- 一个线程任务顺序执行也能完成；两个耗时任务同时等待时，才能看出并发的意义。

所以本章例程会尽量采用"先给最小用法，再增加场景，再对比差异"的方式讲。不是为了把例子写复杂，而是让你看到不同写法在真实工程中的区别。

## 本章重点不是"新"，而是"更合适"

现代 C++ 不是把旧写法全部推翻，而是在更多场景下给你更合适的工具。

| 以前常见写法 | 现代 C++ 更推荐 | 为什么 |
|:---|:---|:---|
| C 数组 | `std::array` / `std::vector` | 自动知道大小，更容易配合算法 |
| `char[]` / `char*` | `std::string` | 自动管理内存，拼接、查找更方便 |
| `NULL` / `0` 表示空指针 | `nullptr` | 类型明确，避免重载歧义 |
| 手动 `new/delete` | RAII / 智能指针 | 异常、提前返回时也能释放资源 |
| 手写循环排序查找 | `<algorithm>` | 意图清晰，减少重复代码 |
| 函数指针回调 | Lambda / `std::function` | 能携带上下文，适合异步和事件回调 |
| 魔法数字状态 | `enum class` | 类型安全，避免不同枚举混用 |
| 手写时间单位换算 | `std::chrono` | 类型化时间单位，减少毫秒/秒混淆 |
| 平台相关文件 API | `std::filesystem` | 跨平台处理路径、目录、文件信息 |

学这些特性时，不要只问"语法是什么"，更要问：

1. 它解决了旧写法的什么问题？
2. 它有没有使用成本？
3. 在什么场景下差异才会明显？
4. 它和后面 ROS2、Boost.Asio、OpenCV、机器人驱动有什么关系？

## 目录

### [10-1 STL 标准模板库](ch10-1-stl-biao-zhun-mu-ban-ku)（大节概览）

- [10-1-1 std::vector](ch10-1-1-std-vector) — 动态数组，最常用容器
- [10-1-2 std::array](ch10-1-2-std-array) — 固定大小数组，代替 C 数组
- [10-1-3 std::string](ch10-1-3-std-string) — 字符串，代替 `char[]`
- [10-1-4 std::map / std::unordered_map](ch10-1-4-std-map-unordered-map) — 键值对容器
- [10-1-5 std::set / std::unordered_set](ch10-1-5-std-set-unordered-set) — 集合容器
- [10-1-6 queue / stack / deque](ch10-1-6-queue-stack-deque) — 队列和栈
- [10-1-7 iterator](ch10-1-7-iterator) — 迭代器，容器和算法之间的桥梁
- [10-1-8 algorithm](ch10-1-8-algorithm) — 算法库，排序、查找、删除、变换

### 10-2 ~ 10-22 现代 C++ 语法

- [10-2 auto](ch10-2-auto) — 自动类型推导
- [10-3 nullptr](ch10-3-nullptr) — 安全空指针
- [10-4 using](ch10-4-using) — 类型别名
- [10-5 enum class](ch10-5-enum-class) — 枚举类
- [10-6 范围 for 循环](ch10-6-fan-wei-for-xun-huan) — 简化容器遍历
- [10-7 结构化绑定](ch10-7-jie-gou-hua-bang-ding) — 解包元组和结构体
- [10-8 constexpr](ch10-8-constexpr) — 编译期常量与函数
- [10-9 RAII](ch10-9-raii) — 资源获取即初始化
- [10-10 智能指针](ch10-10-zhi-neng-zhi-zhen) — unique_ptr / shared_ptr / weak_ptr
- [10-11 右值引用和移动语义](ch10-11-you-zhi-yin-yong-he-yi-dong-yu-yi) — 移动语义
- [10-12 Lambda 表达式](ch10-12-lambda-biao-da-shi) — 匿名函数
- [10-13 std::function](ch10-13-std-function) — 可调用对象包装器
- [10-14 std::bind](ch10-14-std-bind) — 参数绑定
- [10-15 std::optional](ch10-15-std-optional) — 可选值
- [10-16 std::variant](ch10-16-std-variant) — 多类型安全联合体
- [10-17 std::span](ch10-17-std-span) — 视图
- [10-18 std::chrono](ch10-18-std-chrono) — 时间库
- [10-19 std::format / std::print](ch10-19-std-format-print) — 格式化输出
- [10-20 并发编程](ch10-20-bing-fa-bian-cheng) — 线程与互斥量
- [10-21 std::filesystem](ch10-21-std-filesystem) — 文件系统
- [10-22 modules 简介](ch10-22-modules-jian-jie) — 模块

## 推荐学习路线

### 第一阶段：先能写顺手

先学这些：

1. `std::vector`
2. `std::string`
3. `auto`
4. 范围 for 循环
5. `nullptr`

这一阶段的目标是：能不用 C 风格数组和裸指针写常见业务代码。

### 第二阶段：理解 STL 的统一模型

接着学：

1. `std::array`
2. `std::map` / `std::unordered_map`
3. `std::set` / `std::unordered_set`
4. `queue` / `stack` / `deque`
5. iterator
6. algorithm

这一阶段要理解一句话：**容器负责存数据，迭代器描述范围，算法负责处理范围**。

例如：

```cpp
std::sort(v.begin(), v.end());
```

这里 `v` 是容器，`v.begin()` 和 `v.end()` 给出范围，`sort` 只关心这个范围里的元素，不关心它们原来来自哪个容器。

这一阶段请特别注意三组区别：

| 对比 | 差异在哪里 | 典型章节 |
|:---|:---|:---|
| `size` vs `capacity` | 一个是元素个数，一个是底层容量 | `std::vector` |
| `reserve` vs `resize` | 一个只预留空间，一个真的创建元素 | `std::vector` |
| `map` vs `unordered_map` | 一个有序稳定，一个平均查找快但无序 | `map / unordered_map` |
| `set` vs `vector + sort + unique` | 一个自动维护集合，一个适合批量处理后去重 | `set`、`algorithm` |
| 迭代器遍历 vs 算法 | 一个描述怎么走，一个表达要做什么 | `iterator`、`algorithm` |

### 第三阶段：写出不容易出错的工程代码

再学：

1. `enum class`
2. `using`
3. 结构化绑定
4. `constexpr`
5. RAII
6. 智能指针
7. 右值引用和移动语义

这一阶段的目标是：减少资源泄漏、悬空指针、类型混乱、不必要拷贝等问题。

这一阶段请把"生命周期"放在第一位：

- `RAII` 关心资源什么时候释放。
- `unique_ptr` 关心谁唯一拥有对象。
- `shared_ptr` 关心谁共同拥有对象。
- `weak_ptr` 关心如何观察对象但不延长生命周期。
- 移动语义关心大对象如何转移资源，避免不必要拷贝。

如果只写几十行小程序，这些差异可能不明显；一旦对象跨函数、跨线程、跨回调、跨节点传递，生命周期就是工程稳定性的核心。

### 第四阶段：处理回调、异步和系统能力

最后学：

1. Lambda 表达式
2. `std::function`
3. `std::bind`
4. `std::optional`
5. `std::variant`
6. `std::span`
7. `std::chrono`
8. `std::format` / `std::print`
9. 并发编程
10. `std::filesystem`
11. modules

这一阶段会直接服务于 ROS2、Boost.Asio、OpenCV、PCL、机器人驱动封装等工程内容。

这一阶段请特别关注"回调上下文"：

| 场景 | 常用工具 | 为什么 |
|:---|:---|:---|
| 排序、查找时临时写条件 | Lambda | 简洁表达局部逻辑 |
| 把回调保存到容器里 | `std::function` | 统一不同可调用对象类型 |
| 适配旧接口或成员函数回调 | `std::bind` / Lambda | 改变参数顺序或绑定对象 |
| 函数可能失败但不是异常 | `std::optional` | 明确表达"可能没有值" |
| 一个变量可能是几种类型之一 | `std::variant` | 类型安全地替代不安全联合体 |
| 函数只借用一段连续数据 | `std::span` | 避免拷贝，也不拥有数据 |
| 等待、超时、计时 | `std::chrono` | 时间单位安全 |
| 多个耗时任务同时推进 | 并发编程 | 看出同步顺序执行和并发执行的差异 |

比如一个 Lambda 只在当前函数里调用，按值捕获和按引用捕获可能都没事；如果这个 Lambda 被保存为异步回调，按引用捕获局部变量就可能变成悬空引用。这就是本章例程会刻意加入"第二个任务""保存回调""跨作用域"这类场景的原因。

## 阅读建议

1. **先看用法表，再看例程**：用法表解决"有哪些工具"，例程解决"什么时候用哪个工具"。
2. **例程要真的运行**：很多差别只看代码看不出来，运行几次才能理解，比如线程输出顺序、迭代器失效、异常捕获。
3. **重点关注 STL、RAII、智能指针、Lambda、并发**：后面学习 ROS2、Boost.Asio、机器人驱动封装时会反复用到。
4. **不要只背结论**：比如 "`vector` 优先用 `reserve`" 不是永远正确，要看你是提前知道容量，还是要真正创建元素。
5. **把"生命周期"当主线**：对象什么时候创建、什么时候销毁、谁拥有它，是现代 C++ 最重要的思维方式。

## 实战练习建议

每学完一小节，建议不要只复制示例，而是做一个小改动：

1. 学 `vector`：把 5 个数据改成 1000 个，观察 `capacity` 变化。
2. 学 `map`：把 `map` 换成 `unordered_map`，观察输出顺序变化。
3. 学 `algorithm`：把 `find` 改成 `find_if`，用 Lambda 写条件。
4. 学智能指针：故意写一个循环引用，再用 `weak_ptr` 断开。
5. 学 Lambda：把按引用捕获改成按值捕获，观察变量变化差异。
6. 学并发：把两个 `sleep_for` 顺序执行，再放到两个线程里对比耗时。

这些练习的目的不是刷题，而是让你亲眼看到"为什么需要这种写法"。
