---
title: "C++基础初识"
---

## C++初识

### 第一个C++程序

编写一个C++程序总共分为4个步骤

* 创建项目
* 创建文件
* 编写代码
* 运行程序

#### 创建项目

​	Visual Studio是我们用来编写C++程序的主要工具，我们先将它打开

![1541383178746](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/stage1-1541383178746-9b8dda0c6f.webp)

![1541384366413](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/stage1-1541384366413-21c94623c2.webp)

#### 创建文件

右键源文件，选择添加->新建项

![1541383817248](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/stage1-1541383817248-bb996478e0.webp)

给C++文件起个名称，然后点击添加即可。

![1541384140042](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/stage1-1541384140042-11110f8c33.webp)

#### 编写代码

```cpp
#include<iostream>
using namespace std;

int main() {

	cout << "Hello world" << endl;

	system("pause");

	return 0;
}
```

#### 运行程序

![1541384818688](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/10/05/stage1-1541384818688-7408f95c89.webp)

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
	
	system("pause");

	return 0;
}
```

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
	
	
	system("pause");

	return 0;
}
```

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

1.  黑马程序员C++视频：

https://www.bilibili.com/video/BV1et411b73Z

2.  鹏哥C语言视频：

https://www.bilibili.com/video/BV1cq4y1U7sg

3.  菜鸟教程：

https://www.runoob.com/cprogramming/c-tutorial.html

https://www.runoob.com/cplusplus/cpp-tutorial.html
