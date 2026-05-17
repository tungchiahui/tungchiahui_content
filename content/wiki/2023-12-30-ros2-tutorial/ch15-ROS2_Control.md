---
title: "ROS2_Control"
---

### ROS2 Control 概述

#### ROS2 Control 简述

**概念**

`ros2_control` 是 ROS 2 中用于机器人实时控制的框架。它把机器人控制拆成三层：上层控制器、中间的控制器管理器、底层硬件抽象。这样做的目的，是让同一套控制器既可以接真实电机驱动，也可以接仿真插件或模拟硬件。

在 Jazzy 版本中，官方文档将 `ros2_control` 定义为一个用于 ROS 2 机器人实时控制的框架，它的核心仓库是 `ros2_control`，常用控制器仓库是 `ros2_controllers`。官方文档还明确列出 `diff_drive_controller`、`joint_trajectory_controller` 等常用控制器属于 `ros2_controllers` 仓库。

在 Ubuntu 24.04 安装 ROS 2 Jazzy 后，默认安装路径是：

```bash
/opt/ros/jazzy
```

安装 ros2_control 相关软件包后，常见包路径如下：

| 软件包 | 路径 | 说明 |
|:---|:---|:---|
| `hardware_interface` | `/opt/ros/jazzy/share/hardware_interface` | 硬件抽象接口基础包 |
| `diff_drive_controller` | `/opt/ros/jazzy/share/diff_drive_controller` | 差速底盘控制器 |
| `joint_state_broadcaster` | `/opt/ros/jazzy/share/joint_state_broadcaster` | 关节状态发布器 |

其中，`hardware_interface` 的 `package.xml` 描述为“Base classes for hardware abstraction and tooling for them”。`diff_drive_controller` 的 `package.xml` 描述为“Controller for a differential-drive mobile base”。`joint_state_broadcaster` 的 `package.xml` 描述为“Broadcaster to publish joint state”。

#### ROS2 Control 作用

在一个机器人系统中，`ros2_control` 主要解决“谁来读硬件、谁来算控制、谁来写硬件”的问题。

*   **硬件抽象** ：真实机器人通常有电机、编码器、舵机、传感器、驱动板等不同硬件。`hardware_interface` 负责定义统一接口，让上层控制器不直接依赖具体硬件通信方式。

*   **控制器管理** ：`controller_manager` 连接控制器和硬件抽象层。官方文档说明，控制器管理器负责加载、配置、激活、停用和卸载控制器，也负责匹配控制器需要的接口与硬件提供的接口。

*   **控制循环** ：官方文档说明，控制器管理器的 `update()` 会执行控制循环：从硬件组件读取状态，更新所有激活的控制器，再把控制结果写回硬件组件。

*   **控制器复用** ：同一个控制器可以接不同硬件实现。例如差速底盘控制器只关心左右轮关节的速度命令和反馈，不需要知道底层是串口、电机驱动器、Gazebo 还是 mock 硬件。

#### ROS2 Control 安装

官方 Jazzy 文档给出的 Ubuntu 二进制安装命令如下：

```bash
sudo apt install ros-jazzy-ros2-control ros-jazzy-ros2-controllers
```

如果是 RHEL 系统，官方文档给出的命令是：

```bash
sudo dnf install ros-jazzy-ros2-control ros-jazzy-ros2-controllers
```

本章后续命令均按 Ubuntu 24.04 的 deb 包环境书写。每次打开新终端后，都需要先 source 环境：

```bash
source /opt/ros/jazzy/setup.bash
```

可以用如下命令查看是否已经安装相关包：

```bash
ros2 pkg list | grep -E '^(controller_manager|hardware_interface|diff_drive_controller|joint_state_broadcaster|joint_trajectory_controller)$'
```

安装完成后，至少应该能查到：

```bash
diff_drive_controller
hardware_interface
joint_state_broadcaster
```

若要完整运行本章示例，建议先执行官方安装命令安装 `ros-jazzy-ros2-control` 与 `ros-jazzy-ros2-controllers`。

### ROS2 Control 架构

#### Controller Manager

`controller_manager` 是 `ros2_control` 的入口。官方文档说明，它一边管理控制器，一边通过 Resource Manager 访问硬件组件及其接口。

控制器管理器的常用参数是：

| 参数 | 类型 | 说明 |
|:---|:---|:---|
| `update_rate` | `int` | 控制器管理器实时更新循环频率，默认值为 `100` |
| `enforce_command_limits` | `bool` | 是否根据机器人描述中的关节限制约束命令，默认值为 `false` |
| `handle_exceptions` | `bool` | 是否捕获控制器和硬件组件操作中的异常，默认值为 `true` |

典型 YAML 写法如下：

```yaml
controller_manager:
  ros__parameters:
    update_rate: 100

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    diff_drive_controller:
      type: diff_drive_controller/DiffDriveController
```

#### Resource Manager

Resource Manager 负责管理硬件组件。官方文档说明，硬件组件通过 `pluginlib` 动态加载，Resource Manager 管理这些组件的生命周期、状态接口和命令接口。

在控制循环中，Resource Manager 的职责可以概括为：

*   调用硬件组件的 `read()`，把真实硬件或仿真硬件的状态读入 `ros2_control`。
*   把控制器需要的状态接口和命令接口分配给对应控制器。
*   调用硬件组件的 `write()`，把控制器输出的命令写到真实硬件或仿真硬件。

#### Hardware Components

官方文档把硬件组件分为三类：

| 类型 | 说明 |
|:---|:---|
| `system` | 复杂机器人系统，例如移动底盘、机械臂或多自由度机器人。它有读取和写入能力 |
| `sensor` | 传感器硬件，只提供读取能力 |
| `actuator` | 单自由度执行器，例如单个电机或阀门。它通常有写入能力，也可以有读取能力 |

在 URDF 中，`ros2_control` 使用 `<ros2_control>` 标签描述硬件组件。官方文档明确说明，硬件配置应写在机器人 URDF 的 `<ros2_control>` 标签中。

一个最小的差速底盘 mock 硬件写法如下：

```xml
<ros2_control name="DiffBotSystem" type="system">
  <hardware>
    <plugin>mock_components/GenericSystem</plugin>
  </hardware>

  <joint name="left_wheel_joint">
    <command_interface name="velocity"/>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>

  <joint name="right_wheel_joint">
    <command_interface name="velocity"/>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>
</ros2_control>
```

这个例子中的 `mock_components/GenericSystem` 来自插件描述文件：

```bash
/opt/ros/jazzy/share/hardware_interface/mock_components_plugin_description.xml
```

该文件声明的插件类名是：

```bash
mock_components/GenericSystem
```

因此，只要安装了 `hardware_interface`，这个 mock 硬件插件名就可以在 `/opt/ros/jazzy/share/hardware_interface` 下查到。

#### Controllers

控制器是 `ros2_control` 中真正执行控制逻辑的插件。官方文档说明，控制器派生自 `controller_interface` 包中的 `ControllerInterface`，并通过 `pluginlib` 导出。

安装 `ros-jazzy-ros2-controllers` 后，常用的两个控制器插件如下：

| 控制器 | 插件名 | 插件描述文件 |
|:---|:---|:---|
| 差速底盘控制器 | `diff_drive_controller/DiffDriveController` | `/opt/ros/jazzy/share/diff_drive_controller/diff_drive_plugin.xml` |
| 关节状态发布器 | `joint_state_broadcaster/JointStateBroadcaster` | `/opt/ros/jazzy/share/joint_state_broadcaster/joint_state_plugin.xml` |

### Joint State Broadcaster

#### Joint State Broadcaster 简述

`joint_state_broadcaster` 用于发布机器人关节状态。官方文档说明，它会读取所有 state interfaces，并发布到 `/joint_states` 和 `/dynamic_joint_states`。

需要注意，broadcaster 不是普通意义上的控制器。官方文档说明，broadcaster 不接收控制命令。

#### 发布的话题

| 话题 | 类型 | 描述 |
|:---|:---|:---|
| `/joint_states` | `sensor_msgs/msg/JointState` | 发布标准运动接口，主要是 `position`、`velocity`、`effort` |
| `/dynamic_joint_states` | `control_msgs/msg/DynamicJointState` | 发布所有可用 state interfaces，包括自定义接口 |

如果参数 `use_local_topics` 设置为 `true`，这两个话题会发布在控制器自己的命名空间下，例如 `/my_state_broadcaster/joint_states`。默认值为 `false`，因此默认发布到根命名空间。

#### 常用参数

| 参数 | 默认值 | 说明 |
|:---|:---|:---|
| `use_local_topics` | `false` | 是否在控制器局部命名空间发布话题 |
| `joints` | `{}` | 限定要发布的关节列表；为空时发布所有可用关节状态接口 |
| `interfaces` | `{}` | 限定要发布的接口列表；为空时发布所有可用状态接口 |
| `extra_joints` | `{}` | 添加额外关节，状态值为 `0` |
| `use_urdf_to_filter` | `true` | 是否根据 `robot_description` 中的关节过滤 `/joint_states` |
| `frame_id` | `base_link` | 发布 joint states 时使用的 frame_id |
| `publish_dynamic_joint_states` | `true` | 是否发布 `/dynamic_joint_states` |

最常见配置如下：

```yaml
joint_state_broadcaster:
  ros__parameters:
    use_local_topics: false
    use_urdf_to_filter: true
    publish_dynamic_joint_states: true
```

### Diff Drive Controller

#### Diff Drive Controller 简述

`diff_drive_controller` 用于差速移动机器人。官方文档说明，它接收机器人本体速度命令，并把线速度、角速度转换成左右轮命令；同时，它会根据硬件反馈计算并发布里程计。

插件描述文件：

```bash
/opt/ros/jazzy/share/diff_drive_controller/diff_drive_plugin.xml
```

其中声明的插件名为：

```bash
diff_drive_controller/DiffDriveController
```

#### 订阅的话题

| 话题 | 类型 | 描述 |
|:---|:---|:---|
| `~/cmd_vel` | `geometry_msgs/msg/TwistStamped` | 速度命令。控制器使用 `linear.x` 和 `angular.z`，其他分量会被忽略 |

这里的 `~` 表示控制器命名空间。假设控制器名为 `diff_drive_controller`，则默认话题通常是：

```bash
/diff_drive_controller/cmd_vel
```

#### 发布的话题

| 话题 | 类型 | 描述 |
|:---|:---|:---|
| `~/odom` | `nav_msgs/msg/Odometry` | 机器人位姿和速度估计 |
| `/tf` | `tf2_msgs/msg/TFMessage` | 当 `enable_odom_tf=true` 时发布 `odom` 到 `base_link` 的 TF |
| `~/cmd_vel_out` | `geometry_msgs/msg/TwistStamped` | 当 `publish_limited_velocity=true` 时发布限幅后的速度命令 |

#### 硬件接口

官方文档说明，`diff_drive_controller` 的输出接口是轮关节的 `velocity` 命令接口。

反馈接口有两种常见方式：

*   默认使用关节 `position` 状态接口。
*   当 `position_feedback=false` 时，使用关节 `velocity` 状态接口。

如果设置 `open_loop=true`，里程计不使用外部状态反馈，而是根据控制器收到的速度命令计算。

#### 常用参数

| 参数 | 默认值 | 说明 |
|:---|:---|:---|
| `left_wheel_names` | `{}` | 左轮关节名列表，不能为空 |
| `right_wheel_names` | `{}` | 右轮关节名列表，不能为空 |
| `wheel_separation` | `0.0` | 左右轮最短距离，必须大于 `0.0` |
| `wheel_radius` | `0.0` | 轮子半径，必须大于 `0.0` |
| `odom_frame_id` | `odom` | 里程计父坐标系 |
| `base_frame_id` | `base_link` | 机器人底盘坐标系 |
| `position_feedback` | `true` | 是否使用 position 反馈 |
| `open_loop` | `false` | 是否根据命令值计算里程计 |
| `enable_odom_tf` | `true` | 是否发布 `odom` 到 `base_link` 的 TF |
| `cmd_vel_timeout` | `0.5` | 速度命令超时时间，单位为秒 |
| `publish_limited_velocity` | `false` | 是否发布限幅后的速度 |
| `velocity_rolling_window_size` | `10` | 计算平均速度的滚动窗口大小 |
| `publish_rate` | `50.0` | 里程计和 TF 发布频率 |

一个适合差速小车的最小配置如下：

```yaml
diff_drive_controller:
  ros__parameters:
    left_wheel_names: ["left_wheel_joint"]
    right_wheel_names: ["right_wheel_joint"]

    wheel_separation: 0.40
    wheel_radius: 0.05

    odom_frame_id: odom
    base_frame_id: base_link

    position_feedback: true
    open_loop: false
    enable_odom_tf: true
    cmd_vel_timeout: 0.5
    publish_limited_velocity: true
    publish_rate: 50.0
```

如果使用上文的 `mock_components/GenericSystem` 最小 URDF，并且 mock 硬件没有真实编码器数据，可以先把差速控制器配置成开环测试：

```yaml
diff_drive_controller:
  ros__parameters:
    left_wheel_names: ["left_wheel_joint"]
    right_wheel_names: ["right_wheel_joint"]

    wheel_separation: 0.40
    wheel_radius: 0.05

    position_feedback: false
    open_loop: true
    enable_odom_tf: true
```

### 一个最小 Bringup 流程

#### 编写控制器配置

创建控制器配置文件，例如 `config/controllers.yaml`：

```yaml
controller_manager:
  ros__parameters:
    update_rate: 100

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    diff_drive_controller:
      type: diff_drive_controller/DiffDriveController

joint_state_broadcaster:
  ros__parameters:
    use_local_topics: false
    use_urdf_to_filter: true
    publish_dynamic_joint_states: true

diff_drive_controller:
  ros__parameters:
    left_wheel_names: ["left_wheel_joint"]
    right_wheel_names: ["right_wheel_joint"]
    wheel_separation: 0.40
    wheel_radius: 0.05
    odom_frame_id: odom
    base_frame_id: base_link
    position_feedback: false
    open_loop: true
    enable_odom_tf: true
    publish_limited_velocity: true
```

这个 YAML 中的 `type` 字段必须写插件名。上面两个插件名都可以在 `/opt/ros/jazzy/share` 下对应 XML 文件中查到。

#### 编写 URDF ros2_control 标签

在机器人 URDF 或 xacro 中加入：

```xml
<ros2_control name="DiffBotSystem" type="system">
  <hardware>
    <plugin>mock_components/GenericSystem</plugin>
  </hardware>

  <joint name="left_wheel_joint">
    <command_interface name="velocity"/>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>

  <joint name="right_wheel_joint">
    <command_interface name="velocity"/>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>
</ros2_control>
```

这里有三个对应关系必须保持一致：

*   `left_wheel_joint` 和 `right_wheel_joint` 必须是 URDF 中真实存在的关节名。
*   `diff_drive_controller.left_wheel_names` 和 `right_wheel_names` 必须与 URDF 中的轮关节名一致。
*   轮关节必须提供 `velocity` 命令接口，因为 `diff_drive_controller` 输出的是轮关节速度命令。

#### 启动控制器管理器

官方文档建议使用 `controller_manager` 包中的默认节点 `ros2_control_node`。launch 中典型写法如下：

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    control_node = Node(
        package="controller_manager",
        executable="ros2_control_node",
        parameters=[
            {"robot_description": "<这里应传入完整机器人URDF字符串>"},
            "config/controllers.yaml",
        ],
        output="both",
    )

    return LaunchDescription([control_node])
```

在实际工程中，`robot_description` 通常由 xacro 生成，不建议手写一个超长字符串。可以参考前面章节中 `robot_state_publisher` 和 xacro 的写法，把生成后的 URDF 传给 `ros2_control_node`。

#### 加载控制器

官方文档说明，`controller_manager` 提供 `spawner` 脚本，用于在启动时加载、配置并启动控制器。

常用命令如下：

```bash
ros2 run controller_manager spawner joint_state_broadcaster
ros2 run controller_manager spawner diff_drive_controller
```

如果控制器管理器不在默认命名空间，可以用 `-c` 指定：

```bash
ros2 run controller_manager spawner joint_state_broadcaster -c /controller_manager
ros2 run controller_manager spawner diff_drive_controller -c /controller_manager
```

在 launch 文件中也可以把 spawner 写成节点：

```python
joint_state_broadcaster_spawner = Node(
    package="controller_manager",
    executable="spawner",
    arguments=["joint_state_broadcaster"],
)

diff_drive_controller_spawner = Node(
    package="controller_manager",
    executable="spawner",
    arguments=["diff_drive_controller"],
)
```

#### 查看控制器和硬件接口

安装完整 `ros-jazzy-ros2-control` 后，可以使用 `ros2 control` 命令查看状态：

```bash
ros2 control list_controllers
ros2 control list_hardware_components
ros2 control list_hardware_interfaces
```

如果 `diff_drive_controller` 已激活，发送速度命令时要使用 `TwistStamped`：

```bash
ros2 topic pub /diff_drive_controller/cmd_vel geometry_msgs/msg/TwistStamped "{
  header: {stamp: {sec: 0, nanosec: 0}, frame_id: base_link},
  twist: {
    linear: {x: 0.2, y: 0.0, z: 0.0},
    angular: {x: 0.0, y: 0.0, z: 0.5}
  }
}"
```

然后可以查看里程计和关节状态：

```bash
ros2 topic echo /diff_drive_controller/odom
ros2 topic echo /joint_states
ros2 topic echo /dynamic_joint_states
```

### Gazebo 与 ROS2 Control

`ros2_control` 本身不等于仿真器。它只定义控制框架、硬件接口和控制器管理方式。要接 Gazebo，需要使用对应的仿真集成插件。

官方 Jazzy 文档在 ros2_control 仓库列表中列出 `gz_ros2_control`，它是用于 Gazebo 的插件。若要在 Gazebo 中运行 `ros2_control`，通常需要安装：

```bash
sudo apt install ros-jazzy-gz-ros2-control
```

如果没有安装 `gz_ros2_control`，则无法直接在 Gazebo 中加载 ros2_control 硬件插件。此时应先安装上面的 `ros-jazzy-gz-ros2-control` 软件包。

Gazebo 集成时，URDF 中仍然需要 `<ros2_control>` 标签，控制器 YAML 也仍然需要 `controller_manager`、`joint_state_broadcaster` 和具体控制器配置。变化的是 `<hardware><plugin>...</plugin></hardware>` 使用 Gazebo 对应的硬件插件，而不是上文的 `mock_components/GenericSystem`。

### 常见问题

#### 控制器无法激活

优先检查三件事：

*   YAML 中控制器 `type` 是否写成插件名，例如 `diff_drive_controller/DiffDriveController`。
*   URDF 中 `<ros2_control>` 的关节名是否与控制器参数中的关节名一致。
*   硬件提供的 command/state interfaces 是否满足控制器要求。

对 `diff_drive_controller` 来说，轮关节必须有 `velocity` 命令接口；如果 `position_feedback=true`，还需要 `position` 状态接口；如果 `position_feedback=false`，则需要 `velocity` 状态接口。

#### `/cmd_vel` 发了但机器人不动

Jazzy 的 `diff_drive_controller` 订阅的是 `geometry_msgs/msg/TwistStamped`，不是旧教程里常见的 `geometry_msgs/msg/Twist`。因此发布命令时需要带 `header` 和 `twist` 字段。

同时还要确认话题名。控制器订阅的是 `~/cmd_vel`，如果控制器名称是 `diff_drive_controller`，一般应发布到：

```bash
/diff_drive_controller/cmd_vel
```

#### `/joint_states` 没有数据

先确认 `joint_state_broadcaster` 已经激活：

```bash
ros2 control list_controllers
```

再确认硬件组件提供了关节 state interfaces。`joint_state_broadcaster` 默认会发布可用的 `position`、`velocity` 和 `effort` 到 `/joint_states`，并发布所有 state interfaces 到 `/dynamic_joint_states`。

#### 里程计 TF 重复

如果系统中已经有其他节点发布 `odom` 到 `base_link`，再让 `diff_drive_controller` 发布同一段 TF，可能会产生 TF 冲突。此时可以在控制器配置中关闭：

```yaml
enable_odom_tf: false
```

### 参考资料

*   ROS2 Control Jazzy 官方文档：https://control.ros.org/jazzy/index.html
*   Getting Started：https://control.ros.org/jazzy/doc/getting_started/getting_started.html
*   Controller Manager：https://control.ros.org/jazzy/doc/ros2_control/controller_manager/doc/userdoc.html
*   Joint State Broadcaster：https://control.ros.org/jazzy/doc/ros2_controllers/joint_state_broadcaster/doc/userdoc.html
*   Diff Drive Controller：https://control.ros.org/jazzy/doc/ros2_controllers/diff_drive_controller/doc/userdoc.html
*   Jazzy 默认安装路径：`/opt/ros/jazzy`
*   插件描述文件：`/opt/ros/jazzy/share/hardware_interface/mock_components_plugin_description.xml`
*   插件描述文件：`/opt/ros/jazzy/share/diff_drive_controller/diff_drive_plugin.xml`
*   插件描述文件：`/opt/ros/jazzy/share/joint_state_broadcaster/joint_state_plugin.xml`
