---
title: "Lambda 表达式"
---

## 定义
Lambda表达式，很多人把他叫做 **匿名内联函数** ，但是实际上他并不是一个函数，而是一个无名的非联合非聚合的类的类型。

Lambda 表达式会让代码更加的简洁，使得 C++ 支持匿名函数对象，可以在函数内定义小的函数并将其传递给其他函数。

ROS2的一些库使用了该表达式,所以需要学习一下。特别是在回调函数中常用。

```cpp
auto add = [](int a, int b)  -> int
{ 
    return a + b; 
};
std::cout << add(2, 3) << std::endl; // 输出 5
```

核心语法 `[捕获列表] (参数) -> 返回值 { 函数体 }`

*   `[]` 捕获列表：决定 lambda 如何拿外部变量

*   `(参数)` 参数列表，和普通函数一样

*   `->` 返回类型可省略（编译器推断）

*   `{}` 函数体

| Lambda 捕获规则一览（最常用的 90 % 都在这里） |
|:---|
| 语法 | 含义 | 典型场景 | 注意点 |
| [] | 不捕获任何外部变量 | 纯算法、纯回调不需要用到外围数据 | 只用参数和局部变量 |
| [=] | 按值捕获所有在 lambda 里用到的外部变量 | 把外部快照拷一份进来 | 捕获的是拷贝，修改不会影响外部 |
| [&] | 按引用捕获所有在 lambda 里用到的外部变量 | 需要回调里修改外部状态 | 外部对象要保证比 lambda 活得久 |
| [x] | 按值捕获指定变量 x | 只需用到少量变量 | 一旦拷贝，外部后续变化对 lambda 不可见 |
| [&x] | 按引用捕获指定变量 x | 只想改某个变量 | 同 [&] 的生命周期警告 |
| [x,&y] | 按值捕获指定变量 x，按引用捕获指定变量 y | 需要用到x,想改变y | 综合上方两条 |
| [this] | 捕获当前对象指针 | 类成员函数里想访问成员变量 | 其实等价于 [&] 捕获 this |
| [=, &foo] | 混合捕获：其他按值，foo 按引用 | 细粒度控制 | 仅C++17,次序：先默认再特指 |

## 基础用法
直接把num1与num2当参数传入，此时捕获列表不用任何东西：

```cpp
#include <iostream>

int main(int argc,char **argv)
{
    int num1 = 5;
    int num2 = 6;

    auto add = [](int a,int b)
    {
        return a + b;
    };

    std::cout << add(num1,num2) << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image7.webp)

使用捕获列表传入变量：

```cpp
#include <iostream>

int main(int argc,char **argv)
{
    int num1 = 5;
    int num2 = 6;
    int num3 = 1;

    auto add = [num1,&num2](int c)
    {
        num2 = 3;
        return num1 + num2 + c;
    };

    std::cout << add(num3) << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

答案是9

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image8.webp)

接下来是讲捕获this：

在 **类的成员函数中** ，你可能希望 lambda 能访问或修改成员变量。这时，你要捕获当前对象的指针 `this`。

```cpp
#include <iostream>

class MyClass 
{
    public:
        int num2 = 2;

        void do_add() 
        {
            int num1 = 10;
            int num3 = 6;

            // 在类的成员函数中访问自己的成员变量
            auto add = [num1, this](int c) 
            {
                num2 = 3;  // 访问 this->num2
                return num1 + num2 + c;
            };

            std::cout << add(num3) << std::endl;
        }
};

int main() {
    MyClass obj;
    obj.do_add();

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

## 高级用法
| 特性 | C++11 | C++14 | C++17 |
|:---|:---|:---|:---|
| 捕获变量 | ✅ | ✅ | ✅（支持组合、更灵活） |
| mutable | ✅ | ✅（配合 auto 更实用） | ✅ |
| 泛型参数 auto | ❌ | ✅ | ✅ |
| 捕获 *this | ❌ | ❌ | ✅（值拷贝 this） |
| constexpr λ | ❌ | ❌ | ✅ |
| 嵌套递归 lambda | 复杂 | 稍好（配合 std::function） | 更方便 |

1.  mutable

如果我就想修改按值捕获的变量怎么办？

C++11提供了mutable，可以让你在Lambda里直接修改按值捕获的变量。

```cpp
#include <iostream>

int main(int argc,char **argv)
{
    int num1 = 5;
    int num2 = 6;
    int num3 = 1;

    auto add = [num1,&num2](int c)mutable
    {
        num1 = 6;
        num2 = 3;
        return num1 + num2 + c;
    };

    std::cout << add(num3) << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

答案是10

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image9.webp)

2.  泛型 lambda 参数

C++14提供了泛型 lambda 参数，不再需要模板函数了，lambda 直接支持“模板参数”！

```cpp
#include <iostream>

int main(int argc,char **argv)
{

    auto add = [](auto a,auto b)
    {
        return a + b;
    };

    std::cout << add(1,7) << std::endl;
    std::cout << add(1.1,1.7) << std::endl;
    std::cout << add(std::string("hi "), std::string("vinci")) << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get(); // 等待用户按下回车键
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image10.webp)

3.  捕获\*this

| 捕获方式 | 意思 | 会捕获什么 | 生命周期影响 |
|:---|:---|:---|:---|
| [this] | 捕获当前对象的指针 | 指针 this | 外部对象必须活着，否则悬空 |
| [*this] | 捕获当前对象的副本 | 整个对象（值拷贝） | 安全，跟 lambda 活得一样久 |

如果只是\[this\]，`this` 是一个指针，指向当前对象，这个 lambda 只是记住了这个地址（指针），如果这个 lambda 比对象生命周期长，就悬挂指针❌。

如果是\[\*this\]，直接把整个对象拷贝进 lambda，不依赖外部对象是否还活着，lambda比对象活得久也没关系这个 lambda 可以安全延迟执行、在线程中运行等。

`[this]` 就像是说：“我知道对象在哪，我只存地址” 🧷

`[*this]` 是：“我整个对象拷一份，我自己留着用” 📦

```cpp
#include <iostream>

class A {
public:
    int x = 100;

    auto get_this_lambda() 
    {
        return [this]() { std::cout << "this: " << x << "\n"; };
    }

    auto get_star_this_lambda() 
    {
        return [*this]() { std::cout << "*this: " << x << "\n"; };
    }
};

int main() 
{
    A a;
    auto lam1 = a.get_this_lambda();
    auto lam2 = a.get_star_this_lambda();

    lam1();  // 输出 100
    lam2();  // 输出 100

    a.x = 200;

    lam1();  // 输出 200 （引用原对象）
    lam2();  // 还是 100 （因为是副本）

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

我如果想使用\*this,还想修改副本里的数值怎么办呢？那么就得加一个mutable了。

```cpp
#include <iostream>

class MyClass 
{
    public:
        int num2 = 2;

        auto get_add_lambda() 
        {
            int num1 = 10;

            // 在类的成员函数中访问自己的成员变量
            return [num1, *this](int c)mutable
            {
                num2 = 3;  //用mutable才能修改副本的成员变量num2
                return num1 + num2 + c;
            };
        }
};

int main() 
{
    // 用 new 动态创建 obj
    MyClass* obj = new MyClass();

    // 从堆上的 obj 获取 lambda
    auto add_func = obj->get_add_lambda();

    // 手动释放 obj（原对象没了）
    delete obj;

    // lambda 里访问的是副本，安全
    std::cout << add_func(6) << std::endl;  // 输出 10 + 3 + 6 = 19

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

4.  constexpr编译期lambda

```cpp
#include <iostream>

int main() 
{
    // C++17 constexpr lambda，捕获为空
    constexpr auto square = [](int x) constexpr 
    {
        return x * x;
    };

    constexpr int val = square(5);  // 编译期计算
    std::cout << val << std::endl; // 输出 25

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image11.webp)

上面`constexpr auto square = [](int x) constexpr` 有俩`constexpr`，第一个是用来声明`square`的，第二个是用来声明Lambda的，作用不一样，而且这俩都必须要。

要在这句里实现完整的 **编译期调用 + 编译期结果存储** ，你必须要：

*   lambda 自己是 `constexpr` 的（可以编译期执行）；

*   返回值是编译期常量（用 `constexpr` 声明 `square` 变量）；

否则：

*   如果你只写 lambda 是 `constexpr`，但没有 `constexpr auto square =`，`square` 变量还是运行期的；

*   如果你只写 `constexpr auto square =`，但 lambda 不能编译期执行，那编译会报错。
