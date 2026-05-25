---
title: "std::bind"
---

## 定义
`std::bind` 是 C++11 引入的一个函数适配器，可以用来 **绑定函数的参数、重排参数顺序、生成可调用对象（函数对象）** ，常用在需要把函数“提前部分绑定参数”或“适配成某种形式”的场景。

```cpp
#include <functional>

auto new_func = std::bind(函数名, 参数1, 参数2, ...);
```

参数可以是具体值，表示提前绑定；

也可以是 `_1`, `_2` 等占位符，表示“调用时再传入”；

`std::placeholders::_1`, `_2` 表示调用时提供的第一个、第二个参数…

## 基础用法
1.  绑定普通函数的部分参数

```cpp
#include <iostream>
#include <functional>

void add(int a, int b,int c) 
{
    std::cout << a + b + c << std::endl;
}

int main() 
{   
    //把第一个参数固定为 10，第二、三个等调用时再传
    auto func1 = std::bind(add,10,std::placeholders::_1,std::placeholders::_2);
    func1(20,30);   //10+20+30

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image12.webp)

2.  绑定成员函数

注意💡：函数出现了第0个参数（也可看作第一个参数，反正就是多一个默认的参数），这个参数是对象的指针，这是因为成员函数需要一个this。不知道为何要这么做的话，请你好好学学class,先打好基础。

```cpp
#include <iostream>
#include <functional>

class MyClass
{
    public:
        int add(int a, int b,int c) 
        {
            return a+b+c;
        }
};

int main() 
{   
    MyClass obj;
    //函数出现了第0个参数，这个参数是对象的指针，这是因为成员函数需要一个this。
    //把第一个参数固定为 10，第二、三个等调用时再传
    auto func1 = std::bind(&MyClass::add,&obj,10,std::placeholders::_1,std::placeholders::_2);
    int result = func1(20,30);   //10+20+30
    std::cout << "result:" << result << std::endl;

    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image13.webp)

3.  用于高阶函数

高阶函数其实就是把函数当作入口参数或者返回值，不懂的话，本文档也有讲，可以先去看看。（或者学完高阶函数再回来学这个）

比如你要把类的普通成员函数作为入口参数给某个函数，普通函数指针不支持，但 `std::bind` 就可以：

```cpp
#include <iostream>
#include <functional>

void DoSomething(std::function<int(int,int)> func)
{
    std::cout << func(20,30) << std::endl;
}

class MyClass
{
    public:
        int add(int a, int b, int c) 
        {
            return a+b+c;
        }
};

int main() 
{   
    MyClass obj;

    // 绑定 a = 10，b = _1，c = _2
    DoSomething(std::bind(&MyClass::add,&obj,10,std::placeholders::_1,std::placeholders::_2));
scr
    std::cout << "Press Enter to continue...";
    std::cin.get();
    return 0;
}

```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/image14.webp)
