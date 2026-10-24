---
title: "rosbag2"
---

## 基础概念

参考[rosbag2教程](/wiki/2023-12-30-ros2-tutorial/0700-hui-su-rosbag2)

## SLAM

### 功能包的职责

`nav_bag_tools`使用官方`ros2 bag record`和`ros2 bag play`，通过launch统一选择话题、QoS、输出位置、分卷和回放模式。现有C++ Writer／Reader demo保留为学习示例，正式使用不需要运行它们。

本页同时承担该包的使用说明，不再另建内容重复的README。

```text
nav_bag_tools/
├── launch/
│   ├── record_slam.launch.py
│   └── play_slam.launch.py
├── config/
│   ├── slam_topics.yaml
│   └── slam_qos.yaml
├── recordings/               # 每次录制生成一个子目录，已被Git忽略
└── src/
    ├── demo01_writer.cpp
    └── demo02_reader.cpp
```

两份launch采用`import os`、`generate_launch_description()`、`os.path.join(...)`和`return LaunchDescription([...])`的直接写法，并为配置路径、启动参数和进程添加中文注释。`DeclareLaunchArgument`声明终端可传入的参数，`LaunchConfiguration`读取参数值，`ExecuteProcess`启动命令行程序；这里运行官方`ros2 bag`，所以不使用启动ROS节点的`Node`。

话题选择、路径和数值检查直接写在各自的launch中，不再使用额外的包装脚本。两份launch分别由`start_record(context)`和`start_play(context)`读取实际参数、选择YAML话题列表，再通过`ExecuteProcess(cmd=['ros2', 'bag', ...])`直接启动官方工具。

`generate_launch_description()`先声明参数，再用`OpaqueFunction`调用本文件中的启动函数。这里需要等终端传入的`topics_file`、`include_debug`、`mode`等参数确定后才能读取文件、选择话题，所以不能在声明参数之前就把命令固定下来。启动函数中的`LaunchConfiguration('参数名').perform(context)`就是读取这个参数的实际字符串值；`OpaqueFunction`本身不负责录制或回放。

`record`负责录数据，`play`负责发布已有消息；**播放器本身不会重新计算SLAM，也不会自动启动Point-LIO、地形节点、机器人模型、雷达驱动或串口。** 重跑算法需要另外启动对应的离线算法节点。

本版先完成官方工具的启动封装；参数、模型、版本、地图和pose graph按后文清单手动归档，尚未实现自动采集或自动完整性报告。

### 录制哪些数据

默认清单来自2026-10-21的实车检查，包含24路数据／诊断话题和3路事件话题，共27路，定义在`config/slam_topics.yaml`的`record`列表中。

| 数据组 | 实际完整topic名称 | 保存目的 |
|---|---|---|
| 原始雷达与IMU | `/livox/lidar`、`/livox/lidar/pointcloud`、`/livox/imu` | 当前Point-LIO订阅CustomMsg格式的`/livox/lidar`；PointCloud2副本便于分析和接入其他算法；MID360 IMU用于LIO估计 |
| 反馈与关节 | `/serial/gimbal_joint_state`、`/serial/imu`、`/joint_states` | 对比硬件反馈与模型实际收到的关节状态；串口IMU不能代替MID360 IMU |
| 坐标与模型 | `/tf`、`/tf_static`、`/robot_description` | 保存当时的动态／静态坐标关系，以及模型文本 |
| LIO及接口输出 | `/aft_mapped_to_init`、`/cloud_registered`、`/lidar_odometry`、`/registered_scan`、`/odometry`、`/sensor_scan` | 判断问题首先出现于LIO、坐标转换还是机器人里程计生成 |
| 地形 | `/terrain_map`、`/terrain_map_ext` | 检查地面估计、障碍提取和近远场处理 |
| 二维建图 | `/obstacle_scan`、`/map`、`/map_metadata`、`/pose` | 保留slam_toolbox实际输入与结果，供后续比较或单独重跑 |
| 诊断 | `/rosout`、`/parameter_events`、`/diagnostics` | 保存日志、参数变化和运行状态 |
| 状态事件 | `/joy`、`/map_clearing`、`/cloud_clearing` | 保存可能改变地形缓存的按钮或清图输入 |

正式清单如下，可与录制结束后的`ros2 bag info`逐项对照：

```text
/livox/lidar
/livox/lidar/pointcloud
/livox/imu
/serial/gimbal_joint_state
/serial/imu
/joint_states
/tf
/tf_static
/robot_description
/aft_mapped_to_init
/cloud_registered
/lidar_odometry
/registered_scan
/odometry
/sensor_scan
/terrain_map
/terrain_map_ext
/obstacle_scan
/map
/map_metadata
/pose
/rosout
/parameter_events
/diagnostics
/joy
/map_clearing
/cloud_clearing
```

话题名称采用完整绝对名称，不会因为给launch换一个命名空间就自动替换。若实际namespace／remap已变化，需要同时调整话题清单与QoS文件的键。可通过`topics_file`、`qos_file`指定自己的配置副本。

**话题出现在ROS graph里不代表有有效消息。** 三路状态事件在上次检查时没有发布者；录制器允许发现无发布者的话题，并继续等待后来出现的发布者，整个实验未触发时没有消息是正常情况。仅仅启动录制器不构成完整性证明。

`include_debug:=true`会追加以下8路：

```text
/cloud_registered_body
/cloud_effected
/Laser_map
/path
/slam_toolbox/graph_visualization
/slam_toolbox/scan_visualization
/slam_toolbox/update
/slam_toolbox/feedback
```

前4路在上次检查中有发布端但未收到消息，部分发布开关关闭；要先确认对应功能确实出数据。控制响应实验可在自己的配置中补录`/cmd_vel`、`/cmd_gimbal_joint`、`/cmd_spin`，这些命令不属于默认清单。

### 编译与录制

在已安装ROS 2 Humble的环境中执行：

```bash
cd /home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav
source /opt/ros/humble/setup.bash
colcon build --packages-select nav_bag_tools --symlink-install
source install/setup.bash
```

实车录制需要和被录节点使用一致的ROS域、网络和接口环境。2026-10-21实车的域为6，以下仅按该次配置举例：

```bash
export ROS_DOMAIN_ID=6
ros2 launch nav_bag_tools record_slam.launch.py
```

默认保存到当前工程源码包下的`src/nav_bag_tools/recordings/`，每次录制在里面生成一个独立的bag目录。未指定`bag_name`时，目录名为`slam_年月日_时分秒_微秒`，启动时会打印绝对路径。该目录已加入工程的Git忽略规则。

默认路径通过`--symlink-install`安装后的launch链接定位源码目录，不依赖终端当前工作目录。在工控机上编译运行时，指向工控机自己的源码工程；应尽量在运行驱动的容器／机器本地录制。仍可通过`output_dir:=/data/recordings`改存到其他磁盘；使用源码目录作为默认位置时，按上面的命令保留`--symlink-install`编译选项。

也可以指定实验名称：

```bash
ros2 launch nav_bag_tools record_slam.launch.py \
  bag_name:=gimbal_yaw_01 \
  include_debug:=false
```

这个例子会生成`src/nav_bag_tools/recordings/gimbal_yaw_01/`，目录内包含`metadata.yaml`和SQLite的`.db3`分卷。

同名bag目录已存在时拒绝覆盖；下一次换新名称。录制不会自动启动／重启SLAM，可先启动录制器，再启动驱动和算法，让录制覆盖初始化阶段；中途接入运行中的SLAM，需要在实验说明中记下开始时间与此前已经运行的情况。

| 录制参数 | 默认值 | 含义 |
|---|---|---|
| `output_dir` | 源码包内的`recordings/` | bag父目录；可覆盖，支持`~`和相对路径，启动时解析成绝对路径 |
| `bag_name` | 自动生成 | 新bag目录名；不能填写多级路径 |
| `topics_file` | 包内`slam_topics.yaml` | 话题列表配置 |
| `qos_file` | 包内`slam_qos.yaml` | 官方rosbag2 QoS覆盖配置 |
| `include_debug` | `false` | 追加8路调试话题 |
| `use_sim_time` | `false` | 实车使用系统时间；设为`true`时，录制器等待`/clock`后才写消息 |
| `max_bag_size` | `2147483648` | 单卷约2 GiB；单位字节，0关闭按大小分卷 |
| `max_bag_duration` | `300` | 单卷300秒；0关闭按时长分卷；**不是录5分钟自动停止** |
| `max_cache_size` | `104857600` | 每个缓存约100 MiB；官方使用双缓冲，最坏仅缓存部分就可能接近200 MiB |

两种分卷条件同时启用，先满足的条件触发切换。本版固定使用Humble现有的`sqlite3`插件，不启用压缩；存储文件是`.db3`。旧学习demo中的Jazzy／MCAP数据不用于证明这份Humble配置已经可用。

结束时按一次`Ctrl+C`，等待官方录制器完成写盘并退出，再操作或复制文件。launch为结束清理保留了更长的信号等待时间，但不能保证强制断电或强杀后当前分卷仍完整。

### QoS为何单独配置

`slam_qos.yaml`是rosbag2的覆盖文件，顶层是topic名称，**不是**节点的`ros__parameters`。

| 话题 | 本版覆盖策略 | 原因 |
|---|---|---|
| 两种Livox点云、MID360 IMU | Best Effort、Volatile，有限Keep Last队列 | 减少新增可靠高带宽读端对驱动的影响；不保证无丢包 |
| `/obstacle_scan` | Best Effort、Volatile | 匹配上次实测的Best Effort发布者 |
| `/tf_static` | Reliable、Transient Local、深度100 | 晚启动的录制器需要取得所有静态发布者的快照；回放合并为一个发布者后也保留多条静态消息 |
| `/map`、`/map_metadata`、`/robot_description` | Reliable、Transient Local、深度1 | 晚加入时仍取得已有的地图／模型 |
| `/rosout` | Reliable、Volatile、深度1000 | 兼容不同节点提供的Volatile／Transient Local日志发布者，避免只按最先发现的发布者选择持久性 |
| 其他话题 | 官方工具自适应 | 本版没有把所有发现结果机械复制为覆盖值 |

队列深度是本版选择的初始配置，需要实测；不能照抄上次DDS发现返回的`UNKNOWN／0`。录制和播放目前默认使用同一份文件；若某个实验需要不同策略，分别通过两份launch的`qos_file`传入不同文件即可。

### 历史查看与原始输入重算

播放器默认发布100 Hz的`/clock`。使用独立的回放ROS域，并让RViz／算法终端使用同一个域；以下用106演示，不要与仍在运行的实车系统混用。

#### 查看历史现场

```bash
source install/setup.bash
export ROS_DOMAIN_ID=106
ros2 launch nav_bag_tools play_slam.launch.py \
  bag_path:="$PWD/src/nav_bag_tools/recordings/gimbal_yaw_01" \
  mode:=view \
  rate:=1.0
```

`view`发布`record`列表中的历史消息，`include_debug:=true`再增加调试列表。默认不播放清单外的话题，因此即使bag里另外存过控制命令，也不会自动把它们全部播出。此模式不需要启动新算法；需要可视化时另开RViz并使用仿真时间。

对于清单中bag没有录到的可选话题，播放器不会凭空生成数据。`view`包含原始输入和历史结果，不应与正在重算的算法同时运行。

#### 重跑算法时只提供输入

```bash
source install/setup.bash
export ROS_DOMAIN_ID=106
ros2 launch nav_bag_tools play_slam.launch.py \
  bag_path:="$PWD/src/nav_bag_tools/recordings/gimbal_yaw_01" \
  mode:=reprocess \
  start_paused:=true
```

`reprocess`只选择这6路：

```text
/livox/lidar
/livox/imu
/joint_states
/joy
/map_clearing
/cloud_clearing
```

其中当前Point-LIO使用CustomMsg输入，因此不同时播放PointCloud2副本。选择记录的`/joint_states`作为模型输入后，不同时启动另一个JointState发布器，也不再让原串口反馈链生成第二份同名关节消息。

**此launch只启动播放器。** 继续重算还需要在相同回放域中，使用归档参数、URDF及外参，另外启动所需的Point-LIO、loam_interface、sensor_scan_generation、terrain_analysis、terrain_analysis_ext、pointcloud_to_laserscan、slam_toolbox、robot_state_publisher和静态变换发布者，并确保每个算法节点实际`use_sim_time=true`。

静态TF从归档模型和静态变换配置重建；动态TF、里程计、点云和地图由新算法生成，所以`reprocess`不播放旧的`/tf`、`/tf_static`、`/robot_description`和算法输出。缺少模型、静态变换或算法节点时，**仅启动播放器不会得到新地图**。

已有实车总入口会启动真实驱动等硬件节点，不应直接当作完整离线重跑入口。离线算法组合的自动启动尚未包含在本版中；需要分别确认模型来源、关节发布者、参数替换和时钟之后再接入。

播放器暂停就绪后，在算法及模型已准备好的同域终端执行：

```bash
ros2 service call /rosbag2_player/resume rosbag2_interfaces/srv/Resume "{}"
```

launch关闭了播放器的终端键盘控制，以免依赖子进程能否收到空格键。可通过`/rosbag2_player/pause`、`/rosbag2_player/resume`服务控制播放；命令里的ROS域要与播放器一致。

| 回放参数 | 默认值 | 含义 |
|---|---|---|
| `bag_path` | 必填 | 含`metadata.yaml`的bag目录，不是其中一个`.db3`文件 |
| `mode` | `view` | 历史查看或`reprocess`输入重算 |
| `rate` | `1.0` | 播放倍率，必须为有限正数；调试可设0.5 |
| `clock_hz` | `100.0` | 发布`/clock`的频率，必须为有限正数 |
| `start_paused` | `false` | 暂停启动，供准备算法；通过服务恢复 |
| `include_debug` | `false` | 仅`view`可启用；重算模式禁止混入调试输出 |
| `topics_file`、`qos_file` | 包内对应配置 | 覆盖话题清单和QoS |

### bag以外还要保存什么

| 资产 | 保存内容 |
|---|---|
| 运行参数 | 关键节点开始／结束时的有效参数dump；不能只拿仓库默认YAML代替 |
| 启动配置 | 启动命令、namespace／remap、实际参数文件和launch替换结果 |
| 模型与外参 | 运行中的完整URDF、雷达安装外参、LiDAR／IMU外参、MID360配置JSON |
| 软件环境 | Git commit／diff、必要的未跟踪配置、自定义消息定义、ROS包版本、ROS域／RMW及网络配置 |
| ROS graph | 话题类型、发布订阅者、QoS，关键TF边与发布者的对应关系 |
| 地图与状态 | 地图图片及YAML；需要继续二维建图时保存slam_toolbox的`.posegraph`与`.data` |
| 先验资产 | 本次确实启用并读取的PCD／地图；保存路径、内容及校验值 |
| 实验说明 | 运动部件、动作顺序、已知距离、起终点、异常时间、时钟同步状态 |

`/parameter_events`不补齐录制前参数，`/map`不替代pose graph，PCD不替代逐点时间和IMU历史。slam_toolbox state也不包含Point-LIO全部滤波器／地图缓存；从中途开始的bag不能保证恢复启动以来的所有内部状态。

这些资料目前需要手动保存。本次没有实现自动dump、地图保存服务调用、state序列化或归档同步，避免把“录制进程启动成功”误写成“实验资产已全部保存”。

### 建议的首轮验证

先在本地回放ROS域录一份低负载测试数据，核对`bag info`中的类型、数量和时间范围；再短时进行实车静止、底盘静止而云台yaw缓慢转动、已知距离直行往返等独立实验。每次记录真实动作，算法输出的里程计不能作为自己的独立真值。

```bash
ros2 bag info "$PWD/src/nav_bag_tools/recordings/gimbal_yaw_01"
```

对必要原始输入检查是否有实际数据，核对时间戳是否有异常间隔；检查多个静态TF发布者的关键边是否都保存。事件话题未触发和关键传感器断流需要区别记录。实车高带宽话题避免叠加远程`hz`／`echo`／RViz订阅，用短时试录确认驱动和算法没有明显受影响，再延长录制。

### 本次验证范围（2026-10-24）

本机Humble环境已完成包编译、资源安装与两份launch参数展开检查。随后在隔离ROS域中使用合成小消息完成以下检查：

- 默认27路均录到实际消息，包含启动后才出现发布者的事件话题；单独保留的地图、模型和两份静态TF也被录到。
- 测试将分卷时间临时设为1秒，生成5个SQLite分卷并能连续读回。
- `reprocess`暂停启动、通过服务恢复，收到6路选定输入及`/clock`；未收到历史`/tf`、`/map`和`/cloud_registered`。
- `view`能播放历史TF、地图及点云；在静态消息已播过后才加入的Transient Local监听者仍取得两条测试静态边。
- 混合持久性的`/rosout`发布者在加入显式Volatile配置后，最终测试未再出现该QoS不兼容告警。
- 移除独立包装脚本后，重新验证以上流程；另验证默认时间戳命名、含空格的输出及配置路径、debug追加、仿真时间录制、同名目录拒绝覆盖，以及重算模式禁止混入debug输出。

这些检查验证的是官方命令封装、配置与模式选择。**尚未验证实车全带宽长期录制、真实点云内容质量、异常断电恢复或完整离线SLAM重算效果。**

### 参考

- [rosbag2官方Humble分支](https://github.com/ros2/rosbag2/tree/humble)：命令行、分卷、存储、压缩、回放服务和launch用法。
- [Humble录制与回放QoS覆盖说明](https://github.com/ros2/ros2_documentation/blob/humble/source/How-To-Guides/Overriding-QoS-Policies-For-Recording-And-Playback.rst)：QoS文件格式及应用方式。

