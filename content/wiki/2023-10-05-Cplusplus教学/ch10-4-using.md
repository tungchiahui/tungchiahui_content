---
title: "using"
---

## 本节解决什么问题

C 语言中，用 `typedef` 给类型起别名。但 `typedef` 的语法很别扭，特别是给函数指针、模板起别名时，写法不直观、难读。

C++11 引入了 `using` 别名声明，语法更清晰，还支持**模板别名**（`typedef` 做不到）。

## 这个特性是什么

`using` 有两种常见用途：

1. **类型别名**：给已有类型起新名字，类似 `typedef` 但语法更清晰。
2. **模板别名**：给模板类型起别名（`typedef` 做不到这一点）。

## C++ 标准版本

C++11（类型别名语法），C++11 模板别名。

## 需要的头文件

不需要额外头文件。`using` 是语言关键字。

## 基本语法

```cpp
// 类型别名
using 新名字 = 已有类型;

// 模板别名
template<typename T>
using 新名字 = 模板类型<T...>;
```

## 与 typedef 的对比

| 场景 | typedef | using |
|:---|:---|:---|
| 基本类型 | `typedef int Integer;` | `using Integer = int;` |
| 函数指针 | `typedef void (*Func)(int);` | `using Func = void(*)(int);` |
| 数组 | `typedef int Arr10[10];` | `using Arr10 = int[10];` |
| 模板别名 | ❌ 不支持 | `using Vec<T> = std::vector<T>;` |

## 示例代码

### 示例 1：using 基础——替换 typedef

```cpp
#include <iostream>
#include <string>
#include <vector>

// 旧写法（typedef）
typedef std::vector<int> IntVector1;

// 新写法（using）
using IntVector2 = std::vector<int>;

int main()
{
    // 两种写法效果完全一样
    IntVector1 v1 = {1, 2, 3};
    IntVector2 v2 = {4, 5, 6};

    std::cout << "v1: ";
    for (int n : v1) std::cout << n << " ";
    std::cout << "\n";

    std::cout << "v2: ";
    for (int n : v2) std::cout << n << " ";
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
v1: 1 2 3 
v2: 4 5 6 
```

### 示例 2：在示例 1 基础上，using 给函数指针起别名

```cpp
#include <iostream>

// typedef 的函数指针写法（别扭）
typedef int (*FuncPtr1)(int, int);

// using 的函数指针写法（直观）
using FuncPtr2 = int (*)(int, int);

int add(int a, int b)
{
    return a + b;
}

int multiply(int a, int b)
{
    return a * b;
}

int main()
{
    FuncPtr2 op1 = add;
    FuncPtr2 op2 = multiply;

    std::cout << "add(3, 5) = " << op1(3, 5) << "\n";
    std::cout << "multiply(3, 5) = " << op2(3, 5) << "\n";

    return 0;
}
```

**运行结果**：

```
add(3, 5) = 8
multiply(3, 5) = 15
```

### 示例 3：在示例 2 基础上，模板别名（typedef 做不到）

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <string>

// 模板别名：定义一个泛型的 vector
template<typename T>
using Vec = std::vector<T>;

// 模板别名：键为 string 的 map
template<typename T>
using StringMap = std::map<std::string, T>;

int main()
{
    Vec<int> vi = {1, 2, 3};
    Vec<double> vd = {1.1, 2.2, 3.3};
    StringMap<int> scores = {{"Alice", 85}, {"Bob", 92}};

    std::cout << "Vec<int>: ";
    for (int n : vi) std::cout << n << " ";
    std::cout << "\n";

    std::cout << "Vec<double>: ";
    for (double d : vd) std::cout << d << " ";
    std::cout << "\n";

    std::cout << "StringMap<int>: ";
    for (const auto& p : scores)
        std::cout << p.first << ":" << p.second << " ";
    std::cout << "\n";

    return 0;
}
```

**运行结果**：

```
Vec<int>: 1 2 3 
Vec<double>: 1.1 2.2 3.3 
StringMap<int>: Alice:85 Bob:92 
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 基础类型别名 | `using Name = Type;` | 语法更直观，`=` 让人一眼看出"别名 = 原类型" | typedef 也能做，但 using 更清晰 |
| 示例 2 | 函数指针别名 | `using Func = int(*)(int,int);` | 名字在中间，比 typedef 的名字夹在中间更好读 | 实际项目中建议用 `std::function` 代替裸函数指针 |
| 示例 3 | 模板别名 | `template<T> using Vec = vector<T>;` | typedef 做不到模板别名，这是 using 的独特能力 | 非常适合简化嵌套模板类型 |

## 常见错误

**错误 1：把 using 别名的 `=` 方向写反**

```cpp
using int = MyInt;  // ❌ 方向反了！应该是 using 别名 = 原类型
```

正确做法：`using MyInt = int;`

**错误 2：在头文件中不加限制地使用 using**

```cpp
// 头文件中
using namespace std;  // ❌ 污染全局/外部命名空间！
```

正确做法：头文件中避免 `using namespace`，只在 .cpp 的实现文件或函数内部使用。

**错误 3：using 声明和 using 别名混淆**

```cpp
using std::cout;       // using 声明：引入名字
using MyVec = std::vector<int>;  // using 别名：创建别名
```

这是两种不同的用法，前者是引入名字到当前作用域，后者是创建类型别名。

## 使用建议

1. **用 `using` 代替 `typedef`**：语法更清晰，还支持模板别名。
2. **善用模板别名简化代码**：例如 `using TaskMap = std::map<int, std::function<void()>>;`。
3. **不要在头文件中 `using namespace`**：避免命名空间污染。
4. **using 声明（`using std::cout`）可以用在 .cpp 或函数内部**，简化代码。

## 小结

- `using NewName = OldType;` 比 `typedef` 语法更直观。
- `template<typename T> using Alias = ...;` 是 typedef 做不到的模板别名。
- 函数指针别名、容器类型别名在项目中很实用。
- 别混淆 `using` 别名和 `using` 声明（命名空间引入）。
