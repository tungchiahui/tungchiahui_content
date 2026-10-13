---
title: "bt_navigator（行为树导航任务调度）"
---

## bt_navigator（行为树导航任务调度）

### 实车先看：收到目标后，谁负责规划、行走和恢复

**bt_navigator按照行为树XML组织导航任务：请求规划、请求跟踪路径，并在失败时按树中的分支尝试恢复。** 实际路径由planner_server计算，正常行走速度由controller_server计算，恢复动作由behavior_server执行；本块参数决定如何组织这些调用。

当前工程的主要关系为：

```text
单点目标 → NavigateToPose → 单点行为树
多点目标 → NavigateThroughPoses → 多点行为树
                                ├─ ComputePath… → planner_server
                                ├─ FollowPath → controller_server
                                │                    → velocity_smoother
                                └─ 失败恢复：清代价地图／BackUp → 对应服务器

导航速度 → fake_vel_transform → 实际云台系cmd_vel → 串口 → MCU执行底盘运动
```

这里的“多点”指NavigateThroughPoses动作中按顺序经过一组位姿，不等同于waypoint_follower的FollowWaypoints流程。

| 实车上看到的现象 | 先核对什么 | 怎样理解 |
|---|---|---|
| 导航栈无法激活、提示XML或插件加载失败 | XML路径是否展开、文件是否存在、动态库是否匹配版本、树中引用的服务器是否已启动 | 有YAML不等于已经成功加载行为树；树创建时还会建立动作／服务客户端 |
| 误以为每秒重规划100次 | bt_loop_duration与XML中的RateController | 当前树检查目标周期10 ms，重新发起规划受XML的3 Hz节拍限制；正在运行的规划任务仍会继续被检查 |
| 导航超过0.1秒，以为应当超时退出 | default_server_timeout | 100 ms主要约束请求响应，不是整条路径的执行时限 |
| 失败后没有原地旋转或等待 | 实际XML，而不只是plugin_lib_names | 两棵默认树均未放入Spin、Wait；已加载插件不表示执行该动作 |
| 云台转动后机器人位置／速度表现异常 | map至gimbal_yaw_fake的TF、odometry、fake_vel_transform和MCU换系 | 本节点的robot_base_frame是虚拟参考系；不能把它理解成直接控制云台角度 |
| 单点可用，多点规划反复失败 | RemovePassedGoals的XML坐标端口 | 当前多点XML未指定robot_base_frame，该BT节点默认使用base_link，并不自动继承本块的gimbal_yaw_fake |

当前容器ysu_ros_humble中，nav2_bt_navigator、nav2_behavior_tree、nav2_util均为**1.1.20**，BehaviorTree.CPP为**3.8.7**。以下按这些版本及本地工程配置核对；官方在线文档包含后续发行版内容，不能直接把新版本参数和插件加载规则套到Humble。

### 参数

#### ros__parameters（ROS节点参数）

##### 参数表

下面保留当前实物配置。仿真段仅use_sim_time=True不同；两段的其余参数及47项插件列表一致。

```yaml
bt_navigator:
  ros__parameters:
    use_sim_time: False
    global_frame: map
    robot_base_frame: gimbal_yaw_fake
    odom_topic: odometry
    bt_loop_duration: 10
    default_server_timeout: 100
    wait_for_service_timeout: 1000
    default_nav_to_pose_bt_xml: $(find-pkg-share pb2025_nav_bringup)/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml
    default_nav_through_poses_bt_xml: $(find-pkg-share pb2025_nav_bringup)/behavior_trees/navigate_through_poses_w_replanning_and_recovery.xml
    plugin_lib_names:
      - nav2_compute_path_to_pose_action_bt_node
      - nav2_compute_path_through_poses_action_bt_node
      - nav2_smooth_path_action_bt_node
      - nav2_follow_path_action_bt_node
      - nav2_spin_action_bt_node
      - nav2_wait_action_bt_node
      - nav2_assisted_teleop_action_bt_node
      - nav2_back_up_action_bt_node
      - nav2_drive_on_heading_bt_node
      - nav2_clear_costmap_service_bt_node
      - nav2_is_stuck_condition_bt_node
      - nav2_goal_reached_condition_bt_node
      - nav2_goal_updated_condition_bt_node
      - nav2_globally_updated_goal_condition_bt_node
      - nav2_is_path_valid_condition_bt_node
      - nav2_initial_pose_received_condition_bt_node
      - nav2_reinitialize_global_localization_service_bt_node
      - nav2_rate_controller_bt_node
      - nav2_distance_controller_bt_node
      - nav2_speed_controller_bt_node
      - nav2_truncate_path_action_bt_node
      - nav2_truncate_path_local_action_bt_node
      - nav2_goal_updater_node_bt_node
      - nav2_recovery_node_bt_node
      - nav2_pipeline_sequence_bt_node
      - nav2_round_robin_node_bt_node
      - nav2_transform_available_condition_bt_node
      - nav2_time_expired_condition_bt_node
      - nav2_path_expiring_timer_condition
      - nav2_distance_traveled_condition_bt_node
      - nav2_single_trigger_bt_node
      - nav2_goal_updated_controller_bt_node
      - nav2_is_battery_low_condition_bt_node
      - nav2_navigate_through_poses_action_bt_node
      - nav2_navigate_to_pose_action_bt_node
      - nav2_remove_passed_goals_action_bt_node
      - nav2_planner_selector_bt_node
      - nav2_controller_selector_bt_node
      - nav2_goal_checker_selector_bt_node
      - nav2_controller_cancel_bt_node
      - nav2_path_longer_on_approach_bt_node
      - nav2_wait_cancel_bt_node
      - nav2_spin_cancel_bt_node
      - nav2_back_up_cancel_bt_node
      - nav2_assisted_teleop_cancel_bt_node
      - nav2_drive_on_heading_cancel_bt_node
      - nav2_is_battery_charging_condition_bt_node
```

前10项为本次配置；最后两项为相关源码默认值，YAML未显式配置，不代表需要把它们追加到当前文件。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真`True` | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择ROS时钟，影响TF、任务时间和部分超时计算 | 不适用 | 不适用 | 整套launch会重写已有键；实物、仿真及相关节点须采用对应时间域；BT主循环另用WallRate |
| `global_frame` | `map` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定本导航器查询机器人当前位姿的全局参考系 | 不适用 | 不适用 | 需要map与机器人参考系之间的有效TF；不会由此参数创建map、加载地图或完成重定位 |
| `robot_base_frame` | `gimbal_yaw_fake` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定本导航器跟踪的机器人参考点与方向 | 不适用 | 不适用 | 当前由fake_vel_transform提供虚拟系；不是底盘实体link或实际云台机械角；部分BT节点另有自己的XML坐标端口 |
| `odom_topic` | `odometry` | 不适用，话题名称 | **上实物前必须核对并正确配置** | 订阅Odometry的速度，建立约0.3 s窗口的速度均值，供预计剩余时间等使用 | 不适用 | 不适用 | 相对话题名随节点命名空间解析；不是从消息pose代替TF定位；本工程上游速度含义和时间差分问题见sensor_scan_generation |
| `bt_loop_duration` | `10` | ms，BT检查周期 | **通常保持当前值，不作为优先调试项** | 设置行为树运行时反复检查各节点状态的目标周期 | 检查更稀，通常减少调度负担，取消／分支响应可能更迟 | 检查更密，通常增加负担，耗时较长时可能反复提示超过节拍 | 当前目标约100 Hz，不保证实测达到；不直接设置规划器、控制器或雷达频率；应为正整数 |
| `default_server_timeout` | `100` | ms，请求响应超时 | **实物运行时重点观察，有问题再针对性调整** | 为BT动作／服务客户端提供默认响应等待上限 | 更容忍短时响应延迟，失败确认也可能更迟 | 更快判超时，也更容易在负载高时误失败 | 当前0.1 s；动作主要等目标确认及取消相关响应，服务等请求结果；不是路径总执行时限；XML的server_timeout可覆盖 |
| `wait_for_service_timeout` | `1000` | ms，服务器发现等待 | **通常保持当前值，不作为优先调试项** | 创建BT节点客户端时，等待动作／服务服务器出现 | 更容忍启动发现延迟，缺服务器时加载等待更长 | 更快发现缺项，也可能在启动尚未就绪时加载失败 | 当前每次等待上限1 s，不是整棵树启动总超时或运行中请求超时；命名空间／服务器名错误不能靠延长解决 |
| `default_nav_to_pose_bt_xml` | `pb2025_nav_bringup`包内单点XML，路径见上方 | 不适用，文件路径 | **上实物前必须核对并正确配置** | 为NavigateToPose选择默认行为树 | 不适用 | 不适用 | launch展开find-pkg-share；动作请求可指定另一棵树；当前默认树包含重规划、清图及自定义空闲方向移动恢复 |
| `default_nav_through_poses_bt_xml` | `pb2025_nav_bringup`包内多点XML，路径见上方 | 不适用，文件路径 | **上实物前必须核对并正确配置** | 为NavigateThroughPoses选择默认行为树 | 不适用 | 不适用 | 当前有RemovePassedGoals radius=0.7 m；其默认base_link需要单独核对／显式配置，不由本块robot_base_frame自动替换 |
| `plugin_lib_names` | 上方47个动态库名称 | 不适用，字符串列表 | **上实物前必须核对并正确配置** | 加载动态库，注册XML可引用的BT节点类型 | 不适用 | 不适用 | 当前47项均参与库加载；两棵默认树实际引用其中11项；增加库不会自动增加动作，缺库即使未入树也可能导致配置失败 |
| `transform_tolerance` | `0.1`，源码默认；YAML未配置 | s，TF查询等待时间 | **通常保持当前值，不作为优先调试项** | 本导航器及有关BT节点查询机器人位姿时等待TF | 可等更久，缺TF时处理延迟也可能增加 | 更快结束缺失变换的查询 | 当前经getCurrentPose传给TF查询timeout；不是允许定位误差、角度误差或机械外参误差 |
| `always_reload_bt_xml` | `False`，源码默认；YAML未配置 | 不适用，布尔 | **根据实际功能需求开启或设置** | 收到导航目标时，决定相同路径的行为树是否也重新读取、创建 | 不适用 | 不适用 | False会复用同一路径的已加载树；True有助于XML调试，但增加加载开销；不等于运行任务中实时监听文件修改 |

##### 参数详解

###### use_sim_time（时钟选择与launch覆盖）

实物默认False，仿真默认True。navigation_launch.py通过RewrittenYaml重写use_sim_time，并用ParameterFile加载；本节点已有该键，因此能够被覆盖。运行时究竟使用哪个值，应按当前启动参数和有效节点参数判断，不能只看实物YAML里的False。

ROS时钟用于TF和任务时间，也用于BT客户端部分请求的已等待时间。BT主循环则用rclcpp::WallRate，当前RateController的节拍用真实高精度时钟；仿真暂停／变速时，不能认为把use_sim_time设为True后，所有内部等待与检查节拍都跟着仿真时间同比变化。[BT运行循环（1.1.20）](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/src/behavior_tree_engine.cpp)、[RateController实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/plugins/decorator/rate_controller.cpp)

###### global_frame 与 robot_base_frame（在哪里看机器人，跟踪哪个参考点）

当前导航器查询的是**gimbal_yaw_fake原点在map中的位姿**，用于导航反馈等。map至该系的变换需要上游定位、底盘位姿、机械TF以及fake_vel_transform共同组成；这里的参数不会替这些节点建立变换。[导航器坐标与反馈接线（1.1.20）](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_bt_navigator/src/bt_navigator.cpp)

按前面的工程约定，gimbal_yaw_fake与实际gimbal_yaw原点重合，方向抵消输入里程计yaw。在平面运动、输入和机械TF正确时，它的水平朝向与odom对齐；机器人位置仍在移动。它不是map，也不与odom原点重合；源码只抵消yaw，坡道上的roll／pitch仍需核对。详见[fake_vel_transform](/wiki/2026-09-20-dao-hang/0500-0600-fake-vel-transform)。

这套配置为全向移动与云台独立扫描服务。只把此项改成base_footprint，并不会同步修改控制器、代价地图、behavior_server或后面的速度换系；若要改变整套参考系，应先检查各处职责与原点偏置。

**同名ROS参数不会自动替换每个BT节点的XML输入端口。** 有些节点从导航器复制的ROS参数读取坐标，有些从XML端口读取，而且端口带自己的默认值。当前多点树的RemovePassedGoals就是后一种，具体问题见下文。

坐标系名称也不会因为ROS节点有namespace而自动加前缀。应按实际TF frame名称配置；namespace主要影响节点、相对话题、服务和动作名称。

###### odom_topic（用于速度反馈，不代替TF定位）

本工程odometry由sensor_scan_generation发布，位姿参考系预期为odom，child_frame_id为gimbal_yaw；本节点却跟踪gimbal_yaw_fake，两者名称不同有各自的用途，不能要求只为字符串一致就改话题或子系。

BtNavigator用这个话题建立OdomSmoother，窗口为0.3 s，直接平均消息的twist各分量，没有在这里重新做坐标变换。导航器用平面速度大小sqrt(vx²+vy²)估计剩余时间；当前实现仅在速度大于0.01 m/s且路径剩余长度大于0.1 m时计算，其他情况保持默认预计时间。它不是MCU实际轮速，也不是下一条控制指令。[速度窗口实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_util/src/odometry_utils.cpp)、[导航反馈计算](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_bt_navigator/src/navigators/navigate_to_pose.cpp)

当前上游把参考系中的位姿差分直接写入twist，且用回调真实时间差而非消息时间差，首帧历史也有缺口；这些已经记录在[sensor_scan_generation](/wiki/2026-09-20-dao-hang/0500-0300-sensor-scan-generation)。纯平面旋转换轴不会改变单帧速度模长，但错误时间差、首帧异常，以及先平均不同方向分量再取模长，都可能影响反馈。不能凭预计剩余时间看起来合理就证明上游速度正确。

odometry是相对名。例如节点位于/red_standard_robot1时，通常订阅/red_standard_robot1/odometry；最终还受remapping影响。本节点的当前位置取自TF，改odom_topic不能修复map至gimbal_yaw_fake的缺失变换。

###### bt_loop_duration（10 ms检查一次，不是100 Hz重规划）

tick可理解为“检查行为树当前执行到哪里、动作是否完成、是否需要换分支”。10 ms对应目标约100次／秒；某次树检查、客户端等待或其他计算超过这个周期，就达不到目标节拍，可能出现tick rate exceeded提示。

当前相关节拍分别为：

| 所在位置 | 当前设置 | 控制什么 |
|---|---|---|
| bt_navigator.bt_loop_duration | 10 ms，目标约100 Hz | 反复检查行为树状态 |
| 两棵XML中的RateController | hz=3.0 | 规划分支完成后，再发起新一轮规划的节拍，约0.333 s |
| controller_server.controller_frequency | 20 Hz | 路径跟踪器的控制循环，本块不会修改它 |

RateController的子节点已经处于RUNNING时，每次外层检查都会继续检查它，直到结束；因此3 Hz不是“任何情况下都每0.333秒才理一次规划”。实际重规划频率还受规划耗时、恢复分支和BT调度影响。

减小10只会让外层检查更密，不能让服务器、点云或TF自动变快。若CPU忙、定位跳变或控制断续，应先查对应链路；不能把加快BT检查当作提高导航精度。源码读取整数毫秒，没有完整的非正值保护，保持正整数。[周期读取](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/include/nav2_behavior_tree/bt_action_server_impl.hpp)

###### default_server_timeout（100 ms等请求响应，不限制整段导航时长）

100表示**100 ms，即0.1 s**。需要区分动作和服务：

- **动作客户端：** 发出ComputePathToPose、FollowPath或BackUp等目标后，等待服务器确认收到／接受目标；接受之后，任务可以保持RUNNING并继续执行，不会因为走了超过0.1秒就被此项终止。取消动作及停止时等待相关返回，也使用该响应超时。
- **服务客户端：** 例如ClearEntireCostmap，等待本次请求的服务结果；超过上限仍没有结果时，该BT节点会失败。

动作执行本身有另外的规则：路径跟踪有进度检查；BackUp有time_allowance等。这些不能用default_server_timeout替代。[动作确认与取消实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/include/nav2_behavior_tree/bt_action_node.hpp)、[服务响应等待实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/include/nav2_behavior_tree/bt_service_node.hpp)

每个动作／服务节点可以通过XML的server_timeout端口覆盖默认值。当前两棵XML没有显式设置该端口，使用100 ms；当前安装头文件的端口没有另设一个固定默认值，因此不能直接套用其他版本“端口默认值覆盖ROS参数”的结论。

也不能认为一次tick必定阻塞100 ms。当前模板把单次等待片段限制到bt_loop_duration的一半，即5 ms；尚未到总响应上限时返回RUNNING，下次tick继续判断。100 ms是总请求等待尺度，5 ms是单次等待片段，两者不矛盾；停止动作时的取消等待则有其单独调用路径。

实物先用100。若日志明确出现等待目标确认／服务响应超时，先核对服务器是否正常、命名空间、计算负载与通信，再考虑适度增加；延长不会修好碰撞、无路可走或定位失败。

###### wait_for_service_timeout（1000 ms找服务器，何时会使用）

1000表示1 s。虽然名称带service，当前BtActionNode也用它调用wait_for_action_server；服务节点用它调用wait_for_service。

它主要在创建XML中的动作／服务节点客户端时使用。整棵树创建会实例化恢复分支里的节点，**即使机器人还没有遇到失败，也可能已经需要backup动作服务器及清图服务可发现**。但只是加载一个未被XML引用的插件库，不等于已经实例化它对应的动作节点。[客户端创建与发现等待](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/include/nav2_behavior_tree/bt_action_node.hpp)

1 s是每次发现等待的上限，不是整套导航启动限时。增大可容忍服务器出现较慢；若树中action/service名字与namespace错误，服务器一直不会被发现，再长也无法解决。运行中已发出请求的等待，应看上一项。

###### 两项default_nav…bt_xml（默认行为树与路径展开）

两条路径指向本工程pb2025_nav_bringup包，而不是nav2_bt_navigator包的同名默认示例；文件名相同也不能据此认为树的内容相同。

`$(find-pkg-share pb2025_nav_bringup)`是launch替换表达式，navigation_launch.py使用ParameterFile(allow_substs=True)才会展开成包的share目录。YAML文件本身、普通字符串读取以及BtNavigator的文件打开过程不会单独解释这段表达式；若绕过该launch直接给节点传原始文件，应提供可用的实际路径或相应的替换处理。

默认采用当前环境找到的安装包。修改源码里的XML后，需核对安装目录是否为链接／同步副本；本次检查两份已安装XML与当前源码一致。其他机器或重建方式变化后应重新核对。

单点和多点导航动作的请求里都有behavior_tree路径字段：为空时用各自默认树，提供其他路径时加载该文件。当前导航器激活时也会加载默认XML。切换树不等于随意允许同时执行两个导航任务，当前导航器还有任务互斥逻辑。[单点默认路径与目标处理](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_bt_navigator/src/navigators/navigate_to_pose.cpp)、[多点对应实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_bt_navigator/src/navigators/navigate_through_poses.cpp)

两棵当前默认树的共同流程：

1. PipelineSequence组织持续重规划与路径跟踪。规划请求指定planner_id="GridBased"，跟踪指定controller_id="FollowPath"。
2. 规划失败时尝试清全局代价地图再重试，该内层RecoveryNode为number_of_retries=1；跟踪失败时尝试清局部代价地图再重试，该内层为10。
3. 内层仍失败后，进入外层恢复。先检查GoalUpdated是否有新目标；否则RoundRobin轮换“清局部及全局代价地图”与“BackUp”。外层RecoveryNode为10次恢复重试；不是整个任务至多调用所有服务器10次。

清图调用的是代价地图清理服务，不是删除磁盘里的二维地图图片、PCD或SLAM位姿图。静态层和实时障碍输入还可能把占用重新加入地图。

**当前树没有Spin、Wait、SmoothPath或IsStuck节点。** 不会仅因为库列表有这些名称，就自动增加原地转动、等待、路径平滑或该卡住检测条件。当前规划和跟踪结果失败仍可触发RecoveryNode，不依赖IsStuck必须入树。

BackUp在XML中请求backup_dist=1.0 m、backup_speed=1.0 m/s，未写time_allowance，BT端口默认10 s。当前behavior_server的backup后端是**pb_nav2_behaviors/BackUpFreeSpace**：它按代价地图选择空闲方向，再做平面移动，并非固定沿车后方向倒退。请求值是目标距离和速度，不能据此声称实车已经以该速度正确移动；具体方向、碰撞检查与速度输出应在后续behavior_server参数块一起核对。

###### 多点树的RemovePassedGoals（0.7 m中间点门槛与坐标端口缺项）

多点XML在ComputePathThroughPoses前放入RemovePassedGoals，radius=0.7 m。当前实现会从列表开头依次删除距查询位置不超过该门槛的中间目标，**至少保留最后一个目标**；这不是“所有目标都只要靠近0.7 m就算导航完成”，也不是最终停车的xy_goal_tolerance。[移除中间目标实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/plugins/action/remove_passed_goals_action.cpp)

这里有一个可以由当前XML和安装头文件确认的配置缺口：

- 本块ROS参数设置robot_base_frame=gimbal_yaw_fake。
- RemovePassedGoals从自己的XML输入端口读取坐标系，默认global_frame=map、robot_base_frame=base_link。
- 当前XML只给input_goals、output_goals、radius，没有给两个坐标端口，因此查询的是map至base_link，不会自动变成map至gimbal_yaw_fake。

若当前TF中没有base_link，查询会失败，使多点规划分支返回FAILURE；即使存在，也需确认其原点是否就是预期判定参考点。前面的模型与导航主链使用base_footprint、gimbal_yaw及gimbal_yaw_fake，不能凭库已经存在就跳过这个检查。[端口默认值（1.1.20）](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/include/nav2_behavior_tree/plugins/action/remove_passed_goals_action.hpp)

若沿用本工程虚拟参考点，XML可明确写成：

```xml
<RemovePassedGoals input_goals="{goals}"
                   output_goals="{goals}"
                   radius="0.7"
                   global_frame="map"
                   robot_base_frame="gimbal_yaw_fake"/>
```

这是相应XML的配置建议，**不是本次已修改或已运行验证的结果**。只改本块ROS参数并不能补上XML端口。若换global_frame，两个位置也应一起核对；以后使用DistanceController、GoalReached等其他带坐标端口的BT节点时，同样不能假定自动继承。

###### plugin_lib_names（动态库名称、XML标签与执行动作的区别）

列表中每项是动态库名称，Linux下通常对应lib名称.so文件；XML写的是库注册的BT节点标签。两者不是同一个字符串。例如nav2_follow_path_action_bt_node注册FollowPath，实际跟踪算法则来自controller_server中的FollowPath插件配置。

Humble 1.1.20的BehaviorTreeEngine逐项加载这里配置的库并注册节点类型；列表参与加载，不按当前XML自动过滤。47个库中本次均找到对应安装文件，但文件存在不等于所有库加载、树创建和导航执行已经运行通过。[动态库加载实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/src/behavior_tree_engine.cpp)

下面逐项列出当前配置；“入树”指XML中存在该节点，恢复节点仍要满足失败分支条件才执行。两棵默认树合计引用11个库，其余36个库已配置加载，但没有对应节点进入这两棵树。

##### 插件列表详解（当前默认树引用的11个库）

| 动态库名称 | 注册的XML节点 | 作用 | 当前默认树中的用途 |
|---|---|---|---|
| `nav2_compute_path_to_pose_action_bt_node` | `ComputePathToPose` | 请求规划到单个目标的路径 | 单点树使用；planner_id=GridBased |
| `nav2_compute_path_through_poses_action_bt_node` | `ComputePathThroughPoses` | 请求依次经过一组目标的路径 | 多点树使用；planner_id=GridBased |
| `nav2_follow_path_action_bt_node` | `FollowPath` | 请求控制器跟踪给定路径 | 两树使用；controller_id=FollowPath；算法仍由controller_server选择 |
| `nav2_back_up_action_bt_node` | `BackUp` | 请求backup动作服务器执行恢复移动 | 两树的恢复分支使用；当前后端是自定义BackUpFreeSpace |
| `nav2_clear_costmap_service_bt_node` | `ClearEntireCostmap`、`ClearCostmapExceptRegion`、`ClearCostmapAroundRobot` | 调用代价地图清理服务 | 两树只使用ClearEntireCostmap；另外两个标签没有入树 |
| `nav2_goal_updated_condition_bt_node` | `GoalUpdated` | 判断当前目标或目标列表是否更新 | 两树恢复分支检查；有新目标时可结束当前恢复分支并重试导航 |
| `nav2_rate_controller_bt_node` | `RateController` | 按给定节拍重新触发子分支 | 两树规划分支hz=3.0；不设底盘速度 |
| `nav2_recovery_node_bt_node` | `RecoveryNode` | 主子节点失败时执行恢复子节点，再按次数重试 | 两树均有外层和内层；重试次数属于XML端口 |
| `nav2_pipeline_sequence_bt_node` | `PipelineSequence` | 在后续子节点运行时继续检查前面的子节点 | 两树组织重规划与跟踪的流水执行 |
| `nav2_round_robin_node_bt_node` | `RoundRobin` | 轮换选择子分支 | 两树在清图组合与BackUp之间轮换 |
| `nav2_remove_passed_goals_action_bt_node` | `RemovePassedGoals` | 删除已接近的列表前部中间目标，保留最后一个 | 仅多点树使用；radius=0.7 m；需核对默认base_link端口 |

##### 插件列表详解（当前默认树未引用的36个库）

以下是已配置的可用节点类型。当前未入树，不能依靠修改这些库名称开启相应功能；今后采用其他XML时，需要重新判断节点是否使用、端口是否正确和服务器是否存在。

| 动态库名称 | 注册的XML节点 | 作用 | 当前配置下的说明 |
|---|---|---|---|
| `nav2_smooth_path_action_bt_node` | `SmoothPath` | 请求smoother_server平滑路径 | 未入树；启动smoother_server不表示当前路径必经平滑 |
| `nav2_spin_action_bt_node` | `Spin` | 请求spin动作服务器执行转动 | 未入树；与fake_vel_transform叠加底盘自旋不是同一入口 |
| `nav2_wait_action_bt_node` | `Wait` | 请求wait动作等待指定时间 | 未入树；不由wait_for_service_timeout开启 |
| `nav2_assisted_teleop_action_bt_node` | `AssistedTeleop` | 请求带碰撞检查的辅助遥控行为 | 未入树；不等同于普通手柄节点已经启用 |
| `nav2_drive_on_heading_bt_node` | `DriveOnHeading` | 请求按指定方向／距离移动的行为 | 未入树；不直接选择正常路径跟踪算法 |
| `nav2_is_stuck_condition_bt_node` | `IsStuck` | 依据里程计变化判断疑似卡住 | 未入树；此版本主要检查x速度的突变减速，不是通用的静止超时检测 |
| `nav2_goal_reached_condition_bt_node` | `GoalReached` | 检查目标与查询位置之间的平面距离 | 未入树；不替代当前controller_server的目标检查器，也不检查最终yaw误差 |
| `nav2_globally_updated_goal_condition_bt_node` | `GlobalUpdatedGoal` | 用其内部记录检测目标／目标列表变化 | 未入树；XML标签是GlobalUpdatedGoal，不能按库名猜成GloballyUpdatedGoal |
| `nav2_is_path_valid_condition_bt_node` | `IsPathValid` | 调用规划器提供的路径有效性检查服务 | 未入树；当前没有这项独立检查分支 |
| `nav2_initial_pose_received_condition_bt_node` | `InitialPoseReceived` | 检查黑板中的initial_pose_received标志 | 未入树；不会自动接入small_gicp或证明定位成功 |
| `nav2_reinitialize_global_localization_service_bt_node` | `ReinitializeGlobalLocalization` | 请求重新初始化全局定位的服务 | 未入树；当前small_gicp不因加载此库就自动提供该接口 |
| `nav2_distance_controller_bt_node` | `DistanceController` | 按机器人移动距离重新触发子分支 | 未入树；不是底盘行驶距离命令；启用需核对自己的坐标端口 |
| `nav2_speed_controller_bt_node` | `SpeedController` | 按当前速度调整子分支触发节拍 | 未入树；不是直接限速或速度控制器 |
| `nav2_truncate_path_action_bt_node` | `TruncatePath` | 按距离截短路径末端 | 未入树；当前没有由它设定提前停车点 |
| `nav2_truncate_path_local_action_bt_node` | `TruncatePathLocal` | 根据参考位姿提取／截短局部路径段 | 未入树；不替代代价地图的局部窗口 |
| `nav2_goal_updater_node_bt_node` | `GoalUpdater` | 接收目标更新并传给子分支 | 未入树；动态目标跟随需在XML中接线 |
| `nav2_transform_available_condition_bt_node` | `TransformAvailable` | 检查指定两系的变换是否存在 | 未入树；本版本找到过一次后会记住成功，不是持续TF新鲜度检查 |
| `nav2_time_expired_condition_bt_node` | `TimeExpired` | 判断指定计时门槛是否到达 | 未入树；不等同于default_server_timeout |
| `nav2_path_expiring_timer_condition` | `PathExpiringTimer` | 根据路径更新情况和时间门槛判断是否需更新 | 未入树；库名本身没有_bt_node后缀，不能擅自补后缀 |
| `nav2_distance_traveled_condition_bt_node` | `DistanceTraveled` | 判断自记录位置起是否移动到指定距离 | 未入树；是条件判断，不下发运动指令 |
| `nav2_single_trigger_bt_node` | `SingleTrigger` | 使子分支在一次进入期间执行一轮 | 未入树；不是让整套导航永远只运行一次 |
| `nav2_goal_updated_controller_bt_node` | `GoalUpdatedController` | 在首次执行或目标变化时触发子分支，并继续处理运行状态 | 未入树；是装饰节点，不是GoalUpdated条件的同名替代 |
| `nav2_is_battery_low_condition_bt_node` | `IsBatteryLow` | 按电池消息与阈值判断电量低 | 未入树；不因库加载就自动回充 |
| `nav2_navigate_through_poses_action_bt_node` | `NavigateThroughPoses` | 在另一棵树中作为客户端请求多点导航动作 | 未入树；本节点提供多点导航服务器不依赖在自己的树中嵌入此标签 |
| `nav2_navigate_to_pose_action_bt_node` | `NavigateToPose` | 在另一棵树中作为客户端请求单点导航动作 | 未入树；不能为“开启单点功能”就把自己调用自己的标签放入默认树 |
| `nav2_planner_selector_bt_node` | `PlannerSelector` | 从话题／默认输入选规划器ID，输出到黑板 | 未入树；当前XML直接指定GridBased |
| `nav2_controller_selector_bt_node` | `ControllerSelector` | 从话题／默认输入选控制器ID | 未入树；当前XML直接指定FollowPath |
| `nav2_goal_checker_selector_bt_node` | `GoalCheckerSelector` | 从话题／默认输入选目标检查器ID | 未入树；当前没有这项话题选择流程 |
| `nav2_controller_cancel_bt_node` | `CancelControl` | 请求取消正在执行的FollowPath动作 | 未入树；不表示普通任务取消失效，动作节点停止时另有取消逻辑 |
| `nav2_path_longer_on_approach_bt_node` | `PathLongerOnApproach` | 接近终点且新路径明显变长时触发子分支 | 未入树；不自动增加绕障或等待流程 |
| `nav2_wait_cancel_bt_node` | `CancelWait` | 请求取消Wait动作 | 未入树；没有自行执行等待的作用 |
| `nav2_spin_cancel_bt_node` | `CancelSpin` | 请求取消Spin动作 | 未入树；不取消cmd_spin附加自旋 |
| `nav2_back_up_cancel_bt_node` | `CancelBackUp` | 请求取消BackUp动作 | 未入树；当前BackUp节点停止时自身已有取消路径 |
| `nav2_assisted_teleop_cancel_bt_node` | `CancelAssistedTeleop` | 请求取消AssistedTeleop动作 | 未入树；不直接关闭普通手柄节点 |
| `nav2_drive_on_heading_cancel_bt_node` | `CancelDriveOnHeading` | 请求取消DriveOnHeading动作 | 未入树；不替代正常路径跟踪的取消 |
| `nav2_is_battery_charging_condition_bt_node` | `IsBatteryCharging` | 按电池消息判断是否正在充电 | 未入树；不表示本工程已有充电控制与电池接线 |

各库与标签的对应关系按[1.1.20的动态库构建列表](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/CMakeLists.txt)及[各BT节点注册源码](https://github.com/ros-navigation/navigation2/tree/1.1.20/nav2_behavior_tree/plugins)核对。Sequence、ReactiveSequence、ReactiveFallback是BehaviorTree.CPP内置控制节点，因此当前XML中有这些标签，并不要求列表另有同名Nav2动态库。

###### 读取时机、启动来源与验证范围

| 启动方式 | 本块参数与行为树来源 |
|---|---|
| 整套reality | 默认config/reality/nav2_params.yaml，可由params_file替换；navigation_launch重写use_sim_time；XML路径经allow_substs展开 |
| 整套simulation | 默认config/simulation/nav2_params.yaml；本块仅use_sim_time与实物段不同；相同两棵工程XML |
| use_composition=True／False | 本节点分别作为组件／独立进程启动；两者读取同一套处理后的参数文件 |
| slam=True／False | bringup_launch均包含navigation_launch；建图开关不会单独跳过bt_navigator，定位／建图TF来源则不同 |
| 绕过工程launch直接启动 | 不能假定find-pkg-share字符串已经展开；需核对参数、namespace、TF重映射与服务器启动状态 |

global_frame、robot_base_frame、odom_topic及插件列表主要在configure阶段读入；时间参数配置到BT动作服务器及黑板；默认XML在configure时选定、activate时加载。当前没有让运行中修改这些参数后自动重建TF使用者、里程计订阅、插件工厂和超时成员的完整更新回调。修改后重新配置／重启并核对有效值；标准use_sim_time的时钟处理另属ROS机制。

always_reload_bt_xml=False时，同一路径的已加载树会复用。只修改磁盘XML再发同一路径目标，不能假定内容已更新；可在调试时配置该开关后重新配置，或重新启动加载。True也不是在机器人执行任务途中随文件修改即时切换流程。[树文件复用与重新加载实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/include/nav2_behavior_tree/bt_action_server_impl.hpp)

- **本次已核对：** 容器内包版本；动作／服务两个安装头文件与1.1.20源码一致；47个配置库文件全部存在；源码与安装副本的实物／仿真bt_navigator段一致；两份工程XML与安装副本一致，XML内容与插件注册标签已静态对照。
- **源码能确认：** 三项时间单位均为ms；BT检查与3 Hz重规划不同；请求响应超时不限制整段导航时长；列表库均参与加载，但默认树只引用11项；多点RemovePassedGoals当前采用自己的base_link默认端口；BackUp调用自定义空闲方向移动后端。
- **本次验证范围：** 未启动新的导航节点、未发送导航目标、未测试行为树实际加载或恢复移动；前文Point-LIO的静止与短距离记录不能作为本块通过实车导航验证的证据。
- **实物待核实：** 有效map至gimbal_yaw_fake的TF、odometry速度与时间戳、单点／多点实际使用的树、RemovePassedGoals的参考系和中间点移除、请求响应耗时、恢复方向及速度链是否符合实车接口。

