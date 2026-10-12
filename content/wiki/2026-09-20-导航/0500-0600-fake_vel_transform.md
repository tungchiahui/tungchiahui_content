---
title: "fake_vel_transform（云台系速度转换与底盘自旋）"
---

## fake_vel_transform（云台系速度转换与底盘自旋）

### 速度和旋转究竟参考谁

**当前导航链最终下发的 `vx、vy` 按实际云台自身坐标系 `gimbal_yaw` 表达；`wz` 按接口约定是底盘的目标角速度。** “速度用谁的轴表达”与“哪个部件执行动作”是两件事：用云台的轴描述平移，仍然是让底盘移动；输出旋转指令也没有直接控制云台 yaw 关节。

这来自北极熊原包的设计：先用一个朝向较稳定的虚拟系供导航计算，再把速度转换到实际云台系；`cmd_spin` 用于叠加底盘自旋。[北极熊原包说明](https://github.com/SMBU-PolarBear-Robotics-Team/pb2025_sentry_nav/blob/main/fake_vel_transform/README.md)、[原包速度转换源码](https://github.com/SMBU-PolarBear-Robotics-Team/pb2025_sentry_nav/blob/main/fake_vel_transform/src/fake_vel_transform.cpp)。这是这套工程的约定，不能认为所有 ROS2 的 `cmd_vel` 都按云台系表达。

| 链路阶段 | `vx、vy` 参考的方向 | `wz` 的含义 |
|---|---|---|
| 导航控制器计算，再经平滑器得到 `cmd_vel_nav2_result` | 虚拟云台系 `gimbal_yaw_fake` 的 x、y 轴 | 导航提出的旋转分量；恢复行为也可向该话题提供指令 |
| 本节点发布 `cmd_vel` | 实际云台 yaw 层 `gimbal_yaw` 的 x、y 轴 | 输入导航角速度 + 附加自旋角速度 |
| `serial_comm` 接收并发送给MCU | 原样传递上述云台系分量，没有再转成底盘系 | 原样传递已相加的总量，约定由底盘执行 |
| MCU驱动底盘 | 若控制算法需要底盘系速度，须按机械相对 yaw 换算 | 应执行这条底盘目标角速度，不再重复叠加同一自旋量；实际实现待核对 |

`vx、vy` 单位为 m/s，分别沿所选参考系的 x、y 轴；`wz` 单位为 rad/s，表示绕 z 轴的旋转速度指令。下面按平面运动、x朝前、y朝左、z朝上的约定解释；正 `wz` 从上方看为逆时针。实车轴向、MCU正负号和单位仍需核对。

**实际云台系是随 yaw 转动的普通机体系，不是世界坐标系。** 云台朝前时，其 `vx>0` 朝车前；云台向左转90°，同样的正 `vx` 方向就转到车左侧。底盘朝向不变，也不妨碍用另一套轴描述它的运动。

`gimbal_yaw_fake` 是计算用的虚拟系：位置与 `gimbal_yaw` 重合，源码广播 `gimbal_yaw → gimbal_yaw_fake`，旋转取输入 yaw 的相反数，没有平移。在平面运动、输入位姿与机械 TF 正确且及时更新时，它的水平朝向与 `odom` 对齐，而位置跟着机器人走。它不是 `map`，也不是云台或底盘的世界原点；“方向与 odom 对齐”不表示完整坐标系与 odom 重合。源码只抵消 yaw，没有抵消 roll、pitch，坡面上不能直接套用完全水平的结论。

##### 一个直行例子：云台左转90°，车仍沿原方向走

假设底盘朝 `odom` 的 +x，云台原本也朝前；现在只有云台向左转90°。导航仍要求沿 `odom` 的 +x，以0.3 m/s移动，自旋为0：

| 用哪套轴描述同一个平移目标 | `vx` | `vy` |
|---|---:|---:|
| 虚拟系 `gimbal_yaw_fake` | 0.3 | 0 |
| 转换后的实际云台系 `gimbal_yaw`，也是串口发送的数值 | 0 | -0.3 |
| MCU正确换算到底盘系后 | 0.3 | 0 |

**数值变了，运动目标没有变。** MCU若把收到的 `(0,-0.3)` 直接当成底盘系分量，就会理解成往车右侧走。该例只解释坐标转换，没有进行车辆运动测试。

源码使用的角度是 `odometry.pose` 中的 yaw，当前预期为“`gimbal_yaw` 相对 `odom` 的朝向”，同时包含底盘整体朝向与机械相对 yaw 的影响；它不是单独的机械关节角。线速度计算为：

```text
vx_out =  cos(θ) × vx_in + sin(θ) × vy_in
vy_out = -sin(θ) × vx_in + cos(θ) × vy_in
θ      = 输入里程计位姿中的 yaw，单位 rad
```

MCU从云台系换到底盘系时，要另外使用两者的机械相对角。若 q 表示云台系相对底盘系的平面朝向，已计入固定安装关系，方向换算为：

```text
vx_chassis = cos(q) × vx_out - sin(q) × vy_out
vy_chassis = sin(q) × vx_out + cos(q) × vy_out
q          = 云台相对底盘的角度，单位 rad
```

**θ 相对 odom，q 相对底盘，二者不能直接混用。** 不能把MCU世界 IMU 姿态当关节角；q 应由机械相对 yaw与安装零位确定。上述90°例子中，底盘与 odom方向恰好一致，所以两角相同；底盘自己也转向时，就不再相同。

上位机这一步没有查询云台→底盘 TF，也没有处理不同原点之间的旋转速度补偿；上面的MCU公式仅解释平面方向换轴。原点偏置是否需要处理，要结合真实安装和下位机速度定义核对。

##### wz：同一个底盘指令的两项相加

```text
wz_final = wz_nav2 + wz_spin
```

两项单位相同，都是 rad/s：

- `wz_nav2`：导航提出的旋转分量，例如调整朝向；它是控制指令，不是测得的云台或底盘转速。
- `wz_spin`：人为附加的底盘自旋量，由启动参数或 `cmd_spin` 消息提供。
- `wz_final`：发送给底盘的总目标角速度。没有分别给两个机械部件各发一个转速。

| 导航旋转分量 | 附加自旋 | 最终输出 | 含义 |
|---:|---:|---:|---|
| +0.5 | +1.0 | +1.5 | 同向叠加 |
| -0.5 | +1.0 | +0.5 | 反向抵消一部分 |
| 0 | +1.0 | +1.0 | 导航没有要求旋转，仍有附加自旋 |

平面运动时，云台和底盘仅水平朝向不同，z轴方向相同；绕 z 的角速度分量不因水平 yaw 换轴而改变，所以源码只旋转 `vx、vy`，对 `wz` 直接相加。此处没有求导云台 yaw，也没有把云台相对转速作为底盘转速输入。底盘整体转向与 `gimbal_yaw_joint` 的机械相对转动必须分开理解，不能根据话题参考系名称混为同一个动作。

当前实物、仿真的 `FollowPath.enable_rotation` 都是 False，普通路径跟踪时该控制器自身给出的导航角速度为0。因此尚未改自旋设置时，实物默认不附加自旋，仿真默认附加3.14 rad/s，约每2秒一圈。Nav2恢复旋转等行为另有指令来源，速度平滑器也可能保留过渡过程，不能推成整个导航链永远只输出0角速度。

### 参数

#### ros__parameters（ROS节点参数）

##### 参数表

本表以当前实物 YAML 为基准；仿真差异单列。`local_plan_topic` 没写在这两份 YAML 中，但源码会声明并读取，当前使用其默认值。

```yaml
fake_vel_transform:
  ros__parameters:
    use_sim_time: False
    odom_topic: "odometry"
    robot_base_frame: "gimbal_yaw"
    fake_robot_base_frame: "gimbal_yaw_fake"
    input_cmd_vel_topic: "cmd_vel_nav2_result"
    output_cmd_vel_topic: "cmd_vel"
    cmd_spin_topic: "cmd_spin"
    init_spin_speed: 0.0
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真 `True` | 不适用，布尔 | **上实物前必须核对并正确配置** | 控制节点ROS时钟，当前用于输出TF时间戳 | 不适用 | 不适用 | 0.5 s控制器活跃判断另用系统时间，20 ms定时器也用真实时间；True不会统一这些路径 |
| `odom_topic` | `"odometry"` | 不适用，话题名称 | **上实物前必须核对并正确配置** | 提供用于方向转换的位姿 yaw | 不适用 | 不适用 | 当前预期是 odom 中的 gimbal_yaw 位姿；只读 pose，不使用 twist；不校验消息参考系 |
| `robot_base_frame` | `"gimbal_yaw"` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定输出速度的约定参考系及虚拟TF的父系 | 不适用 | 不适用 | 当前是实际 yaw 云台层，名称里的 base 不表示一定为底盘；源码不查询这个系的TF |
| `fake_robot_base_frame` | `"gimbal_yaw_fake"` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定供导航使用的虚拟系及所广播TF的子系 | 不适用 | 不适用 | 需与局部代价地图等使用者一致；与真实云台原点重合，平面朝向抵消输入 yaw；不是机械关节 |
| `local_plan_topic` | `"local_plan"`，源码默认；YAML未配置 | 不适用，话题名称 | **上实物前必须核对并正确配置** | 用路径时间戳与里程计近似配对，并判断控制器活跃 | 不适用 | 不适用 | 不使用路径几何来转速度；与最新无时间戳的速度间接配合，不能保证三者同一时刻 |
| `input_cmd_vel_topic` | `"cmd_vel_nav2_result"` | 不适用，话题名称 | **上实物前必须核对并正确配置** | 接收虚拟系中表达的导航速度指令 | 不适用 | 不适用 | 类型 Twist，无 frame_id／时间戳；不能仅按字段名判断方向；当前接平滑器及导航行为输出 |
| `output_cmd_vel_topic` | `"cmd_vel"` | 不适用，话题名称 | **上实物前必须核对并正确配置** | 发布转换后的线速度与相加后的角速度 | 不适用 | 不适用 | 类型 Twist；串口按原值接收，须与MCU约定一致；直接发同名话题的其他入口应另行核对 |
| `cmd_spin_topic` | `"cmd_spin"` | 不适用，话题名称；消息数值为 rad/s | **根据实际功能需求开启或设置** | 接收并保存新的附加底盘自旋量 | 不适用 | 不适用 | 类型 example_interfaces/msg/Float32；消息覆盖旧值，不是逐次累加；收到消息本身不立即发布速度 |
| `init_spin_speed` | `0.0`；仿真 `3.14` | rad/s，带正负号的附加量 | **根据实际功能需求开启或设置** | 启动时给附加自旋量赋初值 | 数值更偏正方向，最终效果取决于导航分量与符号 | 数值更偏负方向；负值绝对值变大可能反向转得更快 | 并非最大转速；cmd_spin覆盖后使用新值；0只关闭附加量，不保证导航角速度为0 |

##### 参数详解

###### odom_topic、robot_base_frame 与 fake_robot_base_frame（输入里程计与真实、虚拟云台系）

`odom_topic` 当前接 `sensor_scan_generation` 的 `odometry`；消息位姿参考系预期为 `odom`，`child_frame_id` 为 `gimbal_yaw`。本节点读这个位姿的 yaw，发布父系 `gimbal_yaw` 到子系 `gimbal_yaw_fake` 的反向 yaw 变换，约50 Hz，平移为零。

因此，换话题或改坐标系名称前，必须同时检查输入位姿究竟属于谁。改名字不会把底盘位姿自动变成云台位姿，也不会把世界 IMU 姿态变成关节角。源码没有查询或校验这些关系；上游内部 IMU→雷达转换、机械TF失败返回单位变换等已记录问题，也可能使这里用错角度。

MID360继续挂在 `gimbal_yaw`，随实际 yaw、不随实际 pitch 转动；实际 yaw、pitch 分别只在 `gimbal_yaw_joint`、`gimbal_pitch_joint`，两个 odom 关节固定。虚拟系不是增加第二个实际 yaw，串口 JointState仍按机械相对角填 `{relative_pitch, relative_yaw, 0, 0}`。

###### input_cmd_vel_topic、output_cmd_vel_topic 与MCU配合（速度输入与输出话题）

当前跟踪控制器把路径转到局部代价地图的机体系；该系配置为 `gimbal_yaw_fake`，据此计算 x、y速度。跟踪输出经 `cmd_vel_controller → velocity_smoother → cmd_vel_nav2_result`，再进入本节点；导航行为也可输出到 `cmd_vel_nav2_result`。

转换后发布的 `cmd_vel` 是 `geometry_msgs/msg/Twist`，消息本身没有写明坐标系，也没有时间戳。输出变量名字虽然含 `chassis`，源码只是用输入 yaw 旋转平面分量，并未把速度再转换到 `base_footprint` 或 `chassis`。判断方向要看配置、计算和上下游约定，不能靠变量名或“最后给底盘用”推断它已是底盘系。

`serial_comm` 直接把 `linear.x、linear.y、angular.z` 写入协议字段，按周期发送，没有用 JointState再换方向。MCU需核对四件事：线速度是否按云台系接收；机械相对 yaw 的零位与正负号；m/s与rad/s的单位；角速度是否已经包含自旋、是否会重复相加。当前没有核对MCU执行代码或实物响应，不能把上位机约定写成下位机已正确实现。

这条参考系结论只针对上述导航链。手柄等若绕过本节点直接发布 `cmd_vel`，不自动获得同样的转换；它们的速度参考系及同名话题的使用方式需要另行核对，本块不展开修改。

###### init_spin_speed 与 cmd_spin_topic：改的是附加量

当前实物 `init_spin_speed=0.0`，仿真为3.14。本节原先统一写“默认3.14 rad/s”，现按两份配置区分；前文“数据对接”仍保留3.14示例，不代表当前实物默认值，本轮不改写该旧章节。北极熊官方实物 YAML 当前采用3.14，这是原包设置，当前工程的具体值应以本表为准。[北极熊官方配置](https://github.com/SMBU-PolarBear-Robotics-Team/pb2025_sentry_nav/blob/main/pb2025_nav_bringup/config/reality/nav2_params.yaml)

启动初值只用于给 `spin_speed_` 赋值；收到 `cmd_spin` 后直接用消息数值替换，后续一直保留，源码没有自动恢复初值或消息过期规则。重启时才重新使用启动参数。下面保留话题使用示例，仅作为说明，本轮没有执行：

```bash
# 设置附加自旋速度为 +1 rad/s，约每秒 57.3°
ros2 topic pub --once /cmd_spin example_interfaces/msg/Float32 "{data: 1.0}"

# 改为反方向自旋
ros2 topic pub --once /cmd_spin example_interfaces/msg/Float32 "{data: -1.0}"

# 关闭附加自旋
ros2 topic pub --once /cmd_spin example_interfaces/msg/Float32 "{data: 0.0}"
```

上面是无命名空间示例。相对话题名会受启动命名空间影响，常见仿真为 `/red_standard_robot1/cmd_spin`；实际以启动的 namespace为准，实物也不一定就是根目录话题。

**附加量设为0，只关自旋附加项；输入速度全零，也不会自动清掉自旋附加项。** 输入全零且 `spin_speed_≠0` 时，转换仍会输出非零角速度。收到 `cmd_spin` 也只改变量，没有立刻发布新 `cmd_vel`；之后进入速度处理回调才体现新值。它本身不是持续输出速度的独立旋转控制器。

源码直接相加，没有对最终角速度限幅或平滑。上游平滑器处理的是相加之前的速度，不能据其限值认定相加后的总量已受同样限制。是否适合实车需结合底盘能力及下位机限制核对，本轮不调整转速。

###### local_plan_topic、use_sim_time 与源码疑点

本节点近似配对的是 `Odometry` 和 `Path`，不是带时间戳的速度；输入速度没有时间戳，只缓存最近值。收到局部路径会刷新“控制器活跃”时间；最近0.5秒内有路径时，非零速度等待配对回调处理，否则直接按当前保存的 yaw 转换。近似配对不能证明路径、位姿、速度对应同一时刻。

以下是源码能确认的缺口，实际影响尚未运行验证：

- **首帧方向：** `current_robot_base_angle_` 未显式初始化，发布TF与转换速度前也没有“已取得有效 yaw”的保护；首个有效里程计前的结果不能视为可靠。
- **旧速度复用：** 配对回调使用后没有清空缓存速度。零速度走直接发布分支，却没有清掉先前缓存的非零指令；之后存在再次通过配对回调发布旧指令的路径。
- **超时含义：** 写死的0.5秒只检查最近是否收到局部路径，不检查速度指令的新鲜度，也不是自动停车机制；20 ms定时器只发布TF，不会定时发布停车速度。
- **时钟混用：** 活跃时间判断使用默认 `rclcpp::Clock()`，Humble中是系统时间；TF时间戳使用节点时钟，可受 `use_sim_time` 影响，定时器又用真实时间。True不能让这些步骤统一按仿真时钟运行。

这里不展开此前搁置的导航走停或 `map→odom` 时效排查，仅记录本节点的代码前提。

###### 启动来源与当前验证范围

整套 reality／simulation 启动分别默认使用对应目录的 `nav2_params.yaml`，可由 `params_file` 指定；导航 launch给节点或组件加载处理后的参数文件，并重写已有 `use_sim_time`。两种模式都把该节点命名为 `fake_vel_transform`，匹配 YAML。

包内独立 `fake_vel_transform_launch.py` 只传 `use_sim_time`，默认 True，不读取整套 YAML；其余采用C++默认值，例如参考系 `gimbal_link/gimbal_link_fake`、里程计 `odom`、自旋0，输入／输出速度话题默认还是空字符串。不能把这个独立入口当作已配置好的整套导航启动。

8项自定义参数仅在构造时读取，没有动态参数更新回调；运行时写同名参数不会更新成员、重建话题或改变初始自旋。ROS标准的 `use_sim_time` 由节点时钟处理，运行时改变自旋应通过 `cmd_spin`，两者不要混淆。

本轮在 `ysu_ros_humble` 中只查询环境、安装路径与文件，确认独立 launch、导航 launch和两份 YAML 与当前源码文件一致，并核对Humble时钟默认类型；没有启动节点、发送上述消息或进行运动测试。

- **源码能确认：** 导航输入与输出的参考系约定、二维速度换轴、角速度直接相加、串口原样传值、实物／仿真默认差异、参数读取方式及上述代码缺口。
- **仿真已观察：** 2026-10-08仅检查独立 Point-LIO 的静止和约20 cm往返；没有验证本节点、独立云台 yaw、MCU速度换算、附加自旋或停车行为。本节90°例子与相加表均为计算示例。
- **必须实物或下位机源码核实：** 轴向与单位、机械相对 yaw零位及方向、输入位姿是否正确、MCU是否把云台系线速度转到底盘执行方向、最终角速度的执行对象与限制，以及是否重复叠加自旋。此前重力方向和点时间单位问题仍保留待核实。

源码位置：`src/pb2025_sentry_nav/fake_vel_transform/src/fake_vel_transform.cpp`、对应头文件及 README；上游：`sensor_scan_generation`、`pb_omni_pid_pursuit_controller` 和导航 launch；串口：`src/serial_comm/src/serial_comm.cpp`。本轮仅修改日志这一参数块，工程源码、配置和参数值均保留。

