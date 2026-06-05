---
title: "C++开发环境与第一个工程"
---

这一章先不讲复杂理论，只做一件很具体的事：

在 Linux 上把一个 C++ 小工程跑起来，并改成自己的 HelloWorld。

如果你刚开始学 C++，看到 `CMake`、`Ninja`、`target_link_libraries` 这些名字不用紧张。本章不要求你理解它们的全部原理，先照着步骤把工程跑通。等后面学到头文件、分文件编写、第三方库时，再回来看这些内容会更自然。

完整 CMake 模板详解放在后面的 `ch21-CMake工程模板`。本章只关心两件事：

1. 你的电脑能不能编译并运行 C++ 工程。
2. 你能不能把模板改成一个最小的 HelloWorld 工程。

## 本章最终效果

做完本章后，你会得到一个名为 `hello_cpp` 的工程，运行结果类似：

```text
Hello, CMake project!
```

这一章会按下面的顺序来：

1. 在 Ubuntu 或 Fedora 上安装 C++ 开发工具。
2. 下载我准备好的 CMake 模板。
3. 先原样运行模板，确认环境没问题。
4. 把模板里的示例代码改成 HelloWorld。
5. 用 VSCode 按 F5 调试程序。

先认识几个名字，后面看到它们就不会太突然：

| 名字 | 先这样理解 |
|:---|:---|
| `g++` | 真正编译 C++ 代码的编译器 |
| `CMake` | 读取 `CMakeLists.txt`，生成构建规则 |
| `Ninja` | 按照 CMake 生成的规则执行编译 |
| `GDB` | 调试器，VSCode 按 F5 时会用到 |
| `lib1` / `lib2` | 模板里自带的两个示例小库，本章会删掉 `lib2`，把 `lib1` 改成 `hello` |

## 安装基础工具

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install git cmake ninja-build g++ gdb libeigen3-dev
```

这里安装了 `libeigen3-dev`，是因为模板原始示例里用到了 Eigen。先装上它，模板就能直接跑起来。

OpenCV 本章用不到。以后需要 OpenCV 时再安装：

```bash
sudo apt install libopencv-dev
```

### Fedora

```bash
sudo dnf install git cmake ninja-build gcc-c++ gdb eigen3-devel
```

OpenCV 本章用不到。以后需要 OpenCV 时再安装：

```bash
sudo dnf install opencv-devel
```

## 检查工具是否安装成功

```bash
git --version
cmake --version
ninja --version
g++ --version
gdb --version
```

这些命令都能输出版本号，就说明基础环境已经可用。

## 安装 VSCode 扩展

如果你只想先用命令行运行程序，这一节可以先看一遍，后面调试时再回来操作。

建议在 VSCode 里安装这两个扩展：

1. CMake Tools
2. C/C++

它们的分工是：

| 扩展 | 作用 |
|:---|:---|
| CMake Tools | 识别 `CMakePresets.json`，执行 configure/build，选择 target |
| C/C++ | 提供 IntelliSense、断点、GDB 调试 |

简单说：CMake Tools 负责构建工程，C/C++ 扩展负责代码提示和调试。

## clone CMake 模板

```bash
git clone https://github.com/tungchiahui/CMake_Template.git hello_cpp
cd hello_cpp
```

这里把项目目录命名为 `hello_cpp`。后面所有命令都默认在这个目录里执行。

刚开始学习时先不要管 Git 历史，也不要急着改很多文件。第一步只做一件事：确认这个模板能在你的电脑上运行。

## 先原样跑通模板

先不要急着改代码。直接复制下面三行命令：

```bash
cmake --preset linux-debug
cmake --build --preset linux-debug
./build/linux-debug/src/cmake_template
```

如果输出类似：

```text
[lib1] Vector v = 1 2 3
[lib1] Norm = 3.74166
[lib2] Matrix m =
1 2
3 4
[lib2] Determinant = -2
```

说明工程、编译器、CMake、Ninja、Eigen 都工作正常。到这里，最麻烦的环境部分就已经过关了。

注意：日常开发时可以直接运行 build 目录里的程序，不需要每次 install。

只有在验证安装布局时，才需要：

```bash
cmake --install build/linux-debug
./install/linux-debug/bin/cmake_template
```

## 模板当前结构

现在看一下模板里和本章有关的文件。先不用理解每个 CMake 选项，只要知道 `src/` 下面放的是代码：

```text
.
├── CMakeLists.txt
├── CMakePresets.json
├── .vscode/
│   └── launch.json
├── cmake/
│   └── ProjectOptions.cmake
└── src/
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

这里最重要的是：

| 位置 | 作用 |
|:---|:---|
| `src/main.cpp` | 程序入口，运行时从这里开始 |
| `src/lib1/` | 第一个示例小库 |
| `src/lib2/` | 第二个示例小库 |
| `CMakeLists.txt` | 告诉 CMake 这个工程怎么构建 |

接下来要做的改造很简单：

1. 删除 `lib2`，让工程只保留一个小库。
2. 把 `lib1` 改名为 `hello`。
3. 把 Eigen 示例代码改成打印 HelloWorld。
4. 把最终可执行程序名从 `cmake_template` 改成 `hello_cpp`。

改造后结构：

```text
.
├── CMakeLists.txt
├── CMakePresets.json
├── .vscode/
│   └── launch.json
├── cmake/
│   └── ProjectOptions.cmake
└── src/
    ├── CMakeLists.txt
    ├── main.cpp
    └── hello/
        ├── CMakeLists.txt
        ├── inc/hello/hello.hpp
        └── src/hello.cpp
```

## 删除 lib2

`lib2` 只是模板里的第二个示例库，本章用不到，可以删掉。

执行删除命令前，可以先确认自己还在 `hello_cpp` 目录里：

```bash
pwd
```

输出路径最后应该是 `hello_cpp`。

```bash
rm -rf src/lib2
```

只删目录还不够，后面还会修改 `src/CMakeLists.txt`，把对 `lib2` 的引用也删掉。

## 把 lib1 改名为 hello

`lib1` 这个名字只是模板示例名。我们把它改成更好理解的 `hello`。

先改库目录名：

```bash
mv src/lib1 src/hello
```

再改头文件目录名：

```bash
mv src/hello/inc/lib1 src/hello/inc/hello
```

最后把 Eigen 示例文件名改成 HelloWorld 文件名：

```bash
mv src/hello/inc/hello/eigen3_test.hpp src/hello/inc/hello/hello.hpp
mv src/hello/src/eigen3_test.cpp src/hello/src/hello.cpp
```

## 修改顶层 CMakeLists.txt

打开 `CMakeLists.txt`。

这里只改项目名。原来项目名是：

```cmake
project(cmake_template VERSION 1.0.0 LANGUAGES C CXX)
```

改成：

```cmake
project(hello_cpp VERSION 0.1.0 LANGUAGES C CXX)
```

因为 `src/CMakeLists.txt` 使用 `${PROJECT_NAME}` 创建可执行文件，所以最终程序名会从：

```text
cmake_template
```

变成：

```text
hello_cpp
```

如果你不确定自己有没有改对，可以直接把顶层 `CMakeLists.txt` 改成下面这样：

```cmake
cmake_minimum_required(VERSION 3.25)

project(hello_cpp VERSION 0.1.0 LANGUAGES C CXX)

include(GNUInstallDirs)

include("${CMAKE_CURRENT_LIST_DIR}/cmake/ProjectOptions.cmake")

add_subdirectory(src)
```

## 修改 src/CMakeLists.txt

打开 `src/CMakeLists.txt`。

这个文件负责告诉 CMake：

1. 主程序的入口文件是 `src/main.cpp`。
2. 工程里有一个小库目录 `hello`。
3. 主程序需要链接 `hello_src_lib` 这个库。

可以直接把 `src/CMakeLists.txt` 改成：

```cmake
add_executable(${PROJECT_NAME}
  ${CMAKE_CURRENT_SOURCE_DIR}/main.cpp
)

target_link_libraries(${PROJECT_NAME}
  PRIVATE
    project_options
    project_warnings
)

add_subdirectory(hello)

target_link_libraries(${PROJECT_NAME}
  PRIVATE
    hello_src_lib
)

set_target_properties(${PROJECT_NAME} PROPERTIES
  INSTALL_RPATH "$ORIGIN/../${CMAKE_INSTALL_LIBDIR}"
)

install(TARGETS ${PROJECT_NAME}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
```

这里最重要的变化是：

```cmake
add_subdirectory(hello)
```

和：

```cmake
hello_src_lib
```

意思是：主程序要使用 `hello` 小库。

## 修改 src/hello/CMakeLists.txt

HelloWorld 不需要 Eigen，所以删除 `find_package(Eigen3 REQUIRED)` 和 `Eigen3::Eigen`。

打开 `src/hello/CMakeLists.txt`，直接改成：

```cmake
set(PREFIX "hello")

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

这个文件暂时不引入任何第三方库。

如果以后要引入 Eigen、OpenCV 等库，就在 `Third-party dependencies` 区块中添加。完整写法见 `ch21-4-第三方库依赖写法`。

## 修改 hello.hpp

打开：

```text
src/hello/inc/hello/hello.hpp
```

写入：

```cpp
#pragma once

namespace hello {

void print_hello();

}  // namespace hello
```

这个文件只写函数声明。意思是：有一个叫 `print_hello` 的函数，后面会在 `.cpp` 文件里写它的具体内容。

头文件放在这个路径：

```text
inc/hello/hello.hpp
```

这样代码里就可以写：

```cpp
#include "hello/hello.hpp"
```

这种写法可以避免多个库都有同名头文件时发生混乱。

## 修改 hello.cpp

打开：

```text
src/hello/src/hello.cpp
```

写入：

```cpp
#include "hello/hello.hpp"

#include <iostream>

namespace hello {

void print_hello()
{
    std::cout << "Hello, CMake project!" << '\n';
}

}  // namespace hello
```

## 修改 main.cpp

打开：

```text
src/main.cpp
```

写入：

```cpp
#include "hello/hello.hpp"

int main()
{
    hello::print_hello();
    return 0;
}
```

这三个 C++ 文件的关系是：

| 文件 | 作用 |
|:---|:---|
| `hello.hpp` | 声明 `print_hello` 函数 |
| `hello.cpp` | 实现 `print_hello` 函数 |
| `main.cpp` | 调用 `hello::print_hello()` |

## 重新构建

因为改了 CMake 文件，建议重新 configure。

```bash
cmake --fresh --preset linux-debug
cmake --build --preset linux-debug
./build/linux-debug/src/hello_cpp
```

输出：

```text
Hello, CMake project!
```

如果你的 CMake 版本较旧，不支持 `--fresh`，可以删除对应构建目录后重新配置：

```bash
rm -rf build/linux-debug
cmake --preset linux-debug
cmake --build --preset linux-debug
./build/linux-debug/src/hello_cpp
```

## 用 VSCode F5 调试

这个模板已经提供 `.vscode/launch.json`。

它里面最关键的一行是：

```json
"program": "${command:cmake.launchTargetPath}"
```

意思是：让 CMake Tools 告诉 VSCode 当前选择的可执行 target 在哪里。

调试流程：

1. 用 VSCode 打开 `hello_cpp` 目录。
2. 选择 `linux-debug` preset。
3. Configure。
4. Build。
5. 选择 `hello_cpp` 作为 launch target。
6. 在 `main.cpp` 或 `hello.cpp` 打断点。
7. 按 F5，选择 `Debug CMake Target`。

如果断点能停住，说明 GDB、VSCode、CMake Tools 都已经配好。

## 常见问题

### 找不到 Ninja

报错类似：

```text
CMake was unable to find a build program corresponding to "Ninja"
```

安装：

```bash
sudo apt install ninja-build
```

或：

```bash
sudo dnf install ninja-build
```

### 找不到 gdb

F5 调试时如果找不到 GDB：

```bash
sudo apt install gdb
```

或：

```bash
sudo dnf install gdb
```

### F5 找不到程序

先确认已经构建：

```bash
cmake --preset linux-debug
cmake --build --preset linux-debug
```

再确认 VSCode CMake Tools 已经选择 `hello_cpp` 作为 launch target。

### 改了 CMakeLists.txt 后构建异常

重新 configure：

```bash
cmake --fresh --preset linux-debug
```

或者删除构建目录：

```bash
rm -rf build/linux-debug
cmake --preset linux-debug
```

## 下一步学什么

本章只是让你先拥有一个能运行、能调试、能继续扩展的小工程。

后面的章节会从 C++ 基础语法开始讲：

1. C++ 基础初识
2. 数据类型与数据存放
3. 输入输出
4. 运算符
5. 程序流程结构

等你学到工程组织、第三方库和构建系统时，再回来看 `ch21-CMake工程模板`，就会更容易理解每个 CMake 文件为什么这样写。
