---
title: "loam_interface（Point-LIO输出的坐标与话题适配）"
---

## loam_interface（Point-LIO输出的坐标与话题适配）

这个节点把 Point-LIO 的位姿与配准扫描接到后续导航接口：订阅 `aft_mapped_to_init` 和 `cloud_registered`，发布 `lidar_odometry` 和 `registered_scan`。它执行坐标变换，不负责 SLAM 估计，也不直接发布 `odom → base_footprint` 的 TF。

以下按 2026-10-10 当前源码与实物 YAML 核对。参数名与当前值可以确认，但 IMU、雷达和底盘的完整位姿转换存在源码疑点，不能把话题有输出等同于坐标关系正确。

### 参数

#### ros__parameters（ROS节点参数）

##### 参数表

```yaml
loam_interface:
  ros__parameters:
    use_sim_time: False
    state_estimation_topic: "aft_mapped_to_init"
    registered_scan_topic: "cloud_registered"
    odom_frame: "odom"
    base_frame: "base_footprint"
    lidar_frame: "front_mid360"
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False` | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择节点及 TF 缓存使用的时间源 | 不适用 | 不适用 | ROS2 标准参数；实物通常 False，仿真 True 并依赖 `/clock`；整套 launch 会覆盖 YAML 中的值；不转换输入时间戳 |
| `state_estimation_topic` | `"aft_mapped_to_init"` | 不适用，话题名 | **上实物前必须核对并正确配置** | 指定输入位姿话题，类型为 `nav_msgs/msg/Odometry` | 不适用 | 不适用 | 当前对应 Point-LIO 的 `camera_init → body` 位姿消息，`body` 是内部 IMU；源码却按雷达位姿使用，见下文 |
| `registered_scan_topic` | `"cloud_registered"` | 不适用，话题名 | **上实物前必须核对并正确配置** | 指定输入配准扫描，类型为 `sensor_msgs/msg/PointCloud2` | 不适用 | 不适用 | 当前点坐标在 `camera_init` 中，已由 Point-LIO 使用雷达→IMU 外参变换；不能替换成原始扫描或 `cloud_registered_body` |
| `odom_frame` | `"odom"` | 不适用，坐标系名称 | **通常保持当前值，不作为优先调试项** | 指定输出点云及里程计的参考系名称 | 不适用 | 不适用 | 实际用缓存矩阵转换数据，再设置输出名称；不是通过 TF 查询任意目标系，也不发布对应 TF；名称本身不能保证水平或重力方向正确 |
| `base_frame` | `"base_footprint"` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定首次 TF 查询的目标机体系，用于构造初始对齐缓存 | 不适用 | 不适用 | 查询 `lookupTransform(base_frame, lidar_frame, 输入位姿时间戳, 0.5秒)`；首次成功后不再查询；独立 launch 写的是 `gimbal_yaw`，与整套配置不同 |
| `lidar_frame` | `"front_mid360"` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定首次 TF 查询的源系，并设置输出里程计子坐标系名称 | 不适用 | 不适用 | 指向雷达坐标系；设置这个名字不会把内部 IMU 位姿自动转换为雷达位姿；当前遗漏的外参转换需要单独处理 |

六项都有作用，本块没有当前完全不参与运行的参数。五项业务参数只在构造时读取，源码没有运行时更新处理；运行中写入同名参数不会重建订阅或更新缓存，需在另行授权的重启中应用。`use_sim_time` 由 ROS2 管理，不属于这五项。

##### 参数详解

###### use_sim_time（系统时间与仿真时间选择）

False 使用系统时间；True 使用 `/clock` 提供的 ROS 时间。TF 缓存用的是节点时钟，Point-LIO、关节 TF 和本节点必须处于一致的时间域。

本节点不会用当前时间重新标记输入位姿。切换时间源也不会修正雷达点时间单位、传感器时钟同步或旧时间戳；此前仿真转换器的秒／微秒不一致仍未解决。

###### state_estimation_topic 与 registered_scan_topic（输入里程计与配准点云）

| 方向 | 话题 | 消息与坐标含义 |
|---|---|---|
| 订阅 | `aft_mapped_to_init` | `Odometry`；参考系 `camera_init`，子系 `body`，数值是内部 IMU 位姿 |
| 订阅 | `cloud_registered` | `PointCloud2`；当前扫描的配准点坐标，参考系 `camera_init` |
| 发布 | `lidar_odometry` | `Odometry`；参考系名设为 `odom`，子系名设为 `front_mid360`；数值语义仍需核对 |
| 发布 | `registered_scan` | `PointCloud2`；用缓存矩阵转换点坐标，参考系名设为 `odom` |

两订阅和两发布的队列深度均为5。输入话题名是相对名，随节点命名空间和重映射解析；坐标系字符串不自动添加节点命名空间。两个输出话题名在源码中固定，不能通过修改输入参数改名。

位置与点坐标沿用上游的 m，姿态使用无量纲四元数，时间戳由秒和纳秒字段表达；本节点不缩放单位、不再次降采样，也不做两种输入的时间同步。里程计沿用输入位姿时间戳，正常点云转换沿用输入点云时间戳；后者已对照容器安装的 `pcl_ros 2.4.5` [对应实现](https://github.com/ros-perception/perception_pcl/blob/2.4.5/pcl_ros/src/transforms.cpp#L91)核对。

本节点只写输出里程计的位姿，不转传输入速度和协方差，这些字段保持默认零值，不能据此判断车辆静止或估计没有不确定性。

###### odom_frame（输出里程计与点云参考系）

当前保持 `odom`。节点对点云执行刚体变换，对位姿左乘同一缓存矩阵，然后填写输出参考系名称。它没有验证输入位姿的 `header.frame_id` 与 `child_frame_id` 是否符合预期，也没有按输入名称查询 `camera_init → odom`。

因此改为其他名字不会自动获得该参考系的变换。`odom` 是输出命名，实际轴向、原点和重力关系取决于缓存与 Point-LIO 初始化，不能靠名称判定正确。

###### base_frame（初始化基准的底盘坐标系）

首次收到位姿时，节点按该消息时间戳查询 `base_footprint ← front_mid360`，最大等待0.5秒；失败则告警并放弃本次位姿，后续位姿回调重试。首次成功后，把当时的雷达安装与关节变换存为固定矩阵 `C`，后续位姿和点云都使用它。

这项指定初始对齐所依赖的机体系，不表示本节点输出底盘位姿。当前 MID360 挂在 `gimbal_yaw`，所以该 TF 包含底盘到云台、机械 yaw 和雷达安装关系；它随 yaw 变化，不随 pitch 变化。缓存时刻也不一定等于 Point-LIO 初始化时刻，需要核对启动时的位姿与关节状态。

固定缓存本身不等于错误：两个固定参考系之间的对齐通常应保持固定，运动中的雷达→底盘变换由下游 `sensor_scan_generation` 按点云时间查询。需要确认的是 `C` 是否正确代表 `odom` 与 `camera_init` 的关系，不能简单改成每帧跟随云台更新。Point-LIO 的初始旋转还受重力对齐影响，不能默认输入初始姿态一定是单位旋转。

###### lidar_frame（雷达坐标系与输出里程计子系）

当前 `front_mid360` 与模型中的雷达名称一致，但上游 `body` 是内部 IMU。设 `T_AB` 表示把 B 系坐标转换到 A 系，W=`camera_init`、I=内部 IMU、L=`front_mid360`、O=`odom`，完整雷达位姿一般应满足：

\[
T_{OL}(t)=T_{OW}\,T_{WI}(t)\,T_{IL}
\]

当前源码实际计算 `C × 输入位姿`，即 `C × T_WI(t)`，随后将子系名设为 L；没有右乘 `T_IL`，也没有用初始输入位姿核对 `C=T_OW` 的条件。

Point-LIO 配准点云已经使用 `mapping.extrinsic_R/T`，而位姿发布直接取滤波器的 IMU 位置和姿态。当前实物 `extrinsic_R` 为单位矩阵，`extrinsic_T=[-0.011,-0.02329,+0.04412] m` 非零；轴向一致不代表两个原点相同。缺少这一步时，雷达绕 IMU 转动的杆臂效应可能表现为位置误差。单纯改 `lidar_frame` 或话题名无法补齐外参；先记录这一接口缺口，工程修改另行授权。

###### 启动配置与源码缺口

| 启动方式 | 这块参数从哪里来 | 需要区分的值 |
|---|---|---|
| 整套 reality launch | 默认 `pb2025_nav_bringup/config/reality/nav2_params.yaml`，可由 `params_file` 指定其他文件；经 bringup 传给 `navigation_launch.py` | `base_frame=base_footprint`；`use_sim_time` 最终由 launch 参数重写，默认 False |
| 整套 simulation launch | 默认 `pb2025_nav_bringup/config/simulation/nav2_params.yaml`；同样经 `navigation_launch.py` 加载 | `base_frame=base_footprint`；`use_sim_time` 最终由 launch 参数重写，默认 True |
| 独立 `loam_interface_launch.py` | launch 内的参数字典，不读取上述 YAML | `base_frame=gimbal_yaw`；未设置或声明 `use_sim_time` 启动参数，按 ROS2 默认 False |

整套启动的独立进程和组件模式都加载同一份处理后的参数文件；`slam` 选择建图或定位，不会跳过这段导航接口。本次在 `ysu_ros_humble` 中仅加载环境并查询包路径，确认相关安装后的 launch 和 YAML 指向当前源码；没有启动节点，也没有读取运行节点的有效参数。

另一个已确认的源码缺口是初始化保护：点云回调没有检查初始 TF 是否成功，直接使用缓存。容器的 `tf2::Transform` 默认构造不初始化数值，而节点也没有先将缓存设为单位变换。如果点云先到、或首次 TF 查询一直失败，可能使用未初始化矩阵；触发后的运行表现本轮未测试。

源码位置：`loam_interface/src/loam_interface.cpp` 的参数读取、点云回调和位姿回调；`point_lio/src/laserMapping.cpp` 的 `set_posestamp/publish_odometry`，以及 `point_lio/src/Estimator.cpp` 的 `pointBodyToWorld`。上述包均位于 `src/pb2025_sentry_nav/`。

###### 已观察与待核实事项

- **源码能确认：** 六项参数的使用、输入输出话题、首次 TF 缓存、IMU 位姿未补雷达外参、初始化保护缺失，以及 launch 配置来源。
- **仿真已观察：** 2026-10-08 的独立 Point-LIO 检查在静止和约20 cm低速往返时有正常位姿与配准点云；该检查没有覆盖本节点和下游的完整转换，也没有独立云台 yaw 验证，不能据此认定这里的接口正确。
- **必须实物核实：** MID360 实际安装位置与轴向、内部雷达／IMU 外参、机械 yaw/pitch 零位与方向、串口角度单位和含义、各消息的时钟一致性，以及 Point-LIO 重力目标参考系与导航 `odom` 的关系。当前约定要求 JointState 使用机械相对角，世界 IMU 姿态不能代替关节角；本轮不修改重力方向。

机械关系与[Linux与MCU数据对接](/wiki/2026-09-20-dao-hang/0400-0100-linux-yu-mcu-shu-ju-dui-jie)、[MID360安装位置](/wiki/2026-09-20-dao-hang/0300-0200-mid360-an-zhuang-wei-zhi)一致：两个odom关节固定，实际yaw在`gimbal_yaw_joint`、实际pitch在`gimbal_pitch_joint`，串口position为`{relative_pitch, relative_yaw, 0, 0}`。MID360挂在实际yaw关节之后，随yaw、不随pitch；不能用关节之前的`gimbal_yaw_odom`替代其挂载层。

