---
title: "Linux环境准备"
---

## 环境介绍
本教程环境介绍：

1.  系统：Fedora 43 KDE Edition Linux

2.  系统内核：Linux 6.19.12-200.fc43.x86_64

3.  架构：X86_64(amd64)

其他Linux环境也可以。

## 安装各种软件与环境

### 安装CubeMX
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

### 安装VScode
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


### 安装openOCD

#### 安装

我们主要用openocd来进行debug,这样才支持LiveWatch,而pyocd暂时不支持。

```bash
# Debian系（如Ubuntu）
sudo apt install openocd


# 红帽系（如Fedora）
sudo dnf install openocd
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1790778562287-aa09e54e.webp)


#### 安装udev

先要知道：

```text
/etc/udev/rules.d/
    → 你自己手工配置的规则，现在基本清空

/usr/lib/udev/rules.d/
    → Linux 软件包提供的系统规则
       └── 60-openocd.rules
```

查看openocd是否在系统里自动装了udev了：

```bash
# debian系（ubuntu）
dpkg -L openocd | grep -E 'udev|rules'

# 红帽系（fedora）
rpm -ql openocd | grep -E 'udev|rules'
```

如下图，说明openocd安装过udev了。（这比pyocd方便多了）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/04/17/1790820989855-95fc4922.webp)




