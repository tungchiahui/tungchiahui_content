---
title: "sensor_scan_generation（雷达点云换系与底盘、云台位姿转换）"
---

## sensor_scan_generation（雷达点云换系与底盘、云台位姿转换）

这个节点把 `loam_interface` 输出的雷达位姿和配准点云近似配对，再结合机器人机械 TF，发布底盘 TF、指定机体系的里程计和雷达系扫描点云。当前底盘 TF 的子系为 `base_footprint`，`odometry` 消息的子系则为 `gimbal_yaw`，两者用途不同。

以下按源码与实物 YAML 核对。转换成立的前提是上游输入确实为雷达位姿；上一块记录的内部 IMU／雷达位姿接口缺口仍会影响本节点，不能在这里靠改坐标系名称补齐。

### 参数

#### ros__parameters（ROS节点参数）

##### 参数表

```yaml
sensor_scan_generation:
  ros__parameters:
    use_sim_time: False
    lidar_frame: "front_mid360"
    base_frame: "base_footprint"
    robot_base_frame: "gimbal_yaw"
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False` | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择节点与 TF 缓存使用的时间源 | 不适用 | 不适用 | ROS2 标准参数；实物通常 False，仿真 True 并依赖 `/clock`；整套 launch 会覆盖 YAML 值；速度差分仍使用 `steady_clock`，不随此开关切换 |
| `lidar_frame` | `"front_mid360"` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定机械 TF 查询的目标雷达系，以及输出 `sensor_scan` 的参考系 | 不适用 | 不适用 | 输入 `lidar_odometry` 必须实际表示该雷达位姿；每次配对按点云时间查询雷达←底盘、雷达←指定机体系；不是内部 IMU 的名称 |
| `base_frame` | `"base_footprint"` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定底盘位姿转换目标和所发布 TF 的子系 | 不适用 | 不适用 | 当前发布输入里程计参考系→`base_footprint` 的 TF；不决定 `odometry` 的子系；不会自动将 z、roll、pitch 置零；独立 launch 默认 `chassis` |
| `robot_base_frame` | `"gimbal_yaw"` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定输出 `odometry` 的机体系及其位姿转换 | 不适用 | 不适用 | 当前里程计表示 yaw 云台层，角运动包含云台相对底盘的 yaw；不单独广播参考系→`gimbal_yaw` TF；速度字段的时间与坐标表达有源码疑点 |

四项都有作用，本块没有当前完全不使用的参数。三个坐标系参数只在构造时读取，运行时写入同名参数不会更新成员值；后续应用变更需另行授权重启。坐标系名称不属于数值调参，调大／调小均不适用。

##### 参数详解

###### use_sim_time（系统时间与仿真时间选择）

False 使用系统时间；True 使用 `/clock` 的 ROS 时间。输入位姿、配准扫描、机械 JointState／TF 和节点应使用一致的时间域，否则按消息时间查询机械 TF 可能失败。

这个开关影响节点与 TF 缓存时钟，但源码计算速度时另用 `std::chrono::steady_clock`。所以 True 不会让速度自动按仿真时间计算，也不会修复输入点时间单位或时钟同步问题。

###### lidar_frame（扫描点云的目标雷达系）

| 方向 | 话题／TF | 当前消息与坐标含义 |
|---|---|---|
| 订阅 | `lidar_odometry` | `nav_msgs/msg/Odometry`；从位姿字段取得输入参考系中的雷达位姿，预期为 `odom` 中的 `front_mid360` |
| 订阅 | `registered_scan` | `sensor_msgs/msg/PointCloud2`；预期为同一参考系 `odom` 中的配准扫描 |
| 发布 | `sensor_scan` | `PointCloud2`；用输入雷达位姿的逆变换，将点坐标转回 `front_mid360` |
| 发布 | `odometry` | `Odometry`；位姿参考系沿用输入里程计 `header.frame_id`，子系设为 `gimbal_yaw` |
| 广播 | 当前 `odom → base_footprint` | 父系沿用输入里程计 `header.frame_id`，子系由 `base_frame` 指定 |

输入和输出话题名在源码中固定为相对名，可受节点命名空间或重映射影响；本块没有话题名参数，也没有 `odom_frame` 参数。坐标系字符串不自动添加节点命名空间。

`sensor_scan` 是配准扫描转换回雷达系的结果，不会恢复已被 Point-LIO 丢弃的点，也不执行新的估计或运动补偿。位置与点坐标沿用 m，姿态为无量纲四元数，线速度字段按 m/s、角速度字段按 rad/s 表达；速度含义仍需结合下文的源码缺口判断。

节点直接把输入位姿当作雷达位姿，并用其逆变换处理点云；没有核验输入 `child_frame_id` 是否对应 `lidar_frame`，也没有验证两种输入的参考系一致。上游若仍是内部 IMU 位姿，雷达→底盘的机械 TF 不能代替缺失的 IMU→雷达外参。

###### base_frame（底盘位姿TF的目标系）

每个配对回调按点云时间戳查询 `front_mid360 ← base_footprint`，最大等待0.5秒，不使用上游节点那种一次性初始化缓存。设 `T_AB` 把 B 系坐标转换到 A 系，O 为输入里程计参考系、L 为雷达、B 为底盘，则计算：

\[
T_{OB}=T_{OL}\,T_{LB}
\]

当前通过这项结果广播 `odom → base_footprint`。当底盘静止、只有实际 yaw 转动时，正确的雷达位姿和机械相对 yaw 应能互相抵消，得到基本不动的底盘位姿；这是待验证的转换目标，本轮未进行该项测试。

当前模型中 `base_footprint → chassis` 是固定连接，高度为 `chassis_height=0.0625 m`。本节点保留完整三维变换，并不根据名称把底盘投影到地面或强制水平。

两次机械 TF 查询都通过 `getTransform`；任一次失败都会告警并返回单位变换，回调仍继续发布。例如雷达→底盘查询失败时，底盘位姿会直接变成输入雷达位姿，安装偏移和关节旋转未被扣除。有输出不能证明机械 TF 查询成功。

###### robot_base_frame（输出云台层里程计的坐标系）

每次回调还按点云时间查询 `front_mid360 ← gimbal_yaw`，再计算：

\[
T_{OR}=T_{OL}\,T_{LR}
\]

R 为 `robot_base_frame`。该位姿写入 `odometry`，因此当前消息表示 yaw 云台层的原点和姿态。源码的发布器变量虽叫 `pub_chassis_odometry_`，实际子系取的是 `robot_base_frame`，不能据变量名把它认作底盘里程计。

MID360 固定在 `gimbal_yaw`，雷达→yaw 云台层的安装关系固定；雷达→底盘关系则包含实际机械 yaw，随关节角变化。实际 pitch 在 `gimbal_pitch_joint`，位于雷达挂载层之后，不应进入雷达到这两个目标系的机械变换。两个 odom 关节保持固定，JointState 继续使用 `{relative_pitch, relative_yaw, 0, 0}`。

当前 `fake_vel_transform` 也配置为订阅 `odometry`、使用 `gimbal_yaw`，两者名称相符。`robot_base_frame` 决定里程计消息表达的机体系，不单独发布该机体系的 TF；已有机器人 TF 链连接 `base_footprint` 到 `gimbal_yaw`。云台独立 yaw 的角运动不能直接当作底盘角运动。

###### 同步与速度计算的源码疑点

两订阅使用 Best Effort、Keep Last 深度1；近似时间同步器 `ApproximateTime(100)` 的内部队列长度为100；两发布队列深度为2。这些都写在源码中，没有对应 YAML 参数，100 不是输出频率或100毫秒容差。

两个输入必须形成配对才进入发布回调。源码没有设置最大配对时间差，也没有将输入位姿插值到点云时间：位姿取配对的里程计消息，机械 TF 按点云时间查询，输出 TF 与 `odometry` 都标记为点云时间。近似配对不等于同一采样时刻；实际配对误差和运动影响仍需验证。

速度由连续两次转换后的 `robot_base_frame` 位姿差分得到，没有使用输入里程计的 `twist`。有三项明确的实现问题需要记录：

- **时间基准：** 分母是两次发布回调之间的真实单调时间，不是两帧消息时间戳之差。仿真加速、减速、暂停或回调积压时，差分速度可能失真；`use_sim_time=True` 也不会改变这条计算路径。
- **坐标表达：** 位置差在输入参考系（当前 `odom`）中计算，姿态差使用 `q_current × q_previous.inverse()`，也在该参考系表达；写入 `twist` 前没有转到 `gimbal_yaw`。而 ROS 的 [Odometry 定义](https://github.com/ros2/common_interfaces/blob/humble/nav_msgs/msg/Odometry.msg)要求速度在 `child_frame_id` 中表达，当前实现与消息约定不一致。
- **首帧处理：** 没有“已收到有效历史位姿”的标志，首帧也会进入差分；`previous_transform` 未用首帧位姿建立历史，首次 `previous_time` 与当前时间又相隔很短。首个速度结果可能异常，不能把它作为有效测量。

输出位姿和速度协方差均保持默认零值；源码没有估计不确定性。上述实现问题可以由源码确认，实际速度误差、首帧数值和对下游的影响本轮未运行测量。

###### 启动配置来源

| 启动方式 | 参数来源 | 与本块有关的区别 |
|---|---|---|
| 整套 reality launch | 默认 `pb2025_nav_bringup/config/reality/nav2_params.yaml`，可由 `params_file` 指定；经 `navigation_launch.py` 加载 | 三个坐标系参数为本表值；`use_sim_time` 经 launch 重写，默认 False |
| 整套 simulation launch | 默认 `pb2025_nav_bringup/config/simulation/nav2_params.yaml`；同样经 `navigation_launch.py` 加载 | 三个坐标系参数与实物相同；`use_sim_time` 经 launch 重写，默认 True |
| 独立 `sensor_scan_generation.launch.py` | 读取自身声明的 `lidar_frame/base_frame/robot_base_frame` 启动参数，不读取上述 YAML | 默认分别为 `front_mid360/chassis/gimbal_yaw`；未设置或声明 `use_sim_time` 启动参数，按 ROS2 默认 False |

独立默认 `base_frame=chassis` 与当前整套配置不同。若同时运行当前 robot_state_publisher，已有固定的 `base_footprint → chassis`，再发布 `odom → chassis` 会让同一子系出现不同父系的发布；独立默认值不能直接视为与整套启动等价。

整套启动的独立进程和组件模式都使用同一份处理后的参数文件，`slam` 不会跳过这个接口。本轮在 `ysu_ros_humble` 中仅加载环境并查询包路径，确认安装后的本包 launch、导航 launch 和两份 YAML 与当前源码一致；未启动节点，也未读取运行节点的有效参数。

###### 已观察与待核实事项

- **源码能确认：** 四项参数均有作用；底盘 TF 与 yaw 云台层里程计的区别；机械 TF 按每次点云时间查询；TF 失败继续使用单位变换；近似配对方式；速度的时间、坐标与首帧问题。
- **仿真已观察：** 2026-10-08 的静止及约20 cm往返检查覆盖独立 Point-LIO 输出，没有验证本节点的 `sensor_scan`、底盘 TF、云台层里程计及差分速度；不能据此认定本接口正确。
- **必须实物核实：** 雷达与云台安装外参、机械相对 yaw 的零位／方向／弧度单位、关节与雷达数据的时钟一致性，以及上游内部 IMU 位姿到雷达位姿的完整关系。后续获准测试时，应区分底盘运动、独立 yaw 和独立 pitch，分别检查输出；当前重力方向与仿真点时间接口仍保留待核实状态。

### 实际问题与解决方案

已确认的直接故障：

`sensor_scan_generation::getTransform()` 在 TF 查询失败时返回单位变换，调用者仍把这个结果用于发布 `odom → base_footprint`。当用于雷达至底盘的查询失败时，发布的底盘位姿接近直接复制雷达位姿，丢掉雷达到底盘的安装旋转与平移。TF 查询恢复时则重新执行正常转换，因此出现突然腾空、倾斜，随后落回原位置的间歇性跳变。

分类 | 数量 |
|---|---:|
| 正常转换 | 4014 |
| 与复制雷达位姿相符的单位变换回退 | 301 |
| 未归入上述两类 | 81 |
| 回退同时匹配到相同请求时刻的 TF 失败日志 | 301 |
| 记录窗口内 TF 失败日志总数 | 317

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791809046850-041080bd.webp)


| 状态 | 底盘 TF 高度 | roll |
|---|---:|---:|
| 正常 | 0.18 米 | −0.5° |
| 查询失败 | 0.55 米 | 45.6° |
| 恢复 | 0.18 米 | −0.5° |


| 检查项 | 结果及含义 |
|---|---|
| 当前运行状态 | 仿真、Point-LIO、`loam_interface`、`sensor_scan_generation` 均存在，使用域 6 |
| 实际关节 TF 频率 | `robot_state_publisher` 是 **20 Hz**；当前 yaw TF 时间间隔主要为 50 ms |
| TF 接收线程 | 监听器有独立线程，不能简单认定为“回调等待时无人接收 TF” |
| 历史失败类型 | 检查开始时日志已有 378 次失败，全部是向未来查询 |
| 里程计发布量 | 当前开启逐点组发布；约 8 秒观察收到 12,706 条雷达里程计、125 帧扫描，约每帧对应 100 条里程计 |

它读取的主要话题是：

| 话题 | 类型 | 内容与来源 |
|---|---|---|
| `lidar_odometry` | `nav_msgs/Odometry` | 雷达在 `odom` 中的位姿，由 `loam_interface` 发布 |
| `registered_scan` | `sensor_msgs/PointCloud2` | 已转换到 `odom` 坐标系的点云，由 `loam_interface` 发布 |
| `tf`、`tf_static` | `tf2_msgs/TFMessage` | 机器人安装关系和动态关节变换 |
| `/clock` | `rosgraph_msgs/Clock` | 启用仿真时间时使用的时钟 |

它将前两个话题按时间近似配对，然后以点云时间戳查询安装关系。

| 话题 | 内容 |
|---|---|
| `tf` | **`odom → base_footprint`**：通过雷达位姿和当时的云台 yaw，反推出底盘位姿 |
| `odometry` | **`odom` 中的 `gimbal_yaw` 位姿**，以及差分计算的速度；当前配置下不是底盘中心的里程计 |
| `sensor_scan` | 转换到 **`front_mid360` 坐标系的三维点云**，类型仍是 `PointCloud2`，不是二维 `LaserScan` |

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791810391799-9334ffc1.webp)

准确地说，故障顺序是：
1. sensor_scan_generation 查询扫描时刻的雷达到底盘变换失败，这条关系包含动态云台 yaw。
2. 代码用单位变换代替，继续计算并发布 odom → base_footprint。
3. 底盘位姿因此近似变成雷达位姿，带上雷达的安装高度和约 45° 安装倾角。
4. 查询恢复后，底盘 TF 又恢复正常。
查询失败的是雷达与底盘之间的变换；最终“炸掉”的输出是 odom → base_footprint。

