---
title: "C++基础初识"
---

## C++初识

### 本章运行约定

本章的代码示例默认在 `ch1-C++开发环境与第一个工程` 创建的 CMake 工程中运行。

前期语法练习主要编辑这个文件：

```text
src/main.cpp
```

每个示例都可以把代码复制到 `src/main.cpp` 中，然后用 CMake 模板构建运行。

这样做的好处是：后面学习头文件、库、第三方依赖和 VSCode 调试时，都使用同一个工程环境，不需要临时切换到另一套编译方式。

常用命令：

```bash
cmake --build --preset linux-debug
./build/linux-debug/src/hello_cpp
```

如果你还没有完成上一章的工程环境搭建，请先看 `ch1-C++开发环境与第一个工程`。

### 第一个C++程序

打开工程中的：

```text
src/main.cpp
```

把内容替换为：

```cpp
#include <iostream>

using namespace std;

int main()
{
    // 程序从 main 函数开始执行，下面的语句会按顺序运行。
	cout << "Hello world" << endl;

	// 返回 0 表示程序正常结束。
	return 0;
}
```

**运行/观察结果：** 运行后会按输出语句打印对应内容，变量值可结合初始化、赋值和函数调用顺序推导。

#### 构建并运行程序

在工程根目录执行：

```bash
cmake --build --preset linux-debug
./build/linux-debug/src/hello_cpp
```

终端输出：

```text
Hello world
```

### main函数参数

`main` 函数不是必须带参数。最常见的两种写法是：

```cpp
int main()
{
	// 程序从 main 函数开始执行，下面的语句会按顺序运行。
	// 返回 0 表示程序正常结束。
	return 0;
}
```

**运行/观察结果：** 这是最简洁的程序入口，适合不需要读取命令行参数的程序。

```cpp
int main(int argc, char* argv[])
{
	// 程序从 main 函数开始执行，argc/argv 用来接收命令行参数。
	// 返回 0 表示程序正常结束。
	return 0;
}
```

**运行/观察结果：** 这也是标准的程序入口写法，适合需要读取命令行参数的程序。

其中：

* `argc` 表示命令行参数的数量，通常至少为 1，因为程序自身的路径也会算作一个参数。
* `argv` 表示命令行参数数组，`argv[0]` 通常是程序自身的路径或名称，`argv[1]` 开始才是用户传入的参数。
* `char* argv[]` 也可以写成 `char** argv`，两者在这里表达的含义相同。

示例：

```cpp
#include <iostream>

using namespace std;

int main(int argc, char* argv[])
{
    // 程序从 main 函数开始执行，argc/argv 用来接收命令行参数。

	cout << "argc = " << argc << endl;

	for (int i = 0; i < argc; i++) {
		cout << "argv[" << i << "] = " << argv[i] << endl;
	}

	// 返回 0 表示程序正常结束。
	return 0;
}
```

**运行结果**：见下方“运行结果”；`argv[0]` 会随启动方式不同而变化，重点观察 `argc` 和各个 `argv` 的对应关系。

构建并传入命令行参数运行：

```bash
cmake --build --preset linux-debug
./build/linux-debug/src/hello_cpp hello 123
```

**运行结果**：

```text
argc = 3
argv[0] = ./build/linux-debug/src/hello_cpp
argv[1] = hello
argv[2] = 123
```

> 注意：`argv[0]` 的具体内容和启动方式有关，可能是 `./build/linux-debug/src/hello_cpp`，也可能是完整路径。实际处理参数时，通常从 `argv[1]` 开始读取用户输入的内容。

### 注释

**作用**：在代码中加一些说明和解释，方便自己或其他程序员程序员阅读代码

**两种格式**

1. **单行注释**：`// 描述信息` 
   - 通常放在一行代码的上方，或者一条语句的末尾，==对该行代码说明==
2. **多行注释**： `/* 描述信息 */`
   - 通常放在一段代码的上方，==对该段代码做整体说明==

> 提示：编译器在编译代码时，会忽略注释的内容

### 变量

**作用**：给一段指定的内存空间起名，方便操作这段内存

**语法**：`数据类型 变量名 = 初始值;`

**示例：**

```cpp
#include<iostream>
using namespace std;

int main() {

	//变量的定义
	//语法：数据类型  变量名 = 初始值

	int a = 10;

	cout << "a = " << a << endl;
	

	return 0;
}
```

**运行/观察结果：** 运行后会按输出语句打印对应内容，变量值可结合初始化、赋值和函数调用顺序推导。

> 注意：C++在创建变量时，必须给变量一个初始值，否则会报错

### 常量

**作用**：用于记录程序中不可更改的数据

C++定义常量两种方式

1. **\#define** 宏常量： `#define 常量名 常量值`
   * ==通常在文件上方定义==，表示一个常量

2. **const**修饰的变量 `const 数据类型 常量名 = 常量值`
   * ==通常在变量定义前加关键字const==，修饰该变量为常量，不可修改

**示例：**

```cpp
//1、宏常量
#define day 7

int main() {

	cout << "一周里总共有 " << day << " 天" << endl;
	//day = 8;  //报错，宏常量不可以修改

	//2、const修饰变量
	const int month = 12;
	cout << "一年里总共有 " << month << " 个月份" << endl;
	//month = 24; //报错，常量是不可以修改的
	
	

	return 0;
}
```

**运行/观察结果：** 运行后会按输出语句打印对应内容，变量值可结合初始化、赋值和函数调用顺序推导。

### 关键字

**作用：**关键字是C++中预先保留的单词（标识符）

* **在定义变量或者常量时候，不要用关键字**

C++关键字如下：

| asm        | do           | if               | return      | typedef  |
| ---------- | ------------ | ---------------- | ----------- | -------- |
| auto       | double       | inline           | short       | typeid   |
| bool       | dynamic_cast | int              | signed      | typename |
| break      | else         | long             | sizeof      | union    |
| case       | enum         | mutable          | static      | unsigned |
| catch      | explicit     | namespace        | static_cast | using    |
| char       | export       | new              | struct      | virtual  |
| class      | extern       | operator         | switch      | void     |
| const      | false        | private          | template    | volatile |
| const_cast | float        | protected        | this        | wchar_t  |
| continue   | for          | public           | throw       | while    |
| default    | friend       | register         | true        |          |
| delete     | goto         | reinterpret_cast | try         |          |

`提示：在给变量或者常量起名称时候，不要用C++得关键字，否则会产生歧义。`

### 标识符命名规则

**作用**：C++规定给标识符（变量、常量）命名时，有一套自己的规则

* 标识符不能是关键字
* 标识符只能由字母、数字、下划线组成
* 第一个字符必须为字母或下划线
* 标识符中字母区分大小写

> 建议：给标识符命名时，争取做到见名知意的效果，方便自己和他人的阅读

## C 与 C++ 的关系和学习路线

**C/C++介绍：**

C和C++是两种的高级计算机语言，常见的高级语言还有Python，Rust，Go，C#(C Sharp、C++++)，Java，JavaScript，LinuxShell等等。

C++语言是在C语言的基础上，添加了面向对象、模板等现代程序设计语言的特性而发展起来的。两者无论是从语法规则上，还是从运算符的数量和使用上，都非常相似，所以我们常常将这两门语言统称为“C/C++”。

C语言和C++并不是对立的竞争关系： 1)C++是C语言的加强，是一种更好的C语言，实际上C++和C语言是同一门语言的不同版本。 2)C++是以C语言为基础的，并且完全兼容C语言的特性。 C语言和C++语言的学习是可以相互促进。学好C语言，可以为我们将来进一步地学习C++语言打好基础，而C++语言的学习，也会促进我们对于C语言的理解，从而更好地运用C语言。

| 特性 | C 语言 | C++ 语言 |
|:---|:---|:---|
| 编程范式 | 面向过程 | 多范式，支持面向对象 |
| 内存管理 | 手动管理 | 手动管理，提供 RAII（资源获取即初始化） |
| 代码复用性 | 较低 | 高，通过类、继承、模板等实现 |
| 标准库 | 标准 C 库 | 标准模板库（STL）和 C 标准库 |
| 运行效率 | 高 | 稍低于 C，但差距不大 |
| 应用场景 | 操作系统、嵌入式 | 游戏开发、图形处理、大型应用 |
| 类型检查 | 较松散 | 较严格，提供更多类型检查 |

**本文只负责指导一些问题，学****C/C++****还是以下列视频为主:**

C/C++环境配置:[电控组环境搭建大全](https://sdutvincirobot.feishu.cn/wiki/FQszwXIR5iQgCfk7pRwc9rYpnqg)

1.  C++ 视频教程：

https://www.bilibili.com/video/BV1et411b73Z

2.  鹏哥C语言视频：

https://www.bilibili.com/video/BV1cq4y1U7sg

3.  菜鸟教程：

https://www.runoob.com/cprogramming/c-tutorial.html

https://www.runoob.com/cplusplus/cpp-tutorial.html
