---
title: "Docker命令学习"
---

### 参考文档
https://www.runoob.com/docker/docker-tutorial.html

### 常用命令
常用的标红了，偶尔用的标绿了，其他了解就行。

| 命令 | 描述 | 示例 |
|:---|:---|:---|
| docker run | 创建并启动一个新的容器。 | docker run -it ubuntu bash |
| docker build | 通过指定的 Dockerfile 创建一个新的镜像。 | docker build -t myimage . |
| docker pull | 从 Docker 仓库拉取镜像。 | docker pull ubuntu |
| docker push | 将本地镜像推送到 Docker 仓库。 | docker push myimage |
| docker stop | 停止一个正在运行的容器。 | docker stop container_id |
| docker start | 启动一个已经存在的容器。 | docker start container_id |
| docker restart | 重新启动容器。 | docker restart container_id |
| docker ps | 列出当前正在运行的容器。 | docker ps |
| docker rm | 删除一个或多个停止的容器。 | docker rm container_id |
| docker exec | 在一个正在运行的容器中执行命令。 | docker exec -it container_id bash |
| docker logs | 查看容器的日志输出。 | docker logs container_id |
| docker images | 列出本地所有镜像。 | docker images |
| docker rmi | 删除一个或多个镜像。 | docker rmi myimage |
| docker network | 管理 Docker 网络。 | docker network ls |
| docker volume | 管理 Docker 数据卷。 | docker volume ls |
| docker-compose up | 启动 docker-compose.yml 中定义的所有服务。 | docker-compose up |
| docker-compose down | 停止并移除 docker-compose.yml 中定义的所有服务及其相关资源。 | docker-compose down |
| docker info | 显示 Docker 系统的详细信息。 | docker info |
| docker stats | 查看正在运行的容器的实时资源使用情况（CPU、内存等）。 | docker stats |
| docker inspect | 查看容器或镜像的详细信息（JSON 格式）。 | docker inspect container_id |
| docker save | 将一个镜像保存为 tar 文件。 | docker save -o myimage.tar myimage |
| docker load | 从 tar 文件中加载镜像。 | docker load -i myimage.tar |
| docker tag | 为镜像添加标签（tag）。 | docker tag myimage myimage:v1 |
| docker buildx build | 使用 Buildx 构建多架构镜像。 | docker buildx build -t myimage . |
| docker buildx create | 创建一个新的 Buildx 构建实例。 | docker buildx create --use |
| docker buildx ls | 列出所有可用的 Buildx 构建实例。 | docker buildx ls |
| docker buildx use | 设置当前的 Buildx 构建实例。 | docker buildx use mybuilder |
| docker buildx bake | 使用 Bake 文件批量构建镜像。 | docker buildx bake -f bake.hcl |
| docker buildx build --push | 构建镜像并推送到镜像仓库。 | docker buildx build --push -t myimage . |
| docker buildx build --platform | 构建镜像并为多个平台生成支持。 | docker buildx build --platform linux/amd64,linux/arm64 -t myimage . |

#### **run命令的参数（非常重要）**
| 参数/配置 | 功能说明 | 重要性与参考依据 |
|:---|:---|:---|
| --name=ros_jazzy_opencv411_cuda128_cudnn971_noble | 指定容器名称，便于后续管理 | 替代随机生成的容器名。 |
| --gpus all | 允许容器访问宿主机所有GPU资源，需NVIDIA驱动支持 | 用于CUDA加速等GPU依赖任务。 |
| -e NVIDIA_DRIVER_CAPABILITIES=all | 启用NVIDIA驱动的全部功能（如CUDA、图形渲染） | 确保容器内GPU功能完整67。 |
| -dit | 组合参数：- -d：后台运行容器（Detached模式）- -i：保持标准输入（STDIN）开放- -t：分配伪终端（TTY） | 允许容器在后台运行并支持交互操作。 |
| --privileged | 赋予容器完全主机权限（可访问设备、内核模块等） | 用于需要直接操作硬件的场景（如访问USB设备），但存在安全风险。 |
| -v /dev:/dev \ | 映射宿主机的/dev目录 | 用于需要直接操作硬件的场景（如访问USB设备），这样可以热插拔，容器会直接看到宿主机生成的设备别名，但存在安全风险。 |
| --net=host | 共享宿主机网络命名空间（容器使用宿主机IP和端口） | 简化网络配置，无NAT，这样的话，网络效率更高，局域网设备更容易发现。 |
| --group-add audio--group-add video--group-add dialout | 将容器用户加入宿主机用户组：- audio：音频设备访问- video：视频设备访问- dialout：串口设备访问 | 避免权限问题（如避免无法调用摄像头、麦克风）。 |
| -e DISPLAY=$DISPLAY-e XAUTHORITY=/home/tungchiahui/.Xauthority-e WAYLAND_DISPLAY-e XDG_RUNTIME_DIR-e QT_QPA_PLATFORM=xcb | 配置图形显示环境：- 绑定宿主机显示接口（X11或Wayland）- 设置GUI应用渲染后端 | 支持容器内运行图形界面应用（如OpenCV可视化）。 |
| -v /tmp/.X11-unix:/tmp/.X11-unix:rw-v /dev/dri:/dev/dri | 挂载宿主机图形设备：- X11套接字目录- 直接渲染管理器（DRI）设备 | 实现容器内图形显示。 |
| -v $HOME/.Xauthority:/home/tungchiahui/.Xauthority:ro | 挂载X11认证文件（只读） | 确保容器有权连接宿主机显示服务。 |
| -v /run/user/1000/wayland-0-v /run/user/1000 | 挂载Wayland显示协议相关目录 | 支持Wayland协议的图形显示。 |
| --ulimit nofile=1024:524288 | 设置进程**最大可打开文件数（nofile）**的方式，用于控制容器或进程运行时的文件句柄数量限制。--ulimit <限制类型>=<软限制>:<硬限制> | 如果默认限制太小，可能会出现 "too many open files" 的错误。所以在容器运行或系统服务启动时，需要调大这个值。--ulimit nofile=4096:65536 |
| -v /home/tungchiahui:/home/tungchiahui | 挂载宿主机用户目录到容器内同名路径 | 实现宿主机与容器间文件共享（如代码、数据持久化）。 |
| -w /home/tungchiahui | 设置容器启动后的默认工作目录 | 直接进入项目路径，方便执行命令2324。 |
| tungchiahui/ros-opencv:jazzy-411-cuda128-cudnn971-noble | 镜像名称指定镜像及标签，包含：- ROS 2 Jazzy- OpenCV 4.11- CUDA 12.8- cuDNN 9.7.1 | 提供预配置的深度学习与机器人开发环境。 |


**我已经写好了一个非常完备的运行命令了,你直接用我的就可以.**

下方这条命令一定要在普通用户下运行，不要在root用户下运行，其实加不加`sudo`加不加`sudo -E`都无所谓。

用户已经被加到docker组了，不用`sudo`也行跑，其次，`sudo`运行的话，你的`$HOME`变量也不会变，更何况加上-E的话，这样你的`$HOME`更不可能变了。

```bash
sudo docker run --name=ros_opencv_cuda \
--gpus all \
-e NVIDIA_DRIVER_CAPABILITIES=all \
-e DISPLAY=$DISPLAY \
-dit \
--privileged \
--net=host \
--group-add audio \
--group-add video \
--group-add dialout \
-e XAUTHORITY=$HOME/.Xauthority \
-e WAYLAND_DISPLAY=$WAYLAND_DISPLAY \
-e XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR \
-e QT_QPA_PLATFORM=xcb \
-v /tmp/.X11-unix:/tmp/.X11-unix:rw \
-v /dev:/dev \
-v $HOME/.Xauthority:$HOME/.Xauthority:ro \
-v /run/user/$(id -u)/wayland-0:/run/user/$(id -u)/wayland-0 \
-v /run/user/$(id -u):/run/user/$(id -u) \
-v $HOME:$HOME \
-w $HOME \
tungchiahui/ros-opencv:humble-411-cuda128-cudnn970-jammy
```

**你需要自己手动修改的内容：**

1.  `NVIDIA_DRIVER_CAPABILITIES=all` `--gpus all`没有英伟达显卡请注释。

2.  \--name后面请自己为容器起名。

3.  最后一行仓库名称请你自己找对应的镜像填上。

4.  ROS1在Fedora发行版下会爆内存，需要添加上下面这个参数，如果你不是Fedora和ROS1,**请不要加**。(现在其实也很少用ROS1了,ROS1已经逐步淘汰了)

```bash
--ulimit nofile=1024:524288 \
```

5.  如果想用当前用户登陆容器,可以加上下面这几条,但非常非常**不建议**.

```bash
--user $(id -u):$(id -g) \
-v /etc/passwd:/etc/passwd:ro \
-v /etc/group:/etc/group:ro \
```


#### save与load

> 众所周知，因为不可抗因素，dockerhub已于2024年5月被🇨🇳封杀，所以咱们无法直接拉取镜像。

有一种本地的办法，就是把镜像打成压缩包，再扔到另一台电脑上去让docker读取。

打包：

```bash
docker save -o myimage.tar myimage
```

读取：

```bash
docker load -i myimage.tar
```


#### 第三方的dockerhub库（并不是dockerhub mirror库）

可以把自己镜像上传到国内一些镜像库（images hub，不是mirror）里。

##### 腾讯的TCR

###### 环境搭建

这是一个国内docker镜像托管站，可以在中国大陆，中国香港，海外各主要城市托管你的镜像。建议使用中国香港服务器。

TCR个人版：

打开网站：https://console.cloud.tencent.com/tcr

可以看到`docker login`要用的用户名和密码。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795593774544-1c62cda2.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795593951621-e950a940.webp)

新建一个命名空间，类似dockerhub的username一样。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795593666146-3aa84e03.webp)

登陆

```bash
# 中国大陆TCR
docker login ccr.ccs.tencentyun.com

# 中国香港TCR（推荐）
docker login hkccr.ccs.tencentyun.com
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795594000136-f7b775d9.webp)

登陆成功。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795594010602-96e437c3.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795595924634-88346e05.webp)

###### 推送方式一（全新推送）

这种方式就是直接将本地的 Docker 镜像推送到腾讯云 TCR，不需要经过 Docker Hub。

同样以中国香港 TCR 为例。

首先确保已经创建好命名空间，并且登录成功：

```bash
docker login hkccr.ccs.tencentyun.com
```

**1. 本地已经有 Docker 镜像**

如果你已经通过 `docker build` 构建好了镜像，可以直接修改镜像标签，再推送到 TCR。

先查看本地镜像：

```bash
docker images
```

比如我的镜像为 `tungchiahui/ros:humble-curc`，想推送到 TCR，就需要先给它添加一个 TCR 的镜像标签：

```bash
docker tag \
  tungchiahui/ros:humble-curc \
  hkccr.ccs.tencentyun.com/tungchiahui/ros:humble-curc
```

然后推送：

```bash
docker push hkccr.ccs.tencentyun.com/tungchiahui/ros:humble-curc
```

推送完成后，打开 TCR 控制台，就可以看到你的镜像了，不需要提前手动创建仓库。

注意：这种方式适合普通的单架构本地镜像。如果是 `amd64` 和 `arm64` 双架构镜像，建议使用下面的 Buildx 方法。

**2. 从 Dockerfile 直接构建并推送（推荐）**

如果你有 Dockerfile，可以直接构建镜像并推送到 TCR。

先进入 Dockerfile 所在目录：

比如我的：

```bash
cd ~/UserFolder/MySource/ros-docker/ROS2_Humble_Nav2
```

普通单架构构建：

```bash
docker build \
  -t hkccr.ccs.tencentyun.com/tungchiahui/ros:humble-curc \
  .
```

构建完成后推送：

```bash
docker push hkccr.ccs.tencentyun.com/tungchiahui/ros:humble-curc
```

上面的方式默认构建当前设备对应的 CPU 架构。

如果想同时构建 `amd64` 和 `arm64`，可以使用 Docker Buildx。

**3. 使用 Buildx 构建并推送多架构镜像（推荐）**

这种方式比较适合 ROS2 开发环境，因为同一个镜像可以同时给普通 x86 工控机和 ARM 开发板使用。

首先查看 Buildx：

```bash
docker buildx ls
```

如果还没有创建支持多架构构建的 Builder，可以新建一个：

```bash
docker buildx create \
  --name multiarch_builder \
  --driver docker-container \
  --use
```

如果已经有 `multiarch_builder`，直接选择即可：

```bash
docker buildx use multiarch_builder
```

初始化 Builder：

```bash
docker buildx inspect --bootstrap
```

注意：Builder 还需要具备对应架构的构建能力，比如通过 QEMU 或原生 ARM 节点构建 `arm64`。

接下来直接构建并推送：

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t hkccr.ccs.tencentyun.com/tungchiahui/ros:humble-curc \
  --push \
  .
```

参数说明：

| 参数 | 作用 |
|---|---|
| `--platform` | 指定构建的 CPU 架构 |
| `linux/amd64` | 普通 Intel/AMD x86-64 设备 |
| `linux/arm64` | ARM64 设备，例如树莓派 5 |
| `-t` | 指定镜像名称和 Tag |
| `--push` | 构建完成后直接推送到镜像仓库 |
| `.` | 使用当前目录作为 Docker 构建上下文 |

这里的 `--push` 会在构建完成后直接上传到 TCR，不需要再执行 `docker push`。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795598033176-e37d4842.webp)

**4. 验证推送结果**

推送完成后，可以查看 TCR 上的镜像信息：

```bash
docker buildx imagetools inspect \
  hkccr.ccs.tencentyun.com/tungchiahui/ros:humble-curc
```

如果是双架构镜像，正常可以看到：

```text
linux/amd64
linux/arm64
```

之后就可以在其他机器上直接拉取：

```bash
docker pull hkccr.ccs.tencentyun.com/tungchiahui/ros:humble-curc
```

Docker 会自动选择适合当前设备 CPU 架构的镜像。

**注意：**

- TCR 个人版只需要先创建 Namespace，首次推送时可以自动创建镜像仓库。
- 中国大陆 TCR 使用 `ccr.ccs.tencentyun.com`，中国香港 TCR 使用 `hkccr.ccs.tencentyun.com`，不要弄混，否则可能出现 `403 Forbidden`。
- `docker push` 不会自动生成其他 CPU 架构的镜像，多架构构建推荐使用 `docker buildx build --push`。
- 如果构建或上传过程中网络不稳定，可以考虑先推送 Docker Hub，再使用方式二中的 `skopeo copy` 同步到 TCR。但 Skopeo 默认通过本机中转数据，上传速度仍受本地网络影响。

###### 推送方式二（从dockerhub复制）

推荐方式二，可以你先推送到dockerhub上，再从dockerhub去同步到tcr.

安装`skopeo`：

```bash
# Ubuntu
sudo apt install skopeo

# Fedora
sudo dnf install skopeo
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795594206147-065a423c.webp)

先清理 Skopeo 中可能失效的 TCR 凭证：

```bash
# 中国大陆
skopeo logout ccr.ccs.tencentyun.com

# 中国香港
skopeo logout hkccr.ccs.tencentyun.com
```

登陆：

```bash
# 中国大陆
skopeo login ccr.ccs.tencentyun.com

# 中国香港
skopeo login hkccr.ccs.tencentyun.com
```

看看是不是登陆上了，查看下用户名：

```bash
# 中国大陆
skopeo login --get-login ccr.ccs.tencentyun.com

# 中国香港
skopeo login --get-login hkccr.ccs.tencentyun.com
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795594915089-4646e8ac.webp)

比如我的为`tungchiahui/ros:humble-curc`，那么用户名命名空间为`tungchiahui`，镜像名为`ros`，tag为`humble-curc`。

你现在需要知道你的命名空间，和你想给这个镜像起什么名字，比如我的dockerhub和tcr都为`tungchiahui/ros:humble-curc`，那么用户名命名空间为`tungchiahui`，镜像名为`ros`，tag为`humble-curc`。

| 参数 | 作用 |
|---|---|
| --all	| 复制所有架构 |
| --src-no-creds | 匿名读取公开的 Docker Hub 镜像 |
| --retry-times 5 |	遇到可重试错误时，最多重试 5 次 |
| --image-parallel-copies 2	| 最多同时传输 2 个镜像层，降低并发压力 |

下面以中国香港为例：

```bash
skopeo copy \
  --all \
  --src-no-creds \
  --retry-times 5 \
  --image-parallel-copies 2 \
  docker://docker.io/dockerhub用户名命名空间/你的镜像名:tag名 \
  docker://hkccr.ccs.tencentyun.com/你的命名空间/你的镜像名:tag名
```

比如

```bash
skopeo copy \
  --all \
  --src-no-creds \
  --retry-times 5 \
  --image-parallel-copies 2 \
  docker://docker.io/tungchiahui/ros:humble-curc \
  docker://hkccr.ccs.tencentyun.com/tungchiahui/ros:humble-curc
```

###### 验证推送结果

https://console.cloud.tencent.com/tcr/repository

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795598000679-d1f5e480.webp)

但看上图是私有的，需要修改成公开。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795598204650-a4145755.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795598222658-0cf38323.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2024/10/03/1795598241692-6776c11b.webp)