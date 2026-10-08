---
title: "Windows环境准备"
---

## 环境介绍
本教程环境介绍：

1.  系统：Windows 10 LSTC 2021

2.  系统内核：Windows NT

3.  架构：X86_64(amd64)

其他Windows 10 及以上环境也可以。

## 安装各种软件与环境

### 安装CubeMX

### 安装VScode

### 安装openOCD

#### 安装

官方链接：https://openocd.org/pages/getting-openocd.html

下载链接：https://github.com/xpack-dev-tools/openocd-xpack/releases

然后下载`xpack-openocd-版本号-win32-x64.zip`：

下载完后解压到C盘，比如：`C:\xpack-openocd-0.12.0-7`。

#### 添加环境变量

```text
设置
→ 系统
→ 系统信息
→ 高级系统设置
→ 环境变量
```

![alt text](image.png)

![alt text](image-1.png)

编辑 → 新建，把`C:\xpack-openocd-0.12.0-7\bin`加上

![alt text](image-2.png)

保存后，打开`powershell`：

```powershell
openocd --version
```

![alt text](image-3.png)