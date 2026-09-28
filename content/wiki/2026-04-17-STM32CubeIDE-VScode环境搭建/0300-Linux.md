---
title: "Linux"
---

### 环境介绍
本教程环境介绍：

1.  系统：Fedora 43 KDE Edition Linux

2.  系统内核：Linux 6.19.12-200.fc43.x86_64

3.  架构：X86_64(amd64)

其他Linux环境也可以。

### 安装各种软件与环境

#### 安装CubeMX
![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image9.webp)

下载地址：

https://www.st.com.cn/zh/development-tools/stm32cubemx.html

**推荐下载最新版本（此时最新版本是6.15.0）**

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image10.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image11.webp)

解压出来

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image12.webp)

用root权限打开这个软件`SetupSTM32CubeMX-6.15.0`（不建议）

```bash
sudo ./SetupSTM32CubeMX-6.15.0
```

（更建议：也可以不用root权限打开，但在选择安装路径的时候注意以下，自己改成`/home/xxx`里的某个文件夹里）

```bash
./SetupSTM32CubeMX-6.15.0
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image13.webp)

在新弹出的界面一直点下一步就行，安装结束后出现如下图就成功了。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image14.webp)

`/usr/local/STMicroelectronics/STM32Cube/STM32CubeMX`（如果你不是root权限安装的，你的路径不是这个，需要选择对应的路径）进入这个文件夹，然后打开终端输入

```cpp
./STM32CubeMX
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image15.webp)

点击Help

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image16.webp)

选`Manage embedded software packages`，把STM32F1，F4，H7的第一个最新的固件勾上。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image17.webp)

点install

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image18.webp)

登陆上账号

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image19.webp)

然后等下载和安装完

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image20.webp)

下载好就行了。

接下来可以把CubeMX应用配置一个桌面快捷方式等可以快速打开，教程详见[Vinci机器人队Linux入门教程](/wiki/2024-03-30-linux-jiao-cheng/0600-qi-ta-ke-xuan-pei-zhi#appimage)的Appimage章节，可以用ctrl+F快速定位该章节。

桌面快捷方式如下：

（如果你不是root权限安装的，你的路径不是这个`/usr/local/STMicroelectronics/STM32Cube/STM32CubeMX/`，需要选择对应的路径）

```bash
[Desktop Entry]
Name=STM32CubeMX
Exec=/usr/local/STMicroelectronics/STM32Cube/STM32CubeMX/STM32CubeMX
Icon=/usr/local/STMicroelectronics/STM32Cube/STM32CubeMX/help/STM32CubeMX.png
Type=Application
Categories=Development;Electronics;Embedded;
Comment=STM32CubeMX configuration and code generation tool
Terminal=false
```

根据教程做，就可以实现这种效果啦。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image21.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image22.webp)

#### 安装VScode
https://code.visualstudio.com/Download

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image23.webp)

如果是debian系下载deb,如果是rhel系下载rpm.

下载完之后，点击浏览器，找到这个安装包的文件夹，并在该路径打开终端。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image24.webp)

Debian系：输入`sudo apt install ./code`然后按`tab`按键补齐文件名，回车。

RHEL系：输入`sudo dnf install ./code`然后按`tab`按键补齐文件名，回车。

例如补齐后的：

```bash
sudo dnf install ./code-1.102.1-1752598767.el8.x86_64.rpm
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image25.webp)

然后打开VScode，在终端输入下面的命令

```bash
code
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image26.webp)


然后可以配置一个环境单独给CubeIDE插件使用，避免和默认环境冲突。

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776420070033.webp)

进行一些设置，按我的来就可以

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776420416232.webp)

选中STM32

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776420490177.webp)



然后安装一些插件

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image27.webp)

找到下面这个`STM32CubeIDE for Visual Studio Code`插件安装

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776420601935.webp)

右边弹这个提示要选择安装（要有良好的*科学网络*）

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776420688705.webp)


紧接着会进行一些环境的安装

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776420710467.webp)

也可以再安装一些其他的插件，比如Codex等插件
这些看你自己啦

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776420944068.webp)


### 工程创建与测试

#### 使用CubeMX创建工程
点击进入单片机挑选的按钮

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image60.webp)

搜索对应芯片，并双击对应芯片选项。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image61.webp)

进行一些配置，以下都是很基础的东西，你在看这个视频前肯定都会了

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image62.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image63.webp)

随便开一个IO用来测试，比如LED的GPIO

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image64.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image65.webp)

FreeRTOS也要配置一下。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image66.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image67.webp)

这些文件夹也要配置好，最后Toolchain选择CMake,编译器选择GCC(6.14.1及之前没有选择编译器这个选项很正常)

（但是CubeMX6.15.0有bug,这个选择GCC编译器并没有用，还需要后续自己手动选择编译器，以后可能会修复这个bug.）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image68.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image69.webp)


#### 对工程进行配置与编译

在工程文件夹打开终端

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776421629435.webp)

```bash
code .
```

打开VScode后记得切到`STM32·的配置

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776421689049.webp)

选择这里的Yes进行配置CMake预设

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776421794873.webp)

一般选Debug即可

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776421850499.webp)

找个C语言的代码文件打开，然后右下角会提示安装一个C/C++插件，这个可以安装也可以不安装，他也带代码提示，但是他的代码提示对比自带的clangd简直是弱爆了，如果你是新手，你不会设置代码提示，建议按我下面的操作来，直接别装这个插件。

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776422028364.webp)

你可以测试一下代码提示，是不是很强。

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776422393367.webp)

编译的话，图中的这俩build都可以

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776423543189.webp)

#### 移植作者tungchiahui的标准C/C++工程模板

用git clone命令克隆仓库:https://github.com/tungchiahui/STM32HAL_CMake_CPP_Template

```bash
git clone https://github.com/tungchiahui/STM32HAL_CMake_CPP_Template.git
```

把仓库里的 **所有文件与文件夹（除了`.git`以外）** 复制到我们的STM32工程的目录里。

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776427498924.webp)

然后打开applications文件夹，在Src和Inc文件夹分别创建led_task.cpp和led_task.h，内容分别如下:

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image91.webp)

main.c:

这里面加上如下：

```c
#include "cpp_interface.h"
```

在main函数里找合适位置加上如下函数：
一般裸机的话放在`while(1)`**上面**。
RTOS放在main()里仅靠`while(1)`的那几个RTOS相关的开启函数的**上面**。

```c
cpp_main();
```


led_task.cpp:

```cpp
#include "led_task.h"
#include "cmsis_os.h"
#include "stm32f1xx_hal.h" 

GPIO_PinState pinstate = GPIO_PIN_RESET;

extern "C"
void StartDefaultTask(void *argument)
{
  for(;;)
  {
    HAL_GPIO_TogglePin(GPIOC,GPIO_PIN_13);
    osDelay(500);
  }
}
```

led_task.h:

```cpp
#ifndef LED_TASK_H
#define LED_TASK_H

#include "cpp_interface.h"


#endif

```

然后打开`cmake/user`文件夹下的`CMakeLists.txt`，把刚才新建的led_task.cpp添加上去。

详细介绍（可以不看）：这里的`cmake/stm32cubemx`下的`CMakeLists.txt`是被CubeMX管理的，你重新用CubeMX生成新代码后，这个文件里的东西会被覆盖。而工作区根目录下的`CMakeLists.txt`是不会被重新覆盖的，而且给我们留了一些区域加源文件和头文件，但是这样会让这个文件太过于嘈杂。所以我们选择新建一个user文件夹，然后在这里面弄一个`CMakeLists.txt`，再用顶层`CMakeLists.txt`去加载这个子`CMakeLists.txt`，这个子`CMakeLists.txt`方便咱们修改，文件结构也更加明显。（这些都不需要咱们自己创建，我已经给创建到**模板**里了，你在上面复制的时候已经复制过来了）

像下图这样加上cpp文件。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image92.webp)

然后要去最顶层的CMakeLists.txt里加上这句话来引用我们自己的CMakeLists.txt。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2025/07/18/image93.webp)

```cmake
# Add USER generated sources
add_subdirectory(cmake/user)
```

在最顶上，也要把编程语言的标准改一下（这一步在`README.md`里写了）
（注意：Ubuntu22.04默认最高支持C++20，Ubuntu24.04默认最高支持C++23，Fedora默认最高支持C++最高的标准）

```cmake
# Setup compiler settings
set(CMAKE_C_STANDARD 11 CACHE STRING "C language standard")
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)

set(CMAKE_CXX_STANDARD 20 CACHE STRING "C++ language standard")
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)
```

大功告成，编译一次试试。可以看到下图，那些新加的文件都编译上了。

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776429160105.webp)

#### 下载程序到板子

下载之前首先要先配置：

##### 配置调试器

按下图的来点击，你看看你是什么debugger,你就选哪个。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1790606954240-b80cbe26.webp)

###### ST-Link

先更新下STlink的驱动：

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1790607581587-4d65b468.webp)

点击`install ST-Link udev rules`，一般VScode右下角会弹一个框进行下载：

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1790607134879-933d161f.webp)

如果这里看不到对应设备，遇到权限问题，请往下看

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1790606862537-e94b529c.webp)

###### JLink

先安装jlink-gdbserver的bundle，如下图所示：

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776431910132.webp)

如果这里看不到对应设备，遇到权限问题，请往下看

###### Linux 下 USB 权限问题（如果你遇到的话，你解决不了就让AI来解决，AI几秒钟就给你把事办了）

如果调试器插上后能在 `lsusb` 中看到，但 VS Code 找不到设备，或者调试输出中出现下面的报错，通常是当前用户没有访问 USB 设备节点的权限：

```text
libusb couldn't open USB device /dev/bus/usb/001/008, errno=13
libusb requires write access to USB device nodes
ST-Link enumeration failed
```

先在终端运行 `lsusb`，确认调试器已被系统识别。报错中的 `/dev/bus/usb/001/008` 只是示例，实际路径以你自己的输出为准。可以用 `ls -l` 查看该节点的权限：

```bash
lsusb
ls -l /dev/bus/usb/001/008
```

**使用 ST-Link：**先按上文点击插件中的 `install ST-Link udev rules`。如果仍然报权限错误，检查规则是否装到了系统中：

```bash
find /etc/udev/rules.d /usr/lib/udev/rules.d -maxdepth 1 -iname '*stlink*.rules' -print 2>/dev/null
```

如果没有找到规则，可以从 ST 官方的 [STSW-LINK007 下载页](https://www.st.com/en/development-tools/stsw-link007.html)获取安装包。解压后进入 `AllPlatforms/StlinkRulesFilesForLinux`，按其中的 `readme.txt` 安装与你的发行版对应的 udev 规则包。在这个目录中，Fedora 使用 `.rpm` 包，Ubuntu/Debian 使用 `.deb` 包：

```bash
# Fedora：只运行这一行
sudo dnf install ./st-stlink-udev-rules-*.rpm

# Ubuntu/Debian：只运行这一行
sudo apt install ./st-stlink-udev-rules-*.deb
```

如果下载的包名不同，以实际文件名和随包说明为准。ST 的[发行说明](https://www.st.com/resource/en/release_note/dm00107009-firmware-upgrade-for-st-link-st-link-v2-st-link-v2-1-and-stlink-v3-boards-stmicroelectronics.pdf)也说明了 Linux 下需要安装相应的 ST-Link USB 访问规则。

**使用 J-Link：**上文安装的 `jlink-gdbserver` bundle 用于启动调试服务；USB 权限还需要 SEGGER 的 udev 规则。安装 [SEGGER J-Link Software and Documentation Pack](https://www.segger.com/downloads/jlink/) 中对应发行版的 `.rpm` 或 `.deb` 包时，会一并安装规则。如果使用解压版，则按照包内 `README.txt`，将 `99-jlink.rules` 复制到 `/etc/udev/rules.d/`。具体方法见 [SEGGER 的 Linux 排查说明](https://kb.segger.com/J-Link_Troubleshooting)。

安装规则后，运行：

```bash
sudo udevadm control --reload-rules
```

然后**拔下调试器再插回去**，重新运行 `lsusb`，用新的总线号和设备号查看 `/dev/bus/usb/...` 节点。可以将实际路径代入下方命令，检查当前用户能否读写：

```bash
test -r /dev/bus/usb/001/008 && test -w /dev/bus/usb/001/008 && echo 'USB 设备节点可读写'
```

拔插后设备号可能变化，不要照抄示例中的 `001/008`。确认权限后，回到 VS Code 重新选择调试器。如果 `lsusb` 根本看不到调试器，应先检查 USB 线、接口和供电；这不是 udev 权限问题。

这里检查的是 `/dev/bus/usb/...`。如果另外还要打开板载虚拟串口 `/dev/ttyACM*`，那是串口设备的权限，需要分别排查。

##### 进行调试：

如果你是STLink应该是下图所示：

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776431995915.webp)

如果你是JLink应该是下图所示：

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776432039560.webp)


然后会出现这个条，他会下载程序到板子
![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776432143180.webp)

然后就成功下载了程序并进入了Debug

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1776432211211.webp)

按照下图所示，把你要监视的变量输入到最顶上的框里，就可以加入到实时监视框里了。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1790608511630-b448c1d8.webp)