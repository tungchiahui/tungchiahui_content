---
title: "velocity_smoother（速度限幅与加减速平滑）"
---

## velocity_smoother（速度限幅与加减速平滑）

### 实车先看：平滑哪一路速度、反馈来自哪里

**当前velocity_smoother处理controller_server输出的速度指令，逐轴限目标速度、限速度变化，再处理死区。OPEN_LOOP使用上一轮内部指令作当前速度，不用里程计实测速度；scale_velocities=False仍会限速和限加减速。** 本节点处理Twist，不修改规划路径，也不查询代价地图判断碰撞。

```text
普通路径跟踪，两种navigation_launch分支相同：
    controller_server／OmniPID
        → cmd_vel_controller
        → velocity_smoother
            ├─ 最新目标速度先夹到各轴min／max范围
            ├─ OPEN_LOOP：以内部last_cmd作为当前速度
            ├─ 按加／减速参数约束本轮变化
            └─ 内部记录 → 应用死区 → 发布
        → cmd_vel_nav2_result
        → fake_vel_transform：换平移轴，再加cmd_spin
        → cmd_vel → 串口／执行端

恢复行为：
    组件分支behavior_server → cmd_vel_nav2_result，绕过平滑器
    独立分支behavior_server → namespace/cmd_vel，另有接线问题
```

当前launch把平滑器内部cmd_vel订阅重映射为cmd_vel_controller，把cmd_vel_smoothed发布重映射为cmd_vel_nav2_result；不是直接套用教程中的cmd_vel_nav→cmd_vel名称。恢复与下游自旋的限速条件另见[behavior_server](/wiki/2026-09-20-dao-hang/0500-1800-behavior-server)和[fake_vel_transform](/wiki/2026-09-20-dao-hang/0500-0600-fake-vel-transform)。

| 实车上看到的现象 | 先核对什么 | 当前配置怎样理解 |
|---|---|---|
| 认为False关闭了整个平滑功能 | scale_velocities控制的分支 | False只关闭统一比例的变化量缩放，各轴仍限幅与限加减速 |
| 以为min_velocity要求至少2.5 m/s才动 | 带符号的速度范围 | -2.5是负向下界，0和小的正／负速度都在允许范围内 |
| 起步或转向时方向暂时与目标不一致 | 各轴独立约束及目标夹紧 | False各轴分别变化；True主要同比缩放Δ速度，也不能保证所有情况下保持原始目标方向 |
| 修改odom_duration后当前输出没变 | feedback是否仍为OPEN_LOOP | 正常初始化不建立反馈里程计订阅，窗口不参与当前速度估计 |
| 收到零指令后还缓慢降速 | 零目标也经过减速约束 | 当前零指令不是直接透传的急停信号 |
| 指令断流满1秒后没有立刻归零 | 超时先置零目标，再减速 | 1 s是开始超时处理的门槛，不是整个停车时间 |
| 设置非零死区后，小速度直接跳到某值 | 死区应用顺序和内部last_cmd | 先记内部平滑值，再把小幅输出置零；死区切换可造成发布值跳变 |
| 平滑器角速度没超过3，最终角速度却更大 | 下游cmd_spin相加 | 3 rad/s约束本输入目标，不约束相加后的最终总量 |
| 提高频率，以为每秒加速度也必须提高 | 每轮步长和定时器节拍 | 每轮变化量按加速度／频率计算；名义每秒限制保持相同，离散步长更小 |

本次安装的nav2_velocity_smoother和nav2_util均为**1.1.20**。VelocitySmoother与OdomSmoother安装头文件匹配该版本，平滑器库存在；本页依据对应官方源码及当前工程接线说明，不把后续版本的TwistStamped或其他参数直接套入。[平滑器源码](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp)、[里程计窗口源码](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_util/src/odometry_utils.cpp)

### 参数

下面原样保留发送的12项参数。源码与安装副本的reality段一致；simulation段只有use_sim_time=True不同。本次只写文档，未改工程配置或运行参数。

```yaml
velocity_smoother:
  ros__parameters:
    use_sim_time: False
    smoothing_frequency: 20.0
    scale_velocities: False
    feedback: "OPEN_LOOP"
    max_velocity: [2.5, 2.5, 3.0]
    min_velocity: [-2.5, -2.5, -3.0]
    max_accel: [4.5, 4.5, 5.0]
    max_decel: [-4.5, -4.5, -5.0]
    odom_topic: "odometry"
    odom_duration: 0.1
    deadband_velocity: [0.0, 0.0, 0.0]
    velocity_timeout: 1.0
```

所有三元素数组顺序都是**[x，y，theta]**，对应Twist的linear.x、linear.y、angular.z。前两项是平移，第三项是绕z轴的旋转；theta这一项不是以rad为单位的目标角度。

#### ros__parameters（时钟、频率与反馈方式）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `use_sim_time` | `False`；仿真True | 不适用，布尔 | **上实物前必须核对并正确配置** | 选择now()及输入超时比较的ROS时间域 | 不适用 | 不适用 | launch重写已有键；平滑定时器用wall timer，True不会把它变为仿真时钟定时器 |
| `smoothing_frequency` | `20.0`；源码默认20.0 | Hz，平滑定时回调目标频率 | **实物运行时重点观察，有问题再针对性调整** | 决定输出节拍及每轮加减速变化上限 | 名义每轮允许变化更小、回调更密、计算和发布增加 | 名义每轮变化更大、命令更稀、离散过渡更粗 | 当前名义50 ms／轮，线速度步长0.225 m/s、角速度步长0.25 rad/s；用配置频率而非实测dt；必须为有效正数 |
| `scale_velocities` | `False`；源码默认False | 不适用，布尔 | **根据实际功能需求开启或设置** | 是否先选统一eta同比缩放各轴目标变化量，再分别约束 | 不适用 | 不适用 | False仍逐轴平滑；True主要协调Δvx、Δvy、Δwz，不是始终保持最终速度方向；本版没有动态更新该布尔成员的回调分支 |
| `feedback` | `"OPEN_LOOP"`；源码默认同值 | 不适用，模式名称 | **通常保持当前值，不作为优先调试项** | 选择当前速度来自内部last_cmd或里程计平均Twist | 不适用 | 不适用 | OPEN_LOOP使用死区前内部指令；CLOSED_LOOP使用OdomSmoother；改闭环前须核对速度对象、参考系、时间、频率和数据新鲜度 |

##### 参数详解

###### smoothing_frequency（定时器、步长与上游控制频率）

Active阶段用wall timer触发smootherTimer。当前1000／20=50 ms；定时器周期转换为整数毫秒，其他频率可能有截断，例如33 Hz会创建30 ms周期，而变化上限仍按配置33计算。因此不能把任何设置值都视为精确实测频率。[定时器创建](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L120)

每轮允许变化按参数除以smoothing_frequency计算，**没有根据上次实际回调时间测dt**。负载导致周期延迟时，真实每秒的变化过程会受影响；这也是指令平滑约束，不代表实测机械加速度已被限制到同样数值。

当前controller_server也为20 Hz，所以两者名义频率相同；提高平滑频率可在两条控制指令之间继续生成过渡输出，但不会提高规划／路径跟踪本身的计算频率。两定时循环也没有自动锁定相位，不能保证每轮输出都对应刚收到的一条新控制指令。[控制器频率](/wiki/2026-09-20-dao-hang/0500-1100-controller-server)

例如提高到100 Hz而保持加速度不变，名义线速度每轮上限从0.225降为0.045 m/s，角速度从0.25降为0.05 rad/s；理想每秒仍对应4.5 m/s²和5 rad/s²。这是参数计算，未改变当前频率或测量输出。

###### OPEN_LOOP（内部命令反馈，不等于机器人实测速度）

OPEN_LOOP把上一轮last_cmd作为本轮current速度，再向最新目标靠近。当前死区为0时，它等于上一轮发布的三个有效分量；若以后启用非零死区，last_cmd记录在死区置零之前，就不一定等于实际发布值。[反馈选择与记录顺序](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L246)

这种模式默认车辆能跟上此前指令，不能从中知道打滑、卡住、驱动限流或真实速度滞后，也不会因收到odometry自动切成实测闭环。当前启动读取odom_topic和odom_duration，但不创建OdomSmoother，不用它们参与current速度。

CLOSED_LOOP改用窗口内里程计Twist均值；这里的“闭环”是平滑器以实测反馈为限变化的起点，并非额外建立一个车轮速度PID。反馈频率低或一直为0时，每轮可能都从同一个反馈速度重新算，不能像开环那样自然累计已发命令。

###### scale_velocities（缩放速度变化量，不是关闭限幅）

先把目标各轴夹到min／max范围，再计算dv=目标−current。False令eta=1，各轴分别夹紧自己的dv；True从受约束轴中找更严格的统一比例eta，先缩放全部dv，再按各轴自己的约束夹紧。[eta与逐轴约束](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L263)

按当前频率、从零起步、目标已在速度范围内且死区为0，举例：

| 请求目标[vx，vy，wz] | scale_velocities=False首轮 | scale_velocities=True首轮 |
|---|---|---|
| [1.0，0.1，1.0] | [0.225，0.1，0.25] | eta=0.225，得到[0.225，0.0225，0.225] |

前两项单位m/s，第三项rad/s。False能让小需求轴先达到目标，但首轮平移方向不按原请求的10∶1比例；True在此例协调变化比例，代价是其他轴也更慢接近。这是源码公式示例，不是车辆轨迹测试。

current不为0时，同比的是**从current到目标的变化向量**，不是直接把整个目标速度乘同一个数。先行的各轴绝对速度夹紧、后续逐轴夹紧和死区也可能改变最终比例，不能把True理解为任何情况下都保证原始速度方向不变。

#### 速度与加减速数组（每轴范围、带符号下界）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `max_velocity` | `[2.5, 2.5, 3.0]` | [m/s，m/s，rad/s] | **上实物前必须核对并正确配置** | 设置各轴目标指令的上界 | 正上界提高时允许更大的正向目标 | 正上界减小时更早夹紧正向目标 | 独立轴范围，不是平移合速度2.5；只约束经过本节点的目标，不能限制恢复旁路或下游附加自旋 |
| `min_velocity` | `[-2.5, -2.5, -3.0]` | [m/s，m/s，rad/s] | **上实物前必须核对并正确配置** | 设置各轴目标指令的带符号下界 | 负值向0提高时，允许的反向幅值减小 | 负值更负时，允许的反向幅值增大 | 不是克服静摩擦的最低非零速度；当前零、小幅正负都允许；须与每轴max保持min≤max |
| `max_accel` | `[4.5, 4.5, 5.0]` | [m/s²，m/s²，rad/s²] | **实物运行时重点观察，有问题再针对性调整** | 同号且目标幅值不小时，限制每轮速度变化 | 允许更快增速、过渡更短，驱动能力需匹配 | 起步／增速更缓，可能增加跟踪滞后 | 各活动轴用非负值；当前每轮±[0.225,0.225,0.25]；不是车辆jerk限制，也不是PID增益 |
| `max_decel` | `[-4.5, -4.5, -5.0]` | [m/s²，m/s²，rad/s²]，带负号 | **实物运行时重点观察，有问题再针对性调整** | 目标幅值减小或反向时，限制每轮速度变化 | 负值向0提高时，减速幅值变小、降速更慢 | 负值更负时，减速幅值变大、降速更快 | 本版要求非正；当前每轮幅值与加速相同；零目标和超时置零都经过它，不宜把数值更大理解为制动更强 |

##### 参数详解

###### max／min_velocity（x、y独立夹紧，不是合速度上限）

当前目标限制为vx、vy各在[-2.5,2.5] m/s，wz在[-3,3] rad/s。例如目标(vx,vy)=(2.5,2.5)没有超出任何一轴，平移模长却为sqrt(2.5²+2.5²)≈3.54 m/s。仅本平滑器的这组数组不能推出模长≤2.5。[目标夹紧](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L254)

当前OmniPID另有标量平移速度范围和计算过程；普通跟踪实际能达到多少要看上游输出。这里的3.54只是本数组允许的目标组合，不是证明当前PID或实车已经输出该速度。min_velocity中的-2.5允许负向运动，没有把0.1 m/s之类的小速度抬到2.5 m/s。[PID速度约束](/wiki/2026-09-20-dao-hang/0500-1100-controller-server)

速度范围在**平滑输入的坐标轴**上施加。当前平移指令按gimbal_yaw_fake约定，fake_vel_transform后再换到实际gimbal_yaw；平面纯旋转换轴保持平移模长，但不能保证转换后每个分量仍满足同一轴上界。角速度又会加cmd_spin，最终值不由本数组统一夹紧。

源码夹的是目标，没有在加减速计算后再做一次最终速度夹紧。当前开环、固定边界且内部初值在范围内时，会逐步靠近范围内目标；若改闭环且反馈本身超界，或运行中缩小边界，过渡输出仍可能暂时超出新界。例如current.vx=3、目标夹到2.5、20 Hz减速步长0.225时，本轮得2.775 m/s。此例说明实现顺序，未运行闭环或改变限值。

###### max_accel与max_decel（符号、加速判据与反向）

每轴依据下面条件选择变化区间：

```text
若 abs(v_target) >= abs(v_current)，且 v_current × v_target >= 0：
    使用加速：Δv ∈ [-max_accel/f，+max_accel/f]
否则：
    使用减速：Δv ∈ [max_decel/f，-max_decel/f]

v_next = v_current + clamp(eta × (v_target-v_current)，Δv下界，Δv上界)
```

f为配置的smoothing_frequency。负方向增速仍使用正max_accel来定义变化幅值；正方向减速则使用负max_decel，不是按“正vx用accel、负vx用decel”区分。跨零反向的这一轮先按减速区间处理，之后是否改用加速由下一轮current和目标重新判断。[逐轴公式](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L208)

当前名义50 ms一轮：线速度最多变0.225 m/s，角速度最多变0.25 rad/s。从零到单轴2.5 m/s至少需要12次名义更新；从零到3 rad/s也需要12次，约0.6 s周期尺度。上游目标持续变化、其他轴统一缩放、定时器相位及实际驱动反馈会改变具体过程；这里不是实测起步时间。

max_decel从-4.5改成-2，虽然数值提高，幅值却减小，减速更慢；改成-6则幅值更大。应按车辆制动能力和任务响应调整，不能随手去掉负号。当前OmniPID还按0.9 m/s²限制标量速度变化，它与本逐轴4.5 m/s²不是同一变量；最终命令受串联过程共同影响，恢复旁路也不自动受到两者的相同约束。

#### 里程计反馈（当前OPEN_LOOP不参与估计）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `odom_topic` | `"odometry"`；源码默认odom | 不适用，反馈话题名称 | **当前运行不使用这个参数，暂不调整** | CLOSED_LOOP时供OdomSmoother订阅Odometry速度 | 不适用 | 不适用 | 当前开环启动不建立该反馈订阅；闭环直接读twist，不差分pose、不做TF变换；动态改此键仍会重建一个OdomSmoother，但开环current分支不会改用它 |
| `odom_duration` | `0.1`；平滑器声明默认0.1 | s，反馈里程计历史窗口 | **当前运行不使用这个参数，暂不调整** | CLOSED_LOOP按消息时间戳保留窗口并对速度做样本均值 | 闭环通常平均更多样本、减弱波动，但反馈延迟可能增大 | 闭环通常平均更少样本、响应更快，噪声影响可能更大 | 不是输入超时或输出延迟固定值；当前开环不参与current；具体样本数由里程计频率／时间戳决定 |

##### 参数详解

###### odom_duration（消息窗口中的样本平均）

CLOSED_LOOP才在配置时创建OdomSmoother，直接累加Odometry.twist.twist，再除以保留的消息数。新消息到来时按“最新消息时间戳−最早消息时间戳>odom_duration”移除旧消息，因此窗口以消息stamp衡量，且边界等于0.1 s时仍保留。不是连续时间积分平均，也不按协方差加权。[窗口与均值](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_util/src/odometry_utils.cpp#L60)

例如时间戳严格等间隔20 ms且稳定收包时，0.1 s窗口名义可保留6个样本（含两端）；实际时间戳抖动会改变数量，不是固定五帧。odom_duration=0.1也不表示等待0.1 s收满才允许输出。

窗口只在收到新里程计时更新；getTwist返回缓存，没有按now检查其是否过期。里程计断流不由velocity_timeout检测，后者只检测速度指令断流；因此切换闭环前必须另行核对反馈新鲜度。当前开环不受这条里程计缓存分支影响。[缓存读取](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_util/include/nav2_util/odometry_utils.hpp#L75)

###### 当前odometry不能未经核对就直接作为闭环真值

当前发布者sensor_scan_generation给出的是gimbal_yaw位姿及其差分Twist，平移差分在odom轴上计算，时间分母使用回调steady_clock，还存在首帧处理问题；它不等于已经验证的底盘速度测量。平面条件下odom轴可能与gimbal_yaw_fake输入轴近似一致，也不能因此跳过时间、原点和速度对象核对。[上游速度实现与问题](/wiki/2026-09-20-dao-hang/0500-0300-sensor-scan-generation)

尤其angular.z可能包含实际云台独立yaw运动，而平滑器的导航旋转分量与下游附加底盘自旋又是不同来源。OdomSmoother不会根据child_frame_id自动换系或分离这些角运动；切成CLOSED_LOOP前，应明确反馈的是哪个实体、哪个参考点及哪一套轴下的速度。当前保持开环不意味着这些上游问题已经被修复。

#### 死区与输入超时（低速置零、超时后减速）

##### 参数表

| 参数名 | 当前值 | 单位 | 实车处理建议 | 作用 | 调大效果 | 调小效果 | 生效条件／备注 |
|---|---|---|---|---|---|---|---|
| `deadband_velocity` | `[0.0, 0.0, 0.0]`；源码默认同值 | [m/s，m/s，rad/s]，速度幅值阈值 | **通常保持当前值，不作为优先调试项** | 将平滑结果中绝对值严格小于阈值的分量置零 | 更多低速分量被压为0，可能减少微动，也可能丢失小幅指令 | 保留更多小幅输出，低速噪声／驱动能力需观察 | 当前全0不剔除非零小速度；不是Δ速度噪声门槛，也不会把小速度抬高到最小值；阈值按各轴幅值配置 |
| `velocity_timeout` | `1.0`；源码默认1.0 | s，从最后有效Twist接收起算的ROS时长 | **实物运行时重点观察，有问题再针对性调整** | 当now−last_command_time严格超过此值时，把目标改为0并减速 | 可容忍更长指令间断，也更久保留旧目标 | 更快触发降速，也更易受正常抖动触发 | 不是收到零指令后的等待时间；不检测里程计／串口断流；零目标仍经过max_decel，不保证1秒立刻停车 |

##### 参数详解

###### deadband_velocity（比较速度绝对值，不是相邻消息差）

源码先计算受加减速约束的结果并写入last_cmd，再对发布结果做abs(v)<deadband置零。当前全0时，不会因为“速度变化很小”而忽略消息，也不改变任意非零小速度；等于阈值的分量不被置零。[死区处理顺序](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L288)

若以后把x死区设0.3 m/s，按当前开环步长、从零朝1 m/s起步：第一轮内部0.225而发布0，第二轮内部0.45而发布0.45。这会使发布值从0直接跳到0.45，不能据内部0.225步长宣称死区后的每轮输出仍严格遵守同一变化上限。此为顺序计算例，未改当前全0死区。

所以死区适合在确认驱动最低可执行速度、微动现象和到点需求后再设置；它不会增加定位精度，也不是控制器的min_x／min_y_velocity_threshold。控制器那些门槛处理输入里程计，当前这个门槛处理平滑后的发布速度。

###### velocity_timeout（先等待门槛，再把零当目标平滑）

收到有效Twist时保存目标并把last_command_time更新为接收时的now；NaN或Inf消息被拒绝，不刷新这个计时。Twist本身没有时间戳，因此这里测的是接收间隔，不是原始指令生成时刻或网络总延迟。[输入回调](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L168)

没有收到过任何指令时，定时回调直接返回，不持续发布零。收到指令后，即使暂时没有新消息，也继续按最新目标平滑；当ROS时间差严格>1 s，在某次回调把command改为零，再按当前减速参数生成输出。显式收到零指令则立即把它作为新目标，不先等待这1秒。[超时分支](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L229)

例如超时触发时内部单轴速度为2.5 m/s，当前开环、死区0、20 Hz和减速幅值4.5下，需要12次名义减速更新才能归零，约0.6 s周期尺度。加上前面的1 s门槛与定时相位，不能写成“失去输入1秒内必定停住”；物理停车还受执行端影响。本例是离散公式换算，不是断流实测。

超时后内部last_cmd已精确为零或stopped=True时，函数直接返回，不再持续发布零。当前开环能逐步到达精确零，随后停止发布；闭环或非零死区情形仍应结合内部状态与真实反馈判断，不能把停止发布当作车已静止的测量。

本超时使用ROS时钟，平滑定时器却使用墙钟。use_sim_time=True且/clock暂停时，回调可能仍运行、按最新目标继续生成指令，而1 s超时差不再增长。恢复旁路也不受这条输入超时统一管理；平滑器发零后，fake_vel_transform仍可能加上保留的cmd_spin。

#### 动态参数、配置边界与验证范围

##### 当前1.1.20哪些值会动态进入成员

本节点在activate安装回调，deactivate移除；本版不能把“ROS参数设置成功”一律理解成所有内部设置已联动更新。[动态参数实现](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L315)

| 参数组 | Active回调行为 | 本版需要注意 |
|---|---|---|
| smoothing_frequency | 更新double成员，取消旧timer并按新值创建wall timer | 与航点loop_rate不同，本节点确实重建节拍；仍有整数毫秒截断 |
| velocity_timeout | 把double秒值转为Duration更新 | 不清空已保存目标或重设最后接收时间 |
| max_velocity、min_velocity、max_accel、max_decel、deadband_velocity | 接受三元素double数组并更新成员 | 回调不完整复核min≤max；缩小范围不重置last_cmd；应使用完整有效数组 |
| feedback | OPEN_LOOP重置OdomSmoother；CLOSED_LOOP创建新OdomSmoother | 模式切换会改变current来源，不等于已验证反馈；新窗口需重新接收数据 |
| odom_topic、odom_duration | 更新对应成员并重建OdomSmoother | 即使当时OPEN_LOOP也可能建立订阅，但current仍走开环分支；旧窗口不沿用 |
| scale_velocities | 没有PARAMETER_BOOL更新分支 | ROS层值可能改变，成员仍为configure时值；需重新配置／重启后核对 |
| use_sim_time | rclcpp时间源机制处理 | 不把wall timer改成ROS时钟节拍；仍会影响超时比较 |

此外，max_accel／max_decel动态回调检查符号后，即使判为失败也仍有赋值成员的代码，批量设置还可能先更新前面的成员再遇到后项失败。不能依赖“设置失败”证明内部完全未变；本次没有运行这种非法设置实验。[符号检查与赋值](https://github.com/ros-navigation/navigation2/blob/1.1.20/nav2_velocity_smoother/src/velocity_smoother.cpp#L350)

启动配置要保持三元素数组、加速非负、减速非正、各轴min≤max及有效频率／时间；当前这些数值符合源码的基本符号和数组要求，但是否匹配实际驱动能力仍需实物数据。平滑器没有jerk约束，没有统一的车体合速度限幅，也没有代价地图碰撞检查；机械能力与碰撞处理不能只靠这组参数证明。

##### 本次核对与实物观察

- **已核对：** nav2_velocity_smoother／nav2_util版本、两份安装头文件与库；reality／simulation参数及安装副本；两种navigation_launch的控制器和平滑器重映射、恢复旁路；上游odometry生成和下游速度换系／自旋。
- **源码能确认：** 十二项参数的使用条件、三轴顺序与单位；目标夹紧→变化约束→内部记录→死区发布的顺序；开环和闭环窗口区别；超时零目标降速、ROS与墙钟差异；动态参数覆盖及未覆盖分支。
- **实物观察时重点：** 同时对照cmd_vel_controller、cmd_vel_nav2_result及最终cmd_vel，观察起步、减速、横移／方向切换和驱动实际响应；确认附加自旋、恢复旁路和控制指令断流各自的处理结果。若保持开环，先不以odom_duration作为调试输出平滑的入口。
- **本次范围：** 仅更新文档和目录，未改变运行参数或工程代码，未发布速度、未启动导航／恢复、未进行运动或断流测试；所有数值例子均为源码公式计算。
