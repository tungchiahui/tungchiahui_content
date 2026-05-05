---
title: "Gz Sim（Gazebo Harmonic）"
---

### Gz Sim（Gazebo Harmonic 及之后的版本（ROS2 Jazzy及之后的版本））

#### Gazebo添加机器人
Gazebo中可以直接创建机器人模型，或者也可以加载ROS2中URDF格式的机器人模型，此处我们使用后者（也可以选择用自己的urdf小车，但是注意修改launch的路径）。

咱们可以用之前创建的`cpp06_urdf`里的模型。

**准备机器人模型功能包**

在工作空间中输入以下命令从而去创建功能包`mycar_description`

```bash
cd ./src
ros2 pkg create mycar_description --build-type ament_cmake
cd ..
```

在功能包下创建以下文件夹`launch，urdf，rviz，meshes`

**修改以下配置文件：**

**1. package.xml：**
在 package.xml 中需要手动添加一些执行时依赖，核心内容如下：

```xml
<exec_depend>rviz2</exec_depend>
<exec_depend>xacro</exec_depend>
<exec_depend>robot_state_publisher</exec_depend>
<exec_depend>joint_state_publisher</exec_depend>
<exec_depend>ros2launch</exec_depend>
```

**2.CMakeLists.txt**
在功能包下，新建了若干目录，需要为这些目录配置安装路径，核心内容如下：

```cmake
install(
  DIRECTORY launch urdf rviz meshes
  DESTINATION share/${PROJECT_NAME}  
)
```

把之前`cpp06_urdf`功能包中的`urdf`文件夹里的内容覆盖到`mycar_description`功能包中的`urdf`文件夹中。

ign_models中新建mycar_description目录，并将功能包mycar_description下的mesh目录复制进ign_models中的mycar_description目录。（此时meshes里是空的，很正常，你后续用到其他模型的时候可能才会有`.STL`文件）


**添加Gazebo需要的sdf特有的标签**

先修改`car_base.urdf.xacro`，添加上各种惯性，碰撞，摩擦等等：

```xml
<robot xmlns:xacro="http://wiki.ros.org/xacro">
    <!-- PI 值 -->
    <xacro:property name="PI" value="3.1416"/>
    <!-- 定义车辆参数 -->
    <!-- 车体长宽高 -->
    <xacro:property name="base_link_x" value="0.2"/>
    <xacro:property name="base_link_y" value="0.12"/>
    <xacro:property name="base_link_z" value="0.07"/>
    <!-- 离地间距 -->
    <xacro:property name="distance" value="0.015"/>
    <!-- 车轮半径 宽度 -->
    <xacro:property name="wheel_radius" value="0.025"/>
    <xacro:property name="wheel_length" value="0.02"/>

    <!-- Gazebo 新增开始：Gazebo 物理参数，用于给 link 生成 inertial -->
    <xacro:property name="base_footprint_radius" value="0.001"/>
    <xacro:property name="base_footprint_mass" value="0.001"/>
    <xacro:property name="base_link_mass" value="1.0"/>
    <xacro:property name="wheel_mass" value="0.05"/>
    <xacro:property name="wheel_mu1" value="1.0"/>
    <xacro:property name="wheel_mu2" value="0.05"/>
    <!-- Gazebo 新增结束：Gazebo 物理参数 -->

    <!-- Gazebo 新增开始：球体惯性宏，给 base_footprint 使用 -->
    <xacro:macro name="sphere_inertial" params="mass radius">
        <inertial>
            <mass value="${mass}"/>
            <inertia
                ixx="${2.0 / 5.0 * mass * radius * radius}" ixy="0.0" ixz="0.0"
                iyy="${2.0 / 5.0 * mass * radius * radius}" iyz="0.0"
                izz="${2.0 / 5.0 * mass * radius * radius}"/>
        </inertial>
    </xacro:macro>
    <!-- Gazebo 新增结束：球体惯性宏 -->

    <!-- Gazebo 新增开始：长方体惯性宏，给 base_link 使用 -->
    <xacro:macro name="box_inertial" params="mass x y z">
        <inertial>
            <mass value="${mass}"/>
            <inertia
                ixx="${mass / 12.0 * (y * y + z * z)}" ixy="0.0" ixz="0.0"
                iyy="${mass / 12.0 * (x * x + z * z)}" iyz="0.0"
                izz="${mass / 12.0 * (x * x + y * y)}"/>
        </inertial>
    </xacro:macro>
    <!-- Gazebo 新增结束：长方体惯性宏 -->

    <!-- Gazebo 新增开始：圆柱车轮惯性宏，给四个轮子使用 -->
    <xacro:macro name="wheel_inertial" params="mass radius length">
        <inertial>
            <mass value="${mass}"/>
            <inertia
                ixx="${mass / 12.0 * (3.0 * radius * radius + length * length)}" ixy="0.0" ixz="0.0"
                iyy="${mass / 2.0 * radius * radius}" iyz="0.0"
                izz="${mass / 12.0 * (3.0 * radius * radius + length * length)}"/>
        </inertial>
    </xacro:macro>
    <!-- Gazebo 新增结束：圆柱车轮惯性宏 -->

    <!-- 定义颜色 -->
    <material name="yellow">
        <color rgba="0.7 0.7 0 0.8" />
    </material>
    <material name="red">
        <color rgba="0.8 0.1 0.1 0.8" />
    </material>
    <material name="gray">
        <color rgba="0.2 0.2 0.2 0.95" />
      </material>
    <!-- 定义 base_footprint -->
    <link name="base_footprint">
        <visual>
            <geometry>
                <!-- Codex 修改：把原来的固定 0.001 改成上面新增的参数 -->
                <sphere radius="${base_footprint_radius}"/>
            </geometry>
        </visual>
        <!-- Gazebo 新增：base_footprint 的惯性，避免 Gazebo 转 SDF 时丢弃根 link -->
        <xacro:sphere_inertial mass="${base_footprint_mass}" radius="${base_footprint_radius}"/>
    </link>

    <!-- 定义 base_link -->
    <link name="base_link">
        <visual>
            <!-- 形状 -->
            <geometry>
                <box size="${base_link_x} ${base_link_y} ${base_link_z}" />
            </geometry>
            <origin xyz="0 0 0" rpy="0 0 0" />
            <material name="yellow"/>
        </visual>
        <!-- Gazebo 新增开始：base_link 碰撞体，Gazebo 物理仿真需要 collision -->
        <collision>
            <geometry>
                <box size="${base_link_x} ${base_link_y} ${base_link_z}" />
            </geometry>
            <origin xyz="0 0 0" rpy="0 0 0" />
        </collision>
        <!-- Gazebo 新增结束：base_link 碰撞体 -->
        <!-- Gazebo 新增：base_link 惯性，Gazebo 物理仿真需要 inertial -->
        <xacro:box_inertial mass="${base_link_mass}" x="${base_link_x}" y="${base_link_y}" z="${base_link_z}"/>
    </link>
    <joint name="baselink2basefootprint" type="fixed">
        <parent link="base_footprint"/>
        <child link="base_link"/>
        <origin xyz="0.0 0.0 ${distance + base_link_z / 2}"/>
    </joint>
    <!-- 车轮宏定义 -->
    <xacro:macro name="wheel_func" params="wheel_name is_front is_left" >
        <link name="${wheel_name}_wheel">
            <visual>
                <geometry>
                    <cylinder radius="${wheel_radius}" length="${wheel_length}" />
                </geometry>
                <origin xyz="0 0 0" rpy="${PI / 2} 0 0" />
                <material name="gray"/>
            </visual>
            <!-- Gazebo 新增开始：车轮碰撞体，四个轮子都会通过这个宏生成 -->
            <collision>
                <geometry>
                    <cylinder radius="${wheel_radius}" length="${wheel_length}" />
                </geometry>
                <origin xyz="0 0 0" rpy="${PI / 2} 0 0" />
            </collision>
            <!-- Gazebo 新增结束：车轮碰撞体 -->
            <!-- Gazebo 新增：车轮惯性，四个轮子都会通过这个宏生成 -->
            <xacro:wheel_inertial mass="${wheel_mass}" radius="${wheel_radius}" length="${wheel_length}"/>
        </link>
        <joint name="${wheel_name}2baselink" type="continuous">
            <parent link="base_link"  />
            <child link="${wheel_name}_wheel" />
            <origin xyz="${(base_link_x / 2 - wheel_radius) * is_front} ${base_link_y / 2 * is_left} ${(base_link_z / 2 + distance - wheel_radius) * -1}" rpy="0 0 0" />
            <axis xyz="0 1 0" />
        </joint>
        <!-- Gazebo 新增：四轮差速原地转弯需要轮胎横向滑移；降低 mu2 可以减少 Gazebo 中的卡顿和抖动 -->
        <gazebo reference="${wheel_name}_wheel">
            <mu1>${wheel_mu1}</mu1>
            <mu2>${wheel_mu2}</mu2>
            <fdir1>1 0 0</fdir1>
        </gazebo>
    </xacro:macro>
    <!-- 车轮宏调用 -->
    <xacro:wheel_func wheel_name="left_front" is_front="1" is_left="1" />
    <xacro:wheel_func wheel_name="left_back" is_front="-1" is_left="1" />
    <xacro:wheel_func wheel_name="right_front" is_front="1" is_left="-1" />
    <xacro:wheel_func wheel_name="right_back" is_front="-1" is_left="-1" />
</robot>
```

修改`car.urdf.xacro`：

```xml
<robot name="car" xmlns:xacro="http://wiki.ros.org/xacro">
    <xacro:include filename="car_base.urdf.xacro"/>
    <xacro:include filename="car_camera.urdf.xacro"/>
    <xacro:include filename="car_laser.urdf.xacro"/>
</robot>
```


**机器人模型功能包下新建launch文件**

新建launch文件mycar_desc_sim.launch.py，并输入如下内容：

```python
from launch import LaunchDescription
from launch_ros.actions import Node
import os
from ament_index_python.packages import get_package_share_directory
from launch_ros.parameter_descriptions import ParameterValue
from launch.substitutions import Command,LaunchConfiguration
from launch.actions import DeclareLaunchArgument

def generate_launch_description():

    mycar_description = get_package_share_directory("mycar_description")
    default_model_path = os.path.join(mycar_description,"urdf/xacro","car.urdf.xacro")
    model = DeclareLaunchArgument(name="model", default_value=default_model_path)

    # 加载机器人模型
    # 启动 robot_state_publisher 节点并以参数方式加载 urdf 文件
    robot_description = ParameterValue(Command(["xacro ",LaunchConfiguration("model")]))
    robot_state_publisher = Node(
        package="robot_state_publisher",
        executable="robot_state_publisher",
        parameters=[{"robot_description": robot_description}]
    )

    return LaunchDescription([
        model,
        robot_state_publisher,
    ])
```

较之于以往该文件缺少了joint_state_publisher节点，该节点作用是发布活动关节状态，这一功能后续由ignition实现。

**添加机器人模型**

创建gazebo_sim_robot_world.launch.py文件，包含机器人模型的发布文件并在Gazebo中生成机器人模型，修改后的代码如下：

```python
import os

from ament_index_python.packages import get_package_share_directory

from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch_ros.actions import Node

def generate_launch_description():

    this_pkg = get_package_share_directory("demo_gazebo_sim")
    mycar_desc_pkg = get_package_share_directory("mycar_description")
    pkg_ros_gz_sim = get_package_share_directory("ros_gz_sim")
    world_file = os.path.join(this_pkg,"world","house.sdf")

    gz_sim = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            os.path.join(pkg_ros_gz_sim, "launch", "gz_sim.launch.py")),
        launch_arguments={
            "gz_args": "-r " + world_file
        }.items(),
    )
    mycar_desc = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            os.path.join(mycar_desc_pkg,"launch","mycar_desc_sim.launch.py")
        )
    )
    spawn = Node(package="ros_gz_sim", executable="create",
                arguments=[
                "-name", "mycar",
                "-x", "0",
                "-z", "0.01", #设置为0,可能会陷进地里
                "-y", "0",
                "-topic", "/robot_description"],
            output="screen")

    return LaunchDescription([
        gz_sim,
        spawn,
        mycar_desc,
    ])
```

**构建**

终端中进入当前工作空间，编译功能包：

```bash
colcon build --packages-select mycar_description demo_gazebo_sim
```

**执行**

终端中进入当前工作空间，调用如下指令执行launch文件：（执行起来有问题的话，你只要学过urdf怎么跑，应该拥有自我寻找错误的能力了，自己找吧）

```bash
. install/setup.bash

ros2 launch demo_gazebo_sim gazebo_sim_world.launch.py
```

运行结果如下图所示。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1733.webp)

