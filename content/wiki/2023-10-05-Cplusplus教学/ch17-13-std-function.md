---
title: "std::function"
---

## 本节解决什么问题

你有多种"可调用"的东西：普通函数、函数指针、lambda、函数对象（仿函数）、`std::bind` 的结果。但它们的类型各不相同，不能放进同一个容器，也不能用同一个变量接收。

`std::function` 提供了一个**统一的包装器**，可以存储任意可调用对象（只要签名匹配），让你像操作普通变量一样传递、存储、调用它们。

## 这个特性是什么

`std::function` 是一个类模板，**以函数签名为模板参数**，可以存储所有满足该签名的可调用对象。它是**通用的可调用对象包装器**。

## C++ 标准版本

C++11

## 需要的头文件

```cpp
#include <functional>
```

## 基本语法

```cpp
std::function<返回值类型(参数类型列表)> 变量名;

// 示例
std::function<int(int, int)> func;     // 可以存储 int(int, int) 签名的任何可调用对象
std::function<void()> callback;         // 可以存储 void() 签名的任何可调用对象
std::function<bool(int)> predicate;     // 可以存储 bool(int) 签名的任何可调用对象
```

## 什么可以放进 std::function

| 可调用类型 | 能否放入 std::function | 示例 |
|:---|:---|:---|
| 普通函数 | ✅ | `func = my_function;` |
| 函数指针 | ✅ | `func = &my_function;` |
| Lambda（无捕获） | ✅ | `func = [](int x){ return x*2; };` |
| Lambda（有捕获） | ✅ | `func = [n](int x){ return x + n; };` |
| 函数对象（仿函数） | ✅ | `func = MyFunctor();` |
| `std::bind` 结果 | ✅ | `func = std::bind(f, _1, 10);` |
| 成员函数（通过 bind） | ✅ | `func = std::bind(&C::m, &obj, _1);` |

## 示例代码

### 示例 1：std::function 统一存储不同类型的可调用对象

```cpp
#include <iostream>
#include <functional>

// 普通函数
int add(int a, int b)
{
    return a + b;
}

// 函数对象（仿函数）
struct Multiplier
{
    int factor;
    int operator()(int a, int b) const
    {
        return (a + b) * factor;
    }
};

int main()
{
    // 统一类型：std::function<int(int, int)>
    std::function<int(int, int)> func;

    // 存储普通函数
    func = add;
    std::cout << "add(3, 4) = " << func(3, 4) << "\n";

    // 存储 lambda
    func = [](int a, int b) { return a * b; };
    std::cout << "lambda(3, 4) = " << func(3, 4) << "\n";

    // 存储函数对象
    func = Multiplier{10};
    std::cout << "multiplier(3, 4) = " << func(3, 4) << "\n";

    return 0;
}
```

**运行结果**：

```
add(3, 4) = 7
lambda(3, 4) = 12
multiplier(3, 4) = 70
```

### 示例 2：在示例 1 基础上，std::function 作函数参数（回调）

```cpp
#include <iostream>
#include <functional>
#include <vector>

// 高阶函数：接收 std::function 回调
int calculate(int a, int b, std::function<int(int, int)> op)
{
    return op(a, b);
}

int main()
{
    int x = 10, y = 5;

    // 传入 lambda
    int r1 = calculate(x, y, [](int a, int b) { return a + b; });
    std::cout << "10 + 5 = " << r1 << "\n";

    // 传入另一个 lambda
    int r2 = calculate(x, y, [](int a, int b) { return a * a - b * b; });
    std::cout << "10^2 - 5^2 = " << r2 << "\n";

    // 传入有捕获的 lambda
    int offset = 100;
    int r3 = calculate(x, y, [offset](int a, int b) { return a + b + offset; });
    std::cout << "10 + 5 + 100 = " << r3 << "\n";

    return 0;
}
```

**运行结果**：

```
10 + 5 = 15
10^2 - 5^2 = 75
10 + 5 + 100 = 115
```

### 示例 3：在示例 2 基础上，存储多个回调到一个容器

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <string>

void print_int(int n)
{
    std::cout << "int: " << n << "\n";
}

int main()
{
    // 存储多个 void() 回调
    std::vector<std::function<void()>> tasks;

    int counter = 0;

    // 添加不同的可调用对象
    tasks.push_back([]() { std::cout << "Task 1: hello\n"; });
    tasks.push_back([&counter]() { ++counter; std::cout << "Task 2: counter = " << counter << "\n"; });
    tasks.push_back([&counter]() { counter += 10; std::cout << "Task 3: counter = " << counter << "\n"; });

    // 统一执行所有回调
    for (auto& task : tasks)
    {
        task();
    }

    std::cout << "final counter = " << counter << "\n";

    return 0;
}
```

**运行结果**：

```
Task 1: hello
Task 2: counter = 1
Task 3: counter = 11
final counter = 11
```

### 示例 4：在示例 3 基础上，用 std::function 存储成员函数

```cpp
#include <iostream>
#include <functional>
#include <string>

class Calculator
{
    int base_;
public:
    Calculator(int base) : base_(base) {}

    int add(int a, int b) const
    {
        return base_ + a + b;
    }
};

int main()
{
    Calculator calc(100);

    // 用 std::bind 将成员函数包装成 std::function
    std::function<int(int, int)> func =
        std::bind(&Calculator::add, &calc,
                  std::placeholders::_1, std::placeholders::_2);

    std::cout << "func(3, 4) = " << func(3, 4) << "\n";  // 100 + 3 + 4

    // 也可以预先绑定 base_=100，第一个参数也预先绑定
    std::function<int(int)> func2 =
        std::bind(&Calculator::add, &calc, 10, std::placeholders::_1);

    std::cout << "func2(5) = " << func2(5) << "\n";  // 100 + 10 + 5

    return 0;
}
```

**运行结果**：

```
func(3, 4) = 107
func2(5) = 115
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 统一包装不同类型 | `std::function<int(int,int)>` | 同一个变量可以存函数、lambda、函数对象 | 签名必须匹配 |
| 示例 2 | 作为函数参数（回调） | `std::function<int(int,int)> op` | 让函数接受任意可调用对象 | 参数类型比函数指针灵活得多 |
| 示例 3 | 存入容器 | `vector<std::function<void()>>` | 统一管理不同类型的回调 | 每个元素都可以是不同的实现 |
| 示例 4 | 存储成员函数 | `std::bind(&C::m, &obj, _1, _2)` | 成员函数需要 this 指针，bind 把 this 绑定进去 | 更多 bind 用法见下一节 |

## std::function vs 模板参数

`std::function` 的优势是能**存储**和**统一类型**；模板参数的优势是**零开销**，但不能方便地放进容器或成员变量。

| 场景 | 推荐 |
|:---|:---|
| 只是在函数里立刻调用一次回调 | 模板参数 |
| 类成员需要保存一个回调 | `std::function` |
| vector 里保存多个不同 lambda | `std::function` |
| 性能热点路径，高频调用小函数 | 模板参数优先 |

### 示例 5：只临时调用时，模板也可以接收回调

```cpp
#include <iostream>

template<typename Func>
void repeat(int times, Func func)
{
    for (int i = 0; i < times; ++i)
    {
        func(i);
    }
}

int main()
{
    repeat(3, [](int i) {
        std::cout << "i = " << i << "\n";
    });

    return 0;
}
```

**运行结果**：

```
i = 0
i = 1
i = 2
```

这个例子没有保存回调，只是马上调用，所以模板写法很合适。如果要把回调存到类成员里，或者放进 `vector`，就更适合 `std::function`。

## 常见错误

**错误 1：签名不匹配**

```cpp
std::function<int(int)> func = [](double x) { return x * 2; };  // ❌ 签名不匹配
```

正确做法：签名要完全一致：`std::function<int(double)> func = ...`。

**错误 2：std::function 为空时调用**

```cpp
std::function<int(int, int)> func;
func(3, 4);  // ❌ 运行时异常！func 为空
```

正确做法：调用前检查 `if (func) { func(3, 4); }`。

**错误 3：lambda 捕获后不能赋值给函数指针，但可以赋给 std::function**

```cpp
int n = 10;
void (*fp)(int) = [n](int x) { ... };       // ❌ 有捕获不能转函数指针
std::function<void(int)> f = [n](int x) { ... };  // ✅ 没问题
```

**错误 4：滥用 std::function 做参数**

```cpp
void func(std::function<void(int)> callback);  // 有开销
```

如果不需要存储可调用对象，只是临时使用，模板版本可能更好：

```cpp
template<typename F>
void func(F callback);  // 零开销
```

## 使用建议

1. **需要存储可调用对象（成员变量、容器）时用 `std::function`**：这种场景函数指针做不到。
2. **只是传入回调参数时，可以用模板而不是 `std::function`**：模板零开销。
3. **调用前检查是否为空**：`if (func)` 或 `if (func != nullptr)`。
4. **`std::function` 有轻微的性能开销**（类型擦除 + 可能的堆分配），大部分场景可忽略，但热点路径慎用。
5. **保存回调时注意捕获生命周期**：`std::function` 可能比被捕获的局部变量活得更久。

## 小结

- `std::function` 统一包装所有可调用对象：普通函数、函数指针、lambda、函数对象、bind 结果。
- 需要统一的回调接口、存储多种回调到容器时，`std::function` 是最佳选择。
- 成员函数需要通过 `std::bind` 包装后才能放入 `std::function`。
- 调用前记得检查 `std::function` 是否为空。
- 尽量在需要"存储"而非仅仅"传递"可调用对象时才用 `std::function`。
