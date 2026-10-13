---
title: "waypoint_follower（逐点导航与到点等待）"
---

## waypoint_follower（逐点导航与到点等待）

### 实车先看：是否调用逐点导航、是否需要到点等待

**本节点收到FollowWaypoints航点列表后，逐个调用NavigateToPose，到点后执行一次任务插件，再前往下一个点。当前插件是WaitAtWaypoint，每个导航成功的点等待200 ms，即0.2 s，最后一个点也等待。** 它不直接跟踪planner_server输出的稠密路径，也不直接计算底盘速度。

```text
客户端发送FollowWaypoints：poses=[A, B, C]
    → waypoint_follower
        → NavigateToPose(A) → bt_navigator → 规划／控制／必要时恢复
            ├─ 导航成功 → WaitAtWaypoint：等待200 ms
            └─ 导航失败 → 记录A的索引；当前stop_on_failure=false，继续
        → NavigateToPose(B) → 同样处理
        → NavigateToPose(C) → 同样处理，成功时最后一个点也等待
        → 返回Action结果及missed_waypoints
```

当前PB的navigation_launch在独立进程和组件分支都会加载waypoint_follower，并把它交给生命周期管理器；**加载不等于已经收到航点任务。** 普通NavigateToPose和NavigateThroughPoses不经过这个航点列表执行器，也不会因为enabled=True自动在途经点暂停。[当前BT入口](/wiki/2026-09-20-dao-hang/0500-1000-bt-navigator)

本次在当前PB源码中没有找到发送FollowWaypoints的客户端；Fork的另一套目录ysu2026_rm_sentry_nav_cod/src/waypoint_editor中则已有waypoint_to_nav2桥接代码，通过start_waypoint_following服务发送该Action。当前PB启动文件没有加载该桥接；是否另行启动或由外部决策客户端调用，需以实际请求为准，不能直接判断这组参数永远不用。[已有桥接客户端](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav_cod/src/waypoint_editor/src/waypoint_to_nav2.cpp:30)

| 实车上看到的现象 | 先核对什么 | 当前配置怎样理解 |
|---|---|---|
| 普通单点／连续多点导航没有暂停 | 实际请求是哪个Action | 只有FollowWaypoints成功到点后调用本任务插件 |
| 以为200是等待200秒 | 插件转换出的时间单位 | 源码使用milliseconds，当前为0.2秒 |
| 调loop_rate，停留时间没有相应变短 | 状态检查循环和插件等待各自的参数 | 20 Hz是任务处理循环；200 ms由插件单独等待 |
| 某个点没到，却继续下一个点 | stop_on_failure与导航结果 | 当前False记录导航失败索引并跳过，不自动再次尝试该点 |
| Action成功，但有点没有到达 | 返回的missed_waypoints | False允许列表遍历完后返回成功；成功不等于每点到达 |
| 开了等待，底盘仍在自旋 | 到点判定、零速链和cmd_spin | 插件不发零速度、不检查实际停车；附加自旋不由它清除 |
| 仿真暂停后航点等待不结束 | use_sim_time与/clock是否前进 | 插件按节点ROS时钟等待，20 Hz墙钟循环不能让睡眠提前结束 |
| 等待时间很长时，取消响应也变慢 | 同步processAtWaypoint调用 | 插件睡眠期间不检查Action取消；取消在主循环恢复后处理 |

本次安装的nav2_waypoint_follower为**1.1.20**，rclcpp为**16.0.21**。服务器／WaitAtWaypoint安装头文件与Nav2对应版本一致；安装插件XML注册nav2_waypoint_follower::WaitAtWaypoint，库存在；FollowWaypoints安装接口与该版本一致。下文按这套接口解释。[服务器源码](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/src/waypoint_follower.cpp)、[等待插件源码](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/plugins/wait_at_waypoint.cpp)

### 参数

下面原样保留发送的7项参数。源码和安装副本的reality段一致；simulation段只有use_sim_time=True不同。本次没有更改工程YAML，也没有发送导航或航点请求。

```yaml
waypoint_follower:
  ros__parameters:
    use_sim_time: False
    loop_rate: 20
    stop_on_failure: false
    waypoint_task_executor_plugin: "wait_at_waypoint"
    wait_at_waypoint:
      plugin: "nav2_waypoint_follower::WaitAtWaypoint"
      enabled: True
      waypoint_pause_duration: 200
```

#### ros__parameters（时钟、任务循环与失败策略）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真True | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择节点ROS时钟；WaitAtWaypoint使用该时钟等待 | 不适用 | 不适用 | launch会重写已有键；False通常按系统时间，True且ROS时间激活时按/clock；主任务循环另用WallRate |
| `loop_rate` | `20`；源码默认20 | Hz，FollowWaypoints任务处理循环目标频率 | **通常保持当前值，不作为优先调试项** | 控制反馈、导航结果处理以及取消／抢占检查的循环节拍 | 无阻塞时检查更密，调度和反馈开销增加 | 无阻塞时检查更稀，任务切换／取消响应可能更慢 | 正整数，当前名义50 ms／轮；不是controller_frequency，也不改变200 ms等待；动态改值不会重建已进行请求的WallRate |
| `stop_on_failure` | `false`；源码默认True | 不适用，布尔 | **根据实际功能需求开启或设置** | 决定单点导航或到点任务失败时是否终止航点列表 | 不适用 | 不适用 | False记录导航失败并继续；True遇失败终止，尚未尝试的后续点不因此列入missed；不是急停／障碍检测开关 |
| `waypoint_task_executor_plugin` | `"wait_at_waypoint"`；源码默认同值 | 不适用，任务插件ID | **通常保持当前值，不作为优先调试项** | 选择到点后执行的任务插件配置分组 | 不适用 | 不适用 | 是分组ID，不是类名；读取wait_at_waypoint.plugin，只加载一个执行器，不表示串联多个任务 |

##### 参数详解

###### loop_rate（等待导航结果的循环，不是移动控制频率）

收到非空列表后，服务器创建WallRate(loop_rate)，每轮处理取消／新目标、发送下一点、发布current_waypoint、处理导航状态，最后处理NavigateToPose客户端回调并按节拍等待。20 Hz对应名义50 ms／轮，但回调、同步任务插件或取消等待都可能延长这一轮，不能把50 ms写成取消响应的硬上限。[任务循环](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/src/waypoint_follower.cpp#L161)

每个点只在切换到新目标时发送一次NavigateToPose，不是每秒20次重发。规划频率由BT的RateController等决定，控制速度仍由controller_server产生；改loop_rate不会把路径跟踪频率改成同样的值，也不会把机器人线速度直接提高。[控制频率与速度](/wiki/2026-09-20-dao-hang/0500-1100-controller-server)

当前20 Hz与controller_server的20 Hz数值相同，仍是两个独立循环。按数值计算，200 ms相当于四个50 ms间隔，但WaitAtWaypoint是一次同步睡眠，期间不会因此执行四轮航点检查。

###### stop_on_failure（继续任务与结果完整性）

本版将NavigateToPose的ABORTED、CANCELED或目标被拒绝视为导航失败。导航失败先把当前索引加入failed_ids；True立即终止当前FollowWaypoints并返回已记录索引，False则推进到下一个点，不执行失败点的WaitAtWaypoint。[导航失败及推进分支](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/src/waypoint_follower.cpp#L199)

索引从0开始。例如[A, B, C]中B导航失败、其余成功，当前False会尝试C，最终可返回Action成功、missed_waypoints=[1]。即使所有点导航失败，False也可在遍历完后返回成功并列出全部失败索引；客户端必须一起检查返回列表，不能仅看SUCCEEDED就报告每点到达。

| 情况 | stop_on_failure=True | 当前stop_on_failure=False |
|---|---|---|
| 某点NavigateToPose失败 | 记录该索引，终止列表 | 记录该索引，继续下一点 |
| 某点导航成功，任务插件返回False | 记录该索引，终止列表 | 继续下一点；1.1.20这一分支没有记录该任务失败索引 |
| 列表全部处理完 | 无失败才走到正常成功结束 | 即使有导航失败，也成功结束并返回missed_waypoints |

任务插件失败的记录条件与导航失败不同：只有“任务返回False且stop_on_failure=True”才把该点加入missed。当前WaitAtWaypoint正常返回True，但若以后替换为其他任务插件，False策略下不能认为missed_waypoints完整涵盖所有任务执行失败。[任务执行分支](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/src/waypoint_follower.cpp#L219)

Fork另一目录的waypoint_to_nav2桥接在SUCCEEDED分支只打印“All waypoints reached successfully!”，没有检查missed_waypoints。如果以后使用它，当前False下该日志可能与实际漏点不符；需要核对客户端结果处理。本次只记录这个适配事项，没有修改该桥接。[现有结果处理](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav_cod/src/waypoint_editor/src/waypoint_to_nav2.cpp:165)

True终止的是航点任务流程，不等于由本节点直接发急停。单点导航自己的恢复会先在NavigateToPose中进行；这里只依据它最后的结果决定是否继续下一点，不通过改False增加同一点的恢复次数。

###### waypoint_task_executor_plugin（ID、类名与到点条件）

服务器按ID寻找对应的plugin键，再创建类并调用initialize(node, ID)。所以当前ID为wait_at_waypoint时，需要与下面分组同名；如果只改ID而保留下方旧分组，新ID的插件配置不会自动从旧分组继承。[加载任务执行器](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/src/waypoint_follower.cpp#L74)

本节点以NavigateToPose的成功结果作为执行任务的条件，没有在此处再次检查真实车速或严格位置误差。当前general_goal_checker的xy容差0.15 m、yaw容差6.28 rad，解释见控制器页；因此“成功后等待200 ms”不是另加0.2秒精确对准，也不能替代到点判定和实际停车验证。[到达判定](/wiki/2026-09-20-dao-hang/0500-1100-controller-server)

#### wait_at_waypoint（到点后的等待插件）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `nav2_waypoint_follower::WaitAtWaypoint` | 不适用，任务插件类名 | **通常保持当前值，不作为优先调试项** | 到达航点后按节点时钟睡眠，再返回任务成功 | 不适用 | 不适用 | 安装XML注册正确；不是behavior_server的nav2_behaviors/Wait，也不是路径跟踪控制器 |
| `enabled` | `True`；源码默认True | 不适用，布尔 | **根据实际功能需求开启或设置** | 决定WaitAtWaypoint是否执行到点等待 | 不适用 | 不适用 | False直接返回True，导航列表照常继续；True还要求初始化时pause_duration不为0；不直接启用FollowWaypoints任务 |
| `waypoint_pause_duration` | `200`；源码默认0 | ms，成功到点后的ROS时钟等待时长 | **实物运行时重点观察，有问题再针对性调整** | 每个导航成功的航点同步等待指定时长 | 到点停留预算及整次任务耗时增加；阻塞期间取消／切换请求的处理可能更迟 | 停留预算减少；初始化为0会把插件内部enabled置False | 当前200 ms=0.2 s；包括最后一个成功点；不对导航失败点等待；正整数用于有效等待，不是目标精度或整个任务超时 |

##### 参数详解

###### enabled与waypoint_pause_duration（两个条件一起决定是否等）

initialize读取wait_at_waypoint.enabled和wait_at_waypoint.waypoint_pause_duration；如果时长为0，即使YAML写enabled=True，也把内部is_enabled设为False。processAtWaypoint遇内部禁用直接返回True，表示本插件任务成功，不会终止或取消导航列表。[初始化与零时长处理](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/plugins/wait_at_waypoint.cpp#L43)

当前True和200会进入睡眠分支。每个点使用同一时长，插件没有按航点索引设置不同等待时间，也没有根据当前位置、偏差或速度自动延长；需要其他到点任务时应使用相应执行器实现。

同一列表有N个导航成功的点时，名义等待预算为N×0.2 s，包含最后一个点。例如10个点全部成功，合计名义等待2 s；这是参数换算，不含路途、恢复、回调调度和任务切换耗时，也不是本次实测。

###### 200的单位与仿真时间

插件把整数200转成std::chrono::milliseconds，再调用节点clock的sleep_for。因此单位明确为毫秒；5000才表示5秒，不要把当前200解释为200秒，也不要为了表达0.2秒把此整数参数直接写成浮点0.2。[等待调用](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/plugins/wait_at_waypoint.cpp#L67)

这里虽传入chrono时长，等待仍按节点时钟工作。本次rclcpp 16.0.21的sleep_for实际等待now()+时长；use_sim_time=False时ROS时钟通常使用系统时间，True且ROS时间激活时则等待/clock前进到目标时刻。仿真暂停或时钟不再前进，200 ms不保证在200 ms墙钟时间后返回。[对应rclcpp等待实现](https://github.com/ros2/rclcpp/blob/16.0.21/rclcpp/src/rclcpp/clock.cpp#L121)

如仿真时间均匀以真实时间的0.5倍前进，0.2 s仿真等待名义对应0.4 s真实时间；这是时间比例示例，未验证当前仿真实时倍率。loop_rate使用WallRate，也不能改变同步ROS时钟睡眠的结束条件。

###### 等待不会检查停车，也不会即时处理取消

WaitAtWaypoint没有cmd_vel发布器，不检查车轮速度、不持续锁定位置，也不清除fake_vel_transform的cmd_spin。它只推迟下一个目标的发送；实际停止由NavigateToPose完成处理和速度下游保证。当前附加自旋可能继续，因此等待不代表底盘所有速度分量都为零。[速度转换与附加自旋](/wiki/2026-09-20-dao-hang/0500-0600-fake-vel-transform)

processAtWaypoint在航点主循环内同步调用，sleep_for期间不检查FollowWaypoints取消／抢占。普通取消不会直接唤醒这个睡眠；请求要等插件返回、循环再次到达检查分支后处理。加大loop_rate无法让阻塞中的检查更频繁，延长暂停时长则可能延长这段响应等待。

源码忽略sleep_for的bool返回值，之后仍返回True。若ROS关闭或时间源切换使睡眠提前返回False，这个True也不证明已完整等待200 ms；本次没有做时钟切换或取消实验。[睡眠返回约定](https://github.com/ros2/rclcpp/blob/16.0.21/rclcpp/include/rclcpp/clock.hpp#L116)

#### Action接口、启动条件与动态参数

##### FollowWaypoints和NavigateThroughPoses的区别

| 接口 | 当前执行方式 | 本页到点等待是否参与 |
|---|---|---|
| NavigateToPose | bt_navigator执行一个目标位姿 | 普通直接调用不经过航点执行器；FollowWaypoints内部调用成功后才执行插件 |
| FollowWaypoints | waypoint_follower依次发多个独立NavigateToPose，每点成功后调用任务插件 | 当前每个成功点等待200 ms |
| NavigateThroughPoses | bt_navigator运行多位姿行为树，当前ComputePathThroughPoses→FollowPath | 不经过本任务插件，不自动使用200 ms；是否连续通过还受实际BT与控制条件影响 |

本版FollowWaypoints请求只有PoseStamped[] poses，反馈current_waypoint为从0开始的当前点索引，结果missed_waypoints为失败索引列表；**没有循环次数、起始索引或整次任务时限字段**。不能仅启动该节点就自动重复巡逻；需要客户端再次发请求或由上层任务逻辑组织。[1.1.20 Action定义](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_msgs/action/FollowWaypoints.action)

服务器逐点只填写NavigateToPose的pose，behavior_tree保持空字符串，正常走bt_navigator的默认单点XML；这里不是改为执行默认多点XML。航点请求应携带正确frame_id、位置和有效四元数，当前工程通常按map表达，命名空间也须与follow_waypoints／navigate_to_pose服务端一致。[逐点请求构造](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/src/waypoint_follower.cpp#L185)

空poses列表在本版直接返回成功，没有实际导航或到点任务。正常非空请求每次从索引0开始；导航失败跳过后也不会自动回头补点。取消处理会向其NavigateToPose客户端调用async_cancel_all_goals，并等待取消响应，不是本节点直接发布零速度。[取消处理](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/src/waypoint_follower.cpp#L166)

##### 动态参数（服务器有回调，插件值只在初始化读取）

1.1.20服务器在生命周期activate时安装动态参数回调，deactivate时移除。本页需要区分以下更新条件：[动态回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_waypoint_follower/src/waypoint_follower.cpp#L304)

| 参数 | 实际更新路径 | 使用时怎样核对 |
|---|---|---|
| loop_rate | Active回调接受整数并更新成员；当前请求的WallRate已在开始时创建 | 新值用于之后新建的请求循环；正在进行的循环不会因此重建 |
| stop_on_failure | Active回调接受布尔并更新成员；失败分支读取该成员 | 可改变后续失败处理策略，不是撤回已经跳过／结束的点 |
| waypoint_task_executor_plugin、wait_at_waypoint.plugin | 当前回调不重新加载类或更换执行器 | 需重新配置生命周期／重启，单看参数值改变不能证明已切换 |
| wait_at_waypoint.enabled、waypoint_pause_duration | 插件initialize时读入成员，没有动态更新回调 | 需重新配置／重启；运行中改200不保证下一点就采用新时长 |
| use_sim_time | 由rclcpp时间源机制处理，不属于上述两个服务器成员的回调分支 | 仍须核对launch最终值与时钟状态，任务执行中切换还会影响sleep_for |

服务器回调会返回参数设置成功，但只处理loop_rate和stop_on_failure的上述类型；其他插件参数即使ROS层显示新值，也不能据此证明插件缓存已变化。loop_rate和暂停时长在本版按整数读取，数值类型也需保持正确。

##### 本次核对与验证范围

- **已核对：** 安装版本、服务器／插件头文件、注册名和库、FollowWaypoints接口；源码与安装副本的两套参数和navigation_launch；当前PB的客户端检索及另一目录已有桥接。
- **源码能确认：** 七项参数的读取条件；200 ms与最后一点等待；导航失败继续及任务失败记录差异；WallRate与ROS时钟睡眠；取消的阻塞边界和动态更新范围。
- **实物使用时核对：** 实际Action入口、航点坐标与到点判据，是否需要跳过失败点，以及客户端是否读取missed_waypoints；需要暂停时再观察有效停留时间、附加自旋和取消响应。
- **本次范围：** 只整理文档和目录，未修改导航配置、桥接、BT或插件，未发FollowWaypoints／NavigateToPose、未进行实车或仿真运动测试。文中时间与漏点例子均为源码分析或参数换算。
