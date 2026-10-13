---
title: "local_costmap（局部障碍代价地图）"
---

## local_costmap（局部障碍代价地图）

### 实车先看：地图上的障碍来自哪里

**当前局部代价地图是以gimbal_yaw_fake位置为中心、在odom中表达的约5×5 m滚动网格。** 它按顺序合成静态二维地图、实时地形障碍和障碍周围的膨胀代价，供controller_server使用。

```text
/map二维占用图 → static_layer ──────────────────────────────┐
                                                         │
terrain_map（odom系地形点，intensity为离地高差）            │
    → 观测缓冲：TF、来源高度筛选、雷达原点                  │
    → intensity_voxel_layer：高度、高差、量程与体素筛选 ───┤
                                                         ↓
                                        inflation_layer → 局部合成代价地图
                                                         ├─ controller_server读内存中的地图
                                                         └─ costmap／costmap_updates／costmap_raw供订阅者
```

本块直接接收近场terrain_map；全局代价地图接收terrain_map_ext，下一块再整理。pointcloud_to_laserscan生成的obstacle_scan用于二维建图，本局部障碍层不经过那份二维扫描。因此修改扫描转换器的高差门槛，不会直接修改这里的门槛。[地形分析分工](/wiki/2026-09-20-dao-hang/0500-0400-terrain-analysis)、[二维扫描转换](/wiki/2026-09-20-dao-hang/0500-0800-pointcloud-to-laserscan)

| 实车上看到的现象 | 先核对什么 | 当前配置怎样理解 |
|---|---|---|
| 以为局部地图覆盖车周围5 m半径 | width、height与量程 | 5 m是整张网格的边长，各轴只约覆盖中心两侧2.5 m；量程5 m不能扩展网格 |
| 2 m内的高障碍仍没有进入此层 | origin_z、z_resolution、z_voxels | 当前体素高度是odom中的[0,0.8) m，超过0.8 m的点不能进入标记；2 m高度门槛只是前置过滤 |
| 上坡高处或下坡低处的障碍消失 | odom高度基准、两级高度窗口、体素窗口 | 高度窗口固定在odom，不随底盘z跟着升降；地形高差正确也可能被绝对高度裁掉 |
| 写了track_unknown_space=True，未探测区却像空闲 | 参数所在层级 | 当前版本读取节点顶层track_unknown_space；写在intensity_voxel_layer内没有配置到它，顶层缺省False |
| 改unknown_threshold看不到效果 | 自定义层的使用路径 | 本实现没有用它计算未知列／二维未知状态；不能套用标准VoxelLayer的解释 |
| 障碍已经移开，却继续残留 | 最新terrain_map、上游衰减、观测缓冲与静态层 | 本层每轮重建，最新观测仍可能含上游历史点；数据停止时默认还会复用最后一份观测 |
| 以为cost_scaling_factor越大越保守 | 膨胀公式与当前控制器用法 | 越大，外围软代价衰减越快；当前PID只抽样检查高代价值，不用这套软梯度给轨迹打分 |

本次容器内nav2_costmap_2d、nav2_voxel_grid均为**1.1.20**，pb_nav2_plugins为**1.0.1**。本地自定义层实现与头文件和上游提交**e6778ca44d9e88aa612a3c6c87d536a8cfa21705**一致；以下按该版本核对。[自定义层实现](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca44d9e88aa612a3c6c87d536a8cfa21705/src/layers/intensity_voxel_layer.cpp)、[体素换算与类定义](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca44d9e88aa612a3c6c87d536a8cfa21705/include/pb_nav2_plugins/layers/intensity_voxel_layer.hpp)

### 参数

下面保留发送的完整参数与原注释。实物／仿真段只有use_sim_time不同；源码缺省值及配置建议另列。

```yaml
local_costmap:
  local_costmap:
    ros__parameters:
      use_sim_time: False
      update_frequency: 10.0
      publish_frequency: 5.0
      global_frame: odom
      robot_base_frame: gimbal_yaw_fake
      rolling_window: true
      width: 5
      height: 5
      resolution: 0.05
      robot_radius: 0.2
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
        observation_sources: terrain_map
        terrain_map:
          data_type: PointCloud2
          # '<robot_namespace>' keyword shall be replaced with 'namespace' where user defined.
          # It doesn't need to start with '/'
          topic: <robot_namespace>/terrain_map
          sensor_frame: front_mid360
          min_obstacle_height: 0.0
          max_obstacle_height: 2.0
          obstacle_max_range: 5.0
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

两个local_costmap层级用于匹配命名空间和节点名。例如机器人命名空间为/red_standard_robot1时，当前controller_server创建的代价地图节点为/red_standard_robot1/local_costmap/local_costmap；发布话题通常位于/red_standard_robot1/local_costmap下面。坐标系字符串不会随这个命名空间自动加前缀。

#### ros__parameters（更新、坐标、网格与机器人轮廓）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真True | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择ROS时钟，供TF、消息与发布判断使用 | 不适用 | 不适用 | 经工程launch重写；应与点云及TF使用同一时间域；内部更新循环另用WallRate |
| `update_frequency` | `10.0` | Hz，目标更新频率 | **实物运行时重点观察，有问题再针对性调整** | 控制内部地图合成循环，当前目标100 ms／轮 | 理想情况下更快处理变化，计算与锁负担增加 | 内部障碍变化响应可能更迟 | 不提高输入点云采样频率，也不保证每轮有新观测；当前回调不重建该更新线程节拍 |
| `publish_frequency` | `5.0` | Hz，目标发布频率 | **通常保持当前值，不作为优先调试项** | 控制地图话题发布的时间门槛 | 订阅者可能更及时，发布负担增加 | 话题显示／订阅更新更稀，内部地图仍可按10 Hz更新 | 发布判断在更新循环中；不保证实际5 Hz；控制器直接读内存，部分行为读地图话题或服务 |
| `global_frame` | `odom` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定这张局部网格的坐标参考系 | 不适用 | 不适用 | 点云最终转到此系；static_layer仍需将/map的map系转换到odom；高度门槛也按该系z计算 |
| `robot_base_frame` | `gimbal_yaw_fake` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定滚动中心、控制器参考点与轮廓的机体系 | 不适用 | 不适用 | 与当前虚拟参考系设计配合；圆形轮廓中心在此原点，不能默认等于车体几何中心 |
| `rolling_window` | `true` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 随机器人位置移动平面网格原点 | 不适用 | 不适用 | 滚动平面窗口不会使origin_z随车升降，也不让网格轴随底盘yaw旋转；False为固定窗口 |
| `width` | `5` | m，网格x方向总边长 | **实物运行时重点观察，有问题再针对性调整** | 控制局部地图x方向覆盖范围 | 可保留更远的已有障碍和路径，网格与计算增多 | 范围变小，更容易在窗口外裁掉路径／障碍 | 当前按5/0.05=100格；不是5格或5 m半径；本版本使用正整数米 |
| `height` | `5` | m，网格y方向总边长 | **实物运行时重点观察，有问题再针对性调整** | 控制局部地图y方向覆盖范围 | 覆盖更宽，网格与计算增多 | 侧方范围变小，横移时可用环境更少 | 这里的height是平面y边长，不是体素z高度；当前100格 |
| `resolution` | `0.05` | m／平面格，5 cm | **实物运行时重点观察，有问题再针对性调整** | 设置x、y格子边长 | 格子更粗、格数更少，细障碍和窄通道表达可能变差 | 格子更细、计算及缓存增多，但不能补回上游丢点 | 当前100×100共10000格；两方向分辨率减半时格数约4倍；不等于定位精度 |
| `robot_radius` | `0.2` | m，未填充轮廓的圆半径 | **上实物前必须核对并正确配置** | 没有显式footprint时，以此半径生成圆形轮廓 | 车体占用与膨胀内圈变大，狭窄通道更受限 | 占用变小，可能低估真实车体／云台包络 | 当前无显式footprint，另有默认0.01 m padding；不是雷达盲区或膨胀半径，应按实际包络核对 |
| `plugins` | static_layer、intensity_voxel_layer、inflation_layer | 不适用，有序ID列表 | **上实物前必须核对并正确配置** | 按顺序加载并合成三层 | 不适用 | 不适用 | 当前先静态、再实时障碍、最后膨胀；各ID需有plugin类名；增删与换顺序会改变合成路径 |
| `always_send_full_costmap` | `False` | 不适用，布尔 | **根据实际功能需求开启或设置** | 选择OccupancyGrid每次强制全图或按条件使用增量 | 不适用 | 不适用 | False遇到尺寸、分辨率或滚动原点变化仍发全图；costmap_raw有订阅者时始终发完整原始代价图；不改变内部障碍判定 |

##### 参数详解

###### update_frequency、publish_frequency与always_send_full_costmap（内部10 Hz与话题5 Hz各负责什么）

内部更新循环以WallRate运行，调用各层updateBounds、updateCosts合成地图。controller_server通过持有的local_costmap对象直接读取这张地图；它的20 Hz控制循环与本块10 Hz更新各自运行，部分控制轮次可能读到同一轮地图结果。

publish_frequency控制更新循环中是否到了发布时间，不是独立的5 Hz发布定时器；实际受更新时间、负载和ROS时钟影响。若以2.5 m/s运动，理想100 ms更新间隔内已移动0.25 m，即5个当前格距；这是时间尺度计算，不是本次实车测量。调频率需要结合整条感知和执行链延迟。[更新与发布循环（1.1.20）](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_ros.cpp#L455)

False不会保证只发增量：网格原点滚动、尺寸或分辨率变化时，OccupancyGrid会走全图分支；几何信息不变且有更新范围时，才可能发costmap_updates。costmap_raw不走这个增量选择，有订阅者就准备完整nav2_msgs/Costmap。使用地图话题的行为与读取服务的行为，还应各自核对数据接口。[发布选择](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_publisher.cpp#L184)

###### global_frame、robot_base_frame与rolling_window（odom中的移动窗口）

网格轴按odom的x、y表达，中心跟踪gimbal_yaw_fake的平面位置，平面原点以格距对齐滚动；它不是随底盘姿态旋转的车体图。当前虚拟系的方向与速度转换关系见[fake_vel_transform](/wiki/2026-09-20-dao-hang/0500-0600-fake-vel-transform)。

局部地图使用odom可以减少全局重定位修正直接作用到局部点云坐标的影响，但静态层仍从map系地图取样，需要有效map↔odom。只有odom至机器人TF，不证明三层合成所需TF已经全部可用。

窗口滚动只处理平面x、y。体素origin_z=0仍固定在odom的z基准上，机器人上坡、下坡不会自动平移这段垂直窗口。[滚动原点实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/layered_costmap.cpp#L132)

###### width、height与resolution（5×5 m，100×100格）

当前约覆盖参考点两侧各2.5 m，角落的平面距离约3.5 m；实际原点受格距对齐，不能把边界当成精确连续对称的圆。5 m观测量程用于筛点，最终落在网格外的点仍不能标记。

平面共有10000格。自定义层再为每个平面列记录16个高度状态，逻辑上约160000个体素；实现用每列32位打包，不是为每个体素存一份完整点云。减小z_voxels不等于平面列缓存内存按比例减少。

平面resolution与垂直z_resolution是独立参数，当前恰好都为0.05 m。改变平面格距还会影响障碍合并、轮廓栅格化、膨胀和控制器局部路径处理；它不能增加MID360回波密度或恢复地形分析已经过滤掉的小障碍。

###### robot_radius（围绕虚拟参考点的车体轮廓）

当前默认footprint="[]"，因此用robot_radius=0.2生成近似圆形多边形，再按默认footprint_padding≈0.01 m填充。padding按多边形顶点方向实现，不应把最终轮廓简单等同于精确半径0.21 m的圆。[轮廓建立与填充](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_ros.cpp#L407)

该轮廓以gimbal_yaw_fake原点为中心。需按真实底盘、雷达、云台及其活动包络核对0.2是否覆盖，而不是继承示例半径；安装参考点偏置也要计入。膨胀不能替代正确的车体轮廓。

如果以后改成车体矩形footprint，须处理其坐标轴和真实底盘姿态：当前虚拟系在平面上抵消yaw，不会自动随底盘转向。可选择覆盖转动包络的轮廓，或按实际姿态提供正确的动态轮廓，不能直接填一个底盘固连矩形就认定方向已正确。

#### intensity_voxel_layer（地形高差与三维障碍筛选）

##### 参数表（工作方式与合成）

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `pb_nav2_costmap_2d::IntensityVoxelLayer` | 不适用，类名 | **上实物前必须核对并正确配置** | 加载本工程按intensity筛点的自定义障碍层 | 不适用 | 不适用 | 继承ObstacleLayer，另有体素网格；每轮重建，不实现标准VoxelLayer的全部清空／未知列逻辑 |
| `enabled` | `true` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 是否让本层处理并合成实时障碍 | 不适用 | 不适用 | False停止本层标记与合成，不关闭static_layer／inflation_layer，也不自动停止所有已建立的输入订阅 |
| `track_unknown_space` | `true`，但层级未被读取 | 不适用，布尔 | **已确认当前值错误，必须修改** | 当前层内键没有配置到真正的未知空间开关 | 不适用 | 不适用 | 当前1.1.20读取节点顶层track_unknown_space；未配置顶层时默认False；错误在层级，具体True／False取决于未知区域处理需求 |
| `footprint_clearing_enabled` | `true` | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 将本层二维轮廓内代价设为空闲 | 不适用 | 不适用 | 只清本层二维代价，不直接清体素位，也不能通过当前Max合成擦掉静态墙；错误轮廓可能清掉车旁真实点 |
| `publish_voxel_map` | `false` | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否创建并发布三维体素网格话题 | 不适用 | 不适用 | 开启便于核对高度格，增加发布开销；不控制障碍计算；当前自定义层没有运行中重建发布器的回调 |
| `combination_method` | `1` | 不适用，枚举 | **通常保持当前值，不作为优先调试项** | 选择本层二维代价与前面图层的合成方法 | 不适用 | 不适用 | 当前1为Max，保留静态障碍并添加实时障碍；0为Overwrite；此版本其他整数落入不合成分支，不套用新版枚举 |
| `unknown_threshold` | `5` | 计数；当前未参与未知列计算 | **当前运行不使用这个参数，暂不调整** | 声明并保存未知体素门槛，当前没有对应使用路径 | 当前无计算效果 | 当前无计算效果 | 当前内部赋值5+(16−16)=5，但从未用它清线或投影未知列；不能用它修复未知区或坡道 |
| `observation_sources` | `terrain_map` | 不适用，空白分隔的来源ID字符串 | **上实物前必须核对并正确配置** | 建立下方terrain_map来源的缓冲与订阅 | 不适用 | 不适用 | ID用于找来源分组；真正话题由该分组的topic决定；多来源用空白分隔，仍需XYZ与intensity字段 |

##### 参数表（高度、体素与高差）

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `mark_threshold` | `0` | 每个平面列内已标记高度格的数量门槛 | **实物运行时重点观察，有问题再针对性调整** | 标记高度格数严格超过门槛时，该平面格成为障碍 | 要求更多不同高度格被占用，可能减少零散噪声，也漏掉薄／低障碍 | 更少高度格即可成为障碍；当前0已是一格即可 | 不按同一体素内点数累计；重复100个同高度格的点仍只算一格；门槛应小于有效z格数 |
| `origin_z` | `0.0` | m，odom中的体素下界 | **上实物前必须核对并正确配置** | 设置整段垂直网格的起点 | 固定层数／格距时整段窗口上移 | 整段窗口下移 | 不随底盘升降；低于它且已通过高度过滤的点会压到最底格；本自定义层缺省竟为16.0 m，当前显式0不能按标准默认随意删除 |
| `z_resolution` | `0.05` | m／高度格，5 cm | **上实物前必须核对并正确配置** | 设置每个z格的厚度 | 固定16层时覆盖更高，垂直表达更粗 | 覆盖更矮、表达更细，可能删掉高处点 | 当前16×0.05=0.8 m；应为有限正值，独立于平面resolution |
| `z_voxels` | `16` | 高度格数 | **通常保持当前值，不作为优先调试项** | 指定垂直格数 | 合法范围内覆盖更高；当前已达16层上限 | 覆盖变矮，更多高点越界 | 当前库最多16层；减少格数不等于每列打包内存同比减少；不能用40层覆盖2 m |
| `min_obstacle_height` | `0.0` | m，odom中点z的本层下界 | **上实物前必须核对并正确配置** | 本层标记时删除低于0的点 | 下界上移，删更多低处／下坡点 | 下界下移，允许更低点，但来源窗口还可能先删掉 | 当前和来源下界同为0；z不代表相对局部地面的高差，边界可通过 |
| `max_obstacle_height` | `2.0` | m，odom中点z的本层上界 | **上实物前必须核对并正确配置** | 本层标记时删除高于2 m的点 | 前置高度窗口更宽，但不能扩大0.8 m体素窗口 | 降到体素有效窗口内才进一步删掉当前可标记点 | 2 m不是最终有效标记高度；仅在0.8 m以上改变通常不改当前结果，z≥0.8 m仍因体素越界丢掉 |
| `min_obstacle_intensity` | `0.1` | m，当前地形高差下界 | **实物运行时重点观察，有问题再针对性调整** | 地形点高差至少10 cm才进入障碍标记 | 更多小高差被忽略，可能减轻坡面误判，也漏低障碍 | 接纳更多低障碍与地面噪声／坡面误差 | 此输入intensity是离估计地面的高差，不是雷达反射强度；0.1边界可通过 |
| `max_obstacle_intensity` | `2.0` | m，当前地形高差上界 | **通常保持当前值，不作为优先调试项** | 删除高差超过2 m的点 | 接纳更多大高差 | 删掉更多大高差 | 当前正常近场输出另受vehicleHeight=0.5 m限制，2 m通常不再裁剪；参数仍实际执行 |

##### 参数详解

###### 高度、体素窗口与intensity（三道不同的筛选）

当前完整标记流程包括：来源缓冲高度筛选、本层高度筛选、地形高差筛选、相对传感器的三维量程筛选、平面与高度格范围，以及mark_threshold。通过其中一项不保证最终进入地图。

**点的z是odom坐标；intensity是上游估计地面的高差。** 假定量程、平面位置和其他条件都满足：

| 点在odom中的z | 地形高差intensity | 当前结果 |
|---:|---:|---|
| 0.2 m | 0.15 m | 两级高度、高差及体素范围都通过，可标为障碍 |
| 0.2 m | 0.05 m | 高差不足0.1 m，被删 |
| 0.9 m | 0.15 m | 0～2 m高度过滤通过，但超过当前体素上界，无法标记 |
| -0.1 m | 0.15 m | 来源高度下界已删除，尚未到体素处理 |

当前整数高度索引为floor((z−0)/0.05)，要求索引小于16，故有效范围为**[0,0.8) m**。z=0.8对应索引16，越界；上界不包含。这段窗口固定在odom，odom的z=0并不凭名称保证就是实车当前地面。

上坡高处、下坡低处的障碍即使地形高差正确，也可能被高度窗口删掉。若需要改变覆盖，必须一起核对来源高度、本层高度、origin_z与16层限制内的z_resolution；只把max_obstacle_height从2调到3不能解决体素上界。也不能为了放行坡面就把所有危险低障碍一起过滤。[实际筛点与索引转换](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca/src/layers/intensity_voxel_layer.cpp#L135)

###### min_obstacle_intensity与max_obstacle_intensity（是否把地形高差当障碍）

近场terrain_analysis已经把intensity改成相对估计地面的高差，当前下界0.1即10 cm。它不是“车辆能跨10 cm”，也不是坡度门槛。正常坡面若被错误估成15 cm凸起，满足其他窗口时仍可能成为障碍；应先看地面估计与坐标，再按真实车辆和场地调整门槛。

上游当前近场正常高差输出受vehicleHeight=0.5 m限制，所以这里2 m上限通常不会进一步裁剪正常点。若改输入、上游规则或出现特殊标记点，需要重新判断。直接改接原始雷达点云后，intensity可能变回反射强度，不能沿用“0.1 m／2 m”的解释。

###### mark_threshold（数高度格，不数原始点）

VoxelGrid用位记录每个平面列的已标记高度格；已标记格数**严格大于**mark_threshold才返回可标障碍。当前0表示一格就够。设1则至少要两格，设2至少三格；同一高度格里再来很多点，不会增加高度格计数。[标记计数实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_voxel_grid/include/nav2_voxel_grid/voxel_grid.hpp#L99)

本层每次更新都会先重置二维图与体素，所以门槛统计来自本轮取得的观测内容；它不会靠连续多个更新周期逐渐凑够格数。但最新terrain_map本身包含上游历史点，不能把“一份最新观测”理解成一份纯实时原始扫描。

低矮物、薄障碍、稀疏回波可能只占一个高度格，提高门槛可能让它们消失；能否过滤噪声应按实物观察，不保证越大越稳。

###### track_unknown_space与unknown_threshold（层级错误和未使用的门槛）

自定义层调用ObstacleLayer::onInitialize，后者读的是**节点顶层track_unknown_space**；Costmap2DROS及StaticLayer也读该顶层键。当前层内True没有进入这条读取路径，顶层未配置时默认False，主图和障碍层默认按FREE_SPACE重置；静态图的未知值也会按该设置转为空闲。[父类读取位置](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/obstacle_layer.cpp#L79)、[静态未知值解释](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/static_layer.cpp#L263)

若要表达原配置“跟踪未知空间”的True意图，正确层级为：

```yaml
local_costmap:
  local_costmap:
    ros__parameters:
      track_unknown_space: true
```

此时应移除无效的层内同名键，重新核对未知区在规划与控制中的处理。若任务选择不跟踪未知，则应明确顶层False；**错误是当前写入层级，不是要求所有任务必须使用True。** 这里只提供修正说明，没有修改工程配置。

unknown_threshold另属自定义成员：初始化保存5+(16−z_voxels)，当前仍为5，但后续没有用它清体素射线、统计未知列或生成二维未知状态。调整5不会改善当前未知空间判定。[成员赋值与更新路径](https://github.com/SMBU-PolarBear-Robotics-Team/pb_nav2_plugins/blob/e6778ca/src/layers/intensity_voxel_layer.cpp)

即使顶层改True，本层也只是以未知默认二维格重置，再标出障碍和轮廓内空闲，不会因此自动得到标准VoxelLayer的完整射线空闲／未知推断。没有回波或点被过滤，不等于真实空地。

###### 每轮重建、轮廓清空与清地图服务（几种清除各有范围）

该自定义updateBounds先resetMaps，再拿marking observations重建障碍，最后清本层轮廓内二维代价。它不调用getClearingObservations或raytraceFreespace，直接把来源clearing改True也不会给当前实现补上射线清空。

footprint_clearing_enabled=True只清本层二维轮廓，不擦除已保存的体素标记；当前Max合成也不会用FREE_SPACE覆盖static_layer的静态墙。轮廓、点云自车滤除和静态地图错误应分开核对。

清地图服务还需区分：类的isClearable=False使按区域的ClearAround／ClearExcept服务跳过它；但**ClearEntirely调用resetLayers，会重置主图并调用所有层的reset**，包括本层。前面的BT使用的是ClearEntireCostmap，因此不能说它因为isClearable=False就完全不重置。重置也不丢弃上游地形缓存，下一轮相同观测／静态图仍可能把障碍加回来。[区域与整图服务](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/clear_costmap_service.cpp#L102)、[整图层重置](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_ros.cpp#L618)

本层内部重建不保证主图所有旧格立即变空；合成还受各层更新范围、滚动、静态占用和最新观测影响。动态残留应结合上游terrain_analysis衰减与实际话题观察。

###### combination_method（当前Max保留已有障碍）

当前1调用updateWithMax：本层未知格跳过；已知格可以替换主图未知格，或在代价更高时写入。FREE_SPACE不会降低前面静态层的已知障碍；不能简单把255未知当作普通数字最大值来推断全部合成结果。[Max合成](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_layer.cpp#L109)

0为Overwrite，可用本层非未知代价覆盖之前的图层；当前顶层缺省不跟踪未知、重置大量格为FREE_SPACE，改0可能擦掉静态层的已知障碍。其他整数在当前父类switch中不做合成；新版文档中的其他枚举不代表本版本可用。

#### intensity_voxel_layer.terrain_map（地形观测源）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `data_type` | `PointCloud2` | 不适用，消息类型名称 | **上实物前必须核对并正确配置** | 为来源建立PointCloud2订阅及缓冲 | 不适用 | 不适用 | 当前自定义迭代器需要float XYZ与intensity字段；父类可订阅LaserScan，不代表本层改类型后仍有需要的intensity |
| `topic` | `<robot_namespace>/terrain_map` | 不适用，话题名称 | **上实物前必须核对并正确配置** | 指定实际近场地形点云输入 | 不适用 | 不适用 | 工程bringup先展开占位符；通常为/red_standard_robot1/terrain_map；绕过launch时原字符串不会自行展开 |
| `sensor_frame` | `front_mid360` | 不适用，传感器原点系名称 | **上实物前必须核对并正确配置** | 取得量程筛选用的传感器原点 | 不适用 | 不适用 | 按点云时间戳查front_mid360至odom；点坐标仍按消息header.frame_id转换，不被强行当作雷达系点 |
| `min_obstacle_height` | `0.0` | m，odom中点z的来源下界 | **上实物前必须核对并正确配置** | 观测缓冲阶段先删除低于0的点 | 提前删除更多低处点 | 允许更多低处点进入本层，仍受本层及体素限制 | 与上一组同名但属独立阶段；两处均为0，边界可通过；只改本层下界不能恢复此处已删点 |
| `max_obstacle_height` | `2.0` | m，odom中点z的来源上界 | **上实物前必须核对并正确配置** | 观测缓冲阶段先删除高于2 m的点 | 来源窗口扩大，后续仍可删掉高点 | 降到体素有效窗口内才进一步减少当前可标记点 | 当前还受本层2 m和体素0.8 m上界限制；来源缺省上界为0.0，不自动继承本层2 m，不能随意省略；在odom中筛高度 |
| `obstacle_max_range` | `5.0` | m，相对雷达原点的三维距离 | **实物运行时重点观察，有问题再针对性调整** | 删除距离front_mid360原点达到／超过5 m的点 | 接纳更远已有点，仍受地形输出和网格边界限制 | 删除更多远点 | 当前距离包含z；等于上界也删除；不是局部地图宽度或底盘周围5 m圆 |
| `obstacle_min_range` | `0.2` | m，相对雷达原点的三维距离 | **上实物前必须核对并正确配置** | 删除距雷达原点不超过0.2 m的点 | 排掉更多近点，也可能漏掉近车障碍 | 保留更多近点，可能包含自车回波 | 当前自定义代码使用≤下界删除；不是robot_radius，也不是完整车体掩膜 |

##### 参数详解

###### topic、data_type与sensor_frame（点云坐标和测距原点分别取哪里）

bringup_launch先把<robot_namespace>替换为空或“/＋机器人命名空间”，随后参数重写和加载；当前配置通常订阅/red_standard_robot1/terrain_map。占位符没有在层源码中自行处理，直接启动原始参数文件时必须完成对应替换。

terrain_map的点坐标当前在odom中。ObservationBuffer按header.frame_id把点云转换到global_frame；另以sensor_frame指定的原点、同一消息时间戳取得odom中的雷达位置，再供量程计算。因此sensor_frame不会把odom点重新解释成雷达局部点。[观测缓冲换系与原点](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/observation_buffer.cpp#L83)

显式设置front_mid360后，消息过滤器还要求对应传感器系可变换。TF、时间域、命名空间和QoS都需要匹配；有地形话题不证明本层已成功接收缓冲。输入订阅采用SensorDataQoS，父类配置队列深度50；队列不是50 Hz或50 ms。

###### 来源高度与量程（高度在odom，距离从雷达量）

来源高度先在转换后的odom点坐标上筛选，并保留包括intensity在内的原点字段；本层再次检查自己的高度门槛。当前两级0～2 m相同，实际可标记的上界仍受0.8 m体素覆盖制约。

量程使用sqrt((x−x_sensor)²+(y−y_sensor)²+(z−z_sensor)²)，当前严格保留0.2～5 m之间的点。与二维扫描转换器按底盘虚拟中心计算水平距离不同，雷达安装偏移、云台转动和上下坡会影响这个三维距离。

近场地形输出和局部窗口本就有限。增大量程不会扩大5×5 m网格、不能恢复上游被删的点，也不会让MID360采样更远；最小量程也不能滤掉距雷达超过0.2 m的全部自车结构。

#### static_layer（静态二维地图接入）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `nav2_costmap_2d::StaticLayer` | 不适用，类名 | **通常保持当前值，不作为优先调试项** | 将订阅到的二维占用图采样到局部滚动网格 | 不适用 | 不适用 | 常规导航来源为map_server，建图时来源为slam_toolbox；读取地图话题，不直接读PCD或地图图片文件 |
| `map_subscribe_transient_local` | `True` | 不适用，QoS耐久性选择 | **上实物前必须核对并正确配置** | 用Transient Local、Reliable、KeepLast(1)订阅完整地图 | 不适用 | 不适用 | 发布端需兼容，才能取得保留的最近地图；不是保存地图到磁盘，也不是开启map_updates订阅 |

##### 参数详解

###### static_layer（map图仍要经过map↔odom变换）

静态层订阅完整OccupancyGrid，保存其尺寸、分辨率、原点和header.frame_id。滚动模式不会强行把主图变成整张静态地图的大小；它根据当前odom格子的世界位置，查询map_frame←odom的最新变换，到静态图中取样。因此local_costmap在odom中运行，仍需正确的全局定位变换。[滚动静态图采样](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/static_layer.cpp#L398)

当前顶层track_unknown_space缺省False，静态图未知格转为FREE_SPACE；已知墙仍保留。没有收到地图或对应TF查询失败时，点云这一路即使存在，也不能证明静态层已正常合成。当前controller_server还会等待代价地图current状态，地图输入和各层状态要一起看。

###### map_subscribe_transient_local（保留地图消息与更新话题各自独立）

True让较晚启动的订阅者可以从兼容发布端取得保留的最近完整地图。若发布端只提供Volatile耐久性或可靠性不兼容，不能靠延长TF等待取得地图；应先核对发布端与订阅端的QoS。[地图订阅QoS](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/static_layer.cpp#L70)

当前subscribe_to_updates缺省False，不订阅/map_updates；这不阻止处理随后收到的完整/map消息。“static”表示图层输入方式，不表示完整地图永远不能更新。

#### inflation_layer（障碍周围的二维代价）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `plugin` | `nav2_costmap_2d::InflationLayer` | 不适用，类名 | **通常保持当前值，不作为优先调试项** | 在已合成障碍附近写入距离相关的二维代价 | 不适用 | 不适用 | 当前位于三层最后，使静态墙和实时障碍都可成为膨胀源；不恢复被高度／高差筛掉的障碍 |
| `cost_scaling_factor` | `4.0` | m⁻¹，指数衰减系数 | **实物运行时重点观察，有问题再针对性调整** | 控制轮廓内圈之外的软代价随距离衰减 | 同样距离的外围代价更低、衰减更快；不加宽内圈 | 外围代价更高、衰减更慢，在既定半径内更宽地维持较高代价 | 不改0.7 m截断范围；当前PID没有使用1～252软代价评分，地图颜色变化不保证控制变化 |
| `inflation_radius` | `0.7` | m，距致命障碍格的膨胀范围 | **实物运行时重点观察，有问题再针对性调整** | 确定障碍周围多远仍写膨胀代价 | 代价影响范围扩大，计算增多，规划若使用梯度可能更早绕开 | 影响范围缩小；不能小到不足以表达所需轮廓空间 | 当前约14个5 cm格距；不是0.7 m全为禁止区，也不再自动加robot_radius成0.9 m |

##### 参数详解

###### cost_scaling_factor与inflation_radius（范围和代价下降速度）

内部代价值与发布后的OccupancyGrid占用值不同：

| 内部代价 | 含义 |
|---:|---|
| 0 | 空闲 |
| 1～252 | 非致命的软代价，越高通常越不希望经过 |
| 253 | 轮廓内切半径范围的高代价 |
| 254 | 致命障碍 |
| 255 | 未知状态 |

设d为距致命障碍格的实际平面距离、r_ins为**实际填充轮廓**的内切半径。障碍处为254，内圈为253，内圈之外到膨胀范围内按下面计算后取整数：

```text
cost = 252 × exp(−cost_scaling_factor × (d − r_ins))
```

所以4变大时，外围代价更快降低；变小时，较高软代价维持得更远。0.7决定影响范围，4决定范围内的衰减，不能把两个参数当同一作用。[代价公式（1.1.20）](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/include/nav2_costmap_2d/inflation_layer.hpp#L148)

仅为说明公式，假设r_ins=0.2 m、d=0.4 m：当前4对应约113，改为8约50，改为2约169。实际轮廓含padding、栅格化和最近障碍距离，不能把这个假设值当运行读数。

0.7 m以内并非全部致命或253，也不是保证车体距墙至少0.7 m。轮廓定义、真实包络、地图误差和路径使用方式都影响实际通过距离。

###### 当前PID怎样使用这些代价（软梯度不等于已经参与避障评分）

当前OmniPidPursuitController从局部路径抽10个点，检查落格代价是否达到INSCRIBED_INFLATED_OBSTACLE，即≥253；它不按1～252的软梯度给候选轨迹打分。改变局部cost_scaling_factor或扩大外围软代价，可能改变RViz颜色，却没有直接改变本插件的碰撞通过条件。

robot_radius及padding改变的内切半径会影响253内圈；静态墙／实时障碍本身也参与高代价检查。全局路径又由另一张global_costmap规划，其膨胀设置需下一块单独核对，不能把两张地图的参数影响混为一谈。[当前控制器地图使用方式](/wiki/2026-09-20-dao-hang/0500-1100-controller-server)

前文还记录了抽样数量及地图外点的处理缺口。这里的膨胀范围不是完整车体扫掠碰撞检查的替代，也不补回本障碍层遗漏的高度点。

#### 相关源码默认项（原YAML未显式配置）

##### 参数表

以下12项补充真正读取的节点顶层设置、观测缓冲与未知膨胀行为；不表示本次已向工程YAML追加。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `track_unknown_space` | `False`，节点顶层源码默认 | 不适用，布尔 | **上实物前必须核对并正确配置** | 决定主图、障碍层默认未知表示及静态未知格解释 | 不适用 | 不适用 | 真正被读取的键；与前面层内未生效的True不同，需明确实际未知区策略 |
| `footprint` | `"[]"`，源码默认 | 不适用，轮廓顶点字符串 | **上实物前必须核对并正确配置** | 指定显式多边形轮廓，空值时使用圆半径 | 不适用 | 不适用 | 当前用robot_radius；有效多边形会优先使用；还可通过footprint话题更新，须核对参考系 |
| `footprint_padding` | 约`0.01`，源码默认 | m，轮廓填充量 | **上实物前必须核对并正确配置** | 在轮廓外增加填充，影响内切半径和清轮廓区域 | 有效轮廓增大 | 有效轮廓减小 | 不是补偿未量出的完整车体尺寸；按顶点实现，不能简单保证圆半径恰好加0.01 |
| `transform_tolerance` | `0.3`，节点顶层源码默认 | s，TF查询等待 | **通常保持当前值，不作为优先调试项** | 供机器人位姿、观测缓冲和静态层等取得变换 | 可等待更久，也可能增加延迟 | 更快结束缺失TF查询 | 不允许0.3 rad／m误差，也不补偿外参；运行中改顶层值不会自动更新已构建的所有缓冲成员 |
| `intensity_voxel_layer.terrain_map.marking` | `True`，父类默认 | 不适用，布尔 | **通常保持当前值，不作为优先调试项** | 将此来源加入标记观测列表 | 不适用 | 不适用 | 当前自定义层使用该标记列表；关闭后此来源不再贡献障碍 |
| `intensity_voxel_layer.terrain_map.clearing` | `False`，父类默认 | 不适用，布尔 | **当前运行不使用这个参数，暂不调整** | 父类可把来源加入清空列表，本自定义更新未使用该列表 | 不适用 | 不适用 | 改True不会自动开启本层射线清空；对应raytrace量程也不会因添加配置就参与当前处理 |
| `intensity_voxel_layer.terrain_map.observation_persistence` | `0.0`，父类默认 | s，观测缓存保留设置 | **通常保持当前值，不作为优先调试项** | 决定缓存一份还是一定时间内的多份观测 | 正值时可能保留更多历史观测，动态残留增多 | 正值减小时历史窗口变短；0另走特殊分支 | 0表示只留最后一份，不表示最后一份在0 s后过期；不清上游terrain_map历史 |
| `intensity_voxel_layer.terrain_map.expected_update_rate` | `0.0`，父类默认 | s，期望消息间隔 | **实物运行时重点观察，有问题再针对性调整** | 正值时用于判断观测是否按期更新 | 允许更长消息间隔后仍算current | 正值变小使停更更快被判断；0关闭这项检查 | 名字虽带rate，代码按秒建Duration；当前停更不因这项自动变为不current |
| `static_layer.map_topic` | `""`，层私有默认；转用节点默认机器人namespace下的`map` | 不适用，话题名称 | **上实物前必须核对并正确配置** | 指定完整静态图话题 | 不适用 | 不适用 | 当前通常/red_standard_robot1/map；不是/red_standard_robot1/local_costmap/map；以实际namespace／重映射为准 |
| `static_layer.subscribe_to_updates` | `False`，源码默认 | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否另外接收map_topic加_updates的增量图 | 不适用 | 不适用 | False仍接收后续完整地图；与地图Transient Local订阅设置独立 |
| `inflation_layer.inflate_unknown` | `False`，源码默认 | 不适用，布尔 | **根据实际功能需求开启或设置** | 控制已知障碍膨胀到未知格时是否写入软代价 | 不适用 | 不适用 | False也可能在未知格写入≥253的内圈代价；不等于把未知格都当膨胀源 |
| `inflation_layer.inflate_around_unknown` | `False`，源码默认 | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否将未知格本身也作为膨胀起点 | 不适用 | 不适用 | 当前不主动围绕未知格膨胀；与上一项是不同步骤，顶层未知策略变化后需重新核对 |

##### 观测停更与默认未知状态

observation_persistence=0只删除比最新观测更旧的观测，保留最后一份；expected_update_rate=0直接将缓冲判为current。于是点云停止到达时，本层可能继续每轮重建同一份旧地形，而不是自动清空障碍或停止控制。还需结合静态层、其他层状态和完整系统的数据监测判断，不能只看地图话题仍发布。[缓存保留与current判断](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/observation_buffer.cpp#L187)

如果以后设置正的expected_update_rate，按秒的消息间隔理解，再核对正常数据周期与延迟；不能填10就当成10 Hz。只改变buffer保留时间也不去除terrain_map内部已经累计的历史点。

#### 动态参数、启动来源与验证范围

##### 动态修改的实际范围

自定义层没有另写覆盖体素、高差和发布器的动态回调，使用父类ObstacleLayer的回调；因此运行中参数设置成功，不代表所有成员已经更新。

| 参数类别 | 当前实现 | 应怎样核对 |
|---|---|---|
| robot_radius、footprint_padding、footprint、publish_frequency | 节点回调会更新相应轮廓或发布门槛 | 核对实际轮廓、膨胀内圈及地图话题；动态轮廓须符合当前参考系 |
| width、height、resolution | 当前未锁尺寸时回调重建网格，并调用各层matchSize | 注意观察缓存与范围变化；width／height采用正整数，resolution为正double |
| update_frequency、global_frame、rolling_window、plugins、track_unknown_space、always_send_full_costmap | 当前没有完整动态联动更新路径 | 重新配置／重启后核对有效行为，不仅看参数列表 |
| intensity_voxel_layer.enabled、footprint_clearing_enabled、两项层高度、combination_method | 父类回调更新对应成员 | 分支改变后核对实际合成；层高度变了不会同时重建来源高度过滤 |
| origin_z、z_resolution、z_voxels、mark_threshold、unknown_threshold、两项intensity、publish_voxel_map | 当前自定义层无对应动态更新分支 | 重新配置／重启；unknown_threshold重启后也仍无当前使用路径 |
| terrain_map的话题、类型、原点系、来源高度、量程、缓存及期望间隔 | 初始化时建订阅器和ObservationBuffer | 重新配置／重启，不能只修改成员外的同名ROS值 |
| static_layer地图话题、Transient Local、subscribe_to_updates | 当前回调明确拒绝这类运行中修改 | 修改配置并重新启动订阅后验证 |
| inflation_radius、cost_scaling_factor及未知膨胀开关 | 膨胀层回调更新、必要时重算缓存并重新膨胀 | 核对实际代价图及当前消费者是否使用软梯度 |

顶层transform_tolerance有成员更新分支，但已经创建的来源缓冲、消息过滤器和静态层各有初始化时保存的值；不能认为一次改值让所有TF等待同步变化。[节点回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/src/costmap_2d_ros.cpp#L661)、[父类层回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/obstacle_layer.cpp#L289)

##### 启动与核对范围

| 启动方式 | 本块来源与关系 |
|---|---|
| 整套reality | config/reality/nav2_params.yaml，经bringup替换topic占位符、navigation_launch重写时钟并加载 |
| 整套simulation | config/simulation/nav2_params.yaml，本块仅use_sim_time=True不同 |
| 组件／独立进程 | local_costmap由controller_server内部创建，读取同套命名空间下的节点参数；不是这份YAML自己启动一个进程 |
| slam=True | 静态层接建图产生的完整/map；实时地形层仍用terrain_map |
| slam=False | 静态层接map_server的/map；实时地形层仍用terrain_map，地图定位依赖small_gicp等提供map↔odom |

- **本次已核对：** 当前包版本、插件注册与liblayers.so存在；自定义层安装头文件与本地一致，本地实现与上游固定提交一致；Nav2的体素、合成与膨胀安装头文件匹配1.1.20；两份local_costmap段与安装副本一致，仿真仅时钟不同。
- **源码能确认：** 40项原配置的读取／使用条件；层内未知开关位置无效、unknown_threshold未参与计算；当前odom垂直标记区间[0,0.8) m；按不同高度格计标记；每轮重建但不射线清空；区域与整图清理不同；默认旧观测复用、膨胀方向和全图／增量发布条件。
- **本次验证范围：** 未启动新的地图或导航任务，未观察运行节点有效参数、坡道点、未知区、动态障碍消失时间或车旁碰撞；窗口、体素和膨胀例子是配置／源码计算。
- **实物待核实：** odom地面高度基准、上下坡与高低障碍是否过窗口、真实车体轮廓与虚拟参考点、点云字段与TF时间、观察源停更处理、静态图对齐／QoS、未知区策略、动态残留以及控制器实际地图使用。

