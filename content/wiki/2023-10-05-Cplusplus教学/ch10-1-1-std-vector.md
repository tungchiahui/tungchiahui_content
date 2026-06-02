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

| 操作 | 说明 |
|:---|:---|
| `v.push_back(x)` | 在末尾添加元素 |
| `v.pop_back()` | 删除末尾元素 |
| `v[i]` | 随机访问（不检查越界） |
| `v.at(i)` | 随机访问（检查越界，越界抛异常） |
| `v.size()` | 返回元素个数 |
| `v.empty()` | 判断是否为空 |
| `v.clear()` | 清空所有元素 |
| `v.front()` | 返回第一个元素 |
| `v.back()` | 返回最后一个元素 |
| `v.emplace_back(args...)` | 在末尾原地构造元素（比 push_back 高效） |

## 示例代码

### 示例 1：创建 vector、添加元素、遍历

```cpp
#include <iostream>
#include <vector>

int main()
{
    // 创建一个空的 vector
    std::vector<int> nums;

    // 往末尾添加元素
    nums.push_back(10);
    nums.push_back(20);
    nums.push_back(30);

    // 用下标访问
    std::cout << "nums[0] = " << nums[0] << "\n";
    std::cout << "nums[1] = " << nums[1] << "\n";
    std::cout << "nums[2] = " << nums[2] << "\n";
    std::cout << "size = " << nums.size() << "\n";

    return 0;
}
```

**运行结果**：

```
nums[0] = 10
nums[1] = 20
nums[2] = 30
size = 3
```

### 示例 2：在示例 1 基础上，用范围 for 遍历和列表初始化

```cpp
#include <iostream>
#include <vector>

int main()
{
    // 列表初始化（C++11）
    std::vector<int> nums = {10, 20, 30, 40, 50};

    // 用范围 for 遍历
    std::cout << "elements: ";
    for (int n : nums)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    // 删除最后一个元素
    nums.pop_back();
    std::cout << "after pop_back, size = " << nums.size() << "\n";

    return 0;
}
```

**运行结果**：

```
elements: 10 20 30 40 50 
after pop_back, size = 4
```

### 示例 3：在示例 2 基础上，用 at() 安全访问和预分配大小

```cpp
#include <iostream>
#include <vector>

int main()
{
    // 预分配 3 个元素，初始值为 0
    std::vector<int> nums(3, 0);

    // 用下标赋值
    nums[0] = 100;
    nums[1] = 200;
    nums[2] = 300;

    // at() 访问（会检查越界）
    std::cout << "nums.at(1) = " << nums.at(1) << "\n";

    // 尝试越界访问（取消注释会抛出 std::out_of_range 异常）
    // std::cout << nums.at(10) << "\n";

    // emplace_back 直接构造元素（比 push_back 少一次拷贝）
    nums.emplace_back(400);
    std::cout << "after emplace_back, size = " << nums.size() << "\n";

    return 0;
}
```

**运行结果**：

```
nums.at(1) = 200
after emplace_back, size = 4
```

### 示例 4：存储自定义类型（Student）

```cpp
#include <iostream>
#include <vector>
#include <string>

struct Student
{
    std::string name;
    int score;
};

int main()
{
    std::vector<Student> students;

    // 添加学生
    students.push_back({"Alice", 85});
    students.push_back({"Bob", 92});
    students.emplace_back("Charlie", 78);  // 原地构造

    // 遍历
    for (const auto& s : students)
    {
        std::cout << s.name << ": " << s.score << "\n";
    }

    return 0;
}
```

**运行结果**：

```
Alice: 85
Bob: 92
Charlie: 78
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 最基础的创建、添加、访问 | `push_back()`、`size()`、`operator[]` | 最简单的方式了解 vector 基本操作 | `v[i]` 不检查越界，需自己保证 i < size() |
| 示例 2 | 列表初始化和范围 for 遍历 | `{...}` 初始化、`for (int n : v)`、`pop_back()` | 列表初始化更简洁；范围 for 比下标循环更安全 | `pop_back()` 前要确保 vector 非空 |
| 示例 3 | 预分配、安全访问、原地构造 | `vector(n, val)`、`at()`、`emplace_back()` | 预分配避免多次扩容；`at()` 安全；`emplace_back` 高效 | `emplace_back` 参数直接传给构造函数 |
| 示例 4 | 存储自定义类型 | struct 放入 vector、`const auto&` 遍历 | 使用引用遍历避免拷贝大对象 | `const auto&` 表示只读引用 |

## 常见错误

**错误 1：访问空 vector 的元素**

```cpp
std::vector<int> v;
v[0] = 10;  // ❌ 未定义行为！v 是空的，没有第 0 个元素
```

正确做法：先用 `push_back()` 或 `resize()` 分配元素。

**错误 2：迭代器失效**

```cpp
std::vector<int> v = {1, 2, 3};
auto it = v.begin();
v.push_back(4);  // 可能导致扩容，it 失效！
std::cout << *it; // ❌ 危险！
```

正确做法：添加元素后重新获取迭代器。

**错误 3：用 `v[i]` 添加元素**

```cpp
std::vector<int> v;
v[0] = 10;  // ❌ v 是空的，v[0] 不会自动创建元素
```

正确做法：用 `push_back()` 或先 `resize()`。

## 使用建议

1. **首选 `vector`**：除非有明确理由，否则动态数组请用 `vector` 而非 `new[]`。
2. **知道大小时先 `reserve()`**：如果预先知道元素数量，调用 `reserve(n)` 预分配内存，避免反复扩容造成的拷贝开销。
3. **大对象用 `emplace_back`**：比 `push_back` 少一次拷贝/移动。
4. **遍历用范围 for 或 `const auto&`**：避免不必要的拷贝。

## 小结

- `std::vector` 是动态数组，支持随机访问，元素连续存储。
- 用 `push_back` / `emplace_back` 添加元素，用 `pop_back` 删除末尾。
- `v[i]` 不检查越界（快），`v.at(i)` 检查越界（安全）。
- 遍历优先用范围 for，大对象用 `const auto&` 避免拷贝。
