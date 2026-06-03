---
title: "std::bind"
---

## 本节解决什么问题

你准备好了函数，但参数顺序不对、或者某个参数想提前固定，而这个函数是别人写的你不能改。

`std::bind` 可以**绑定部分参数、调整参数顺序**，生成新的可调用对象，而不用修改原函数。

## 这个特性是什么

`std::bind` 是一个函数适配器，它接受一个可调用对象和一组参数，返回一个新的可调用对象。你可以：

1. **提前绑定（固定）部分参数**。
2. **用占位符 `_1, _2, ...` 调整参数顺序**。
3. **绑定成员函数**（需要提供对象指针）。

## C++ 标准版本

C++11

> **注意**：在 C++14+ 中，很多时候 lambda 比 `std::bind` 更清晰、更推荐。但 `std::bind` 在某些场景（如适配成员函数给旧接口）仍然有用，理解它有助于阅读旧代码。

## 需要的头文件

```cpp
#include <functional>
```

## 基本语法

```cpp
auto new_func = std::bind(可调用对象, 参数1, 参数2, ...);

// 占位符：表示"调用时传入的第 N 个参数"
using namespace std::placeholders;  // 引入 _1, _2, _3...
// 或者用 std::placeholders::_1, std::placeholders::_2
```

## 常用用法

| 用法 | 说明 | 示例 |
|:---|:---|:---|
| 绑定普通函数参数 | `std::bind(f, 固定值, _1, _2)` | 固定第一个参数 |
| 调整参数顺序 | `std::bind(f, _2, _1)` | 参数 1 和 2 交换 |
| 绑定成员函数 | `std::bind(&C::method, &obj, _1, _2)` | 成员函数需要对象指针 |
| 和 `std::function` 配合 | `std::function<T(T)> f = std::bind(...)` | 统一接口 |

## 示例代码

### 示例 1：绑定普通函数的部分参数

```cpp
#include <iostream>
#include <functional>
#include <string>

void print_info(const std::string& name, int age, double score)
{
    std::cout << name << " | age: " << age << " | score: " << score << "\n";
}

int main()
{
    using namespace std::placeholders;

    // 原始函数：需要 3 个参数
    print_info("Alice", 20, 95.5);

    // bind：固定 name="Bob"，调用时只需传 age 和 score
    auto bob_info = std::bind(print_info, "Bob", _1, _2);
    bob_info(22, 88.0);   // Bob | age: 22 | score: 88

    // bind：固定 name 和 age，调用时只需传 score
    auto charlie_info = std::bind(print_info, "Charlie", 25, _1);
    charlie_info(92.5);   // Charlie | age: 25 | score: 92.5

    return 0;
}
```

**运行结果**：

```
Alice | age: 20 | score: 95.5
Bob | age: 22 | score: 88
Charlie | age: 25 | score: 92.5
```

### 示例 2：在示例 1 基础上，调整参数顺序

```cpp
#include <iostream>
#include <functional>

void divide(int a, int b)
{
    std::cout << a << " / " << b << " = " << (a / b) << "\n";
}

int main()
{
    using namespace std::placeholders;

    // 正常顺序
    divide(10, 2);  // 10 / 2 = 5

    // 交换参数顺序
    auto swapped = std::bind(divide, _2, _1);
    swapped(2, 10);  // 相当于 divide(10, 2) = 5

    // 固定第二个参数
    auto half = std::bind(divide, _1, 2);
    half(10);  // 10 / 2 = 5
    half(20);  // 20 / 2 = 10

    return 0;
}
```

**运行结果**：

```
10 / 2 = 5
10 / 2 = 5
10 / 2 = 5
20 / 2 = 10
```

### 示例 3：在示例 2 基础上，绑定成员函数

```cpp
#include <iostream>
#include <functional>
#include <memory>

class Calculator
{
    int base_;
public:
    Calculator(int base) : base_(base) {}

    int add(int a, int b)
    {
        return base_ + a + b;
    }
};

int main()
{
    using namespace std::placeholders;

    Calculator calc(10);

    // 绑定成员函数：第一个参数是成员函数指针，第二个参数是对象指针
    auto bound = std::bind(&Calculator::add, &calc, _1, _2);
    std::cout << "bound(3, 4) = " << bound(3, 4) << "\n";  // 10 + 3 + 4 = 17

    // 也可以固定更多参数
    auto bound2 = std::bind(&Calculator::add, &calc, 5, _1);
    std::cout << "bound2(6) = " << bound2(6) << "\n";      // 10 + 5 + 6 = 21

    // 也可以传 std::shared_ptr
    auto calc_ptr = std::make_shared<Calculator>(100);
    auto bound3 = std::bind(&Calculator::add, calc_ptr, _1, _2);
    std::cout << "bound3(7, 8) = " << bound3(7, 8) << "\n"; // 100 + 7 + 8 = 115

    return 0;
}
```

**运行结果**：

```
bound(3, 4) = 17
bound2(6) = 21
bound3(7, 8) = 115
```

### 示例 4：在示例 3 基础上，bind 配合 std::function 用于回调

```cpp
#include <iostream>
#include <functional>
#include <string>

class Button
{
    std::function<void()> on_click_;
public:
    void set_on_click(std::function<void()> callback)
    {
        on_click_ = callback;
    }

    void click()
    {
        if (on_click_)
        {
            std::cout << "Button clicked! ";
            on_click_();
        }
    }
};

class App
{
    std::string name_;
public:
    App(const std::string& name) : name_(name) {}

    void handle_click(const std::string& msg)
    {
        std::cout << "App " << name_ << " says: " << msg << "\n";
    }
};

int main()
{
    using namespace std::placeholders;

    Button btn;
    App app("MyApp");

    // 将成员函数 handle_click 绑定 msg="Hello"，变成 void() 签名
    btn.set_on_click(std::bind(&App::handle_click, &app, "Hello"));

    btn.click();  // Button clicked! App MyApp says: Hello

    return 0;
}
```

**运行结果**：

```
Button clicked! App MyApp says: Hello
```

## 运行结果

见上方每个示例的"运行结果"。

## 示例中的关键语法解释

| 示例 | 讲了什么 | 新出现的语法 | 为什么这样写 | 注意事项 |
|:---|:---|:---|:---|:---|
| 示例 1 | 绑定普通函数参数 | `std::bind(f, val, _1, _2)` | 固定不需要变的参数，减少调用时的参数个数 | `_1` 表示调用时传入的第 1 个参数 |
| 示例 2 | 调整参数顺序 | `std::bind(f, _2, _1)` | 不修改原函数就能交换参数 | `_2` 和 `_1` 的顺序确定了传参顺序 |
| 示例 3 | 绑定成员函数 | `std::bind(&C::m, &obj, _1, _2)` | 成员函数需要对象指针（或 shared_ptr），bind 把它"额外"的参数先填上 | 对象存活时间要覆盖 bind 返回的可调用对象的使用时间 |
| 示例 4 | bind + function 回调 | `set_on_click(std::bind(...))` | 成员函数签名不匹配回调签名，bind 适配 | bind 缩减/调整参数个数让签名匹配 |

## 常见错误

**错误 1：忘记 `using namespace std::placeholders`**

```cpp
auto f = std::bind(add, _1, 10);  // ❌ _1 未定义！
```

正确做法：加 `using namespace std::placeholders;` 或用 `std::placeholders::_1`。

**错误 2：以为 bind 默认按引用保存参数**

```cpp
int value = 10;
auto f = std::bind([](int& x) { x = 100; }, value);
// f();  // ❌ value 被拷贝进 bind，不能绑定到 int&
```

正确做法：需要引用时用 `std::ref`：

```cpp
int value = 10;
auto f = std::bind([](int& x) { x = 100; }, std::ref(value));
f();
std::cout << value << "\n";  // 100
```

**错误 3：成员函数绑定后传参数量不对**

```cpp
class C { public: void f(int a, int b); };
C obj;
auto bound = std::bind(&C::f, &obj);  // 没留占位符
bound(1, 2);  // ❌ 编译错误！
```

正确做法：`std::bind(&C::f, &obj, _1, _2)`。

**错误 4：绑定后的对象比原对象活得久**

```cpp
auto make_bound()
{
    C obj;
    return std::bind(&C::f, &obj, _1);  // ❌ obj 是局部变量，但 bind 保存了它的地址！
}
```

正确做法：用智能指针：`auto ptr = std::make_shared<C>(); return std::bind(&C::f, ptr, _1);`

## 使用建议

1. **优先用 lambda，其次才考虑 bind**：lambda 通常更清晰、编译更快。
   ```cpp
   // std::bind
   auto f1 = std::bind(add, 10, _1, _2);
   // lambda：更直观
   auto f2 = [](int x, int y) { return add(10, x, y); };
   ```

2. **bind 在适配成员函数给旧接口时很有用**：ROS2 的回调注册就是典型场景。

3. **用 `std::placeholders::_1` 而不是 `_1`**（或 `using namespace`）：避免名字污染。

4. **了解 bind 有助于阅读旧代码**：很多旧项目大量使用 bind。
5. **bind 默认拷贝参数**：需要引用语义时用 `std::ref` / `std::cref`。

## 小结

- `std::bind` 提前绑定部分参数，生成新的可调用对象。
- 占位符 `_1, _2, ...` 表示调用时传入的参数位置。
- 可绑定普通函数、成员函数、函数对象。
- C++14+ 建议优先用 lambda 代替 bind（更清晰）。
- bind 在适配成员函数给已有回调接口时仍然很有用。
