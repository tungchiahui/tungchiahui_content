---
title: "函数与头文件"
---

## 函数

### 概述

**作用：**将一段经常使用的代码封装起来，减少重复代码

一个较大的程序，一般分为若干个程序块，每个模块实现特定的功能。

### 函数的定义

函数的定义一般主要有5个步骤：

1、返回值类型 

2、函数名

3、参数表列

4、函数体语句 

5、return 表达式

**语法：** 

```cpp
返回值类型 函数名 （参数列表）
{

       函数体语句

       return表达式

}
```

**运行/观察结果：** 这段是语法或接口示例，重点观察写法；放入完整程序后再运行验证。

* 返回值类型 ：一个函数可以返回一个值。在函数定义中
* 函数名：给函数起个名称
* 参数列表：使用该函数时，传入的数据
* 函数体语句：花括号内的代码，函数内需要执行的语句
* return表达式： 和返回值类型挂钩，函数执行完后，返回相应的数据

**示例：**定义一个加法函数，实现两个数相加

```cpp
//函数定义
int add(int num1, int num2)
{
	int sum = num1 + num2;
	return sum;
}
```

**运行/观察结果：** 这段是语法或接口示例，重点观察写法；放入完整程序后再运行验证。

### 函数的调用

**功能：**使用定义好的函数

**语法：**` 函数名（参数）`

**示例：**

```cpp
//函数定义
int add(int num1, int num2) //定义中的num1,num2称为形式参数，简称形参
{
	int sum = num1 + num2;
	return sum;
}

int main() {

	int a = 10;
	int b = 10;
	//调用add函数
	int sum = add(a, b);//调用时的a，b称为实际参数，简称实参
	cout << "sum = " << sum << endl;

	a = 100;
	b = 100;

	sum = add(a, b);
	cout << "sum = " << sum << endl;


	return 0;
}
```

**运行/观察结果：** 运行后会按输出语句打印对应内容，变量值可结合初始化、赋值和函数调用顺序推导。

> 总结：函数定义里小括号内称为形参，函数调用时传入的参数称为实参

### 值传递

* 所谓值传递，就是函数调用时实参将数值传入给形参
* 值传递时，==如果形参发生，并不会影响实参==

**示例：**

```cpp
void swap(int num1, int num2)
{
	cout << "交换前：" << endl;
	cout << "num1 = " << num1 << endl;
	cout << "num2 = " << num2 << endl;

	int temp = num1;
	num1 = num2;
	num2 = temp;

	cout << "交换后：" << endl;
	cout << "num1 = " << num1 << endl;
	cout << "num2 = " << num2 << endl;

	//return ; 当函数声明时候，不需要返回值，可以不写return
}

int main() {

	int a = 10;
	int b = 20;

	swap(a, b);

	cout << "mian中的 a = " << a << endl;
	cout << "mian中的 b = " << b << endl;


	return 0;
}
```

**运行/观察结果：** 运行后会按输出语句打印对应内容，变量值可结合初始化、赋值和函数调用顺序推导。

> 总结： 值传递时，形参是修饰不了实参的

### **函数的常见样式**

常见的函数样式有4种

1. 无参无返
2. 有参无返
3. 无参有返
4. 有参有返

**示例：**

```cpp
//函数常见样式
//1、 无参无返
void test01()
{
	//void a = 10; //无类型不可以创建变量,原因无法分配内存
	cout << "this is test01" << endl;
	//test01(); 函数调用
}

//2、 有参无返
void test02(int a)
{
	cout << "this is test02" << endl;
	cout << "a = " << a << endl;
}

//3、无参有返
int test03()
{
	cout << "this is test03 " << endl;
	return 10;
}

//4、有参有返
int test04(int a, int b)
{
	cout << "this is test04 " << endl;
	int sum = a + b;
	return sum;
}
```

**运行/观察结果：** 运行后会打印示例中的变量值或地址；地址值与运行环境有关，以同类对象的相对位置和指针变化为观察重点。

### 函数的声明

**作用：** 告诉编译器函数名称及如何调用函数。函数的实际主体可以单独定义。

*  函数的**声明可以多次**，但是函数的**定义只能有一次**

**示例：**

```cpp
//声明可以多次，定义只能一次
//声明
int max(int a, int b);
int max(int a, int b);
//定义
int max(int a, int b)
{
	return a > b ? a : b;
}

int main() {

	int a = 100;
	int b = 200;

	cout << max(a, b) << endl;


	return 0;
}
```

**运行/观察结果：** 运行后会按输出语句打印对应内容，变量值可结合初始化、赋值和函数调用顺序推导。

### 函数的分文件编写

**作用：**让代码结构更加清晰

前面几章我们主要在 `src/main.cpp` 里写代码。这样适合学习语法，但是程序稍微大一点，就会变得很乱。

函数的分文件编写，就是把代码拆成：

1. 头文件 `.h`：写函数声明。
2. 源文件 `.cpp`：写函数定义。
3. `main.cpp`：调用函数。

从 `ch1` 开始，本教程一直使用同一个 CMake 模板。写到这里时，你的工程大致还是这个结构：

```text
src/
├── CMakeLists.txt
├── main.cpp
├── lib1/
│   ├── CMakeLists.txt
│   ├── inc/lib1/eigen3_test.hpp
│   └── src/eigen3_test.cpp
└── lib2/
    ├── CMakeLists.txt
    ├── inc/lib2/eigen3_test.hpp
    └── src/eigen3_test.cpp
```

`lib1` 和 `lib2` 是模板自带的两个示例库。本节第一次整理它们：删掉暂时用不到的 `lib2`，把 `lib1` 改成我们自己的 `swap` 小库，然后把前面写过的 `swap.h` 和 `swap.cpp` 放进去。

改完之后，目录结构变成：

```text
src/
├── CMakeLists.txt
├── main.cpp
└── swap/
    ├── CMakeLists.txt
    ├── inc/swap/swap.h
    └── src/swap.cpp
```

也就是说：

1. `lib2` 不参与这个例子。
2. `lib1` 被改名为 `swap`。
3. `swap.h` 放到 `src/swap/inc/swap/`。
4. `swap.cpp` 放到 `src/swap/src/`。

#### 整理 lib1 和 lib2 (你也可以用VScode图形界面删,不用敲以下命令)

执行删除命令前，可以先确认自己还在工程根目录：

```bash
pwd
```

输出路径最后应该是你 clone 下来的工程目录，比如 `hello_cpp`。

删掉暂时用不到的 `lib2`：

```bash
rm -rf src/lib2
```

把 `lib1` 改名为 `swap`：

```bash
mv src/lib1 src/swap
```

然后把头文件改成原教程里的名字：

```bash
mv src/swap/inc/lib1 src/swap/inc/swap
mv src/swap/inc/swap/eigen3_test.hpp src/swap/inc/swap/swap.h
```

再把源文件改成原教程里的名字：

```bash
mv src/swap/src/eigen3_test.cpp src/swap/src/swap.cpp
```

你也可以在 VSCode 左侧文件树里手动删除、重命名，结果一致即可。

现在 `swap.h` 放在：

```text
src/swap/inc/swap/swap.h
```

`swap.cpp` 放在：

```text
src/swap/src/swap.cpp
```

#### 修改 swap.h

打开：

```text
src/swap/inc/swap/swap.h
```

写入：

```cpp
#include<iostream>
using namespace std;

void swap(int a, int b);
```

这段代码和前面的分文件示例保持一致。

> 补充说明：真实工程里通常不建议在头文件中写 `using namespace std;`，函数名也尽量避免直接叫 `swap`，因为标准库里有 `std::swap`。不过这里先专心理解"头文件声明、源文件定义、主函数调用"这条线，所以仍然沿用前面的教学代码。

#### 修改 swap.cpp

打开：

```text
src/swap/src/swap.cpp
```

写入：

```cpp
#include "swap/swap.h"

void swap(int a, int b)
{
	int temp = a;
	a = b;
	b = temp;

	cout << "a = " << a << endl;
	cout << "b = " << b << endl;
}
```

这里的 `#include "swap/swap.h"` 表示：`swap.cpp` 要先看到函数声明，再提供函数定义。

#### 修改 main.cpp

打开：

```text
src/main.cpp
```

写入：

```cpp
#include "swap/swap.h"

int main()
{
	int a = 100;
	int b = 200;

	swap(a, b);

	return 0;
}
```

`main.cpp` 只负责调用函数，不再把 `swap` 函数的具体实现写在自己里面。

#### 修改 src/swap/CMakeLists.txt

打开：

```text
src/swap/CMakeLists.txt
```

这个文件原来来自 `lib1`，所以里面还有 `lib1` 和 Eigen 示例的痕迹。

先把第一行从：

```cmake
set(PREFIX "lib1")
```

改成：

```cmake
set(PREFIX "swap")
```

这样 CMake 会创建一个名为：

```text
swap_src_lib
```

的库。

`swap` 这个例子不需要 Eigen，所以删除 `Third-party dependencies` 区块里的 Eigen 依赖：

```cmake
find_package(Eigen3 REQUIRED)

target_link_libraries(${PREFIX}_src_lib
  PUBLIC
    Eigen3::Eigen
)
```

整理后，`src/swap/CMakeLists.txt` 可以写成：

```cmake
set(PREFIX "swap")

file(GLOB_RECURSE ${PREFIX}_SRC_LIST CONFIGURE_DEPENDS
  "${CMAKE_CURRENT_LIST_DIR}/src/*.c"
  "${CMAKE_CURRENT_LIST_DIR}/src/*.cpp"
)

add_library(${PREFIX}_src_lib SHARED
  ${${PREFIX}_SRC_LIST}
)

target_include_directories(${PREFIX}_src_lib
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/inc>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

target_link_libraries(${PREFIX}_src_lib
  PUBLIC
    project_options
  PRIVATE
    project_warnings
)

# ========================
# Third-party dependencies
# ========================

# =======
# Install
# =======
install(TARGETS ${PREFIX}_src_lib
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(DIRECTORY "${CMAKE_CURRENT_LIST_DIR}/inc/"
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
```

这里最重要的几行是：

```cmake
file(GLOB_RECURSE ${PREFIX}_SRC_LIST CONFIGURE_DEPENDS
  "${CMAKE_CURRENT_LIST_DIR}/src/*.c"
  "${CMAKE_CURRENT_LIST_DIR}/src/*.cpp"
)
```

它会把 `src/swap/src/` 里面的 `.cpp` 文件收集进来，所以 `swap.cpp` 会参与编译。

还有：

```cmake
target_include_directories(${PREFIX}_src_lib
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/inc>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
```

它把 `src/swap/inc/` 加入头文件搜索路径，所以代码里才能写：

```cpp
#include "swap/swap.h"
```

注意：不要写成 `#include "inc/swap/swap.h"`。CMake 已经把 `src/swap/inc/` 告诉编译器了，所以 include 时从 `inc/` 里面继续往后写，也就是 `swap/swap.h`。

完整 CMake 写法会在 `ch21-CMake工程模板` 中系统讲解。这里先知道"头文件放进 `inc/`，源文件放进 `src/`，库名由 `PREFIX` 决定"即可。

#### 修改 src/CMakeLists.txt

打开：

```text
src/CMakeLists.txt
```

原来模板里有：

```cmake
add_subdirectory(lib1)
add_subdirectory(lib2)
```

改成：

```cmake
add_subdirectory(swap)
```

原来模板里还会链接：

```cmake
target_link_libraries(${PROJECT_NAME}
  PRIVATE
    lib1_src_lib
    lib2_src_lib
)
```

改成：

```cmake
target_link_libraries(${PROJECT_NAME}
  PRIVATE
    swap_src_lib
)
```

整理后，`src/CMakeLists.txt` 可以写成：

```cmake
add_executable(${PROJECT_NAME}
  ${CMAKE_CURRENT_SOURCE_DIR}/main.cpp
)

target_link_libraries(${PROJECT_NAME}
  PRIVATE
    project_options
    project_warnings
)

add_subdirectory(swap)

target_link_libraries(${PROJECT_NAME}
  PRIVATE
    swap_src_lib
)

set_target_properties(${PROJECT_NAME} PROPERTIES
  INSTALL_RPATH "$ORIGIN/../${CMAKE_INSTALL_LIBDIR}"
)

install(TARGETS ${PROJECT_NAME}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
```

这两处最关键：

```cmake
add_subdirectory(swap)
```

意思是：让 CMake 进入 `src/swap/`，读取 `src/swap/CMakeLists.txt`。

```cmake
target_link_libraries(${PROJECT_NAME}
  PRIVATE
    swap_src_lib
)
```

意思是：主程序调用了 `swap` 函数，所以主程序要链接 `swap_src_lib`，否则链接器找不到 `swap` 函数的定义。

#### 重新构建并运行

因为改了目录名和 `CMakeLists.txt`，建议重新 configure：

```bash
cmake --fresh --preset linux-debug
cmake --build --preset linux-debug
./build/linux-debug/src/cmake_template
```

如果你的 CMake 版本较旧，不支持 `--fresh`，可以改成：

```bash
rm -rf build/linux-debug
cmake --preset linux-debug
cmake --build --preset linux-debug
./build/linux-debug/src/cmake_template
```

输出类似：

```text
a = 200
b = 100
```

可以看到，`swap.cpp` 里的函数确实被主程序调用到了。

#### 分文件编写的核心理解

在这个模板中，函数分文件编写不是只把文件拆开，还要让 CMake 知道这些文件属于哪个库。

对应关系如下：

| 文件 | 作用 |
|:---|:---|
| `src/swap/inc/swap/swap.h` | 写函数声明，让其他 `.cpp` 文件知道怎么调用 |
| `src/swap/src/swap.cpp` | 写函数定义，提供真正的实现 |
| `src/swap/CMakeLists.txt` | 把 `swap.cpp` 编译成 `swap_src_lib` |
| `src/CMakeLists.txt` | 把 `swap_src_lib` 链接到主程序 |
| `src/main.cpp` | `#include "swap/swap.h"` 后调用 `swap(a, b)` |

如果漏了 `#include "swap/swap.h"`，编译器不知道 `swap` 函数的声明。

如果漏了 `add_subdirectory(swap)`，CMake 不会进入 `src/swap`。

如果漏了：

```cmake
target_link_libraries(${PROJECT_NAME}
  PRIVATE
    swap_src_lib
)
```

链接器找不到 `swap` 函数的实现。

## 头文件的组织方式(只是一个规范,现在暂时不用知道)

头文件的作用：头文件含有某个库的外部声明函数和变量，方便我们调用库中的API。

注意事项：

1.  常见的头文件stdio.h stdlib.h iostream string等

2.  头文件的扩展名：.h或者.hpp，其实没必要写扩展名，但是建议还是写。

3.  预处理：#include <> 和 #include " "

4.  条件编译

5.  extern "C" { } 用来实现C语言和C++的混合编译，表明它按照类C的编译和连接规约来编译和连接，而不是C++的编译的连接规约。

```cpp
#ifndef __FILE_NAME_H_    //头文件防止引用重复的条件编译(记得修改宏定义的名字)
#define __FILE_NAME_H_   //头文件防止引用重复的条件编译

#ifdef __cplusplus    //混合编译的条件编译
extern "C"           //混合编译的条件编译
{                   //混合编译的条件编译
#endif             //混合编译的条件编译
/*  头文件内容开始   */

//头文件内容：预处理、函数声明、变量声明(上面学的内容全部写在这里,其他的都是模板)

/*   头文件内容结束  */
#ifdef __cplusplus     //混合编译的条件编译
}                      //混合编译的条件编译
#endif                 //混合编译的条件编译

#endif   //头文件防止引用重复的条件编译

```

**运行/观察结果：** 这段是语法或接口示例，重点观察写法；放入完整程序后再运行验证。
