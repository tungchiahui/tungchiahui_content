---
title: "planner_server（Theta*全局路径规划）"
---

## planner_server（Theta*全局路径规划）

### 实车先看：规划什么时候运行，怎样在距离与障碍代价之间取舍

**当前planner_server使用GridBased这个插件ID加载Theta*，在global_costmap中从起点搜索到目标的二维路径。** GridBased是工程指定的名称，实际算法由plugin字符串决定；它不表示此节点必定运行NavFn或普通A*。

```text
BT：ComputePathToPose／ComputePathThroughPoses，planner_id=GridBased
    → planner_server：等待全局图current、取得起点、转换起终点到全局坐标
    → GridBased：ThetaStarPlanner
        ├─ 读内存中的global_costmap，当前坐标系map
        ├─ 4／8邻居展开 + 视线检查，尝试连接更远的父节点
        ├─ 路径长度费用 + 地图经过费用 + 到目标的启发估计
        └─ 回溯折线路径，按实际地图格距补充路径点
    → Action结果path，另向plan话题发布可视化路径
    → FollowPath：当前PID控制器跟踪，输出速度
```

规划器直接使用它内部创建的global_costmap对象，不靠订阅2 Hz的costmap图片规划。它输出位置路径，不直接发底盘速度，也不规划整车爬坡能力或动态障碍的未来运动。[全局图与当前输入](/wiki/2026-09-20-dao-hang/0500-1300-global-costmap)、[路径执行](/wiki/2026-09-20-dao-hang/0500-1100-controller-server)

| 实车上看到的现象 | 先核对什么 | 当前配置怎样理解 |
|---|---|---|
| 以为10.0就是每秒重规划十次 | expected_planner_frequency与BT RateController | 10.0只给成功规划过程设置100 ms耗时警告门槛；当前默认BT规划节奏目标约3 Hz |
| 改w_heuristic_cost，路径和耗时看不出变化 | 插件是否读取该YAML键 | 1.1.20未读取该配置键；内部启发权重按min(w_euc_cost,1.0)初始化 |
| 以为w_traversal_cost=10代表离墙10 m | 经过费用公式与膨胀层 | 10是无量纲权重，惩罚经过高代价格；不是距离、速度或硬安全间隔 |
| 从8改4，以为车只能走四个方向 | 邻居展开与视线连接 | 4／8选择搜索邻居，Theta*仍会做直线连接；不限制底盘只能四向或八向运动 |
| 路径经过灰色未知区 | 全局图未知身份与allow_unknown | 当前未写allow_unknown，缺省True；常规搜索可经过255未知格 |
| 明明有空路却规划失败 | 起终点是否在图内、是否落在≥252代价格、TF与地图current | 当前Theta*没有NavFn式目标tolerance参数，不自动找目标附近的另一个空闲点 |
| 开了smoother_server，以为路径已经平滑 | 实际BT是否调用SmoothPath | 当前两棵默认树未调用SmoothPath；Theta*的线性补点不是曲率连续平滑 |

本次安装的nav2_planner、nav2_theta_star_planner及nav2_costmap_2d均为**1.1.20**。Theta*算法和插件封装安装头文件与该版本一致，插件XML确实注册nav2_theta_star_planner/ThetaStarPlanner，库文件存在；以下按这套实现说明。[插件封装](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/src/theta_star_planner.cpp)、[算法及费用定义](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/include/nav2_theta_star_planner/theta_star.hpp)

### 参数

下面保留发送的完整8项参数和原注释。当前源码／安装副本的reality段与此一致；simulation段只有use_sim_time=True不同。

```yaml
planner_server:
  ros__parameters:
    use_sim_time: False
    expected_planner_frequency: 10.0
    planner_plugins: ["GridBased"]
    GridBased:
      plugin: "nav2_theta_star_planner/ThetaStarPlanner" # In Iron and older versions, "/" was used instead of "::"
      how_many_corners: 8
      w_euc_cost: 1.0
      w_traversal_cost: 10.0
      w_heuristic_cost: 1.0
```

#### ros__parameters（服务器时钟、耗时检查与插件ID）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真True | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择服务器ROS时钟，用于请求耗时、结果时间与路径时间戳 | 不适用 | 不适用 | navigation_launch重写已有键；需与全局地图、目标和TF时间域匹配；插件另外有steady_clock耗时调试日志 |
| `expected_planner_frequency` | `10.0` | Hz，期望完成频率的倒数用于耗时警告 | **通常保持当前值，不作为优先调试项** | 将成功规划过程的告警时长设为1／10=0.1 s | 警告门槛变短、更易提示超时；不主动加快或增加规划 | 正值变小时警告门槛变长；0关闭这项时长警告 | 不是规划定时器、Action截止时间或硬中断；成功过程超门槛仍可返回路径；失败／卡在等待时不会靠它自动终止 |
| `planner_plugins` | `["GridBased"]` | 不适用，插件ID列表 | **上实物前必须核对并正确配置** | 指定服务器加载哪些规划器配置分组 | 不适用 | 不适用 | 当前BT明确请求GridBased，下方同名分组提供类和参数；新增ID不会自动并行规划或自动选最优插件 |

##### 参数详解

###### expected_planner_frequency（100 ms是警告门槛）

服务器在configure时读取正值并计算max_planner_duration=1／expected_planner_frequency。当前为0.1 s；完成有效路径、准备结果后，若测得时长超过它，打印Planner loop missed its desired rate警告，然后仍按成功返回。值为0或负值时将门槛置0，关闭这项完成后检查；需要关闭时可明确设0，不能把负值当正常频率。[门槛初始化](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_planner/src/planner_server.cpp#L133)、[完成后的警告](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_planner/src/planner_server.cpp#L501)

计时从进入规划处理、取得服务器参数锁后开始，使用节点now；包括之后等待costmap current、取得／转换位姿、插件搜索及准备／发布路径的耗时，不只是Theta*展开节点的时间。ComputePathThroughPoses检查的是本次多个目标分段规划的总耗时，不是每段各有100 ms独立限时。

插件内部另用steady_clock打印createPlan耗时，其范围和时钟与服务器planning_time不同。仿真暂停或时间跳变时，不能把ROS时间结果直接当墙钟实耗时，也不能据两种日志不同就认定算法计时错误。

| 当前时间设置 | 控制什么 | 理想时间尺度 |
|---|---|---|
| BT bt_loop_duration=10 ms | 行为树tick | 约100次／s；不等于100次规划／s |
| 默认BT RateController hz=3.0 | 发起新一轮规划子树的节奏 | 约333 ms；运行中Action、恢复等还会影响实际节奏 |
| global_costmap.update_frequency=5.0 | 内部地图合成 | 目标200 ms／轮 |
| 本块expected_planner_frequency=10.0 | 完成规划过程的警告门槛 | 100 ms；不生成周期请求 |
| controller_frequency=20.0 | 跟踪当前路径并输出控制 | 目标50 ms／轮 |

把10改20，只把警告门槛从100 ms变50 ms；不会让BT从3 Hz变20 Hz，也不会使地图从5 Hz变20 Hz。需要改变实际规划节奏时，应查看当前BT及系统耗时。[行为树调度](/wiki/2026-09-20-dao-hang/0500-1000-bt-navigator)

###### planner_plugins（ID和算法类名各有职责）

当前服务器按GridBased读取GridBased.plugin，构造ThetaStarPlanner并配置。两棵默认树的ComputePathToPose／ComputePathThroughPoses都显式写planner_id="GridBased"；若改ID，需要同步列表、分组和请求来源。单改类名而保留ID，可以更换该ID对应算法，但参数也须按新实现核对。

1.1.20只有一个插件且请求ID为空时，服务器可警告并使用唯一插件；**显式传入错误非空ID不会走这个兜底**，将得到空路径并失败。当前BT显式ID正确，不依赖兜底。[按ID选择规划器](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_planner/src/planner_server.cpp#L521)

#### GridBased（Theta*邻居与费用权重）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `nav2_theta_star_planner/ThetaStarPlanner` | 不适用，插件查找名称 | **上实物前必须核对并正确配置** | 为GridBased加载Theta*全局规划器 | 不适用 | 不适用 | 当前1.1.20安装XML注册此带斜杠名称，与原注释相符；C++类名为双冒号写法，不能仅按新版本示例随意替换配置 |
| `how_many_corners` | `8` | 邻居方向数，合法选项4或8 | **通常保持当前值，不作为优先调试项** | 决定每次搜索展开的相邻格方向 | 4→8加入四个对角邻居，每次可检查更多方向；不保证总搜索耗时更少或更多 | 8→4只展开上下左右，搜索／路径可能变化，仍可做视线捷径连接 | 不是转弯半径、路径折点数或车体角数；初始化非法值会回退8，动态回调却不做同样校验 |
| `w_euc_cost` | `1.0` | 无量纲权重；乘的是格索引距离 | **实物运行时重点观察，有问题再针对性调整** | 惩罚路径的欧氏长度，相对地图费用形成取舍 | 同样地图费用下更重长度，通常更偏向较短路线；启发相对关系也可能改变 | 地图费用相对占比提高，可能接受更长但更低代价路线 | 应使用有限正值；距离按格数而非米；初始化还决定内部启发权重，动态修改不会同步重算它 |
| `w_traversal_cost` | `10.0`；插件默认2.0 | 无量纲，经过地图的费用权重 | **实物运行时重点观察，有问题再针对性调整** | 放大经过高代价格的费用，当前比插件默认更重此项 | 更强惩罚高代价，也放大空闲格的非零基础费用；可能改变绕行取舍 | 地图经过费用下降，长度项相对更重要，路径可能更贴近高代价区域 | 0可使该费用项为0，但不放行硬禁止格；不能恢复漏掉的障碍，不等于10 m安全距离 |
| `w_heuristic_cost` | `1.0`，但此YAML键未读取 | 无量纲；当前无独立配置效果 | **当前运行不使用这个参数，暂不调整** | 原配置意图设置到目标的启发估计权重；当前插件自行计算内部权重 | 单改此YAML值无当前计算效果 | 单改此YAML值无当前计算效果 | 当前内部值=min(w_euc_cost,1.0)=1.0；内部启发项实际参与搜索，不是把整个启发项关掉 |

##### 参数详解

###### how_many_corners（4／8邻居不限制最终连线方向）

算法的moves数组先列上下左右四个邻居，再列四个对角邻居；当前8全部参与展开。同时Theta*会检查当前节点与更远父节点之间的视线，如果可连通且费用更低，就改变父关系形成更直接的折线。因此4并不保证最终路径只含水平／竖直段，8也不是只允许八种最终航向。[邻居数组](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/include/nav2_theta_star_planner/theta_star.hpp#L139)、[视线连接与邻居展开](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/src/theta_star.cpp#L84)

8每次最多检查更多邻居，但能否更快到目标、打开多少节点，取决于地图和费用；不能直接说“8一定更快”或“4一定更省总计算”。当前8是合法默认，初次测试通常保持；要比较，应在相同地图、起终点和有效分辨率下看实际路径、耗时与nodes_opened。

configure阶段只接受4或8，否则成员回退8并警告；ROS参数列表仍可能显示原非法值。动态回调直接把整数写入成员，没有同样校验，超过8还可能让展开循环越过固定数组。因此只在合法4／8间选择，不用任意整数试探；本文未运行非法配置。[初始化检查与动态分支](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/src/theta_star_planner.cpp#L39)

###### w_euc_cost与w_traversal_cost（长度与经过费用怎样相加）

算法为候选节点使用f=g+h排序。g累计已经走过的长度和地图经过费用，h估计当前位置到目标的距离；相邻一步的费用可按以下式子理解，视线捷径则按其实际检查的格子累计经过费用：

```text
长度费用 = w_euc_cost × hypot(相邻／连接端点的格索引差)
普通格的转换费用 c_scaled = 26 + 0.9 × 内部代价值c
单格经过费用 = w_traversal_cost × (c_scaled / 252)²
h = 内部w_heuristic_cost × hypot(当前格到目标格的索引差)
```

源码真正的偏置是**26**，不能照附近注释中的25代替。距离使用格索引差：横／纵一步为1，对角一步为√2；在有效格距0.05 m时，平面移动1 m相当于20格距离，而不是直接给长度费用乘1。实际全局格距由静态地图决定。[费用函数](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/include/nav2_theta_star_planner/theta_star.hpp#L204)

只为说明当前w_traversal_cost=10的公式，计算普通格和邻居扩展的未知格费用如下；不代表实车地图读数或整条路径总分：

| 内部代价值c | 当前单格经过费用，约值 | 含义 |
|---:|---:|---|
| 0 | 0.10645 | 空闲格仍有非零基础经过费用 |
| 100 | 2.11892 | 可用的较低软代价值 |
| 200 | 6.68241 | 更高软代价，经过费用明显增加 |
| 251 | 9.99207 | 常规搜索可用普通格的高端 |
| 255，且allow_unknown=True | 10.27971 | 邻居扩展中的未知格；视线分支另按253转换费用处理 |

例如横向走入c=200格，当前相邻g增量约1+6.68241；走入空闲格约1+0.10645。只把w_traversal_cost从10改20，会将上述经过费用翻倍，但横向长度项仍为1。直线捷径的检查／计费流程另有视线分支，不能把这张单格表简单乘路径点数量作为实际总费用。

在地图已经正确表达障碍和膨胀的前提下，提高经过权重通常更重绕开高代价格，提高长度权重通常更重少绕路；二者是相对取舍，不保证某次路径长度、耗时或最小离墙距离单调变化。空闲格费用也不为0，绕得更长同样有经过费用，不是只对彩色格子收费。

当前全局图cost_scaling_factor=4、inflation_radius=0.7先决定代价场，w_traversal_cost=10再决定这套代价怎样进入搜索。系数4越大，外围代价下降越快；此时单靠权重10不能说明一定离墙更远。应一起看真实轮廓、有效格距、膨胀和路径。[膨胀参数与Theta*](/wiki/2026-09-20-dao-hang/0500-1300-global-costmap)

###### w_heuristic_cost（外部配置未使用，内部h仍参与）

当前configure没有声明或读取GridBased.w_heuristic_cost，也没有在动态回调处理它；代码直接按下面赋值：

```text
内部w_heuristic_cost = w_euc_cost < 1.0 ? w_euc_cost : 1.0
```

所以当前内部值确实是1，但它来自w_euc_cost=1，不来自这一行YAML。**“数值碰巧一样”不能作为参数已生效的证据。** 算法getHCost仍读取内部成员计算h并参与f排序，不应把此配置无效误解成没有启发搜索。[内部赋值](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/src/theta_star_planner.cpp#L60)、[h计算](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/include/nav2_theta_star_planner/theta_star.hpp#L241)

| 重新配置时的w_euc_cost | YAML中w_heuristic_cost的示例值 | 真正初始化的内部启发权重 |
|---:|---:|---:|
| 1.0，当前 | 1.0 | 1.0 |
| 1.0 | 20.0 | 1.0，YAML值不参与 |
| 0.5 | 1.0 | 0.5 |
| 2.0 | 0.3 | 1.0 |

这些是源码分支计算，不是本次运行参数或路径对比。当前无需通过反复调该YAML键试验快慢；如希望独立暴露启发权重，需要改变插件实现并重新核对算法条件，不能只新增参数。本文保留原值，没有修改插件源码。

###### 动态改w_euc_cost不会重新计算内部启发权重

初始化会设置w_euc_cost和对应h权重；动态回调只更新w_euc_cost成员，没有执行上述min公式。因此同一个表面配置可能因“重启初始化”或“运行中改值”得到不同内部组合：

| 操作 | 长度权重 | 内部启发权重 |
|---|---:|---:|
| 以当前1.0启动 | 1.0 | 1.0 |
| 此后动态把w_euc_cost改为0.5 | 0.5 | 仍为1.0 |
| 按0.5重新配置／重启 | 0.5 | 0.5 |

动态降低到0.5后，h与长度费用的关系也与启动时的设计不同，不能继续沿用重启后的推断来解释路径和搜索最优性。比较参数组合时应使用一致的初始化方式，尤其降低w_euc_cost时，优先重新配置／重启并核对有效行为。动态修改GridBased.w_heuristic_cost也不能补上缺少的成员更新。[当前动态回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/src/theta_star_planner.cpp#L216)

#### 相关源码默认项（原YAML未显式配置）

##### 参数表

这两项是当前Theta*插件实际支持的默认参数；本次没有向工程YAML追加。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `GridBased.allow_unknown` | `True`，插件默认 | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否允许未知255格参与常规搜索及视线检查 | 不适用 | 不适用 | True仍计经过费用，False常规搜索拒绝未知；前提是地图仍保留未知身份；同格起终点有不同快捷分支 |
| `GridBased.use_final_approach_orientation` | `False`，插件默认 | 不适用，布尔 | **根据实际功能需求开启或设置** | 决定返回路径最后一姿态使用目标方向还是末段接近方向 | 不适用 | 不适用 | False通常将最后姿态设为请求目标四元数；True多点路径用末段方向，单点／同格特殊情况用起点方向；不直接控制底盘转动 |

##### 未知区、不可用代价与起终点条件

当前常规搜索只放行内部代价c<252的普通格，255另按allow_unknown判断；因此252、253、254普通格都不可用，不能把“不是254致命障碍”理解成一定可规划经过。视线分支对转换费用采用<252比较，普通整数格对应同样的0～251通过范围；放行未知时用253作为该分支的转换费用进行计费。[普通格与未知格检查](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/include/nav2_theta_star_planner/theta_star.hpp#L96)

| 当前全局图中的情况 | 当前常规Theta*行为 |
|---|---|
| 0～251，图内 | 可参与搜索，按对应费用计分 |
| 252～254 | 拒绝；起点或目标落在这些格也会使常规规划失败 |
| 255，allow_unknown=True | 可参与搜索，费用较高；不是保证永不选择未知路线 |
| 255，allow_unknown=False | 拒绝 |
| 起点或目标在地图外 | 插件返回空路径，服务器失败 |
| 目标所在格不可用、附近有空格 | 当前没有NavFn式tolerance寻找邻近替代目标 |

同一格的起终点另走快捷分支：此版本只显式拒绝254，否则返回一个位于起点的姿态；没有执行上述常规allow_unknown／252～253检查。因此allow_unknown=False不能描述成插件所有分支都绝对不返回未知格路径。这里只记录源码边界，没有运行这个特殊请求。[同格快捷分支](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/src/theta_star_planner.cpp#L114)

实物global_costmap顶层track_unknown_space=True，仿真顶层未写而默认False；此外当前三态地图保存／加载还有灰205在free=0.25下变空闲的问题。若未知已被上游改成0，此处allow_unknown=False也无法恢复它。[全局未知表示](/wiki/2026-09-20-dao-hang/0500-1300-global-costmap)、[已验证的地图读写条件](/wiki/2026-09-20-dao-hang/0500-1500-map-saver)

通用Humble笔记中的GridBased示例使用NavFn，tolerance、use_astar等参数不能直接套给当前Theta*。当前插件以起终点落格进行搜索，常规回溯得到网格中心位置，再线性补点；它没有独立目标位置容差参数。controller_server的xy_goal_tolerance=0.15则是执行到达判定，不能用来让本插件自动替换被占用的目标格。

##### 最后朝向、路径补点与当前PID

当前use_final_approach_orientation=False时，常规路径最后姿态使用请求目标朝向；True时多点路径用最后两点的atan2方向，特定单点情况使用起点方向。这个开关修改返回路径的最后四元数，不修改机器人本体姿态，也不自动发角速度。

本版本linearInterpolation主要填写中间路径点的x、y，没有为每个中间姿态生成沿路径的有效朝向四元数；不能把路径中每个orientation当成已经规划好的车辆航向。当前PID的enable_rotation=False、use_rotate_to_heading=False，goal_checker.yaw_goal_tolerance=6.28，也不会因末端朝向设置就要求最终转到指定yaw。[末端方向与补点](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/src/theta_star_planner.cpp#L140)、[当前控制器条件](/wiki/2026-09-20-dao-hang/0500-1100-controller-server)

补点使用global_costmap的有效格距，不能固定认定就是YAML初始0.05。它在Theta*折线段上补位置点，没有自动生成曲率连续曲线、转弯半径约束或速度轨迹；与下一块smoother_server的服务是不同步骤。当前两棵默认XML均未调用SmoothPath，单有节点或BT库并不证明平滑已经运行。

#### 请求等待、路径验证、动态参数与本次核对

##### 地图等待与取消的边界

服务器规划前调用waitForCostmap，以100 Hz检查global_costmap是否current；这不是让地图以100 Hz更新。1.1.20该等待循环没有本块可配置的超时，也没有在循环内检查本次Action取消，Theta*搜索循环同样没有取消检查。expected_planner_frequency的100 ms只在成功完成后检查，不能终止持续等待或搜索。[等待实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_planner/src/planner_server.cpp#L272)

若请求长时间无结果，应先查静态地图、TF和各层current，再看搜索耗时；把expected_planner_frequency调小只是放宽警告门槛。反过来，current=True也不证明感知持续新鲜：当前地形来源缺省expected_update_rate=0，可以继续使用最后一份观测，详见全局图章节。

##### 返回非空路径与is_path_valid不是同一检查

服务器取得插件路径后，validatePath在此版本主要检查是否非空；不是再次完整执行每个路径段的车体扫掠碰撞验证。另有is_path_valid服务检查已有路径的剩余点；当前圆半径模式下，拒绝253／254及地图外点，但没有复用Theta*的252门槛和allow_unknown策略，不能把二者当成一致的通行判据。[返回路径检查](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_planner/src/planner_server.cpp#L345)、[已有路径检查](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_planner/src/planner_server.cpp#L561)

当前两棵默认BT未调用IsPathValid，不能因为已加载对应库就认定它每轮验证路径。全局规划、该服务和局部PID抽样检查各有范围；实际通过空间仍需正确地图、轮廓与车辆执行验证。

##### 动态修改的实际范围

| 参数类别 | 当前实现 | 核对方式 |
|---|---|---|
| expected_planner_frequency | 服务器动态回调重算正值倒数；非正值关闭门槛 | 看耗时警告门槛变化，不把它当规划请求频率变化 |
| planner_plugins、GridBased.plugin | 当前没有动态重建插件列表／实例的路径 | 改配置并重新配置／重启，核对加载日志、ID和对应参数 |
| GridBased.how_many_corners | 插件回调直接更新整数成员，未复做4／8检查 | 只用合法4／8；不要以参数设置成功代表非法值安全 |
| GridBased.w_euc_cost | 更新长度权重，但不重算内部启发权重 | 比较权重组合时使用一致初始化方式，尤其降低权重时重新配置 |
| GridBased.w_traversal_cost | 插件回调更新经过权重 | 观察同地图、同起终点的路径与耗时，不改变硬通行门槛 |
| YAML GridBased.w_heuristic_cost | 初始化和动态回调均无读取路径 | 暂不调；内部h仍按已初始化成员运行 |
| GridBased.allow_unknown、use_final_approach_orientation | 插件回调更新对应成员 | 核对真正的255格、常规／同格分支及最终控制器是否使用朝向 |

当前插件动态回调对权重和how_many_corners缺少完整范围校验，也没有像服务器那样围绕整次规划处理使用同一参数锁；不要假设一组运行中修改与重启初始化完全等价。本文没有修改运行节点参数。[服务器回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_planner/src/planner_server.cpp#L630)、[插件回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/src/theta_star_planner.cpp#L216)

##### 启动、接口与验证范围

| 项目 | 当前来源与用途 |
|---|---|
| 整套reality／simulation | navigation_launch加载对应nav2_params.yaml，经RewrittenYaml重写已有时钟；lifecycle_manager_navigation管理planner_server |
| 独立进程／组件 | 两种启动分支均加载相同配置；planner_server内部创建global_costmap节点 |
| compute_path_to_pose | 规划到一个目标的Action，当前BT指定GridBased |
| compute_path_through_poses | 按目标顺序逐段调用同一插件并拼接路径，下一段从上一段实际路径末端继续 |
| plan | nav_msgs/Path可视化话题；发布器Active且有订阅者才发布，不是控制器必须订阅的唯一传递方式 |
| is_path_valid | 对已有路径的独立服务检查，当前默认BT未调用 |

机器人namespace为/red_standard_robot1时，节点名为/red_standard_robot1/planner_server，Action／plan／is_path_valid通常位于该机器人namespace下，不自动在话题中再加planner_server一层。global_costmap另有自己的子namespace。

- **本次已核对：** 相关包版本、Theta*注册XML和库存在；算法及封装安装头文件与1.1.20一致；源码／安装副本的两套planner_server段一致，差异仅时钟；两棵默认BT均请求GridBased且目标规划节奏为3 Hz，未调用SmoothPath或IsPathValid。
- **源码能确认：** 8项原配置的使用条件；10 Hz仅对应100 ms完成后警告；4／8邻居与视线连接；长度按格数、经过费用偏置26及平方公式；YAML启发权重未读取、内部初始化公式与动态不同步；未知／普通格门槛、同格快捷分支、末端方向和等待／取消边界。
- **本次验证范围：** 费用和权重表为源码公式计算，未启动新的规划Action、未比较实车路线或运行中参数，不将库／头文件存在当成已验证整套运行行为。
- **实物待核实：** 有效全局图格距与代价、map／TF对齐、真实起终点落格、未知策略、绕行与通道通过空间、参数初始化方式、实际耗时及完整感知到执行链；原工程配置和插件源码保持原样。
