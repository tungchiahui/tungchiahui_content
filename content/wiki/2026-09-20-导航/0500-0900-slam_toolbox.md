---
title: "slam_toolbox（二维地图生成与轨迹优化）"
---

## slam_toolbox（二维地图生成与轨迹优化）

### 二维图、PCD与实时定位：各自负责什么

**当前工程的二维栅格地图由slam_toolbox生成，三维PCD由Point-LIO保存。** 二维地图记录每个平面格子的占用／空闲／未知状态；PCD保留三维点的位置，两者用途不同。

建图时的两条输出链为：

```text
Point-LIO配准点云
    ├─ 累计并保存三维PCD
    └─ loam_interface → registered_scan
           → terrain_analysis / terrain_analysis_ext（估计地面与高差）
           → pointcloud_to_laserscan（筛选、转到底盘系、压成二维扫描）
           → obstacle_scan → slam_toolbox → /map（二维栅格）
```

pointcloud_to_laserscan提供二维扫描，slam_toolbox把扫描放到地图上并生成栅格；前者本身不保存二维地图。当前扫描来自地形点云，不是MID360原始回波直接截一层。

| 应保留的文件 | 用途 | 当前方案是否需要 |
|---|---|---|
| 二维地图图片（如PGM／PNG）及配套YAML | map_server读取；给Nav2提供静态道路／障碍布局 | **需要成套保留**；YAML记录图片路径、分辨率、原点等，只有图片不够 |
| 对应场地的三维PCD | small_gicp_relocalization的定位目标点云 | **需要保留**；不能用二维图片代替；当前launch通过prior_pcd_file指定 |
| slam_toolbox序列化的.posegraph及.data | 保留其内部建图状态与扫描，供继续建图或使用它自己的定位模式 | 当前常规导航不读取；如需后续继续建图，可额外成套保存，仍需验证当前设置下的可用性 |

序列化接口分别写出.posegraph和.data；它们与二维图片、三维PCD是不同的保存结果。[slam_toolbox序列化实现（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/include/slam_toolbox/serialization.hpp)

**开启保存功能不等于文件已经落盘。** use_map_saver=True提供地图保存服务；当前slam_launch还启动了Nav2的map_saver_server，但launch没有自动调用保存服务，二维地图仍需执行保存操作。[地图保存功能说明（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/README.md#toolbox-params)

PCD这一路，集成slam_launch把`pcd_save.pcd_save_en`覆盖为True；当前interval=-1，且scan_publish_en=True，点云持续累计，在程序正常退出的保存分支写出scans.pcd。异常强制结束可能来不及保存，长时间累计还会增加内存占用；新一轮保存可能覆盖同名文件，需要按场地和轮次归档。

当前安装的Point-LIO程序中，编译目录对应的保存位置为：

```text
/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/point_lio/PCD/scans.pcd
```

保存目录由编译时ROOT_DIR确定，详见[Point_LIO的PCD保存说明](/wiki/2026-09-20-dao-hang/0500-0100-point-lio)；在其他路径重新编译后需要重新核对。文件不会自动成为导航launch指定的场地PCD。

已有地图导航（`slam:=False`）的关系为：

```text
保存的二维图片 + YAML → map_server → 静态地图 → Nav2规划
实时registered_scan + 保存的三维PCD → small_gicp → map与odom之间的定位变换
实时terrain_map / terrain_map_ext → 局部／全局代价地图 → 更新当前障碍
```

因此，**导航时会读取二维图和PCD；实时匹配的是“当前三维点云与保存的三维PCD”。** 不是把PCD与二维图片互相匹配，也不是每帧先存一份PCD再匹配。当前small_gicp按约0.5 s的定时器处理累计点云，只有匹配收敛才更新结果；这不保证每0.5 s都得到正确定位。常规导航不启动slam_toolbox和这个二维扫描转换节点，也没有用它们进行实时二维地图匹配。接线见[slam_launch.py](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/launch/slam_launch.py:126)、[localization_launch.py](/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav/src/pb2025_sentry_nav/pb2025_nav_bringup/launch/localization_launch.py:135)。

二维图和PCD应来自同一场地、采用一致的尺度、原点和方向；同一轮保存也不能自动证明两者对齐。Point-LIO输出内部IMU位姿，PCD处于camera_init中；后续接口还有IMU／雷达／底盘转换和先验PCD启动变换的疑点，应按前面loam_interface、small_gicp章节核对。改二维图片的大小、原点或旋转方向时，也必须考虑PCD定位坐标，不能只看图片外观。

### 上下坡能否建图，是否需要PS改图

**坡道可以在二维图上留下平面布局，但二维栅格不记录坡面高度、坡度或车辆爬坡能力。** 单层地图也不能分别表达同一平面位置上下两层都可通行的结构。

正常坡面如果被地形节点正确认作地面，高差应接近0，通常被当前`min_intensity=0.1 m`筛掉；墙、台阶等明显高差点才进入二维障碍轮廓。坡面被筛掉，并不保证整条坡道已经标成可通行：仍取决于其他有效回波、扫描射线覆盖，以及下游对空闲／未知区域的处理。

例如，正常坡点被错误估成15 cm高差，且通过其他窗口时，会越过10 cm门槛并可能画成障碍；下坡点若在base_footprint系中低于-0.5 m，又会被min_height删掉。这分别涉及地面估计和投影高度窗口，不能靠同一项参数处理。

| 坡道问题 | 应先核对什么 |
|---|---|
| 连续坡面被画成墙或障碍带 | 地形节点的地面估计、useSorting／quantileZ，以及进入二维扫描的高差门槛；两套地形节点在近远场的正负高差处理不同 |
| 下坡缺点、坡道轮廓不完整 | 上游高度窗口、底盘投影系的方向，以及转换器固定min_height／max_height范围 |
| 实物地面整体倾斜、转云台后地图变形 | 重力方向、IMU／雷达／底盘外参、机械关节角与点时间；应先排除接口问题，再调筛选门槛 |
| 二维图看着通畅，导航仍绕开坡道 | 实时代价地图也接收地形点云；静态地图可通行不代表实时障碍层允许通过 |
| 识别正确，车辆仍无法上下坡 | 实车坡度、抓地力、离地间隙、坡顶／坡底过渡与制动能力；地图参数不能提高机械能力 |

调参的目标是让真实地面、障碍和缺数据区得到合理表达；不能为了让坡道变白而把真实台阶或危险落差一起过滤。实际可通行的坡度和落差需要测量并验证，当前没有实物坡道结论。

PS等图片编辑可以用于经过现场确认后的静态地图清理，**不是必须步骤，也不能修复实时地形识别。** 即使把静态图上的误障碍擦白，实时terrain_map／terrain_map_ext仍可能把同一坡面重新加入代价地图。优先核对坐标、单位、时间和上下坡筛选，再评估是否需要人工修图。

### 当前建图方式与配置来源

整套reality／simulation入口把各自`config/reality/nav2_params.yaml`、`config/simulation/nav2_params.yaml`传给slam_launch，启动`sync_slam_toolbox_node`；这6项在两份配置中一致。默认通过已安装的pb2025_nav_bringup包目录寻找配置，指定params_file时以指定文件为准；当前安装副本与源码对应配置一致。

当前安装版本为slam_toolbox **2.6.10**、Ceres **2.0.0**。以下读取和生效条件按对应版本复核；配置里的humble分支链接可供查说明，但分支内容可能继续变化。

另有启动配置疑点：两份YAML的slam_toolbox段都没有显式use_sim_time。当前安装的RewrittenYaml只替换已有键，而slam_launch没有给此节点单独传入该参数；按当前默认接线，仿真入口传入True也不会自动补到此节点，标准默认仍为False。这里是静态配置判断，实际节点参数尚未观察；ROS参数分区需继续核对这一项。

**当前use_scan_matching=False、do_loop_closing=False。** 从空地图进行常规建图时，Karto仍接收扫描并形成栅格，但跳过扫描匹配，也不把新扫描加入位姿图；因此不会通过自动闭环调用Ceres修正轨迹。这里的图主要依靠上游里程计摆放扫描，长距离、转弯后的累计误差不能指望这组求解器设置自动消除。[当前处理路径（Mapper.cpp，2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

求解器插件仍会在节点配置时加载并读取参数；**“已读入”与“参与了实际优化”是两件事。** 加载已有位姿图或通过其他功能触发优化，应再按对应路径判断。这些开关决定本组求解器设置是否有实际优化任务。

原文件的Plugin、ROS、General、Correlation及Scan Matcher分区只是注释分类；参数仍直接写在ros__parameters下面，不增加一层算法参数键。

### 参数

#### Plugin params（位姿图求解器设置）

##### 参数表

```yaml
slam_toolbox:
  ros__parameters:
    solver_plugin: solver_plugins::CeresSolver
    ceres_linear_solver: SPARSE_NORMAL_CHOLESKY
    ceres_preconditioner: SCHUR_JACOBI
    ceres_trust_strategy: LEVENBERG_MARQUARDT
    ceres_dogleg_type: TRADITIONAL_DOGLEG
    ceres_loss_function: None
```

这6项都是类型名称，没有米、秒或角度单位，也不存在数值意义上的“调大／调小”。

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `solver_plugin` | `solver_plugins::CeresSolver` | 不适用，插件名称 | **通常保持当前值，不作为优先调试项** | 选择二维位姿图的优化后端 | 不适用 | 不适用 | 启动时加载；当前安装的solver_plugins.xml只注册了Ceres，插件名称错误或不可用会导致加载失败 |
| `ceres_linear_solver` | `SPARSE_NORMAL_CHOLESKY` | 不适用，算法枚举 | **通常保持当前值，不作为优先调试项** | 选择每轮图优化中求解线性方程的方法 | 不适用 | 不适用 | 当前为稀疏直接求解；只有实际调用Ceres优化时影响计算；当前常规建图不触发自动图优化 |
| `ceres_preconditioner` | `SCHUR_JACOBI` | 不适用，算法枚举 | **当前运行不使用这个参数，暂不调整** | 为适用的迭代线性求解方法提供加速方式 | 不适用 | 不适用 | 参数被读取，但当前SPARSE_NORMAL_CHOLESKY不使用预条件器；切换算法后须核对组合是否支持 |
| `ceres_trust_strategy` | `LEVENBERG_MARQUARDT` | 不适用，算法枚举 | **通常保持当前值，不作为优先调试项** | 决定图优化每轮如何提出、调整修正步长 | 不适用 | 不适用 | 当前使用LM；另一暴露选项为DOGLEG；实际优化触发后才影响求解，不控制车辆运动 |
| `ceres_dogleg_type` | `TRADITIONAL_DOGLEG` | 不适用，算法枚举 | **当前运行不使用这个参数，暂不调整** | 选择Dogleg策略内部的求解方式 | 不适用 | 不适用 | 仅ceres_trust_strategy=DOGLEG时使用；当前LM下不参与计算 |
| `ceres_loss_function` | `None`，字符串 | 不适用，类型名称 | **通常保持当前值，不作为优先调试项** | 决定图约束误差是否采用鲁棒损失，减弱异常约束的影响 | 不适用 | 不适用 | None采用普通平方损失；只有图约束进入优化后才影响结果；不是关闭地图保存、扫描匹配或整个SLAM |

枚举含义见[官方求解器参数说明（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/README.md#solver-params)；预条件器适用于迭代线性求解的条件见[Ceres 2.0.0选项定义](https://github.com/ceres-solver/ceres-solver/blob/2.0.0/include/ceres/solver.h)。

##### 参数详解

###### solver_plugin（选择二维轨迹优化后端）

“位姿图”可理解为：记录各帧扫描时车辆的位置、朝向，以及帧与帧之间的相对关系。求解器在有这些约束且触发优化时，调整二维x、y、yaw，使约束更一致。它不优化Point-LIO的三维IMU状态，也不替代small_gicp的PCD定位。

当前Ceres插件在启动阶段仍需成功加载。官方说明提到的其他插件名称，不代表当前安装包已经提供；不能只改名字就假定能切换。

###### ceres_linear_solver（图优化内部怎样解方程）

SPARSE_NORMAL_CHOLESKY可理解为利用方程中大量零项，直接求出一轮修正量。其他暴露选项为SPARSE_SCHUR、ITERATIVE_SCHUR、CGNR，它们是算法选择，不是精度或速度档位；不能笼统说换一个就一定更准、更快。

当前没有求解耗时或内存异常的证据，而且常规建图没有自动图优化，先沿用。它不改变地图分辨率、雷达采样率、地形高差或爬坡能力。

###### ceres_preconditioner（迭代求解的加速辅助，当前不用）

预条件器是帮助某些“反复逼近答案”的线性算法更快求解的辅助方式。当前采用直接求解，因此SCHUR_JACOBI不会参与这一计算；日志显示已读取，也不等于有实际加速。

若以后切换到迭代求解，需要一起核对预条件器支持范围；SCHUR_JACOBI适用于ITERATIVE_SCHUR，不能当成所有求解器通用的开关。

###### ceres_trust_strategy（图优化的修正策略）

LEVENBERG_MARQUARDT简称LM，可以理解为根据误差改善情况调整每轮轨迹修正的力度。DOGLEG是另一种提出修正步骤的方法；使用它时，才进一步选择TRADITIONAL_DOGLEG或SUBSPACE_DOGLEG。

###### ceres_dogleg_type（Dogleg内部方式，当前不生效）

当前是LM，因此改ceres_dogleg_type不会改变当前结果。这里的“步长”指计算中的位姿修正量，不是车辆每次走多远、底盘速度或坡度限制。

###### ceres_loss_function（异常图约束的影响）

None表示不增加鲁棒损失，按约束误差的平方处理。HuberLoss、CauchyLoss用于减弱大误差约束的影响；它们不能保证删掉全部错误匹配，也会影响大误差但真实有效的约束。

这项处理的是位姿图约束，不能直接清除点云里的行人、修改坡面高差，或修复IMU外参／点时间。当前自动图优化没有触发，切换损失函数也不能使坡道自动变成可通行区域。

读取路径为slam_toolbox的setSolver加载插件，再由CeresSolver::Configure读取5项ceres设置；该路径没有随参数修改自动重建求解器的回调。未识别的线性算法／信赖域／损失名称可能落回默认分支，不应把“能启动”当成拼写正确。[参数读取与配置分支（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/solvers/ceres_solver.cpp)

- **源码能确认：** 二维建图与PCD保存是两条输出链；常规导航同时使用二维静态地图和三维先验点云；当前关闭扫描匹配和闭环；预条件器与Dogleg类型在当前算法组合下不参与求解。
- **已有验证范围：** 仅独立Point-LIO静止与约20 cm低速往返；二维建图、坡道投影、长距离轨迹、闭环及这组求解器的实际优化效果尚未验证。
- **实物待核实：** 二维图与PCD坐标一致性、真实坡道和台阶区分、车辆上下坡能力、云台转动时的地图稳定性，以及地图文件保存／重新加载结果。


#### ROS Parameters（坐标接口、建图模式与续建入口）

##### 参数表

前6项是当前启用值；后3项在YAML中被注释，示例内容不是实际参数值。另补列ROS标准use_sim_time，说明前面发现的仿真时钟缺项。

```yaml
slam_toolbox:
  ros__parameters:
    odom_frame: odom
    map_frame: map
    base_frame: base_footprint
    scan_topic: obstacle_scan
    use_map_saver: true
    mode: mapping

    # 可选：载入此前序列化的地图，从指定位置或第一帧位置继续建图
    # 两种起点同时设置时，优先采用map_start_pose
    # map_file_name: test_steve
    # map_start_pose: [0.0, 0.0, 0.0]
    # map_start_at_dock: true
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `odom_frame` | `odom` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定从哪个参考系取得底盘里程计位姿 | 不适用 | 不适用 | 从TF查询扫描时刻的odom←base_footprint；不直接订阅aft_mapped_to_init或某个Odometry话题，也不生成底盘里程计 |
| `map_frame` | `map` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定二维地图及建图位姿输出的参考系名称 | 不适用 | 不适用 | 应与Nav2静态地图及定位坐标一致；改名字不会自动变换PCD或旧地图；也不代表一定由本节点发布map→odom |
| `base_frame` | `base_footprint` | 不适用，坐标系名称 | **上实物前必须核对并正确配置** | 指定建图跟踪的机器人参考点 | 不适用 | 不适用 | 当前为底盘参考系，须与扫描投影及机械TF一致；取出的二维位姿只有x、y、yaw，不含z、roll、pitch |
| `scan_topic` | `obstacle_scan` | 不适用，话题名称 | **上实物前必须核对并正确配置** | 指定二维LaserScan输入 | 不适用 | 不适用 | 当前由pointcloud_to_laserscan发布，输入是地形点筛选后的虚拟扫描；相对话题名按节点命名空间解析 |
| `use_map_saver` | `true` | 不适用，布尔 | **根据实际功能需求开启或设置** | 创建slam_toolbox自己的二维地图保存服务，并订阅地图 | 不适用 | 不适用 | 提供slam_toolbox/save_map；不会自动保存文件，不控制Point-LIO的PCD保存或位姿图序列化；launch另启动Nav2 map_saver_server |
| `mode` | `mapping` | 不适用，模式名称 | **通常保持当前值，不作为优先调试项** | 为当前Ceres插件提供建图／定位的内部配置提示 | 不适用 | 不适用 | 本版本localization值影响约束快速移除设置；它本身不切换sync建图节点或整套small_gicp导航链 |
| `map_file_name` | 未设置，默认空字符串；`test_steve`被注释 | 不适用，文件路径前缀 | **当前运行不使用这个参数，暂不调整** | 指定启动时载入的slam_toolbox序列化地图 | 不适用 | 不适用 | 非空才尝试续建入口；读取“前缀.posegraph”及“前缀.data”，不是二维地图YAML或PCD |
| `map_start_pose` | 未设置；`[0.0, 0.0, 0.0]`被注释 | `[m, m, rad]`，`[x, y, yaw]` | **当前运行不使用这个参数，暂不调整** | 提供机器人在旧二维地图中的近似起始位置与方向 | 启用后：各分量沿旧地图+x、+y或增大yaw方向变化 | 启用后：各分量向相反方向变化 | map_file_name非空时使用；须3个浮点数；同时设置dock时优先采用pose；不是z、roll、pitch，也不是小GICP的6项init_pose |
| `map_start_at_dock` | 未设置；`true`被注释 | 不适用，布尔 | **当前运行不使用这个参数，暂不调整** | 续建时选择旧地图第一帧所在位置作为起点 | 不适用 | 不适用 | map_file_name非空且未设置map_start_pose时，True选择第一帧；dock不是自动寻找充电桩，也不指令车辆移动 |
| `use_sim_time` | 实物、仿真两份slam_toolbox段均未配置；标准默认`False` | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择节点及TF缓存的时间来源 | 不适用 | 不适用 | 实物应与系统／驱动时间一致；仿真应使用/clock；当前launch的True替换不会补入缺失键，实际节点值尚未观察 |

##### 参数详解

###### odom_frame（底盘位姿从哪套坐标中取得）

可以把odom理解为“上游里程计持续记录的位置坐标”。本节点在每帧扫描的时间戳查询`odom←base_footprint`的TF，得到底盘在odom中的位置和朝向；该时刻查询失败就跳过这帧。

它没有直接订阅Point-LIO的aft_mapped_to_init，也不靠一个叫/odom的话题来取得位姿。当前链中，Point-LIO先经loam_interface适配，sensor_scan_generation再发布odom到base_footprint的TF；只有里程计消息而缺这条有效变换，仍不足以建图。

取位姿时只保留x、y和yaw。位置以m表达，yaw以rad表达；上坡时的高度、roll和pitch没有进入二维建图位姿。[二维位姿读取实现（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/include/slam_toolbox/get_pose_helper.hpp)

###### map_frame（二维地图采用的参考系）

map是最终二维栅格和建图位姿的参考系；odom是当前里程计参考系；base_footprint是随底盘移动的参考点。正常关系可读成：

```text
map → odom → base_footprint
旧场地图    当前里程计    底盘参考点
```

实车必须让二维图片／YAML、PCD定位与Nav2所理解的map具有一致尺度、原点和方向。将map_frame换成另一个名字，不会把旧地图或PCD旋转、平移过去。

当前建图launch提供单位变换的静态map→odom，并配置transform_publish_period=0，slam_toolbox不发布这条动态TF。这里只说明坐标来源；已有地图导航由small_gicp提供定位关系，和建图入口不同。续建旧图若从不同位置或朝向重新启动，旧map与新odom未必重合，需一起核对启动链，不能只取消map_file_name的注释。

###### base_frame（跟踪底盘参考点，不是独立转动的云台）

当前base_frame是base_footprint；pointcloud_to_laserscan的target_frame也是base_footprint，所以`obstacle_scan.header.frame_id=base_footprint`。虚拟扫描中心与这里跟踪的底盘参考点一致，扫描到base的变换为单位变换。

雷达点在上游已通过TF转换到底盘投影系；这里不应再次手工叠加front_mid360的安装偏移或云台yaw。若两边仅改frame_id而没有真正变换坐标，地图仍会错误；参数名称一致也不能证明上游IMU／雷达／底盘转换正确。

MID360仍挂在gimbal_yaw，随实际yaw转动、不随pitch转动；两条odom关节固定。建图使用底盘系，不意味着可以将MCU世界姿态当机械关节角，也不改变这些机械约定。

`base_footprint`这个名称不会自动得到水平投影。当前上游保留完整三维姿态，扫描按该系x-y平面投影，而slam_toolbox只取二维x、y、yaw；坡道上是否符合预期，需要核对实际投影平面，不能指望本节点补回已丢失的高度或自动把倾斜扫描拉平。扫描到base的安装关系由[LaserAssistant实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/laser_utils.cpp)取得。

###### scan_topic（接收哪份二维扫描）

当前输入为：

```text
terrain_map_ext（地形点云）
    → pointcloud_to_laserscan（高度／高差筛选，转到底盘系）
    → obstacle_scan（LaserScan）
    → slam_toolbox
```

它不是原始三维点云，也不直接读取terrain_map_ext。上游已删掉的坡点、低障碍或缺数据，不能靠更换这个话题名恢复。

当前值obstacle_scan是相对名称：默认空命名空间下为`/obstacle_scan`；若节点在/sentry命名空间中，则为`/sentry/obstacle_scan`。前面的转换器输出也使用相对名称，需在一致的命名空间和重映射下接通。

教程及官方README将scan_topic说明为绝对名称；当前2.6.10订阅实现没有强制这一形式，本工程使用相对名称obstacle_scan，按ROS规则解析。话题名称与消息中的frame_id也是两件事：名称决定接收哪条消息，frame_id决定扫描坐标如何解释。[ROS2话题名称规则](https://design.ros2.org/articles/topic_and_service_names.html)

输入使用SensorDataQoS，并经过TF消息过滤；既要有LaserScan，也要能在扫描时间取得odom与扫描系／底盘系之间的关系。[订阅与TF过滤实现（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_toolbox_common.cpp)

###### use_map_saver（提供二维地图保存入口）

True会创建`slam_toolbox/save_map`服务，并订阅地图；收到地图后，服务调用才会触发二维图片及YAML的保存。没有地图时会报告无法保存，True本身不会定时落盘。[MapSaver实现（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/map_saver.cpp)

这项只控制slam_toolbox自己的保存入口。Point-LIO保存PCD由pcd_save控制；.posegraph与.data由serialize_map接口保存。设为False也不等于关闭二维建图或关闭所有保存渠道。

当前slam_launch另外启动Nav2的map_saver_server，它是独立的地图保存服务，不能把两者视为同一个节点。保存出的二维图应与对应PCD一起归档，供当前导航方案使用。

###### mode（内部模式提示与实际启动流程的区别）

当前保持mapping，与正在启动的sync_slam_toolbox_node建图流程一致。本版本中，mode由Ceres插件读取；localization会启用更便于移除约束的内部设置，**仅改这个字符串不会把当前程序自动变成定位节点。** [Ceres模式读取](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/solvers/ceres_solver.cpp)、[同步建图节点入口](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_toolbox_sync_node.cpp)

本工程整套流程的选择来自launch参数：

| 启动选择 | 当前实际工作链 |
|---|---|
| slam=True | Point-LIO、二维扫描转换和slam_toolbox建图；另提供静态map→odom |
| slam=False | Point-LIO、small_gicp三维PCD定位及map_server二维地图；不启动slam_toolbox |

所以，做已有地图导航时，这份YAML中的mode仍写mapping，也不代表slam_toolbox在后台继续建图，因为该节点根本没有启动。若以后要使用slam_toolbox自己的定位方案，需要相应定位节点、序列化地图和匹配流程；那是另一套启动需求。[官方定位节点实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_toolbox_localization.cpp)

###### map_file_name（续建所需的序列化地图前缀，当前未启用）

三行示例前面都有#，当前没有载入旧位姿图，而是从空地图开始本次建图。map_file_name不是保存目标名称，也不是map_server的二维地图YAML路径，更不是small_gicp的prior_pcd_file。

若设置为`/某个目录/site`，读取的是`site.posegraph`与`site.data`两份文件；后缀由程序补上，不能只给一张PGM或一份PCD。两份文件应来自同一套序列化结果。[序列化读写实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/include/slam_toolbox/serialization.hpp)

当前use_scan_matching=False，从空图建图时不建立新的匹配图节点；有序列化文件也不能直接当作已经验证可续建／定位的地图。续建还涉及旧图结构、起点和新旧坐标的关系，当前未验证。

###### map_start_pose（在旧地图中的二维起点，当前未启用）

启用旧图加载时，数组含义是`[x, y, yaw]`，位置用m，朝向用rad，均相对旧地图；当前base_frame下表示底盘的二维初始位姿。例如yaw=1.5708约为90°，不是填写90。

它是供旧图匹配使用的近似起点，不是要求车辆移动到该点，也不是给新地图定义原点。还需给非空map_file_name；没有旧图路径时，单独设置这项不会启用续建。

###### map_start_at_dock（从旧地图第一帧继续，当前未启用）

这里的dock指旧图第一帧所在位置，不是检测物理充电桩，更不是自动回桩。True表示新一轮从旧图的这个起点对接；车辆实际起点和方向仍需对应。

启用续建时选择map_start_pose，或选择map_start_at_dock=True。两者都给时pose优先；pose数量不是3时，源码报错后采用`[0,0,0]`，并不会因此自动改用dock。只将dock设为False也不等于关闭旧图加载，关闭加载的入口是空map_file_name。[起点选择与加载逻辑（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_toolbox_common.cpp)

###### use_sim_time（ROS标准时钟参数：仿真段存在缺项）

当前实物和仿真YAML的slam_toolbox段均没有这一项。slam_launch将use_sim_time交给RewrittenYaml替换，但当前安装的实现只改已有键，没有为此段补键；Node也没有单独传入use_sim_time覆盖值。

因此，按当前默认启动接线可推断该节点采用标准默认False，仿真入口的True没有作用到这一段。仿真应与上游点云、机械TF共同使用/clock；实物通常使用False并核对驱动时间。这是根据配置与启动逻辑得出的判断，实际运行节点值及仿真效果仍待验证。

时钟选择由use_sim_time控制，改odom_frame、scan_topic或mode不能替代这一设置。

###### 读取时机与验证范围

坐标、话题、保存开关和Ceres模式在节点配置时读取；3项旧图参数在启动载入入口读取。相关路径没有在参数改动后自动重建订阅、求解器或重新加载旧图的回调；ROS标准use_sim_time由节点时钟机制处理，应与这些启动参数分开理解。

- **源码能确认：** 当前6项实物／仿真配置相同；底盘位姿来自扫描时刻TF，保留二维x、y、yaw；输入是底盘系虚拟扫描；保存服务与文件落盘不同；续建3项未设置；mode不单独切换整个工作链；仿真段缺少显式use_sim_time。
- **已有验证范围：** 独立Point-LIO静止与约20 cm往返；未验证本节点建图、续建、地图保存、上下坡投影或实际仿真时钟值。
- **实物待核实：** IMU／雷达／底盘转换与关节机械角、扫描中心和投影平面、TF与扫描时间一致性、二维图与PCD共同基准；若需续建，还须核对旧图第一帧及新一轮起点。

##### 参数表（扫描处理、地图输出与运行资源）

这13项继续属于原文件ROS Parameters分区。扫描接收、加入建图和输出地图是不同阶段；以下按当前sync_slam_toolbox_node及2.6.10版本说明，实物与仿真配置值相同。

```yaml
    debug_logging: false
    throttle_scans: 1
    scan_queue_size: 10
    transform_publish_period: 0.0   # 0：本节点不发布map→odom TF
    map_update_interval: 0.5
    resolution: 0.05
    min_laser_range: 0.3
    max_laser_range: 10.0
    minimum_time_interval: 0.2
    transform_timeout: 0.2
    tf_buffer_duration: 10.
    stack_size_to_use: 40000000     # B，申请的栈软限制目标
    enable_interactive_mode: true
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `debug_logging` | `false` | 不适用，布尔 | **根据实际功能需求开启或设置** | 请求调试日志，并允许Ceres输出完整求解报告 | 不适用 | 不适用 | 不提高定位精度；当前自动图优化未触发，没有相应Ceres报告；ROS调试日志还受实际日志等级影响，见源码疑点 |
| `throttle_scans` | `1` | 无量纲，扫描抽取因子N | **实物运行时重点观察，有问题再针对性调整** | 按计数每N份候选扫描保留一份 | 候选扫描减少，负载可能降低，也可能遗漏移动中信息 | 候选扫描增多，负载可能增加 | 当前1不额外抽帧，但还受TF、时间与位移门槛限制；首帧例外；应为正整数，0会回退为1 |
| `scan_queue_size` | `10` | 消息数，计数 | **通常保持当前值，不作为优先调试项** | 限制等待TF的扫描消息缓存 | 可容纳更多等待消息，也可能保留更旧数据 | 更易因TF未及时到达而丢消息 | 队列满时丢最旧消息；0在过滤器中表示无限；不是同步节点后续处理队列q_的长度或工作线程数 |
| `transform_publish_period` | `0.0` | s，TF发布周期；0为特殊值 | **根据实际功能需求开启或设置** | 控制本节点发布map→odom TF的循环 | 正数范围：发布更慢 | 正数范围：发布更快 | 当前0直接停发；建图launch另发静态单位map→odom；不是关闭Point-LIO里程计或odom→base TF |
| `map_update_interval` | `0.5` | s，二维图更新循环间隔 | **实物运行时重点观察，有问题再针对性调整** | 定期重新生成并发布二维栅格 | 地图显示更新更慢，重建负担可能降低 | 更新更密，栅格重建负担可能增加 | 名义约2 Hz，实际受耗时影响；不是扫描接收／处理频率，也不自动保存文件；需有/map订阅者及可生成的地图 |
| `resolution` | `0.05` | m／格，栅格边长 | **实物运行时重点观察，有问题再针对性调整** | 决定生成二维地图的格子大小 | 地图更粗、格数减少，窄通道／小结构更易合并 | 地图更细、格数增加，计算与内存负担通常增大 | 当前每格5 cm；不是定位误差或可跨越高度；不改变PCD、地形降采样或实际回波 |
| `min_laser_range` | `0.3` | m，二维扫描距离下界 | **上实物前必须核对并正确配置** | 设置建图采用的最低扫描距离 | 忽略更多近距离读数，可能漏车旁结构 | 允许更近读数，但不能恢复上游删点 | 首次初始化该扫描系时读取；不会低于输入LaserScan.range_min；当前中心在base_footprint，不是MID360外壳 |
| `max_laser_range` | `10.0` | m，建图射线／端点使用阈值 | **实物运行时重点观察，有问题再针对性调整** | 限定有效回波参与栅格与匹配的距离范围 | 有效范围内可利用更远结构，也增加远处稀疏点影响 | 远处端点不再作为障碍；部分有效射线会截短 | 首次初始化时读取并受消息量程约束；当前输入range_max也是10 m，调大不会看得更远；超阈值不全等于整束删除 |
| `minimum_time_interval` | `0.2` | s，候选扫描时间戳之差 | **实物运行时重点观察，有问题再针对性调整** | 限制连续放行扫描的最小时间间隔 | 时间筛选更严格，可能留下更稀疏的移动轨迹 | 时间筛选放宽，负载可能增大 | 按header.stamp比较；通常放行上限约5 Hz，不保证每0.2 s入图；还须通过抽帧和水平位移门槛 |
| `transform_timeout` | `0.2` | s，等待TF时间；条件下也作TF时间戳前移量 | **实物运行时重点观察，有问题再针对性调整** | 设置扫描TF过滤器等待变换的期限 | 可等待更久，也可能延后扫描放行 | 更快放弃未就绪的变换 | 若本节点发布map→odom，还把TF戳设为最近扫描戳+此值；当前停发，这一附加用途未生效；不补偿点时间错误 |
| `tf_buffer_duration` | `10.0` | s，动态TF历史跨度 | **通常保持当前值，不作为优先调试项** | 保留动态TF历史供扫描查询 | 可查更早的变换，缓存占用增加 | 历史更短，旧扫描可能无法找到变换 | 不是等待10 s、允许时间错开10 s或发布频率；无法修复不同时间域、错误frame或不存在的TF |
| `stack_size_to_use` | `40000000` | B，字节；约40 MB／38.15 MiB | **通常保持当前值，不作为优先调试项** | 节点入口申请提高进程栈软限制 | 允许更大的栈使用，不等于建图提速 | 目标降低，较深调用可能受限制；源码不主动降低已有软限制 | 主要服务于较大地图序列化；不是PCD缓存、总内存或所有线程内存上限；实际受系统限制 |
| `enable_interactive_mode` | `true` | 不适用，布尔 | **根据实际功能需求开启或设置** | 允许人工调整位姿图，并保留额外扫描缓存 | 不适用 | 不适用 | True只是允许功能，启动时实际interactive_mode仍为False；当前关闭扫描匹配，没有新建图节点可供拖动，但附加扫描缓存仍会增长 |

##### 参数详解（扫描处理、地图输出与运行资源）

###### debug_logging（调试信息，不改变建图精度）

当前False适合普通观察；True可让实际Ceres求解输出完整报告，但当前自动图优化没有触发，这一报告不会凭开关自动出现。

源码调试等级设置使用字面日志器名称`logger_name`，而不是节点的slam_toolbox日志器；因此单开True不能保证节点的DEBUG日志显示，还要核对launch传入的日志等级。它不是“开启更准确算法”的开关。

###### throttle_scans（按份数抽取扫描）

N=1不增加抽帧；N=2约每两份保留一份，N=3约每三份保留一份。计数针对进入筛选函数的扫描，不是MID360的每个点或全部原始消息，首帧会直接放行。

例如候选输入20 Hz、N=2，抽帧后的候选上限约10 Hz；后面的0.2 s时间门槛仍可能将其压到不超过约5 Hz，位移不足时还会更低。改N不会控制雷达发布频率，也不保证实际入图频率等于输入频率除以N。

###### scan_queue_size（等待TF的队列，不能包办处理积压）

等待对应时刻TF的扫描，最多先缓存10份；超出时丢最旧消息。增大可缓解短暂TF迟到，但不会增加计算能力，也不会生成缺失变换。[Humble MessageFilter实现](https://github.com/ros2/geometry2/blob/humble/tf2_ros/include/tf2_ros/message_filter.hpp)

TF就绪后，当前同步节点还将通过筛选的扫描放进独立FIFO队列q_，再逐份处理。这个队列没有受scan_queue_size限制；源码在剩余长度超过10时报警，但没有在此处自动丢弃积压。调大前面的等待队列不能解决后面的处理过慢。[同步处理实现（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_toolbox_sync.cpp)

###### transform_publish_period（本节点的map→odom发布周期）

原配置注释“if 0 never publishes odometry”容易让人误认为关闭了里程计。实际控制的是**slam_toolbox自己的map→odom TF**；当前0使这一线程直接返回，Point-LIO、lidar_odometry和sensor_scan_generation的odom→base_footprint不由它关闭。

正数时，0.05 s约20 Hz、0.1 s约10 Hz；改变发布节奏不意味着同样频率产生新的定位结果。当前建图launch已有静态单位map→odom，启用动态发布前必须明确这条关系由谁提供，避免两个来源同时发布。

###### map_update_interval（二维图更新间隔，与入图间隔分开）

当前0.5 s对应名义约2 Hz地图更新循环；minimum_time_interval=0.2 s控制的是扫描筛选，两个频率可以不同。同一份已处理扫描集合也可以被多次重新画成地图，更新更快不等于获得更多环境信息。

当前实现从全部已处理扫描重新生成栅格，并非只改最新一小块；地图扩大、历史扫描增多时，缩短间隔可能明显增加负担。无/map订阅者会跳过生成；当前地图保存组件会自行订阅，所以即使不看RViz也可能持续生成地图。[栅格生成入口](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_mapper.cpp)

该循环使用rclcpp::Rate的主机时钟，不绑定节点的/clock。地图消息戳取最近进入扫描回调的header.stamp，也不保证那份扫描已完成入图；应将“消息变新”“栅格变了”“新扫描已处理”分开判断。间隔应为正数，0没有像TF发布周期那样的停发分支。[Humble Rate时钟实现](https://github.com/ros2/rclcpp/blob/humble/rclcpp/include/rclcpp/rate.hpp)

###### resolution（二维图每格多大）

0.05 m表示一格5 cm。调大如0.1 m，格子变粗，小杆、墙边或窄口可能合并；调小如0.025 m，格子更细，但不会补出雷达没采到或上游已删掉的点。

同样覆盖面积，格子边长减半，格数约变为4倍。更细也不等于更准：定位漂移、地形误分类、投影方向或点时间错误仍会把错误轮廓画进更细的格子。

这项不是“允许5 cm台阶”，也不决定坡面能否识别为地面；它只影响二维表达的颗粒度。生成地图后，保存的YAML应记录对应分辨率；仅改旧YAML数字会改变旧图的实际米制尺寸，不能作为重新建图的替代。

###### min_laser_range（建图采用的最近距离）

当前下界0.3 m，与转换器的range_min一致；距离以虚拟扫描中心base_footprint计算。减小这项不能恢复转换器已删除的0.3 m内点，源码也会将低于消息range_min的设置提高到消息下界。

增大可能排掉近处自车回波，但也会排掉车旁障碍；圆形距离门槛不能完整代替车体与云台活动包络的自车过滤。当前Karto栅格射线处理中，读数小于或等于实际下界会被跳过。

###### max_laser_range（远处回波怎样参与建图）

当前阈值10 m，转换器输出消息的range_max也为10 m；超过消息上限的参数会被压回该上限。增大这项无法使当前输入获得更远回波，缩短则会减少可用于画远处障碍的结构。[范围初始化实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/laser_utils.cpp)

这项不是简单的“超过阈值全部删掉”：若阈值设为5 m，输入有一份合法8 m回波且消息上限仍为10 m，栅格射线会截到5 m，不把8 m端点画成障碍；这段射线仍可参与空闲区域的统计。

**当前∞无回波被跳过，不会自动把整条方向画成10 m空地。** 大于／等于消息上限的值和NaN同样被跳过。因此，上游将正常坡面滤掉后，若该方向没有其他有效回波提供射线覆盖，坡面区域不保证成为空闲；调分辨率或输出频率不能补出这份观测。占用／空闲还依赖栅格累计统计。[Karto射线栅格实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/include/karto_sdk/Karto.h)

###### minimum_time_interval（扫描最短放行间隔，还有位移门槛）

当前按扫描时间戳与上一份放行扫描比较，差值不足0.2 s则拒绝；首帧例外。它只给出通常约5 Hz的时间上限，不是固定5 Hz处理定时器。增大可降低入图数据量，减小只放宽时间条件。

**当前还有一个与实车原地转动有关的前置限制：** 公共筛选函数检查水平位移平方，未用yaw放行。当前minimum_travel_distance=0.1 m，对应这里约8.94 cm的水平位移门槛，另有初期计数限制。以0.2 m/s移动0.2 s只有4 cm，仍可能不够，所以低速入图间隔会更长。

首帧之后，原地转底盘或独立转云台时，若底盘TF的x、y基本不变，后续扫描会被这层拦住；单独缩短minimum_time_interval并不能解决。后续Karto层的minimum_travel_heading还没机会介入，不能凭设置了转角阈值就认定原地旋转会正常更新地图。这里是源码条件，尚无本节点转弯或独立yaw验证。

###### transform_timeout（等待TF；条件下还影响发布戳）

当前0.2 s传给TF消息过滤器，表示等对应扫描时刻的变换最多等多久。调大可容忍更晚到达的有效TF，也可能让旧消息等待更久；TF根本不存在或时间域不一致时，等待更久没有修复作用。

若transform_publish_period为正，源码还把发布的map→odom戳设为“最近扫描戳+transform_timeout”；这不是预测未来运动。当前period=0，这个用途没有参与输出。它也不会给输入点云补运动补偿、把逐点时间单位改正确，或将TF历史缓存变长。

###### tf_buffer_duration（动态TF历史保留多久）

10 s表示动态TF历史缓存跨度，用来查询扫描当时的变换；不是等10 s再处理，也不是允许扫描与TF随意差10 s。静态机械变换的保存规则与动态历史不同。

积压扫描太旧时，适当增大可能保留所需历史，但不会消除处理积压；减小则可能更快丢掉旧变换。前面已确认仿真段use_sim_time缺项，增大历史缓存不能替代统一时间来源。

###### stack_size_to_use（栈限制，通常沿用）

40000000 B为约40 MB，约38.15 MiB。节点入口在现有栈软限制低于目标时申请提高，源码不主动将较高的现有限制降下来；设置更小不一定让实际栈变小。[同步节点入口](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_toolbox_sync_node.cpp)

它主要为大地图序列化等调用准备，不限制Point-LIO累计PCD、扫描缓存或程序总内存。系统是否接受申请仍受资源限制影响，入口没有检查setrlimit返回值；当前没有相关失败证据，通常保持。

###### enable_interactive_mode（允许人工改位姿图，不等于已开启交互）

当前True允许切换到RViz位姿图交互功能；启动时实际interactive_mode=False，不会自动进入拖动模式。进入后处理线程暂停，便于修改已有图节点；退出后恢复，期间新候选扫描仍可能进入同步队列，移动中使用会造成积压。[交互切换实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/loop_closure_assistant.cpp)

当前use_scan_matching=False，从空图建图时没有新图节点可供拖动；但enable=True仍会额外保存已成功处理的原始扫描，即使实际交互未开启。没有人工调整需求时，关闭可减少附加缓存，Karto自身的历史扫描仍存在，不能当作总内存开关。

人工拖动／优化的是二维位姿图，不会同步修正Point-LIO保存的三维PCD。当前二维图与PCD需保持共同基准，也不能用交互修图替代坡面分类或坐标接口核对。

###### 读取时机与验证范围

debug、抽帧、TF队列、时间门槛、栅格分辨率、TF缓存及交互许可在节点配置／线程初始化时读取；两项雷达范围在首次创建该扫描系的元数据时读取，随后缓存。stack_size_to_use由可执行程序入口读取。相关路径没有参数变化后自动重建这些成员／队列／激光元数据的回调；交互切换使用专门服务，不能混同为直接改参数即可生效。

- **源码能确认：** 本组13项实物／仿真配置一致；扫描筛选与地图输出分离；等待TF队列与同步处理队列不同；当前不发布动态map→odom；二维范围受上游量程约束；∞不自动产生空闲射线；位移前置门槛可能拦住原地转动后的扫描。
- **已有验证范围：** 独立Point-LIO静止与约20 cm低速往返；未验证slam_toolbox实际处理频率、原地转动、坡道空闲覆盖、长时间队列／内存、序列化或交互功能。
- **实物待核实：** 移动与转动下的入图连续性、雷达／底盘时间与坐标、近场自车回波和低障碍、坡道是否被有效射线覆盖、地图细节与计算负载。重力方向、内部IMU转换及点时间问题仍需先核对。

#### General Parameters（扫描入图、匹配与闭环）

**当前use_scan_matching=False、do_loop_closing=False。** slam_toolbox仍按上游里程计摆放扫描并生成二维图；距离与转角门槛仍会检查，但普通扫描匹配、图节点关联和自动闭环分支被跳过。表中“当前运行不使用”表示参数已读入、作用分支没有执行，不是参数拼写无效。

这些参数决定二维扫描如何取样和关联，不判断车辆能否爬坡。坡面／障碍区分仍由前面的地形分析与二维扫描筛选处理。

##### 参数表

```yaml
    # General Parameters
    use_scan_matching: false
    use_scan_barycenter: true
    minimum_travel_distance: 0.1
    minimum_travel_heading: 0.1
    scan_buffer_size: 10
    scan_buffer_maximum_scan_distance: 10.0
    link_match_minimum_response_fine: 0.1
    link_scan_maximum_distance: 1.5
    loop_search_maximum_distance: 3.0
    do_loop_closing: false
    loop_match_minimum_chain_size: 10
    loop_match_maximum_variance_coarse: 3.0
    loop_match_minimum_response_coarse: 0.35
    loop_match_minimum_response_fine: 0.45
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_scan_matching` | `false` | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否用二维扫描匹配修正里程计给出的扫描位置 | 不适用 | 不适用 | 当前跳过匹配和新图节点／关联构建，但仍入图；不关闭Point-LIO三维估计；开启可能改变二维轨迹与PCD的对应关系 |
| `use_scan_barycenter` | `true` | 不适用，布尔 | **当前运行不使用这个参数，暂不调整** | 选择扫描间距离判断的参考点：回波点重心或扫描中心 | 不适用 | 不适用 | 当前图关联分支未执行；True不是车辆重心，也不改变base_frame；启用关联后影响1.5 m／3 m候选距离的含义 |
| `minimum_travel_distance` | `0.1` | m，平面位移 | **实物运行时重点观察，有问题再针对性调整** | 限制两份扫描之间的移动取样间隔 | 取样通常更稀，负担降低，也可能漏掉转弯细节 | 取样通常更密，负担增加 | 当前仍生效；外层约8.94 cm门槛，Karto内层约10 cm条件；与时间、转角两层组合，不保证每走10 cm入一帧 |
| `minimum_travel_heading` | `0.1` | rad，约5.73° | **实物运行时重点观察，有问题再针对性调整** | Karto内层允许因朝向变化而采用新扫描 | 需转更多角度才满足内层转角条件 | 更小转角即可满足内层条件 | 当前仍被内层检查，但必须先通过外层水平位移门槛；不是坡度、云台转速或车辆控制角 |
| `scan_buffer_size` | `10` | 份，匹配历史扫描计数 | **当前运行不使用这个参数，暂不调整** | 限制连续扫描匹配的近期窗口长度 | 启用后：可保留更多近期结构，匹配工作量增加 | 启用后：窗口更短，历史结构支撑减少 | 当前不加入running scans；不是等待TF的scan_queue_size，也不限制全部建图历史；非正值会回退10 |
| `scan_buffer_maximum_scan_distance` | `10.0` | m，近期窗口首尾扫描中心间距 | **当前运行不使用这个参数，暂不调整** | 限制近期匹配窗口的空间跨度 | 启用后：允许更大的窗口跨度，仍受份数上限限制 | 启用后：较早扫描更易被移出窗口 | 当前匹配窗口未使用；不是雷达10 m量程，不是累计行驶里程；这项按扫描中心距离判断 |
| `link_match_minimum_response_fine` | `0.1` | 无量纲，归一化匹配分数 | **当前运行不使用这个参数，暂不调整** | 接纳普通近邻扫描链关联的分数门槛 | 启用后：关联更严格，可能少连有效结构 | 启用后：更易关联，也可能接纳错误对应 | 仅当前关闭的图关联分支使用；不是所有逐帧匹配的统一拒帧门槛；0.1不是10%成功概率 |
| `link_scan_maximum_distance` | `1.5` | m，扫描参考点的平面距离 | **当前运行不使用这个参数，暂不调整** | 限定普通近邻链与当前扫描可以关联的距离 | 启用后：候选范围扩大，可能比较更多相似区域 | 启用后：范围缩小，也可能漏关联 | 当前图关联未执行；使用barycenter所选参考点；不是激光回波距离或允许定位误差 |
| `loop_search_maximum_distance` | `3.0` | m，闭环候选参考点距离 | **当前运行不使用这个参数，暂不调整** | 找出可能属于同一旧区域的历史扫描链 | 启用后：纳入更远历史候选，也增加歧义 | 启用后：候选更局限，漂移较大时易漏闭环 | 当前自动闭环未执行；不是后续匹配搜索网格的边长；3 m内也不保证匹配通过 |
| `do_loop_closing` | `false` | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否自动识别重访区域并优化闭环轨迹 | 不适用 | 不适用 | 只有use_scan_matching=True才进入此开关所在分支；当前两个False，单改为True也不会启用自动闭环 |
| `loop_match_minimum_chain_size` | `10` | 份，历史候选扫描计数 | **当前运行不使用这个参数，暂不调整** | 要求候选链包含足够扫描，以提供匹配结构 | 启用后：要求更长历史链，可能减少有效候选 | 启用后：短链更易参与，结构证据较少 | 当前图关联／闭环均跳过；还用于普通近邻链，非只用于闭环；不是激光点数 |
| `loop_match_maximum_variance_coarse` | `3.0` | m，配置尺度；内部平方为m²门槛 | **当前运行不使用这个参数，暂不调整** | 粗匹配后排除位置估计过于分散的候选 | 正值范围内：内部方差门槛随平方增大，接受更分散的结果 | 正值范围内：内部方差门槛降低，要求更集中，也可能拒绝有效候选 | 当前闭环未执行；配置3.0先平方为9.0 m²，检查x、y方差各自小于9，不检查yaw方差；不是3 m实际误差上限或small_gicp的max_dist_sq |
| `loop_match_minimum_response_coarse` | `0.35` | 无量纲，归一化匹配分数 | **当前运行不使用这个参数，暂不调整** | 闭环粗匹配进入精匹配的最低分数门槛 | 启用后：更严格，较少候选进入精匹配 | 启用后：更多候选进入，也可能带入假匹配 | 当前闭环未执行；须分数>0.35，且x、y方差都过门槛；不是35%定位置信度 |
| `loop_match_minimum_response_fine` | `0.45` | 无量纲，归一化匹配分数 | **当前运行不使用这个参数，暂不调整** | 最终闭环精匹配的分数门槛 | 启用后：更难接纳闭环，也可能漏掉真实重访 | 启用后：更易接纳，也更容易错连重复结构 | 当前闭环未执行；该步分数≥0.45才进入闭环连接／优化；不是45%成功概率或定位精度 |

##### 参数详解

###### use_scan_matching（是否用二维轮廓修正里程计）

开启后的扫描匹配，可以理解为：先按里程计估计把当前二维轮廓摆上去，再与近期已有轮廓比较，寻找更吻合的位置和朝向。它可能减轻二维里程计误差，也可能在重复走廊、稀疏轮廓或误分类地形上找到错误对应。

当前False沿用上游Point-LIO提供的运动估计，仍会生成二维栅格；源码只是跳过本节点的二维匹配、图节点和关联构建。不能据此说整车SLAM已关闭，也不能据此证明长距离二维图不会漂移。[普通处理路径（2.6.10）](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

是否需要开启，应结合长距离、转弯和重访结果判断。二维修正不会回写Point-LIO保存的PCD；当前建图又采用静态单位map→odom，启用前需一并核对二维图、PCD和定位参考系的关系。该开关不能修复重力、外参、点时间或坡面误分类。

###### use_scan_barycenter（用回波重心衡量扫描间距，当前不用）

True时，将一份扫描中有效回波端点的平均位置作为图距离判断参考；False用扫描中心。这里的重心属于“看到的环境点”，不是底盘质量重心，也不把导航参考系改成云台系。

遮挡、可见墙面或动态物体变化会让点重心改变，即使车辆位移很小。它会影响图关联中的近邻范围，但当前use_scan_matching=False不走这一路；入图位移仍按底盘／虚拟扫描中心判断。[参考点定义](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/include/karto_sdk/Karto.h)

###### minimum_travel_distance（走多远才采用下一份扫描）

当前有两道筛选，不能只按“每10 cm一帧”理解：

1. 外层slam_toolbox入口：通常要求扫描戳间隔至少0.2 s，还要求相对上一次放行扫描的水平位移至少约8.94 cm，并受抽帧和初期计数限制。
2. 内层Karto：相对上一次实际采用的扫描，约10 cm位移或约5.73°朝向变化可放行；这里使用的是扫描中心，当前与base_footprint重合。

外层先执行，且它在放行后就更新自己的比较基准；内层若拒绝该扫描，两层基准会暂时不同。因此实际步距还取决于速度、转角、输入周期和处理结果。调大通常减少入图份数，调小通常增加；过稀可能让转弯附近的小结构覆盖不足。

还有一个版本相关的区别：ROS参数minimum_time_interval=0.2用于外层；当前SMapper::configure没有将它下发给Karto。新建地图的Karto内部时间兜底仍是默认3600 s，普通短时间建图主要依靠内层距离／转角条件。0.2 s参数仍有效，但不能当作两层都采用同一时间门槛。[ROS到Karto参数接线](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_mapper.cpp)

###### minimum_travel_heading（入图角度间隔，0.1约为5.73°）

单位是rad，0.1不是0.1°。内层比较两份扫描的平面yaw变化；调小可让更小转角满足内层条件，调大要求转得更多，但它不能绕过前面的时间／位移筛选。

首帧之后，原地转底盘或独立转云台时，若底盘TF的x、y不变，扫描会先被外层拦住；即使内层转角本可达标，也没有机会参与判断。**只调小minimum_travel_heading，不能解决当前入口对原地转动扫描的限制。** 这里涉及建图取样，不控制底盘wz、云台机械yaw，也不是可爬坡角。

###### scan_buffer_size（近期匹配保留几份扫描，当前不用）

启用匹配后，最多保留10份近期扫描供连续匹配；它和之前scan_queue_size=10含义不同，后者缓存等待TF的消息。这里也不是“整张地图只保留10份历史”，移出近期窗口的扫描仍可留在建图历史中。

如果直线入图步距恰好0.1 m，10份扫描的首尾约隔9段，即0.9 m；实际步距未必为0.1 m，所以不能由份数直接推成固定米数或秒数。当前不向该匹配窗口加入扫描，先不调整。

###### scan_buffer_maximum_scan_distance（近期窗口首尾间距，当前不用）

启用后，近期窗口同时受10份和首尾中心10 m间距限制，超限就从最旧扫描开始移出，直到两个条件都满足。距离是首尾直线距离，绕圈回到附近不等于累计路程小；它也不是雷达回波的10 m量程。

份数上限先达到时，增大10 m通常不会让窗口更大；两项需结合真实入图步距理解。当前匹配关闭，窗口删除逻辑不参与结果。

###### link_match_minimum_response_fine（普通近邻链关联门槛，当前不用）

启用图关联后，当前扫描与附近历史扫描链匹配，分数够高才接纳这类关联。0.1是轮廓匹配的归一化分数，不是“有10%概率匹配正确”。

**它不是每帧连续匹配的统一质量检查。** 当前版本的逐帧MatchScan返回分数没有在Mapper::Process中用这项拒帧；近邻链门槛也不能兜住所有错误逐帧匹配。当前整条匹配分支关闭，改这项不会改善现有地图。

###### link_scan_maximum_distance（普通关联的近邻范围，当前不用）

启用后，只考虑参考点相距约1.5 m内的近邻链关联；当前barycenter=True时，参考点是环境回波重心。调大可纳入更远候选，也可能增加重复结构歧义；调小则更局限。

它不是“激光只看1.5 m”，也不是车辆碰撞距离或允许地图误差；改变这项不改变上游扫描量程。

###### loop_search_maximum_distance（从哪些旧区域寻找闭环，当前不用）

开启闭环后，3 m用于筛选当前扫描与旧扫描参考点的距离，不代表在3 m内就认定回到旧位置。候选还需经过链和匹配条件；后续匹配搜索网格的范围是另一组参数。

例如车物理上回到旧位置，但估计参考点已经漂得较远，旧区域可能落在3 m候选范围外；增大能放宽寻找范围，却也会增加把相似走廊误当同一区域的机会。当前不执行自动闭环，先不调整。

###### do_loop_closing（自动重访识别与闭环修正）

闭环可理解为：绕行后认出曾经经过的区域，用这条新约束重新调整二维轨迹，使前后轮廓更一致。成功闭环可能减少累积偏差；错误闭环也可能把整片地图拉错。

当前False，且它位于use_scan_matching=True的分支内，因此只将do_loop_closing改成True仍不会运行。反过来，开启扫描匹配但保持闭环False，仍可运行近期匹配和普通近邻链关联，只是不自动走闭环这一路。

这是slam_toolbox自己的二维功能，不是small_gicp实时点云对先验PCD的定位。即使成功闭环，也不自动修正独立保存的三维PCD或解决当前静态map→odom的坐标衔接。

###### loop_match_minimum_chain_size（历史链至少几份，当前不用）

10表示候选历史链的扫描份数，不是10个雷达点或10 m。增加要求更多历史结构，减少则允许较短的链参与。

源码中的普通近邻链LinkNearChains也使用它，所以即使以后do_loop_closing仍False，只要开启扫描匹配，它也可能影响普通图关联。当前这两条关联路径都未启用，先不调整。

###### loop_match_maximum_variance_coarse（配置先平方，内部门槛9 m²）

粗匹配可能有多个近似答案。当前2.6.10读取YAML值3.0后，setParamLoopMatchMaximumVarianceCoarse会先平方，内部方差门槛实际为3.0²=9.0 m²。按赋值链与方差比较的量纲，配置值按位置分散尺度（m）理解，内部才使用m²。

这里检查算法给出的x、y方差各自是否小于9.0 m²；正值范围内，配置值越小要求结果越集中。不检查yaw方差，也不保证实际位置误差小于3 m。[平方赋值与方差判定源码](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

它与small_gicp的max_dist_sq含义及配置尺度都不同：本项配置3.0先平方为9.0 m²的匹配结果方差门槛；max_dist_sq的配置值本身就是m²，3.0直接表示3.0 m²的点对应距离平方门槛。不能因为两项都写3.0就认为容差相同。当前闭环关闭，无须调整工程数值。

###### loop_match_minimum_response_coarse（粗匹配进入精匹配的门槛，当前不用）

分数须大于0.35，且x、y方差同时通过上一项，才进入精匹配。增大更严格，减小放宽；分数来自轮廓与相关性网格的吻合程度，范围归一化到0～1，不是经过标定的定位置信度。

###### loop_match_minimum_response_fine（最终闭环匹配门槛，当前不用）

粗匹配通过后再精匹配，最终分数达到0.45才进入闭环连接与轨迹优化。更严格可能漏掉有效重访，更宽松可能错连重复结构；不能把达到0.45理解为位置已准确。

这3项质量门槛共同作用，不能只看粗分数、细分数或方差中的一个；粗、细两级使用的匹配过程也不同，两个分数不宜直接当成同一置信度尺度比较。[候选关联与闭环判定源码](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

###### 读取时机与验证范围

参数名按当前源码区分大小写：do_loop_closing与小写loop_*是实际键名；教程中的do_loop_close、Loop_*等写法不能直接作为本工程参数名。

本组14项经SMapper::configure读取并设置到Karto。常规从空图建图时，use_scan_matching=False使近期匹配、图关联及闭环分支跳过；距离和转角仍用于入图判断。相关路径没有动态参数更新回调，外层还静态缓存距离门槛，不能把运行中设置新值等同于这些条件已全部更新。

- **源码能确认：** 当前配置关闭二维匹配和自动闭环；距离／转角仍有作用但受两层门槛约束；重心只改变图距离参考；近期窗口不限制全部历史；链长度还用于普通关联；闭环配置尺度按m理解，先平方为m²方差门槛，当前3.0对应9.0 m²；响应值不是成功概率。
- **已有验证范围：** 仅独立Point-LIO静止与约20 cm低速往返；未验证slam_toolbox长距离、转弯、独立云台yaw、闭环或与PCD的一致性。
- **实物待核实：** 移动／原地转动时实际入图连续性，长距离及重访误差，坡道有效二维轮廓、遮挡与动态干扰；若需启用匹配／闭环，还须核对二维修正与PCD及定位参考系的衔接。

#### Correlation Parameters - Correlation Parameters（普通扫描匹配搜索）

这3项可理解为：**附近多大范围找位置、隔多远试一次、轮廓稍微错开时还给多少匹配分值。** 它们控制内部匹配网格，与输出地图resolution=0.05 m不是同一个网格。

当前use_scan_matching=False，不执行普通匹配评分；但首次合法扫描进入Karto时仍会创建匹配器，使用这3项分配网格并检查数值组合。因此，当前先保持，不能把“未执行匹配”理解为任意设置都无影响。

##### 参数表

```yaml
    # Correlation Parameters - Correlation Parameters
    correlation_search_space_dimension: 0.5
    correlation_search_space_resolution: 0.01
    correlation_search_space_smear_deviation: 0.1
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `correlation_search_space_dimension` | `0.5` | m，粗平移搜索区边长 | **通常保持当前值，不作为优先调试项** | 在当前二维位姿估计周围搜索位置修正 | 合法组合内：候选平移范围扩大，匹配工作量和初始化网格可能增加，也更易出现相似位置 | 范围缩小，候选减少，真实修正可能落在范围外 | 当前不参与匹配，初始化仍使用；0.5 m对应粗搜索x、y各约±0.25 m，不是半径0.5 m；角度范围另设 |
| `correlation_search_space_resolution` | `0.01` | m／格，相关网格格距 | **通常保持当前值，不作为优先调试项** | 决定匹配网格及平移候选的离散尺度 | 合法组合内：网格／步距变粗，资源负担通常降低，小错位区分更粗 | 网格／步距变细，候选和网格内存增多；不保证更准确 | 当前不参与匹配，初始化仍使用；此版本粗步距为2倍格距即2 cm，精步距1 cm；不是输出地图的5 cm分辨率 |
| `correlation_search_space_smear_deviation` | `0.1` | m，匹配评分涂抹的标准差σ | **通常保持当前值，不作为优先调试项** | 给回波端点周围逐渐减弱的分值，允许轮廓轻微错位仍有响应 | 合法组合内：评分轮廓更宽平滑，更容忍错位，也更难区分相近结构 | 评分更尖锐，但稀疏点／小噪声更容易降低响应 | 当前不用于评分，初始化仍建核；不是允许10 cm定位误差或障碍膨胀半径；0.1已是当前格距允许的上限，不能单独再增大 |

##### 参数详解

###### correlation_search_space_dimension（在估计位置附近找多大范围）

以当前内部二维扫描位姿为中心，在建图平面的x、y方向尝试平移。0.5 m表示粗搜索正方形边长，所以两个方向各约±0.25 m；搜索中心来自里程计／已有扫描修正，不是地图原点、云台机械零位或Nav2目标点。

例如，真实轮廓在估计位置旁偏了约15 cm，范围上可能包含正确平移；若一次匹配需要修正约40 cm，默认粗搜索可能够不到。包含候选只表示“会尝试”，不保证能在重复走廊或稀疏点中选对位置。

这是粗平移范围，不直接扩大yaw搜索角，也不要求车辆移动到候选位置。精匹配还会在粗结果附近再搜索；当前局部精搜索平移约±1 cm，因此±0.25 m也不是最终修正量的绝对边界。

###### correlation_search_space_resolution（1 cm格距，粗搜索实际按2 cm试）

当前内部相关网格每格1 cm。源码粗平移每次跨两格，即2 cm；正常精化阶段回到1 cm步距，在粗结果周围小范围细找。[搜索范围与粗细步距实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

因此，不能把0.01简单理解为“整个0.5 m范围每隔1 cm全部精搜”，也不能把它当成1 cm定位精度。输出二维图片仍采用resolution=0.05 m，Point-LIO与地形降采样尺度也不会因此改变。

在有效组合内，固定搜索范围而步距减半，平移候选数量约增为4倍；总工作量还受角度样本和扫描点数影响。相关网格还要覆盖有效回波范围，并非只分配0.5×0.5 m的小方格，减小格距也会增加初始化内存。

###### correlation_search_space_smear_deviation（给轮廓周围渐弱的匹配分值）

可以理解为给内部匹配轮廓加一圈“渐弱的软边”：当前点没有恰好落在旧轮廓上，稍微错开仍能得到分值，离得更远则分值更低。这里用的是高斯标准差，σ=0.1 m不是“10 cm内全部算匹配、外面全部不算”。

较宽的软边可能帮助稀疏／有小噪声的轮廓匹配，也会削弱相近位置的区别。它只影响内部匹配评分，不会把保存的二维墙体直接加厚，不是Nav2障碍膨胀，也不清除动态物体或改变坡面高差。

当前关闭匹配，不靠调它改善现有地图。以后若启用，仍应先核对IMU／雷达／底盘转换、点时间和有效二维轮廓；扩大搜索或涂抹不能代替这些接口核对。

###### 三项必须配套：当前数值合法，涂抹已到上限

当前版本有这些组合约束：

- 维度与格距需为正；源码按dimension/resolution建立离散网格，断言要求这个比值为整数，并使相关网格边长为奇数。按该创建逻辑，应选**正偶整数比值**；当前0.5/0.01=50满足条件。
- 涂抹标准差须在`0.5×格距～10×格距`内，边界可取。当前格距0.01 m，对应σ为0.005～0.1 m，当前0.1 m处于上限。
- 例如，只把格距改为0.005 m而保持σ=0.1 m，此时允许上限变为0.05 m，初始化会抛异常；即使扫描匹配关闭，也不能规避这项检查。

范围或格距组合的检查使用assert，实际是否触发还取决于编译时是否启用断言；不能仅凭程序能启动就判断组合合理。涂抹范围检查则明确抛runtime_error，错误文案可能写“too small”，即使实际是超过上限。[涂抹核及取值限制](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/include/karto_sdk/Mapper.h)

###### 读取与使用路径

3项在SMapper::configure中读取并设置到Karto，非正值分别回退到0.5、0.01、0.1；网格在首次有效扫描初始化时创建，没有动态修改参数后自动重建的回调。[参数读取实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_mapper.cpp)

启用扫描匹配后，这套普通匹配器用于近期匹配和普通近邻关联；若以后启用闭环，闭环的精匹配也会复用它。后面的loop_search_space_*控制另一个闭环匹配器，分区不同不代表两条流程完全独立。

- **源码能确认：** 搜索边长、粗细步距、涂抹标准差与组合限制；当前不执行评分，但仍初始化匹配网格。不能把这3项全当作未读取参数。
- **已有验证范围：** 独立Point-LIO静止与约20 cm低速往返；未验证本节点的搜索成功率、资源占用、转弯、独立yaw或坡道轮廓匹配。
- **实物待核实：** 若需启用匹配，检查真实位姿修正量是否在可用范围、二维结构是否足够区分、噪声／遮挡与计算负担，并核对二维修正与PCD定位基准的一致性。

#### Correlation Parameters - Loop Closure Parameters（闭环粗匹配搜索）

这3项用于绕行后重访旧区域时的**大范围粗搜索**：找多大的位置范围、用多细的网格、旧轮廓周围给多宽的匹配分值。后续精匹配还会复用上一组普通匹配器，两个分区并非完全独立。

当前实物／仿真配置均为use_scan_matching=False、do_loop_closing=False，不执行自动闭环评分；首次有效扫描初始化时仍创建闭环匹配网格，数值组合和内存分配仍有影响。

##### 参数表

```yaml
    # Correlation Parameters - Loop Closure Parameters
    loop_search_space_dimension: 8.0
    loop_search_space_resolution: 0.05
    loop_search_space_smear_deviation: 0.03
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `loop_search_space_dimension` | `8.0` | m，闭环粗平移搜索区边长 | **通常保持当前值，不作为优先调试项** | 在当前估计扫描位置周围寻找与候选旧轮廓吻合的位置 | 合法组合内：搜索范围扩大，计算／网格负担增加，也更可能遇到相似结构 | 范围缩小，搜索更快，但真实修正可能落在范围外 | 当前不用于闭环评分，初始化仍使用；8 m是正方形边长，x、y各约±4 m；与3 m历史候选距离分工不同 |
| `loop_search_space_resolution` | `0.05` | m／格，闭环相关网格格距 | **通常保持当前值，不作为优先调试项** | 决定闭环粗匹配网格及平移候选的离散尺度 | 合法组合内：网格和粗搜索步距变粗，资源负担通常降低，也更容易漏掉窄的匹配峰 | 网格更细、候选更多、初始化内存增加；不保证定位更准 | 当前仍参与初始化；5 cm格距对应10 cm粗搜索步距，最终精匹配复用普通匹配器；不是输出地图分辨率 |
| `loop_search_space_smear_deviation` | `0.03` | m，闭环评分涂抹的标准差σ | **通常保持当前值，不作为优先调试项** | 给旧轮廓周围逐渐减弱的分值，使轻微错位仍有响应 | 合法组合内：评分轮廓更宽，更容忍错位，也更难区分相近结构 | 评分更尖锐，稀疏点或轮廓变化更容易降低响应 | 当前仍初始化涂抹核；3 cm是标准差，不是允许定位误差；当前格距要求σ在0.025～0.5 m内 |

##### 参数详解

###### loop_search_space_dimension（8 m搜索边长，与3 m候选距离不同）

闭环先挑选可能重访的历史扫描，再移动当前扫描去对比旧轮廓。这两步分别受不同参数控制：

- **loop_search_maximum_distance=3.0 m：** 筛选附近的历史候选，比较扫描参考点之间的距离。当前use_scan_barycenter=True，参考点采用有效回波端点的重心。
- **loop_search_space_dimension=8.0 m：** 对已经选出的候选，以当前估计扫描位姿为中心，在建图平面的x、y方向各约±4 m内尝试平移；这是8×8 m的粗搜索正方形。

因此，扩大8 m搜索区不能找回已被3 m候选距离排除的旧区域。8 m也不是车辆必须走够的距离、雷达量程或绕地图全局找位置的范围；角度搜索另有参数。[候选筛选与平移搜索实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

实物若经过重复走廊、相似柱子或大量动态障碍，范围更大可能找到更多“看起来像”的位置。应先确认真实二维轮廓是否稳定、位置偏差是否超出搜索范围，再考虑调整；这些参数不负责识别坡度或判定坡道可通行。

###### loop_search_space_resolution（5 cm网格，闭环先按10 cm粗搜）

此版本粗搜索跨两格，所以当前实际平移步距为2×0.05=0.1 m。闭环流程为：

1. 用本组8 m范围、5 cm网格完成粗匹配；这一阶段没有在本组网格上再做5 cm步距的内部精化。
2. 粗分数和x、y方差通过前面门槛后，以粗结果为中心，调用**普通匹配器**：使用上一组0.5 m范围、1 cm网格，先按2 cm粗搜，再在所得结果附近约±1 cm内按1 cm细搜。
3. 精分数通过门槛后，才连接闭环并优化二维轨迹。

因此，0.05既不代表最终定位误差为5 cm，也不控制后续全部精搜索。输出二维地图resolution同为0.05 m，只是数值相同，属于另一参数。[闭环粗细匹配调用](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

固定边长而将格距减半，平移候选数约增为4倍，网格内存也增加；总计算量还受角度和点数影响。最终精匹配还可能越过原粗搜索边缘，±4 m不是最终修正量的绝对上限。

###### loop_search_space_smear_deviation（给旧轮廓加一圈渐弱的评分）

与普通匹配的涂抹含义相同，但这里只用于闭环粗匹配。σ=0.03 m可理解为评分轮廓较窄，稍微错开仍有分值；不能理解成“3 cm内全部正确、外面全部错误”。

涂抹会按网格离散：当前组合生成3×3格的核，从中心向x、y各扩一格，即5 cm，邻格分值逐渐降低。因此，标准差3 cm与核的实际覆盖边界不是同一量；它也不直接加厚保存的墙体或改变导航障碍膨胀。

当前5 cm格距允许σ为0.025～0.5 m，0.03 m合法。例如，单独把格距增到0.1 m，仍保留σ=0.03 m，新下限会变成0.05 m，初始化将抛异常；即使闭环关闭也会检查。[容许范围与涂抹核实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/include/karto_sdk/Mapper.h)

###### 组合限制、读取与实物核对

范围与格距还需符合前一组的离散网格约束：按创建逻辑选正偶整数比值；当前8/0.05=160满足条件。相关assert是否执行取决于编译设置，涂抹越界则明确抛异常。

3项在SMapper::configure读取并设置到Karto，非正值分别回退为8.0、0.05、0.03；首次有效扫描创建网格，没有运行中修改这3项后自动重建的参数回调。[参数读取实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_mapper.cpp)

- **源码能确认：** 当前不执行闭环评分，但仍初始化网格；8 m与3 m分别控制粗搜索和历史候选；粗搜索步距10 cm，闭环精匹配复用普通匹配器。
- **已有验证范围：** 仅独立Point-LIO静止和约20 cm低速往返，未观察本节点闭环、长距离重访或计算负担。
- **实物待核实：** 若以后需要闭环，检查重访偏差、重复结构与动态遮挡、坡道／云台转动时二维轮廓变化，以及计算负担。二维闭环不会自动修正独立保存的PCD，仍须核对两者定位基准的一致性。

#### Scan Matcher Parameters（扫描匹配评分与搜索）

这两项可以理解为：**轮廓匹配想把估计位置／朝向改动一些时，要扣多少分。** 在轮廓吻合程度接近时，惩罚使算法更倾向于当前估计附近的答案。

最容易看反的是：**配置值越大，扣分越轻；配置值越小，扣分越重。** 当前use_scan_matching=False，这两项虽被读取，但不参与匹配评分，暂不调整。

##### 位姿偏离惩罚参数表

```yaml
    # Scan Matcher Parameters
    distance_variance_penalty: 5.5
    angle_variance_penalty: 5.0
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `distance_variance_penalty` | `5.5` | m，距离惩罚尺度；内部取平方 | **当前运行不使用这个参数，暂不调整** | 对偏离本次搜索中心的平移候选降低分值 | 正值范围内：同样偏移扣分更轻，更允许轮廓匹配修正估计；不扩大搜索范围 | 扣分更重，更偏向估计附近；也可能压制必要修正，达到系数下限后不再继续降低 | 当前扫描匹配关闭；配置5.5先平方为30.25 m²，再进入公式；不是允许偏差5.5 m或实测里程计方差 |
| `angle_variance_penalty` | `5.0` | rad，角度惩罚尺度；内部取平方 | **当前运行不使用这个参数，暂不调整** | 对偏离本次搜索中心朝向的yaw候选降低分值 | 正值范围内：同样角度偏移扣分更轻，更允许匹配修正朝向；不扩大角度搜索范围 | 扣分更重，更偏向估计朝向；也可能压制必要修正，达到系数下限后不再继续降低 | 当前扫描匹配关闭；配置5.0先平方为25 rad²；不是5°、角速度或云台机械角限制 |

##### 位姿偏离惩罚详解

###### distance_variance_penalty（估计位置附近的候选要扣多少分）

例如，估计扫描位置在A，算法尝试把当前轮廓放到旁边B去与旧轮廓对齐。此项根据A、B之间的平面距离扣分；**不按车从上一帧实际走了多少米扣分，也不把候选位置发给底盘执行。**

值较小会更偏向A附近，可能减少被相似结构拉走的机会，也可能让正确的位姿修正受到压制；值较大则减轻这项约束。它只改变已有候选的分值，不代替前面的搜索范围、轮廓质量或匹配门槛。

###### angle_variance_penalty（候选朝向偏离估计时要扣多少分）

含义相同，但比较的是内部二维扫描位姿的yaw候选与本次搜索中心朝向之差。估计已经包含的正常转动不会直接当作这项偏离；惩罚针对算法额外尝试的朝向修正。

这里不是串口JointState中的云台机械相对yaw，也不是Nav2／spin输出的wz。调它不能规定底盘或云台最多转几度，不能修复随云台转动时雷达与底盘转换错误。

###### 配置值先平方，单位与名称容易混淆

按当前2.6.10的赋值链，setParamDistanceVariancePenalty和setParamAngleVariancePenalty都会先将传入值平方。表中的m、rad按这个赋值链及评分公式的量纲解释；内部使用的才是m²、rad²，不能因为名称带variance就把YAML值直接当方差。[赋值与评分实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

设d为候选相对搜索中心的平面偏移，θ为朝向偏移（弧度）。使用当前配置，计算方式为：

- 距离系数：`max(0.5, 1 − 0.2 × d² / 30.25)`。
- 角度系数：`max(0.9, 1 − 0.2 × θ² / 25)`。
- 候选分值＝轮廓吻合原始分值×距离系数×角度系数。

0.5、0.9来自后面的minimum_distance_penalty、minimum_angle_penalty，表示系数下限：距离项至多将分值乘0.5，角度项至多乘0.9，并非允许距离或角度门槛。

纯公式算例：偏移20 cm时，距离系数约0.99974；朝向偏移10°时，角度系数约0.99976。当前数值对这种程度的偏移扣分很轻；这是公式结果，不是定位精度或实车验证结果。

粗搜索参考当前估计扫描位姿，内部精搜索参考粗匹配所得位姿；并非两个阶段始终相对同一个原始里程计位置扣分。

###### 生效路径与实物核对

近期扫描匹配默认启用这项惩罚；普通近邻链关联、自动闭环的粗匹配和精匹配调用则明确关闭惩罚。因此，**开启扫描匹配后也不是所有匹配都使用这两项**；闭环虽复用普通匹配器精化，仍会跳过这里的扣分。

实物／仿真nav2_params.yaml及安装副本的两项当前值一致，由slam_launch传给sync_slam_toolbox_node。SMapper::configure读取后设置到Karto；ROS包装层未提供参数时的默认值为0.5、1.0，与本工程5.5、5.0不同。[参数读取实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_mapper.cpp)

应使用有限正数：此读取链没有非正值自动回退；负数会被平方，0会使评分分母为0。不能用0表示关闭惩罚，也不能把运行中设置ROS参数视为内部评分尺度已同步更新。

- **源码能确认：** 先平方再评分、正值越大惩罚越轻；参考本次搜索中心；当前未参与评分，近邻关联与闭环也不使用这项惩罚。
- **已有验证范围：** 独立Point-LIO静止与约20 cm低速往返，未验证本节点的平移／转角修正或惩罚效果。
- **实物待核实：** 若以后启用扫描匹配，观察是否被重复结构拉偏、是否压制必要修正；先核对坐标转换、点时间和稳定二维轮廓。坡道识别仍应检查地形分析与扫描生成，不能靠这两个评分尺度补救。

##### 角度搜索与扣分下限参数表

当前use_scan_matching=False，这6项会被读取，但不参与当前自动建图流程的匹配搜索和评分。若以后开启匹配，当前角度搜索可概括为：**粗搜左右各约20°，每2°试一次；精搜在粗结果左右各约1°内，每0.2°试一次。**

```yaml
    fine_search_angle_offset: 0.00349
    coarse_search_angle_offset: 0.349
    coarse_angle_resolution: 0.0349
    minimum_angle_penalty: 0.9
    minimum_distance_penalty: 0.5
    use_response_expansion: true
```

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `fine_search_angle_offset` | `0.00349` | rad／步，约0.2° | **当前运行不使用这个参数，暂不调整** | 控制内部精搜索的yaw采样步距 | 精搜索更稀，候选通常减少，可能漏掉较窄的匹配峰 | 精搜索更密、计算增加；不保证角度更准 | 当前不用；此版本实际作为步距，不是精搜索半范围；当前精搜索半范围约1° |
| `coarse_search_angle_offset` | `0.349` | rad，粗搜索半范围，约20° | **当前运行不使用这个参数，暂不调整** | 在估计扫描朝向两侧尝试yaw修正 | 粗搜索范围扩大、候选和计算增加，也更易遇到相似朝向 | 范围缩小、候选减少，真实朝向修正可能落在范围外 | 当前不用；粗搜索约±20°，总宽约40°；不是底盘／云台转角上限 |
| `coarse_angle_resolution` | `0.0349` | rad／步，约2° | **当前运行不使用这个参数，暂不调整** | 控制粗yaw步距，同时决定内部精搜索半范围 | 粗角度采样更稀；精搜索半范围同时扩大，计算不能只看粗候选数量 | 粗角度采样更密；精搜索半范围同时缩小 | 当前不用；精搜索半范围＝本项÷2，当前约±1°；需与fine步距配套 |
| `minimum_angle_penalty` | `0.9` | 无量纲，角度扣分系数下限 | **当前运行不使用这个参数，暂不调整** | 限制角度项最多能扣多少分 | 下限提高，触底时扣分更轻；设1可取消该项扣分 | 下限降低，允许角度项扣更多分；未触底时无变化 | 当前不用；仅在内部调用启用惩罚时使用，0.9不是0.9 rad或90%的匹配成功率 |
| `minimum_distance_penalty` | `0.5` | 无量纲，距离扣分系数下限 | **当前运行不使用这个参数，暂不调整** | 限制距离项最多能扣多少分 | 下限提高，触底时扣分更轻；设1可取消该项扣分 | 下限降低，允许距离项扣更多分；未触底时无变化 | 当前不用；0.5不是0.5 m，亦非最终分数门槛；当前公式通常未达到这个下限 |
| `use_response_expansion` | `true` | 布尔值 | **当前运行不使用这个参数，暂不调整** | 粗响应近似为0时扩大角度范围重试 | True：满足触发条件时追加角度搜索，增加计算和相似朝向候选 | False：不追加扩角尝试，仍可执行正常粗搜／精搜 | 当前不进入匹配；最多追加3次，每次半范围增加20°；只扩角，不扩平移或雷达量程 |

##### 角度搜索与扣分下限详解

###### fine_search_angle_offset（精搜索每隔约0.2°试一次）

名称和部分教程把它说成“精搜索角度范围”，但当前源码将它传给角度步距位置；真正的精搜索半范围取0.5×coarse_angle_resolution，当前为0.01745 rad，约1°。

因此，当前是在粗匹配朝向附近约±1°内，每0.00349 rad（约0.2°）试一次，共11个角度候选。减小本项只让这段范围内试得更密，不会扩大范围，也不表示最终朝向误差能达到0.2°。[粗细搜索的实际调用](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

这里的内部精搜索用于普通匹配器精化；自动闭环的大范围粗匹配不会在闭环网格上执行这一精化，后续调用普通匹配器时才使用它。

###### coarse_search_angle_offset（估计朝向左右各约20°粗搜）

0.349是弧度，约20°，表示当前估计二维扫描朝向两侧的粗搜索半范围。它搜的是**估计还需要修正多少角度**，不是车辆两帧之间允许转多少度。

例如车正常转弯30°，若里程计已正确反映这次转动，匹配无需额外修正30°。不能看到底盘或云台转得多，就直接把此项按同样角度增大。它也不负责pitch／roll、坡度识别或云台机械限位。

###### coarse_angle_resolution（粗搜步距约2°，还影响精搜范围）

当前粗搜索从约−20°到+20°，每约2°取一个候选，共21个角度；随后在粗结果两侧各半个粗步距，即约±1°内精化。

调整时要一起看前两项：本项变大，粗候选减少，但内部精搜索范围会变宽；本项变小，粗候选增多，但精搜索范围会变窄。源码按round(总范围／步距)+1取候选数，宜保持范围与步距能够合理分段，不能把粗、细两项当成互不相关的独立旋钮。

###### minimum_angle_penalty（角度项至少保留多少分）

0.9表示**仅角度惩罚这一项最多扣掉10%**。实际系数取“公式算出的系数”和0.9中较大的一个：若公式算0.98，就用0.98，并非每次固定乘0.9；若算0.7，则保留到0.9。

它不是角度门槛或最终匹配分数下限。后面还要乘距离系数，所以总分并不保证保留90%。

###### minimum_distance_penalty（距离项至少保留多少分）

同理，0.5表示仅距离惩罚这一项最多扣掉50%。**增大下限会减轻触底后的惩罚，减小才允许扣更多分。**

结合前一块已经核实的平方分母，当前普通粗搜索x、y各约±0.25 m时，距离系数最低约0.99917；即使粗角度扩至约±80°，角度系数仍约0.9844。二者均高于当前0.5、0.9下限，所以即使以后开启普通匹配，单改这两个下限也可能看不到变化。这是按当前搜索边界计算的结果，未做运行验证。

两项下限按正常评分语义取0～1。设1会使对应项不再扣分；设0仅允许系数降到0，不代表关闭惩罚。源码没有完整的区间校验，不应以超出区间的值表示更强或更弱的正常惩罚。[评分系数与下限实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

###### use_response_expansion（粗响应接近0时，只扩大角度重试）

开启后，源码先完成正常粗搜；仅当最佳粗响应与0近似相等，才追加尝试：

- 每次把粗搜索半范围再增加20°，最多3次。当前约±20°起步，依次尝试约±40°、±60°、±80°。
- 一旦粗响应不再近似为0，就停止追加扩角；不是低于闭环0.35／0.45分数门槛就一直扩展。
- 平移范围、角度步距和雷达量程保持原值；它不去全图寻找位置，也不属于small_gicp的PCD重定位。

这些角度参数及扩展可用于普通匹配、普通近邻关联和自动闭环搜索；两项扣分下限则只用于启用惩罚的调用，普通近邻关联和闭环调用仍跳过扣分。约±80°描述粗搜索，后续精化还能略越过边缘，不能把它当成车辆转动限制。

扩角可能增加耗时或匹配到相似轮廓；拿到非零响应也不证明定位正确。若实物中轮廓缺失、动态遮挡，或云台转动／坡道导致二维轮廓变化，扩大yaw范围不会恢复缺失信息。[零响应扩角与停止条件](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/lib/karto_sdk/src/Mapper.cpp)

###### 读取、教程差异与验证范围

实物／仿真nav2_params.yaml及安装副本的6项值一致，随slam_launch传给sync_slam_toolbox_node。SMapper::configure读取后直接设置到Karto；minimum_distance_penalty的ROS包装层缺省值为0.05，本工程显式覆盖为0.5。[参数读取实现](https://github.com/SteveMacenski/slam_toolbox/blob/2.6.10/src/slam_mapper.cpp)

fine和coarse步距应使用有限正数；读取链没有对非正值自动回退的处理，也没有运行中修改这些ROS参数后同步更新内部匹配器的回调。教程里的rough_search_angle_offset不属于此处实际读取的参数名，当前键名为coarse_search_angle_offset；“自动增大网格”的泛化描述也应按这里实际只扩角理解。

- **源码能确认：** 弧度单位、粗搜约±20°／2°步距、精搜约±1°／0.2°步距；下限控制最多扣分；响应近0才扩角，最多3次，平移不扩展。当前自动匹配未启用。
- **已有验证范围：** 仅独立Point-LIO静止与约20 cm低速往返，未验证本节点转弯匹配、扩角、闭环或耗时。
- **实物待核实：** 若以后启用匹配，观察残余朝向误差、重复结构误配、轮廓变化与计算耗时；先核对雷达／IMU／底盘转换及点时间，云台机械相对角不应直接替代内部二维扫描朝向。

