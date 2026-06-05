---
title: "CMakePresets与构建安装"
---

`CMakePresets.json` 用来保存常用的 CMake 配置。它解决的问题是：不要每次都手写一长串 `cmake -S ... -B ... -G ... -D...` 命令。

本模板的 `CMakePresets.json` 如下：

```json
{
  "version": 6,
  "cmakeMinimumRequired": {
    "major": 3,
    "minor": 25,
    "patch": 0
  },
  "configurePresets": [
    {
      "name": "linux-debug",
      "displayName": "Linux Debug",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/linux-debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "CMAKE_INSTALL_PREFIX": "${sourceDir}/install/linux-debug"
      }
    },
    {
      "name": "linux-release",
      "displayName": "Linux Release",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/linux-release",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "CMAKE_INSTALL_PREFIX": "${sourceDir}/install/linux-release"
      }
    }
  ],
  "buildPresets": [
    {
      "name": "linux-debug",
      "configurePreset": "linux-debug"
    },
    {
      "name": "linux-release",
      "configurePreset": "linux-release"
    }
  ]
}
```

## 根字段

### `version`

```json
"version": 6
```

表示 `CMakePresets.json` 文件格式版本，不是项目版本。

可以填什么：

| 值 | 含义 |
|:---|:---|
| `1`、`2`、`3`、... | CMake preset 文件格式版本 |
| `6` | 本模板使用的版本，适合 CMake 3.25 及以上 |

注意：

1. 这里不是 `cmake_template` 的版本号。
2. 如果想兼容更老的 CMake，要降低 `version`，但某些字段可能不能再用。
3. 如果本机 CMake 很新，也不一定要用最高 preset 版本；够用即可。

### `cmakeMinimumRequired`

```json
"cmakeMinimumRequired": {
  "major": 3,
  "minor": 25,
  "patch": 0
}
```

表示读取这个 preset 文件所需的最低 CMake 版本。

可以填什么：

| 字段 | 示例 | 含义 |
|:---|:---|:---|
| `major` | `3` | 主版本号 |
| `minor` | `25` | 次版本号 |
| `patch` | `0` | 补丁版本号 |

这个模板要求 CMake >= 3.25，是因为顶层 `CMakeLists.txt` 也写了：

```cmake
cmake_minimum_required(VERSION 3.25)
```

二者最好保持一致。

## `configurePresets`

`configurePresets` 是"配置阶段"的 preset。

配置阶段对应命令：

```bash
cmake --preset linux-debug
```

它等价于告诉 CMake：

1. 使用哪个生成器。
2. 构建目录在哪里。
3. 构建类型是什么。
4. 安装目录在哪里。
5. 是否生成 `compile_commands.json`。

### `name`

```json
"name": "linux-debug"
```

preset 的机器可读名称。

可以填什么：

| 写法 | 说明 |
|:---|:---|
| `linux-debug` | 本模板 Debug preset |
| `linux-release` | 本模板 Release preset |
| `debug` | 也可以，但不如 `linux-debug` 清楚 |
| `gcc-debug` | 如果区分编译器，可以这样命名 |
| `clang-debug` | 如果添加 Clang preset，可以这样命名 |

要求：

1. 同一个数组里不能重名。
2. `buildPresets` 里的 `configurePreset` 要引用这里的名字。
3. 命令行使用的就是这个名字。

示例：

```bash
cmake --preset linux-debug
```

### `displayName`

```json
"displayName": "Linux Debug"
```

给人看的名字，主要用于 IDE 显示，例如 VSCode CMake Tools 的 preset 列表。

可以填什么：

| 写法 | 说明 |
|:---|:---|
| `Linux Debug` | 清晰 |
| `Linux Release` | 清晰 |
| `Debug` | 可以，但不区分平台 |
| `Fedora GCC Debug` | 如果 preset 很多，可以写更具体 |

这个字段不影响实际构建逻辑。

### `generator`

```json
"generator": "Ninja"
```

指定 CMake 要生成什么构建系统文件。

Linux 常见可选值：

| 值 | 说明 |
|:---|:---|
| `Ninja` | 推荐，速度快，输出干净，适合 VSCode CMake Tools |
| `Unix Makefiles` | 传统 Makefile 工作流 |
| `Ninja Multi-Config` | 多配置 Ninja，可同时管理 Debug / Release |

本模板只支持 Linux，并且使用 Ninja，所以固定写：

```json
"generator": "Ninja"
```

如果使用 `Unix Makefiles`，构建命令仍然可以用：

```bash
cmake --build --preset linux-debug
```

但底层就不是 Ninja 了。

### `binaryDir`

```json
"binaryDir": "${sourceDir}/build/linux-debug"
```

指定构建目录。

可以填什么：

| 写法 | 说明 |
|:---|:---|
| `${sourceDir}/build/linux-debug` | 推荐，构建产物留在项目内的 build 子目录 |
| `${sourceDir}/build/debug` | 也可以，但不如 `linux-debug` 清楚 |
| `/tmp/my-build` | 可以放到项目外，但不适合作为模板默认值 |

常见宏：

| 宏 | 含义 |
|:---|:---|
| `${sourceDir}` | 项目源码根目录 |
| `${presetName}` | 当前 preset 名称 |
| `${sourceParentDir}` | 源码根目录的父目录 |

也可以写成：

```json
"binaryDir": "${sourceDir}/build/${presetName}"
```

这样 `linux-debug` 会自动生成到：

```text
build/linux-debug
```

本模板显式写出目录，是为了让初学时更直观。

### `cacheVariables`

```json
"cacheVariables": {
  "CMAKE_BUILD_TYPE": "Debug",
  "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
  "CMAKE_INSTALL_PREFIX": "${sourceDir}/install/linux-debug"
}
```

`cacheVariables` 等价于命令行中的 `-D` 参数。

例如：

```json
"CMAKE_BUILD_TYPE": "Debug"
```

等价于：

```bash
-DCMAKE_BUILD_TYPE=Debug
```

## 常用 cache 变量

### `CMAKE_BUILD_TYPE`

```json
"CMAKE_BUILD_TYPE": "Debug"
```

单配置生成器中使用的构建类型。`Ninja` 和 `Unix Makefiles` 都属于常见的单配置生成器。

可以填什么：

| 值 | 作用 |
|:---|:---|
| `Debug` | 调试构建，通常带 `-g`，优化较少 |
| `Release` | 发布构建，通常开启优化 |
| `RelWithDebInfo` | 优化 + 调试信息 |
| `MinSizeRel` | 优化体积 |

常用选择：

| 场景 | 推荐 |
|:---|:---|
| 写代码、调试断点 | `Debug` |
| 发布、性能测试 | `Release` |
| 性能测试但还想保留符号 | `RelWithDebInfo` |
| 嵌入式或体积敏感 | `MinSizeRel` |

### `CMAKE_EXPORT_COMPILE_COMMANDS`

```json
"CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
```

是否生成 `compile_commands.json`。

可以填什么：

| 值 | 作用 |
|:---|:---|
| `ON` | 生成 `compile_commands.json` |
| `OFF` | 不生成 |

`compile_commands.json` 记录每个源文件真实的编译命令。`clangd`、VSCode、很多静态分析工具都会用它来获得 include 路径、宏定义、编译标准等信息。

建议模板里固定开启：

```json
"CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
```

### `CMAKE_INSTALL_PREFIX`

```json
"CMAKE_INSTALL_PREFIX": "${sourceDir}/install/linux-debug"
```

指定安装目录。

可以填什么：

| 写法 | 说明 |
|:---|:---|
| `${sourceDir}/install/linux-debug` | 本模板 Debug 安装目录 |
| `${sourceDir}/install/linux-release` | 本模板 Release 安装目录 |
| `/usr/local` | 系统级安装路径，需要权限，不适合作为模板默认 |
| `/opt/my_project` | 系统级或部署路径 |

本模板把安装目录放在项目内，是为了方便学习和删除。

安装后运行：

```bash
./install/linux-debug/bin/cmake_template
```

## `buildPresets`

`buildPresets` 是"构建阶段"的 preset。

构建阶段对应命令：

```bash
cmake --build --preset linux-debug
```

本模板写法：

```json
"buildPresets": [
  {
    "name": "linux-debug",
    "configurePreset": "linux-debug"
  },
  {
    "name": "linux-release",
    "configurePreset": "linux-release"
  }
]
```

### `name`

```json
"name": "linux-debug"
```

构建 preset 名称。

一般建议和对应的 configure preset 同名，这样命令比较一致：

```bash
cmake --preset linux-debug
cmake --build --preset linux-debug
```

### `configurePreset`

```json
"configurePreset": "linux-debug"
```

表示这个 build preset 使用哪个 configure preset 生成的构建目录。

可以填什么：

| 值 | 说明 |
|:---|:---|
| `linux-debug` | 使用 Debug 构建目录 |
| `linux-release` | 使用 Release 构建目录 |

必须对应 `configurePresets` 中已经存在的 `name`。

## 常见可扩展字段

初学时可以先不用，但以后可能会见到。

### `description`

给 preset 添加更长的说明。

```json
"description": "Debug build for Linux using Ninja"
```

### `hidden`

隐藏某个 preset，不直接给用户选择，通常用于继承。

```json
"hidden": true
```

### `inherits`

继承另一个 preset，减少重复。

```json
{
  "name": "linux-debug",
  "inherits": "linux-base",
  "cacheVariables": {
    "CMAKE_BUILD_TYPE": "Debug"
  }
}
```

### `environment`

设置环境变量。

```json
"environment": {
  "CC": "/usr/bin/gcc",
  "CXX": "/usr/bin/g++"
}
```

一般不建议模板默认写死编译器，除非你的项目明确要求某个工具链。

### `targets`

指定默认构建哪些 target。

```json
"targets": ["cmake_template"]
```

如果不写，默认构建全部 target。

### `jobs`

指定并行编译数量。

```json
"jobs": 8
```

不写时由底层构建工具决定。

## 为什么没有 `installPresets`

很多人会猜测可以写：

```bash
cmake --install --preset linux-debug
```

但常见 CMake preset 文件并不使用这个根字段。这个模板采用更通用的安装命令：

```bash
cmake --install build/linux-debug
```

安装命令需要传入构建目录，因为安装规则是在 configure 阶段生成到构建目录里的。

## VSCode CMake Tools 工作流

安装 VSCode 的 CMake Tools 扩展后，它会自动识别 `CMakePresets.json`。

推荐流程：

1. 打开项目目录。
2. 选择 Configure Preset：`linux-debug` 或 `linux-release`。
3. 执行 Configure。
4. 执行 Build。
5. 需要安装时，在终端执行 `cmake --install build/linux-debug`。

这样不需要手写 `.vscode/tasks.json`。

## VSCode F5 调试工作流

如果要按 F5 调试，还需要 Microsoft C/C++ 扩展，因为 GDB 调试由它提供。

这个模板保留一个极简 `.vscode/launch.json`：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug CMake Target",
      "type": "cppdbg",
      "request": "launch",
      "program": "${command:cmake.launchTargetPath}",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb",
      "miDebuggerPath": "/usr/bin/gdb",
      "setupCommands": [
        {
          "description": "Enable pretty printing",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        }
      ]
    }
  ]
}
```

这里最关键的是：

```json
"program": "${command:cmake.launchTargetPath}"
```

它不是写死：

```text
./install/linux-debug/bin/cmake_template
```

而是让 CMake Tools 返回当前选择的 launch target 路径。通常是：

```text
build/linux-debug/src/cmake_template
```

因此日常调试不需要每次 install。

### F5 前需要做什么

推荐顺序：

1. 选择 Configure Preset：`linux-debug`。
2. 执行 Configure。
3. 执行 Build。
4. 选择 Launch Target：`cmake_template`。
5. 按 F5。

如果你改了代码，只需要重新 Build 后再 F5。

如果你改了 `CMakeLists.txt`、新增源文件、改依赖，建议先 Configure，再 Build，再 F5。

### 为什么不写 `preLaunchTask`

旧模板常用 `.vscode/tasks.json` 写一长串命令：

```text
cmake ..
make install
source setup.bash
run app
```

这个模板不这样做。原因是构建已经交给 CMake Tools 和 preset 管理，`launch.json` 只负责"启动调试器"。

这样职责更清楚：

| 文件 | 负责 |
|:---|:---|
| `CMakePresets.json` | configure/build 参数 |
| `.vscode/launch.json` | F5 调试 |
| `CMakeLists.txt` | target、依赖、安装规则 |

### 需要安装 gdb

Ubuntu/Debian：

```bash
sudo apt install gdb
```

Fedora：

```bash
sudo dnf install gdb
```

## 常见错误

### 忘记安装 Ninja

报错可能类似：

```text
CMake was unable to find a build program corresponding to "Ninja"
```

解决：

```bash
sudo apt install ninja-build
```

或：

```bash
sudo dnf install ninja-build
```

### preset 名称写错

错误命令：

```bash
cmake --preset debug
```

如果文件里没有 `debug` 这个 preset，就会失败。

查看可用 preset：

```bash
cmake --list-presets
```

### build 前没有 configure

如果第一次直接运行：

```bash
cmake --build --preset linux-debug
```

可能会因为构建目录还没有生成而失败。

正确顺序：

```bash
cmake --preset linux-debug
cmake --build --preset linux-debug
```
