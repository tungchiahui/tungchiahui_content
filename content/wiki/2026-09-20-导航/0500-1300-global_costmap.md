---
title: "global_costmap（全局规划代价地图）"
---

## global_costmap（全局规划代价地图）

### 实车先看：整张地图和车周围的实时障碍怎样一起参与规划

**当前全局代价地图在map中表达，默认不滚动；接收静态二维地图后，整张网格的尺寸、分辨率与平面原点由静态地图决定。** 它先铺静态图，再加入terrain_map_ext筛出的障碍，最后膨胀，供planner_server中的Theta*规划器选择全局路径。

```text
map_server／slam_toolbox的/map → static_layer ───────────────────────┐
    └─ 地图宽高、分辨率、平面原点 → 调整并锁定全局网格              │
                                                                  │
terrain_map_ext（odom系，intensity为地形高差）                       │
    → 按消息时间转到map，并取得front_mid360原点                     │
    → 来源高度、本层高度、高差、三维量程、体素筛选                  │
    → intensity_voxel_layer：加入符合条件的障碍 ────────────────────┤
                                                                  ↓
                                      inflation_layer → 全局合成代价地图
                                                       ├─ Theta*直接读内存规划
                                                       ├─ get_costmap服务提供当前图
                                                       └─ costmap／updates／raw供订阅者
```

全局图可以覆盖整个静态地图，但本实时障碍来源只保留距雷达严格0.2～10 m之间、又落在地图和高度窗口内的有效点。10 m不是整张地图的宽度，也不保证10 m内每处都已被观测。上游扩展地形当前在4 m内借用近场结果，4 m外使用自己的远场结果，两部分都从这里进入全局障碍层。[扩展地形分析](/wiki/2026-09-20-dao-hang/0500-0500-terrain-analysis-ext)

| 实车上看到的现象 | 先核对什么 | 当前配置怎样理解 |
|---|---|---|
| YAML写5 cm，显示／服务中的格距却不同 | /map的info.resolution与实际costmap元数据 | 非滚动static_layer会按完整地图重建全局网格；YAML值不保证是最终格距 |
| 以为全局图也是5×5 m随车移动 | rolling_window缺省值与静态地图 | 当前缺省False，网格覆盖和原点随静态地图确定，不围绕车滚动 |
| 开了未知区域记录，路径仍经过灰色区域 | 顶层track_unknown_space与规划器allow_unknown | 实物顶层True保留255未知状态；当前Theta*缺省allow_unknown=True，仍可能规划经过 |
| 仿真和实物对未探索区域表现不同 | 两份YAML的顶层未知开关 | 仿真缺少顶层键，实际缺省False；层内同名True没有被读取 |
| 0.9 m处的障碍有地形点却没进入此层 | 转到map后的z及体素范围 | 两级0～2 m高度过滤之外，当前体素标记范围只有map中的[0,0.8) m |
| 局部有障碍，全局却没有，或反过来 | 输入、TF、高度窗口和两张地图 | 局部用terrain_map与odom；全局用terrain_map_ext与map，同一组高度数值不保证选中同一批点 |
| 增大cost_scaling_factor后路径反而更贴墙 | 膨胀衰减公式与Theta*路径评分 | 数值越大，外围软代价降得越快；当前全局规划器实际使用这些代价 |
| 清全局图后障碍很快回来 | 静态图、最后观测与上游历史点 | 整图服务会重置本层，但不会删除静态墙或terrain_map_ext的历史观测内容 |

本次容器内nav2_costmap_2d、nav2_voxel_grid、nav2_planner、nav2_theta_star_planner均为**1.1.20**，pb_nav2_plugins为**1.0.1**。自定义层实现与头文件和上游提交**e6778ca44d9e88aa612a3c6c87d536a8cfa21705**一致；Theta*安装头文件也与1.1.20一致。以下按这些版本解释，通用Humble笔记用来对照术语；自定义层的实际使用条件以源码为准。[自定义层实现](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca44d9e88aa612a3c6c87d536a8cfa21705/src/layers/intensity_voxel_layer.cpp)、[类与体素索引](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca44d9e88aa612a3c6c87d536a8cfa21705/include/pb_nav2_plugins/layers/intensity_voxel_layer.hpp)

### 参数

下面保留发送的完整38项参数与原注释。当前源码及安装副本的reality段均与此配置一致；simulation段除use_sim_time=True外，还**缺少顶层track_unknown_space**，不是只差时钟。

```yaml
global_costmap:
  global_costmap:
    ros__parameters:
      use_sim_time: False
      update_frequency: 5.0
      publish_frequency: 2.0
      global_frame: map
      robot_base_frame: gimbal_yaw_fake
      robot_radius: 0.2
      resolution: 0.05
      track_unknown_space: true
      plugins: ["static_layer", "intensity_voxel_layer", "inflation_layer"]
      intensity_voxel_layer:
        plugin: pb_nav2_costmap_2d::IntensityVoxelLayer
        enabled: true
        track_unknown_space: true
        footprint_clearing_enabled: true
        publish_voxel_map: false
        combination_method: 1
        mark_threshold: 0
        origin_z: 0.0
        unknown_threshold: 5
        z_resolution: 0.05
        z_voxels: 16
        min_obstacle_height: 0.0
        max_obstacle_height: 2.0
        min_obstacle_intensity: 0.1
        max_obstacle_intensity: 2.0
        observation_sources: terrain_map_ext
        terrain_map_ext:
          data_type: PointCloud2
          # '<robot_namespace>' keyword shall be replaced with 'namespace' where user defined.
          # It doesn't need to start with '/'
          topic: <robot_namespace>/terrain_map_ext
          sensor_frame: front_mid360
          min_obstacle_height: 0.0
          max_obstacle_height: 2.0
          obstacle_max_range: 10.0
          obstacle_min_range: 0.2
      static_layer:
        plugin: "nav2_costmap_2d::StaticLayer"
        map_subscribe_transient_local: True
      inflation_layer:
        plugin: "nav2_costmap_2d::InflationLayer"
        cost_scaling_factor: 4.0
        inflation_radius: 0.7
      always_send_full_costmap: False
```

两个global_costmap层级用于匹配命名空间和节点名。当前planner_server内部创建此地图节点；机器人namespace为/red_standard_robot1时，其全名为/red_standard_robot1/global_costmap/global_costmap，相关地图话题通常在/red_standard_robot1/global_costmap下。坐标系字符串map、gimbal_yaw_fake、front_mid360不会随节点namespace自动加前缀。[全局地图节点创建](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_planner/src/planner_server.cpp#L64)

#### ros__parameters（更新、坐标、轮廓与未知区域）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真True | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择ROS时钟，供TF、消息和发布判断使用 | 不适用 | 不适用 | 工程launch还会重写；需与地形点和TF同一时间域，内部更新循环使用WallRate |
| `update_frequency` | `5.0` | Hz，目标更新频率 | **实物运行时重点观察，有问题再针对性调整** | 控制内部三层地图合成循环，目标200 ms／轮 | 理想情况下更快更新实时障碍，计算和锁竞争增加 | 内部障碍变化响应可能更迟 | 不等于BT规划请求频率，也不提高雷达／地形输入频率；当前动态回调不重建此线程节拍 |
| `publish_frequency` | `2.0` | Hz，目标发布频率 | **通常保持当前值，不作为优先调试项** | 控制地图话题的发布时间门槛 | 订阅者可能更及时，发布负担增多 | 显示／订阅更新更稀 | 判断在5 Hz更新循环内，实际不保证独立精确2 Hz；Theta*读内存地图，服务也不等待下次话题发布 |
| `global_frame` | `map` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 设置全局网格及地形观测的参考系 | 不适用 | 不适用 | 当前非滚动静态层按相同格索引合成，要求与/map.header.frame_id的坐标约定一致；高度也按map中的z筛选 |
| `robot_base_frame` | `gimbal_yaw_fake` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 取得机器人在map中的位姿，并放置本图轮廓 | 不适用 | 不适用 | 当前不滚动，位姿变化不会移动整图原点；仍影响轮廓清空和规划参考点，需正确map至此系的TF |
| `robot_radius` | `0.2` | m，未填充轮廓的圆半径 | **上实物前必须核对并正确配置** | 无显式footprint时生成圆形多边形轮廓 | 车体占用与253膨胀内圈增大，通道可能受限 | 占用减小，可能低估真实转动／横移包络 | 当前还用默认约0.01 m padding；以虚拟系原点为中心，不等于雷达盲区或inflation_radius |
| `resolution` | `0.05`；收图后以/map元数据为准 | m／平面格，初始5 cm | **上实物前必须核对并正确配置** | 为初始化网格给出格距 | 若最终地图格距也增大，格子更粗、计算通常减少 | 若最终地图格距也减小，细节更细、格数和负担增加 | 当前static_layer会重建并锁尺寸；单独改这个YAML值不保证有效网格改变，也不能据此断言全图格数 |
| `track_unknown_space` | `true`，节点顶层 | 不适用，布尔 | **根据实际功能需求开启或设置** | 让主图、障碍层默认格及静态图未知格保留未知表示 | 不适用 | 不适用 | 此键实际被读取；True不等于禁止规划经过未知格，当前Theta*缺省allow_unknown=True；仿真缺省False |
| `plugins` | static_layer、intensity_voxel_layer、inflation_layer | 不适用，有序ID列表 | **上实物前必须核对并正确配置** | 加载并按顺序合成三层 | 不适用 | 不适用 | 当前先静态、再实时障碍、最后膨胀；各ID对应下方plugin类名，换顺序会改变最终图 |
| `always_send_full_costmap` | `False` | 不适用，布尔 | **根据实际功能需求开启或设置** | 选择OccupancyGrid强制全图或条件增量发布 | 不适用 | 不适用 | False遇尺寸、格距或原点变化仍发全图；costmap_raw有订阅者时始终发完整原始代价图，不影响内存规划 |

##### 参数详解

###### update_frequency与publish_frequency（内部5 Hz、话题2 Hz、规划请求各自运行）

内部更新循环以WallRate运行，目标每200 ms完成一次三层合成；更新时会锁地图，Theta*创建路径也锁同一张地图。增大更新频率需要看整图大小、点云处理、膨胀和规划耗时，不能只凭局部图10 Hz就照搬。

Theta*直接使用planner_server持有的Costmap2D对象；它不靠订阅2 Hz的OccupancyGrid取得地图。当前BackUpFreeSpace配置的global_costmap/get_costmap服务同样提供当前内存图。使用costmap_raw话题的其他消费者，则还受发布周期影响。

publish_frequency只是更新循环中的ROS时间门槛，理想500 ms发布间隔也会受200 ms循环采样和负载影响。若以2.5 m/s运动，理想200 ms内移动0.5 m；这是配置时间尺度计算，未包含感知、TF、规划、控制和执行延迟，也不是本次实测。[更新／发布循环](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_ros.cpp#L455)

BT中的RateController hz=3控制规划请求节奏，planner_server.expected_planner_frequency主要用于耗时检查，二者都不会把此5 Hz地图循环改成另一个频率。[行为树调度](/wiki/2026-09-20-dao-hang/0500-1000-bt-navigator)

###### resolution与未配置的rolling_window（最终尺寸来自/map）

当前rolling_window未写，源码缺省False。初始化暂用缺省width=5 m、height=5 m、origin_x=origin_y=0和本块resolution=0.05；这组初始参数**不是收到静态地图后的最终全局网格**。

非滚动StaticLayer收到完整地图时，如果主图的宽高格数、分辨率、平面原点不匹配，或尚未锁定尺寸，就调用resizeMap并锁定尺寸，所有层随之matchSize。最终依据是：

```text
全局x格数 = /map.info.width
全局y格数 = /map.info.height
全局平面格距 = /map.info.resolution
全局平面原点 = /map.info.origin.position.x、y
平面总格数 = width格数 × height格数
```

例如假设收到800×600、0.05 m／格的地图，全局覆盖40×30 m、共480000格；它不会缩成5×5 m。这仅说明换算，未读取当前运行/map，不能作为本次实际尺寸。

自定义体素层的x、y格数也跟随整张全局图，不只是为10 m量程单独分配一个局部窗口。每个平面列用32位打包高度状态；在固定物理覆盖下，把实际地图格距减半会使平面格数约4倍，也增加体素列、膨胀缓存和规划负担。

锁尺寸后，运行中设置resolution、width、height、origin_x／y可能改变ROS参数值及节点成员，但当前节点回调不会重新resize已锁地图。后续完整/map元数据改变仍可让StaticLayer重新调整。要核对有效尺寸，应看实际costmap消息／服务元数据及StaticLayer的Resizing日志，不能只看参数列表。[地图重建与锁定](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/static_layer.cpp#L179)、[运行中尺寸回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_ros.cpp#L758)

###### global_frame与robot_base_frame（同一张map中的点云、地图和机器人）

当前完整地图通常以map为header.frame_id，全局网格也选择map。1.1.20的StaticLayer在**非滚动分支直接按对应格索引合成**；只有滚动分支才查询地图坐标系与global_frame之间的TF并取样。因此不能把本块global_frame随意改为odom，再期待静态层自动重投影整图。[非滚动与滚动合成分支](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/static_layer.cpp#L419)

地形点云仍按自己的header.frame_id，经消息时间戳的TF转到map；机器人轮廓放在map中的gimbal_yaw_fake位姿上。应一起核对静态地图、先验定位和点云的坐标约定，以及map↔odom↔front_mid360／gimbal_yaw_fake链。话题名带机器人namespace不证明这些坐标系已经隔离或正确对齐。

robot_radius=0.2生成近似圆形多边形，再按默认footprint_padding约0.01 m填充。实现按顶点方向填充，最终内切半径不能简单等同于精确0.21 m圆。需测量虚拟原点到真实车体边界、云台和转动包络；局部与全局轮廓都应符合这套几何关系。[轮廓生成与填充](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_ros.cpp#L407)

如果改成矩形footprint，还须处理当前虚拟参考系抵消底盘yaw的设计：底盘固连矩形不会仅因填入顶点就自动按真实底盘方向转动。相关参考系关系见[fake_vel_transform](/wiki/2026-09-20-dao-hang/0500-0600-fake-vel-transform)。

###### track_unknown_space（保留未知状态与是否允许规划经过是两件事）

实物顶层True让主图和障碍层以NO_INFORMATION=255为默认值，并让静态图未知像素保留255。已知空闲仍为0，墙仍为254；这不是把整张静态地图都设为未知。

当前planner_server.GridBased实际使用nav2_theta_star_planner/ThetaStarPlanner，未显式配置allow_unknown，1.1.20缺省True。常规搜索会允许未知格参与候选，同时给它较高的经过代价；因此未知区保留为灰色后，仍可能出现穿过它的路径。若任务要求禁止未知区，应单独核对规划器策略，不能用unknown_threshold=5实现。[Theta*缺省allow_unknown](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/src/theta_star_planner.cpp#L49)、[搜索中的未知格判断](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/include/nav2_theta_star_planner/theta_star.hpp#L96)

当前仿真文件缺少顶层track_unknown_space，缺省False会把静态图未知值转为空闲，并让障碍层默认空闲。此时原来的未知身份已经丢掉，之后只把规划器allow_unknown改False也不能从0恢复它。需要一起明确**地图保留未知的方式、膨胀怎样处理未知、规划器是否放行**。

| 当前配置 | 节点顶层值 | 层内同名值 | 实际未知表示 |
|---|---|---|---|
| reality/global_costmap | True | True，但未读取 | 主图／障碍层默认未知；静态未知像素保留255 |
| simulation/global_costmap | 未写，默认False | True，但未读取 | 主图／障碍层默认空闲；静态未知像素转0 |
| 当前local_costmap，两套配置 | 未写，默认False | True，但未读取 | 与全局实物图的未知策略不同 |

这些差异是配置与源码核对结果，未对运行节点做参数快照。切换到仿真时不要只检查use_sim_time；是否统一未知策略取决于任务，本文保留原YAML并记录差异。[真正读取顶层键](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/obstacle_layer.cpp#L99)、[静态未知像素解释](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/static_layer.cpp#L263)

#### intensity_voxel_layer（扩展地形障碍的筛选与合成）

##### 参数表（工作方式）

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `pb_nav2_costmap_2d::IntensityVoxelLayer` | 不适用，类名 | **上实物前必须核对并正确配置** | 加载按地形intensity筛点的自定义层 | 不适用 | 不适用 | 继承ObstacleLayer，另建体素网格；不是标准VoxelLayer，不能直接套其射线清空和未知列逻辑 |
| `enabled` | `true` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 是否标记并合成本层实时障碍 | 不适用 | 不适用 | False不关闭静态图与膨胀，也不自动停止所有输入订阅；只靠静态图仍可能规划 |
| `track_unknown_space` | `true`，层内键未读取 | 不适用，布尔 | **当前运行不使用这个参数，暂不调整** | 当前层内同名设置没有使用路径 | 不适用 | 不适用 | 实物真正生效的是前面的节点顶层True；此处冗余，不能靠它让仿真开启未知记录 |
| `footprint_clearing_enabled` | `true` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 将本层二维轮廓内代价设为空闲 | 不适用 | 不适用 | 不直接清体素位；当前Max合成不会擦掉静态墙，但可将静态未知格中的本层已知轮廓区写为空闲 |
| `publish_voxel_map` | `false` | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否建立并发布voxel_grid调试话题 | 不适用 | 不适用 | 发布整个体素网格便于核对map高度窗口，增加开销；不控制计算；当前无运行中重建发布器回调 |
| `combination_method` | `1` | 不适用，枚举 | **通常保持当前值，不作为优先调试项** | 选择本层与静态结果的合成方法 | 不适用 | 不适用 | 1为Max；0为Overwrite；本版本其他整数没有对应合成分支，不套用新版枚举2的说明 |
| `unknown_threshold` | `5` | 计数；当前未参与未知列计算 | **当前运行不使用这个参数，暂不调整** | 初始化声明并保存未知体素门槛 | 当前无计算效果 | 当前无计算效果 | 保存5+(16−16)=5，后续没有读取它统计未知列或清线；不能用它控制Theta*放行未知格 |
| `observation_sources` | `terrain_map_ext` | 不适用，空白分隔的来源ID字符串 | **上实物前必须核对并正确配置** | 建立下方来源分组的缓冲与订阅 | 不适用 | 不适用 | ID用于找参数分组，实际输入由其topic决定；多来源需空白分隔并逐个配置 |

##### 参数表（体素、高度与地形高差）

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `mark_threshold` | `0` | 每个平面列的已标记高度格数量门槛 | **实物运行时重点观察，有问题再针对性调整** | 占用高度格数严格超过门槛时标成二维障碍 | 要更多不同高度格才能标记，可能减噪也漏薄／低障碍 | 更少高度格即可标记；当前0已是一格就够 | 不是同一格内的点数；重复很多点仍只算一个高度格；门槛应小于有效层数 |
| `origin_z` | `0.0` | m，map中的体素下界 | **上实物前必须核对并正确配置** | 设置垂直体素窗口起点 | 窗口整体上移 | 窗口整体下移 | 不随车升降；通过高度过滤但低于它的点会压到最底格；本自定义缺省16.0 m，不能按标准默认随意省略显式0 |
| `z_resolution` | `0.05` | m／高度格，5 cm | **上实物前必须核对并正确配置** | 决定每个高度格厚度 | 固定层数时覆盖更高，垂直表达更粗 | 覆盖更矮、表达更细，高处点更易越界 | 应为有限正值；当前16×0.05=0.8 m；独立于从静态图取得的平面格距 |
| `z_voxels` | `16` | 高度格数 | **通常保持当前值，不作为优先调试项** | 设置垂直格数 | 合法范围内覆盖更高；当前已达16上限 | 垂直覆盖变小，高处点更易越界 | 当前库最多16层，不能直接设40层覆盖2 m；每列打包内存不会随层数同比缩小 |
| `min_obstacle_height` | `0.0` | m，map中点z的本层下界 | **上实物前必须核对并正确配置** | 本层删除z低于0的点 | 删除更多低处／下坡点 | 接纳更低点，但仍受来源过滤与体素范围限制 | 两级下界都为0，边界可通过；不是点离局部地面的高差 |
| `max_obstacle_height` | `2.0` | m，map中点z的本层上界 | **上实物前必须核对并正确配置** | 本层删除z高于2 m的点 | 前置窗口变宽，不会扩展0.8 m体素窗口 | 降到有效体素窗口内才进一步裁掉当前可标记点 | 2 m不是最终有效标记高度；z≥0.8 m仍会因体素越界被删 |
| `min_obstacle_intensity` | `0.1` | m，当前地形高差下界 | **实物运行时重点观察，有问题再针对性调整** | 高差至少10 cm才进入标记 | 忽略更多小高差，可能减轻误判也漏低障碍 | 接纳更多低障碍、地面噪声与坡面误差 | 当前不是雷达反射强度；0.1边界可通过；上游远场高差取绝对值，低侧也可能过门槛 |
| `max_obstacle_intensity` | `2.0` | m，当前地形高差上界 | **通常保持当前值，不作为优先调试项** | 删除高差超过2 m的点 | 接纳更多大高差 | 删掉更多大高差 | 近场借入点受0.5 m上游限制，自算远场受1.0 m限制；正常输出通常不被2 m再裁剪，但代码确实执行此筛选 |

##### 参数详解

###### map中的z、上游高差与体素覆盖（三个不同概念）

ObservationBuffer先把terrain_map_ext从消息坐标系转到map，再按来源0～2 m筛z；本层再次按0～2 m筛z，同时按intensity=0.1～2 m筛地形高差。通过这些门槛之后，还需满足三维量程、平面地图边界、垂直格索引和mark_threshold。

**高度窗口用的是map中的点z；intensity沿用上游计算的地形高差。** TF改变XYZ，不会把intensity重算成map地面高差。地形点并未被压平到地面上。

当前高度格索引为floor((z_map−0)/0.05)，要求0≤索引<16；两级高度下界又排除了负z，因此有效标记区间为**map中的[0,0.8) m**。z=0.8对应索引16，越界。网格不会随底盘爬坡自动向上移动，map的z=0也不能仅凭名称认定就是每处实际地面。

| 假定点已满足量程与平面位置 | 地形高差 | 当前筛选结果 |
|---|---:|---|
| z_map=0.2 m | 0.15 m | 高度、高差与体素都通过，可标记 |
| z_map=0.2 m | 0.05 m | 高差不足10 cm，被删 |
| z_map=0.9 m | 0.15 m | 通过0～2 m过滤，但体素越界，不能标记 |
| z_map=-0.1 m | 0.15 m | 来源高度过滤已删除 |

局部使用odom中的[0,0.8)，全局使用map中的[0,0.8)。只有map←odom变换保持相同z且不混合竖直方向时，这两个窗口才选中相同高度。若有z平移或roll／pitch修正，同一个odom点转入map后的z可能不同；定位修正也可能让地图上的实时障碍位置变化。

例如假设odom点z=0.6 m、map←odom只有z平移+0.3 m，则全局看到z_map=0.9 m并丢掉，而局部仍可能标记。这只是坐标换算例子，未测得当前TF偏移。实车应先核对静态图对齐、地面高度基准和定位变换，再决定怎样共同配置两级高度、origin_z与z_resolution；只把max_obstacle_height从2调到3无效。[实际筛点](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca/src/layers/intensity_voxel_layer.cpp#L135)、[体素换算](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca/include/pb_nav2_plugins/layers/intensity_voxel_layer.hpp)

###### min_obstacle_intensity（10 cm高差并不等于可通过台阶）

当前4 m内借入近场terrain_map，远处自己计算abs(点高度−估计地面高度)。远场的较高侧和较低侧都可能出现0.15 m高差，满足其他窗口时都可成为障碍；正常坡面也可能因分片地面估计而过门槛。4 m交接处还存在两套上游网格、点密度和高度规则的差异。[近远场交接及绝对高差](/wiki/2026-09-20-dao-hang/0500-0500-terrain-analysis-ext)

0.1只决定高差筛选下界，不是车辆可跨10 cm或可爬某坡度的结论。先看地面估计和真实障碍回波，再结合车辆能力调整；提高门槛可能放掉需要避让的低矮物。

上游当前正常近场高差不足0.5 m，自算远场不足1.0 m，所以本层2.0 m高差上限通常不会进一步裁剪正常输出。若换输入或改上游规则，就要重新解释。接原始雷达点云时intensity可能是反射强度，不能沿用此处的米单位。

###### mark_threshold与unknown_threshold（已标记高度格和未使用成员）

mark_threshold按一个平面列中**不同高度格的占用数量**判断，严格超过门槛才标二维障碍。当前0表示1个高度格就够，1表示至少2格，2表示至少3格；同一高度格里有100个点也只算1格。每轮体素重置后重新计数，不靠多个更新周期累计凑数；最新地形点云自身仍可能包含历史点。[VoxelGrid标记计数](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_voxel_grid/include/nav2_voxel_grid/voxel_grid.hpp#L99)

unknown_threshold在初始化保存为5+(16−z_voxels)，当前为5；本自定义实现后续没有使用它计算未知列、清射线或投影二维未知状态。调这个值不会改变当前未知区域，也不会改变规划器的allow_unknown。它与节点顶层track_unknown_space的使用路径不同。

层内track_unknown_space=True同样没有被父类读取；实物已有有效的顶层True，所以本块是冗余键，不能照搬局部那块“缺少顶层设置”的结论。即使顶层True，本层也不会自动获得标准VoxelLayer的射线空闲推断。[自定义初始化与更新](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca/src/layers/intensity_voxel_layer.cpp)

###### combination_method与footprint_clearing_enabled（实时障碍如何加到静态图）

当前1为Max，但不是简单把255当最大数：本层自身未知格被跳过；主图原来未知、本层为已知时会写入本层值；两者都是已知则取较大代价。当前合成顺序和默认use_maximum=False使静态层先写静态结果，之后本层叠加实时障碍。[Max的未知格处理](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_layer.cpp#L109)

| static_layer当前结果 | 本层当前结果 | Max合成后，膨胀前 |
|---:|---:|---:|
| 0，已知空闲 | 254，实时障碍 | 254 |
| 254，静态墙 | 0，轮廓内空闲 | 254，不擦静态墙 |
| 0，已知空闲 | 255，本层未标记 | 0 |
| 255，静态未知 | 255，本层未标记 | 255 |
| 255，静态未知 | 0，轮廓内空闲 | 0 |
| 255，静态未知 | 254，实时障碍 | 254 |

本层每轮resetMaps重建，不是把本轮10 m范围外全部判成空闲。顶层True时，未标记区域默认未知；合成会跳过这些未知格，原静态图仍起作用。没有点或者点被过滤，不能由此证明区域可通行。

轮廓清空仅将本层二维多边形内设0，不清体素位，也不通过Max降低静态墙。错误或过大的轮廓可能删掉本层车旁真实障碍；在静态未知区还会写出局部已知空闲，需按实际几何核对。

###### 每轮重建、旧观测与清全局图服务

自定义updateBounds每轮先重置自身二维图和体素，再取标记观测重新标记。它不调用getClearingObservations或raytraceFreespace，直接把来源clearing设True不会补上射线清空。默认observation_persistence=0只留最后一份观测，**最后一份并不会立即过期**；expected_update_rate=0也不检查停更。

所以动态障碍残留可能来自最新terrain_map_ext中的历史点、最后一份停更观测或静态地图，而不是简单由“全局图不滚动”导致。主图按各层提供的更新边界重新合成，膨胀层还会扩展到上一轮边界；本层重置整张内部图不意味着每轮都完整发布或重写整张合成图。[合成更新边界](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/layered_costmap.cpp#L159)、[膨胀更新边界](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/inflation_layer.cpp#L134)

本类isClearable=False使ClearAround／ClearExcept等区域清理跳过它；但BT所用ClearEntireCostmap对应整图服务，会resetLayers并调用所有层reset，包含本层。重置不删除上游地形缓存，也不把最后观测彻底丢弃，下一轮相同内容仍可重建障碍；静态墙也会重新铺回。[区域／整图清理](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/clear_costmap_service.cpp#L102)、[整图重置](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_ros.cpp#L618)

#### intensity_voxel_layer.terrain_map_ext（观测来源）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `data_type` | `PointCloud2` | 不适用，消息类型名称 | **上实物前必须核对并正确配置** | 建立点云订阅与观测缓冲 | 不适用 | 不适用 | 自定义迭代器需要float XYZ和intensity字段；父类支持LaserScan不代表换成扫描仍满足本层字段要求 |
| `topic` | `<robot_namespace>/terrain_map_ext` | 不适用，话题名称 | **上实物前必须核对并正确配置** | 指定扩展地形点云输入 | 不适用 | 不适用 | bringup展开占位符；通常/red_standard_robot1/terrain_map_ext；原字符串不会被本层自行替换 |
| `sensor_frame` | `front_mid360` | 不适用，传感器原点系名称 | **上实物前必须核对并正确配置** | 获取量程筛选所用的雷达原点 | 不适用 | 不适用 | 按点云时间戳查此系到map；点坐标按消息header.frame_id转换，不被重新解释为雷达系点 |
| `min_obstacle_height` | `0.0` | m，map中点z的来源下界 | **上实物前必须核对并正确配置** | 缓冲阶段先删除z低于0的点 | 提前删更多低处／下坡点 | 允许更低点进入后续筛选 | 与本层同名键属独立阶段；只改本层下界不能恢复这里已删点；边界可通过 |
| `max_obstacle_height` | `2.0` | m，map中点z的来源上界 | **上实物前必须核对并正确配置** | 缓冲阶段先删除z高于2 m的点 | 来源窗口变宽，后续体素仍可越界 | 缩窄来源窗口，可能漏高处障碍 | 源码缺省0.0，不自动继承本层2 m；不能随意省略；当前最终仍受体素0.8 m限制 |
| `obstacle_max_range` | `10.0` | m，距雷达原点的三维距离 | **实物运行时重点观察，有问题再针对性调整** | 删除距离达到／超过10 m的点 | 接纳更远已有点，仍受上游和地图边界限制 | 实时障碍参与范围缩小 | 当前包含z，严格小于10 m才通过；不是静态地图范围或扩展节点4 m交接半径 |
| `obstacle_min_range` | `0.2` | m，距雷达原点的三维距离 | **上实物前必须核对并正确配置** | 删除距雷达不超过0.2 m的点 | 排掉更多近点，也可能漏近车障碍 | 接纳更多近点，也可能包含自车回波 | 当前使用≤下界删除；不是围绕gimbal_yaw_fake的robot_radius，也不能代替完整自车滤除 |

##### 参数详解

###### topic、sensor_frame与TF（点在哪里，距离从哪里量）

工程bringup先用机器人namespace替换<robot_namespace>，再加载参数；绕过该启动流程时需完成替换。当前terrain_map_ext输出header.frame_id为odom；ObservationBuffer按消息时间把XYZ转到map，并按同一时间取得front_mid360原点在map中的位置，保留包括intensity在内的点字段。

sensor_frame决定量程原点，不覆盖消息header.frame_id。显式设置front_mid360后，消息过滤器还要求此系可变换。输入采用SensorDataQoS，父类队列深度50；有点云话题不证明本层缓冲已经成功接收，也不证明TF时间匹配。[观测换系与原点](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/observation_buffer.cpp#L83)

量程计算为sqrt((x−x_sensor)²+(y−y_sensor)²+(z−z_sensor)²)，严格保留0.2<距离<10 m。相同刚体变换下距离保持不变，但变换后的绝对z可以改变；所以量程通过不证明map高度窗口通过。

10 m筛选也不能创造上游没输出的点，不能把地图外点写进网格，不能补回远处被地形高差上限删掉的障碍。近场／远场交接按上游输入位置的水平距离判断，此处量程按雷达原点的三维距离判断，两种边界不能混用。

#### static_layer（完整二维地图与网格几何）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `nav2_costmap_2d::StaticLayer` | 不适用，类名 | **通常保持当前值，不作为优先调试项** | 读取完整OccupancyGrid并铺入全局网格 | 不适用 | 不适用 | 当前非滚动，静态图还决定整张网格几何；读取/map消息，不直接读PCD或图片文件 |
| `map_subscribe_transient_local` | `True` | 不适用，QoS耐久性选择 | **上实物前必须核对并正确配置** | 用Transient Local、Reliable、KeepLast(1)订阅完整地图 | 不适用 | 不适用 | 发布端需兼容才能接到保留地图；不是保存地图到磁盘，也不是启用map_updates |

##### 参数详解

正常导航时完整图来自map_server，建图时来自slam_toolbox。缺省层私有map_topic为空，转用节点默认的机器人namespace/map，通常/red_standard_robot1/map。它不是global_costmap/costmap，更不是先验PCD直接输入。

当前默认trinary_costmap=True、lethal_cost_threshold=100、unknown_cost_value=255：OccupancyGrid的-1转成255后按顶层track_unknown_space保留未知或转空闲；已知值达到100为致命障碍，其余低于门槛的已知值按三态规则转为空闲。占用概率与膨胀后的内部软代价不是同一套数值。[静态像素解释](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/static_layer.cpp#L263)

当前use_maximum=False使非滚动静态层在更新区域先完整覆写静态结果，再由本实时层Max添加障碍；不要把本层combination_method=1误认为静态层也使用同一个参数。静态层自己的footprint_clearing_enabled缺省False，本块明确开启的是实时层的轮廓清空。

True的地图订阅QoS使晚启动的订阅者可取得兼容发布端保留的完整图。如果发布端耐久性或可靠性不兼容，应核对QoS；延长TF等待无法修复订阅不兼容。当前subscribe_to_updates=False不订阅map_topic加_updates的增量，但仍处理后来收到的完整/map；建图产生的新完整地图也可更新尺寸与内容。[订阅QoS及更新话题](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/static_layer.cpp#L70)

#### inflation_layer（全局路径的障碍距离代价）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `nav2_costmap_2d::InflationLayer` | 不适用，类名 | **通常保持当前值，不作为优先调试项** | 对已合成的静态墙和实时障碍建立距离代价 | 不适用 | 不适用 | 位于最后一层；不能恢复被高度、量程或高差过滤掉的障碍 |
| `cost_scaling_factor` | `4.0` | m⁻¹，指数衰减系数 | **实物运行时重点观察，有问题再针对性调整** | 控制内切轮廓之外软代价下降速度 | 同样距离的外围代价更低，衰减更快，路径可能更贴近障碍 | 外围代价更高、衰减更慢，通常更强地惩罚近障碍经过 | 当前Theta*使用软代价；影响还取决于w_traversal_cost等，不改0.7 m范围或轮廓内切半径 |
| `inflation_radius` | `0.7` | m，距致命障碍格的膨胀范围 | **实物运行时重点观察，有问题再针对性调整** | 决定多远仍写距离代价 | 影响范围扩大、计算增加，可能更早绕离障碍 | 影响范围缩小，外围路径代价减少 | 仅有效格距确为0.05时约14格距；不是0.7 m全禁行，也不再加robot_radius成为0.9 m |

##### 参数详解

###### 两项膨胀参数与当前Theta*（软代价实际参与路径评分）

内部代价0表示空闲，1～252通常称软代价，253为轮廓内切半径的高代价，254为致命障碍，255为未知。发布成OccupancyGrid后数值经过转换，不能拿RViz消息中的占用数直接代替内部值。

设d为距致命障碍格的实际平面距离，r_ins为**填充后轮廓**的内切半径。障碍格为254，内圈为253；内圈外、膨胀范围内按下式计算并取整数：

```text
cost = 252 × exp(−cost_scaling_factor × (d − r_ins))
```

因此4增大时外圈代价更快降到低值，减小时高代价保持得更远；0.7决定写代价的范围。仅为说明公式，假设r_ins=0.2 m、d=0.4 m，系数4约得113，8约得50，2约得169；实际轮廓含padding，格距以/map为准，不能把这些假设数当实测。[膨胀代价公式](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/include/nav2_costmap_2d/inflation_layer.hpp#L148)

当前Theta*常规搜索使用地图代价计算经过费用，当前配置w_traversal_cost=10.0，因此外围代价变化可以改变全局路径选择。1.1.20常规搜索将普通格的内部代价**252及以上**视为不可用，另对255按allow_unknown处理；这与当前局部PID抽样使用≥253阈值不同。“非致命软代价”不保证每一种规划器都放行。[Theta*代价与通行判断](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_theta_star_planner/include/nav2_theta_star_planner/theta_star.hpp#L191)

0.7 m膨胀区不会全部变成障碍，也不保证路径距墙至少0.7 m。需要结合有效轮廓、地图格距、定位误差、规划器代价权重和实际路径观察。局部图中改变外圈颜色不一定改变当前PID的抽样通过条件；全局图中这些代价则会进入Theta*路径评分，两张图的调参效果不能直接互推。[当前局部控制器用法](/wiki/2026-09-20-dao-hang/0500-1100-controller-server)

###### 未知格与膨胀开关（当前不把未知本身当障碍种子）

当前inflate_around_unknown缺省False，不主动以255未知格为膨胀起点；inflate_unknown缺省False，已知障碍扩到未知格时，不用1～252外围软代价覆盖未知，但仍可能写入≥253的内圈高代价。

这两个开关独立于顶层track_unknown_space，也独立于Theta*的allow_unknown。未知身份保留、边界膨胀和规划经过策略需一起检查；改其中一个不能默认得到另外两个的效果。[未知格写入条件](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/inflation_layer.cpp#L285)

###### always_send_full_costmap（固定整图也可以发增量）

False时，尺寸、分辨率和平面原点不变且有更新范围，可发布costmap_updates；首次发布或完整地图几何变化仍走全图分支。当前没有随车滚动原点，比局部图更容易满足几何不变条件，但不能保证每次都是增量。

costmap_raw有订阅者时始终准备完整nav2_msgs/Costmap，不受此开关改成增量；内部地图和get_costmap服务也不按该开关缩成更新块。因此这个开关主要影响OccupancyGrid发布方式，不能提高内部障碍刷新或控制Theta*读图频率。[全图／增量与raw发布](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_publisher.cpp#L184)

#### 相关源码默认项（原YAML未显式配置）

##### 参数表

下面20项为网格初始化、轮廓、观测缓冲和未知膨胀的源码默认设置；最后一项属于planner_server，用于解释本图的当前消费者。不是本次向原工程追加了这些参数。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `rolling_window` | `False`，节点默认 | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 选择固定全局网格或随车滚动窗口 | 不适用 | 不适用 | 当前非滚动，static_layer决定整图几何；开启后会换成另一套尺寸与静态取样逻辑，不能只当显示开关 |
| `width` | `5`，节点初始化默认 | m，初始x总边长 | **通常保持当前值，不作为优先调试项** | 无静态地图前的初始x范围 | 仅未被静态图接管时增加覆盖与格数 | 仅未被接管时减少覆盖 | 收到/map后以info.width格数为准；缺省5不证明全局覆盖5 m |
| `height` | `5`，节点初始化默认 | m，初始y总边长 | **通常保持当前值，不作为优先调试项** | 无静态地图前的初始y范围 | 仅未被静态图接管时增加覆盖与格数 | 仅未被接管时减少覆盖 | 是平面y，不是z高度；之后以/map.info.height格数为准 |
| `origin_x` | `0.0`，节点初始化默认 | m，初始平面原点x | **通常保持当前值，不作为优先调试项** | 放置初始网格x下界 | 初始网格沿x正向平移 | 初始网格沿x负向平移 | 后续以/map原点x为准，不是跟随机器人或先验定位偏移的调参入口 |
| `origin_y` | `0.0`，节点初始化默认 | m，初始平面原点y | **通常保持当前值，不作为优先调试项** | 放置初始网格y下界 | 初始网格沿y正向平移 | 初始网格沿y负向平移 | 后续以/map原点y为准；与体素origin_z独立 |
| `footprint` | `"[]"`，节点默认 | 不适用，轮廓顶点字符串 | **上实物前必须核对并正确配置** | 指定显式多边形，空值时使用圆半径 | 不适用 | 不适用 | 当前使用robot_radius；有效多边形优先；也可通过footprint话题更新，需符合虚拟参考系约定 |
| `footprint_padding` | 约`0.01`，节点默认 | m，轮廓填充量 | **上实物前必须核对并正确配置** | 扩大有效轮廓，影响清空范围与膨胀内圈 | 轮廓及高代价内圈增大 | 轮廓及内圈减小 | 按顶点填充；不能代替测量真实车体、活动包络与中心偏置 |
| `transform_tolerance` | `0.3`，节点默认 | s，TF查询等待 | **通常保持当前值，不作为优先调试项** | 供机器人位姿、观测缓冲等取得变换 | 可等待更久，延迟也可能增加 | 缺失TF时更快结束查询 | 不是允许0.3 m／rad外参误差；运行中改顶层值不会同步改所有已建缓冲中的副本 |
| `intensity_voxel_layer.terrain_map_ext.marking` | `True`，父类默认 | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 把来源加入标记观测列表 | 不适用 | 不适用 | 本自定义更新使用该列表，关闭后此来源不再贡献实时障碍 |
| `intensity_voxel_layer.terrain_map_ext.clearing` | `False`，父类默认 | 不适用，布尔 | **当前运行不使用这个参数，暂不调整** | 父类可加入清空列表，但本实现未使用 | 不适用 | 不适用 | True不自动开启raytrace；追加raytrace量程也不会让当前自定义更新使用它 |
| `intensity_voxel_layer.terrain_map_ext.observation_persistence` | `0.0`，父类默认 | s，观测缓存保留设置 | **通常保持当前值，不作为优先调试项** | 选择最后一份或一定时间内的多份观测 | 正值时保留更长历史，可能增多残留 | 正值减小时历史窗口缩短；0走特殊分支 | 0只保留最新一份，并非0 s后过期；不能清除上游扩展地形的历史点 |
| `intensity_voxel_layer.terrain_map_ext.expected_update_rate` | `0.0`，父类默认 | s，期望消息间隔 | **实物运行时重点观察，有问题再针对性调整** | 正值时检查来源是否按期更新 | 允许更长停顿仍算current | 正值变小更快判停更；0关闭检查 | 名字带rate，代码按秒建Duration；当前停更可能继续用旧观测且仍current |
| `static_layer.map_topic` | `""`，层默认；转用节点namespace/map | 不适用，话题名称 | **上实物前必须核对并正确配置** | 指定完整二维地图输入 | 不适用 | 不适用 | 当前通常/red_standard_robot1/map，不在global_costmap子namespace下；以实际重映射为准 |
| `static_layer.subscribe_to_updates` | `False`，层默认 | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否另外订阅map_topic加_updates的增量图 | 不适用 | 不适用 | False仍可接收后续完整地图；与Transient Local独立 |
| `static_layer.footprint_clearing_enabled` | `False`，层默认 | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 是否在静态层自身清轮廓区 | 不适用 | 不适用 | 与实时层同名True不同；当前静态墙不会被实时层Max空闲值擦掉 |
| `use_maximum` | `False`，节点默认，供静态层读取 | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 决定静态层覆写或取Max合成 | 不适用 | 不适用 | 当前非滚动用TrueOverwrite铺静态图；不等于实时层combination_method，也不是多雷达强度合并开关 |
| `trinary_costmap` | `True`，节点默认 | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 将静态占用值按未知、障碍、空闲解释 | 不适用 | 不适用 | 缺省致命门槛100、未知值255；True时低于致命门槛的已知格为空闲，膨胀软代价另算 |
| `inflation_layer.inflate_unknown` | `False`，层默认 | 不适用，布尔 | **根据实际功能需求开启或设置** | 控制已知障碍膨胀到未知格时是否写软代价 | 不适用 | 不适用 | False仍可写≥253内圈；不能简单理解成保留所有未知格毫不变化 |
| `inflation_layer.inflate_around_unknown` | `False`，层默认 | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否把未知格本身也作为膨胀起点 | 不适用 | 不适用 | 当前不围绕未知本身膨胀；与前一项及allow_unknown各有独立作用 |
| `planner_server.GridBased.allow_unknown` | `True`，Theta*默认；不属于本地图节点 | 不适用，布尔 | **根据实际功能需求开启或设置** | 决定当前规划器是否允许未知格参与常规搜索 | 不适用 | 不适用 | 地图先保留255才有身份可判断；False不能恢复已被静态层转成0的未知区 |

observation_persistence=0和expected_update_rate=0共同意味着：消息停止时最后一份观测可能继续被重建，且不会因该来源超时自动变为不current。地图继续发布不证明感知继续更新，应观察terrain_map_ext时间戳、来源缓冲与系统的数据停更处理。[缓存保留和current判断](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/observation_buffer.cpp#L187)

#### 与local_costmap的关系、动态修改与验证范围

##### 当前两张地图的主要区别

| 项目 | global_costmap | local_costmap |
|---|---|---|
| 主要消费者 | planner_server中的Theta*，决定全局路径 | controller_server中的OmniPidPursuitController，跟踪路径及抽样碰撞检查 |
| 坐标参考 | map | odom |
| 平面范围 | 默认不滚动，完整/map决定网格几何 | 5×5 m、0.05 m格距的滚动窗口 |
| 实时地形输入 | terrain_map_ext，包含借入近场与自算远场 | terrain_map，近场结果 |
| 三维量程筛选 | 严格0.2～10 m，且必须落在静态地图边界内 | 严格0.2～5 m，且必须落在局部窗口内 |
| 内部更新／话题目标频率 | 5 Hz／2 Hz | 10 Hz／5 Hz |
| 当前有效体素高度 | map中的[0,0.8) m | odom中的[0,0.8) m |
| 顶层未知设置 | 实物True；仿真未写，默认False | 两套均未写，默认False |
| 同样膨胀数值的使用 | Theta*常规搜索使用软代价，普通格≥252不放行，未知另判断 | 当前PID仅抽10点检查≥253，不按外围软梯度评分 |

障碍出现差异时，先沿“上游输入→TF后的z→高差→量程→平面范围→合成→消费者”定位。只让两张地图的YAML数字相同，不能保证实际选点、未知策略或路径通过条件相同。[局部代价地图](/wiki/2026-09-20-dao-hang/0500-1200-local-costmap)

##### 动态修改的实际范围

| 参数类别 | 当前实现 | 核对方式 |
|---|---|---|
| robot_radius、footprint_padding、footprint、publish_frequency | 节点回调会更新轮廓或发布门槛 | 观察实际轮廓、内切膨胀圈与发布结果；不能只看设置成功 |
| robot_base_frame | 节点回调先检查到global_frame的TF，再接受变更 | 即使接受，也需核对整套参考点、轮廓和控制链是否使用同一约定 |
| width、height、resolution、origin_x／y | 未锁尺寸时可resize；当前static_layer收图后锁定 | 当前锁定状态下参数值改变不代表网格改变，以实际地图元数据为准 |
| update_frequency、global_frame、rolling_window、plugins、track_unknown_space、always_send_full_costmap | 无完整动态联动更新路径 | 重新配置／重启并检查有效行为；改变未知设置需让静态图重新按策略解释 |
| 本层enabled、footprint_clearing_enabled、两项层高度、combination_method | 父类ObstacleLayer回调更新对应成员 | 改层高度不更新来源缓冲自己的高度窗口 |
| origin_z、z_resolution、z_voxels、mark_threshold、unknown_threshold、两项intensity、publish_voxel_map | 自定义层没有对应动态分支 | 重新配置／重启；unknown_threshold重启后仍未参与当前计算 |
| terrain_map_ext的来源参数 | 初始化时创建订阅器及ObservationBuffer | 修改配置后重新配置／重启，再核对来源、TF、过滤和停更状态 |
| 静态图话题、Transient Local、subscribe_to_updates | StaticLayer回调明确拒绝这些运行中修改 | 重建订阅后验证QoS与实际图来源 |
| 膨胀半径、衰减系数和未知膨胀开关 | 膨胀层回调更新成员，必要时重算缓存／重膨胀 | 核对全局代价和实际Theta*路径，结合有效地图格距判断 |

顶层transform_tolerance虽有节点成员更新分支，已建ObservationBuffer、消息过滤器及静态层另有初始化时保存的值；一次修改不保证所有等待同步更新。[节点动态回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_ros.cpp#L662)、[障碍父类回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/obstacle_layer.cpp#L289)

##### 启动来源与本次核对

| 启动方式 | 本块来源与关系 |
|---|---|
| 整套reality | config/reality/nav2_params.yaml，经bringup展开topic占位符、navigation_launch重写时钟加载；顶层未知True |
| 整套simulation | config/simulation/nav2_params.yaml；时钟True，顶层未知缺省False，不能说只差时钟 |
| 组件／独立进程 | 此地图节点由planner_server内部创建，读取相应namespace下参数；不是这份YAML自行启动进程 |
| slam=True | 静态图来自建图产生的完整/map；实时层仍用terrain_map_ext |
| slam=False | 静态图来自map_server；实时层仍用terrain_map_ext，需要定位链对齐map与odom |

- **本次已核对：** 相关安装包版本、已有自定义层注册／库／头文件；本地自定义实现与固定上游提交一致；Theta*安装头文件与1.1.20一致；源码和安装副本的两套global_costmap段一致，实物／仿真差异为时钟与顶层未知键。
- **源码能确认：** 38项原参数的实际读取／使用条件；静态图重建并锁定非滚动网格；实物顶层未知开关生效、层内同名键与unknown_threshold未参与相应计算；map垂直标记窗口[0,0.8) m；三维严格量程；每轮重建但不射线清空；旧观测保留；Max合成、膨胀方向及当前Theta*的未知／软代价使用。
- **本次验证范围：** 未启动新的导航或建图任务，未取得运行/map与costmap元数据、有效参数快照或路径对比；没有实测map←odom的高度偏移、坡道筛点、动态障碍残留或未知区经过行为。文中尺寸、高度和时间例子均为配置／源码计算。
- **实物待核实：** 实际静态图尺寸／格距／原点与坐标约定、先验地图对齐、map高度基准、真实轮廓与虚拟参考点、点云字段和TF时间、来源停更处理、近远场交接、未知规划策略以及全局路径到局部执行的通过空间。
