---
title: "pb_teleop_twist_joy_node（手柄输入与可替换控制入口）"
---

## pb_teleop_twist_joy_node（手柄输入与可替换控制入口）

### 是否必须使用北极熊的手柄节点

**可以不用，可以自己写。对当前工程，更建议保留现成joy／joy_linux手柄驱动，继续完善已有joy_bringup的映射与控制逻辑；北极熊节点保留作原工程复现参考。** 自主导航需要目标请求和速度执行链，不要求必须通过本手柄节点发目标。

本节点提供两种不同功能：manual_control把摇杆映射成底盘速度，auto_control把摇杆映射成相对机器人的导航目标；两种模式都会把云台摇杆积分成关节目标位置。当前配置为auto_control，因此底盘x、y的2.5／4.0在这条分支是**目标偏移米数**，不是实车速度上限。[上游模式说明与源码](https://github.com/SMBU-PolarBear-Robotics-Team/pb_teleop_twist_joy/blob/4a26966cb7958553677f4fe876d003c682c52b6d/src/pb_teleop_twist_joy.cpp)

| 方案 | 适用情况 | 对当前工程的建议 |
|---|---|---|
| 沿用pb_teleop_twist_joy | 需要复现原包的手柄发Nav2前视目标、云台联动功能 | 可以作对照，但先核对本页的目标、取消、限位和启动问题 |
| 完善自己的joy_bringup | 手动测试底盘／云台，需要符合自己的协议与控制方式 | **建议作为后续维护方案**；现有实现可继续用，补齐使能、模式选择和统一输出出口 |
| 官方teleop_twist_joy加独立云台节点 | 主要需要通用Joy→Twist，愿意分开处理云台 | 可以复用底盘映射；官方通用节点不会自动提供本包的云台积分或手柄发导航目标功能 |

上游README本身说明代码较乱、不建议继续基于它扩展，且README更新不及时；这可作为维护选择的参考，具体功能仍以源码为准。官方teleop_twist_joy提供Joy→Twist映射，同样依赖joy处理输入重发，不自带整套机器人控制模式管理。[北极熊README](https://github.com/SMBU-PolarBear-Robotics-Team/pb_teleop_twist_joy/blob/4a26966cb7958553677f4fe876d003c682c52b6d/README.md)、[官方Humble说明](https://github.com/ros2/teleop_twist_joy/blob/humble/README.md)

#### 当前“不使用北极熊节点”的注释与启动不一致

reality参数块前面已经写了“我们用的自己的teleop_twist_joy_node，不使用北极熊的节点”。但当前实物和仿真主入口仍无条件添加joy_teleop_launch；这个launch实际启动joy的joy_node和pb_teleop_twist_joy_node，安装副本也相同。**注释没有停掉节点，本参数块不能仅凭注释标成已废弃。**

对应位置：[实物入口](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/launch/rm_navigation_reality_launch.py:194)、[仿真入口](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/launch/rm_navigation_simulation_launch.py:173)、[实际启动节点](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/launch/joy_teleop_launch.py:100)。这里只核对文件，没有据此声称当前进程一定在运行。

自己的joy_bringup是另一套独立启动入口，不读取本pb参数分组。若确定替换，应同时调整启动入口、参数与速度出口；仅删YAML块会让仍被启动的pb节点改用源码默认值，不等于禁用它。

```text
北极熊当前auto_control：
    joy → 摇杆x/y × scale → 机器人参考系中的偏移点
        → TF换到map → NavigateToPose → Nav2规划／控制 → 速度链
    joy → 云台角速度 × ROS dt积分 → cmd_gimbal_joint目标位置

自己的joy_bringup：
    joy_linux → /joy → joy_node_cpp
        ├─ 直接底盘Twist → cmd_vel
        └─ 定时积分云台目标 → cmd_gimbal_joint
    不读取本块的enable_button、scale_chassis等参数

替换时的建议接线，尚未实施：
    导航最终速度 ─┐
                  ├─ 明确的模式／指令选择 → 唯一执行端输入
    手动速度 ─────┘
    两路先统一坐标、单位和自旋含义
```

| 实车上看到的现象 | 先核对什么 | 当前源码怎样理解 |
|---|---|---|
| 已有自己的手柄节点，却还出现pb发目标 | 主入口是否仍包含joy_teleop_launch | 当前两套主入口仍会启动pb，注释没有改变启动逻辑 |
| 把auto_control理解为MCU的自动模式 | 发布的是Twist还是NavigateToPose | 本字符串只选择pb内部发送方式，不调用serial的set_mode |
| 设scale_chassis.x=2.5，以为车速必为2.5 | 当前control_mode | auto中为目标偏移2.5 m；manual才是线速度比例m/s |
| 按R1没按L1也能控制 | turbo优先判断 | R1本身可进入turbo，不要求同时按L1 |
| 摇杆回中但上一导航目标仍在走 | 回中分支与取消分支 | auto回中只不发新目标；释放使能才走取消路径 |
| 以为关节scale是目标角度上限 | 云台积分公式 | pitch／yaw scale是角速度比例，积分位置没有本包内置上下限 |
| 自己节点空闲仍覆盖导航速度 | 是否持续向最终cmd_vel发零 | 现有joy_bringup每10 ms发布，失去输入也发零；需要模式选择，不能与导航直接竞争同一最终话题 |

本次本地及安装pb包版本为**1.0.2**，安装头文件与本地一致；本地cpp／头文件去除空白后与上游提交**4a26966cb7958553677f4fe876d003c682c52b6d**一致。原块和安装副本一致，simulation仅时钟为True。本文保留参数说明，同时明确它是可替换的手柄入口，不是必须选用的导航组件。

### 参数

下面原样保留发送的24项参数及注释。表格解释**pb实现**，不会把这些值套到自己的joy_bringup。若以后从启动中完整移除pb，本组参数才不再参与该节点运行。

```yaml
pb_teleop_twist_joy_node:
  ros__parameters:
    use_sim_time: False
    robot_base_frame: gimbal_yaw
    control_mode: auto_control    # Option: auto_control, manual_control

    require_enable_button: true
    enable_button: 4              # L1 shoulder button
    enable_turbo_button: 5        # R1 shoulder button

    axis_chassis:
      x: 1                        # Left thumb stick vertical
      y: 0                        # Left thumb stick horizontal
      yaw: 6                      # button_left and button_right
    scale_chassis:
      x: 2.5
      y: 2.5
      yaw: 3.0
    scale_chassis_turbo:
      x: 4.0
      y: 4.0
      yaw: 6.0

    axis_gimbal:
      roll: -1                    # Disable
      pitch: 4                    # Right thumb stick vertical
      yaw: 3                      # Right thumb stick horizontal
    scale_gimbal:
      roll: 0.0
      pitch: -1.0
      yaw: 2.5
    scale_gimbal_turbo:
      roll: 0.0
      pitch: -1.5
      yaw: 3.5
```

#### ros__parameters（时钟、参考系、模式与使能）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真True | 不适用，布尔 | **上实物前必须核对并正确配置** | 决定云台积分dt、目标节流时间和消息stamp使用的ROS时钟 | 不适用 | 不适用 | launch重写已有键；本pb源码没有自己的断流定时器；暂停／回跳／长间隔会影响积分和目标发送 |
| `robot_base_frame` | `gimbal_yaw`；源码默认base_link | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | auto中指定偏移点所在系，查询map←此系TF | 不适用 | 不适用 | 当前目标偏移沿实际云台轴；manual默认Twist无frame_id，不因这个值自动换轴；Stamped模式仅写header |
| `control_mode` | `auto_control`；源码默认manual_control | 不适用，发送方式枚举 | **根据实际功能需求开启或设置** | 选择手柄发送导航目标或直接底盘速度 | 不适用 | 不适用 | auto走NavigateToPose，manual走Twist；不是MCU模式、不是导航接管仲裁；需用这两个准确名称 |
| `require_enable_button` | `true`；源码默认True | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 普通档是否要求按enable_button才能发送底盘／云台指令 | 不适用 | 不适用 | True普通档需L1；turbo另有优先分支，R1可单独使能；False普通档无需按键，不等于提供急停功能 |
| `enable_button` | `4`；注释L1 | 无量纲，Joy.buttons索引 | **上实物前必须核对并正确配置** | 普通档使能按钮 | 不适用，换成其他按钮索引 | 不适用，换成其他按钮索引 | 按实际Joy按钮数组核对，注释不保证所有手柄／驱动都相同；require=True且无turbo时使用 |
| `enable_turbo_button` | `5`；注释R1 | 无量纲，Joy.buttons索引 | **根据实际功能需求开启或设置** | 优先选择turbo档并使能控制 | 不适用，换成其他按钮索引 | 不适用；-1表示禁用turbo | 两键都按时turbo优先；R1无需L1；高速档是否需要由真实操作需求决定 |

##### 参数详解

###### auto_control不是“开关自动驾驶模式”

auto按axis_chassis.x／y读取摇杆，再乘当前档scale，作为robot_base_frame中的目标位置偏移，查询最新map←robot_base_frame TF转换后发NavigateToPose。节流为ROS时间至少间隔0.25 s，名义最多4 Hz；实际频率还取决于Joy回调、摇杆幅度和TF。[目标生成](https://github.com/SMBU-PolarBear-Robotics-Team/pb_teleop_twist_joy/blob/4a26966cb7958553677f4fe876d003c682c52b6d/src/pb_teleop_twist_joy.cpp#L195)

当前普通档摇杆x=1表示在云台轴前方取2.5 m偏移，turbo取4 m；不是要求底盘分别以2.5／4 m/s行驶。实际速度由Nav2控制器及速度链决定。目标会随新Joy更新成新的前视点，不是直接转发摇杆速度，也不通过axis_chassis.yaw给目标设朝向。

源码只填写局部Pose的position.x／y，没有给orientation设置单位四元数；目标姿态构造需要修正／核对，不能把这条分支直接当作已经验证的完整导航目标接口。没有等待Action服务就绪或保留本节点的已接受目标句柄，本次未调用该Action验证结果。

manual则直接发缩放后的Twist，不经本节点TF换轴，也没有内置加减速平滑。默认输出cmd_vel，当前导航平滑器订阅的是cmd_vel_controller；不能因导航有velocity_smoother就认为手柄直发速度也自动经过它。[速度链与范围](/wiki/2026-09-20-dao-hang/0500-2000-velocity-smoother)

###### 松开使能、摇杆回中与断流

enable_turbo_button先判断，满足时直接turbo；否则检查普通档使能。失去使能时，在此前已发送过控制的条件下调用一次sendZeroCommand：auto向navigate_to_pose取消当前时刻以前的目标，同时发布零Twist；不是持续发送零，也不等待确认取消已经完成。[按键分支与停止](https://github.com/SMBU-PolarBear-Robotics-Team/pb_teleop_twist_joy/blob/4a26966cb7958553677f4fe876d003c682c52b6d/src/pb_teleop_twist_joy.cpp#L119)

auto中abs(缩放后x)≤0.1且abs(缩放后y)≤0.1，只提前返回，不会取消上一目标。仍按着使能而松开摇杆时，车可能继续完成此前目标，不能用“摇杆回中”代替Nav2取消。0.1比较的还是缩放后的坐标偏移，普通档与turbo对应的原始摇杆范围不同。

取消按时间作用于同一个navigate_to_pose服务端，不只限定本节点保存的某个目标句柄；若其他决策客户端也发目标，需核对目标所有权。自己实现时宜保存并取消本次手柄目标，而不是靠跨客户端的时间取消充当模式仲裁。

本pb节点没有Joy输入年龄检查，完全断流时不会再进入释放按键回调。云台积分dt也没有限制长间隔或回跳；重新来消息时可能把新角速度乘整个间隔。头文件的sent_disable_msg_没有显式初始化，这个停止标志还需要修正后才能依赖其初始状态。上述为源码边界，本次没有断流或运动验证。

#### 底盘轴与比例（auto目标偏移、manual速度）

##### 参数表

这里使用完整参数名，明确每个分组的作用。轴索引是Joy.axes，不是Joy.buttons；注释中的左右按钮可能由驱动表达为一根轴，需核对实际消息。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `axis_chassis.x` | `1` | 无量纲，axes索引 | **上实物前必须核对并正确配置** | 选择底盘前后输入轴 | 不适用，改输入轴 | 不适用；-1禁用 | auto用于目标x偏移，manual用于linear.x；不是速度或距离数值 |
| `axis_chassis.y` | `0` | 无量纲，axes索引 | **上实物前必须核对并正确配置** | 选择底盘左右输入轴 | 不适用，改输入轴 | 不适用；-1禁用 | auto用于目标y偏移，manual用于linear.y；需与参考系正方向一致 |
| `axis_chassis.yaw` | `6` | 无量纲，axes索引 | **当前运行不使用这个参数，暂不调整** | manual选择底盘旋转输入轴 | 不适用，改输入轴 | 不适用；-1禁用 | 当前auto目标仅取x、y，不用此轴生成底盘转角；6不是buttons[6] |
| `scale_chassis.x` | `2.5` | 当前auto为m／单位轴值；manual为m/s／单位轴值 | **根据实际功能需求开启或设置** | 普通档前后目标偏移／速度比例 | 正值增大时偏移更远；manual目标速度更大 | 正值减小时偏移更近；manual速度更小 | 当前是目标偏移，不是Nav2速度限制；系数符号决定方向 |
| `scale_chassis.y` | `2.5` | 当前auto为m／单位轴值；manual为m/s／单位轴值 | **根据实际功能需求开启或设置** | 普通档左右目标偏移／速度比例 | 正值增大时横向偏移更远；manual横移更快 | 正值减小时偏移更近；manual横移更慢 | auto的输入门槛在乘比例后检查；不按合距离统一限幅 |
| `scale_chassis.yaw` | `3.0` | rad/s／单位轴值，仅manual | **当前运行不使用这个参数，暂不调整** | 普通档底盘角速度比例 | manual正值增大时转动幅值更大 | manual正值减小时转动幅值更小 | 当前auto未使用；不是自动导航目标yaw或云台角速度 |
| `scale_chassis_turbo.x` | `4.0` | 当前auto为m／单位轴值；manual为m/s／单位轴值 | **根据实际功能需求开启或设置** | turbo前后偏移／速度比例 | 正值增大时目标更远；manual速度更大 | 正值减小时目标更近；manual速度更小 | R1档才使用；manual4 m/s直发不自动受导航2.5限制 |
| `scale_chassis_turbo.y` | `4.0` | 当前auto为m／单位轴值；manual为m/s／单位轴值 | **根据实际功能需求开启或设置** | turbo左右偏移／速度比例 | 正值增大时横向目标更远；manual横移更快 | 正值减小时目标更近；manual横移更慢 | x、y分别乘比例，若轴均1，auto偏移模长约5.66 m；这是计算例 |
| `scale_chassis_turbo.yaw` | `6.0` | rad/s／单位轴值，仅manual | **当前运行不使用这个参数，暂不调整** | turbo底盘角速度比例 | manual正值增大时转动幅值更大 | manual正值减小时转动幅值更小 | 当前auto未使用；与fake_vel_transform的cmd_spin是不同输入 |

getVal按axes[index]×scale计算；-1、缺少映射或超过消息轴数时返回0。它没有把输入轴统一夹到[-1,1]、检查有限值或在这里做死区；假定正常轴值范围的满杆输出只是映射推算，不能当作异常输入的硬上限。手柄型号、joy与joy_linux驱动不同，索引和正负方向均须用实际Joy消息核对。

#### 云台轴与比例（角速度积分成目标位置）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `axis_gimbal.roll` | `-1` | 无量纲，axes索引 | **当前运行不使用这个参数，暂不调整** | 配置了roll映射，当前禁用 | 不适用，换轴 | 不适用，-1禁用 | fillJointStateMsg实际上只输出pitch与yaw，改此轴不会新增roll关节控制 |
| `axis_gimbal.pitch` | `4` | 无量纲，axes索引 | **上实物前必须核对并正确配置** | 选择云台pitch输入轴 | 不适用，换轴 | 不适用，-1禁用 | 两种control_mode都使用；方向由比例正负及机械零位共同核对 |
| `axis_gimbal.yaw` | `3` | 无量纲，axes索引 | **上实物前必须核对并正确配置** | 选择云台yaw输入轴 | 不适用，换轴 | 不适用，-1禁用 | 两种control_mode都使用；是云台相对关节目标，不是底盘旋转 |
| `scale_gimbal.roll` | `0.0` | 当前未进入关节计算 | **当前运行不使用这个参数，暂不调整** | 保存普通档roll比例 | 当前实现无roll输出变化 | 当前实现无roll输出变化 | 不是已实现的roll限位或角度设置 |
| `scale_gimbal.pitch` | `-1.0` | rad/s／单位轴值 | **上实物前必须核对并正确配置** | 普通档pitch积分速度比例，负号反向 | 负值向0提高时速度幅值减小 | 负值更负时速度幅值增大 | 不是pitch目标角度下限；持杆位置按dt累计，没有本包内置软限位 |
| `scale_gimbal.yaw` | `2.5` | rad/s／单位轴值 | **根据实际功能需求开启或设置** | 普通档yaw积分速度比例 | 正值增大时目标角累计更快 | 正值减小时累计更慢 | 不是yaw目标角度上限；没有自动按实际关节反馈重新对齐 |
| `scale_gimbal_turbo.roll` | `0.0` | 当前未进入关节计算 | **当前运行不使用这个参数，暂不调整** | 保存turbo的roll比例 | 当前实现无roll输出变化 | 当前实现无roll输出变化 | 改此值不产生roll关节命令 |
| `scale_gimbal_turbo.pitch` | `-1.5` | rad/s／单位轴值 | **上实物前必须核对并正确配置** | turbo的pitch积分速度比例 | 负值向0提高时幅值减小 | 负值更负时幅值增大 | R1档使用，当前幅值为普通档1.5倍；仍需另行处理pitch位置限位 |
| `scale_gimbal_turbo.yaw` | `3.5` | rad/s／单位轴值 | **根据实际功能需求开启或设置** | turbo的yaw积分速度比例 | 正值增大时累计更快 | 正值减小时累计更慢 | R1档使用；比例不是输出位置的限幅 |

##### 云台积分、初始位置与时间

源码保存从0开始的static pitch、yaw，每次已使能Joy回调更新：

```text
pitch_target += axes[4] × 当前档pitch比例 × ROS dt
yaw_target   += axes[3] × 当前档yaw比例   × ROS dt
发布JointState：name=[gimbal_pitch_joint, gimbal_yaw_joint]
                position=[pitch_target, yaw_target]
```

position单位为rad，比例单位为rad/s；例如普通档pitch轴为+1且dt=0.05 s，一次目标增量为-0.05 rad。这是公式示例，不是实测云台角度。两个目标没有按当前实测关节初始化，也没有pitch夹紧、yaw行程保护或dt异常保护。[积分实现](https://github.com/SMBU-PolarBear-Robotics-Team/pb_teleop_twist_joy/blob/4a26966cb7958553677f4fe876d003c682c52b6d/src/pb_teleop_twist_joy.cpp#L182)

松开使能不会发布新的云台停止位置，而是停止调用该积分发送分支；末次位置是否继续执行由接收端决定。控制节点也没有自己的固定积分定时器，回调频率由上游Joy消息和重发决定。当前PB joy launch配置deadzone=0.3、autorepeat_rate=20 Hz，这两项属于输入驱动配置，不在原24项参数中。

#### 相关源码默认项（原YAML未显式配置）

##### 参数表

下面仅补两项与底盘输出格式有关的默认值，没有向原YAML追加。pb源码另有其他映射与发布功能，本页不把未发送的配置当作已经完整关闭全部额外输出。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `publish_stamped_twist` | `False`，源码默认 | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 选择cmd_vel发布Twist还是TwistStamped | 不适用 | 不适用 | 当前接收链用Twist；True只加header，不自动转换参考系，也不能与Twist订阅器直接匹配 |
| `inverted_reverse` | `False`，源码默认 | 不适用，布尔 | **当前运行不使用这个参数，暂不调整** | manual倒退且linear.x<0时，可反转底盘angular.z | 不适用 | 不适用 | 当前auto不走fillCmdVelMsg；不是反转云台yaw／pitch，也不是通用坐标转换 |

除use_sim_time由rclcpp时间源管理外，本页pb功能参数在构造时读取，没有对应的参数更新回调。运行中改control_mode、按键或比例，不等于已有成员和Action客户端已经联动改变；需重启该节点再核对。也不能把自己的节点命名成同一个名字，就认为它自动拥有这些参数实现。

#### 自己的joy_bringup已经有哪些功能，还缺什么

本次读到自己的Joy_Node／joy_node_cpp源码及已安装launch：

| 内容 | 自己的当前实现 | 相对于本块的含义 |
|---|---|---|
| 手柄读取 | joy_linux_node，/dev/input/js0，配置50 Hz重发 | 可继续复用驱动，不必从USB设备读取重新写起 |
| 底盘映射 | vx=axes[1]、vy=axes[0]、wz=axes[6]×π | 没读取pb的2.5／4.0或L1／R1；按正常轴范围为±1 m/s、±π rad/s的请求 |
| 云台映射 | yaw速度axes[3]×2.5，pitch速度axes[4]×1.0 | pitch符号与pb的-1.0不同；结合机械和操作方向核对，不凭注释认定哪边正确 |
| 云台积分 | 10 ms墙钟定时，按ROS dt积分；仅0<dt≤0.1时积分 | 已处理暂停、回跳及过长dt；不是pb的Joy回调间隔积分 |
| 输入保护 | 检查至少7轴和已用轴有限值；steady_clock超过0.5 s置零速度 | 已有断流置零路径，不依赖再收到松开按键消息 |
| pitch范围 | 当前源码夹到[-0.60,0.45] rad | 已有软限位，但这是当前代码值，仍需核对真实机械范围 |
| 按键与档位 | 未读取Joy.buttons，未做L1／R1使能和turbo | 不是完整复刻本pb配置，按需要补功能即可 |
| 导航／手动切换 | 没有Nav2目标客户端或统一指令选择；每10 ms发Twist | 空闲也持续发零，若与导航共享最终cmd_vel可能覆盖导航，必须处理接线与控制权 |

源码：[自己的手柄控制器](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/joy_bringup/src/joy_controller.cpp:22)、[实物启动](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/joy_bringup/launch/joy_controller.launch.py:22)、[仿真启动](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/joy_bringup/launch/joy_controller_sim.launch.py:28)。源码中有这些路径，不代表已验证编译后的运动效果；本次没有启动它。

继续完善自己的方案时，优先明确以下四件事：

1. **启动选择：** 替换后主入口只启动选定的手柄驱动和控制器；移除／关闭pb的Include，不靠删参数来停节点，避免两套驱动与映射一起启动。
2. **指令出口与模式：** 导航和手动输入分别进入可辨认的话题，由一个模式选择环节送到执行端；手动未取得控制权时不要持续向最终话题发零。切换时处理已有导航目标和云台目标所有权。
3. **坐标与单位：** 导航最后cmd_vel按实际gimbal_yaw轴表达。自己的源码只直接拷贝摇杆，未查询TF；手动速度究竟按底盘还是云台轴定义，必须与MCU协议一致。输出JointState仍按pitch、yaw位置rad理解。[当前最终速度约定](/wiki/2026-09-20-dao-hang/0500-0600-fake-vel-transform)
4. **使能、限值与反馈：** 保留已有断流和dt保护，按需要加手柄使能、速度范围、云台初始化和实际机械限位；auto若要用手柄发导航目标，再单独实现有效Pose和目标句柄管理。

替换手柄节点不要求重写Nav2的规划器、控制器或速度平滑器，也不要求同时改MCU协议；是否需要改协议由当前接口是否能表达选定控制方式决定。本次只提供选择和适配依据，没有代替用户修改启动、手柄代码或协议。

#### 本次核对与验证范围

- **已核对：** pb包1.0.2与头文件、固定上游源码；24项原参数和两套安装副本；实物／仿真入口及joy_teleop安装副本；自己的joy_bringup源码、输入驱动配置和安装launch；串口订阅cmd_vel及cmd_gimbal_joint。
- **源码能确认：** auto偏移与manual速度的不同单位；turbo优先使能；云台角速度积分；当前yaw／roll不使用项；启动注释与实际Include不一致；两套节点在指令输出上的适配缺口。
- **建议：** 继续完善自己的joy_bringup，保留北极熊配置作参考；如暂时完整复现北极熊auto功能，可先使用其节点，但须处理本页列出的实现条件，不能同时任由两套节点向同一执行入口发送。
- **本次范围：** 仅整理文档与目录，未更改主入口、节点配置或源代码，未查询／改变当前控制模式，未发送Joy、速度、关节或导航目标，未做实车／仿真运动验证。
