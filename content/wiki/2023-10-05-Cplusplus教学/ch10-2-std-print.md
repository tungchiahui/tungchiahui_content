---
title: "std::print"
---

std::print 是 C/C++ 在 C++23 新添加的功能（来自 `<print>` 头文件，类似 Python 的 print）。

运行速度最快 → 最慢：
std::print ≈ fmt::print > printf > std::cout（关闭同步） > std::cout（默认） > std::endl

1. 特点:
    因为它本质上是 C++ 标准把 `{fmt}` 库（fmtlib）正式吸收进来。

    fmt 库以极高性能著称，特性包括：

    ✔ 零开销抽象（zero overhead）

    模板 + 智能优化，生成的代码非常紧凑。

    ✔ 直接写到 OS buffer

    不像 cout 那样层层包装和同步机制。

    ✔ 没有 printf 的可变参数开销

    printf 要解析格式字符串，是运行时开销；
    std::print 在编译期做了大量优化。

    ✔ 无需同步 C stdio

    和 cout 不同，它默认不做同步。

    ✔ 类型安全且格式强大

    性能比 printf 更高，但无格式化字符串崩溃风险。

2. 使用方法
   1. 打印一串字符串
   std::print 会直接将字符串输出到标准输出（stdout），不会自动换行。
    ```cpp
    #include <print>
    int main(int argc,char **argv)
    {
        std::print("Hello, world!");
        return 0;
    }
    ```
   2. 打印并自动换行：std::println
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        std::println(std::cout, "HelloWorld!");
    }
   ```
   3. 打印一个变量
   {} 是占位符，会被变量自动格式化。
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        int value = 42;
        std::print("value = {}", value);
    }
    ```
    4. 打印多个变量
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        int a = 10;
        float b = 3.14f;
        std::print("a = {}, b = {}", a, b);
    }
   ```
    5. 控制小数点输出
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        double pi = 3.1415926;
        std::print("pi = {:.2f}", pi);  // 输出：pi = 3.14
    }
   ```
   6. 对齐方式（左对齐 / 右对齐 / 居中）
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        std::print("[{:>10}]", 42);  // 右对齐
        std::print("[{:<10}]", 42);  // 左对齐
        std::print("[{:^10}]", 42);  // 居中
    }
   ```
   7. 打印整数为十六进制
   会加上 0x 前缀。
   ```cpp
   #include <print>
    int main(int argc,char **argv)
    {
        int x = 255;
        std::print("hex = {:#x}", x);   // 输出：hex = 0xff
    }
   ```
