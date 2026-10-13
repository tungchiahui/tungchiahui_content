---
title: "smoother_server（全局路径平滑）"
---

## smoother_server（全局路径平滑）

### 实车先看：节点已经加载，默认导航是否真的调用平滑

**当前navigation_launch会启动并管理smoother_server，但两棵默认行为树都没有SmoothPath节点。** 当前正常路径链是ComputePath→FollowPath，所以这组平滑参数虽在插件配置时读取，默认导航不会因此自动平滑Theta*路径。

```text
当前默认链路：
    ComputePath → planner_server／Theta* → path → FollowPath／PID

后续若明确接入路径平滑：
    ComputePath → 原路径 → SmoothPath Action → smoother_server
                                              └─ simple_smoother
                                                  → 返回平滑路径与完成标志
    FollowPath必须使用返回的平滑路径，才会改变实际跟踪输入

另一条速度处理链：
    controller_server输出速度 → velocity_smoother → fake_vel_transform
```

smoother_server处理nav_msgs/Path中的位置，不直接平滑cmd_vel；velocity_smoother处理速度指令，是另一节点。Theta*自身的线性补点也不等于已经调用本平滑器。[规划与路径补点](/wiki/2026-09-20-dao-hang/0500-1600-planner-server)、[当前控制链](/wiki/2026-09-20-dao-hang/0500-1100-controller-server)

| 实车上看到的现象 | 先核对什么 | 当前配置怎样理解 |
|---|---|---|
| 改平滑参数，默认导航路径没变 | 实际使用的BT、是否有SmoothPath请求 | 当前两棵默认树未调用；加载BT库／启动服务器不等于执行平滑 |
| 以为tolerance=1e-10要求机器人到达误差极小 | 收敛量的实际计算 | 比较一轮中所有中间点x、y更新绝对值的总和，不是终点定位误差或车体精度 |
| 以为max_its=1000保证一定平滑1000次 | 提前收敛、时间、碰撞及计数分支 | 是单次smoothImpl的计数上限；当前第1000计数会在遍历前退出，最多999轮完整遍历 |
| 以为do_refinement增加点数或提高地图分辨率 | 递归细化实现 | 重复平滑已有点，不增加路径点；内部最多额外调用4次，计数共享本次请求 |
| 10个点的短路径没有变 | 分段后点数条件 | 只平滑每段至少11个点的路径；较短段直接保留 |
| Action返回成功，以为所有平滑都充分收敛 | was_completed、日志与实际路径 | 服务器可返回Action成功但was_completed=False；细化递归也有返回值不向外传播的边界 |
| 路径看着更圆，却变得贴墙或穿过未知区 | 平滑目标、地图点检查与后置轮廓检查 | SimpleSmoother不沿软代价梯度避障；内部放行未知255；后置check_for_collisions另由请求决定 |

本次nav2_smoother、nav2_behavior_tree及nav2_costmap_2d均为**1.1.20**。SimpleSmoother、分段工具和服务器安装头文件与该版本一致；安装XML注册nav2_smoother::SimpleSmoother，相关库存在。以下按该实现解释，不能直接套后续版本新增的细化次数或约束参数。[SimpleSmoother实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/simple_smoother.cpp)、[服务器实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/nav2_smoother.cpp)

### 参数

下面保留发送的完整6项原参数。当前源码／安装副本的reality段一致；simulation段只有use_sim_time=True不同。

```yaml
smoother_server:
  ros__parameters:
    use_sim_time: False
    smoother_plugins: ["simple_smoother"]
    simple_smoother:
      plugin: "nav2_smoother::SimpleSmoother"
      tolerance: 1.0e-10
      max_its: 1000
      do_refinement: True
```

“当前运行不使用”在本页表示：**插件已配置，但当前默认BT未发起平滑计算。** 数值和开关的调节效果描述的是后续实际调用SmoothPath时的条件，不表示改完本块就会影响当前导航。

#### ros__parameters（时钟与平滑器ID）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真True | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择服务器ROS时钟，供结果smoothing_duration和TF使用 | 不适用 | 不适用 | navigation_launch重写已有键；SimpleSmoother内部限时使用steady_clock，不随/clock暂停而暂停 |
| `smoother_plugins` | `["simple_smoother"]` | 不适用，插件ID列表 | **通常保持当前值，不作为优先调试项** | 加载下方simple_smoother配置分组 | 不适用 | 不适用 | ID用于SmoothPath请求的smoother_id，不是算法类名；新增ID不会自动串联多个平滑器，也不会自动修改BT |

##### 参数详解

###### smoother_plugins（加载与实际调用分开核对）

当前configure读取列表，按simple_smoother.plugin加载类，再把此ID登记到服务器。启用SmoothPath后，可明确请求smoother_id="simple_smoother"；只有一个插件且请求ID为空时，1.1.20会警告并使用唯一插件，显式错误非空ID则失败。[加载与ID选择](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/nav2_smoother.cpp#L125)

navigation_launch把smoother_server列入生命周期管理，并提供独立进程／组件两种启动方式；bt_navigator也配置了nav2_smooth_path_action_bt_node库。这些说明服务器和BT节点可用，**不能证明当前树调用了它**。本次已检查两份默认XML及安装副本，SmoothPath数量均为0。

如果以后选用其他XML或外部客户端，是否平滑以实际请求为准。本页没有修改默认行为树；原路径只有被请求平滑、返回结果又接到FollowPath后，才改变执行输入。[当前BT调度](/wiki/2026-09-20-dao-hang/0500-1000-bt-navigator)

#### simple_smoother（迭代停止与重复细化）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `nav2_smoother::SimpleSmoother` | 不适用，类名 | **通常保持当前值，不作为优先调试项** | 选择基于原点位置和三点邻域的二维平滑实现 | 不适用 | 不适用 | 当前注册名正确；配置时读取参数，真正计算需要SmoothPath调用；不包含车辆转弯半径／速度约束优化 |
| `tolerance` | `1.0e-10` | m，单轮中间点x、y更新绝对值总和的门槛 | **当前运行不使用这个参数，暂不调整** | 当一轮总更新量小于它时，结束当前平滑遍历 | 通常更早达到停止条件、迭代减少，未充分收敛时更易提前停止 | 收敛要求更严、迭代可能增多，更易碰到时间／次数上限 | 实际调用且该段≥11点时才计算；不是xy_goal_tolerance；应为有限正值，门槛变小不保证路径更安全 |
| `max_its` | `1000` | 单次smoothImpl的迭代计数上限 | **当前运行不使用这个参数，暂不调整** | 限制一轮平滑调用继续迭代的次数 | 可给未收敛段更多遍历机会，耗时可能增加；仍受时间／碰撞退出影响 | 更早按未完成返回，结果可能只做了部分平滑 | 当前计数先加1再检查≥上限，1000最多999轮完整遍历；细化调用重新计数，不是整次请求总次数 |
| `do_refinement` | `True` | 不适用，布尔 | **当前运行不使用这个参数，暂不调整** | 当前段正常收敛后，允许在结果上重复平滑 | 不适用 | 不适用 | True最多额外调用4次smoothImpl，计数共享本次请求；False不进入细化递归；不增加点数／提高地图分辨率，也不保证全部细化完成 |

##### 参数详解

###### SimpleSmoother实际修改什么（原位置约束与邻居拉平）

对符合点数条件的路径段，算法保留段首、段尾的位置，只更新中间点x和y，z不参与迭代。当前未显式写w_data与w_smooth，使用源码默认0.2与0.3；每个坐标按下面规则更新：

```text
新坐标 ← 当前新坐标
       + w_data ×（本次smoothImpl输入的原坐标 − 当前新坐标）
       + w_smooth ×（前一个新坐标 + 后一个新坐标 − 2 × 当前新坐标）
```

w_data把点拉回本轮输入位置，w_smooth把相邻点的局部折线拉平。遍历过程中使用new_path的邻居，其中一部分已经在本轮更新，不是所有点只依据同一份上一轮快照同时更新。

这里的data不是障碍距离，平滑项也不是直接求车辆曲率或转弯半径；地图主要用来拒绝某些候选点，不按1～252软代价给平滑方向连续打分。因此它可能削弱Theta*原先按w_traversal_cost选择的离障碍取舍；是否更适合跟踪，需要看结果路径和实际车体空间。[更新公式与点检查](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/simple_smoother.cpp#L143)

###### tolerance（所有中间坐标更新量之和，不是位置精度）

每轮先将change清零，然后累加各中间点两次坐标更新的abs(更新后−更新前)。循环条件为change≥tolerance，所以必须**严格小于**1e-10才因正常收敛退出。

例如10个中间点的x、y各改变1e-6 m，合计change=20×1e-6=2e-5 m，远大于当前1e-10，即使单个点变化已经很小仍继续。这只是公式示例，不是本次平滑输出。

相同单点变化下，长段的change通常更大；不能把它理解成每个点都允许1e-10 m误差，也不能把它和控制器0.15 m的终点到达容差比较为同一精度要求。这个门槛检测的是迭代变化是否足够小，不是原路径偏离量、真实定位误差、离墙距离或平滑后的最大曲率。

调大可能减少遍历次数，调小可能增加次数；结果形状还受两项权重、输入点密度、细化及退出原因共同影响。当前1e-10很严格，但默认BT未调用，没有实测耗时问题，不能仅凭数值小就宣布当前导航必须改它。[change统计与停止条件](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/simple_smoother.cpp#L112)

###### max_its（次数上限、时间限制与部分结果）

smoothImpl每轮先its+=1，再检查its≥max_its。当前1000意味着最多完整执行1～999轮；若第999轮后仍未收敛，下一轮计数到1000便返回False。若已经收敛，则不会为了凑次数继续运行。

超过次数、超过本次时间或候选点进入禁止格时，函数会把path恢复为last_path，即上一次完整可接受遍历的路径，并更新近似朝向，返回False；它不总是恢复到最初未平滑路径。第一次遍历完成前失败时，last_path才仍是本轮原输入。

因此“未完成”可能带有部分平滑结果；增大max_its不保证得到更多有效结果，也不保证克服时间或碰撞退出。时间来自SmoothPath请求的max_smoothing_duration，与这个计数上限各自作用。[次数／时间退出](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/simple_smoother.cpp#L118)

各长段的成功标志用短路&&累计：一个长段返回False后，后续长段不会再调用smoothImpl，而会保留对应输入。was_completed=False不能解释为每段都已尝试迭代、只是最后没有达到同一收敛门槛。

###### do_refinement（最多四次额外调用，不是一次额外遍历）

第一次smoothImpl正常收敛后，True允许递归调用smoothImpl(new_path,...)。每次递归有自己的its计数和输入路径；前一次平滑结果成为下一次“原位置”约束，所以细化可以继续改变形状，并不一直把所有点拉向最初Theta*路径。

refinement_ctr在每次smooth请求开始时置0，最多增加到4；一个正常收敛长段可以经历基础调用加4次细化，合计最多5次smoothImpl。**计数在整次请求内共享**，不是每个分段都保证另外做4次；若前面长段消耗了计数，后面段就没有同样的细化次数。

每次调用都重新计算自己的迭代计数，因此max_its=1000不是整次请求最多1000遍。细化也不插点、不改变点数量；此处的“refinement”主要表示重新平滑已有位置。[细化递归](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/simple_smoother.cpp#L180)

当前还有两个使用边界需要明确：递归传入同一个max_time值并重新开始内部计时，没有按上一层已耗时间持续扣减；递归调用的bool返回值也没有传回上一层。因此True时总时长可能超出表面单次预算，外层返回True也不能证明每次细化均完整收敛。这里只记录代码路径，未测试其实际超时幅度。

##### 分段、短路径与朝向

路径先按相邻位置向量点积为负的折返，以及“位置几乎不动而yaw变化”的原地转向条件分段；只有end−start>9才调用平滑，相当于包含首尾至少11个点。10点及更短段保持输入位置和朝向，不能通过调tolerance或max_its让当前这条短段条件消失。[分段工具](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/include/nav2_smoother/smoother_utils.hpp#L45)、[长段条件](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/simple_smoother.cpp#L74)

11点不等于固定米数：若恰好等间距5 cm、没有切段，首尾相隔约0.5 m只是一个直线例子；地图格距、Theta*插值和分段规则都会改变实际长度。平滑保留各段端点位置，因此折返分界点不被跨段一起抹平。

对参与平滑的长段，输出工具按相邻点方向更新近似yaw，并用输入第1→第2点方向与第1点已有yaw判断是否倒车；最后一点的输入朝向保留，重合点还可能跳过更新。不是每个路径点的四元数都无条件被修正。

当前Theta*中间点没有逐点生成有效沿路径朝向，而虚拟gimbal_yaw_fake也不等于真实底盘朝向；若后续调用平滑，要核对倒车推断和轮廓方向。当前PID关闭旋转且yaw容差较大，输出yaw变了不等于底盘自动跟着转向。[近似朝向更新](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/include/nav2_smoother/smoother_utils.hpp#L89)、[规划器朝向条件](/wiki/2026-09-20-dao-hang/0500-1600-planner-server)

#### 相关源码默认项（原YAML未显式配置）

##### 参数表

以下6项为两项平滑权重和服务器地图／轮廓输入。原参数块没有显式配置，本次也未向工程追加。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `simple_smoother.w_data` | `0.2`，源码默认 | 无量纲，位置保持项权重 | **当前运行不使用这个参数，暂不调整** | 将更新点拉向本次smoothImpl输入位置 | 更新时回拉作用更强，通常更偏向保持本轮形状；过大可能影响迭代稳定性 | 更弱地保留本轮形状，邻居项相对影响更大 | 不是障碍距离权重；细化时输入参考会更新；不能用两个独立“调大”结论保证整组权重稳定收敛 |
| `simple_smoother.w_smooth` | `0.3`，源码默认 | 无量纲，三点邻域平滑权重 | **当前运行不使用这个参数，暂不调整** | 用邻居位置减轻局部折线变化 | 更新时拉平作用更强，可能偏离原路线更多；过大可能不稳定 | 更新时拉平作用更弱，可能保留更多折线 | 不直接约束最小转弯半径／速度／加速度；需和w_data、次数及时间一起核对 |
| `costmap_topic` | `global_costmap/costmap_raw`，服务器默认 | 不适用，话题名称 | **上实物前必须核对并正确配置** | 提供平滑点检查及可选后置检查的地图 | 不适用 | 不适用 | 接收nav2_msgs/Costmap，经机器人namespace解析；不是OccupancyGrid的costmap话题，也不直接读planner_server持有的内存图 |
| `footprint_topic` | `global_costmap/published_footprint`，服务器默认 | 不适用，话题名称 | **上实物前必须核对并正确配置** | 为服务层后置碰撞检查提供发布轮廓 | 不适用 | 不适用 | SimpleSmoother.configure本身忽略footprint参数，只用中心点检查；请求check_for_collisions=True时另由服务器使用 |
| `robot_base_frame` | `base_link`，服务器默认 | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 供FootprintSubscriber把发布轮廓转回机体系 | 不适用 | 不适用 | 不自动继承costmap的gimbal_yaw_fake；后置检查启用前需核对参考原点、姿态和轮廓换系，不是本插件的路径优化坐标系 |
| `transform_tolerance` | `0.1`，服务器默认 | s，轮廓换系的TF等待设置 | **通常保持当前值，不作为优先调试项** | 初始化FootprintSubscriber时传入TF查询等待 | 可等待更久，也可能延迟检查 | 缺TF时更快结束查询 | 不是允许0.1 m／rad几何误差，不自动变换输入path；与其他节点同名参数独立 |

##### 权重之外，地图怎么参与

SimpleSmoother内部对候选中间点查代价：超过MAX_NON_OBSTACLE=252且不等于NO_INFORMATION=255时退出，即**253、254拒绝，255未知放行**。它不对1～252建立避障优化梯度，也没有继承Theta*的allow_unknown参数。Theta*常规搜索拒绝252，而本平滑点检查允许252，两者条件不同。[内部候选检查](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/simple_smoother.cpp#L157)

因此不能仅凭原规划器禁止未知就保证平滑路径仍不会进入未知。内部检查只检查更新点，不能称为完整车体扫掠验证；该代码还没有检查worldToMap的bool返回值就使用格索引，输入和更新点是否始终在地图内需要核对，不能把地图外候选理解成可靠拒绝。

smoother_server从话题取得地图快照，默认使用global_costmap/costmap_raw；与planner_server直接读内存不同，地图新鲜度受全局图目标2 Hz发布及接收影响，实际发布频率仍受更新循环和负载制约。没接到地图时getCostmap会抛出“Costmap is not available”，不是通过增大迭代次数就可继续平滑。[地图订阅与快照](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_subscriber.cpp#L49)

服务器复制输入path后直接交给插件，SimpleSmoother没有把输入路径自动转换到地图坐标系。当前路径应和全局代价图一样在map中表达；仅有TF可用，也不证明输入odom路径会被自动换到map。[服务器传入路径](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/nav2_smoother.cpp#L280)

#### SmoothPath请求、结果与后置检查（不是原6项节点参数）

##### 请求与BT端口

| Action字段／BT端口 | 单位或类型 | 当前接口含义 |
|---|---|---|
| path／unsmoothed_path | nav_msgs/Path | 调用者提交待平滑路径，不会自动监听plan话题并处理每一张路径 |
| smoother_id | 插件ID字符串 | 当前可用ID为simple_smoother；BT端口默认空，唯一插件可兜底 |
| max_smoothing_duration | Action中Duration；BT端口按s输入 | 本次请求的时间预算；BT端口缺省3.0 s，直接Action客户端必须自行提供所需值，不从max_its推导 |
| check_for_collisions | 布尔 | BT端口默认False；True才执行服务层后置轮廓检查，与插件内部中心点检查不同 |
| 返回path／smoothed_path | nav_msgs/Path | 结果路径，可能是完整平滑、部分结果或短段原样返回；须接到FollowPath才用于执行 |
| smoothing_duration | Duration／BT输出秒数 | 服务器用ROS now计算，到插件返回后取值；后面的轮廓检查耗时不在该字段计时范围内 |
| was_completed | 布尔 | 插件smooth的bool结果；不是Action状态，也不能完全覆盖当前细化递归中的失败返回 |

这些端口由nav2_smooth_path_action_bt_node提供，当前树没有实例化SmoothPath，故3.0 s／False只是**该BT节点若被使用且省略端口时的默认值**，不是当前已有平滑请求。[BT端口默认](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/include/nav2_behavior_tree/plugins/action/smooth_path_action.hpp#L65)、[Action接口](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_msgs/action/SmoothPath.action)

##### 时间预算与结果状态要一起看

插件smooth以steady_clock统计整次处理的已耗时间，进入每个长段前计算剩余预算；smoothImpl再以自己的steady_clock计时，每轮遍历前检查。预算不是可以中断单轮循环的硬实时截止，也不涵盖获取地图等计时前工作和服务器后置检查的所有成本；细化递归重复使用预算的问题见前文。

服务器把插件返回值放入was_completed。即使False，只要没有异常且可选后置检查通过，它仍可以将Action设为SUCCEEDED，返回部分路径；日志里的“未在指定时间完成”还可能对应次数或内部碰撞退出，不能只凭这句日志认定唯一原因是超时。[服务层结果处理](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/nav2_smoother.cpp#L280)

BT插件on_success会写smoothed_path、smoothing_duration、was_completed并返回SUCCESS，**不会自动因为was_completed=False转为FAILURE**。后续若要求“必须完成才跟踪”，需要在调用流程中明确处理完成标志；当前未接入平滑，本文没有替用户增加这套流程。[BT成功回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_behavior_tree/plugins/action/smooth_path_action.cpp#L48)

##### check_for_collisions（服务层另外检查轮廓）

True时，服务器对结果中的每个离散姿态调用CostmapTopicCollisionChecker，使用地图和发布轮廓；取到的轮廓先转到robot_base_frame，再放到各路径姿态上检查。这与SimpleSmoother迭代里的中心点代价检查是两条路径，False不会关闭插件内部点检查。

当前服务器默认robot_base_frame=base_link，而全局轮廓参考gimbal_yaw_fake。启用后要核对二者的实际TF、原点偏移、圆／多边形轮廓及路径yaw语义，不能只复制两个话题就认定几何已经一致。[轮廓换系](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/footprint_subscriber.cpp#L90)、[服务层碰撞检查](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_smoother/src/nav2_smoother.cpp#L294)

该检查器按轮廓评分≥254拒绝，并会在缺图／缺轮廓、地图外等异常条件返回False；仍是对离散路径姿态的检查，不能当成包含点间运动、定位误差和执行偏差的完整实车验证。服务器在后置检查前已经发布plan_smoothed，因此**看见该话题有路径，也不能单独证明Action最终成功或后置检查通过**。[检查器与阈值](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_topic_collision_checker.cpp#L49)

#### 动态参数、启动与验证范围

##### 运行中修改的实际范围

当前SimpleSmoother没有为tolerance、max_its、do_refinement、w_data、w_smooth设置动态参数回调；服务器也没有动态重建平滑器或地图／轮廓订阅的路径。ROS参数设置成功不表示这些已读取成员更新，应修改配置后重新配置／重启，并核对加载值和实际请求。use_sim_time属于ROS时钟机制，应与整套节点时间域一起处理。

| 当前流程 | 本块状态与作用 |
|---|---|
| reality／simulation导航 | 两份配置仅时钟不同；navigation_launch启动服务器并交由lifecycle_manager_navigation管理 |
| 独立进程／组件 | 两种分支均加载本配置；组件类为nav2_smoother::SmootherServer |
| 两棵默认BT | 没有SmoothPath，实际ComputePath结果直接进入FollowPath |
| 后续其他BT／外部客户端 | 只有实际发起smooth_path请求、使用返回路径时才影响跟踪 |
| 默认接口 | smooth_path Action、plan_smoothed话题；机器人namespace存在时位于其下，不额外自动加smoother_server一层 |

例如namespace为/red_standard_robot1，节点名为/red_standard_robot1/smoother_server，接口通常为/red_standard_robot1/smooth_path和/red_standard_robot1/plan_smoothed；地图默认订阅/red_standard_robot1/global_costmap/costmap_raw。名称与输入参考系需要各自核对。

##### 本次已核对与后续实物检查

- **本次已核对：** nav2_smoother 1.1.20、注册XML与库；SimpleSmoother、分段工具及服务器安装头文件与固定版本一致；源码／安装副本两套参数一致且仅时钟不同；navigation_launch与默认XML安装副本一致，两棵树SmoothPath数量均为0。
- **源码能确认：** 6项原配置的读取和使用条件；总更新量收敛门槛；每调用的次数及提前退出；≥11点分段条件；端点位置保留、已有点重复细化、最多4次额外调用及共享计数；递归预算／返回值边界；默认权重、地图快照、内部点检查与服务后置检查的区别；Action状态与was_completed分离。
- **本次验证范围：** 没有发起SmoothPath、修改运行参数或默认BT，也没有实测平滑耗时、路径形状、碰撞或底盘跟踪效果；迭代更新量及点数长度示例为源码计算。
- **实物待核实：** 若后续接入，检查实际请求与FollowPath输入、地图新鲜度／参考系、轮廓TF和yaw语义、原路径到平滑路径的通过空间、部分结果处理、总规划链延迟及跟踪表现；原工程参数与代码保持原样。
