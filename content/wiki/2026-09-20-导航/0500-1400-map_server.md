---
title: "map_server（已有二维地图加载与发布）"
---

## map_server（已有二维地图加载与发布）

### 实车先看：空路径由谁填写，读的是哪一张地图

**map_server读取已有二维地图的YAML及配套图片，发布OccupancyGrid给静态代价层；当前工程通过launch的map参数填写yaml_filename。** 因此这里的空字符串是启动时重写的占位值，不能单看这行就判断没有地图。

```text
实物／仿真入口：map启动参数
    → bringup_launch → localization_launch
    → 重写map_server.ros__parameters.yaml_filename
    → map_server：读取地图YAML → 找到配套PGM／PNG → 转为二维占用网格
    → 配置、激活生命周期节点 → 发布namespace/map
    → local_costmap、global_costmap中的static_layer

prior_pcd_file → small_gicp：用实时三维点云匹配先验PCD，提供定位TF
```

二维图加载与三维PCD定位是不同的输入链。map_server不读取PCD，不把当前雷达点云拿去匹配二维图片，也不自行产生map→odom定位变换。两种地图仍须采用一致的尺度、方向和坐标约定。[二维图与PCD的分工](/wiki/2026-09-20-dao-hang/0500-0900-slam-toolbox)、[先验点云定位](/wiki/2026-09-20-dao-hang/0500-0700-small-gicp-relocalization)

| 实车上看到的现象 | 先核对什么 | 当前配置怎样理解 |
|---|---|---|
| YAML路径为空，却能够加载地图 | launch传入的map及有效yaml_filename | 当前重写流程会替换已有空值；实际路径来自启动参数 |
| 修改nav2_params.yaml中的路径，启动后仍用旧图 | 上层map启动参数 | 当前launch总会用map值重写该键；单改参数文件中的字符串可能被覆盖 |
| 实物入口默认启动失败，提示找不到地图 | 实物默认地图文件是否存在 | 当前默认map/reality/rmul_2024.yaml在源码和安装目录均不存在，需提供自己的有效二维图 |
| 节点启动了，/map却没有地图消息 | 最终路径、文件读取及生命周期状态 | 有节点／发布器不等于已经加载并激活地图；空路径可以配置，但不会自动产生地图 |
| 建图时没看到map_server | slam启动分支 | slam=True使用slam_toolbox发布/map；此节点在slam=False的定位分支启动 |
| 图片看起来灰色，加载后却成了空闲 | 地图文件的mode、free_thresh与像素灰度 | 保存和读取的门槛不是同一计算；本版本三态灰205与0.25的组合有已验证的未知状态丢失问题 |

本次安装的nav2_map_server为**1.1.20**，安装的map_server／map_io头文件与该版本一致。以下按此版本及当前工程启动文件说明。[MapServer实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_map_server/src/map_server/map_server.cpp)

### 参数

保留发送的参数与上方原注释。注释中的tb3_simulation_launch.py是通用示例名称；当前工程实际使用rm_navigation_reality_launch.py、rm_navigation_simulation_launch.py、bringup_launch.py和localization_launch.py。

```yaml
# The yaml_filename does not need to be specified since it going to be set by defaults in launch.
# If you'd rather set it in the yaml, remove the default "map" value in the tb3_simulation_launch.py
# file & provide full path to map below. If CLI map configuration or launch default is provided, that will be used.
map_server:
  ros__parameters:
    yaml_filename: ""
```

#### ros__parameters（地图文件入口）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `yaml_filename` | `""`；整套启动时由map参数重写 | 不适用，地图YAML文件路径字符串 | **上实物前必须核对并正确配置** | 指定初始加载的二维地图元数据文件 | 不适用 | 不适用 | 当前可保留空值占位，通过map:=实际绝对路径配置；文件与图片均需存在；直接启动并留空则不加载初始地图 |

##### 参数详解

###### yaml_filename（节点参数文件与地图文件是两种YAML）

nav2_params.yaml是ROS节点参数文件，里面有map_server、map_saver、costmap等参数分组。yaml_filename指向的是**一张二维地图的元数据YAML**，其中写image、resolution、origin等；不能把它设成nav2_params.yaml本身，也不能直接指向PGM、PNG或PCD。

1.1.20在configure阶段读取yaml_filename。非空时读取地图YAML和图片，失败会使配置过程失败；空字符串则跳过初始加载，建立话题与服务，允许之后通过load_map服务装入地图。空字符串不表示自动建图、不表示读取当前目录的某张图，也不表示使用PCD替代二维图。[初始加载与空路径分支](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_map_server/src/map_server/map_server.cpp#L81)

当前安装的RewrittenYaml只替换已有叶子键，bare key形式的yaml_filename重写不会自动补入缺失键。**保留yaml_filename这一行，空值可交给launch填写；不能按注释把整个键删掉。** MapServer源码声明的是字符串类型而不是一个自动场地路径默认值，路径默认来自工程launch。[1.1.20参数重写](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_common/nav2_common/launch/rewritten_yaml.py#L108)

###### 当前launch怎样决定最终地图

| 启动位置 | map参数来源 | 对yaml_filename的作用 |
|---|---|---|
| rm_navigation_reality_launch.py | 用户map:=路径；未指定时按world生成share/map/reality/场地名.yaml | 传给bringup；当前world缺省rmul_2024 |
| rm_navigation_simulation_launch.py | 用户map:=路径；未指定时按world生成share/map/simulation/场地名.yaml | 传给bringup；当前world缺省rmuc_2025 |
| bringup_launch.py | 接收上层map | 用RewrittenYaml重写已有yaml_filename，再传给定位分支 |
| localization_launch.py | 接收map，单独启动时此参数无默认值 | 再用map重写yaml_filename，独立进程与组件两种分支均加载结果 |
| 直接启动map_server并提供节点参数 | 用户传入yaml_filename | 未经过上述重写时，直接使用节点得到的值；生命周期另需配置和激活 |

**当前整套启动以map启动参数为准。** 想改用自己的地图，优先明确传map:=地图YAML的绝对路径；修改nav2_params.yaml中的值并不能阻止两处重写。若要改成完全由节点参数文件决定，必须调整当前bringup与localization的重写逻辑，并核对上层必需参数；不是只去修改注释提到的TB3文件。本文只记录接线，没有修改启动文件。

实际重写位置见[bringup_launch.py](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/launch/bringup_launch.py:55)、[localization_launch.py](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/launch/localization_launch.py:47)。

###### 本次默认文件检查（实物默认图尚未提供）

当前安装包share目录为：

```text
/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/install/pb2025_nav_bringup/share/pb2025_nav_bringup
```

| 默认入口 | 默认相对share目录的路径 | 本次静态检查 |
|---|---|---|
| 实物，world=rmul_2024 | map/reality/rmul_2024.yaml | 源码和安装副本均没有该文件；当前源码map/reality仅有reserve占位文件 |
| 仿真，world=rmuc_2025 | map/simulation/rmuc_2025.yaml | 源码和安装副本均存在，配套rmuc_2025.pgm也存在 |

这是对当前这份工作空间和默认入口的文件检查，不表示其他工作空间没有实物图，也不表示用户曾经指定的map路径失败。使用自己建的二维图时，还要给prior_pcd_file选择与之坐标一致的PCD，不能只让两份文件名看起来相同。

当前仿真默认地图元数据为：

```yaml
image: rmuc_2025.pgm
mode: trinary
resolution: 0.05
origin: [-3.58, -9.44, 0]
negate: 0
occupied_thresh: 0.65
free_thresh: 0.25
```

这里只引用已有地图文件说明格式，不是map_server的额外ROS参数。原文件见[rmuc_2025.yaml](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/map/simulation/rmuc_2025.yaml)。

###### 地图元数据（图片如何变成真实坐标和占用值）

| 地图文件字段 | 单位／含义 | 当前应怎样核对 |
|---|---|---|
| image | 配套图片路径 | 相对路径按地图YAML所在目录解析；搬移时需同时保留图片，不能只复制YAML |
| resolution | m／图片像素 | 决定真实平面格距；此例5 cm，不是定位精度；修改它会改变整图物理尺度 |
| origin | [x米、y米、yaw弧度] | 第三个数是绕z的偏航角，**不是z高度**；载入的二维原点z设为0 |
| negate | 0／1，像素明暗是否反转 | 此例0：深色表示较高占用；不是把地图坐标轴反转 |
| mode | trinary／scale／raw | trinary按阈值输出0、100、-1；scale还表达中间值；raw按像素数直接解释，不能随意换模式后继续按普通黑白图理解 |
| free_thresh、occupied_thresh | 无量纲，像素换算后的占用门槛 | 加载时读的是本地图文件的值，不读取map_saver节点的default门槛 |

普通无透明通道灰度图、negate=0、trinary时，令灰度g为0～255，则占用比例为1−g/255；严格大于occupied_thresh设100，严格小于free_thresh设0，其余设-1。保存时对输入0～100占用值采用另一套包含边界的比较，不能把两种门槛比较混写。[元数据解析与像素加载](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_map_server/src/map_io.cpp#L146)

**本版本默认三态保存的未知灰205重新读取时，1−205/255≈0.19608，小于0.25，会变成空闲0。** 本次已直接用安装的libmap_io在临时文件中验证：输入未知-1，用trinary、0.25／0.65保存再读取得到0；free改0.196后该未知格仍为-1。这个问题只针对满足上述读写条件的图片，不能据此断言所有来源的地图都丢未知。完整阈值条件、验证数据和处理建议见[map_saver](/wiki/2026-09-20-dao-hang/0500-1500-map-saver)。

###### 与global_costmap和三维定位的几何关系

前面global_costmap默认不滚动，StaticLayer会按/map元数据调整主图尺寸、格距和平面原点；这里加载哪张图，直接决定它的有效网格，而不是由costmap中的初始0.05独立决定。[全局图的尺寸接管](/wiki/2026-09-20-dao-hang/0500-1300-global-costmap)

MapServer把origin第三个数写入地图四元数，但当前Nav2 1.1.20的StaticLayer平面网格主要使用origin.position.x／y与格距，非滚动合成按同格索引，不按任意origin yaw重新旋转网格。当前示例yaw=0；若将地图YAML改为非零yaw，应单独核对消费者，不能认为所有节点都会自动正确旋转。[静态层网格建立](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_costmap_2d/plugins/static_layer.cpp#L179)

frame_id只是消息坐标系名称。给图片改原点、旋转、裁剪或重采样后，需同时考虑PCD定位和map↔odom；仅把frame_id写成map不证明实际对齐。

#### 相关源码默认项（原YAML未显式配置）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `topic_name` | `map`，MapServer默认 | 不适用，话题名称 | **上实物前必须核对并正确配置** | 选择完整OccupancyGrid发布话题 | 不适用 | 不适用 | 相对名称按节点namespace解析；例如/red_standard_robot1/map；需与两张costmap的static_layer输入一致 |
| `frame_id` | `map`，MapServer默认 | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 设置地图消息header.frame_id | 不适用 | 不适用 | 不创建TF、不变换图片或原点；当前应与全局网格、PCD定位约定一致 |
| `use_sim_time` | 原分组未写；标准ROS默认False | 不适用，布尔 | **上实物前必须核对并正确配置** | 决定MapServer填写加载时间和消息时间戳时使用的时钟 | 不适用 | 不适用 | 当前bare-key重写只改已有键，没有给本分组补入此键；仿真不能只凭入口True推断本节点也True，外部覆盖需另查 |

两份节点参数文件的map_server段完全一致，均只有yaml_filename空字符串。当前默认接线未在此分组补入use_sim_time；在没有其他覆盖时标准默认为False，仿真入口传True不会经该重写自动补到这里。此结论来自参数及启动源码，未对运行节点取快照；需要统一时间域时应显式配置，再检查有效参数。[加载后的时间戳](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_map_server/src/map_server/map_server.cpp#L245)

#### 生命周期、换图与验证范围

##### 发布与服务

MapServer是生命周期节点。configure时加载初始图并建立服务／发布器；activate时发布已加载完整图。当前localization_launch的lifecycle_manager_localization管理map_server，autostart=True时自动配置、激活；单独运行可执行程序不能只停在“进程已创建”。

完整地图使用Transient Local、Reliable、KeepLast(1)，保留最近一张图供兼容的晚加入订阅者读取。它不是周期性重读磁盘或定时发布新地图；原文件改了不会自动热更新。[地图发布与激活](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_map_server/src/map_server/map_server.cpp#L119)

| 接口 | 默认名称，带机器人namespace时示例 | 行为 |
|---|---|---|
| 完整地图话题 | /red_standard_robot1/map | 初始激活有图时发布；成功load_map后发布新图；保留最近完整图 |
| GetMap服务 | /red_standard_robot1/map_server/map | 返回当前内存OccupancyGrid；Active状态才处理；Active但尚未加载图时不能当作已有有效地图 |
| LoadMap服务 | /red_standard_robot1/map_server/load_map | 请求map_url加载地图YAML；Active状态才处理，成功后更新内存并发布新图 |

当前MapServer没有为yaml_filename、topic_name、frame_id建立运行中重读／重建的参数回调。参数设置成功不代表换图成功；换图应走LoadMap服务或重新配置／重启。服务加载成功也不自动更换small_gicp的目标PCD，切换场地时还要同步定位目标与坐标约定。[服务加载](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_map_server/src/map_server/map_server.cpp#L207)

##### 本次核对与后续实物检查

- **本次已核对：** nav2_map_server 1.1.20及安装头文件；两套节点参数与安装副本一致；五份相关launch与安装副本一致；实际map重写接线；实物默认地图缺失、仿真默认YAML／图片存在。
- **本次离线验证：** 在临时目录直接调用已安装地图IO库完成小地图保存／读取，确认trinary灰205、free=0.25会丢未知身份，0.196的对应例子保留；未启动ROS地图节点、话题或服务，未写入工程地图目录。
- **源码能确认：** 空路径分支、configure／activate行为、完整图QoS、Active服务条件、相对图片路径解析、地图元数据含义，以及参数重写与运行中换图的边界。
- **实物待核实：** 启动时实际map与有效yaml_filename、二维图／PCD对齐、图中未知区域保存和重新加载、时间域、生命周期状态及静态层的有效地图元数据；本次未运行整套导航或进行实车验证。
