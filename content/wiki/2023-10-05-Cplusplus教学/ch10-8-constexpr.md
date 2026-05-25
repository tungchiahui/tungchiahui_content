---
title: "constexpr"
---

`constexpr`是从C++11开始引入的新特性，在后续的好几个C++版本做出了增强：

| 标准版本 | constexpr 相关支持 |
|:---|:---|
| C++11 | 支持 constexpr 变量和简单函数 |
| C++14 | constexpr 函数支持更多语句和循环 |
| C++17 | 支持 constexpr lambda，更多扩展 |
| C++20 | consteval 和更强的 constexpr |

`const` 和 `constexpr` 虽然都表示“不可变”，但本质和用途有明显区别，给你一份简明对比：

| 方面 | const | constexpr |
|:---|:---|:---|
| 含义 | 对象不可修改 | 表示值或函数可以在编译期求值 |
| 变量修饰 | 变量值不可变，但不保证编译期计算 | 变量必须是编译期常量 |
| 函数修饰 | 不用于函数（只能修饰成员函数） | 函数可在编译期执行 |
| 初始化要求 | 可以在运行时初始化 | 必须在编译时初始化 |
| 用途 | 防止变量被修改 | 做编译期计算，提高效率 |
| 示例 | const int x = 5; | constexpr int x = 5; |
| 函数示例 | int f() const;（成员函数常量） | constexpr int f() { return 5; } |

`const` 强调“值不可改”，运行时或编译时都可用

`constexpr` 强调“编译期计算”，只能初始化为编译时常量，函数可做编译期计算

```cpp
#include <iostream>

constexpr int square(int n) 
{
    return n * n;
}

int main() 
{
    const int a = 10;  // a 是只读变量，但可以是运行时初始化
    int x = 5;
    const int b = x;   // 合法，只要 x 在运行时确定

    constexpr int c = 10;      // c 必须编译期确定
    // constexpr int d = x;    // 错误，x 不是编译期常量

    int val = 3;
    const int r1 = square(val);       // OK，但 val 运行时确定，square 运行时计算
    constexpr int r2 = square(5);     // OK，编译期计算

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```
