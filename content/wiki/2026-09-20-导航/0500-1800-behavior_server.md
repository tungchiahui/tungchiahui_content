---
title: "behavior_server（恢复行为与空闲方向移动）"
---

## behavior_server（恢复行为与空闲方向移动）

### 实车先看：当前恢复动作、版本接口与运动检查

**当前默认行为树实际调用的是BackUp，后端为自定义BackUpFreeSpace；它寻找空闲方向做平面移动，不保证朝车后方倒退。** Spin、DriveOnHeading、AssistedTeleop、Wait会加载，但没有进入当前两棵默认树。清代价地图则调用costmap服务，不是由这五个行为插件完成。

本块还有两类需要先解决的问题，不能只按数值大小调参：

- **版本接口：** 本次安装的Nav2 1.1.20读取单组costmap_topic、footprint_topic，不读取本YAML中的四个local／global话题键或local_frame。实际碰撞检查默认接局部图，位姿却按global_frame=map取得，需要统一。
- **自定义实现：** BackUpFreeSpace覆盖onConfigure而未调用父类，继承的预测时长保持0，周期碰撞预测循环不执行；选向函数没有安全方向时仍返回0。仅增大simulate_ahead_time或max_radius不能修复这些路径。

```text
默认BT导航失败
    → 清局部／全局代价地图服务，与BackUp轮换恢复
    → BackUp：target.x=1 m，speed=1 m/s，time_allowance默认10 s
        → BackUpFreeSpace
            ├─ global_costmap/get_costmap服务：取一次当前全局图
            ├─ map中的机器人位置：扫描各方向0～max_radius射线
            ├─ 试图选择较宽的安全方向区间
            └─ 按所选角度发vx、vy，按离初始位置的平面距离判断完成

Nav2 1.1.20共有的碰撞检查输入：
    costmap_topic、footprint_topic → 单个CostmapTopicCollisionChecker
    当前默认为local_costmap话题；不会因四个双地图键自动创建两套检查器
```

这里的1 m、1 m/s来自默认XML端口，并非本YAML中的max_radius=2；不能据此认定实车已经正确以该速度恢复。图中的移动流程还受后面列出的选向、坐标、预测和输出接线问题影响。[当前BT恢复分支](/wiki/2026-09-20-dao-hang/0500-1000-bt-navigator)

| 实车上看到的现象 | 先核对什么 | 当前配置怎样理解 |
|---|---|---|
| 以为恢复自动包含转动、等待和后退 | 实际XML及backup后端类 | 默认树只调用自定义BackUp；列表不代表按顺序执行所有行为 |
| 写了四个地图话题，以为各行为已经选用正确地图 | 安装版本及真正读取的参数 | 1.1.20只有单组costmap_topic／footprint_topic；四个双地图键未使用 |
| 设simulate_ahead_time=2，却没有周期碰撞预测 | 自定义onConfigure与父类成员 | Spin、标准DriveOnHeading读取2；自定义BackUp成员仍为0，循环直接跳过 |
| 周围没有安全路线，却仍选正x方向 | 无安全方向的返回分支 | 初始扇区边界为0，函数没有失败标志；全障碍离线例子也返回0 |
| 可视化箭头方向与运动不一致 | 地图角度与Twist隐含机体系 | 自定义角度在map轴上计算，但没有按机器人theta转换后再发vx、vy |
| 以为2 m就是倒退距离 | max_radius与BackUp请求 | 2 m是选向射线需要检查的长度；当前请求完成距离是1 m |
| 修改PID／velocity_smoother限值，恢复速度没有同样变化 | 行为输出与两种launch分支 | 行为使用自己的请求速度；组件输出跳过velocity_smoother，独立分支还缺少到fake_vel_transform输入的重映射 |
| 认为所有失败都会发零速度 | 实际失败分支与下游自旋 | TF失败等路径未统一stopRobot；组件发零Twist也不会自动清掉cmd_spin附加量 |

本次nav2_behaviors为**1.1.20**，pb_nav2_plugins为**1.0.1**。服务器、TimedBehavior、DriveOnHeading安装头文件与对应Nav2版本一致；本地BackUpFreeSpace实现／头文件与上游提交**e6778ca44d9e88aa612a3c6c87d536a8cfa21705**一致，安装头文件也一致。下文按这套实现说明。[1.1.20服务器](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/src/behavior_server.cpp)、[自定义恢复实现](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca44d9e88aa612a3c6c87d536a8cfa21705/src/behaviors/back_up_free_space.cpp)

### 参数

下面原样保留发送的23项参数与原注释。源码和安装副本的reality段一致；simulation段仅use_sim_time=True不同。版本适配建议另列，没有改工程YAML或行为代码。

```yaml
behavior_server:
  ros__parameters:
    use_sim_time: False
    local_costmap_topic: local_costmap/costmap_raw
    global_costmap_topic: global_costmap/costmap_raw
    local_footprint_topic: local_costmap/published_footprint
    global_footprint_topic: global_costmap/published_footprint
    cycle_frequency: 10.0
    behavior_plugins: ["spin", "backup", "drive_on_heading", "assisted_teleop", "wait"]
    spin:
      plugin: "nav2_behaviors/Spin"
    backup:
      plugin: "pb_nav2_behaviors/BackUpFreeSpace"
    drive_on_heading:
      plugin: "nav2_behaviors/DriveOnHeading"
    wait:
      plugin: "nav2_behaviors/Wait"
    assisted_teleop:
      plugin: "nav2_behaviors/AssistedTeleop"
    local_frame: odom
    global_frame: map
    robot_base_frame: gimbal_yaw_fake
    transform_tolerance: 0.1
    simulate_ahead_time: 2.0
    max_rotational_vel: 1.0
    min_rotational_vel: 0.4
    rotational_acc_lim: 3.2
    # params for pb_nav2_behaviors/BackUpFreeSpace
    service_name: "global_costmap/get_costmap"
    max_radius: 2.0
    visualize: false
```

本页把“实现未读取的键”“已读取但默认BT未调用对应行为”“当前BackUp实际使用的参数”分别写在备注中。表中的调大／调小效果仅指对应实现分支，不表示这些值同时限制所有导航速度。

#### ros__parameters（更新、地图接口与坐标）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真True | 不适用，布尔 | **上实物前必须核对并正确配置** | 决定行为期限、等待时长、反馈时间及TF的ROS时间域 | 不适用 | 不适用 | navigation_launch重写已有键；行为执行循环另用WallRate，ROS时钟暂停不等于循环暂停 |
| `local_costmap_topic` | `local_costmap/costmap_raw`，当前键未读取 | 不适用，话题名称 | **已确认当前值错误，必须修改** | 原配置意图指定局部图输入；1.1.20不支持此键 | 不适用 | 不适用 | 错误在版本参数接口，不是话题字符串；本版应使用单组costmap_topic并明确选择哪张图 |
| `global_costmap_topic` | `global_costmap/costmap_raw`，当前键未读取 | 不适用，话题名称 | **已确认当前值错误，必须修改** | 原配置意图指定全局图输入；1.1.20不支持此键 | 不适用 | 不适用 | 若检查器选全局图，应把实际costmap_topic设为该话题；当前自定义选向另通过service_name取全局图 |
| `local_footprint_topic` | `local_costmap/published_footprint`，当前键未读取 | 不适用，话题名称 | **已确认当前值错误，必须修改** | 原配置意图指定局部轮廓输入；1.1.20不支持此键 | 不适用 | 不适用 | 应使用单组footprint_topic；需与所选地图、robot_base_frame约定一致 |
| `global_footprint_topic` | `global_costmap/published_footprint`，当前键未读取 | 不适用，话题名称 | **已确认当前值错误，必须修改** | 原配置意图指定全局轮廓输入；1.1.20不支持此键 | 不适用 | 不适用 | 若选择全局检查，应配置真正读取的footprint_topic；四键不会在本版自动建立两套检查器 |
| `cycle_frequency` | `10.0` | Hz，活动行为的目标执行循环频率 | **实物运行时重点观察，有问题再针对性调整** | 控制TimedBehavior调用onCycleUpdate及反馈／速度的节拍 | 目标循环更密，计算与发布增加；标准预测采样也更密 | 指令与取消检查可能更稀，标准预测采样更粗 | 当前目标100 ms／轮，不是重规划／地图更新频率；应为正有限值；不能让自定义零时长预测重新生效 |
| `behavior_plugins` | spin、backup、drive_on_heading、assisted_teleop、wait | 不适用，有序插件ID列表 | **上实物前必须核对并正确配置** | 加载五个行为并为各ID提供Action | 不适用 | 不适用 | 顺序是加载顺序，不是恢复执行顺序；当前BT只调用backup，各ID需有对应plugin分组 |
| `local_frame` | `odom`，当前键未读取 | 不适用，坐标系名称 | **当前运行不使用这个参数，暂不调整** | 原配置意图设置局部参考系；1.1.20无此使用路径 | 不适用 | 不适用 | 本版TimedBehavior读取global_frame作为共同位姿参考系；此处odom不会让Spin／BackUp自动在odom统计 |
| `global_frame` | `map` | 不适用，位姿参考系名称 | **上实物前必须核对并正确配置** | 各TimedBehavior取得机器人位姿的共同参考系，自定义选向也用它 | 不适用 | 不适用 | 当前默认检查器接odom局部图，存在参考系组合不一致；须同时核对实际地图键和service_name，不宜单改此值为odom |
| `robot_base_frame` | `gimbal_yaw_fake` | 不适用，机体参考系名称 | **上实物前必须核对并正确配置** | 位姿、位移／转角反馈及轮廓换系的机器人参考点 | 不适用 | 不适用 | 虚拟系不等于真实底盘方向；圆形轮廓原点、map角度换到Twist，以及Spin转角反馈均需核对 |
| `transform_tolerance` | `0.1` | s，TF查询等待 | **通常保持当前值，不作为优先调试项** | 传给行为位姿查询和轮廓订阅的变换等待 | 可等更久，也可能拖慢100 ms行为循环 | 缺TF时更快返回失败 | 不是0.1 m／rad定位容差；运行中改参数不会同步改已缓存成员；TF失败分支还需核对零速处理 |

##### 参数详解

###### 当前1.1.20真正使用的是单地图接口

BehaviorServer声明并读取costmap_topic和footprint_topic，建立一个地图订阅、一个轮廓订阅和一个CostmapTopicCollisionChecker，再把同一个检查器交给各行为。没有读取上述四个双地图话题键，也没有为local_frame建立分支。[声明与订阅创建](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/src/behavior_server.cpp#L35)

本YAML没有写实际的两个键，因此当前取源码默认：

```text
costmap_topic   = local_costmap/costmap_raw
footprint_topic = local_costmap/published_footprint
```

话题是相对名称，例如namespace为/red_standard_robot1时，默认检查输入位于该机器人的local_costmap下。写了global_costmap_topic并不会改变这组订阅。另一方面，BackUpFreeSpace自己的service_name确实被读取，仍从全局服务取一次图；**选向服务的地图和共有检查器的地图是两条输入链**。

###### global_frame与地图坐标必须一起适配

当前TimedBehavior统一取global_frame=map中的gimbal_yaw_fake位姿。共有检查器却默认使用local_costmap的odom网格；它直接把候选pose.x／y当作所订阅地图坐标，没有在内部将map位姿变到odom。仅TF链存在不能自动修复这处使用方式。[共同位姿参数](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/include/nav2_behaviors/timed_behavior.hpp#L122)、[检查器坐标使用](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_topic_collision_checker.cpp#L68)

如果map与odom暂时恰好重合，错误组合可能不明显；定位有平移／转角修正后，不应继续把两系坐标混用。BackUp当前预测循环为0又会掩盖这一输入问题，不能以它“还能移动”证明检查有效。

若继续使用此Nav2版本，应明确选择一组自洽输入：

| 选择 | 行为位姿参考系 | 真正的地图／轮廓键 | 自定义选向服务 |
|---|---|---|---|
| 使用全局图 | global_frame=map | costmap_topic=global_costmap/costmap_raw；footprint_topic=global_costmap/published_footprint | 保持global_costmap/get_costmap，并核对返回frame |
| 使用局部图 | global_frame=odom | costmap_topic=local_costmap/costmap_raw；footprint_topic=local_costmap/published_footprint | 同时改为local_costmap/get_costmap，并核对窗口覆盖 |

**不能只把global_frame改成odom而保留全局选向服务**，否则自定义射线会把odom位置直接放进map网格。也不能只更换服务而不统一检查器与位姿。

下面仅展示与当前map／全局服务一致的1.1.20接口写法，不是完整修复补丁，未写入工程：

```yaml
behavior_server:
  ros__parameters:
    costmap_topic: global_costmap/costmap_raw
    footprint_topic: global_costmap/published_footprint
    global_frame: map
    robot_base_frame: gimbal_yaw_fake
    service_name: global_costmap/get_costmap
```

之后仍须处理本页列出的预测初始化、侧向预测、速度换系和无安全方向返回等代码缺口。如果改用支持双地图接口的其他Nav2版本，也要重新核对pb插件继承接口，不能只继续保留新名字就认定适配完成。

###### cycle_frequency（行为循环与预测采样）

TimedBehavior在onRun成功后，以WallRate(cycle_frequency)循环检查取消／抢占并执行onCycleUpdate。目标10 Hz即100 ms／轮；TF、碰撞、计算和发布耗时都会影响实际周期。onRun期间等待服务还没有进入这个循环，10 Hz不会让那段等待主动检查取消。

标准Spin与DriveOnHeading还用1／cycle_frequency作为预测采样步长，最多int(cycle_frequency×simulate_ahead_time)个采样。当前10与2名义最多20个，时间索引从0起，不是严格检查到第2秒；达到其提前结束条件时更少。按1 m/s计算，100 ms对应0.1 m指令位移尺度，这是配置计算，不是实测。

自定义BackUp继承成员时长为0，当前int(10×0)=0；改10或YAML中的2都不能替代缺少的成员初始化。[执行循环与取消](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/include/nav2_behaviors/timed_behavior.hpp#L210)

#### 行为插件（加载了哪些，默认树调用哪些）

##### 参数表

这里的“当前运行不使用”指默认BT没有发起该Action；插件仍会在生命周期配置时加载、读取自己的参数。外部客户端或更换XML后，使用条件会改变。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `spin.plugin` | `nav2_behaviors/Spin` | 不适用，插件查找名 | **当前运行不使用这个参数，暂不调整** | 提供相对转角的Spin Action | 不适用 | 不适用 | 未入默认树；读取三项旋转值与simulate_ahead_time；不是fake_vel_transform的cmd_spin附加自旋入口 |
| `backup.plugin` | `pb_nav2_behaviors/BackUpFreeSpace` | 不适用，插件查找名 | **上实物前必须核对并正确配置** | 提供按空闲方向移动的BackUp Action | 不适用 | 不适用 | 默认恢复会调用；不是标准nav2 BackUp的固定负x倒退；本实现预测初始化、选向及换系缺口需先处理 |
| `drive_on_heading.plugin` | `nav2_behaviors/DriveOnHeading` | 不适用，插件查找名 | **当前运行不使用这个参数，暂不调整** | 按机体系x方向请求移动 | 不适用 | 不适用 | 未入默认树；拒绝非零target.y／z，要求target.x与speed同号；不能按任意二维目标点直接驱动 |
| `wait.plugin` | `nav2_behaviors/Wait` | 不适用，插件查找名 | **当前运行不使用这个参数，暂不调整** | 按请求ROS时长等待并反馈剩余时间 | 不适用 | 不适用 | 未入默认树；正常等待／结束路径不发布零速度，不能把Wait本身当成刹停命令 |
| `assisted_teleop.plugin` | `nav2_behaviors/AssistedTeleop` | 不适用，插件查找名 | **当前运行不使用这个参数，暂不调整** | 对遥控Twist做预测碰撞检查后输出或缩放 | 不适用 | 不适用 | 未入默认树；使用自己的projection_time／simulation_time_step，不用simulate_ahead_time；侧向投影符号需核对 |

五个查找名均有当前安装XML注册，标准行为使用带斜杠别名，自定义也按pb别名加载。列表加载顺序不会让Wait自动排在BackUp后执行；是否调用和恢复顺序由BT或外部客户端决定。

#### simulate_ahead_time与旋转参数（对应行为分别使用）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `simulate_ahead_time` | `2.0`；自定义BackUp成员实际保持0 | s，标准Spin／DriveOnHeading预测时长 | **上实物前必须核对并正确配置** | 标准行为按当前速度向前投影候选位姿检查碰撞 | 标准行为可检查更远，样本和计算增加，受提前终止条件影响 | 标准检查范围缩短；过小可令采样数为0 | 各插件有独立成员；Spin和标准Drive读2，BackUp覆盖初始化未读取；不是仿真时间倍率或所有行为的统一预测时长 |
| `max_rotational_vel` | `1.0` | rad/s，Spin角速度幅值上限 | **当前运行不使用这个参数，暂不调整** | 限制Spin计算的角速度幅值 | 允许更快转动，离散反馈／碰撞检查需匹配 | Spin最大命令更小，任务可能更慢 | Spin未入默认树；不限制BackUp平移、PID输出或下游cmd_spin相加后的角速度 |
| `min_rotational_vel` | `0.4` | rad/s，Spin尚未结束时的幅值下限 | **当前运行不使用这个参数，暂不调整** | 将接近目标时过小的计算角速度抬到下限 | 末段命令更大，可能增加越过目标的离散步长 | 可更低速接近，实际低速能力需验证 | 是幅值不是方向；完成时仍发0；宜0≤min≤max，不表示最终角误差容差 |
| `rotational_acc_lim` | `3.2` | rad/s²，Spin制动速度公式中的系数 | **当前运行不使用这个参数，暂不调整** | 用sqrt(2×系数×剩余转角)计算速度，再按上下限夹紧 | 相同剩余角的计算速度更高，较晚离开最大速度区 | 更早进入较低计算速度区，仍受0.4下限限制 | 不是逐周期限制Δ角速度的完整加速器；当前Spin可在首轮直接发到上限，不保证真实角加速度≤3.2 |

##### 参数详解

###### 自定义BackUp的预测时长为何没有得到2

BackUpFreeSpace继承DriveOnHeading<BackUp>。父类构造函数把simulate_ahead_time_设为0；本应由父类onConfigure读取YAML中的2，但自定义覆盖函数只读取global_frame、max_radius、service_name、visualize，没有调用父类版本，也没有自行设置该成员。[父类初始化](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/include/nav2_behaviors/plugins/drive_on_heading.hpp#L39)、[自定义初始化](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca/src/behaviors/back_up_free_space.cpp#L21)

所以自定义onCycleUpdate虽然调用继承的isCollisionFree，后者max_cycle_count=int(10×0)=0，while不进入，直接返回True。Spin对同一个节点参数的读取只更新Spin自己的成员，不会替BackUp初始化。

本次离线辅助检查在匹配安装头文件的自定义对象上观察到构造后时长0；将循环频率设10、目标距离设1、检查器保持未配置的空指针后，继承检查仍返回True而未访问检查器。再结合源码覆盖函数可确认此初始化缺口。**没有调用真实configure／onRun或运动Action**，此检查不能写成已经完成整套运行验证。

仅补调用父类还不构成完整修复：继承预测只用cmd_vel.linear.x沿pose.theta推算x、y，忽略自定义真正发送的linear.y。需要同时让预测覆盖实际二维速度，并统一候选位姿与检查地图参考系。[继承预测公式](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/include/nav2_behaviors/plugins/drive_on_heading.hpp#L176)

###### Spin的三项速度参数（幅值、末段下限与制动包络）

Spin根据global_frame中robot_base_frame的yaw变化累计relative_yaw。剩余角为abs(请求转角)−abs(已转角)，小于1e-6时发零并结束；否则计算：

```text
角速度幅值 = clamp(sqrt(2 × rotational_acc_lim × 剩余转角),
                    min_rotational_vel, max_rotational_vel)
角速度正负 = 请求target_yaw的正负
```

当前剩余角较大时上限1 rad/s；理想剩余角降到1²／(2×3.2)=0.15625 rad附近才离开最大速度区，再降到0.4²／(2×3.2)=0.025 rad附近后受最小速度限制。数值为公式计算，不是实车转动记录。

rotational_acc_lim在这里构成剩余距离制动速度包络，没有按上一轮速度和实际dt做起步加速度限幅。10 Hz下0.4 rad/s对应每理想周期约0.04 rad指令转动尺度，但是否正确停到目标还取决于真实反馈和执行链。[Spin速度与反馈](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/plugins/spin.cpp#L111)

当前gimbal_yaw_fake在前文的平面条件下抵消输入yaw，其方向与odom近似对齐，不是底盘真实yaw。把它用于Spin累计转角可能无法反映真实底盘转动；map↔odom定位修正也可能进入该反馈。启用Spin前应先确定要转动哪个实体及相应位姿参考，不能仅调角速度解决；本次未调用Spin验证这个坐标设计推论。[虚拟系方向](/wiki/2026-09-20-dao-hang/0500-0600-fake-vel-transform)

1.1.20的Spin预测提前结束条件使用relative_yaw已转角，而非剩余目标角；首次relative_yaw=0时会在第一次检查前退出。即便Spin成员读取了2，也不能据此保证每轮完整预测未来2 s。这与自定义BackUp时长成员为0是不同问题。[Spin预测分支](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/plugins/spin.cpp#L151)

#### 自定义BackUp参数（地图服务、选向半径与显示）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `service_name` | `global_costmap/get_costmap` | 不适用，服务名称 | **上实物前必须核对并正确配置** | 在BackUp启动时请求一份完整原始代价图供选向 | 不适用 | 不适用 | 真实被读取；与global_costmap_topic未生效不同；响应网格的frame必须与global_frame位姿一致；不经2 Hz地图话题等下一次发布 |
| `max_radius` | `2.0`；自定义默认1.0 | m，从机器人参考点出发的选向射线长度 | **实物运行时重点观察，有问题再针对性调整** | 要求候选方向从r=0到此范围的采样均不遇≥253或地图外 | 检查更远，更多方向可能被远障碍／边界淘汰，采样增多 | 检查更近，可能放行在目标距离内有障碍的方向 | 不是BackUp目标距离或雷达量程；当前请求距离1 m，半径2 m；增大不能修复无安全方向返回0或预测绕过 |
| `visualize` | `false` | 不适用，布尔 | **根据实际功能需求开启或设置** | 建立back_up_free_space_markers并显示选中的扇区／方向 | 不适用 | 不适用 | True便于核对一次选向结果，不直接改速度／碰撞逻辑；按global_frame显示，箭头不证明实际Twist已正确换系或方向安全 |

##### 参数详解

###### service_name（一次服务取图，等待不受10 s运动期限完整覆盖）

自定义onRun先每次等待服务可用1 s，直到服务存在或ROS结束；该循环没有Action取消或任务期限检查。发请求后最多等待响应1 s，超时返回FAILED。随后取位姿、选方向、读取请求值，才设置end_time=ROS now+time_allowance并进入运动循环。[服务等待与取图](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca/src/behaviors/back_up_free_space.cpp#L54)

因此BT缺省10 s是运动阶段的请求期限，不覆盖前面可能反复等待服务的全部时长。common TimedBehavior的elapsed结果计时也在onRun成功后开始；等待阶段不能靠cycle_frequency=10或BT请求响应超时100 ms推断为10 s内自动结束。

该服务在一次onRun中取一次图用于选方向，不在每周期重新选向。GetCostmap返回当前服务内存图，不需等待global_costmap下一次2 Hz发布；但取到后障碍变化仍需真正生效的周期检查处理，当前这一步存在前述缺口。

###### max_radius（检查一整条射线，不是挑最远空闲点）

源码扫描−π到+π，固定角步长π／32，即5.625°，约64个方向；每条射线以服务返回的地图resolution为径向步长，从r=0检查到max_radius。点必须在网格内且代价<253；253、254、255都让该方向不安全，1～252软代价则不会比较高低。[射线与代价检查](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca/src/behaviors/back_up_free_space.cpp#L160)

若有效分辨率恰为0.05 m，2 m对应约40个格距。实际全局图格距由静态地图决定，不能总按本costmap YAML初始0.05推算；方向角度固定，也不随更细格距自动加密。它检查的是采样中心线，不逐点放置真实车体轮廓。

例如目标移动1 m，却在某方向1.5 m处有障碍：max_radius=2会淘汰该方向，1可能不淘汰。这说明半径是选向检查条件，不是目标移动长度。反过来，半径小于目标距离也不能保证目标范围内都已检查；更何况当前无安全方向处理没有可靠失败出口。

gatherFreePoints这个辅助函数虽然会收集radius内代价0的格子，但当前onRun没有调用它；实际选向不是“寻找距离最近的空闲点”。

###### 找较宽扇区的实现边界（没有安全方向也返回0）

函数尝试记录第一个安全角和之后的第一个不安全角，把更宽区间保存，再取两边界平均值。当前实现不能直接称为完整的“最大安全扇区算法”：

- 若扫描结束前一直安全，没有最后不安全角，尾部区间不结算；−π／+π两端的相邻区间也不做环绕合并。
- 一个已关闭区间若不比当前最优更宽，first_safe_angle和last_unsafe_angle没有复位，可能阻止后续区间继续被正确记录。
- final_safe_angle与final_unsafe_angle初始均为0；没有记录到有效区间时，最终返回(0+0)／2=0，不区分全空闲、全障碍或其他未闭合情况，也不返回“不可移动”。

本次直接调用**安装的自定义库**选向函数，用101×101、0.05 m格距、原点(-2.525,-2.525)、机器人在(0,0)、半径2 m的合成网格得到：

| 离线输入 | 返回角度 | 能证明什么 |
|---|---:|---|
| 所有格均为254障碍 | 0 rad | 没有安全射线仍返回默认角，不是返回失败 |
| 所有格均为0空闲 | 0 rad | 全安全未闭合时也得到同一默认角，不能从0判断是有效安全扇区 |

这两个辅助函数检查没有建立ROS节点或地图服务，更没有发运动指令。实际onRun得到该角后不会再次检查“是否找到安全方向”，仍可设置速度并返回初始化成功；要实现可靠失败处理，需要修改选向／调用逻辑，而不是继续调半径。

###### 选向角在map，速度输出却没有转换到机体系

射线用pose.x／y和地图x／y计算，angle相对global_frame轴；pose.theta在findBestDirection中没有参与角度换系。之后直接写：

```text
vx = cos(angle) × 请求speed
vy = sin(angle) × 请求speed
```

Twist没有frame_id，后续导航转换链按gimbal_yaw_fake速度约定解释它。若机器人在map中的yaw为theta，地图方向angle对应的机体系方向应核对angle−theta，而当前代码没有减theta。map←odom有yaw修正时，显示在map的射线与实际输出方向可能不同；不能因虚拟系平时与odom对齐就认定它也与map对齐。

标准BackUp会把距离／速度取负绝对值，实现固定机体系负x倒退；本自定义类覆盖了onRun，不执行该逻辑。当前BT直接把backup_dist=1、backup_speed=1写入请求，故自定义速度模长按请求为1 m/s，方向按其选择角；请求target.x正负不改变完成用的abs距离，负speed则会把输出方向反过来。[BT请求字段](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/plugins/action/back_up_action.cpp#L36)

###### 完成距离、恒定速度与停止路径

进入运动后，自定义循环保持启动时的twist_x／y，不用标准DriveOnHeading的加减速或最小速度成员，也没有读取当前PID的线速度限值。完成判据为map中当前位置与第二次取得的initial_pose之间hypot差达到abs(target.x)。这是**起终位置位移**，不是沿运动轨迹累计里程，也不是沿选中射线投影距离。

定位修正可能改变map位移统计，横向偏移同样能增加该距离；只看反馈达到1 m不能证明沿选定方向实际走了1 m。理想恒定1 m/s且方向正确时1 m约需1 s，只是请求换算，未验证实车。

完成、运动期限超时、明确碰撞失败会调用stopRobot；但当前预测绕过意味着不能依赖其“Collision Ahead”分支。取得当前TF失败时直接返回FAILED，common TimedBehavior的FAILED分支也不统一发零，不能笼统写“任何失败都会刹停”。取消／抢占则在进入执行循环后显式发零。[自定义周期处理](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca/src/behaviors/back_up_free_space.cpp#L109)、[共同执行状态](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/include/nav2_behaviors/timed_behavior.hpp#L229)

#### 相关源码默认项（原YAML未显式配置）

##### 参数表

以下8项是真正读取的地图键、标准DriveOnHeading和AssistedTeleop默认项；没有向原YAML追加。Drive参数不自动进入当前自定义BackUp的覆盖实现。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `costmap_topic` | `local_costmap/costmap_raw`，1.1.20默认 | 不适用，话题名称 | **上实物前必须核对并正确配置** | 单个共有检查器的真实地图输入 | 不适用 | 不适用 | 当前读取此键，需与global_frame坐标一致；消息为nav2_msgs/Costmap，不是普通OccupancyGrid |
| `footprint_topic` | `local_costmap/published_footprint`，1.1.20默认 | 不适用，话题名称 | **上实物前必须核对并正确配置** | 共有检查器的真实轮廓输入 | 不适用 | 不适用 | FootprintSubscriber转回robot_base_frame后放到候选pose；不能靠未读取的双地图键切换 |
| `drive_on_heading.acceleration_limit` | `0.0`，父类默认 | m/s²，加速限制 | **当前运行不使用这个参数，暂不调整** | 标准DriveOnHeading执行时按cycle_frequency限制速度增加 | 正值可允许更快升速；0是禁用条件 | 正值减小时升速更缓 | 当前该Action未入树；任一加／减速限制为0时标准实现直接用请求速度；自定义BackUp不使用此逻辑 |
| `drive_on_heading.deceleration_limit` | `0.0`，父类默认 | m/s²，减速限制，启用时取负 | **当前运行不使用这个参数，暂不调整** | 标准DriveOnHeading限减速并算剩余距离制动速度 | 负值向0靠近时减速幅度更小；0禁用 | 负值更负时允许更大减速幅度 | 不是简单“数值越大越强”；需与正加速限制配套；不受rotational_acc_lim控制 |
| `drive_on_heading.minimum_speed` | `0.0`，父类默认 | m/s，线速度幅值下限 | **当前运行不使用这个参数，暂不调整** | 标准DriveOnHeading尚未结束时抬高过小线速度 | 保持更高末段速度，可能增加停止误差 | 允许更低速接近 | 不等于PID的min_approach_linear_velocity，也不作用于当前自定义BackUp恒定二维命令 |
| `projection_time` | `1.0`，AssistedTeleop默认 | s，遥控预测时间范围 | **当前运行不使用这个参数，暂不调整** | 辅助遥控按输入Twist逐步投影检查 | 可检查更远，计算增加，可能更早缩速 | 检查更近，提前发现空间更少 | 与simulate_ahead_time=2不同；当前默认BT未调用，需要实际遥控Action激活 |
| `simulation_time_step` | `0.1`，AssistedTeleop默认 | s，遥控投影步长 | **当前运行不使用这个参数，暂不调整** | 决定遥控预测位姿的离散采样间隔 | 采样更粗、次数减少，可能跨过障碍 | 采样更密、计算增加 | 不是Gazebo仿真步长；宜有限正值并小于projection_time，过大可使检查循环不进入 |
| `cmd_vel_teleop` | `cmd_vel_teleop`，AssistedTeleop默认 | 不适用，输入话题名称 | **根据实际功能需求开启或设置** | 订阅辅助遥控使用的Twist | 不适用 | 不适用 | 订阅存在不等于Action已执行；该实现没有输入消息过期停速检查，需核对实际操控与下游停更处理 |

##### AssistedTeleop与Wait的使用边界

AssistedTeleop把最新遥控Twist保存在成员中，执行Action时才做投影和输出；收到preempt_teleop的Empty消息会在周期处理时发零并成功结束，time_allowance正值到期则发零失败。正常结束后清成员，但没有按最后一条遥控消息时间自动判过期，不能把输入断流等同于立即刹停。

遇预测碰撞时，第一步碰撞把全部速度分量置0；之后某步碰撞则按time／projection_time等比缩放vx、vy、wz。当前projectPose的侧向项符号还需核对：

```text
源码：Δx = dt ×（vx cos(theta) + vy sin(theta)）
      Δy = dt ×（vx sin(theta) − vy cos(theta)）
```

按ROS常见机体系正y向左约定，theta=0、vy>0时真实指令方向应为正y，而这里预测负y；例如vy=1 m/s、dt=0.1 s计算得到−0.1 m。若后续启用辅助横移，应核对／修正这处投影，不能只加大projection_time就认定已正确检查。此例是源码计算，未发遥控指令。[辅助遥控实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/plugins/assisted_teleop.cpp#L148)

Wait只计算请求结束时间和发布time_left，正常执行不调用stopRobot。是否已经停住必须由调用流程和速度链保证，而不是在树中放一个Wait就证明车辆停止。[Wait实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behaviors/plugins/wait.cpp#L35)

#### 输出接线、动态参数与验证范围

##### 两种启动分支的恢复速度输出不同

每个TimedBehavior发布相对cmd_vel。当前navigation_launch存在下列差异，源码与安装副本一致：

| 启动分支 | behavior_server实际重映射 | 当前速度路径 |
|---|---|---|
| use_composition=True | cmd_vel→cmd_vel_nav2_result | 行为输出直接进入fake_vel_transform，跳过velocity_smoother |
| use_composition=False | behavior_server节点没有上述重映射；相同重映射放在bt_navigator节点上 | 行为仍发布namespace/cmd_vel，不能据组件分支推断它也经过fake_vel_transform |

整套实物／仿真入口的use_composition缺省True；单独navigation_launch缺省False，用户显式切换时也会进入不同分支。bt_navigator的重映射不会替另一个节点修改cmd_vel发布器。独立分支需要修正behavior_server接线，不能只修改本YAML的cycle_frequency。[独立分支](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/launch/navigation_launch.py:203)、[组件分支](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/launch/navigation_launch.py:306)

当前组件恢复不经过velocity_smoother，因此其速度／加速度设置不自动平滑本行为命令。进入fake_vel_transform后，cmd_spin附加角速度还会相加；Spin的1 rad/s上限不是最终总角速度上限，stopRobot的零Twist也不自动清除附加量。具体坐标与MCU协议见[fake_vel_transform](/wiki/2026-09-20-dao-hang/0500-0600-fake-vel-transform)。

各行为还有各自的Action执行上下文，列表本身不提供多个运动Action之间的统一速度仲裁；当前BT按恢复顺序调用，不应通过并发发起多个行为来测试参数。

##### 动态参数与启动时读取

1.1.20的BehaviorServer、TimedBehavior、当前标准行为以及BackUpFreeSpace没有为本组参数建立完整动态更新路径。cycle_frequency、global_frame、robot_base_frame、transform_tolerance及插件专用成员在configure阶段读取；话题和服务客户端也在初始化建立。运行中ROS参数值改变不代表成员、循环频率或订阅已经联动更新，应按配置重新配置／重启再核对。

重新启动也不会自动修复未读取的五个键、缺少的父类初始化或无安全方向返回逻辑。自定义BackUp的目标距离、速度和运动期限则来自每次Action请求；它们不能用本YAML的max_radius、max_rotational_vel替代。

##### 本次核对与实物待处理事项

- **本次已核对：** nav2_behaviors 1.1.20、pb插件1.0.1，五个插件注册与库存在；服务器／TimedBehavior／DriveOnHeading安装头文件匹配；自定义源码／头文件匹配固定上游提交且安装头文件一致；两套参数和安装副本一致，仅时钟不同；默认XML与相关launch安装副本一致。
- **本次离线辅助验证：** 不初始化ROS、不创建节点、不调用configure／onRun或Action，仅构造辅助对象并调用继承预测／安装库选向函数；观察到预测时长0、未配置检查器时预测直接True，以及全254／全0合成网格均返回0 rad。临时可执行文件随临时目录清理，没有发布任何速度。
- **源码能确认：** 23项原参数的实际使用条件；单地图接口与五个无读取键；共同帧与地图组合问题；自定义预测初始化遗漏、x-only继承预测、扇区结算／无安全方向退回值、角度未换到机体系；Spin制动公式与反馈／预测条件；等待和失败停速边界；辅助横移投影公式及两种输出分支。
- **实物待处理：** 先适配当前版本真正读取的地图／轮廓键并统一坐标，修正BackUp预测、选向失败处理及速度换系，核对独立分支重映射和各失败停速路径，再验证真实车体、恢复方向、有效速度与运动期限；仅改变本表数值不能替代这些修正。
- **本次验证范围：** 未启动新的恢复行为，未修改运行参数、工程YAML、默认树或插件代码，未进行实车／仿真运动和碰撞测试；本文不把辅助函数结果当整套恢复安全验证。
