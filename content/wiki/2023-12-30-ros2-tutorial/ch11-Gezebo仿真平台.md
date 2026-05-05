---
title: "Gezebo仿真平台"
---

### 简述
Gazebo目前主要分为两个版本，

一个是旧版Gazebo Classic，一个是新版Ignition Gazebo（ Gazebo Fortress）。

本文中称呼的Gazebo均代指Gazebo Classic，而Ignition Gazebo会加以区别。

然后现在ROS2Humble自带的是Ignition Gazebo（ Gazebo Fortress），但是ROS2 Jazzy自带的是	Gazebo Harmonic，建议学习	Gazebo Harmonic ！

**场景**

在ROS机器人开发中，实体机器人虽然具有直接性和真实性的优势，但也存在一些不足，比如：

> 1.  高昂的成本：研发一款自主移动机器人需要购买昂贵的硬件组件，如传感器、电机、控制器等，且这些硬件在研发初期可能需要频繁更换或升级，导致成本急剧上升。
>     
> 2.  资源限制：由于资金和资源有限，可能无法同时拥有多台实体机器人进行测试。这会导致测试周期延长，研发进度受阻。
>     
> 3.  环境的不确定性和复杂性：在真实环境中测试机器人时，可能会遇到各种不可预测的情况，如光线变化、地面不平整、电磁干扰等，这些因素都可能影响机器人的性能。
>     
> 4.  安全风险：在测试过程中，如果机器人的控制算法或硬件出现故障，可能会导致机器人失控，对人员或环境造成损害。
>     

在ROS机器人开发的领域里，仿真技术被广泛应用以弥补实体机器人测试中的不足。

**概念**

**机器人仿真** 是一种利用计算机模型和仿真技术来模拟机器人在虚拟环境中的行为和性能的过程。它通过创建虚拟的机器人和环境模型，模拟机器人的感知、控制和运动能力，以及与环境和其他对象的交互。

**作用**

通过仿真测试，可以降低机器人研发成本和风险，提高机器人系统的性能和可靠性，并为实际机器人部署提供参考和指导。

**仿真优势:**

仿真在机器人系统研发过程中占有举足轻重的地位，在研发与测试中较之于实体机器人实现，仿真有如下几点的显著优势:

1.  **低成本:** 当前机器人成本居高不下，动辄几十万，仿真可以大大降低成本，减小风险

2.  **高效:** 搭建的环境更为多样且灵活，可以提高测试效率以及测试覆盖率

3.  **高安全性:** 仿真环境下，无需考虑耗损问题

仿真技术为开发者构建了一个既高效又安全，且成本低廉的全方位测试和验证平台。

**仿真缺陷:**

机器人在仿真环境与实际环境下的表现差异较大，换言之，仿真并不能完全做到模拟真实的物理世界，存在一些"失真"的情况，原因:

1.  仿真器所使用的物理引擎目前还不能够完全精确模拟真实世界的物理情况

2.  仿真器构建的是关节驱动器（电机&齿轮箱）、传感器与信号通信的绝对理想情况，目前不支持模拟实际硬件缺陷或者一些临界状态等情形

总之，仿真技术虽然重要，但并不能完全替代实体测试。实体测试可以验证仿真结果的准确性，并发现仿真中可能忽略的问题。

### Gazebo([Gazebo Classic](https://classic.gazebosim.org/))
这个是老版Gazebo，仅在ROS1和ROS2 Humble上可用。（但是ROS2并不推荐使用老版Gazebo，更加建议使用新版Gazebo）

ROS2老版Gazebo仅在Humble版本上可用，在Jazzy及以后的版本已移除。

**（不想学老版Gazebo的，直接跳到下一节的Ignition Gazebo即可）**

这俩对咱们的区别不算太大，也就是一个教程多，一个教程少的区别，你直接学老版Gazebo也是一样用的。

**（初学者也可以只学教程多的Gazebo Classic,可以少走一些弯路）**

Gezebo官网：

https://gazebosim.org/home

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1627.webp)

如果你想从老版的Gazebo迁移到新版的Gazebo，那么请看下方官方教程：

从Gazebo Classic迁移到ROS2 Humble的Ign Gazebo(Gazebo Fortress)：

https://gazebosim.org/docs/fortress/gazebo\_classic\_migration/

从Gazebo Classic迁移到ROS2 Jazzy的Gazebo Sim(Gazebo Harmonic)：

Jazzy之后的版本应该变化不会太大，也可以暂时参照下面这个教程（或者去官网找对应版本的教程）：

https://gazebosim.org/docs/harmonic/gazebo\_classic\_migration/

#### Gazebo Classic安装与运行
ROS2只有humble有老版Gazebo

```bash
sudo apt install ros-humble-gazebo-ros ros-humble-gazebo-ros-pkgs
```

安装完成后，我们就可以通过下面的命令行来启动gazebo并加载ros2插件。

```bash
gazebo --verbose -s libgazebo_ros_init.so -s libgazebo_ros_factory.so 
```

看到下面这个日志和gazebo界面，没啥大问题就说明成功了。（一些警告是说Gazebo Classic已经弃用，鼓励使用新版Ingition Gazebo，忽略这些警告即可）

```sql
root@Dell-G15-5511:/home/tungchiahui/UserFolder/MySource/ROS_WS/ROS2_WS/6.ws_simulations$ gazebo --verbose -s libgazebo_ros_init.so -s libgazebo_ros_factory.so 
Gazebo multi-robot simulator, version 11.10.2
Copyright (C) 2012 Open Source Robotics Foundation.
Released under the Apache 2 License.
http://gazebosim.org

[Msg] Waiting for master.
Gazebo multi-robot simulator, version 11.10.2
Copyright (C) 2012 Open Source Robotics Foundation.
Released under the Apache 2 License.
http://gazebosim.org

[Wrn] [gazebo_ros_init.cpp:178] 

#     # ####### ####### ###  #####  #######

##    # #     #    #     #  #     # #

# #   # #     #    #     #  #       #

#  #  # #     #    #     #  #       #####

#   # # #     #    #     #  #       #

#    ## #     #    #     #  #     # #

#     # #######    #    ###  #####  #######

This version of Gazebo, now called Gazebo classic, reaches end-of-life
in January 2025. Users are highly encouraged to migrate to the new Gazebo
using our migration guides (https://gazebosim.org/docs/latest/gazebo_classic_migration?utm_source=gazebo_ros_pkgs&utm_medium=cli)

[Msg] Waiting for master.
[Msg] Connected to gazebo master @ http://127.0.0.1:11345
[Msg] Publicized address: 192.168.31.60
[Msg] Loading world file [/usr/share/gazebo-11/worlds/empty.world]
XDG_RUNTIME_DIR (/run/user/1000) is not owned by us (uid 0), but by uid 1000! (This could e.g. happen if you try to connect to a non-root PulseAudio as a root user, over the native protocol. Don't do that.)
ALSA lib pcm_dmix.c:1032:(snd_pcm_dmix_open) unable to open slave
AL lib: (EE) ALCplaybackAlsa_open: Could not open playback device 'default': No such file or directory
[Err] [OpenAL.cc:84] Unable to open audio device[default]
 Audio will be disabled.
[Msg] Connected to gazebo master @ http://127.0.0.1:11345
[Msg] Publicized address: 192.168.31.60
[Wrn] [GuiIface.cc:298] Couldn't locate specified .ini. Creating file at "/root/.gazebo/gui.ini"
[Wrn] [GuiIface.cc:120] QStandardPaths: runtime directory '/run/user/1000' is not owned by UID 0, but a directory permissions 0700 owned by UID 1000 GID 1000
[Wrn] [Event.cc:61] Warning: Deleting a connection right after creation. Make sure to save the ConnectionPtr from a Connect call

libcurl: (35) error:0A000126:SSL routines::unexpected eof while reading
[Wrn] [ModelDatabase.cc:212] Unable to connect to model database using [http://models.gazebosim.org//database.config]. Only locally installed models will be available.
[Wrn] [Event.cc:61] Warning: Deleting a connection right after creation. Make sure to save the ConnectionPtr from a Connect call
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1628.webp)

#### 插件及节点服务介绍
使用之前的命令启动Gazebo并加载gazebo\_ros插件，我们使用下面的指令来看插件的节点，以及改节点为我们提供的服务有哪些？

节点列表

```bash
ros2 node list
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1629.webp)

然后我们看看这个节点对外提供的服务有哪些？

```bash
ros2 service list
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1630.webp)

除去和参数相关的几个服务，我们可以看到另外三个特殊服务：

*   /spawn\_entity，用于加载模型到gazebo中

*   /get\_model\_list，用于获取模型列表

*   /delete\_entity，用于删除gazbeo中已经加载的模型

我们想要让gazebo显示出我们配置好的机器人模型使用/spawn\_entity来加载即可。

接着我们可以来请求服务来加载模型，先带你看一下服务的接口类型。

```bash
ros2 service type /spawn_entity
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1631.webp)

```bash
ros2 interface show gazebo_msgs/srv/SpawnEntity
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1632.webp)

可以看到服务的请求内容包括：

*   string name ，需要加载的实体的名称 (可选的)。

*   string xml ，实体的XML描述字符串, URDF或者SDF。

*   string robot\_namespace ，产生的机器人和所有的ROS接口的命名空间，多机器人仿真的时候很有用。

*   geometry\_msgs/Pose initial\_pose ，机器人的初始化位置

*   string reference\_frame ，初始姿态是相对于该实体的frame定义的。如果保持"empty"或"world"或“map”，则使用 gazebo的world作为frame。如果指定了不存在的实体，则会返回错误

#### 调用服务加载模型
我们这里教程使用鱼香ROS的fishbot模型：https://github.com/fishros/fishbot/blob/navgation2/src/fishbot\_description/urdf/fishbot\_gazebo.urdf

```xml
<?xml version="1.0"?>
<robot name="fishbot">

  <link name="base_footprint"/>

  <joint name="base_joint" type="fixed">
    <parent link="base_footprint"/>
    <child link="base_link"/>
    <origin xyz="0.0 0.0 0.076" rpy="0 0 0"/>
  </joint>

  <link name="base_link">
          <visual>
      <origin xyz="0 0 0.0" rpy="0 0 0"/>
      <geometry>
                <cylinder length="0.12" radius="0.10"/>
      </geometry>
      <material name="blue">
              <color rgba="0.1 0.1 1.0 0.5" /> 
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 0.0" rpy="0 0 0"/>
      <geometry>
                <cylinder length="0.12" radius="0.10"/>
      </geometry>
      <material name="blue">
              <color rgba="0.1 0.1 1.0 0.5" /> 
      </material>
    </collision>
    <inertial>
      <mass value="0.2"/>
      <inertia ixx="0.0122666" ixy="0" ixz="0" iyy="0.0122666" iyz="0" izz="0.02"/>
    </inertial>
  </link>

  <link name="laser_link">
    <visual>
      <origin xyz="0 0 0" rpy="0 0 0"/>
      <geometry>
        <cylinder length="0.02" radius="0.02"/>
      </geometry>
      <material name="black">
        <color rgba="0.0 0.0 0.0 0.5" /> 
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 0" rpy="0 0 0"/>
      <geometry>
        <cylinder length="0.02" radius="0.02"/>
      </geometry>
      <material name="black">
        <color rgba="0.0 0.0 0.0 0.5" /> 
      </material>
    </collision>
    <inertial>
    <mass value="0.1"/>
      <inertia ixx="0.000190416666667" ixy="0" ixz="0" iyy="0.0001904" iyz="0" izz="0.00036"/>
    </inertial>
  </link>

  <joint name="laser_joint" type="fixed">
      <parent link="base_link" />
      <child link="laser_link" />
      <origin xyz="0 0 0.075" />
  </joint>

  <link name="imu_link">
          <visual>
      <origin xyz="0 0 0.0" rpy="0 0 0"/>
      <geometry>
                    <box size="0.02 0.02 0.02"/>
      </geometry>
    </visual>
    <collision>
      <origin xyz="0 0 0.0" rpy="0 0 0"/>
      <geometry>
                    <box size="0.02 0.02 0.02"/>
      </geometry>
    </collision>
    <inertial>
      <mass value="0.1"/>
        <inertia ixx="0.000190416666667" ixy="0" ixz="0" iyy="0.0001904" iyz="0" izz="0.00036"/>
      </inertial>
  </link>

  <joint name="imu_joint" type="fixed">
      <parent link="base_link" />
      <child link="imu_link" />
      <origin xyz="0 0 0.02" />
  </joint>

  <link name="left_wheel_link">
      <visual>
        <origin xyz="0 0 0" rpy="1.57079 0 0"/>
        <geometry>
          <cylinder length="0.04" radius="0.032"/>
        </geometry>
          <material name="black">
            <color rgba="0.0 0.0 0.0 0.5" /> 
          </material>
      </visual>
      <collision>
        <origin xyz="0 0 0" rpy="1.57079 0 0"/>
        <geometry>
          <cylinder length="0.04" radius="0.032"/>
        </geometry>
          <material name="black">
            <color rgba="0.0 0.0 0.0 0.5" /> 
          </material>
      </collision>
      <inertial>
        <mass value="0.2"/>
          <inertia ixx="0.000190416666667" ixy="0" ixz="0" iyy="0.0001904" iyz="0" izz="0.00036"/>
        </inertial>
  </link>

  <link name="right_wheel_link">
      <visual>
        <origin xyz="0 0 0" rpy="1.57079 0 0"/>
        <geometry>
          <cylinder length="0.04" radius="0.032"/>
        </geometry>
          <material name="black">
            <color rgba="0.0 0.0 0.0 0.5" /> 
          </material>
      </visual>
      <collision>
        <origin xyz="0 0 0" rpy="1.57079 0 0"/>
        <geometry>
          <cylinder length="0.04" radius="0.032"/>
        </geometry>
          <material name="black">
            <color rgba="0.0 0.0 0.0 0.5" /> 
          </material>
      </collision>
      <inertial>
      <mass value="0.2"/>
      <inertia ixx="0.000190416666667" ixy="0" ixz="0" iyy="0.0001904" iyz="0" izz="0.00036"/>
    </inertial>
  </link>

  <joint name="left_wheel_joint" type="continuous">
      <parent link="base_link" />
      <child link="left_wheel_link" />
      <origin xyz="-0.02 0.10 -0.06" />
      <axis xyz="0 1 0" />
  </joint>

  <joint name="right_wheel_joint" type="continuous">
      <parent link="base_link" />
      <child link="right_wheel_link" />
      <origin xyz="-0.02 -0.10 -0.06" />
      <axis xyz="0 1 0" />
  </joint>

  <link name="caster_link">
      <visual>
        <origin xyz="0 0 0" rpy="1.57079 0 0"/>
        <geometry>
            <sphere radius="0.016"/>
        </geometry>
          <material name="black">
            <color rgba="0.0 0.0 0.0 0.5" /> 
          </material>
      </visual>
      <collision>
        <origin xyz="0 0 0" rpy="1.57079 0 0"/>
        <geometry>
            <sphere radius="0.016"/>
        </geometry>
          <material name="black">
            <color rgba="0.0 0.0 0.0 0.5" /> 
          </material>
      </collision>
      <inertial>
      <mass value="0.02"/>
      <inertia ixx="0.000190416666667" ixy="0" ixz="0" iyy="0.0001904" iyz="0" izz="0.00036"/>
    </inertial>
  </link>

  <joint name="caster_joint" type="fixed">
      <parent link="base_link" />
      <child link="caster_link" />
      <origin xyz="0.06 0.0 -0.076" />
      <axis xyz="0 1 0" />
  </joint>

  <gazebo reference="caster_link">
    <material>Gazebo/Black</material>
  </gazebo>

  <gazebo reference="caster_link">
    <mu1 value="0.0"/>
    <mu2 value="0.0"/>
    <kp value="1000000.0" />
    <kd value="10.0" />

  </gazebo>

  <gazebo>
    <plugin name='diff_drive' filename='libgazebo_ros_diff_drive.so'>
          <ros>
            <namespace>/</namespace>
            <remapping>cmd_vel:=cmd_vel</remapping>
            <remapping>odom:=odom</remapping>
          </ros>
          <update_rate>30</update_rate>

          <left_joint>left_wheel_joint</left_joint>
          <right_joint>right_wheel_joint</right_joint>

          <wheel_separation>0.2</wheel_separation>
          <wheel_diameter>0.065</wheel_diameter>

          <max_wheel_torque>20</max_wheel_torque>
          <max_wheel_acceleration>1.0</max_wheel_acceleration>

          <publish_odom>true</publish_odom>
          <publish_odom_tf>true</publish_odom_tf>
          <publish_wheel_tf>false</publish_wheel_tf>
          <odometry_frame>odom</odometry_frame>
          <robot_base_frame>base_footprint</robot_base_frame>
      </plugin>

      <plugin name="fishbot_joint_state" filename="libgazebo_ros_joint_state_publisher.so">
        <ros>
          <remapping>~/out:=joint_states</remapping>
        </ros>
        <update_rate>30</update_rate>
        <joint_name>right_wheel_joint</joint_name>
        <joint_name>left_wheel_joint</joint_name>
      </plugin>    
      </gazebo> 

      <gazebo reference="laser_link">
        <material>Gazebo/Black</material>
      </gazebo>

    <gazebo reference="imu_link">
      <sensor name="imu_sensor" type="imu">
      <plugin filename="libgazebo_ros_imu_sensor.so" name="imu_plugin">
          <ros>
            <namespace>/</namespace>
            <remapping>~/out:=imu</remapping>
          </ros>
          <initial_orientation_as_reference>false</initial_orientation_as_reference>
        </plugin>
        <always_on>true</always_on>
        <update_rate>100</update_rate>
        <visualize>true</visualize>
        <imu>
          <angular_velocity>
            <x>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>2e-4</stddev>
                <bias_mean>0.0000075</bias_mean>
                <bias_stddev>0.0000008</bias_stddev>
              </noise>
            </x>
            <y>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>2e-4</stddev>
                <bias_mean>0.0000075</bias_mean>
                <bias_stddev>0.0000008</bias_stddev>
              </noise>
            </y>
            <z>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>2e-4</stddev>
                <bias_mean>0.0000075</bias_mean>
                <bias_stddev>0.0000008</bias_stddev>
              </noise>
            </z>
          </angular_velocity>
          <linear_acceleration>
            <x>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>1.7e-2</stddev>
                <bias_mean>0.1</bias_mean>
                <bias_stddev>0.001</bias_stddev>
              </noise>
            </x>
            <y>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>1.7e-2</stddev>
                <bias_mean>0.1</bias_mean>
                <bias_stddev>0.001</bias_stddev>
              </noise>
            </y>
            <z>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>1.7e-2</stddev>
                <bias_mean>0.1</bias_mean>
                <bias_stddev>0.001</bias_stddev>
              </noise>
            </z>
          </linear_acceleration>
        </imu>
      </sensor>
    </gazebo>

    <gazebo reference="laser_link">
      <sensor name="laser_sensor" type="ray">
      <always_on>true</always_on>
      <visualize>true</visualize>
      <update_rate>5</update_rate>
      <pose>0 0 0.075 0 0 0</pose>
      <ray>
          <scan>
            <horizontal>
              <samples>360</samples>
              <resolution>1.000000</resolution>
              <min_angle>0.000000</min_angle>
              <max_angle>6.280000</max_angle>
            </horizontal>
          </scan>
          <range>
            <min>0.120000</min>
            <max>3.5</max>
            <resolution>0.015000</resolution>
          </range>
          <noise>
            <type>gaussian</type>
            <mean>0.0</mean>
            <stddev>0.01</stddev>
          </noise>
      </ray>

      <plugin name="laserscan" filename="libgazebo_ros_ray_sensor.so">
        <ros>

          <remapping>~/out:=scan</remapping>
        </ros>
        <output_type>sensor_msgs/LaserScan</output_type>
        <frame_name>laser_link</frame_name>
      </plugin>
      </sensor>
    </gazebo>

</robot>
```

看到这里你是不是迫不及待敲起来命令行来加载我们的机器人到gazebo了，别着急，小鱼再推荐一个可视化服务请求工具，其实在第六章中小鱼介绍过，在rqt工具集里有一个叫服务请求工具。

命令行输入rqt，在插件选项中选择Services->Service Caller,然后再下拉框选择/spawn\_entity服务，即可看到下面的界面。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1633.webp)

接着我们把我们的FishBot的URDF模型复制粘贴，放到xml中（注意要把原来的''删掉哦！）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1634.webp)

然后点右上角的call，可以显示下图成功了。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1635.webp)

接着就可以看到工厂返回说成功把机器人制作出来送入gazebo了。

此时再看我们的Gazebo,一个小小的，白白的机器人出现了。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1636.webp)

shift+鼠标左键，或者直接点滚轮中键，都可以拖动视角。很多玩过第三人称游戏的学弟学妹肯定都不陌生这种操控吧。

#### 在不同位置加载多个机器人
可以再生产一个fishbot（为了后面需要多机器人仿真的小伙伴）。

修改rqt中的参数，增加一个命名空间，然后修改一个位置，让第二个机器人和第一个相距1m的地方生产，然后点击Call。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1637.webp)

返回成功，此时拖送Gazebo观察一下，发现多出了一个机器人，距离刚好是在X轴（红色）1米（一个小格子一米）处。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1638.webp)

#### 查询和删除机器人
利用rqt工具，我们再对另外两个服务接口进行请求。

首先先查询有几个模型在仿真环境中

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1639.webp)

查到了三个模型，一个大地，一个fishbot，一个fishbot\_0。

我们接着尝试把fishbot\_0删掉，选择删除实体，输入fishbot\_0的名字，拿起小电话通知工厂回收我们的0号fishbot。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1640.webp)

调用成功，观察gazebo发现机器人，人没了

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1641.webp)

#### 创建工作空间
本节代码参考鱼香ROS：https://github.com/fishros/fishbot/tree/navgation2

```bash

# 创建工作空间
mkdir -p ws_simulations/src
cd ws_simulations/src
```

创建功能包

```bash
ros2 pkg create fishbot_description --build-type ament_cmake
cd fishbot_description
```

然后配置package.xml

```xml
  <exec_depend>rviz2</exec_depend>
  <exec_depend>xacro</exec_depend>
  <exec_depend>robot_state_publisher</exec_depend>
  <exec_depend>joint_state_publisher</exec_depend>
  <exec_depend>ros2launch</exec_depend>
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1642.webp)

然后修改cmakelists.txt

```bash
install(
  DIRECTORY launch urdf rviz meshes
  DESTINATION share/${PROJECT_NAME}  
)
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1643.webp)

克隆下鱼香ROS的仓库，并复制下里面的文件到咱们的目录下。这都是上一节的东西，与本节学习无关，直接复制就行。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1644.webp)

打开工作空间，在`src/fishbot_description/launch`中添加一个`gazebo.launch.py`文件，我们开始编写launch文件来在gazebo中加载机器人模型。

我们主要需要做两件事：

1.  启动gazebo，我们可以将命令行写成一个launch节点

```python
ExecuteProcess(
        cmd=['gazebo', '--verbose','-s', 'libgazebo_ros_init.so', '-s', 'libgazebo_ros_factory.so', gazebo_world_path],
        output='screen')

```

2.  上面我们加载机器人是直接将XML格式的URDF复制过去进行加载的，这样很不方便，我们可以使用gazebo\_ros为我们提供好的一个叫做`spawn_entity.py`节点，该节点支持从文件地址直接生产机器人到Gazebo。

该节点需要两个参数，一个机器人的模型名字和urdf的文件地址，这个简单，前面我们曾经使用package\_share来拼接过urdf路径。

```python
spawn_entity_cmd = Node(
    package='gazebo_ros', 
    executable='spawn_entity.py',
    arguments=['-entity', robot_name_in_model,  '-file', urdf_model_path ], output='screen')
```

先加载赵虚左老师的模板

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1645.webp)

我们首先需要ExecuteProcess来输入终端命令，所以要先把from launch.actions import ExecuteProcess的注释先打开。

为了方便修改加载的机器人模型和urdf,我们还需要使用share目录，所以也要把from ament\_index\_python.packages import get\_package\_share\_directory的注释打开，并import os。

写完后的launch文件如下：

```python
from launch import LaunchDescription
from launch_ros.actions import Node

# 封装终端指令相关类
from launch.actions import ExecuteProcess

# from launch.substitutions import FindExecutable

# 参数声明与获取

# from launch.actions import DeclareLaunchArgument

# from launch.substitutions import LaunchConfiguration

# 文件包含相关

# from launch.actions import IncludeLaunchDescription

# from launch.launch_description_sources import PythonLaunchDescriptionSource

# 分组相关

# from launch_ros.actions import PushRosNamespace

# from launch.actions import GroupAction

# 事件相关

# from launch.event_handlers import OnProcessStart,OnProcessExit

# from launch.actions import ExecuteProcess,RegisterEventHandler,LogInfo

# 获取功能包下share目录或路径
from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    robot_name_in_model = 'fishbot'
    package_name = 'fishbot_description'
    urdf_name = "fishbot_gazebo.urdf"

    pkg_share = get_package_share_directory(f"{package_name}")
    urdf_model_path = os.path.join(pkg_share, f'urdf/urdf/{urdf_name}')

    # Start Gazebo server
    start_gazebo_cmd = ExecuteProcess(
        cmd=['gazebo', '--verbose','-s', 'libgazebo_ros_init.so', '-s', 'libgazebo_ros_factory.so'],
        output='screen')

    # Launch the robot
    spawn_entity_cmd = Node(
        package='gazebo_ros', 
        executable='spawn_entity.py',
        arguments=['-entity', robot_name_in_model,  '-file', urdf_model_path ], output='screen')

    return LaunchDescription([start_gazebo_cmd,spawn_entity_cmd])
```

编译运行

```python
colcon build --packages-select fishbot_description
source install/setup.bash
ros2 launch fishbot_description gazebo.launch.py
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1646.webp)

#### 插件
使用下面的指令可以查看所有的动态链接库：

```python
ls /opt/ros/humble/lib/libgazebo_ros*
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1647.webp)

```bash
/opt/ros/humble/lib/libgazebo_ros2_control.so
/opt/ros/humble/lib/libgazebo_ros_ackermann_drive.so
/opt/ros/humble/lib/libgazebo_ros_bumper.so
/opt/ros/humble/lib/libgazebo_ros_camera.so
/opt/ros/humble/lib/libgazebo_ros_diff_drive.so
/opt/ros/humble/lib/libgazebo_ros_elevator.so
/opt/ros/humble/lib/libgazebo_ros_factory.so
/opt/ros/humble/lib/libgazebo_ros_force.so
/opt/ros/humble/lib/libgazebo_ros_force_system.so
/opt/ros/humble/lib/libgazebo_ros_ft_sensor.so
/opt/ros/humble/lib/libgazebo_ros_gps_sensor.so
/opt/ros/humble/lib/libgazebo_ros_hand_of_god.so
/opt/ros/humble/lib/libgazebo_ros_harness.so
/opt/ros/humble/lib/libgazebo_ros_imu_sensor.so
/opt/ros/humble/lib/libgazebo_ros_init.so
/opt/ros/humble/lib/libgazebo_ros_joint_pose_trajectory.so
/opt/ros/humble/lib/libgazebo_ros_joint_state_publisher.so
/opt/ros/humble/lib/libgazebo_ros_node.so
/opt/ros/humble/lib/libgazebo_ros_p3d.so
/opt/ros/humble/lib/libgazebo_ros_planar_move.so
/opt/ros/humble/lib/libgazebo_ros_projector.so
/opt/ros/humble/lib/libgazebo_ros_properties.so
/opt/ros/humble/lib/libgazebo_ros_ray_sensor.so
/opt/ros/humble/lib/libgazebo_ros_state.so
/opt/ros/humble/lib/libgazebo_ros_template.so
/opt/ros/humble/lib/libgazebo_ros_tricycle_drive.so
/opt/ros/humble/lib/libgazebo_ros_utils.so
/opt/ros/humble/lib/libgazebo_ros_vacuum_gripper.so
/opt/ros/humble/lib/libgazebo_ros_video.so
/opt/ros/humble/lib/libgazebo_ros_wheel_slip.so
```

##### 运动控制插件+里程计odom
本节课通过配置两轮差速控制插件，让我们的机器人动起来

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1648.webp)

插件介绍：

Gazebo是一个独立于ROS的软件，对外提供了丰富的API可以使用，gazebo的插件按照用途大致可以分为两种：

1.  用于控制的插件，通过插件可以控制机器人关节运动，可以进行位置、速度、力的控制，比如我们这节课的两轮差速控制器。

2.  用于数据采集的插件，比如IMU传感器用于采集机器人的惯性，激光雷达用于采集机器人周围的点云信息。

当然上面两类插件功能也可以写到一个插件里，两轮差速插件（gazebo\_ros\_diff\_drive）就是一个二合一加强版。（差速控制+odom）

两轮差速插件用于控制机器人轮子关节的位置变化，同时该插件还会获取轮子的位置以及速度的信息的反馈，根据反馈的位置信息结合运动学模型即可计算出当前机器人的位姿（里程计）。

两轮差速控制器可以将轮子的目标转速发送给Gazebo，并从Gazebo获取到实际的速度和位置。（注意：发送给Gazebo是目标速度，反馈回来的是实际速度。目标!=实际，比如轮子卡住了，无论你发什么目标速度，实际速度都是0。）

要想快速了解一个系统的功能，最直接的就是看系统的对外的输入和输出是什么？什么都不要说，看下图：

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1649.webp)

上图就是对gazebo\_ros\_diff\_drive的输入和输出信息的总结，可以很直观的看到该插件主要输入控制指令，主要输出里程计信息。接着小鱼带你分别认识一下输入和输出两个部分。

这个插件需要配置一系列参数如下图：

不知道你是否还记得在第七章中，小鱼对两轮差速底盘的运动学正的介绍。如果要完成底盘的正逆解和里程计的推算就必须要知道轮子的直径和间距。

同时该插件还提供了一些可以控制输出的选项，因为是仿真，所以还要告诉插件轮子对应的joint名称等信息，这样就有了下面这个参数表格：

| 配置项 | 含义 |
|:---|:---|
| ros | ros相关配置，包含命名空间和话题重映射等 |
| update_rate | 数据更新速率 |
| left_joint | 左轮关节名称 |
| right_joint | 右轮关节名称 |
| wheel_separation | 左右轮子的间距 |
| wheel_diameter | 轮子的直径 |
| max_wheel_torque | 轮子最大的力矩 |
| max_wheel_acceleration | 轮子最大的加速度 |
| publish_odom | 是否发布里程计 |
| publish_odom_tf | 是否发布里程计的tf开关 |
| publish_wheel_tf | 是否发布轮子的tf数据开关 |
| odometry_frame | 里程计的framed ID，最终体现在话题和TF上 |
| robot_base_frame | 机器人的基础frame的ID |

控制指令：两轮差速控制器默认通过订阅话题`cmd_vel`来获取目标线速度和角速度。该话题的类型为：`geometry_msgs/msg/Twist`

该接口里主要是一些线速度和角速度都包含在x、y、z，代表坐标系的三个方向上的对应速度。 **（详细的接口内容请看硬件平台章节）**

两轮差速控制器收到这个话题数据后将其中的角速度和线速度转换上两个轮子的转动速度发送给Gazebo。

输出的信息：

里程计信息默认的输出话题为`odom`，其消息类型为：`nav_msgs/msg/Odometry`

其数据主要包含三个部分： **（详细的接口内容请看硬件平台章节）**

*   header，表示该消息发布的时间

*   pose，表示当前机器人位置和朝向

*   twist，表示当前机器人的线速度和角速度

*   数据中还包含一个covariance，其代表协方差矩阵，后面小鱼写篇文章来介绍下，这里只需了解其含义即可。

里程计TF信息也可以输出：设为true，订阅tf话题里你就可以看到像下面的msg，建议后面配置好后，手动修改下，对比区别

```yaml
- header:
    stamp:
      sec: 6157
      nanosec: 907000000
    frame_id: odom
  child_frame_id: base_footprint
  transform:
    translation:
      x: 0.0005557960241049835
      y: -0.0007350446303238693
      z: 0.01599968753145574
    rotation:
      x: 4.691143395208505e-07
      y: 7.115496626557812e-06
      z: -0.018531475772549166
      w: 0.9998282774331005
```

轮子TF信息也可以输出：设为true，订阅tf话题里你就可以看到像下面的msg，建议后面配置好后，手动修改下，对比区别

```yaml
- header:
    stamp:
      sec: 6157
      nanosec: 941000000
    frame_id: base_link
  child_frame_id: left_wheel_link
  transform:
    translation:
      x: -0.02
      y: 0.1
      z: -0.06
    rotation:
      x: 0.0
      y: 0.049519025127821005
      z: 0.0
      w: 0.9987731805321918
- header:
    stamp:
      sec: 6157
      nanosec: 941000000
    frame_id: base_link
  child_frame_id: right_wheel_link
  transform:
    translation:
      x: -0.02
      y: -0.1
      z: -0.06
    rotation:
      x: 0.0
      y: -0.0663387077034509
      z: 0.0
      w: 0.9977971616817898

```

咱们之前下载的那个鱼香ROS的fishbot的urdf已经包含了差速插件的内容，如下：

因为是给Gazebo的插件，所以在`URDF`中，我们需要使用`<gazebo>`进行配置，因为是要给`gazebo`配置插件，所有要在`gazebo`标签下添加`plugin`子插件。

```xml
  <gazebo>
    <plugin name='diff_drive' filename='libgazebo_ros_diff_drive.so'>
          <ros>
            <namespace>/</namespace>
            <remapping>cmd_vel:=cmd_vel</remapping>
            <remapping>odom:=odom</remapping>
          </ros>
          <update_rate>30</update_rate>

          <left_joint>left_wheel_joint</left_joint>
          <right_joint>right_wheel_joint</right_joint>

          <wheel_separation>0.2</wheel_separation>
          <wheel_diameter>0.065</wheel_diameter>

          <max_wheel_torque>20</max_wheel_torque>
          <max_wheel_acceleration>1.0</max_wheel_acceleration>

          <publish_odom>true</publish_odom>
          <publish_odom_tf>true</publish_odom_tf>
          <publish_wheel_tf>true</publish_wheel_tf>
          <odometry_frame>odom</odometry_frame>
          <robot_base_frame>base_footprint</robot_base_frame>
      </plugin>
    </gazebo> 
```

编译一下：

```python
colcon build
source install/setup.bash
ros2 launch fishbot_description gazebo.launch.py
```

然后可以看看下面这俩命令：

```python
ros2 node list
ros2 topic list
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1650.webp)

可以看到了我们插件订阅的的/cmd\_vel和发布的/odom了。

然后我们可以通过teleop-twist-keyboard节点发布cmd\_vel来控制fishbot。

```python
sudo apt install ros-humble-teleop-twist-keyboard
```

使用下方节点来控制

```python
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

接着尝试使用来控制机器人运动

```python
   U    I    O
   J    K    L
   M    <    >
```

点一下，你就能看到fishbot在Gazebo中飞速的移动。接着打开终端，打印一下odom话题和tf话题，移动机器人观察数据变化。

也可以使用rqt显示速度数据

```python
rqt
```

选择Plugin->Visualization->Plot

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1651.webp)

在上方Topic输入`/cmd_vel/linear/x`，再输入`/cmd_vel/angular/z`，然后用键盘控制机器人移动。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1652.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1653.webp)

也可以在rviz2里看odom

```python
rviz2
```

点键盘控制节点按U，让机器人转圈。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1654.webp)

虽然机器人的轨迹已经在RVIZ中显示出来了，但是并没有机器人的模型，也看不到轮子的转动，咱们来带你一起解决这个问题。

前面咱们介绍过，要发布机器人模型我们所使用的节点是`robot_state_publisher`,所以我们在`gazebo.launch.py`中加入这个节点，同时再加上rviz2的启动节点，最终的`gazebo.launch.py`内容如下：

```python
from launch import LaunchDescription
from launch_ros.actions import Node

# 封装终端指令相关类
from launch.actions import ExecuteProcess

# from launch.substitutions import FindExecutable

# 参数声明与获取

# from launch.actions import DeclareLaunchArgument

# from launch.substitutions import LaunchConfiguration

# 文件包含相关

# from launch.actions import IncludeLaunchDescription

# from launch.launch_description_sources import PythonLaunchDescriptionSource

# 分组相关

# from launch_ros.actions import PushRosNamespace

# from launch.actions import GroupAction

# 事件相关

# from launch.event_handlers import OnProcessStart,OnProcessExit

# from launch.actions import ExecuteProcess,RegisterEventHandler,LogInfo

# 获取功能包下share目录或路径
from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    robot_name_in_model = 'fishbot'
    package_name = 'fishbot_description'
    urdf_name = "fishbot_gazebo.urdf"

    pkg_share = get_package_share_directory(f"{package_name}")
    urdf_model_path = os.path.join(pkg_share, f'urdf/urdf/{urdf_name}')

    # gazebo_world_path = os.path.join(pkg_share, 'world/fishbot.world')

    # Start Gazebo server

    # start_gazebo_cmd = ExecuteProcess(

    #     cmd=['gazebo', '--verbose','-s', 'libgazebo_ros_init.so', '-s', 'libgazebo_ros_factory.so',gazebo_world_path],

    #     output='screen')
    start_gazebo_cmd = ExecuteProcess(
        cmd=['gazebo', '--verbose','-s', 'libgazebo_ros_init.so', '-s', 'libgazebo_ros_factory.so'],
        output='screen')

    # Launch the robot
    spawn_entity_cmd = Node(
        package='gazebo_ros', 
        executable='spawn_entity.py',
        arguments=['-entity', robot_name_in_model,  '-file', urdf_model_path ], output='screen')

    # Start Robot State publisher
    start_robot_state_publisher_cmd = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        arguments=[urdf_model_path]
    )

    # Launch RViz
    start_rviz_cmd = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        output='screen',

        # arguments=['-d', default_rviz_config_path]
        )

    return LaunchDescription([start_gazebo_cmd,spawn_entity_cmd,start_robot_state_publisher_cmd,start_rviz_cmd])
```

保存编译启动

```python
colcon build
ros2 launch fishbot_description gazebo.launch.py
```

这样rviz2里就有模型了。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1655.webp)

可以保存下rviz2的配置

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1656.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1657.webp)

然后在launch里添加上rviz2的路径配置：

```python
    default_rviz_config_path = os.path.join(pkg_share, 'rviz/rviz2.rviz')

    # Launch RViz
    start_rviz_cmd = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        output='screen',
        arguments=['-d', default_rviz_config_path]
        )
```

如下为全部的launch：

```python
from launch import LaunchDescription
from launch_ros.actions import Node

# 封装终端指令相关类
from launch.actions import ExecuteProcess

# from launch.substitutions import FindExecutable

# 参数声明与获取

# from launch.actions import DeclareLaunchArgument

# from launch.substitutions import LaunchConfiguration

# 文件包含相关

# from launch.actions import IncludeLaunchDescription

# from launch.launch_description_sources import PythonLaunchDescriptionSource

# 分组相关

# from launch_ros.actions import PushRosNamespace

# from launch.actions import GroupAction

# 事件相关

# from launch.event_handlers import OnProcessStart,OnProcessExit

# from launch.actions import ExecuteProcess,RegisterEventHandler,LogInfo

# 获取功能包下share目录或路径
from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    robot_name_in_model = 'fishbot'
    package_name = 'fishbot_description'
    urdf_name = "fishbot_gazebo.urdf"

    pkg_share = get_package_share_directory(f"{package_name}")
    urdf_model_path = os.path.join(pkg_share, f'urdf/urdf/{urdf_name}')

    # gazebo_world_path = os.path.join(pkg_share, 'world/fishbot.world')
    default_rviz_config_path = os.path.join(pkg_share, 'rviz/rviz2.rviz')

    # Start Gazebo server

    # start_gazebo_cmd = ExecuteProcess(

    #     cmd=['gazebo', '--verbose','-s', 'libgazebo_ros_init.so', '-s', 'libgazebo_ros_factory.so',gazebo_world_path],

    #     output='screen')
    start_gazebo_cmd = ExecuteProcess(
        cmd=['gazebo', '--verbose','-s', 'libgazebo_ros_init.so', '-s', 'libgazebo_ros_factory.so'],
        output='screen')

    # Launch the robot
    spawn_entity_cmd = Node(
        package='gazebo_ros', 
        executable='spawn_entity.py',
        arguments=['-entity', robot_name_in_model,  '-file', urdf_model_path ], output='screen')

    # Start Robot State publisher
    start_robot_state_publisher_cmd = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        arguments=[urdf_model_path]
    )

    # Launch RViz
    start_rviz_cmd = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        output='screen',
        arguments=['-d', default_rviz_config_path]
        )

    return LaunchDescription([start_gazebo_cmd,spawn_entity_cmd,start_robot_state_publisher_cmd,start_rviz_cmd])
```

可以自行编译测试。

##### 惯性计IMU
上节课通过配置两轮差速控制器我们已经成功的让fishbot在gazebo中动了起来，本节课我们通过给fishbot的URDF配置IMU传感器插件，让IMU模块工作起来。

惯性测量单元是测量物体三轴姿态角(或角速率)以及加速度的装置。一般的，一个IMU包含了三个单轴的加速度计和三个单轴的陀螺，加速度计检测物体在载体坐标系统独立三轴的加速度信号，而陀螺检测载体相对于导航坐标系的角速度信号，测量物体在三维空间中的角速度和加速度，并以此解算出物体的姿态。在导航中有着很重要的应用价值。

上面这段话是小鱼从百科中摘抄出来的，你需要知道的一个关键点是IMU可以测量以下三组数据：

*   三维加速度计加速度

*   三维陀螺仪角速度

*   三维磁力计（有的也没有磁力计）

用六轴、九轴算法或其他算法等可以输出三轴欧拉角（Yaw,Pitch,Roll），欧拉角可以转化为四元数。

IMU长啥样？直接线下找控制组要就行，他们经常会玩这个。

便宜的长这样（MPU6050,MPU9050等）：

MPU6050是六轴的，MPU9050是九轴的。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1658.webp)

贵的长这样（HWT101CT，HWT605等）：

其中HWT101CT是三轴的，HWT605是六轴的。（各有优缺点）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1659.webp)

不要钱的长什么样？

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1660.webp)

仿真的不要钱哈哈，接着我们来配置一下仿真的IMU。

IMU对应的消息类型为`sensor_msgs/msg/Imu`

ROS的imu信息只有三轴加速度，角速度和四元数。（并没有磁力计和欧拉角，磁力计并非必须要填的，而欧拉角和四元数可以互相转换，四元数更好被算法运算，所以选择用四元数）

具体imu接口请看硬件平台章节。

可以看到除了每个数据对应的三个协方差之外，每一个还都对应一个`3*3`的协方差矩阵。

有了上节课的经验，我们可以很轻松的添加IMU传感器，但是还有一个需要注意的地方，为了更真实的模拟IMU传感器，我们需要给我们的仿真IMU传感器加点料。

加什么？加点高斯噪声，高斯噪声只需要指定平均值和标准差两个参数即可，不过因为IMU传感器的特殊性，我们还需要给模型添加两个偏差参数，分别是 `平均值偏差`和`标准差偏差`。

有关Gazebo仿真和噪声模型更深入的介绍可以参考鱼香ROS发的两篇推文：

*   [Gazebo仿真进阶教程之传感器高斯噪声（一）](https://mp.weixin.qq.com/s/0-OEATkyfMf6wEyrP5csGw)

*   [Gazebo仿真进阶教程之传感器高斯噪声（二）](https://mp.weixin.qq.com/s/5k1SEGdASjUMbwWdSpf1PQ)

下面是IMU传感器的URDF配置代码，大家结合文章对应可以理解一下，IMU对应的插件库`libgazebo_ros_imu_sensor.so`：

```xml
    <gazebo reference="imu_link">
      <sensor name="imu_sensor" type="imu">
      <plugin filename="libgazebo_ros_imu_sensor.so" name="imu_plugin">
          <ros>
            <namespace>/</namespace>
            <remapping>~/out:=imu</remapping>
          </ros>
          <initial_orientation_as_reference>false</initial_orientation_as_reference>
        </plugin>
        <always_on>true</always_on>
        <update_rate>100</update_rate>
        <visualize>true</visualize>
        <imu>
          <angular_velocity>
            <x>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>2e-4</stddev>
                <bias_mean>0.0000075</bias_mean>
                <bias_stddev>0.0000008</bias_stddev>
              </noise>
            </x>
            <y>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>2e-4</stddev>
                <bias_mean>0.0000075</bias_mean>
                <bias_stddev>0.0000008</bias_stddev>
              </noise>
            </y>
            <z>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>2e-4</stddev>
                <bias_mean>0.0000075</bias_mean>
                <bias_stddev>0.0000008</bias_stddev>
              </noise>
            </z>
          </angular_velocity>
          <linear_acceleration>
            <x>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>1.7e-2</stddev>
                <bias_mean>0.1</bias_mean>
                <bias_stddev>0.001</bias_stddev>
              </noise>
            </x>
            <y>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>1.7e-2</stddev>
                <bias_mean>0.1</bias_mean>
                <bias_stddev>0.001</bias_stddev>
              </noise>
            </y>
            <z>
              <noise type="gaussian">
                <mean>0.0</mean>
                <stddev>1.7e-2</stddev>
                <bias_mean>0.1</bias_mean>
                <bias_stddev>0.001</bias_stddev>
              </noise>
            </z>
          </linear_acceleration>
        </imu>
      </sensor>
    </gazebo>
```

我们之前下载的fishbot已经包含该内容了，所以不用再添加了，直接运行即可。

```python
ros2 launch fishbot_description gazebo.launch.py
```
```python
ros2 topic list
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1661.webp)

```python
ros2 topic info /imu
ros2 topic echo /imu
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1662.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1663.webp)

```yaml
header:
  stamp:
    sec: 150
    nanosec: 599000000
  frame_id: base_footprint
orientation:
  x: 3.434713830866392e-07
  y: 7.119913105768616e-06
  z: -0.00028312437320413914
  w: 0.9999999598948884
orientation_covariance:
- 0.0
- 0.0
- 0.0
- 0.0
- 0.0
- 0.0
- 0.0
- 0.0
- 0.0
angular_velocity:
  x: -0.00013597855247901325
  y: 0.0006306135617081868
  z: -0.00015794894627685146
angular_velocity_covariance:
- 4.0e-08
- 0.0
- 0.0
- 0.0
- 4.0e-08
- 0.0
- 0.0
- 0.0
- 4.0e-08
linear_acceleration:
  x: 0.08679200038530369
  y: 0.07753419258567491
  z: 9.687910969061628
linear_acceleration_covariance:
- 0.00028900000000000003
- 0.0
- 0.0
- 0.0
- 0.00028900000000000003
- 0.0
- 0.0
- 0.0
- 0.00028900000000000003

```

用rqt可视化：

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1664.webp)

##### 雷达Laser
本节我们来认识一个新的传感器，该传感器在自动驾驶、室内导航等应用非常多，比如扫地机器人上就是用的它作为感知环境的重要工具，该传感器是激光雷达。

激光雷达（Light Detection And Ranging）,缩写`LiDAR`，英文也叫laser,翻译一下叫——激光探测与测距。

激光雷达的原理也很简单，就像蝙蝠的定位方法一样，蝙蝠定位大家都知道吧，像下面这样子的回声定位。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1665.webp)

普通的单线激光雷达一般有一个发射器，一个接收器，发射器发出激光射线到前方的目标上，物品会将激光反射回来，然后激光雷达的接受器可以检测到反射的激光。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1666.webp)

通过计算发送和反馈之间的时间间隔，乘上激光的速度，就可以计算出激光飞行的距离，该计算方法成为TOF（飞行时间法Time of flight，也称时差法）。

除了TOF之外还有其他方法进行测距，比如三角法，这里就不拓展了放一篇文章，大家自行阅读。[激光三角测距原理详述](https://www.slamtec.com/cn/News/Detail/190)

目前市面上的激光雷达，几乎都是采用三角测距，比如思岚的：

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1667.webp)

需要注意的是虽然只有一个发射器和一个接受器，激光雷达通过电机可以进行旋转，这样就可以达到对周围环境360度测距的目的。

五位数的长这样：

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1668.webp)

四位数的长这样（咱们有一台）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1669.webp)

三位数的长这样（咱们也有）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1670.webp)

两位数的长这样

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1671.webp)

不要钱的长这样

仿真的，不要钱

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1672.webp)

因为激光雷达是属于射线类传感器，该类传感在在Gazebo插件中都被封装成了一个动态库`libgazebo_ros_ray_sensor.so`。

接着我们来看看LiDAR的话题消息接口`sensor_msgs/msg/LaserScan`。

雷达的数据结构有些复杂，但通过注释和名字相信你可以看的七七八八，看不懂也没关系，一般情况下我们不会直接的对雷达的数据做操作。

雷达的模型不需要collision，请删掉，否则会挡激光射出。

有了前面的经验，我们需要在URDF添加以下内容即可，但我们下载的是鱼香ROS添加好的，所以不用改了：

```xml
  <gazebo reference="laser_link">
      <sensor name="laser_sensor" type="ray">
      <always_on>true</always_on>
      <visualize>true</visualize>
      <update_rate>10</update_rate>
      <pose>0 0 0.075 0 0 0</pose>
      <ray>
          <scan>
            <horizontal>
              <samples>360</samples>
              <resolution>1.000000</resolution>
              <min_angle>0.000000</min_angle>
              <max_angle>6.280000</max_angle>
            </horizontal>
          </scan>
          <range>
            <min>0.120000</min>
            <max>3.5</max>
            <resolution>0.015000</resolution>
          </range>
          <noise>
            <type>gaussian</type>
            <mean>0.0</mean>
            <stddev>0.01</stddev>
          </noise>
      </ray>

      <plugin name="laserscan" filename="libgazebo_ros_ray_sensor.so">
        <ros>
          <remapping>~/out:=scan</remapping>
        </ros>
        <output_type>sensor_msgs/LaserScan</output_type>
        <frame_name>laser_link</frame_name>
      </plugin>
      </sensor>
    </gazebo>
```

可以看到:

1.  雷达也可以设置更新频率`update_rate`，这里设置为5

2.  雷达可以设置分辨率，设置为1，采样数量360个，最终生成的点云数量就是360

3.  雷达也有噪声，模型为`gaussian`

4.  雷达有扫描范围`range`，这里配置成0.12-3.5，0.015分辨率

5.  雷达的`pose`就是雷达的joint中位置的设置值

下面这个蓝色的就是激光雷达的光线覆盖范围：

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1673.webp)

```python
ros2 topic list
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1674.webp)

```python
ros2 topic info /scan
ros2 topic echo /scan
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1675.webp)

接着我们尝试使用rviz2进行可视化激光雷达数据

添加和修改RVIZ2的如下：（通过LaserScan插件可以看到激光数据）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1676.webp)

相信你改完之后依然是看不到任何激光雷达的数据的，反看topic的echo出来的数据，不是0就是inf(无限大)，再看看gazebo你会发现，激光雷达并没有达到任何一个物体上。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1677.webp)

所以我们可以手动的给激光雷达周围添加一下东西，点击Gazebo工具栏的正方体，圆球或者圆柱，随意放置几个到我们激光雷达的最大扫描半径内。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1678.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1679.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1680.webp)

##### 超声波Ultrasonic
这玩意对于ROS2算法意义不是很大，控制组直接在MCU上实现即可，有需求可以学本节。 **（可以跳过本节）**

在实际的机器人开发过程中，我们可能会利用超声波传感器实现实时避障的功能，毕竟超声波的价格相较于激光雷达要便宜很多（便宜的几块钱）。

所以本节我们来说一下如何使用ROS2+Gazebo来仿真超声波传感器。

百科来一段：

超声波传感器是将超声波信号转换成其它能量信号（通常是电信号）的传感器。超声波是[振动频率](https://baike.baidu.com/item/%E6%8C%AF%E5%8A%A8%E9%A2%91%E7%8E%87/8068137)高于20kHz的机械波。它具有频率高、波长短、绕射现象小，特别是方向性好、能够成为[射线](https://baike.baidu.com/item/%E5%B0%84%E7%BA%BF/327964)而定向传播等特点。超声波对液体、固体的穿透本领很大，尤其是在阳光不透明的固体中。超声波碰到杂质或分界面会产生显著反射形成反射回波，碰到活动物体能产生[多普勒效应](https://baike.baidu.com/item/%E5%A4%9A%E6%99%AE%E5%8B%92%E6%95%88%E5%BA%94/115710)。超声波传感器广泛应用在工业、国防、生物医学等方面。

接着看看长什么样子：

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1681.webp)

便宜的就长这样子，一共两个头，一个头用于发送波，一个头接收波。这个还稍微高级一点，带一个光敏电阻，可以为超声波数据做一些补偿。

超声波传感器原理是什么呢？

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1682.webp)

```bash
距离=(发送时间-接收时间)*速度/2Copy to clipboardErrorCopied
```

看了超声波的原理，你有没有发现和前面的激光雷达传感器是一样的，是的，所以超声波传感器插件和激光雷达传感器插件在Gazebo插件中是同一个：

```bash
libgazebo_ros_ray_sensor.so
```

超声波总要装在机器人身上某个位置，所以我们先添加一个关节和Joint，为了省事，link我们就只写个名字，你如果有需要可以按照前面的章节那样添加一下。

```xml
  <link name="ultrasonic_sensor_link" />

  <joint name="ultrasonic_sensor_joint" type="fixed">
    <parent link="base_link"/>
    <child link="ultrasonic_sensor_link"/>
    <origin xyz="0.07 0.0 0.076" rpy="0 0 0"/>
  </joint>
```

添加完了关节，我们就可以配置gazebo的插件了，gazebo插件配置如下

```xml
  <gazebo reference="ultrasonic_sensor_link">
    <sensor type="ray" name="ultrasonic_sensor">
      <pose>0 0 0 0 0 0</pose>

      <visualize>true</visualize>

      <update_rate>5</update_rate>
      <ray>
        <scan>

          <horizontal>
            <samples>5</samples>
            <resolution>1</resolution>
            <min_angle>-0.12</min_angle>
            <max_angle>0.12</max_angle>
          </horizontal>

          <vertical>
            <samples>5</samples>
            <resolution>1</resolution>
            <min_angle>-0.01</min_angle>
            <max_angle>0.01</max_angle>
          </vertical>
        </scan>

        <range>
          <min>0.02</min>
          <max>4</max>
          <resolution>0.01</resolution>
        </range>

        <noise>
          <type>gaussian</type>
          <mean>0.0</mean>
          <stddev>0.01</stddev>
        </noise>
      </ray>
      <plugin name="ultrasonic_sensor_controller" filename="libgazebo_ros_ray_sensor.so">
        <ros>

          <remapping>~/out:=ultrasonic_sensor_1</remapping>
        </ros>

        <output_type>sensor_msgs/Range</output_type>

        <radiation_type>ultrasound</radiation_type>

        <frame_name>ultrasonic_sensor_link</frame_name>
      </plugin>
    </sensor>
  </gazebo>

```

添加完成后就可以编译测试下代码

```python
colcon build --packages-select fishbot_description
source install/setup.bash
ros2 launch fishbot_description gazebo.launch.py
```

没有物体的前面可以放个东西

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1683.webp)

打开终端，输入下面指令

```bash
ros2 topic list 
ros2 topic info /ultrasonic_sensor_1
ros2 topic echo /ultrasonic_sensor_1Copy to clipboardErrorCopied
```

不出意外可以看到下面的数据

```bash
header:
  stamp:
    sec: 4458
    nanosec: 1000000
  frame_id: ultrasonic_sensor_link
radiation_type: 0
field_of_view: 0.23999999463558197
min_range: 0.019999999552965164
max_range: 4.0
range: 2.6798219680786133
```

这里的range就是fishbot到墙之间的距离：2.67982

我们来讲一讲超声波传感器的数据类型`sensor_msgs/msg/Range`

```python

# ros2 topic info /ultrasonic_sensor_1
Type: sensor_msgs/msg/Range
Publisher count: 1
Subscription count: 0

```

你可以使用`ros2 interface show sensor_msgs/msg/Range`看到详细的解释，我们翻译一下

```python

# Single range reading from an active ranger that emits energy and reports

# one range reading that is valid along an arc at the distance measured.

# This message is  not appropriate for laser scanners. See the LaserScan

# message if you are working with a laser scanner.
#

# This message also can represent a fixed-distance (binary) ranger.  This

# sensor will have min_range===max_range===distance of detection.

# These sensors follow REP 117 and will output -Inf if the object is detected

# and +Inf if the object is outside of the detection range.

std_msgs/Header header # timestamp in the header is the time the ranger

                             # returned the distance reading

# Radiation type enums

# If you want a value added to this list, send an email to the ros-users list
uint8 ULTRASOUND=0
uint8 INFRARED=1

uint8 radiation_type    # 传感器射线类型

                        # (sound, IR, etc) [enum]

float32 field_of_view   # 距离数据对应的弧[rad]的大小，测量物体的范围介于         

                        # -field_of_view/2 到 field_of_view/2 之间。

                        # 0 角度对应于传感器的 x 轴。

float32 min_range       # 最小范围值 [m]
float32 max_range       # 最大范围值 [m]

                        #  固定距离需要 min_range==max_range

float32 range           # 范围数据 [m]

                        # (Note: values < range_min or > range_max should be discarded)

                        # Fixed distance rangers only output -Inf or +Inf.

                        # -Inf represents a detection within fixed distance.

                        # (Detection too close to the sensor to quantify)

                        # +Inf represents no detection within the fixed distance.

                        # (Object out of range)

```

结论，主要关注range就可以了。

在rviz2里添加超声波数据

Add ->By topic->Range

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1684.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1685.webp)

#### 搭建世界地图
本节我们要在Gazebo中建立一个测试的环境，其实也很简单，利用Gazebo的画墙工具即可完成。

world即世界，gazebo的world文件就是用于描述世界模型的，也就是环境模型。

Gazebo已经为我们准备了很多常用的物体模型，除了基础的圆球，圆柱，立方体外的，其实还有飞机、汽车、房子等你现实中无法拥有的。

但是一开始安装Gazebo的时候并不会帮你下载好这些模型，需要我们手动下载，找一个你要存模型的文件夹，打开终端，复制粘贴下面这句

```python
git clone https://github.com/osrf/gazebo_models
```

并把存放模型的路径加到~/.bashrc里（第二个冒号后面的可以不写，第二个冒号后面的是多个路径。）

GAZEBO\_MODEL\_PATH是老版Gazebo Classic的宏。

IGN\_GAZEBO\_RESOURCE\_PATH是新版Igntion Gazebo的宏。

模型都通用，可以一起配置上。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1686.webp)

刷新环境变量

```python
source ~/.bashrc
```

此时再次打开终端，输入`gazebo`，把选项卡切换到Insert

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1687.webp)

在Insert选项卡下可以看到一个目录，以及目录下的模型名称，随着下载脚本的不断下载，这里的模型会越来越多。

随手拖几个，搭建一个漂亮的环境出来~

每个成功的男人都有一辆车，咱们也不例外

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1688.webp)

上面是Gazebo为我们准备好的开源模型，我们也可以通过Gazebo的工具来自己画一个环境。

然后也可以用墙壁工具建墙

Gazebo左上角->Edit->Building Editor

接着可以看到这样一个编辑界面

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1689.webp)

点击左边的Wall,你就可以在上方的白色区域进行建墙了，这个和模拟人生游戏不一样，这个是画二维的墙生成三维的墙，模拟人生是直接在三维里画。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1690.webp)

建完后还可以用选Add Color或者Add Texture，然后点击下方墙，给墙添加颜色或者纹理。

首先你要有一个地图，小鱼为你准备了两个，两个图片都是800\*600像素的。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1691.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1692.webp)

打开Gazebo->Gazebo左上角->Edit->Building Editor->左下方选Import

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1693.webp)

将上面两个图片存到本地，在这个界面选图片，记着选Next

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1694.webp)

左边选尺寸对应关系

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1695.webp)

我们选择默认的，100像素/米。点击OK（需要手动将100改变一下才能点击OK哦），之后就可以用图片画墙了。

注意：导入完图片不会直接出来墙，图片只是提供了墙的大概位置，需要你手动用墙再将边描一遍。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1696.webp)

建完后点击File->Exit,在退出的弹框中选Exit。

接着在Gazebo界面中就可以看到墙了，我们再手动添加几个物体，就可以用于下面的导航使用了。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1697.webp)

添加完，接着点击File->SaveWorld，将文件保存到我们的fishbot\_descrption的world下。

没有world目录的小伙伴可以先手动创建下

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1698.webp)

加载world其实也很简单，可以先启动Gazebo，再手动的加载文件，也可以在Gazebo启动时加载：

比如在前面加载ROS2插件基础上再加载fishbot.world。

```python
gazebo --verbose  -s libgazebo_ros_init.so -s  libgazebo_ros_factory.so 你的world文件目录/fishbot.world
```

修改launch文件，将上面的命令行写到`gazebo.launch.py`中即可。

```python
    gazebo_world_path = os.path.join(pkg_share, 'world/fishbot.world')

    # Start Gazebo server
    start_gazebo_cmd =  ExecuteProcess(
        cmd=['gazebo', '--verbose','-s', 'libgazebo_ros_init.so', '-s', 'libgazebo_ros_factory.so', gazebo_world_path],
        output='screen')

```

下面是整个launch文件：

```python
from launch import LaunchDescription
from launch_ros.actions import Node

# 封装终端指令相关类
from launch.actions import ExecuteProcess

# from launch.substitutions import FindExecutable

# 参数声明与获取

# from launch.actions import DeclareLaunchArgument

# from launch.substitutions import LaunchConfiguration

# 文件包含相关

# from launch.actions import IncludeLaunchDescription

# from launch.launch_description_sources import PythonLaunchDescriptionSource

# 分组相关

# from launch_ros.actions import PushRosNamespace

# from launch.actions import GroupAction

# 事件相关

# from launch.event_handlers import OnProcessStart,OnProcessExit

# from launch.actions import ExecuteProcess,RegisterEventHandler,LogInfo

# 获取功能包下share目录或路径
from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    robot_name_in_model = 'fishbot'
    package_name = 'fishbot_description'
    urdf_name = "fishbot_gazebo.urdf"
    world_name = "fishbot.world"

    pkg_share = get_package_share_directory(f"{package_name}")
    urdf_model_path = os.path.join(pkg_share, f'urdf/urdf/{urdf_name}')
    gazebo_world_path = os.path.join(pkg_share, f'world/{world_name}')
    default_rviz_config_path = os.path.join(pkg_share, 'rviz/rviz2.rviz')

    # Start Gazebo server
    start_gazebo_cmd = ExecuteProcess(
        cmd=['gazebo', '--verbose','-s', 'libgazebo_ros_init.so', '-s', 'libgazebo_ros_factory.so',gazebo_world_path],
        output='screen')

    # Launch the robot
    spawn_entity_cmd = Node(
        package='gazebo_ros', 
        executable='spawn_entity.py',
        arguments=['-entity', robot_name_in_model,  '-file', urdf_model_path ], output='screen')

    # Start Robot State publisher
    start_robot_state_publisher_cmd = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        arguments=[urdf_model_path]
    )

    # Launch RViz
    start_rviz_cmd = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        output='screen',
        arguments=['-d', default_rviz_config_path]
        )

    return LaunchDescription([start_gazebo_cmd,spawn_entity_cmd,start_robot_state_publisher_cmd,start_rviz_cmd])
```

最后记得修改cmakelists.txt文件，让编译后将world文件拷贝到install目录下

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1699.webp)

```python
colcon build
source install/setup.bash
ros2 launch fishbot_description gazebo.launch.py 
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1700.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1701.webp)

#### 框架优化
本节代码学长也发到仓库了，有需要的学弟学妹请看:https://github.com/tungchiahui/ROS\_WS/tree/main/ROS2\_WS%2F6.ws\_simulations%2Fsrc%2Ffishbot\_description

比如说支持xacro等优化。

首先先把原来的fishbot\_gazebo.urdf里的内容分成好几个urdf和xacro再用一个总的fishbot.urdf.xacro去引用。（学过xacro的肯定都会）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1702.webp)

优化后的launch如下：

```python
from launch import LaunchDescription
from launch_ros.actions import Node

# 封装终端指令相关类
from launch.actions import ExecuteProcess

# from launch.substitutions import FindExecutable

# 参数声明与获取
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration

# 文件包含相关

# from launch.actions import IncludeLaunchDescription

# from launch.launch_description_sources import PythonLaunchDescriptionSource

# 分组相关

# from launch_ros.actions import PushRosNamespace

# from launch.actions import GroupAction

# 事件相关

# from launch.event_handlers import OnProcessStart,OnProcessExit

# from launch.actions import ExecuteProcess,RegisterEventHandler,LogInfo

# 获取功能包下share目录或路径
from ament_index_python.packages import get_package_share_directory
from launch_ros.parameter_descriptions import ParameterValue
from launch.substitutions import Command,LaunchConfiguration
import os

def generate_launch_description():
    robot_name_in_model = 'fishbot'
    package_name = 'fishbot_description'

    # urdf_xacro_name = "fishbot_gazebo.urdf"
    world_name = "fishbot.world"

    pkg_share = get_package_share_directory(f"{package_name}")
    urdf_xacro_model_path = os.path.join(pkg_share, "urdf/xacro","fishbot.urdf.xacro")
    gazebo_world_path = os.path.join(pkg_share, f'world/{world_name}')
    default_rviz_config_path = os.path.join(pkg_share, 'rviz/rviz2.rviz')

    model = DeclareLaunchArgument(name="model", default_value=urdf_xacro_model_path)
    robot_description = ParameterValue(Command(["xacro ",LaunchConfiguration("model")]))

    # Start Gazebo server
    start_gazebo_cmd = ExecuteProcess(
        cmd=['gazebo', '--verbose','-s', 'libgazebo_ros_init.so', '-s', 'libgazebo_ros_factory.so',gazebo_world_path],
        output='screen')

    # Launch the robot
    spawn_entity_cmd = Node(
        package='gazebo_ros', 
        executable='spawn_entity.py',
        arguments=['-entity', robot_name_in_model,  '-topic', '/robot_description' ], output='screen')

    # Start Robot State publisher
    start_robot_state_publisher_cmd = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        parameters=[{"robot_description": robot_description}]
    )

    # Launch RViz
    start_rviz_cmd = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        output='screen',
        arguments=['-d', default_rviz_config_path]
        )

    return LaunchDescription([model,start_gazebo_cmd,spawn_entity_cmd,start_robot_state_publisher_cmd,start_rviz_cmd])
```
```python
colcon build
source install/setup.bash
ros2 launch fishbot_description gazebo.launch.py
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1703.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1704.webp)

然后就可以去搞导航啦！也可以接着学新版Ignition Gazebo，但这东西教程巨少，截止2024年只有赵虚左老师讲了。
** 建议学习新的	Gazebo Harmonic （ROS2 Jazzy默认的版本） **

### Ignition Gazebo（Gazebo Fortress，基于ROS2 Humble）

** 建议使用ROS2 Jazzy的，这个ROS2 Humble的Gazebo像过渡版本，代码可能和后续版本又冲突！ **

详见<NuxtLink to="/wiki/ros2-tutorial#Gz Sim（Gazebo Harmonic 及之后的版本（ROS2 Jazzy及之后的版本））">Gz Sim教程</NuxtLink>

#### Ign Gazebo安装与运行
Gazebo每个版本的变化都很大。

特别是ROS1用的老版Gazebo(黑色界面)和ROS2用的新版Gazebo(白色界面)。

ROS2的不同版本的Gazebo跨度也很大，比如Humble和Jazzy及Jazzy之后的版本之间很多标签区别很大。

本文使用humble版本(即Ignition Gazebo)当做教程。

当然，为了兼容以后的Gazebo，在下方也会有教程教你如何从ign gazebo迁移到gazebo sim（最最最新版gazebo）。

Ignition Gazebo 是 ROS2 中使用的全新机器人仿真工具，**它是 Gazebo 的升级版本**。在Humble他还叫Ignition Gazebo(也叫Gazebo Fortress)，在Jazzy中叫Gazebo Harmonic(去掉了Ignition的名字)(https://community.gazebosim.org/t/a-new-era-for-gazebo/1356)。它具备更好的性能和可用性，并通过紧密集成 ROS2 来提供强大的仿真环境。Ignition Gazebo 支持各种机器人平台和传感器，并提供灵活的配置选项和易于使用的界面。它的物理引擎和传感器模型可以帮助开发人员进行机器人系统的开发、测试和验证。无论是研究还是教育，Ignition Gazebo 都是一个强大的工具。

如果想从Ignition Gazebo(ROS2 Humble)迁移到Gazebo(ROS2 Jazzy)，请往下翻翻，下面有一节是讲如何迁移的。

https://docs.ros.org/en/humble/Tutorials/Advanced/Simulators/Gazebo/Gazebo.html

下面这个网站是官方教程(ROS2 Humble的Ignition Gazebo Fortress)：

https://gazebosim.org/docs/fortress/getstarted/

https://gazebosim.org/docs/fortress/library\_reference\_nav/

源码：https://github.com/gazebosim/docs/blob/master/fortress/tutorials

**安装**

Ignition Gazebo 是不依赖于ROS2的一个独立的项目，可以独自安装。但是如果安装了ROS2，在ROS2存储库中已经集成了对应版本的 Ignition Gazebo，可以调用如下指令直接安装：

```bash

# 通用命令
sudo apt install ros-${ROS_DISTRO}-ros-gz

# Humble版本
sudo apt install ros-humble-ros-gz

# Jazzy版本
sudo apt install ros-jazzy-ros-gz
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1705.webp)

**运行**

Ignition Gazebo 安装完毕之后，可以通过两种方式启动。

方式1，以Ignition Gazebo 的方式启动，指令如下：

```bash

# Humble版本
ign gazebo

# Jazzy版本
gz sim
```

方式2，以ROS2的方式 启动，指令如下 ：

```bash
ros2 launch ros_gz_sim gz_sim.launch.py
```

二者运行结果一致，如下图所示：在弹出窗口中，选择仿真环境然后点击`run`按钮即可运行。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1706.webp)

**界面介绍**

接下来以Empty仿真环境为例，介绍一下Ignition Gazebo的界面组成。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1707.webp)

注意：如果你的Gazebo不卡，但是Ignition Gazebo巨卡的话，请确认Ignition Gazebo是以独显打开的，而不是核显。

如果不会切换应用显卡，可以直接把核显关闭掉，从混合输出切换为独立显卡输出。

**工具栏**

*   顶部的工具栏包含两个按钮，左侧的文件菜单按钮（水平条纹）和右侧的插件按钮（垂直省略号）。

1.  文件菜单按钮（水平条纹）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1708.webp)

*   文件菜单按钮包含将仿真环境保存到文件、保存和加载界面配置以及自定义界面样式等设置。

2.  右侧的插件按钮（垂直省略号）

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1709.webp)

*   插件按钮列出了所有可用的插件。点击后会弹出插件列表，向下滚动此列表以查看所有插件。 当选择一个时，其界面将出现在右侧面板中。

**3D视窗**

*   左上方工具栏包含多种几何体（球体、框体、圆柱体）按钮和变换控件。通过集合体按钮可以直接将盒子、球体或圆柱体模型插入仿真环境。只需单击要插入的形状，然后将其放入环境中。该形状将自动捕捉到地平面上。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1710.webp)![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1711.webp)

*   主视图会显示仿真环境，我们可以通过鼠标以不同方式来导航场景，相关操作如下：

```bash
左键单击：选择实体
右键单击：打开带有选项的菜单：
   Move to：移动到以实体为中心的场景
   Follow：选择一个实体让视图保持居中，无论是移动还是平移
   Remove：从模拟中删除实体
   Copy：复制实体
   Past: 粘贴实体
   View：显示实体的重心（Center of Mass）、碰撞边界（Collisions）、惯性（Inertia）、
         关节（Joints）、坐标系（Frames）、透明度（Transparent）、线框（Wireframe）等属性
左键单击并拖动：在场景中平移
右键单击并拖动：放大和缩小
滚轮向前/向后：放大和缩小
滚轮单击并拖动：旋转场景
```

*   想移动这个球，需要点左上角的移动模式，再左键单击选中物体

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1712.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1713.webp)

*   在视窗的底部，从左到右分别是是播放、步长按钮和实时因子（Real-Time Factor，RTF）。点击播放按钮将开始运行仿真环境， 再次点击可以暂停运行。步长按钮用于设置仿真时间的离散单位，可以通过将鼠标悬停在按钮上来自定义步长。实时因子表示仿真运行速度相对于真实时间的比例。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1714.webp)

**右侧面板**

右侧面板用于显示插件，当前仿真环境默认包含两个插件：Model和Entity Tree。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1715.webp)

*   Entity Tree 中会显示仿真环境中的实体列表；

*   点击Entity Tree中的实体后，可以在Model中显示该实体的相关信息。

*   也可以按住 Ctrl 并单击以选择多个实体；

*   还可以右键单击任何插件以打开基本设置或关闭。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1716.webp)

在Ignition Gazebo中内置了许多插件，可以点击工具栏的右侧按钮自行添加，比如：可以选择 Grid Config 插件调整世界网格的特征，包括单元格大小、网格位置、单元格计数、或颜色等。

后期随着应用的深入，也会陆续介绍其他一些插件。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1717.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1718.webp)

#### 与ROS2集成
本节将介绍如何实现Ignition Gazebo与ROS2的集成，以实现二者之间的交互，比如，可以通过ROS2的键盘控制节点控制机器人运动，并且在rviz2中显示机器人的里程计(odom)数据。其流程大致如下：

1.  启动 Ignition Gazebo 仿真环境；

2.  通过 ros\_gz\_bridge 建立 ROS2 与 Ignition Gazebo 的连接；

3.  启动 ROS2 相关节点实现与 Ignition Gazebo 的数据收发。

Ignition Gazebo与ROS2的的所有集成实现，基本都遵循上述流程。

**启动仿真环境**

在 Ignition Gazebo 安装时，已经内置了一些仿真环境，直接启动即可。在此我们可以使用名为`visualize_lidar.sdf`的仿真文件，该文件对应的仿真环境中包括了差速机器人以及激光雷达的仿真。启动指令如下：

```bash
ign gazebo -v 4 -r visualize_lidar.sdf
#或者
gz sim -v 4 -r visualize_lidar.sdf
```

或者也可以以ROS2 launch的方式启动，指令如下：

```bash
ros2 launch ros_gz_sim gz_sim.launch.py gz_args:="-v 4 -r visualize_lidar.sdf" # 启动状态
```

两种方式本质相同，都是启动了Ignition Gazebo并且加载了visualize\_lidar.sdf文件。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1719.webp)

**建立连接**

虽然仿真环境中的机器人已经配置了运动控制插件，可以通过`/model/vehicle_blue/cmd_vel`话题订阅速度指令并运动，但是Ignition Gazebo与ROS2中的消息格式并不一致，所以还需要通过ros\_gz\_bridge这一桥接功能包，实现二者之间消息的转换，调用指令如下：

```bash
ros2 run ros_gz_bridge parameter_bridge /model/vehicle_blue/cmd_vel@geometry_msgs/msg/Twist]gz.msgs.Twist
```

通过该指令可以将发布在`/model/vehicle_blue/cmd_vel`话题上的`geometry_msgs/msg/Twist`类型的ROS2消息转换成可以被Ignition Gzebo识别的`gz.msgs.Twist`类型的消息。

**启动ROS2节点**

启动ROS2的键盘控制节点，并将话题重映射为`/model/vehicle_blue/cmd_vel`，指令如下：

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -r /cmd_vel:=/model/vehicle_blue/cmd_vel
```

接下来就可以使用键盘控制机器人运动了。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1720.webp)

#### ros\_gz\_bridge
ros\_gz\_bridge是连接ROS2与Ignition Gazebo的桥梁，ROS2与Ignition Gazebo使用的消息并不兼容，必须通过ros\_gz\_bridge进行转换。

**ros\_gz\_bridge使用语法**

ROS2与Ignition Gazebo的桥接是通过ros\_gz\_bridge包中的parameter\_bridge节点实现，其使用语法如下：

```bash
parameter_bridge [<topic@ROS2_type@Ign_type> ..]  [<service@ROS2_srv_type[@Ign_req_type@Ign_rep_type]> ..]
```

在话题Topic中， **第一个@** 符号是话题名称和消息类型的 **分隔符** 。

第一个@符号后面是ROS消息类型。

ROS消息类型后面是@、\[或\]符号：

*   **@**  表示双向桥接；

*   **\[**  表示从Ignition Gazebo到ROS的桥接；

*   **\]**  表示从ROS到Ignition Gazebo的桥接。

方向符号后是Gazebo Transport消息类型。

（两个@不是同一个含义）

在服务Service中， **第一个@** 符号是服务名称和类型的 **分隔符** 。

第一个@符号后面是ROS服务类型。可以选择地包括Gazebo请求和响应类型，在它们之间用@符号分隔。

**仅** 支持将Gazebo服务公开为ROS服务，即ROS服务将请求转发到Gazebo服务，然后将响应转发回ROS客户端。

双向桥接示例：

```bash
parameter_bridge /chatter@std_msgs/String@gz.msgs.StringMsg
```

从Gazebo到ROS的桥接示例：

```bash
parameter_bridge /chatter@std_msgs/String[gz.msgs.StringMsg
```

从ROS到Gazebo的桥接示例：

```bash
parameter_bridge /chatter@std_msgs/String]gz.msgs.StringMsg
```

服务桥接示例：

```bash
parameter_bridge /world/default/control@ros_gz_interfaces/srv/ControlWorld
或者：
parameter_bridge /world/default/control@ros_gz_interfaces/srv/ControlWorld@gz.msgs.WorldControl@gz.msgs.Boolean
```

也可以运行`ros2 run ros_gz_bridge parameter_bridge -h`指令查看官方说明文档。

**ros\_gz\_bridge支持的消息类型**

以下是ROS2与Ignition Gazebo中话题消息类型对应表：

| ROS2消息类型 | Gazebo Transport 类型 |
|:---|:---|
| builtin_interfaces/msg/Time | gz.msgs.Time |
| geometry_msgs/msg/Point | gz.msgs.Vector3d |
| geometry_msgs/msg/Pose | gz.msgs.Pose |
| geometry_msgs/msg/PoseArray | gz.msgs.Pose_V |
| geometry_msgs/msg/PoseStamped | gz.msgs.Pose |
| geometry_msgs/msg/PoseWithCovariance | gz.msgs.PoseWithCovariance |
| geometry_msgs/msg/Quaternion | gz.msgs.Quaternion |
| geometry_msgs/msg/Transform | gz.msgs.Pose |
| geometry_msgs/msg/TransformStamped | gz.msgs.Pose |
| geometry_msgs/msg/Twist | gz.msgs.Twist |
| geometry_msgs/msg/TwistWithCovariance | gz.msgs.TwistWithCovariance |
| geometry_msgs/msg/TwistWithCovarianceStamped | gz.msgs.TwistWithCovariance |
| geometry_msgs/msg/Vector3 | gz.msgs.Vector3d |
| geometry_msgs/msg/Wrench | gz.msgs.Wrench |
| geometry_msgs/msg/WrenchStamped | gz.msgs.Wrench |
| nav_msgs/msg/Odometry | gz.msgs.Odometry |
| nav_msgs/msg/Odometry | gz.msgs.OdometryWithCovariance |
| rcl_interfaces/msg/ParameterValue | gz.msgs.Any |
| ros_gz_interfaces/msg/Altimeter | gz.msgs.Altimeter |
| ros_gz_interfaces/msg/Contact | gz.msgs.Contact |
| ros_gz_interfaces/msg/Contacts | gz.msgs.Contacts |
| ros_gz_interfaces/msg/Dataframe | gz.msgs.Dataframe |
| ros_gz_interfaces/msg/Entity | gz.msgs.Entity |
| ros_gz_interfaces/msg/Float32Array | gz.msgs.Float_V |
| ros_gz_interfaces/msg/GuiCamera | gz.msgs.GUICamera |
| ros_gz_interfaces/msg/JointWrench | gz.msgs.JointWrench |
| ros_gz_interfaces/msg/Light | gz.msgs.Light |
| ros_gz_interfaces/msg/SensorNoise | gz.msgs.SensorNoise |
| ros_gz_interfaces/msg/StringVec | gz.msgs.StringMsg_V |
| ros_gz_interfaces/msg/TrackVisual | gz.msgs.TrackVisual |
| ros_gz_interfaces/msg/VideoRecord | gz.msgs.VideoRecord |
| ros_gz_interfaces/msg/WorldControl | gz.msgs.WorldControl |
| rosgraph_msgs/msg/Clock* | gz.msgs.Clock* |
| sensor_msgs/msg/BatteryState | gz.msgs.BatteryState |
| sensor_msgs/msg/CameraInfo | gz.msgs.CameraInfo |
| sensor_msgs/msg/FluidPressure | gz.msgs.FluidPressure |
| sensor_msgs/msg/Image | gz.msgs.Image |
| sensor_msgs/msg/Imu | gz.msgs.IMU |
| sensor_msgs/msg/JointState | gz.msgs.Model |
| sensor_msgs/msg/Joy | gz.msgs.Joy |
| sensor_msgs/msg/LaserScan | gz.msgs.LaserScan |
| sensor_msgs/msg/MagneticField | gz.msgs.Magnetometer |
| sensor_msgs/msg/NavSatFix | gz.msgs.NavSat |
| sensor_msgs/msg/PointCloud2 | gz.msgs.PointCloudPacked |
| std_msgs/msg/Bool | gz.msgs.Boolean |
| std_msgs/msg/ColorRGBA | gz.msgs.Color |
| std_msgs/msg/Empty | gz.msgs.Empty |
| std_msgs/msg/Float32 | gz.msgs.Float |
| std_msgs/msg/Float64 | gz.msgs.Double |
| std_msgs/msg/Header | gz.msgs.Header |
| std_msgs/msg/Int32 | gz.msgs.Int32 |
| std_msgs/msg/String | gz.msgs.StringMsg |
| std_msgs/msg/UInt32 | gz.msgs.UInt32 |
| tf2_msgs/msg/TFMessage | gz.msgs.Pose_V |
| trajectory_msgs/msg/JointTrajectory | gz.msgs.JointTrajectory |

以及服务消息类型对应表：

| ROS2消息类型 | Gazebo 请求 | Gazebo 响应 |
|:---|:---|:---|
| ros_gz_interfaces/srv/ControlWorld | gz.msgs.WorldControl | gz.msgs.Boolean |

#### 与ROS2集成优化
在 **Ignition Gazebo与ROS2集成** 实现中需要在终端中使用不同的指令启动不同模块，该流程实现稍显复杂，本节将介绍如何以launch文件的方式进行优化。

**新建功能包**

请首先调用如下指令创建一个功能包：

```bash
ros2 pkg create demo_gazebo_sim
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1721.webp)

**添加目录**

在新建的功能包下添加目录： launch、rviz、world。并在CmakeLists.txt中添加如下代码：

```cmake
install(DIRECTORY rviz world launch DESTINATION share/${PROJECT_NAME})
```

launch目录用于存储launch文件，rviz目录由于存储rviz2的配置文件，而world目录则用于存储Ignition Gazebo仿真环境的相关文件。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1722.webp)

**rviz目录中生成rviz2的配置文件**

启动 rviz2，直接将默认配置保存至当前功能包的rviz目录，保存文件命名为sim.rviz。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1723.webp)

**复制world文件**

在ignition安装路径下的worlds目录（`/usr/share/ignition/ignition-gazebo6/worlds`）中复制visualize\_lidar.sdf文件至world目录。

如果该路径下没有，那可能在ROS的安装路径下：

`/opt/ros/jazzy/opt/gz_sim_vendor/share/gz/gz-sim8/worlds/`

如果还没有的话，手动查找一下：

```bash
sudo find / -name "visualize_lidar.sdf"
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1724.webp)

**编写launch文件**

launch目录下新建launch文件gazebo\_sim\_demo.launch.py，并输入如下内容：

```python
import os

from ament_index_python.packages import get_package_share_directory

from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.actions import IncludeLaunchDescription
from launch.conditions import IfCondition
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration

from launch_ros.actions import Node

def generate_launch_description():

    this_pkg = get_package_share_directory('demo_gazebo_sim')
    pkg_ros_gz_sim = get_package_share_directory('ros_gz_sim')
    world_file = os.path.join(this_pkg,'world','visualize_lidar.sdf')

    gz_sim = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            os.path.join(pkg_ros_gz_sim, 'launch', 'gz_sim.launch.py')),
        launch_arguments={
            'gz_args': '-r ' + world_file
        }.items(),
    )

    # RViz
    rviz = Node(
       package='rviz2',
       executable='rviz2',
       arguments=['-d', os.path.join(this_pkg, 'rviz', 'sim.rviz')],
       condition=IfCondition(LaunchConfiguration('rviz'))
    )

    # Bridge
    bridge = Node(
        package='ros_gz_bridge',
        executable='parameter_bridge',
        arguments=['/model/vehicle_blue/cmd_vel@geometry_msgs/msg/Twist@gz.msgs.Twist',
                   '/model/vehicle_blue/odometry@nav_msgs/msg/Odometry@gz.msgs.Odometry',
                   '/model/vehicle_blue/tf@tf2_msgs/msg/TFMessage[gz.msgs.Pose_V',
                   ],
        parameters=[{'qos_overrides./model/vehicle_blue.subscriber.reliability': 'reliable'}],
        remappings=[
                ('/model/vehicle_blue/tf', '/tf'),
                ('/model/vehicle_blue/cmd_vel','cmd_vel')
            ],
        output='screen'
    )

    return LaunchDescription([
        gz_sim,
        DeclareLaunchArgument('rviz', default_value='true',
                              description='Open RViz.'),
        bridge,
        rviz
    ])
```

该launch文件中，启动了Ignition Gazebo仿真环境、通过ros\_gz\_bridge建立了仿真与ROS2的连接，并且启动了rviz2节点。其中建立连接时，实现了速度指令、里程计以及坐标变换等消息的转换。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1725.webp)

**构建**

终端中进入当前工作空间，编译功能包：

```bash
colcon build  --packages-select demo_gazebo_sim
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1726.webp)

**执行**

终端中进入当前工作空间，调用如下指令执行launch文件：

```bash
. install/setup.bash
ros2 launch demo_gazebo_sim gazebo_sim_demo.launch.py
```

新开终端，启动键盘控制节点：

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

再配置rviz2，将`Fixed Frame`设置为`vehicle_blue/odom`，添加TF插件，添加Odometry插件并将话题设置为`/model/vehicle_blue/odometry`，当通过键盘控制发送速度指令时，仿真环境的机器人开始运动，并且在rviz2中可以回显坐标变换以及里程计等消息。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1727.webp)

#### 仿真环境创建 **SDF文件**
前面几节内容我们使用的是Ignition Gazebo内置的仿真环境，本节开始将介绍如何自行搭建仿真环境。本节案例将仿真一个长10m宽5m的矩形房间。该案例可以先启动Ignition Gazebo以拖拽的方式搭建仿真环境，然后再修改仿真环境对应的文件以调整细节。

**SDF、URDF 和 Xacro 的关系：**

*   **URDF 和 SDF 的区别：**

    *   **复杂性：** SDF 支持的功能更强大，能够描述完整的仿真环境；URDF 更适合定义机器人模型。

    *   **用途：** URDF 是 ROS 的标准；SDF 是 Gazebo 的标准。

    *   **物理引擎支持：** URDF 通过插件支持 Gazebo；SDF 原生支持 Gazebo。

    *   **格式转换：** URDF 可以转换为 SDF（通过 ROS 提供的工具`gz sdf -p`）。

*   **Xacro 的作用：**

    *   Xacro 是 URDF 的生成工具，帮助用户高效编写 URDF 文件，但它与 SDF 无直接关系。

* * *

**实践建议**

*   **在 Gazebo 仿真中：** 如果你用的是 ROS 2 和 Gazebo，可以直接使用 SDF 文件，功能更强大。

*   **在 ROS 中：** 如果主要用于机器人控制和规划，推荐使用 URDF 或由 Xacro 生成的 URDF。

*   **两者结合：** 使用 URDF 进行控制，使用 SDF 进行仿真。例如，使用 URDF 定义机器人结构后，借助 Gazebo 插件将其转换为 SDF。

**示例对比**

URDF 示例：

```xml
<robot name="example_robot">
  <link name="base_link">
    <inertial>
      <mass value="1.0" />
      <inertia ixx="1.0" ixy="0.0" ixz="0.0" iyy="1.0" iyz="0.0" izz="1.0" />
    </inertial>
  </link>
</robot>

```

Xacro 示例（生成 URDF）：

```xml
<xacro:robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="example_robot">
  <xacro:macro name="base_link" params="mass">
    <link name="base_link">
      <inertial>
        <mass value="${mass}" />
        <inertia ixx="1.0" ixy="0.0" ixz="0.0" iyy="1.0" iyz="0.0" izz="1.0" />
      </inertial>
    </link>
  </xacro:macro>

  <xacro:base_link mass="1.0" />
</xacro:robot>
```

SDF 示例：

```xml
<sdf version="1.6">
  <model name="example_robot">
    <link name="base_link">
      <inertial>
        <mass>1.0</mass>
        <inertia>
          <ixx>1.0</ixx>
          <iyy>1.0</iyy>
          <izz>1.0</izz>
        </inertia>
      </inertial>
    </link>
  </model>
</sdf>

```

**1.创建sdf文件**

首先请调用指令`ign gazebo`启动Ignition Gazebo，选择Empty仿真环境，然后添加立方体，每一个立方体都对应一堵墙。

上下左右立方体box、box\_1、box\_2、box\_3对应的坐标分别为(5.0,0.0,0.5)、(-5.0,0.0,0.5)、(0.0,2.5,0.5)、(0.0,-2.5,0.5)。

（以上坐标是指X，Y，Z坐标，没有旋转度）

保存文件到功能包的world目录下，保存的文件名称需要以.sdf为后缀，此处文件名为house.sdf。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1728.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1729.webp)

```xml
<sdf version='1.9'>
  <world name='empty'>
    <physics name='1ms' type='ignored'>
      <max_step_size>0.001</max_step_size>
      <real_time_factor>1</real_time_factor>
      <real_time_update_rate>1000</real_time_update_rate>
    </physics>
    <plugin name='gz::sim::systems::Physics' filename='ignition-gazebo-physics-system'/>
    <plugin name='gz::sim::systems::UserCommands' filename='ignition-gazebo-user-commands-system'/>
    <plugin name='gz::sim::systems::SceneBroadcaster' filename='ignition-gazebo-scene-broadcaster-system'/>
    <plugin name='gz::sim::systems::Contact' filename='ignition-gazebo-contact-system'/>
    <gravity>0 0 -9.8</gravity>
    <magnetic_field>6e-06 2.3e-05 -4.2e-05</magnetic_field>
    <atmosphere type='adiabatic'/>
    <scene>
      <ambient>0.4 0.4 0.4 1</ambient>
      <background>0.7 0.7 0.7 1</background>
      <shadows>true</shadows>
    </scene>
    <model name='ground_plane'>
      <static>true</static>
      <link name='link'>
        <collision name='collision'>
          <geometry>
            <plane>
              <normal>0 0 1</normal>
              <size>100 100</size>
            </plane>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='visual'>
          <geometry>
            <plane>
              <normal>0 0 1</normal>
              <size>100 100</size>
            </plane>
          </geometry>
          <material>
            <ambient>0.8 0.8 0.8 1</ambient>
            <diffuse>0.8 0.8 0.8 1</diffuse>
            <specular>0.8 0.8 0.8 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <inertial>
          <pose>0 0 0 0 -0 0</pose>
          <mass>100</mass>
          <inertia>
            <ixx>1</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>1</iyy>
            <iyz>0</iyz>
            <izz>1</izz>
          </inertia>
        </inertial>
        <enable_wind>false</enable_wind>
      </link>
      <pose>0 0 0 0 -0 0</pose>
      <self_collide>false</self_collide>
    </model>
    <model name='box'>
      <pose>5.0 0 0.5 -0 0 0</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <model name='box_0'>
      <pose>-5.0 -0 0.50000 -0 0 0</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <model name='box_1'>
      <pose>-0 -2.5 0.5 -0 -0 -0</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <model name='box_2'>
      <pose>-0 2.5 0.5 0 -0 -0</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <light name='sun' type='directional'>
      <pose>0 0 10 0 -0 0</pose>
      <cast_shadows>true</cast_shadows>
      <intensity>1</intensity>
      <direction>-0.5 0.1 -0.9</direction>
      <diffuse>0.8 0.8 0.8 1</diffuse>
      <specular>0.2 0.2 0.2 1</specular>
      <attenuation>
        <range>1000</range>
        <linear>0.01</linear>
        <constant>0.90000000000000002</constant>
        <quadratic>0.001</quadratic>
      </attenuation>
      <spot>
        <inner_angle>0</inner_angle>
        <outer_angle>0</outer_angle>
        <falloff>0</falloff>
      </spot>
    </light>
  </world>
</sdf>
```

**2.修改sdf文件**

修改sdf文件，调整立方体的尺寸，实现墙体的合围。在sdf文件中，四个立方体分别对应了四个`<model>`标签，其`name`属性分别为`box`、`box_1`、`box_2`、`box_3`，将`box`和`box_1`中的`<size>1 1 1</size>`修改为`<size>0.1 5 1</size>`，将`box_2`和`box_3`中的`<size>1 1 1</size>`修改为`<size>10 0.1 1</size>`（*注意：每个*`<model>`*标签下，都包含两个*`<size>`*标签，分别位于*`<collision>`*标签和*`<visual>`*标签下，两个*`<size>`*标签内容都需要修改*）。

修改后与的house.sdf文件内容如下：

```xml
<sdf version='1.9'>
  <world name='empty'>
    <physics name='1ms' type='ignored'>
      <max_step_size>0.001</max_step_size>
      <real_time_factor>1</real_time_factor>
      <real_time_update_rate>1000</real_time_update_rate>
    </physics>
    <plugin name='ign::gazebo::systems::Physics' filename='ignition-gazebo-physics-system'/>
    <plugin name='ign::gazebo::systems::UserCommands' filename='ignition-gazebo-user-commands-system'/>
    <plugin name='ign::gazebo::systems::SceneBroadcaster' filename='ignition-gazebo-scene-broadcaster-system'/>
    <plugin name='ign::gazebo::systems::Contact' filename='ignition-gazebo-contact-system'/>
    <gravity>0 0 -9.8</gravity>
    <magnetic_field>6e-06 2.3e-05 -4.2e-05</magnetic_field>
    <atmosphere type='adiabatic'/>
    <scene>
      <ambient>0.4 0.4 0.4 1</ambient>
      <background>0.7 0.7 0.7 1</background>
      <shadows>true</shadows>
    </scene>
    <model name='ground_plane'>
      <static>true</static>
      <link name='link'>
        <collision name='collision'>
          <geometry>
            <plane>
              <normal>0 0 1</normal>
              <size>100 100</size>
            </plane>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='visual'>
          <geometry>
            <plane>
              <normal>0 0 1</normal>
              <size>100 100</size>
            </plane>
          </geometry>
          <material>
            <ambient>0.8 0.8 0.8 1</ambient>
            <diffuse>0.8 0.8 0.8 1</diffuse>
            <specular>0.8 0.8 0.8 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <inertial>
          <pose>0 0 0 0 -0 0</pose>
          <mass>100</mass>
          <inertia>
            <ixx>1</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>1</iyy>
            <iyz>0</iyz>
            <izz>1</izz>
          </inertia>
        </inertial>
        <enable_wind>false</enable_wind>
      </link>
      <pose>0 0 0 0 -0 0</pose>
      <self_collide>false</self_collide>
    </model>
    <model name='box'>
      <pose>5.0 0 0.5 -0 0 0</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <model name='box_0'>
      <pose>-5.0 -0 0.50000 -0 0 0</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <model name='box_1'>
      <pose>-0 -2.5 0.5 -0 -0 -0</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <model name='box_2'>
      <pose>-0 2.5 0.5 0 -0 -0</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <light name='sun' type='directional'>
      <pose>0 0 10 0 -0 0</pose>
      <cast_shadows>true</cast_shadows>
      <intensity>1</intensity>
      <direction>-0.5 0.1 -0.9</direction>
      <diffuse>0.8 0.8 0.8 1</diffuse>
      <specular>0.2 0.2 0.2 1</specular>
      <attenuation>
        <range>1000</range>
        <linear>0.01</linear>
        <constant>0.90000000000000002</constant>
        <quadratic>0.001</quadratic>
      </attenuation>
      <spot>
        <inner_angle>0</inner_angle>
        <outer_angle>0</outer_angle>
        <falloff>0</falloff>
      </spot>
    </light>
  </world>
</sdf>
```

**3.编写launch文件**

在launch目录下新建launch文件gazebo\_sim\_world.launch.py，并输入如下内容：

```python
import os

from ament_index_python.packages import get_package_share_directory

from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource

def generate_launch_description():

    this_pkg = get_package_share_directory('demo_gazebo_sim')
    pkg_ros_gz_sim = get_package_share_directory('ros_gz_sim')
    world_file = os.path.join(this_pkg,"world","house.sdf")

    gz_sim = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            os.path.join(pkg_ros_gz_sim, 'launch', 'gz_sim.launch.py')),
        launch_arguments={
            'gz_args': '-r ' + world_file
        }.items(),
    )
    return LaunchDescription([
        gz_sim
    ])
```

**4.构建**

终端中进入当前工作空间，编译功能包：

```bash
colcon build  --packages-select demo_gazebo_sim
```

**5.执行**

终端中进入当前工作空间，调用如下指令执行launch文件：

```bash
. install/setup.bash
ros2 launch demo_gazebo_sim gazebo_sim_world.launch.py
```

运行结果如下图所示。

也可以根据个人喜好，继续设计房间模型。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1730.webp)

#### **IgnG添加模型**
在Ignition Gazebo官网提供了许多仿真模型，可以自行下载并使用以优化仿真环境，使其更多样、美观且真实。

**资源下载**

仿真Ignition Gazebo的官方模型链接：

http://app.ignitionrobotics.org/fuel/models

自行选择仿真模型点击进入详情页面，然后点击下载按钮即可将模型资源保存到本地。

在用户目录下新建ign\_models目录，将下载的资源解压缩到该目录以作备用。

**资源配置**

为了可以让Ignition Gazebo识别到模型资源，下一步还需要修改用户目录下的 .bashrc 文件，添加如下代码：

```bash

# Humble版本一般是下面的，但是有可能会更新，如果不生效，请尝试Jazzy的宏
export IGN_GAZEBO_RESOURCE_PATH=~/ign_models

# Jazzy版本的宏改了，如下：
export GZ_SIM_RESOURCE_PATH=~/ign_models
```

https://gazebosim.org/docs/latest/fuel\_insert/

**模型添加**

终端下进入功能包demo\_gazebo\_sim的world目录，使用指令`ign gazebo house.sdf` 或者`gz sim house.sdf`启动仿真环境，点击窗口右上的折叠按钮，搜索`Resource Spawner`并打开，点击`Local resources`并选择模型拖拽至仿真环境中。将修改后的内容保存至house.sdf文件。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1731.webp)

正常下载资源后，这个local resources这里就会显示了

house.sdf文件示例内容如下：

```xml
<sdf version='1.9'>
  <world name='empty'>
    <physics name='1ms' type='ignored'>
      <max_step_size>0.001</max_step_size>
      <real_time_factor>1</real_time_factor>
      <real_time_update_rate>1000</real_time_update_rate>
    </physics>
    <plugin name='gz::sim::systems::Physics' filename='ignition-gazebo-physics-system'/>
    <plugin name='gz::sim::systems::UserCommands' filename='ignition-gazebo-user-commands-system'/>
    <plugin name='gz::sim::systems::SceneBroadcaster' filename='ignition-gazebo-scene-broadcaster-system'/>
    <plugin name='gz::sim::systems::Contact' filename='ignition-gazebo-contact-system'/>
    <gravity>0 0 -9.8</gravity>
    <magnetic_field>6e-06 2.3e-05 -4.2e-05</magnetic_field>
    <atmosphere type='adiabatic'/>
    <scene>
      <ambient>0.4 0.4 0.4 1</ambient>
      <background>0.7 0.7 0.7 1</background>
      <shadows>true</shadows>
    </scene>
    <model name='ground_plane'>
      <static>true</static>
      <link name='link'>
        <collision name='collision'>
          <geometry>
            <plane>
              <normal>0 0 1</normal>
              <size>100 100</size>
            </plane>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='visual'>
          <geometry>
            <plane>
              <normal>0 0 1</normal>
              <size>100 100</size>
            </plane>
          </geometry>
          <material>
            <ambient>0.8 0.8 0.8 1</ambient>
            <diffuse>0.8 0.8 0.8 1</diffuse>
            <specular>0.8 0.8 0.8 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <inertial>
          <pose>0 0 0 0 -0 0</pose>
          <mass>100</mass>
          <inertia>
            <ixx>1</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>1</iyy>
            <iyz>0</iyz>
            <izz>1</izz>
          </inertia>
        </inertial>
        <enable_wind>false</enable_wind>
      </link>
      <pose>0 0 0 0 -0 0</pose>
      <self_collide>false</self_collide>
    </model>
    <model name='box'>
      <pose>5.02632 -2e-06 0.500002 -0 4.4e-05 4.6e-05</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <model name='box_0'>
      <pose>-5.01336 -0.00029 0.500002 0 -4.2e-05 -0.005335</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>0.1 5 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <model name='box_1'>
      <pose>-0 -2.5 0.5 1e-06 0 0</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <model name='box_2'>
      <pose>-0.000154 2.52488 0.500821 -0.018068 -0 -0.003156</pose>
      <link name='box_link'>
        <inertial>
          <inertia>
            <ixx>16.666</ixx>
            <ixy>0</ixy>
            <ixz>0</ixz>
            <iyy>16.666</iyy>
            <iyz>0</iyz>
            <izz>16.666</izz>
          </inertia>
          <mass>100</mass>
          <pose>0 0 0 0 -0 0</pose>
        </inertial>
        <collision name='box_collision'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <surface>
            <friction>
              <ode/>
            </friction>
            <bounce/>
            <contact/>
          </surface>
        </collision>
        <visual name='box_visual'>
          <geometry>
            <box>
              <size>10 0.1 1</size>
            </box>
          </geometry>
          <material>
            <ambient>0.3 0.3 0.3 1</ambient>
            <diffuse>0.7 0.7 0.7 1</diffuse>
            <specular>1 1 1 1</specular>
          </material>
        </visual>
        <pose>0 0 0 0 -0 0</pose>
        <enable_wind>false</enable_wind>
      </link>
      <static>false</static>
      <self_collide>false</self_collide>
    </model>
    <include>
      <uri>file://Bed</uri>
      <name>Bed</name>
      <pose>2.82155 1.18752 0 0 -0 0</pose>
    </include>
    <include>
      <uri>file://Office Desk</uri>
      <name>Desk</name>
      <pose>2.78306 -1.97796 0 0 -0 1.57</pose>
    </include>
    <include>
      <uri>file://Bathtub</uri>
      <name>Bathtub</name>
      <pose>-3.87509 1.82783 0 0 -0 0</pose>
    </include>
    <include>
      <uri>file://Vanity</uri>
      <name>Vanity</name>
      <pose>-2.5974 1.85613 -0.010992 0.021648 0 -1.57</pose>
    </include>
    <include>
      <uri>file://Vanity</uri>
      <name>Vanity_1</name>
      <pose>-2.5974 0.634325 -0.010992 0.021648 0 -1.57</pose>
    </include>
    <include>
      <uri>file://Dining Table</uri>
      <name>DiningTable</name>
      <pose>-0.374337 1.33602 0 0 0 -1.57</pose>
    </include>
    <include>
      <uri>file://Chair</uri>
      <name>Chair</name>
      <pose>2.79762 -1.26474 -0 -0 0 -2.3062</pose>
    </include>
    <include>
      <uri>file://Sofa</uri>
      <name>Sofa</name>
      <pose>-0.546136 -1.92328 0.000119 -0 0 1.57</pose>
    </include>
    <light name='sun' type='directional'>
      <pose>0 0 10 0 -0 0</pose>
      <cast_shadows>true</cast_shadows>
      <intensity>1</intensity>
      <direction>-0.5 0.1 -0.9</direction>
      <diffuse>0.8 0.8 0.8 1</diffuse>
      <specular>0.2 0.2 0.2 1</specular>
      <attenuation>
        <range>1000</range>
        <linear>0.01</linear>
        <constant>0.90000000000000002</constant>
        <quadratic>0.001</quadratic>
      </attenuation>
      <spot>
        <inner_angle>0</inner_angle>
        <outer_angle>0</outer_angle>
        <falloff>0</falloff>
      </spot>
    </light>
  </world>
</sdf>
```

**构建**

终端中进入当前工作空间，编译功能包：

```bash
colcon build  --packages-select demo_gazebo_sim
```

**执行**

终端中进入当前工作空间，调用如下指令执行launch文件：

```bash
. install/setup.bash
ros2 launch demo_gazebo_sim gazebo_sim_world.launch.py
```

运行结果如下图所示。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1732.webp)

#### IgnG添加机器人
Ignition Gazebo中可以直接创建机器人模型，或者也可以加载ROS2中URDF格式的机器人模型，此处我们使用后者（也可以选择用自己的urdf小车，但是注意修改launch的路径）。

我没有用赵虚左老师的mycar\_description，用的自己的小车模型，后面的一系列源码都在下方这个Github仓库中，需要的可以自行clone.

https://github.com/tungchiahui/ROS2\_WS/tree/main/6.ws\_simulations

**准备机器人模型功能包**

复制机器人描述功能包mycar\_description到工作空间的src目录，ign\_models中新建mycar\_description目录，并将功能包mycar\_description下的mesh目录复制进ign\_models中的mycar\_description目录。

**机器人模型功能包下新建launch文件**

新建launch文件mycar\_desc\_sim.launch.py，并输入如下内容：

```python
from launch import LaunchDescription
from launch_ros.actions import Node
import os
from ament_index_python.packages import get_package_share_directory
from launch_ros.parameter_descriptions import ParameterValue
from launch.substitutions import Command,LaunchConfiguration
from launch.actions import DeclareLaunchArgument

#示例： ros2 launch cpp06_urdf display.launch.py model:=`ros2 pkg prefix --share cpp06_urdf`/urdf/urdf/demo01_helloworld.urdf
def generate_launch_description():

    MYCAR_MODEL = os.environ['MYCAR_MODEL']

    mycar_description = get_package_share_directory("mycar_description")
    default_model_path = os.path.join(mycar_description,"urdf",MYCAR_MODEL + ".urdf")
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

较之于以往该文件缺少了joint\_state\_publisher节点，该节点作用是发布活动关节状态，这一功能后续由ignition实现。

**添加机器人模型**

修改gazebo\_sim\_world.launch.py文件，包含机器人模型的发布文件并在Gazebo中生成机器人模型，修改后的代码如下：

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

# MYCAR_MODEL值可以设置为arduino、stm32_2w 或stm32_4w（这个是具体的urdf文件名，在mycar_description包下的）
export MYCAR_MODEL=stm32_4w
ros2 launch demo_gazebo_sim gazebo_sim_world.launch.py
```

运行结果如下图所示。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1733.webp)

####    IgnG运动控制器(实质上就是<gazebo>标签)
本节将介绍如何让你的机器人动起来。

原理就是给urdf或xacro等添加<gazebo>标签：

http://sdformat.org/tutorials?tut=sdformat\_urdf\_extensions&cat=specification&

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1734.webp)

https://gazebosim.org/api/plugin/2/index.html

安装库：

https://gazebosim.org/api/plugin/2/installation.html

```bash
sudo apt-get update
sudo apt-get install lsb-release
sudo sh -c 'echo "deb http://packages.osrfoundation.org/gazebo/ubuntu-stable `lsb_release -cs` main" > /etc/apt/sources.list.d/gazebo-stable.list'
wget http://packages.osrfoundation.org/gazebo.key -O - | sudo apt-key add -
sudo apt-get update
sudo apt install libgz-plugin2-dev
```

利用插件去让小车动，比如有两轮差速插件，四轮麦轮插件等等

同时该插件还提供了一些可以控制输出的选项，因为是仿真，所以还要告诉插件轮子对应的joint名称等信息，这样就有了下面这个参数表格：

| 配置项 | 含义 |
|:---|:---|
| ros | ros相关配置，包含命名空间和话题重映射等 |
| update_rate | 数据更新速率 |
| left_joint | 左轮关节名称 |
| right_joint | 右轮关节名称 |
| wheel_separation | 左右轮子的间距 |
| wheel_diameter | 轮子的直径 |
| max_wheel_torque | 轮子最大的力矩 |
| max_wheel_acceleration | 轮子最大的加速度 |
| publish_odom | 是否发布里程计 |
| publish_odom_tf | 是否发布里程计的tf开关 |
| publish_wheel_tf | 是否发布轮子的tf数据开关 |
| odometry_frame | 里程计的framed ID，最终体现在话题和TF上 |
| robot_base_frame | 机器人的基础frame的ID |

**修改URDF文件**

arduino.urdf 和 stm32\_2w.urdf 文件中，在`<robot>`根标签下添加如下代码：

```xml

  <gazebo>
      <plugin filename="libignition-gazebo-diff-drive-system.so"
        name="ignition::gazebo::systems::DiffDrive">
        <left_joint>left_joint</left_joint>
        <right_joint>right_joint</right_joint>
        <wheel_separation>0.2097</wheel_separation>
        <wheel_radius>0.03415</wheel_radius>
        <odom_publish_frequency>10</odom_publish_frequency>
        <frame_id>odom</frame_id>
        <child_frame_id>base_footprint</child_frame_id>
        <topic>/cmd_vel</topic>
        <max_linear_acceleration>10</max_linear_acceleration>
        <min_linear_acceleration>-10</min_linear_acceleration>
        <max_angular_acceleration>10</max_angular_acceleration>
        <min_angular_acceleration>-10</min_angular_acceleration>
        <max_linear_velocity>0.5</max_linear_velocity>
        <min_linear_velocity>-0.5</min_linear_velocity>
        <max_angular_velocity>1</max_angular_velocity>
        <min_angular_velocity>-1</min_angular_velocity>
      </plugin>

  </gazebo>

  <gazebo>
    <plugin filename="ignition-gazebo-joint-state-publisher-system"
      name="ignition::gazebo::systems::JointStatePublisher">
    </plugin>
  </gazebo>
```

stm32\_4w.urdf 文件中，在`<robot>`根标签下添加如下代码：

```xml

<gazebo>
    <plugin
        filename="ignition-gazebo-diff-drive-system"
        name="ignition::gazebo::systems::DiffDrive">
        <left_joint>left_former_joint</left_joint>
        <left_joint>left_rear_joint</left_joint>
        <right_joint>right_former_joint</right_joint>
        <right_joint>right_rear_joint</right_joint>
        <wheel_separation>0.4</wheel_separation>
        <wheel_radius>0.0415</wheel_radius>
        <odom_publish_frequency>50</odom_publish_frequency>
        <frame_id>odom</frame_id>
        <child_frame_id>base_footprint</child_frame_id>
        <topic>/cmd_vel</topic>
        <max_linear_acceleration>10</max_linear_acceleration>
        <min_linear_acceleration>-10</min_linear_acceleration>
        <max_angular_acceleration>10</max_angular_acceleration>
        <min_angular_acceleration>-10</min_angular_acceleration>
        <max_linear_velocity>0.5</max_linear_velocity>
        <min_linear_velocity>-0.5</min_linear_velocity>
        <max_angular_velocity>1</max_angular_velocity>
        <min_angular_velocity>-1</min_angular_velocity>
      </plugin>
  </gazebo>

  <gazebo>
    <plugin filename="ignition-gazebo-joint-state-publisher-system"
      name="ignition::gazebo::systems::JointStatePublisher">
    </plugin>
  </gazebo>
```

如果是麦轮（ROS1的，ROS2的待更新）（把轮子关节设置为自己轮子关节名即可）：

```xml
<robot name="mycar" xmlns:xacro="http://wiki.ros.org/xacro">

    <xacro:macro name="joint_trans" params="joint_name">

        <transmission name="${joint_name}_trans">
            <type>transmission_interface/SimpleTransmission</type>
            <joint name="${joint_name}">
                <hardwareInterface>hardware_interface/VelocityJointInterface</hardwareInterface>
            </joint>
            <actuator name="${joint_name}_motor">
                <hardwareInterface>hardware_interface/VelocityJointInterface</hardwareInterface>
                <mechanicalReduction>1</mechanicalReduction>
            </actuator>
        </transmission>
    </xacro:macro>

    <xacro:joint_trans joint_name="LeftFrontwheelToBase" />
    <xacro:joint_trans joint_name="LeftBackwheelToBase" />
    <xacro:joint_trans joint_name="RightFrontwheelToBase" />
    <xacro:joint_trans joint_name="RightBackwheelToBase" />

    <gazebo>
        <plugin name="mecanum_controller" filename="libgazebo_ros_planar_move.so">
            <commandTopic>cmd_vel</commandTopic>
            <odometryTopic>odom</odometryTopic>
            <odometryFrame>odom</odometryFrame>
            <leftFrontJoint>LeftFrontwheelToBase</leftFrontJoint>
            <rightFrontJoint>RightFrontwheelToBase</rightFrontJoint>
            <leftRearJoint>LeftBackwheelToBase</leftRearJoint>
            <rightRearJoint>RightBackwheelToBase</rightRearJoint>
            <odometryRate>100</odometryRate>
            <robotBaseFrame>base_footprint</robotBaseFrame>
            <broadcastTF>1</broadcastTF>
        </plugin>
    </gazebo>

</robot>
```

**修改launch文件**

修改gazebo\_sim\_world.launch.py文件，修改后的代码如下：

```JavaScript
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
    world_file = os.path.join(this_pkg,"world","base.sdf")

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
                "-x", "-4",
                "-z", "0.01", #设置为0,可能会陷进地里
                "-y", "0",
                "-topic", "/robot_description"],
            output="screen")

    # Bridge
    bridge = Node(
        package="ros_gz_bridge",
        executable="parameter_bridge",
        arguments=["/cmd_vel@geometry_msgs/msg/Twist@gz.msgs.Twist",
                   "/model/mycar/odometry@nav_msgs/msg/Odometry@gz.msgs.Odometry",
                   "/model/mycar/tf@tf2_msgs/msg/TFMessage[gz.msgs.Pose_V",
                   "/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock",
                   "/world/empty/model/mycar/joint_state@sensor_msgs/msg/JointState[gz.msgs.Model",
                   ],
        parameters=[{"qos_overrides./model/mycar.subscriber.reliability": "reliable"}],
        remappings=[
                ("/model/mycar/tf", "/tf"),
                ("/world/empty/model/mycar/joint_state","joint_states"),
                ("/model/mycar/odometry","/odom")
            ],
        output="screen"
    )

    return LaunchDescription([
        gz_sim,
        spawn,
        mycar_desc,
        bridge
    ])
```

**构建**

终端中进入当前工作空间，编译功能包：

```bash
colcon build --packages-select mycar_description demo_gazebo_sim
```

**执行**

终端中进入当前工作空间，调用如下指令执行launch文件：

```bash
. install/setup.bash
export MYCAR_MODEL=stm32_4w # MYCAR_MODEL值可以设置为arduino、stm32_2w 或stm32_4w（这个是具体的urdf文件名，在mycar_description包下的）
ros2 launch demo_gazebo_sim gazebo_sim_world.launch.py
```

再启动键盘控制节点，就可以控制机器人运动了。

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

还可以启动rviz2，以查看里程计消息以及坐标变换。终端中进入当前工作空间，调用如下指令执行launch文件：

启动rviz2

```bash
. install/setup.bash
rviz2
```

RVIZ2软件配置如下图所示：

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1735.webp)![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1736.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1737.webp)

#### Ignition Gazebo仿真之传感器
本节将介绍如何为仿真机器人添加雷达、相机等传感器。本节代码部分内容对于我们教程中涉及到的arduino、stm32\_2w以及stm32\_4w等机器人模型而言是完全通用的。

**添加传感器插件**

在进行传感器模拟之前，需要先添加一个名为`ignition-gazebo-sensors-system`的插件，打开urdf文件，在`<robot>`根标签内添加如下代码：

```xml

<gazebo>
    <plugin
      filename="ignition-gazebo-sensors-system"
      name="ignition::gazebo::systems::Sensors">
      <render_engine>ogre2</render_engine>
    </plugin>
  </gazebo>
```

ignition-gazebo-sensors-system是Ignition Gazebo仿真环境的插件，提供传感器模型和相关功能，用于创建、模拟和测试各种传感器设备。它包含常见传感器模型，如摄像头、激光雷达等。

**添加各种传感器**

**(注意，你的模型一定要有以下几个传感器的模型)**

雷达的模型不需要collision，请删掉，否则会挡激光射出。

```xml

  <gazebo reference="laser">
    <sensor name='gpu_lidar' type='gpu_lidar'>
      <topic>scan</topic>
      <update_rate>30</update_rate>
      <lidar>
        <scan>
          <horizontal>
            <samples>640</samples>
            <resolution>1</resolution>
            <min_angle>-3.1415926</min_angle>
            <max_angle>3.1415926</max_angle>
          </horizontal>
          <vertical>
            <samples>16</samples>
            <resolution>1</resolution>
            <min_angle>-0.261799</min_angle>
            <max_angle>0.261799</max_angle>
          </vertical>
        </scan>
        <range>
          <min>0.08</min>
          <max>10.0</max>
          <resolution>0.01</resolution>
        </range>
      </lidar>
      <alwaysOn>1</alwaysOn>
      <visualize>true</visualize>
      <ignition_frame_id>laser</ignition_frame_id>
    </sensor>
  </gazebo>

  <gazebo reference="camera" >
    <sensor name="cam_link" type="camera">
      <update_rate>10.0</update_rate>
      <always_on>true</always_on>
      <ignition_frame_id>camera</ignition_frame_id>
      <pose>0 0 0 0 0 0</pose>
      <topic>/image_raw</topic>
      <camera name="my_camera">
        <horizontal_fov>1.3962634</horizontal_fov>
        <image>
           <width>600</width>
           <height>600</height>
           <format>R8G8B8</format>
        </image>
        <clip>
          <near>0.02</near>
          <far>300</far>
        </clip>
      </camera>
    </sensor>
  </gazebo>

  <gazebo reference="camera">
    <sensor name="depth_camera" type="depth_camera">
          <update_rate>10</update_rate>
          <topic>depth_camera</topic>
          <camera>
            <horizontal_fov>1.05</horizontal_fov>
            <image>
              <width>256</width>
              <height>256</height>
              <format>R_FLOAT32</format>
            </image>
            <clip>
              <near>0.1</near>
              <far>10.0</far>
            </clip>
          </camera>
          <alwaysOn>1</alwaysOn>
          <ignition_frame_id>camera</ignition_frame_id>
      </sensor>
  </gazebo>
```

从官网找到的imu传感器的

```xml

    <gazebo>
        <plugin filename="libignition-gazebo-imu-system.so"
                name="ignition::gazebo::systems::Imu">
        </plugin>
    </gazebo>

    <gazebo reference="base_link">
        <sensor name="imu_sensor" type="imu">
            <always_on>1</always_on>
            <update_rate>30</update_rate>
            <visualize>true</visualize>
            <topic>imu</topic>
        </sensor>
    </gazebo>
```

可以用`ign topic -e -t /imu`测试gazebo是否发布了话题，后面再用gazebo\_bridge把话题给ROS2就行了。

默认情况下，rviz2没有显示imu消息的插件，需要自行安装相关插件，具体安装指令如下：

```cpp
sudo apt install ros-${ROS_DISTRO}-imu-tools
```

SolidWorks自动生成的模型可能翻转了laser\_joint,请你修改回正，这样可能rivz2就有激光了，然后修改一下可视化的模型，让模型正常，不要给碰撞，不然可能会遮挡激光。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1738.webp)

修改gazebo\_sim\_world.launch.py文件，修改后的代码如下：

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
            "-x", "-4",
            "-z", "0.01", #设置为0,可能会陷进地里
            "-y", "0",
            "-topic", "/robot_description"],
        output="screen")

    # Bridge
    bridge = Node(
        package="ros_gz_bridge",
        executable="parameter_bridge",
        arguments=["/cmd_vel@geometry_msgs/msg/Twist@gz.msgs.Twist",
            "/model/mycar/odometry@nav_msgs/msg/Odometry@gz.msgs.Odometry",
            "/model/mycar/tf@tf2_msgs/msg/TFMessage[gz.msgs.Pose_V",
            "/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock",
            "/world/empty/model/mycar/joint_state@sensor_msgs/msg/JointState[gz.msgs.Model",
            "/scan@sensor_msgs/msg/LaserScan@gz.msgs.LaserScan",
            "/scan/points@sensor_msgs/msg/PointCloud2@gz.msgs.PointCloudPacked",
            "/image_raw@sensor_msgs/msg/Image@gz.msgs.Image",
            "/camera_info@sensor_msgs/msg/CameraInfo@gz.msgs.CameraInfo",
            "/depth_camera@sensor_msgs/msg/Image@gz.msgs.Image",
            "/imu@sensor_msgs/msg/Imu[gz.msgs.IMU",
            "/imu/angular_velocity@geometry_msgs/msg/Vector3[gz.msgs.Vector3d"
        ],
        parameters=[{"qos_overrides./model/mycar.subscriber.reliability": "reliable"}],
        remappings=[
            ("/model/mycar/tf", "/tf"),
            ("/world/empty/model/mycar/joint_state","joint_states"),
            ("/model/mycar/odometry","/odom")
        ],
        output="screen"
    )

    return LaunchDescription([
        gz_sim,
        spawn,
        mycar_desc,
        bridge
    ])
```

**构建**

终端中进入当前工作空间，编译功能包：

```bash
colcon build --packages-select mycar_description demo_gazebo_sim
```

**执行**

终端中进入当前工作空间，调用如下指令执行launch文件：

```bash
. install/setup.bash
export MYCAR_MODEL=stm32_4w # MYCAR_MODEL值可以设置为arduino、stm32_2w 或stm32_4w
ros2 launch demo_gazebo_sim gazebo_sim_world.launch.py
```

再启动键盘控制节点，就可以控制机器人运动了。

还可以启动rviz2，以查看机器人发布的诸多数据。终端中进入当前工作空间，调用如下指令执行launch文件：

```bash
. install/setup.bash
rviz2
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1739.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1740.webp)

我没有用赵虚左老师的mycar\_description，用的自己的小车模型，后面的一系列源码都在下方这个Github仓库中，需要的可以自行clone.

https://github.com/tungchiahui/ROS2\_WS/tree/main/6.ws\_simulations

(把上面的全部复现，才能够进行下一章导航，下一章导航依然基于仿真)


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



### 将Ign Gazebo迁移至Gz Sim
**（即ROS2Humble迁移至ROS2Jazzy及之后的版本，只用Humble的也要看一下）**

由于只有ROS2 Humble的Gazebo叫做ign gazebo，到了ROS2 Jazzy及之后的版本，应该会都改名叫gazebo（简称gz sim）。

如果你要把代码从ROS2 Humble迁移到ROS2 Jazzy，那么兼容性问题最大的地方就是Gazebo，需要修改很多东西。

但是Ign Gazebo与Gz Sim的最大区别其实就是重命名了一下，所以迁移很好迁移。

为何都是ROS2，且gazebo界面功能几乎都一模一样还代码不互通呢？

是因为gazebo组织早期在为ROS2开发新版Gazebo时，为了和Gazebo Classic做出区分，所以给新版Gazebo命名为Ignition Gazebo，并在ROS2 Humble上推行。

后来Gazebo组织发现这种命名方式及其麻烦，所以在新版ROS2 Jazzy上又修改回了Gazebo的名字(简称Gz Sim)。

所以并不是功能不互通，仅仅是名字不互通罢了，只需要重命名，就这么简单。

https://gazebosim.org/docs/harmonic/migration\_from\_ignition/

主要修改以下部分：

#### SDF文件
![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1741.webp)

根据官方所说，普通的要把ignition全部替换为gz.

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1742.webp)

插件要按照上面的修改，比如ignition::gazebo改为gz::sim。然后ignition::修改为gz::。

*   所有 `ignition-` 前缀 → `gz-`

*   所有 `ignition::gazebo` 命名空间 → `gz::sim`

*   插件系统名称 `ignition-gazebo-XXX-system` → `gz-sim-XXX-system`

*   新版本中`alwaysOn`已被always\_on替代，`gz_frame_id`改为pose relative\_to""

*   确保所有环境变量（如`IGN_GAZEBO_RESOURCE_PATH`）更新为`GZ_SIM_RESOURCE_PATH`

| 原字段 | 新字段 | 示例场景 |
|:---|:---|:---|
| ignition-gazebo-* | gz-sim-* | 插件文件名 |
| ignition::gazebo::* | gz::sim::* | 插件命名空间 |
| IGN_GAZEBO环境变量前缀 | GZ_SIM前缀 | 资源路径配置 |

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1743.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1744.webp)

具体代码看我的仓库：

https://github.com/CyberNaviRobot/CyberRobot\_ROS2\_Jazzy\_WS

### 本章小结
本章主要介绍了2D仿真工具stage\_ros2以及3D仿真工具ignition gazebo在ROS2中的应用。

相关知识点如下：

*   仿真的理论知识；

*   stage\_ros2在ROS2中的使用；

*   ignition gazebo在ROS2中的使用。

通过上述内容我们介绍了仿真在ROS2中的应用场景、概念，以及较之于实体机器人，仿真的优势和不足。还学习了如何基于stage\_ros2以及ignition gazebo分别搭建仿真环境并模拟机器人，为后续的学习奠定了基础。

另外，除了stage\_ros2和ignition gazebo之外，还有其他一些可用于ROS2仿真的软件。以下是一些常见的选择：

*   Webots: Webots是一个强大的开源多机器人仿真软件平台，它支持各种机器人硬件和传感器模拟，并且可以与ROS 2进行集成。

*   Gazebo Classic: Gazebo Classic是ROS 1中常用的仿真器，也可以通过ROS 2进行连接和使用。

*   MORSE: MORSE是一个基于Python的开源仿真器，可以用于ROS 2仿真和机器人开发，支持多种传感器和行为模型。

*   V-REP: V-REP是一个强大的多机器人仿真平台，它支持ROS 2与其进行通信，可以实现各种机器人模型和场景的仿真。

这些都是一些常见的ROS 2仿真软件，你可以根据具体的需求选择合适的软件来进行仿真和开发。请注意，某些仿真器可能需要进行适配和配置以与ROS 2兼容。
