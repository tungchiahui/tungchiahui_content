---
title: "controller_server（路径跟踪、进度检查与到达判定）"
---

## controller_server（路径跟踪、进度检查与到达判定）

### 实车先看：速度怎样算出，什么情况下停止或报失败

**controller_server接收FollowPath任务，用当前位姿、里程计速度和局部代价地图调用跟踪插件，发布速度，并检查是否有移动进展及是否到达路径终点。** 当前FollowPath插件是pb_omni_pid_pursuit_controller::OmniPidPursuitController：先选前视点，用PID求一个平移速度标量，再沿前视点方向分成vx、vy。

Humble笔记里的FollowPath示例介绍了DWB的critics、轨迹采样和加速度限制。本工程采用全向PID跟踪器，下面按这个插件的实际实现解释参数，不把那套评分器参数套进来。

当前主要链路为：

```text
行为树FollowPath → controller_server
                     ├─ TF／local_costmap → 当前虚拟参考点位姿
                     ├─ odometry → 速度阈值过滤 → 速度前视距离
                     ├─ 路径转到gimbal_yaw_fake → 选前视点
                     ├─ 平移PID → 曲率减速 → 终点减速 → vx、vy
                     ├─ progress_checker → 是否有足够位置移动
                     └─ general_goal_checker → 是否到达路径终点

cmd_vel_controller → velocity_smoother → cmd_vel_nav2_result
                   → fake_vel_transform → 实际云台系cmd_vel → 串口／MCU
```

本节点的机器人参考点由local_costmap.robot_base_frame决定，当前为gimbal_yaw_fake；位姿在local_costmap.global_frame中取得，当前为odom。本块没有另一个robot_base_frame参数。进度、到达距离以及前视点都要按这套参考点理解，不能只看bt_navigator的同名设置。

| 实车上看到的现象 | 先核对什么 | 当前配置应怎样理解 |
|---|---|---|
| 以为0.5 m/s以下不能横移 | min_y_velocity_threshold与输出速度 | 此项过滤输入里程计的y速度，不直接禁止输出vy；它会影响速度前视计算 |
| 改lookahead_dist=2.0却看不到变化 | use_velocity_scaled_lookahead_dist | 当前启用速度前视，距离为过滤后平面速度×1 s，再夹到0.5～1.0 m；固定2 m当前不用 |
| 改progress_checker_plugins却没有切换检查器 | Humble的实际参数名 | 1.1.20读取单数progress_checker_plugin；当前恰好由默认ID选到progress_checker |
| 已到位置，朝向没对齐也成功 | yaw_goal_tolerance与enable_rotation | 6.28是rad，大于π，正常最短yaw误差均能通过；当前旋转PID关闭 |
| 终点前抖动、冲过或突然反向 | PID输出、前视点跳变、终点减速与上游速度 | 平移误差是前视点距离；导数可正可负，速度下限允许负标量；不能只提高D就当作增加阻尼 |
| 弯道没有按注释降到一半 | 曲率倍率、减速速度下限与变化速率 | 0.5是目标保留倍率；当前曲率阶段另有1.0 m/s减速下限，并不能把已经更慢的速度抬高 |
| 界面显示PID新值，行为仍没变 | 动态回调与内部PID对象 | 增益成员更新后没有重建PID；速度上下限的动态分支为空；需重新配置／重启验证 |

本次容器内nav2_controller、nav2_core、nav2_util均为**1.1.20**，PID插件安装包为**1.0.4**。本地控制器实现、PID实现及控制器头文件与北极熊上游提交**b542c5794b91fba2bb3fe2cbdc72fcf854fedb1a**逐字一致；以下按对应代码核对。[当前控制器源码](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c5794b91fba2bb3fe2cbdc72fcf854fedb1a/src/omni_pid_pursuit_controller.cpp)、[PID源码](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c5794b91fba2bb3fe2cbdc72fcf854fedb1a/src/pid.cpp)

### 参数

下面保留本次发送的完整实物参数，包含原注释。仿真段只有use_sim_time=True不同。源码缺口和建议另列，不直接替换这里的原值。

```yaml
controller_server:
  ros__parameters:
    use_sim_time: False
    odom_topic: odometry
    controller_frequency: 20.0
    min_x_velocity_threshold: 0.001
    min_y_velocity_threshold: 0.5
    min_theta_velocity_threshold: 0.001
    failure_tolerance: 0.3
    progress_checker_plugins: ["progress_checker"]
    goal_checker_plugins: ["general_goal_checker"]
    controller_plugins: ["FollowPath"]
    progress_checker:
      plugin: "nav2_controller::SimpleProgressChecker"
      required_movement_radius: 0.5
      movement_time_allowance: 10.0
    general_goal_checker:
      stateful: True
      plugin: "nav2_controller::SimpleGoalChecker"
      xy_goal_tolerance: 0.15
      yaw_goal_tolerance: 6.28
    FollowPath:
      plugin: "pb_omni_pid_pursuit_controller::OmniPidPursuitController"
      translation_kp: 3.0
      translation_ki: 0.1
      translation_kd: 0.3
      enable_rotation: false
      rotation_kp: 3.0
      rotation_ki: 0.1
      rotation_kd: 0.3
      transform_tolerance: 0.1
      min_max_sum_error: 1.0
      lookahead_dist: 2.0
      use_velocity_scaled_lookahead_dist: true
      lookahead_time: 1.0
      min_lookahead_dist: 0.5
      max_lookahead_dist: 1.0
      use_interpolation: false
      use_rotate_to_heading: false
      use_rotate_to_heading_treshold: 0.1
      min_approach_linear_velocity: 0.5
      approach_velocity_scaling_dist: 1.0
      v_linear_min: -2.5
      v_linear_max: 2.5
      v_angular_min: -3.0
      v_angular_max: 3.0
      curvature_min: 0.4                                # 低曲率阈值，低于此值不减速。
      curvature_max: 0.7                                # 高曲率阈值，高于此值大幅减速。
      reduction_ratio_at_high_curvature: 0.5            # 高曲率时的速度降低比例。 0.5 (即减速 50%)
      curvature_forward_dist: 0.7                       # 前向距离，用于曲率计算
      curvature_backward_dist: 0.3                      # 后向距离，用于曲率计算
      max_velocity_scaling_factor_rate: 0.9             # 速度缩放因子的最大变化率
```

#### ros__parameters（服务器节拍、输入与插件选择）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真True | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择ROS时钟，影响TF、进度时间和失败计时 | 不适用 | 不适用 | 整套launch重写已有键；服务器控制循环用WallRate，PID另用配置时算出的固定dt |
| `odom_topic` | `odometry` | 不适用，话题名称 | **上实物前必须核对并正确配置** | 取得输入twist的x、y及绕z角速度 | 不适用 | 不适用 | 当前订阅器保留最新分量，不负责重新换系或验证速度含义；当前位置从local_costmap的TF取得 |
| `controller_frequency` | `20.0` | Hz | **通常保持当前值，不作为优先调试项** | 设置FollowPath期间控制循环的目标频率，并在插件配置时用于dt | 理想检查更密；计算负担增加，PID离散步长也改变 | 检查更稀；响应可能更迟，PID离散步长也改变 | 当前目标50 ms／轮，不保证实际达到；插件固定dt=0.05 s，不测真实循环间隔；改频率须重新配置插件 |
| `min_x_velocity_threshold` | `0.001` | m/s，输入x分量门槛 | **通常保持当前值，不作为优先调试项** | 将输入vx绝对值不超过门槛的部分置零 | 过滤更多低速与噪声，可能删掉真实慢速运动 | 保留更多小速度，也保留更多噪声 | 不限制输出vx；过滤后的vx、vy参与前视距离；边界等于门槛也置零 |
| `min_y_velocity_threshold` | `0.5` | m/s，输入y分量门槛 | **实物运行时重点观察，有问题再针对性调整** | 将输入vy绝对值不超过0.5 m/s的部分置零 | 更多横向运动被视为零，混合运动的速度前视可能缩短 | 保留更多真实低速横移，也保留更多y噪声 | 不是最小可执行横移速度；对全向实车偏大的过滤门槛需按有效速度与静止噪声核对，不因原示例这样写就认定适合 |
| `min_theta_velocity_threshold` | `0.001` | rad/s，输入绕z分量门槛 | **通常保持当前值，不作为优先调试项** | 将输入角速度绝对值不超过门槛的部分置零 | 过滤更多小角速度 | 保留更多小角速度 | 不限制输出wz；当前PID前视只用平移速度，SimpleGoalChecker不检查速度，本项不会把当前输出旋转变小 |
| `failure_tolerance` | `0.3` | s，插件连续异常容忍时间 | **实物运行时重点观察，有问题再针对性调整** | 容忍computeVelocityCommands抛出的PlannerException一小段时间 | 较长时间发布零速度并等待恢复，跟踪失败上报可能更迟 | 更快上报持续计算失败，短暂异常更易结束当前FollowPath | 当前0.3 s，不是0.3 m；仅特定异常路径；0立即传播，-1为无限容忍，不能用来掩盖问题 |
| `progress_checker_plugins` | `["progress_checker"]` | 不适用，列表 | **已确认当前值错误，必须修改** | 该复数键未被本版本用于选择进度检查器 | 不适用 | 不适用 | 1.1.20实际读取progress_checker_plugin字符串；当前靠默认ID恰好选到同名配置，未因此必然启动失败；建议改为单数键 |
| `goal_checker_plugins` | `["general_goal_checker"]` | 不适用，ID列表 | **上实物前必须核对并正确配置** | 指定加载的目标检查器配置ID | 不适用 | 不适用 | 当前仅一个，BT的FollowPath未指定goal_checker_id时会选择它；ID与下方分组名称须一致 |
| `controller_plugins` | `["FollowPath"]` | 不适用，ID列表 | **上实物前必须核对并正确配置** | 指定加载的路径跟踪插件配置ID | 不适用 | 不适用 | 当前BT明确请求controller_id=FollowPath；算法类由FollowPath.plugin决定，ID本身不说明算法 |
| `progress_checker_plugin` | `"progress_checker"`，源码默认；原YAML未配置 | 不适用，ID字符串 | **上实物前必须核对并正确配置** | 当前版本真正用于选择单个进度检查器的键 | 不适用 | 不适用 | 此行说明实际默认，不与复数列表叠加；按当前Humble修正键名后沿用这个ID |

##### 参数详解

###### controller_frequency 与 odom_topic（目标20 Hz、固定PID步长与实际速度）

服务器在FollowPath任务执行期间反复算速度，目标20 Hz。它仍可能等待代价地图变为current，或受TF、路径变换、计算耗时影响；把20设大不能自动加快点云、地图或TF。[服务器控制循环（1.1.20）](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_controller/src/controller_server.cpp)

PID插件配置时取controller_frequency，算dt=1/20=0.05 s，此后积分、微分和曲率速度变化步长都用这个固定值。实际有一轮拖到100 ms，也不会自动把PID的dt改成0.1 s。因此调频率既改调度目标，也改算法离散计算；不能只当作刷新显示频率。

当前odometry的位姿、速度来源与坐标缺口见[sensor_scan_generation](/wiki/2026-09-20-dao-hang/0500-0300-sensor-scan-generation)。这里的OdomSubscriber直接取最新twist分量，没有bt_navigator那种0.3 s平均窗口，也没有替上游修正速度表达系或时间差分。[里程计订阅实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_dwb_controller/nav_2d_utils/include/nav_2d_utils/odom_subscriber.hpp)

当前插件用输入平面速度决定前视距离，PID本身的误差来自前视点位置，并非“目标速度减里程计速度”的底盘速度闭环。底盘电机执行与速度反馈仍属下位机链路。

###### 三项min…velocity_threshold（过滤测得的速度，不夹住输出速度）

本版本规则为：绝对值**严格大于**门槛才保留，否则置零；正负速度都按同一绝对值规则处理。[过滤函数](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_controller/include/nav2_controller/controller_server.hpp)

例如原始输入vx=0.4、vy=0.4 m/s，平面速度约0.566 m/s。当前x保留、y清零，交给PID插件的平面速度变为0.4 m/s；乘1 s后再经0.5 m下界夹取，前视距离变为0.5 m。若保留原输入，这个例子会取约0.566 m。这里只计算配置影响，不是实测结果。

纯横移0.3 m/s时，vy虽会被清零，但本来与过滤后都可能落在0.5 m前视下界，选出的前视距离未必改变。不能把“过滤了速度”简单推导为“所有低速横移的控制输出都消失”。当前输出vy仍由前视点方向与PID标量计算。

调y门槛应先看静止噪声和实际低速横移数据，尤其上游当前的twist未严格按child_frame_id换系；按哪个轴过滤本身也可能偏离预期。阈值不能修复坐标或时间问题。theta门槛不会代替enable_rotation、旋转限幅或云台机械关节限位。

###### failure_tolerance（0.3 s等待有效控制恢复）

成功计算一次速度后，服务器更新last_valid_cmd_time。若下一次插件计算抛出PlannerException，当前容忍分支发布零速度，继续尝试；从最后一次成功计算算起超过0.3 s仍未成功，抛出Controller patience exceeded，让当前FollowPath失败。理想20 Hz下约对应6个周期，实际不是固定“允许失败6次”。

插件中的路径转换失败、空局部路径或碰撞检测可进入这条异常路径；但是**取机器人位姿失败、进度检查失败发生在这段catch之外，会直接上报**。代价地图等待也不由这0.3 s限定。因此不能把它当成覆盖全部导航问题的统一等待时间。[异常范围与计时实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_controller/src/controller_server.cpp#L465)

增加它只是让特定异常分支多等待，不会越过障碍继续发上一条非零速度；减少则更快触发前面行为树中的清图与恢复。进度检查的10 s、BT请求响应的100 ms各有不同用途。

###### 三项插件选择（ID、类名与Humble单数键）

ID是下方参数分组的名称，plugin是加载的C++类名。BT请求FollowPath时，选的是ID；此ID内部可以加载不同算法，当前明确加载全向PID。

Nav2 1.1.20在构造及configure阶段都读取**progress_checker_plugin**，只选择一个进度检查器。当前复数progress_checker_plugins未用于此选择，默认单数ID刚好是progress_checker，所以仍读取下方同名的SimpleProgressChecker配置。改变复数列表的内容并不能实际切换检查器。

沿用本版本时，建议将该键修正为：

```yaml
progress_checker_plugin: "progress_checker"
```

这是配置修正示例，原参数代码块仍保留发送内容；本次没有修改工程的nav2_params.yaml。goal_checker_plugins与controller_plugins在当前版本则确实为复数列表，不能一起改成单数。[插件键声明与加载](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_controller/src/controller_server.cpp#L51)

#### progress_checker（位置移动进展检查）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `nav2_controller::SimpleProgressChecker` | 不适用，类名 | **通常保持当前值，不作为优先调试项** | 按机器人参考点的平面位移检查是否有进展 | 不适用 | 不适用 | 当前看x、y位移，不比较离目标是否更近，不计纯yaw转动 |
| `required_movement_radius` | `0.5` | m，重置基准所需净位移 | **实物运行时重点观察，有问题再针对性调整** | 当前参考点离上次基准位置超过0.5 m后重置计时 | 需要更大位移才算进展，慢速／小范围移动更易超时 | 更小位移也能重置，定位噪声或小抖动也更易被算进展 | 严格大于0.5 m；不是每轮要走0.5 m、目标误差或累计路程 |
| `movement_time_allowance` | `10.0` | s，基准未更新的允许时间 | **实物运行时重点观察，有问题再针对性调整** | 允许当前位置基准在10 s内暂时不更新 | 可等更久，卡住发现也更迟 | 更快判无进展，低速接近和正常等待更易失败 | 首次检查建立基准；超时严格大于10 s；进展足够时位置与时间同时重置 |

##### 参数详解

###### 0.5 m与10 s（检查净位移，不要求一直接近目标）

首次检查记住参考点位置和时间。以后，只要当前位置与该基准的平面距离超过0.5 m，就重设基准；尚未超过时，允许等待10 s。检查取自local_costmap的机器人参考点位姿，当前为odom中的gimbal_yaw_fake原点。

因此，在原地附近反复绕行，累计路程已经超过0.5 m，却可能始终未离基准超过0.5 m；反之，朝错误方向移动0.6 m也会算“有进展”。纯yaw转动不直接算位移；若参考点与旋转中心有偏置，位置变化须按实际TF理解。[SimpleProgressChecker实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_controller/plugins/simple_progress_checker.cpp)

0.5/10=0.05 m/s只能帮助估算持续直行的要求，不是此处配置了最低速度。低速接近终点、短路径、小范围避让或等待可能受这组参数影响；应结合真实任务和定位噪声观察，不能只放宽到“永不失败”。换一条正在跟踪的路径不会在当前服务器的updateGlobalPath中自动重置进度检查，开始一轮新的FollowPath执行才有对应reset。

#### general_goal_checker（路径终点判定）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `nav2_controller::SimpleGoalChecker` | 不适用，类名 | **通常保持当前值，不作为优先调试项** | 检查路径终点的平面位置与yaw误差 | 不适用 | 不适用 | 此类忽略传入的速度；成功不等于已验证物理停稳，不检查z／roll／pitch |
| `stateful` | `True` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 本轮位置已合格后，后续只继续检查yaw | 不适用 | 不适用 | 收到新路径时reset重新检查位置；当前yaw门槛极宽，通常位置通过同一次就成功 |
| `xy_goal_tolerance` | `0.15` | m，终点平面距离容差 | **实物运行时重点观察，有问题再针对性调整** | 参考点进入终点15 cm半径内时，位置合格 | 更早满足、更容易成功，但停车参考点偏差可能更大 | 要求更近，可能在噪声、惯性和离散控制下更难完成 | 是x、y合成的圆形距离，不是两轴各允许15 cm；按gimbal_yaw_fake参考点，不保证车体外形已到指定位置 |
| `yaw_goal_tolerance` | `6.28` | rad，最短yaw误差容差 | **根据实际功能需求开启或设置** | 决定终点朝向误差是否合格 | 正常门槛内放宽朝向；大于π后对正常yaw误差已无进一步区别 | 缩到π以内才开始约束朝向；过小而无旋转控制时可能无法完成 | 6.28约359.82°，最短角差绝对值不超过π，当前等效不要求朝向；不是6.28°或允许转一整圈 |

##### 参数详解

###### xy_goal_tolerance、yaw_goal_tolerance 与 stateful（当前主要按15 cm位置判成功）

位置检查使用dx²+dy²≤0.15²。进入该圆后，stateful=True会记住“位置已通过”，随后不再重复位置条件，直到检查器reset。服务器接收新路径时会reset，因此不能把这个状态理解成永久记住曾经到过终点。

yaw采用angles::shortest_angular_distance，误差落在约[-π,π]中，再判断其绝对值是否严格小于容差。6.28大于π，所以正常有效姿态都能通过；与enable_rotation=False配合，当前正常导航无需为了停车对齐指定朝向。[SimpleGoalChecker实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_controller/plugins/simple_goal_checker.cpp)

若以后把yaw容差收紧，须一起核对旋转输出、真实反馈参考系和任务要求，不能只把它改小就期待机器人自动转正。gimbal_yaw_fake的水平轴在平面条件下与odom对齐，底盘自转不等于该虚拟参考系随底盘yaw转动。

SimpleGoalChecker不检查实测速度；当前服务器成功后按默认publish_zero_velocity=True发零速度，然后仍要经过平滑、换系、串口与MCU。15 cm表示估计参考点满足判定，不是定位精度保证，也不是整车最终静止误差的实测结果。

#### FollowPath（全向PID路径跟踪）

##### 平移、旋转与TF参数表

以下增益单位按“位置误差m／角度误差rad，输出m/s／rad/s，dt用s”的本实现解释；不是设备单位或ROS参数系统强制的单位约束。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `pb_omni_pid_pursuit_controller::OmniPidPursuitController` | 不适用，类名 | **上实物前必须核对并正确配置** | 加载本工程全向PID跟踪器 | 不适用 | 不适用 | 需匹配安装插件与接口；当前输出平移标量再分解vx、vy，旋转另有开关 |
| `translation_kp` | `3.0` | s⁻¹ | **实物运行时重点观察，有问题再针对性调整** | 前视点距离误差的比例贡献 | 未限幅时同样距离给更强速度贡献，更易饱和或越过目标 | 比例贡献减弱，可能更慢或更难克服阻力 | 不是直接给3 m/s；误差通常随前视点变化；当前运行中改成员不会更新内部PID |
| `translation_ki` | `0.1` | s⁻² | **实物运行时重点观察，有问题再针对性调整** | 距离误差积分的速度贡献 | 积分影响更强，长期残留与任务切换影响可能更大 | 积分影响更弱；设0移除积分对输出的贡献 | 平移误差是非负距离，不是带符号的横向偏差；积分内部另有固定限幅，未按本YAML门槛配置 |
| `translation_kd` | `0.3` | 无量纲 | **实物运行时重点观察，有问题再针对性调整** | 前视距离误差变化率的速度贡献 | 误差变化贡献更强，前视点跳变可产生更大冲击／负标量 | 对误差变化更不敏感，也会减弱已有动态作用 | 差分除以固定0.05 s；不能概括成越大越稳定；首次pre_error=0 |
| `enable_rotation` | `false` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 是否用旋转PID计算本跟踪器的wz | 不适用 | 不适用 | 当前本插件wz恒为0；恢复Spin与cmd_spin附加自旋仍是其他入口；本开关无动态更新分支 |
| `rotation_kp` | `3.0` | s⁻¹ | **当前运行不使用这个参数，暂不调整** | 旋转yaw误差的比例贡献 | 启用旋转后：同样角差贡献增强 | 启用旋转后：比例贡献减弱 | 当前enable_rotation=False；需要实际参考系朝向能响应旋转反馈；运行中增益改值也未更新内部PID |
| `rotation_ki` | `0.1` | s⁻² | **当前运行不使用这个参数，暂不调整** | 旋转yaw误差积分贡献 | 启用后积分作用增强，残留可能更明显 | 启用后积分作用减弱 | 当前不计算旋转PID；不是云台电机的积分参数 |
| `rotation_kd` | `0.3` | 无量纲 | **当前运行不使用这个参数，暂不调整** | 旋转yaw误差变化率贡献 | 启用后对角差变化更敏感 | 启用后角差变化贡献减弱 | 当前不计算；与实际角速度输入门槛及MCU控制参数不同 |
| `transform_tolerance` | `0.1` | s，路径／位姿TF查询等待 | **通常保持当前值，不作为优先调试项** | 转换路径与位姿时等待所需TF | 可等更久，也可能拖慢控制循环 | 更快结束缺失TF的查询 | 不是角度、位置容差或时间戳补偿；部分转换失败返回值未检查，详见源码缺口 |
| `min_max_sum_error` | `1.0` | 未形成生效的单位约定 | **当前运行不使用这个参数，暂不调整** | 名称提示积分相关限制，但当前未接入计算 | 当前无计算效果 | 当前无计算效果 | 只声明、读取和更新成员；PID内部积分限幅硬编码±1，setSumError未被该控制器调用 |

##### 平移与旋转详解

###### translation_kp、translation_ki 与 translation_kd（控制的是前视点距离）

路径先转到local_costmap使用的机器人系，当前为gimbal_yaw_fake。选出前视点后计算：

```text
e = hypot(x_carrot, y_carrot)
theta = atan2(y_carrot, x_carrot)
平移PID输入：set_point=e，pv=0
PID标量 → 曲率减速 → 终点减速 → v
vx = v × cos(theta)
vy = v × sin(theta)
```

因此不是x、y各自独立一套PID，也不是用里程计vx去纠正目标vx。只有一个距离误差标量负责速度大小，方向来自前视点。[速度计算与分解](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c57/src/omni_pid_pursuit_controller.cpp#L219)

dt=0.05 s时，PID先算P=kp×e，积分累加e×dt，D=kd×(e−上一轮e)/dt，最后夹到v_linear_min～v_linear_max。以首次e=0.5 m为例：P=1.5 m/s，I=0.0025 m/s，D=3.0 m/s，合计4.5025 m/s，先被上限夹到2.5 m/s，随后才进入两级减速。此例只展示公式，未预测最终实车速度。

前视点距离突然增大时，D可能加强加速；突然减小时，D可能为负，甚至使总标量为负。当前下限-2.5允许这种结果，负v会朝前视点的反方向输出vx、vy。需要结合前视点跳变、实际循环延迟与速度链观察，不能把kd简单解释成通用的“刹车强度”。

积分项也有本实现特征：e是非负距离，长时间跟踪时可不断累计；PID先用当前积分算I，再把内部积分夹到±1，当前输出并非始终受到严格的I项上限保护。控制器setPlan只更新路径，没有重置PID积分和上一轮误差，任务或路径变化时的历史状态要一起考虑。[PID离散计算与状态](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c57/src/pid.cpp)

###### enable_rotation与三项rotation增益（当前只输出平移）

当前False使本插件angular.z=0。正常路径跟踪不因此失去横移能力；全向底盘可沿前视点方向平移。后续fake_vel_transform仍可叠加cmd_spin，行为服务器也有自己的恢复动作，不能据此断言底盘在整套系统中永不转动。

若以后启用旋转，默认旋转误差取前视位姿四元数的yaw，**不是**atan2(y_carrot,x_carrot)这个平移方向角；use_rotate_to_heading=True时改取当前转换后路径末端的yaw。需要确认虚拟参考系的朝向定义、目标路径姿态、底盘执行以及反馈，不能把这组参数当云台关节控制器。

###### min_max_sum_error与transform_tolerance（未接线的积分参数与真实TF等待）

min_max_sum_error=1.0不能据名称解释成“积分限制由1.0控制”。当前成员在configure和动态回调中赋值，但计算路径没有使用；PID内部固定夹±1，控制器也没有调用PID::setSumError。调整这个YAML值不会改变当前积分限制。

transform_tolerance则确实转成TF查询的timeout，供transformPose使用。它不移动时间戳、不补外参，也不把超时转换自动变成正确位姿。当前部分调用没有检查返回的False，问题见后面的源码缺口；单纯延长等待不能修复这一处理路径。

##### 前视点与先转向参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `lookahead_dist` | `2.0` | m，固定前视半径 | **当前运行不使用这个参数，暂不调整** | 关闭速度前视后，作为选点的固定距离 | 切换固定模式后：看更远，可能更平顺也更易切弯 | 切换后：看更近，可能更敏感或抖动 | 当前速度前视True覆盖本值；0.5～1.0 m边界也不会在固定模式自动夹住2 m |
| `use_velocity_scaled_lookahead_dist` | `true` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 选择速度×时间、上下界夹取的前视方式 | 不适用 | 不适用 | 当前采用过滤后vx、vy的模长；False才使用lookahead_dist |
| `lookahead_time` | `1.0` | s，速度前视系数 | **实物运行时重点观察，有问题再针对性调整** | 将测得的平面速度换算成前视距离 | 未触上下界时看更远，可能减少方向波动也更易跨过弯角 | 未触界时看更近，可能更敏感 | 不是预测运动1 s后的完整轨迹；到0.5／1.0 m夹限后改变可能无效果 |
| `min_lookahead_dist` | `0.5` | m，速度前视下界 | **实物运行时重点观察，有问题再针对性调整** | 低速时至少以0.5 m半径寻找前视点 | 低速也看更远，可能切弯或增加距离PID输入 | 低速看更近，可能更敏感或受点间距影响 | 当前只在速度前视模式使用；需0≤min≤max；没有足够远点时仍取局部路径末端 |
| `max_lookahead_dist` | `1.0` | m，速度前视上界 | **实物运行时重点观察，有问题再针对性调整** | 高速时将选点距离参数夹到1 m | 允许更远前视，但仍受局部路径范围约束 | 更早达到上界，高速也看更近 | 上界限制的是选点半径参数；关闭插值时，实际所选离散点可更远 |
| `use_interpolation` | `false` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 是否在跨越前视圆的路径线段上插入精确交点 | 不适用 | 不适用 | 当前直接选原路径点；True分支没有填有效姿态四元数，若配合旋转需核对；不控制曲率取样中的全部插值 |
| `use_rotate_to_heading` | `false` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 启用后，以当前局部路径末端yaw判定是否先将平移误差设零 | 不适用 | 不适用 | 该判断不受enable_rotation开关保护；旋转关闭却启用它，可能停止正常平移却没有wz去对齐 |
| `use_rotate_to_heading_treshold` | `0.1` | rad，约5.73° | **当前运行不使用这个参数，暂不调整** | 先转向模式下，超出该yaw误差就把平移距离误差设零 | 启用后允许更大朝向误差再抑制平移 | 启用后更小误差也触发 | 当前use_rotate_to_heading=False；拼写treshold是源码实际键名，不擅自改为threshold；置零误差不保证PID历史输出立刻为零 |
| `max_robot_pose_search_dist` | 约`2.49`，按当前网格配置推算的源码默认；YAML未配置 | m，沿路径搜索窗口 | **通常保持当前值，不作为优先调试项** | 限制寻找最近路径点时，从剩余路径开头向后搜索的累计长度 | 搜索更长，可能重新接到路径后段或自交另一支 | 搜索更短，偏离路径后可能找不到合适段 | 默认取局部地图米制尺寸getter的较大值÷2；当前100格、0.05 m时为2.4875 m；实际值随网格尺寸，当前无动态更新分支 |

##### 前视点与先转向详解

###### lookahead_dist与速度前视四项（当前L=clamp(速度×1 s，0.5 m，1 m)）

速度前视开启时，采用过滤后的平面速度模长：

```text
L = clamp(hypot(vx_odom_filtered, vy_odom_filtered) × lookahead_time,
          min_lookahead_dist, max_lookahead_dist)
```

纯计算例：输入有效平面速度为0、0.3、0.8、2.0 m/s时，L分别为0.5、0.5、0.8、1.0 m。固定lookahead_dist=2.0在这条分支不会使用；False才改用固定2 m，而且不会自动套用这两个速度前视上下界。[选点距离与分支](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c57/src/omni_pid_pursuit_controller.cpp#L472)

本实现按**距机器人原点的水平直线距离**找第一个不小于L的路径点，不是沿路径累计走L米。当前关闭插值，若相邻点距原点0.9、1.2 m而L=1 m，会选1.2 m这个点；因此1 m上界不是实际选点距离的绝对上限。没有足够远点时，改选当前局部路径末端。

前视变远、变近同时改变方向和PID距离误差，不能只将其理解成平滑度旋钮。弯道切角、终点减速过晚、低速抖动需要结合局部路径密度、曲率取样和速度过滤一起看。

###### use_interpolation（当前选原路径点；启用后还要核对姿态）

开启时，在圆内外相邻点形成的线段上算与前视圆的交点，使位置更接近指定半径。但当前这个新PoseStamped仅写位置和header，没有填有效的姿态四元数；当前关闭此分支且旋转关闭，不能把它说成已影响当前旋转输出。以后启用插值与旋转时需要一起检查。

曲率计算另用findPoseAtDistance插值取样，其路径不由use_interpolation控制。关闭前视插值不等于所有内部取样都使用原离散点。

###### use_rotate_to_heading及其treshold（看末端yaw，且不检查旋转是否开启）

开启后，算法改用**当前转换后局部路径末端**的yaw；超过0.1 rad就将lin_dist设为0，随后仍照常运行平移PID。局部路径可能只是全局路径的一段，因此不能一概说它检查的是最终目标朝向，更不是前视点位置的atan2方向。

如果enable_rotation仍False，wz继续为0，却可能反复触发上述平移抑制；如果PID还有积分和导数历史，lin_dist=0又不等于即时输出零。这些是此组合的源码行为，不应按“先转好、再直走”的注释直接保证动作效果。当前两开关均关闭这一问题组合。

###### max_robot_pose_search_dist（当前配置推算约2.49 m的最近路径点搜索）

本插件先在剩余路径开头一定累计长度内找最接近当前参考点的位置，剪掉此前的已过路径，再取局部地图范围附近的路径去换系。默认搜索长度取局部地图米制尺寸getter的较大值÷2，而不是直接把YAML的width除以2。

当前width=height=5 m、resolution=0.05 m，按100×100格推算，Costmap2D::getSizeInMeters返回(100−0.5)×0.05=4.975 m，故默认搜索长度为2.4875 m，约2.49 m。这里是按配置与源码推算，实际有效值应随运行网格尺寸核对。[地图米制尺寸getter（1.1.20）](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d.cpp#L511)

这与0.5～1.0 m前视半径、曲率前后取样距离都是不同步骤。路径自交、机器人偏离路径或重规划时，应看究竟选到了哪个段；它不是全局规划距离限制。[路径剪裁与搜索](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c57/src/omni_pid_pursuit_controller.cpp#L288)

##### 速度范围与终点减速参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `v_linear_min` | `-2.5` | m/s，平移PID标量下界 | **上实物前必须核对并正确配置** | 限制平移PID的负向标量输出 | 下界上移，如-2.5→-1，反向标量幅度更小 | 下界更负，允许更强的反向标量 | 不是vx、vy各自下界，也不是最低前进速度；负标量相对前视点方向反向；动态回调该分支为空 |
| `v_linear_max` | `2.5` | m/s，平移PID标量上界 | **上实物前必须核对并正确配置** | 限制未减速前的正向平移标量 | 允许更大的平移速度，仍受后续减速和底层限制 | 更早饱和在较小速度，可能改变跟踪表现 | 正常有限数值下，分解后的平移模长不因同时有vx、vy而增到sqrt(2)倍；需按实车运动能力设置 |
| `v_angular_min` | `-3.0` | rad/s，旋转PID下界 | **当前运行不使用这个参数，暂不调整** | 旋转开启后限制负wz输出 | 下界上移，允许的负转速幅度缩小 | 下界更负，允许更强负转速 | 当前旋转关闭；不限制其他行为或cmd_spin；动态回调分支为空 |
| `v_angular_max` | `3.0` | rad/s，旋转PID上界 | **当前运行不使用这个参数，暂不调整** | 旋转开启后限制正wz输出 | 允许更大正转速 | 正转速上限更低 | 当前旋转关闭；3 rad/s约171.89°/s，不是3°/s或云台角限位 |
| `min_approach_linear_velocity` | `0.5` | m/s，减速分支的速度下限 | **实物运行时重点观察，有问题再针对性调整** | 防止终点减速项压到太低，同时参与曲率阶段的两倍下限 | 在已有较快速度上，减速可能停在更高值 | 允许减速到更低，需观察低速执行与停车 | 不是所有输出至少0.5；最终取min不会主动抬高已有更慢速度；当前还使曲率下限为1.0 m/s |
| `approach_velocity_scaling_dist` | `1.0` | m，局部路径长度触发范围 | **实物运行时重点观察，有问题再针对性调整** | 局部转换路径长度不足1 m时启用接近减速 | 更早进入该条件，但不保证等同于更远离真实终点就减速 | 较短路径才进入，可能更晚减速 | 触发看局部路径累计长度，倍率看原点至末点直线距离／本值；宜为有限正数，注意局部截断 |

##### 速度范围与终点减速详解

###### 四项v…min／max（限制PID标量，不是每轴独立速度框）

平移PID先夹在[-2.5,+2.5] m/s，再按前视点theta分解。例如有限v=2 m/s、theta=45°时，vx、vy各约1.414 m/s，合成仍为2 m/s；不是两轴各取2 m/s。theta决定前视点方向，v的正负决定沿该方向或反方向移动。

当前enable_rotation=False，角速度上下界与旋转增益不会参与输出；恢复行为和附加自旋也不受这两项夹限。应按完整速度链核对实车限速，而不能把这一组参数视为MCU最后一道限幅。

此外，虽然controller_server订阅speed_limit并会调用插件setSpeedLimit，**这个插件的setSpeedLimit目前只有“未实现”的告警**，不会按消息改变限速。不能用该话题的存在证明这一路限速已经生效。[限速接口实现](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c57/src/omni_pid_pursuit_controller.cpp#L282)

###### min_approach_linear_velocity与approach_velocity_scaling_dist（0.5并非强制最低速度）

先计算当前局部转换路径自身的累计长度；小于1 m时，取“机器人原点到这个局部末点的直线距离÷1 m”作为接近倍率，其他情况倍率为1。它使用局部路径末点，不保证始终是全局最终目标；局部窗口或路径剪裁也会影响触发。

当前正向速度v进入该阶段后，可按下面理解：

```text
v_out = min(v, max(0.5, v × 接近倍率))
```

例如倍率0.2、进入速度1.5 m/s时，乘后0.3，减速项取下限0.5，输出0.5；若进入速度已是0.2，最终取min仍输出0.2，不会主动加到0.5。它不能保证解决低速死区，也不代替电机控制或停车判定。[接近减速实现](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c57/src/omni_pid_pursuit_controller.cpp#L486)

对于负PID标量，该代码的max下限与min组合会保留原负速度，并没有按相同倍率平滑地缩小其绝对值。反向突变要先核对D项与PID状态，不以“存在终点减速”保证不会发生。

当前局部地图约5 m边长、1 m触发距离未超出其约2.49 m的米制尺寸半值；若以后把触发距离设得很大，局部截断可能让车一直处于减速状态。15 cm目标容差与该减速分支也要一起看，机械制动效果仍待实测。

##### 曲率减速参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `curvature_min` | `0.4` | m⁻¹，开始减速的曲率 | **实物运行时重点观察，有问题再针对性调整** | κ严格大于此值才进入曲率减速分支 | 更多缓弯不进入减速，起点向更急弯移动 | 更缓的弯也进入减速 | 当前约对应2.5 m半径；需与curvature_max配套，保持min<max；不是坡度或朝向角 |
| `curvature_max` | `0.7` | m⁻¹，最大减速倍率的曲率界 | **实物运行时重点观察，有问题再针对性调整** | 曲率达到附近时采用配置的高曲率保留倍率 | 中间段同样κ通常保留更多速度，更急弯才到最低倍率 | 中间段减速更强，更早到最低倍率 | 当前约对应1.43 m半径；不禁止更急弯，不是最大可转角 |
| `reduction_ratio_at_high_curvature` | `0.5` | 无量纲，目标保留倍率 | **实物运行时重点观察，有问题再针对性调整** | 高曲率时目标速度取当前速度×0.5 | 保留更多速度，减速更轻 | 保留更少速度，目标减速更强 | 是保留比例，不是一般意义的“减掉比例”；实际还受变化率、1 m/s下限及后续接近减速影响 |
| `curvature_forward_dist` | `0.7` | m，曲率前方取样偏移 | **实物运行时重点观察，有问题再针对性调整** | 配合前视点和后方样本拟合三点圆 | 跨更长范围，可能平滑局部噪声，也可能跨过弯角 | 更局部，可能更敏感或遇到重复点 | 当前取样中心用前视点直线距离近似路径里程，非严格沿弧长以该点为中心 |
| `curvature_backward_dist` | `0.3` | m，曲率后方取样偏移 | **实物运行时重点观察，有问题再针对性调整** | 取得拟合圆的后方样本 | 取样跨度更大，可能更早受前段路径影响 | 跨度更小，局部变化与退化影响可能更大 | 路径不足会夹到首点／末点；三点退化时实现回退为极小曲率，不保证真实尖角被充分减速 |
| `max_velocity_scaling_factor_rate` | `0.9` | m/s²，源码实际限制速度标量变化率 | **实物运行时重点观察，有问题再针对性调整** | 曲率分支将目标减速速度向历史标量靠拢，单次步长按本值×dt夹取 | 此阶段能更快接近目标速度，过渡更急 | 此阶段更缓，但减速响应也可能更迟 | 当前单次步长0.045 m/s；不是无量纲倍率每秒变化0.9；不是整套vx、vy或底盘加速度的统一限制 |

##### 曲率减速详解

###### 曲率三项（越大越急；0.5表示目标保留一半）

算法从局部路径选三点拟合圆，κ=1/R，单位1/m。当前κ≤0.4时不进入曲率减速；0.4～0.7之间将保留倍率线性从1降到0.5；κ>0.7时目标保留倍率为0.5。

例如κ=0.55时位于两门槛中点，目标倍率为0.75；同样有限正速度2 m/s，目标减速速度为1.5 m/s。这里是目标，后面的速度变化步骤和下限还会改变实际输出，不能把注释直接当作每一轮都恰好减半。

κ越小，拟合半径越大；它与车辆yaw角、转速或坡道倾角不同。修改这几项不能修复地形节点把坡面当障碍、定位漂移或机械爬坡能力。保持合理的非负曲率门槛、min<max和0～1范围的保留倍率；当前代码没有完整的组合校验。

###### curvature_forward_dist与curvature_backward_dist（三点从哪里来）

代码先算路径各点从首点起的累计距离，却把前视点到机器人原点的直线距离当成取样中心里程，再按这个量减0.3、加0.7去路径里程数组中插值找两点。弯曲路径、首点偏置或选点距离与路径弧长不一致时，不能严格称为“从前视点沿路径后退0.3、前进0.7”。[取样与拟合实现](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c57/src/omni_pid_pursuit_controller.cpp#L549)

超出路径范围会夹到首末点。重复、共线或极小半径导致拟合半径无效时，当前实现回退R=1e9 m，所得曲率约1e-9 m⁻¹，反而基本不触发减速。应在RViz查看curvature_points_marker_array、lookahead_point和local_plan，判断取样是否对应真实转弯；调大窗口不是必然更安全或更准。

###### max_velocity_scaling_factor_rate与两倍最低速度（实际控制的是速度数值）

名称中的factor容易误导：成员last_velocity_scaling_factor_实际存的是**曲率阶段输出的速度标量**，而不是0～1的倍率。若κ>0.4，代码向目标速度最多跨过±0.9×0.05=±0.045 m/s；接着把减速项抬到至少2×min_approach_linear_velocity，即当前1.0 m/s，再与原速度取较小值。

因此，在已建立有效历史、忽略变化过渡并只看这个阶段时：原速度1.5 m/s、高曲率目标0.75，会被减速项的1.0下限限制为1.0，而非0.75；原速度0.8时最终仍为0.8，不会被主动加到1.0。进入后面的终点减速后，还可能进一步下降。

这个变化率不是底盘统一加速度上限：κ≤0.4时不做该步长限制；PID原值突然变小、最终min取值或后续接近缩放仍可能更快降低速度；方向变化引起的vx、vy变化也没有在这里逐轴限制。运动加减速需继续结合velocity_smoother和MCU参数检查。[曲率减速与历史状态实现](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c57/src/omni_pid_pursuit_controller.cpp#L519)

#### 源码缺口、动态修改与验证范围

##### 当前需要先明确的实现问题

| 项目 | 源码能确认什么 | 对参数整理与实车核对的影响 |
|---|---|---|
| 进度检查器键名 | 本版本读取单数progress_checker_plugin；当前复数键未用于选择 | 修正键名，避免将默认同名碰巧可用误认为复数列表生效；本次只给文档建议 |
| 积分限制参数 | min_max_sum_error成员没有进入计算；PID内部硬编码±1，且先算I再夹积分 | 不把该YAML值当有效积分限幅，不靠反复调它改善结果 |
| PID与任务历史 | setPlan未重置PID积分／上一轮误差 | 路径更新和再次接目标时可能承接历史；不能假定每个任务都是全新PID状态 |
| 曲率首轮历史 | last_velocity_scaling_factor_无成员初始化器，configure未赋初值；当前插件工厂用new C创建 | 首次就进入κ>0.4分支时会先读未初始化的历史标量；首次平缓路径会先赋值而掩盖此缺口。未有运行触发证据，需实现修复与运行验证，调0.9不能补初始化 |
| TF失败处理 | 转机器人位姿到全局路径的失败会抛异常；逐点转到机器人系以及碰撞取样换系的返回值未全部检查 | 日志有TF异常却继续产生输出，不证明数据有效；延长transform_tolerance不保证修复 |
| 碰撞检查边界 | 当前抽10个局部路径点查代价；遇到一个点落在地图外会立即返回未检测到碰撞 | 不是完整扫掠车体轨迹的保证；需核对地图、变换、膨胀和真实车体，不能靠PID参数覆盖检查缺口 |
| 动态限速接口 | setSpeedLimit只告警未实现 | 不依靠speed_limit消息保证此插件已限速 |

这些是源码核对结果，实际触发概率和实车后果尚未测量。本次仅整理文档，没有修改控制器实现。

##### 动态参数修改（哪些会更新，哪些只改变参数显示）

当前有动态回调，但并非所有设置都完整联动：

| 参数类别 | 当前回调行为 | 应怎样验证修改 |
|---|---|---|
| 服务器频率、三项输入速度阈值、failure_tolerance | 更新服务器成员；FollowPath运行期间顶层参数修改可能因锁被拒绝 | 停止任务后确认设置结果；频率还需重新配置PID以同步固定dt |
| 平移／旋转PID增益 | 更新外层增益成员，没有更新已创建PID对象中的增益 | 重新配置／重启，让configure重新创建PID，再验证输出 |
| 四项v…min／max | 动态回调的对应分支为空，既未更新内部PID限幅也未完整更新成员 | 重新配置／重启；不把设置服务成功当作已改限幅 |
| enable_rotation | 未列入布尔动态更新分支 | 重新配置／重启后核对实际wz与参考系 |
| 前视距离、速度前视开关、前视插值、先转向开关与门槛 | 相应类型匹配时会更新相关成员 | 仍要核对功能分支，当前固定前视／转向门槛关闭时不产生作用 |
| 接近减速、曲率六项、TF等待 | 更新有关成员 | 核对组合、实际选点和输出；当前缺口不会随参数更新自动消失 |
| max_robot_pose_search_dist | configure读取，未列入当前动态更新分支 | 重新配置／重启后检查路径搜索段 |
| 进度与目标检查器数值／stateful | 各自有动态更新回调 | 同时考虑已有基准或位置通过状态；新路径／新任务的reset时机仍适用 |

上表按当前源代码判断，不表示已在运行节点上逐项试改。[插件动态回调](https://github.com/SMBU-PolarBear-Robotics-Team/pb_omni_pid_pursuit_controller/blob/b542c57/src/omni_pid_pursuit_controller.cpp#L684)

##### 启动来源与本次核对范围

| 启动方式 | 本块来源与输出 |
|---|---|
| 整套reality | 默认config/reality/nav2_params.yaml，经navigation_launch加载；use_sim_time默认False，可被launch重写 |
| 整套simulation | 默认config/simulation/nav2_params.yaml，本块仅use_sim_time=True不同 |
| 组件／独立进程 | 本controller_server均读取处理后的同套参数，cmd_vel均重映射到cmd_vel_controller |
| slam=True／False | navigation_launch均启动本服务器；地图与定位来源不同，需要有效local_costmap及TF |

控制器返回TwistStamped，Humble服务器发布时取其中twist作为Twist，所以输出消息本身不带frame_id／时间戳；当前分量按gimbal_yaw_fake表达，由后续fake_vel_transform转换到实际云台系。话题和namespace接线见前面的[fake_vel_transform](/wiki/2026-09-20-dao-hang/0500-0600-fake-vel-transform)与[bt_navigator](/wiki/2026-09-20-dao-hang/0500-1000-bt-navigator)。

- **本次已核对：** 当前安装包版本、PID插件注册与库路径；服务器及两检查器安装头文件与Nav2 1.1.20一致；PID两安装头文件与本地源码一致；本地PID实现与上游固定提交一致；两份controller_server配置与安装副本一致，实物／仿真仅时钟值不同。
- **源码能确认：** 输入速度门槛的过滤对象与边界、0.3 s异常范围、单数进度键、净位移进度判定、6.28 rad朝向放宽、当前0.5～1.0 m速度前视、未接线积分参数、曲率速度下限／步长与动态修改缺口。
- **本次验证范围：** 未启动新的导航任务，未测量PID跟踪、急弯、低速横移、停车或恢复速度；文中的数值例是公式计算。库与配置存在不证明当前二进制全部行为已经运行验证。
- **实物待核实：** 上游速度系与时间戳、虚拟参考点TF、实际控制周期与输出断续、前视点跳变、低速横移过滤、终点停车及参考点偏差、曲率取样与首轮状态、碰撞检查边界、平滑器／串口／MCU限速和旋转接口。

