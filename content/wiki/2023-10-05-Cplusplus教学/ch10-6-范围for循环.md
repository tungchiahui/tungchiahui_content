---
title: "范围 for 循环"
---

## 本节解决什么问题

传统的 `for` 循环遍历容器需要写三要素（初始化、条件、递增），容易出错：下标越界、用错边界条件、类型不匹配等等。

范围 for 循环让你写"对容器中每个元素做..."的代码时，不再需要关心下标和迭代器细节，代码更简洁安全。

## 这个特性是什么

范围 for 循环（Range-based for loop）是 C++11 引入的语法糖，让你直接遍历容器（或任何有 `begin()` / `end()` 的类型）中的每个元素，编译器自动生成对应的迭代器代码。

## C++ 标准版本

C++11（基础），C++20 增加了初始化语句支持。

## 需要的头文件

不需要额外头文件。范围 for 是语言特性。但需要遍历的容器需要包含相应的头文件（如 `<vector>`）。

## 基本语法

```cpp
for (元素类型 变量名 : 容器)
{
    // 使用变量名
}

// 推荐：不修改元素用 const auto&
for (const auto& 变量名 : 容器)
{
    // 只读访问
}

// 要修改元素用 auto&
for (auto& 变量名 : 容器)
{
    // 可以修改元素
}
```

## 常用用法

| 用法 | 说明 | 何时使用 |
|:---|:---|:---|
| `for (auto x : c)` | 值拷贝遍历 | 元素类型小（int/double 等） |
| `for (const auto& x : c)` | 只读引用遍历 | **最常用**，不想拷贝大对象 |
| `for (auto& x : c)` | 可修改引用遍历 | 需要修改容器中元素 |
| `for (int x : {1, 2, 3})` | 遍历初始值列表 | 快速遍历已知小集合 |

## 示例代码

### 示例 1：用范围 for 遍历 vector

```cpp
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> v = {10, 20, 30, 40, 50};

    // 范围 for 遍历
    std::cout << "elements: ";
    for (int n : v)
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
elements: 10 20 30 40 50 
```

### 示例 2：在示例 1 基础上，const auto& 和 auto& 的区别

```cpp
#include <iostream>
#include <vector>
#include <string>

int main()
{
    std::vector<std::string> names = {"Alice", "Bob", "Charlie"};

    // const auto&：只读遍历，不拷贝（推荐）
    std::cout << "const auto&: ";
    for (const auto& s : names)
    {
        std::cout << s << " ";
        // s = "X";  // ❌ 编译错误！const 引用不能修改
    }
    std::cout << "\n";

    // auto&：可修改遍历
    for (auto& s : names)
    {
        s = s + "!";  // 修改原容器中的元素
    }
    std::cout << "after modify: ";
    for (const auto& s : names)
    {
        std::cout << s << " ";
    }
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
const auto&: Alice Bob Charlie 
after modify: Alice! Bob! Charlie! 
```

### 示例 3：在示例 2 基础上，遍历初始值列表和 map

```cpp
#include <iostream>
#include <map>
#include <string>

int main()
{
    // 直接遍历初始值列表
    std::cout << "init list: ";
    for (int n : {3, 1, 4, 1, 5, 9})
    {
        std::cout << n << " ";
    }
    std::cout << "\n";

    // 遍历 map
    std::map<std::string, int> scores = {
        {"Alice", 85},
        {"Bob", 92},
        {"Charlie", 78}
    };

    std::cout << "scores:\n";
    for (const auto& pair : scores)
    {
        std::cout << "  " << pair.first << ": " << pair.second << "\n";
    }

    return 0;
}
```

**运行结果**：

```
init list: 3 1 4 1 5 9 
scores:
  Alice: 85
  Bob: 92
  Charlie: 78
```

### 示例 4：在示例 3 基础上，C++17 结构化绑定遍历 map

```cpp
#include <iostream>
#include <map>
#include <string>

int main()
{
    std::map<std::string, int> scores = {
        {"Alice", 85},
        {"Bob", 92},
        {"Charlie", 78}
    };

    // C++17 结构化绑定：直接解包 key 和 value
    std::cout << "scores (structured binding):\n";
    for (const auto& [name, score] : scores)
    {
        std::cout << "  " << name << ": " << score << "\n";
    }

    return 0;
}
```

**运行结果**：

```
scores (structured binding):
  Alice: 85
  Bob: 92
  Charlie: 78
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 最基础的范围 for | `for (int n : v)` | 直接读懂意图："对 v 中每个元素 n" | 这种写法会拷贝每个 int，但 int 很小无所谓 |
| 示例 2 | const auto& vs auto& | `const auto&`、`auto&` | `const auto&` 避免拷贝；`auto&` 可修改元素 | 遍历 string 等大对象时一定要用引用 |
| 示例 3 | 遍历初始值列表和 map | 遍历 `{...}` 和 map | 初始值列表可以直接放在 for 里 | map 遍历出来的是 `std::pair` |
| 示例 4 | 结构化绑定解包 map | `for (const auto& [k, v] : m)` | C++17 语法，key/value 分别起名更清晰 | 如果不用 C++17，可以用示例 3 的方式 |

## 常见错误

**错误 1：在遍历过程中增删元素**

```cpp
for (auto x : v)
{
    v.push_back(x * 2);  // ❌ 可能导致迭代器失效、无限循环！
}
```

正确做法：先收集到新容器，循环结束后统一操作，或者用传统迭代器循环。

**错误 2：auto 不加引用，修改无效**

```cpp
for (auto s : names)
{
    s = "X";  // 修改的是拷贝！
}
// names 没有变化！
```

正确做法：要修改容器元素，用 `for (auto& s : names)`。

**错误 3：遍历时用下标访问另一个容器**

```cpp
for (int n : v1)
{
    v2[i++] = n;  // i 从哪来？
}
```

正确做法：需要下标时，传统 for 循环可能更合适，或者用枚举技巧。

## 使用建议

1. **不修改元素：用 `const auto&`** — 这是默认选择。
2. **修改元素：用 `auto&`** — 明确意图是修改。
3. **基本类型（int/double）且不修改：可以用 `auto`** — 拷贝开销可忽略。
4. **需要下标时老老实实用传统 for** — 范围 for 不适合所有场景。
5. **遍历 map 优先用 `[key, value]` 结构化绑定**（C++17 起）。

## 小结

- 范围 for 遍历容器的语法是 `for (const auto& x : container)`。
- `const auto&` 只读引用（默认选择），`auto&` 可修改引用，`auto` 值拷贝。
- 适用于任何有 `begin()` / `end()` 的类型（STL 容器、初始值列表等）。
- 遍历过程中不能增删容器元素，会导致迭代器失效。

### 工程拓展

在 ROS2 中，遍历传感器数据列表、遍历节点列表等场景频繁使用范围 for。在 Boost.Asio 中，也常用范围 for 遍历连接列表。掌握了 `const auto&` 这个惯用法，你能写出和优秀开源项目风格一致的代码。
