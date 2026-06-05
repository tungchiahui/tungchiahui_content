---
title: "CMake工程模板"
---

本章用一个 Linux-only 的 CMake 工程模板来讲解 C++ 项目如何组织、构建、安装和引入第三方库。

这个模板的目标不是把 CMake 所有语法一次讲完，而是围绕一个真实可运行的工程，理解现代 CMake 最常用的一套写法：

1. 顶层 `CMakeLists.txt` 只做工程总控。
2. `CMakePresets.json` 固化 Debug / Release 构建配置。
3. `cmake/ProjectOptions.cmake` 保存项目公共编译选项。
4. `src/CMakeLists.txt` 管理主程序。
5. 每个库目录自己管理源码、头文件、安装规则和第三方依赖。
6. 第三方库采用"谁使用，谁 `find_package`"的方式。

## 模板目录结构

模板的核心目录如下：

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
    │   ├── inc/
    │   │   └── lib1/
    │   │       └── eigen3_test.hpp
    │   └── src/
    │       └── eigen3_test.cpp
    └── lib2/
        ├── CMakeLists.txt
        ├── inc/
        │   └── lib2/
        │       └── eigen3_test.hpp
        └── src/
            └── eigen3_test.cpp
```

其中：

| 文件 | 职责 |
|:---|:---|
| `CMakeLists.txt` | 顶层入口，声明项目、加载公共配置、进入 `src` |
| `CMakePresets.json` | 保存构建预设，例如 `linux-debug` 和 `linux-release` |
| `.vscode/launch.json` | 保存 VSCode F5 调试配置，启动 CMake Tools 当前选择的 target |
| `cmake/ProjectOptions.cmake` | 保存 C/C++ 标准、公共 warning 等配置 |
| `src/CMakeLists.txt` | 创建最终可执行文件，并链接 `lib1`、`lib2` |
| `src/lib1/CMakeLists.txt` | 创建 `lib1_src_lib`，管理 lib1 自己的 include、依赖、安装 |
| `src/lib2/CMakeLists.txt` | 创建 `lib2_src_lib`，管理 lib2 自己的 include、依赖、安装 |

## 构建命令

这个模板使用 CMakePresets + Ninja。

Debug 构建：

```bash
cmake --preset linux-debug
cmake --build --preset linux-debug
cmake --install build/linux-debug
./install/linux-debug/bin/cmake_template
```

Release 构建：

```bash
cmake --preset linux-release
cmake --build --preset linux-release
cmake --install build/linux-release
./install/linux-release/bin/cmake_template
```

这几条命令分别做了四件事：

| 命令 | 作用 |
|:---|:---|
| `cmake --preset linux-debug` | 配置工程，生成 Ninja 构建文件 |
| `cmake --build --preset linux-debug` | 编译源码并链接目标 |
| `cmake --install build/linux-debug` | 把可执行文件、库、头文件安装到 `install/linux-debug` |
| `./install/linux-debug/bin/cmake_template` | 运行安装后的程序 |

日常开发时，不需要每次都安装。编译后可以直接运行 build 目录里的程序：

```bash
./build/linux-debug/src/cmake_template
```

`install` 更适合用来验证"安装后的目录结构和动态库路径是否正确"。

## configure、build、install 的区别

CMake 工程一般有三个阶段：

### configure

```bash
cmake --preset linux-debug
```

这个阶段读取所有 `CMakeLists.txt` 和 `CMakePresets.json`，检查编译器、第三方库、变量，并生成构建系统文件。

使用 Ninja 时，会在 `build/linux-debug` 下生成 `build.ninja`。

### build

```bash
cmake --build --preset linux-debug
```

这个阶段调用 Ninja 编译 `.cpp` 文件，生成 `.o`、`.so`、可执行文件等构建产物。

### install

```bash
cmake --install build/linux-debug
```

这个阶段按照 `install(...)` 规则，把构建产物复制到安装目录。

在本模板中，Debug 安装到：

```text
install/linux-debug/
```

Release 安装到：

```text
install/linux-release/
```

安装后的典型结构如下：

```text
install/linux-debug/
├── bin/
│   └── cmake_template
├── include/
│   ├── lib1/
│   │   └── eigen3_test.hpp
│   └── lib2/
│       └── eigen3_test.hpp
└── lib64/
    ├── liblib1_src_lib.so
    └── liblib2_src_lib.so
```

有些 Linux 发行版会使用 `lib`，有些会使用 `lib64`。本模板通过 `GNUInstallDirs` 让 CMake 自动选择合适目录。

## 为什么不用手写 build/install/log 脚本

旧式模板里常见这种做法：

1. 自己创建 `build` 目录。
2. 在 `build` 里执行 `cmake ..`。
3. 手写 `make install`。
4. 手写脚本设置 `LD_LIBRARY_PATH`。
5. 自己维护日志目录。

这个模板改成 CMakePresets 后，这些内容都不再需要：

| 旧做法 | 新做法 |
|:---|:---|
| 手动 `cd build` | `CMakePresets.json` 统一设置 `binaryDir` |
| 手动设置 Debug / Release | preset 里设置 `CMAKE_BUILD_TYPE` |
| 手动设置安装路径 | preset 里设置 `CMAKE_INSTALL_PREFIX` |
| 手动调用 Makefile | preset 指定 Ninja |
| 手动设置动态库路径 | 安装目标设置 `INSTALL_RPATH` |
| 手写 VSCode task | VSCode CMake Tools 直接读取 preset |
| 手写复杂调试脚本 | `.vscode/launch.json` 调试当前 CMake target |

## VSCode F5 调试

这个模板可以按 F5 调试，但 F5 不是由 `CMakePresets.json` 单独完成的。

两个文件分工如下：

| 文件 | 职责 |
|:---|:---|
| `CMakePresets.json` | 决定如何 configure/build，构建目录在哪里 |
| `.vscode/launch.json` | 决定按 F5 时启动哪个程序、用哪个调试器 |

模板中的 `launch.json` 使用：

```json
"program": "${command:cmake.launchTargetPath}"
```

它表示让 VSCode CMake Tools 提供当前选择的可执行 target 路径。选择 `cmake_template` 后，F5 调试的就是类似下面这个 build 产物：

```text
build/linux-debug/src/cmake_template
```

推荐调试流程：

1. 安装 VSCode CMake Tools 扩展和 Microsoft C/C++ 扩展。
2. 选择 `linux-debug` configure preset。
3. Configure。
4. Build。
5. 选择 `cmake_template` 作为 launch target。
6. 按 F5，选择 `Debug CMake Target`。

## 本章建议阅读顺序

1. `ch21-1-CMakePresets与构建安装`：先理解 preset 工作流。
2. `ch21-2-顶层CMake与公共编译选项`：理解顶层入口和公共配置。
3. `ch21-3-src与lib模块CMake详解`：理解可执行文件、库、include、install。
4. `ch21-4-第三方库依赖写法`：学习 Eigen、OpenCV、Boost、PCL 等库如何引入。

学完后，你应该能做到：

1. 新建一个 C++ 工程并用 preset 构建。
2. 添加新的库目录。
3. 添加新的可执行文件。
4. 正确管理头文件 include 路径。
5. 在模块内部引入第三方库。
6. 安装后直接运行程序，不依赖手写脚本。
