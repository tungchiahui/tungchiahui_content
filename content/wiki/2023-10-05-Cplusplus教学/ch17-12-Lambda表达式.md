---
title: "Lambda 表达式"
---

## 本节解决什么问题

有时候你需要一个小函数：给 `sort` 传自定义排序规则、给 `find_if` 传查找条件、给定时器设置回调。在传统 C++ 中，你只能：

1. 在远处定义一个普通函数（离开现场，代码不连贯）
2. 定义一个函数对象（写一个完整的类，太繁琐）

Lambda 表达式让你**在位定义**一个小函数，代码紧凑、逻辑连贯。

## 这个特性是什么

Lambda 表达式是**匿名函数**（严格来说是可调用的类对象），可以在需要的地方直接定义。语法简洁，支持捕获外部变量。

## C++ 标准版本

C++11（基础），C++14 增强了泛型 lambda 和初始化捕获，C++17 增加了 `constexpr` lambda 和 `*this` 捕获。

## 需要的头文件

不需要额外头文件。Lambda 是语言特性。

## 基本语法

```cpp
[捕获列表](参数列表) -> 返回值类型 { 函数体 }

// 简写形式（返回值由编译器推断）
[捕获列表](参数列表) { 函数体 }
```

## 捕获列表速查

| 语法 | 含义 | 说明 |
|:---|:---|:---|
| `[]` | 不捕获任何外部变量 | 只用参数和局部变量 |
| `[=]` | 按值捕获所有用到的外部变量 | 拷贝一份外部变量 |
| `[&]` | 按引用捕获所有用到的外部变量 | 外部对象要保证比 lambda 活得久 |
| `[x]` | 按值捕获指定变量 x | 只捕获用到的变量 |
| `[&x]` | 按引用捕获指定变量 x | 只捕获用到的变量 |
| `[x, &y]` | 按值捕获 x，按引用捕获 y | 混合捕获 |
| `[=, &x]` | 默认按值，x 按引用 | 细粒度控制 |
| `[this]` | 捕获当前对象指针 | 可访问成员变量 |
| `[*this]` | 捕获当前对象的副本（C++17） | 安全，异步友好 |

## 示例代码

### 示例 1：最简单的 lambda——两数相加

```cpp
#include <iostream>

int main()
{
    // 最简单的 lambda：不捕获，两个参数，返回和
    auto add = [](int a, int b) -> int {
        return a + b;
    };

    std::cout << "add(3, 5) = " << add(3, 5) << "\n";

    // 返回值类型可省略（编译器推断）
    auto multiply = [](int a, int b) {
        return a * b;
    };
    std::cout << "multiply(3, 5) = " << multiply(3, 5) << "\n";

    return 0;
}
```

**运行结果**：

```
add(3, 5) = 8
multiply(3, 5) = 15
```

### 示例 2：在示例 1 基础上，按值和按引用捕获外部变量

```cpp
#include <iostream>

int main()
{
    int x = 10;
    int y = 20;

    // 按值捕获：拷贝 x, y
    auto add_val = [x, y]() {
        // x = 100;  // ❌ 编译错误！按值捕获的变量默认是 const
        return x + y;
    };

    // 按引用捕获：可以修改外部变量
    auto add_ref = [&x, &y]() {
        x = 100;       // ✅ 修改了外部的 x
        return x + y;
    };

    std::cout << "add_val: " << add_val() << "\n";  // 10 + 20 = 30
    std::cout << "add_ref: " << add_ref() << "\n";  // 100 + 20 = 120
    std::cout << "x after add_ref: " << x << "\n";   // 100

    return 0;
}
```

**运行结果**：

```
add_val: 30
add_ref: 120
x after add_ref: 100
```

### 示例 3：在示例 2 基础上，lambda 配合 STL 算法

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main()
{
    std::vector<int> v = {5, 2, 8, 1, 9, 3};

    // 用 lambda 做降序排序
    std::sort(v.begin(), v.end(),
              [](int a, int b) { return a > b; });

    std::cout << "sorted (desc): ";
    for (int n : v) std::cout << n << " ";
    std::cout << "\n";

    // 用 lambda 做条件查找
    int threshold = 5;
    auto it = std::find_if(v.begin(), v.end(),
                           [threshold](int n) { return n < threshold; });

    if (it != v.end())
    {
        std::cout << "first element < " << threshold << " is " << *it << "\n";
    }

    // 用 lambda 做计数
    int cnt = std::count_if(v.begin(), v.end(),
                            [](int n) { return n % 2 == 0; });
    std::cout << "even count: " << cnt << "\n";

    return 0;
}
```

**运行结果**：

```
sorted (desc): 9 8 5 3 2 1 
first element < 5 is 3
even count: 2
```

### 示例 4：在示例 3 基础上，lambda 捕获 this 和 mutable

```cpp
#include <iostream>

class Counter
{
    int value = 0;

public:
    // 返回一个 lambda，每次调用 value 加 1
    auto make_incrementer()
    {
        // 按值捕获 *this（C++17），加 mutable 允许修改副本
        return [*this]() mutable {
            ++value;
            return value;
        };
    }

    // 按引用捕获 this：修改原对象
    auto make_incrementer_ref()
    {
        return [this]() {
            ++value;
            return value;
        };
    }

    int get_value() const { return value; }
};

int main()
{
    Counter c;

    // 按值捕获 *this：修改的是副本
    auto inc1 = c.make_incrementer();
    std::cout << "inc1(): " << inc1() << "\n";  // 1
    std::cout << "inc1(): " << inc1() << "\n";  // 2
    std::cout << "c.value = " << c.get_value() << " (unchanged)\n";

    // 按引用捕获 this：修改的是原对象
    auto inc2 = c.make_incrementer_ref();
    std::cout << "inc2(): " << inc2() << "\n";  // 1
    std::cout << "inc2(): " << inc2() << "\n";  // 2
    std::cout << "c.value = " << c.get_value() << " (modified)\n";

    return 0;
}
```

**运行结果**：

```
inc1(): 1
inc1(): 2
c.value = 0 (unchanged)
inc2(): 1
inc2(): 2
c.value = 2 (modified)
```

### 示例 5：泛型 lambda（C++14）

```cpp
#include <iostream>
#include <string>

int main()
{
    // auto 参数：泛型 lambda（C++14）
    auto add = [](auto a, auto b) {
        return a + b;
    };

    std::cout << "add(1, 7) = " << add(1, 7) << "\n";           // int + int
    std::cout << "add(1.1, 2.2) = " << add(1.1, 2.2) << "\n";   // double + double
    std::cout << "add(string, string) = "
              << add(std::string("hi "), std::string("world")) << "\n";

    return 0;
}
```

**运行结果**：

```
add(1, 7) = 8
add(1.1, 2.2) = 3.3
add(string, string) = hi world
```

### 示例 6：同步调用看不出问题，保存回调后引用捕获才危险

```cpp
#include <functional>
#include <iostream>
#include <string>
#include <vector>

int main()
{
    std::vector<std::function<void()>> callbacks;

    {
        std::string name = "Alice";

        auto print_now = [&name]() {
            std::cout << "sync: " << name << "\n";
        };
        print_now();  // 立刻调用，name 还活着，所以没问题

        callbacks.push_back([name]() {
            std::cout << "saved by value: " << name << "\n";
        });

        // callbacks.push_back([&name]() {
        //     std::cout << "saved by ref: " << name << "\n";
        // });  // ❌ 离开作用域后 name 销毁，之后再调用会悬空
    }

    for (const auto& cb : callbacks)
    {
        cb();
    }

    return 0;
}
```

**运行结果**：

```
sync: Alice
saved by value: Alice
```

这个例子是 Lambda 生命周期最重要的场景：

- 立刻同步调用时，引用捕获通常看起来没问题。
- 保存到 `std::function`、线程、定时器、异步回调里以后，Lambda 可能比局部变量活得更久。
- 异步或保存回调时，优先按值捕获需要的数据，或者用智能指针管理生命周期。

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 最简 lambda | `[](int a, int b) -> int {...}` | lambda 可以像"字面量函数"一样随处定义 | 返回值类型 `-> int` 通常可省略 |
| 示例 2 | 按值和按引用捕获 | `[x, y]`、`[&x, &y]` | 按值捕获默认 const，按引用可修改外部变量 | 按引用捕获要保证外部变量比 lambda 活得久 |
| 示例 3 | lambda + STL 算法 | sort/find_if/count_if 的第三个参数 | lambda 是 STL 算法的最佳搭档 | 捕获的变量在 lambda 体内可直接使用 |
| 示例 4 | this 捕获和 mutable | `[this]`、`[*this]`、`mutable` | `[*this]` 拷贝整个对象，异步安全 | mutable 让按值捕获的变量可修改（但不影响外部） |
| 示例 5 | 泛型 lambda | `[](auto a, auto b) {...}` | C++14 起参数类型可以是 auto | 泛型 lambda 底层是模板 |
| 示例 6 | 保存回调的生命周期 | `std::function`、按值捕获 | 回调可能晚于局部变量执行 | 异步/保存回调不要随便引用捕获局部变量 |

## 常见错误

**错误 1：按引用捕获的变量在 lambda 执行时已销毁**

```cpp
auto make_bad_lambda()
{
    int local = 42;
    return [&local]() { return local; };  // ❌ local 已销毁，悬挂引用！
}
```

正确做法：用按值捕获 `[local]` 或 `[=]`。

**错误 2：lambda 不能转为函数指针（如果捕获了变量）**

```cpp
void (*func)() = []() {};              // ✅ 无捕获可以
void (*func2)() = [x]() { return x; };  // ❌ 有捕获不能转为函数指针
```

正确做法：用 `std::function` 保存捕获变量的 lambda。

**错误 3：忘记 `mutable` 想修改按值捕获的变量**

```cpp
int x = 10;
auto f = [x]() { x++; };  // ❌ 编译错误！按值捕获的变量是 const
```

正确做法：`auto f = [x]() mutable { x++; };`

**错误 4：异步回调里默认 `[&]` 捕获**

```cpp
void start()
{
    std::string msg = "hello";
    register_callback([&] {
        std::cout << msg;  // ❌ callback 晚点执行时 msg 可能已经销毁
    });
}
```

正确做法：明确按值捕获 `[msg]`，或者把数据放到 `std::shared_ptr` 中延长生命周期。

## 使用建议

1. **能用 lambda 的地方就不要写远处的函数**：代码更紧凑、意图更清晰。
2. **STL 算法 + lambda 是绝配**：`sort`、`find_if`、`count_if`、`transform` 等。
3. **默认用按值捕获 `[=]` 还是 `[&]`？**：都不推荐！明确列出要捕获的变量（如 `[x, &y]`）能让意图更清晰。
4. **异步场景用 `[*this]`**：避免 this 悬挂。
5. **lambda 本质是可调用对象**：理解这一点对接下来的 `std::function` 很重要。

## 小结

- Lambda 是匿名函数，可在需要的地方直接定义：`[捕获](参数) { 函数体 }`。
- `[]` 不捕获，`[=]` 按值，`[&]` 按引用，可混合使用。
- Lambda + STL 算法是非常强大的组合。
- 注意引用捕获的生命周期问题，异步场景优先用按值捕获。
- C++14 起支持泛型 lambda（`auto` 参数）。
