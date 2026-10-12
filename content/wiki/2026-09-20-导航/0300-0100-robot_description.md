---
title: "robot_description（机器人模型与关节定义）"
---

## robot_description（机器人模型与关节定义）

本功能包复制于 `pb2025_robot_description`。为便于复用，将功能包目录命名为 `robot_description`，并将 `package.xml` 的 `<name>`、`CMakeLists.txt` 的 `project()` 和 launch 中的包查找名称同步改为 `robot_description`。

以下源码路径均相对于导航工作区 `/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry/ysu2026_rm_sentry_nav`。

将复制后的 `src/robot_description/resource/xmacro/pb2025_sentry_robot.sdf.xmacro` 重命名为 `reality_robot.sdf.xmacro`，保留 `simulation_robot.sdf.xmacro`，删掉本包中不再使用的 `pb2025_infantry_robot.sdf.xmacro`。上游 `pb2025_robot_description` 包保留作为参考。

之前本地包名为 `ysu2026_robot_description`，实车入口文件为 `ysu2026_sentry_robot.sdf.xmacro`；现在分别改为 `robot_description` 和 `reality_robot.sdf.xmacro`。当前相关目录如下：

```text
src/
├── robot_description/
│   ├── package.xml
│   ├── CMakeLists.txt
│   ├── README.md
│   ├── dependencies.repos
│   ├── launch/robot_description_launch.py
│   ├── params/robot_description.yaml
│   ├── env-hooks/gazebo.dsv.in
│   ├── rviz/visualize_robot.rviz
│   └── resource/
│       ├── xmacro/
│       │   ├── reality_robot.sdf.xmacro
│       │   └── simulation_robot.sdf.xmacro
│       └── models/
│           ├── mid360/
│           ├── industrial_camera/
│           └── rplidar_a2/
└── rmoss_gz_resources/
    └── resource/models/ysu2026_sentry_robot/
        ├── ysu2026_sentry_robot.def.xmacro
        ├── model.sdf.xmacro
        ├── model.config
        └── model.sdf
```

`robot_description` 是通用的描述加载与 TF 发布功能包；`reality_robot` 和 `simulation_robot` 分别是实车、仿真描述入口。底层 `ysu2026_sentry_robot.def.xmacro` 定义当前车型的底盘、轮子、云台和装甲板结构，保留车型名称。两份入口仍通过 include 和宏调用使用它。

入口名称通用化后，当前机械尺寸、雷达安装位置和关节状态来源仍是当前机器人的配置。换车时可以更换入口中的模型 include、宏调用与安装参数，也可以用 `robot_xmacro_file` 指定其他描述文件、用 `params_file` 指定对应节点配置。

### 扩展知识点

#### xmacro（机器人模型宏文件）

1. 变量基本完全对应

xacro

```xml
<xacro:property name="wheel_radius" value="0.076"/>

<cylinder radius="${wheel_radius}" length="0.03"/>
```

xmacro

```xml
<xmacro_define_value name="wheel_radius" value="0.076"/>

<cylinder>
    <radius>${wheel_radius}</radius>
    <length>0.03</length>
</cylinder>
```

2. 宏定义也几乎一一对应

xacro

```xml
<xacro:macro name="wheel" params="name radius width">
    <link name="${name}">
        ...
    </link>
</xacro:macro>
```

调用：

```xml
<xacro:wheel
    name="front_left_wheel"
    radius="0.076"
    width="0.03"
/>
```

xmacro

```xml
<xmacro_define_block
    name="wheel"
    params="name radius width">

    <link name="${name}">
        ...
    </link>

</xmacro_define_block>
```

调用：

```xml
<xmacro_block
    name="wheel"
    name_arg="front_left_wheel"
    radius="0.076"
    width="0.03"
/>
```

3. include

xacro

```xml
<xacro:include filename="wheel.xacro"/>
```

xmacro

```xml
<xmacro_include uri="file://wheel.sdf.xmacro"/>
```

在当前项目中还可以看到：

```xml
<xmacro_include
    uri="model://rm25_example_robot/rm25_example_robot.def.xmacro"
/>
```

这里的`model://`是 `sdformat_tools` 针对 Gazebo / SDF 做的 URI 解析支持。

4. ${} 表达式

xmacro 和 xacro 都支持类似：

```xml
${radius}
```

以及数学表达式：

```xml
${2 * radius}
```

例如：

```xml
<ixx>${m*(y*y+z*z)/12}</ixx>
```

所以可以在宏参数中进行简单数学运算。

5. 条件

xacro

```xml
<xacro:if value="${use_lidar}">
    ...
</xacro:if>
```

xmacro

xmacro 中通常可以通过宏调用的 condition 控制：

```xml
<xmacro_block
    name="lidar"
    condition="${use_lidar}"
    ...
/>
```

条件为真时展开这个宏，条件为假时不展开。

6. .def.xmacro

例如：

```text
rm25_example_robot.def.xmacro
```

这里的`.def`并不是 xmacro 的特殊语法，只是一种文件命名习惯。

通常表示：这个文件主要负责定义宏。

类似：

```text
xxx.def.xmacro
    ↓
定义机器人结构、link、joint 等宏
```

然后：

```text
simulation_robot.sdf.xmacro
    ↓
include 前面的 def.xmacro
    ↓
调用其中定义的宏
    ↓
生成完整 SDF
```

例如以后可以写：

```text
ysu2026_sentry_robot.def.xmacro
```

里面定义：

```xml
<xmacro_define_block
    name="ysu2026_sentry_robot"
    params="chassis_height gimbal_yaw_height">

    ...

</xmacro_define_block>
```

然后在：

```xml
simulation_robot.sdf.xmacro
```

中引入并调用：

```xml
<xmacro_include
    uri="model://ysu2026_sentry_robot/ysu2026_sentry_robot.def.xmacro"
/>

<xmacro_block
    name="ysu2026_sentry_robot"
    chassis_height="${chassis_height}"
    gimbal_yaw_height="${gimbal_yaw_height}"
/>
```

记忆：

```text
xmacro_define_value
    ≈ xacro:property

xmacro_define_block
    ≈ xacro:macro

xmacro_block
    ≈ 调用 xacro macro

xmacro_include
    ≈ xacro:include

${...}
    ≈ ${...}
```


### 总体框架

#### xmacro介绍（机器人模型宏文件）

这个xmacro是本项目里自定义的一种格式，为了实现：

```text
                  ┌→ SDF → Gazebo
.sdf.xmacro ──────┤
                  └→ SDF → URDF → robot_state_publisher
```

而之前学过的xacro+urdf,这个项目主要是xmacro+sdf。（对比着理解）

然后我们之前接触的都是

```text
robot.urdf.xacro
      ↓
    xacro
      ↓
robot_description
      ↓
robot_state_publisher
      ↓
TF
```

而这个项目是：

```text
simulation_robot.sdf.xmacro
          ↓
      xmacro4sdf
          ↓
         SDF
       ↙     ↘
   Gazebo    sdf2urdf
                 ↓
               URDF
                 ↓
      robot_state_publisher
                 ↓
                TF
```

| 功能 | xacro | xmacro |
|---|---|---|
| 定义变量 | `<xacro:property name="r" value="0.1"/>` | `<xmacro_define_value name="r" value="0.1"/>` |
| 使用变量 | `${r}` | `${r}` |
| 数学表达式 | `${2 * r}` | `${2 * r}` |
| 定义宏 | `<xacro:macro name="wheel" params="r w">` | `<xmacro_define_block name="wheel" params="r w">` |
| 调用宏 | `<xacro:wheel r="..." w="..."/>` | `<xmacro_block name="wheel" r="..." w="..."/>` |
| include | `<xacro:include filename="xxx.xacro"/>` | `<xmacro_include uri="file://xxx.xmacro"/>` |
| 条件 | `<xacro:if value="${...}">...</xacro:if>` | 主要通过 `<xmacro_block ... condition="${...}"/>` |
| namespace | `xmlns:xacro=...` | 不需要 |
| 参数默认值等高级功能 | 较丰富 | 简化很多 |

#### 如何先建立自己的功能包和模型包

这个`sdf.xmacro`会被依赖去发布仿真里的模型和实物的tf关系。

这里是include了下面这个文件，所以要找这个文件。

```xml
<xmacro_include uri="model://rm25_example_robot/rm25_example_robot.def.xmacro" />
```

他在`/src/rmoss_gz_resources/resource/models/rm25_example_robot`里，
你复制下这个大文件夹，并重命名为`ysu2026_sentry_robot`，里面的xmacro也命名为`ysu2026_sentry_robot.def.xmacro`。

这个文件里面需要修改的是：

```xml
<!--macro defination: rm25_example_robot-->
<xmacro_define_block
    name="rm25_example_robot"
    params="sticker_type chassis_height gimbal_pitch_height gimbal_yaw_height">
```

把这里的name改成

```xml
<!--macro definition: ysu2026_sentry_robot-->
<xmacro_define_block
    name="ysu2026_sentry_robot"
    params="sticker_type chassis_height gimbal_pitch_height gimbal_yaw_height">
```

接下来去修改同目录的`model.sdf.xmacro`：

```xml
    <!--include defination-->
    <xmacro_include uri="model://rm25_example_robot/rm25_example_robot.def.xmacro" />

    <!--rm25_example_robot model-->
    <model name="rm25_example_robot">
        <pose>0 0 0.15 0 0 0</pose>
        <xmacro_block name="rm25_example_robot" sticker_type="${global_armor_sticker_type}" chassis_height="${chassis_height}" gimbal_yaw_height="${gimbal_yaw_height}" gimbal_pitch_height="${gimbal_pitch_height}"/>
```

把上面`rm25_example_robot`全部改成`ysu2026_sentry_robot`。

```xml
    <!--include defination-->
    <xmacro_include uri="model://ysu2026_sentry_robot/ysu2026_sentry_robot.def.xmacro" />

    <!--ysu2026_sentry_robot model-->
    <model name="ysu2026_sentry_robot">
        <pose>0 0 0.15 0 0 0</pose>
        <xmacro_block name="ysu2026_sentry_robot" sticker_type="${global_armor_sticker_type}" chassis_height="${chassis_height}" gimbal_yaw_height="${gimbal_yaw_height}" gimbal_pitch_height="${gimbal_pitch_height}"/>
```


接下来去修改同目录的`model.config`：

```xml
<?xml version="1.0"?>
<model>
  <name>rmua19_standard_robot</name>
  <version>1.0</version>
  <sdf version="1.7">model.sdf</sdf>

  <author>
    <name>Lihan Chen</name>
    <email>lihanchen2004@163.com</email>
  </author>

  <description>
    The Basic Version of Standard Robot for RoboMaster University Series.
  </description>
</model>
```

修改成：

```xml
<?xml version="1.0"?>
<model>
  <name>ysu2026_sentry_robot</name>
  <version>1.0</version>
  <sdf version="1.7">model.sdf</sdf>

  <author>
    <name>Tung Chia-hui</name>
    <email>tungchiahui@gmail.com</email>
  </author>

  <description>
    Yanshan University RoboMaster 2026 Sentry Robot.
  </description>
</model>
```

这里的`model.sdf`不需要手动改，这玩意是自动生成的：

```text
model.sdf.xmacro
      ↓
xmacro4sdf
      ↓
model.sdf
```

你可以看看`model.sdf`开头，明确说了这件事，而且不建议你直接修改：

```xml
<!-- =================================================================================== -->
<!-- |    This document was autogenerated by xmacro4sdf from model.sdf.xmacro          | -->
<!-- |    EDITING THIS FILE BY HAND IS NOT RECOMMENDED                                 | -->
<!-- =================================================================================== -->
```

所以以后真正维护的是：

```text
关于模型：
ysu2026_sentry_robot.def.xmacro
model.sdf.xmacro

关于（这个不改问题也不大）：
model.config
```

回到`/src/robot_description/resource/xmacro/reality_robot.sdf.xmacro`：

```xml
    <!--include defination-->
    <xmacro_include uri="model://rm25_example_robot/rm25_example_robot.def.xmacro" />

    <!--pb2025_sentry_robot model-->
    <model name="pb2025_sentry_robot">

        <pose>0 0 0.15 0 0 0</pose>
        <xmacro_block name="rm25_example_robot" sticker_type="${global_armor_sticker_type}" chassis_height="${chassis_height}" gimbal_yaw_height="${gimbal_yaw_height}" gimbal_pitch_height="${gimbal_pitch_height}"/>
```

这里分开修改：include 的目录和 `.def.xmacro` 文件改为 `ysu2026_sentry_robot`，`xmacro_block` 调用的宏名也改为 `ysu2026_sentry_robot`；入口中的顶层 `<model name>` 改为 `reality_robot`。

顶层模型名表示当前描述入口，宏名表示被展开的具体机械结构，两者不必相同：

```xml
    <!--include defination-->
    <xmacro_include uri="model://ysu2026_sentry_robot/ysu2026_sentry_robot.def.xmacro" />

    <!--reality_robot model-->
    <model name="reality_robot">

        <pose>0 0 0.15 0 0 0</pose>
        <xmacro_block name="ysu2026_sentry_robot" sticker_type="${global_armor_sticker_type}" chassis_height="${chassis_height}" gimbal_yaw_height="${gimbal_yaw_height}" gimbal_pitch_height="${gimbal_pitch_height}"/>
```


至此上面这些在`reality`环境下已经正常了，但`simulation`还不行，打开同目录下的`simulation_robot.sdf.xmacro`：

```xml
    <!--include defination-->
    <xmacro_include uri="model://rm25_example_robot/rm25_example_robot.def.xmacro" />

    <!--simulation_robot model-->
    <model name="simulation_robot">

        <pose>0 0 0.15 0 0 0</pose>
        <xmacro_block name="rm25_example_robot" sticker_type="${global_armor_sticker_type}" chassis_height="${chassis_height}" gimbal_yaw_height="${gimbal_yaw_height}" gimbal_pitch_height="${gimbal_pitch_height}"/>
```

改成：

```xml
    <!--include defination-->
    <xmacro_include uri="model://ysu2026_sentry_robot/ysu2026_sentry_robot.def.xmacro" />

    <!--simulation_robot model-->
    <model name="simulation_robot">

        <pose>0 0 0.15 0 0 0</pose>
        <xmacro_block name="ysu2026_sentry_robot" sticker_type="${global_armor_sticker_type}" chassis_height="${chassis_height}" gimbal_yaw_height="${gimbal_yaw_height}" gimbal_pitch_height="${gimbal_pitch_height}"/>
```

至此，你就修改完了。

```bash
grep -R "rm25_example_robot" src/robot_description src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot

grep -R "pb2025_sentry_robot" src/robot_description src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot
```

下面是当时使用旧包名 `ysu2026_robot_description` 的终端记录，保留原始输出；现在复查时使用上面的 `src/robot_description` 路径：

```bash
root@Desktop-Fedora:/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry$ grep -R "rm25_example_robot" src/ysu2026_robot_description src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot
src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot/model.sdf:  <!--rm25_example_robot model-->
src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot/model.sdf:  <model name="rm25_example_robot">
root@Desktop-Fedora:/home/tungchiahui/UserFolder/MySource/ysu2026_rm_sentry$ grep -R "pb2025_sentry_robot" src/ysu2026_robot_description src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot
```

当时只有 `model.sdf` 里还有旧模型名残留，下面记录生成文件的更新步骤：

找到这个脚本：`/src/rmoss_gz_resources/update_model.sh`

```bash
#!/bin/bash
path=`pwd`
for dir in $path/models/*; do
    echo "parse "$dir
    cd $dir
    tempfile=`mktemp temp.XXXXXX`
    xmacro4sdf model.sdf.xmacro > $tempfile
    lines_num=`cat $tempfile | wc -l`
    if (($lines_num > 3 )) ;then
        cat $tempfile > model.sdf
    else
        cat $tempfile
    fi
    rm -f temp.*
done
```

修改为：

```bash
#!/bin/bash

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
MODEL_DIR="$SCRIPT_DIR/resource/models"

# 让 xmacro4sdf 能解析 model://xxx/...
export IGN_GAZEBO_RESOURCE_PATH="$MODEL_DIR:${IGN_GAZEBO_RESOURCE_PATH:-}"
export GAZEBO_MODEL_PATH="$MODEL_DIR:${GAZEBO_MODEL_PATH:-}"

for dir in "$MODEL_DIR"/*; do
    [ -d "$dir" ] || continue
    [ -f "$dir/model.sdf.xmacro" ] || continue

    echo "parse $dir"

    cd "$dir" || exit 1

    tempfile=$(mktemp temp.XXXXXX)

    if xmacro4sdf model.sdf.xmacro > "$tempfile"; then
        lines_num=$(wc -l < "$tempfile")

        if (( lines_num > 3 )); then
            cat "$tempfile" > model.sdf
            echo "generated: $dir/model.sdf"
        else
            echo "ERROR: generated content too short"
            cat "$tempfile"
        fi
    else
        echo "ERROR: xmacro4sdf failed: $dir"
    fi

    rm -f "$tempfile"
done
```

执行：

```bash
chmod +x ./src/rmoss_gz_resources/update_model.sh
./src/rmoss_gz_resources/update_model.sh
```


生成完后，返回ws目录：

```bash
grep -R "rm25_example_robot" src/robot_description src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot
```

会发现也没有输出了。

#### 修改机器人参数

主要是机器人的**底盘长宽、轮距/轴距、轮径、云台高度、Mid360 安装位置**：

```text
1. 底盘长宽高 / collision
2. 四个轮子的位置
3. 轮子半径
4. chassis_height
5. gimbal_yaw_height
6. gimbal_pitch_height
7. Mid360 的 parent 和 pose
```

首先你得知道ROS默认规定的方向：

```text
+x：车头方向
+y：车体左侧
+z：竖直向上
```

```text
+roll：绕x轴逆时针
+pitch：绕y轴逆时针
+yaw：绕z轴逆时针
```

角度正方向则统一遵守`右手定则`：
如果你站在每根轴的正方向一端，朝原点看，那么正旋转看起来都是逆时针。
所以对于机器人坐标系，可以更直观地记：
- +yaw：车头向左转。
- +pitch：车头向下压。
- +roll：左侧抬高、右侧降低。


而且北极熊现在的模型里，base_footprint 在地面，chassis 原点基本位于四个轮子的轮轴中心高度。

| 参数 | 你实际应该量什么 | 当前代码含义 |
|---|---|---|
| 底盘长度 | 车体刚性主体前沿到后沿，沿 X | `geometry_box x=...` |
| 底盘宽度 | 车体刚性主体左沿到右沿，沿 Y | `geometry_box y=...` |
| 底盘高度 | 刚性主体底面到顶面，沿 Z | `geometry_box z=...` |
| 轴距 | 前轮轴心到后轮轴心的 X 向距离 | `x_front - x_rear` |
| 轮距 | 左轮中心面到右轮中心面的 Y 向距离 | `y_left - y_right` |
| 轮径 | 轮子外圆直径 | 当前 `2 × 0.0758` |
| `chassis_height` | 地面 → `chassis` 坐标系原点 | 当前 0.076 m |
| `gimbal_yaw_height` | `chassis` 原点 → 云台 yaw 旋转轴原点 | 当前 0.1376 m |
| `gimbal_pitch_height` | yaw 轴原点 → pitch 轴原点 | 当前 0.16 m |
| Mid360 `pose` | parent 坐标系原点 → `front_mid360` 原点 | 当前相对 `chassis` |

1. ysu2026_sentry_robot.def.xmacro    ← 机器人“本体结构”的唯一真源
2. reality_robot.sdf.xmacro           ← 实车入口：传感器安装、实车专用高度/姿态
3. simulation_robot.sdf.xmacro        ← 仿真入口：仿真传感器和 Gazebo 插件
4. model.sdf.xmacro                   ← rmoss_gz_resources 里的 standalone Gazebo 模型入口

改“机器人本体长什么样” → 改 ysu2026_sentry_robot.def.xmacro。
改“实车传感器装在哪里、实车采用哪些参数” → 改 reality_robot.sdf.xmacro。
改“Gazebo 里怎么跑” → 改 simulation_robot.sdf.xmacro。
model.sdf.xmacro 只是 rmoss_gz_resources 里的完整 Gazebo model 入口，不是你实车 TF 的主入口。（所以这个model.sdf.xmacro其实没那么重要，但顺手改了也行。）


找到`/src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot`：

主要修改下面几个：

1. 底盘长度和底盘宽度（只影响Gazebo）

底盘长宽和轴距/轮距不是一个东西

比如现在北极熊：

```xml
        <!--chassis-->
        <link name="chassis">
            <inertial>
                <pose>0 0 0.055 0 0 0</pose>
                <xmacro_block name="inertia_box" m="15.0" x="0.36" y="0.24" z="0.13"/>
```

这些参数主要是用于Gazebo。

```text
底盘 collision:
底盘刚体的质量 m = 15.0 kg
长度 x = 0.36
宽度 y = 0.24
高度 z = 0.13
```

经测量车长X=525mm

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791189092339.webp)

经测量车宽Y=525mm

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791189093804.webp)

经测量车高度Z=20mm

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791189095266.webp)

注意，这里的`<pose>0 0 0.055 0 0 0</pose>`里的`0.055`不是底盘高度。
实际上是chassis 坐标原点 → 底盘质心 COM 的 Z 偏移。
北极熊这套规定`chassis z=0 等于 轮轴中心平面`。
所以如果机器人实际重量都在顶上，质心大概在高出轮轴中心平面 xxx mm的地方，那就填 `<pose>0 0 xxx 0 0 0</pose>`。

这东西只影响Gazebo，放心大胆填就完了，没必要很精细。

基于上面的数据，我修改成了

```xml
        <!--chassis-->
        <link name="chassis">
            <inertial>
                <pose>0 0 0.055 0 0 0</pose>
                <xmacro_block name="inertia_box" m="15.0" x="0.525" y="0.525" z="0.20"/>
            </inertial>
```

| 参数 | 实车 TF | Gazebo |
|---|---:|---:|
| Mid360 `<sensor><pose>0.03` | ❌ | ✅ |
| wheel collision radius | 基本 ❌ | ✅ |
| chassis collision | ❌ | ✅ |
| `chassis_height` | ✅ | ✅ |
| `gimbal_yaw_height` | ✅ | ✅ |
| `gimbal_pitch_height` | ✅ | ✅ |
| Mid360 `<xmacro_block pose=...>` | ✅ | ✅ |


下面还有一堆北极熊原来的：

```xml
<collision name="chassis_base_collision">
<collision name="chassis_projectile_container_collision">
<collision name="chassis_fornt_wheel_holder_collision">
<collision name="chassis_rear_wheel_holder_collision">
```

这里的`pose z`是底盘刚体的中心面的中心点在chassis 0点的什么位置，可以由SW测量得知，是65.46mm

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791350000418-1e7d4b6b.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791350056801-72f20409.webp)

可以修改成：

```xml
            <collision name="chassis_base_collision">
                <pose>0 0 0.06546 0 0 0</pose>
                <xmacro_block name="geometry_box" x="0.525" y="0.525" z="0.20"/>
            </collision>
```

而下面这仨可以全注释掉，因为咱们舵轮底盘的碰撞区域就是近似一个正方形，用`chassis_base_collision`足够了。

```xml
<collision name="chassis_projectile_container_collision">
<collision name="chassis_fornt_wheel_holder_collision">
<collision name="chassis_rear_wheel_holder_collision">
```

```xml
            <!-- <collision name="chassis_projectile_container_collision">
                <pose>-0.122 0 0.177 0 0 0</pose>
                <xmacro_block name="geometry_box" x="0.1" y="0.18" z="0.146"/>
            </collision>
            <collision name="chassis_fornt_wheel_holder_collision">
                <pose>0.208 0 0 0 0 0</pose>
                <xmacro_block name="geometry_box" x="0.055" y="0.367" z="0.055"/>
            </collision>
            <collision name="chassis_rear_wheel_holder_collision">
                <pose>-0.208 0 0 0 0 0</pose>
                <xmacro_block name="geometry_box" x="0.055" y="0.367" z="0.055"/>
            </collision> -->
```


1. 底盘高度

这个参数如其字面意思一样，就是底盘的厚度，也就是底盘的上顶面到下底面的距离。


3. 轴距和轮距也要量“轮中心”，不是底盘外壳

北极熊的是：

```xml
        <!--front_left wheel link and joint-->
        <xmacro_block name="chassis_wheel_l" prefix="front_left" parent="chassis" pose="0.207 0.194 0 0 0 0"/>
        <!--front_right wheel link and joint-->
        <xmacro_block name="chassis_wheel_r" prefix="front_right" parent="chassis" pose="0.207 -0.194 0 0 0 0"/>
        <!--rear_left wheel link and joint-->
        <xmacro_block name="chassis_wheel_r" prefix="rear_left" parent="chassis" pose="-0.207 0.194 0 0 0 0"/>
        <!--rear_right wheel link and joint-->
        <xmacro_block name="chassis_wheel_l" prefix="rear_right" parent="chassis" pose="-0.207 -0.194 0 0 0 0"/>
```

```text
FL:  0.207   0.194
FR:  0.207  -0.194
RL: -0.207   0.194
RR: -0.207  -0.194
```

那么轴距是 = 0.207 - (-0.207) = 0.414 m

那么轮距是 = 0.194 - (-0.194) = 0.388 m

示意图：

```text
                 +X 前

       FL ●────────────● FR
          │            │
          │            │
          │            │
          │            │
       RL ●────────────● RR

          ←  轮距  →

前后两个轮轴中心的距离
        = 轴距
```

所以要测量的数据是：
前左轮中心 到 后左轮中心：X 方向距离
左轮中心面 到 右轮中心面：Y 方向距离
不是量轮胎最外侧。

如下图显示轮距为324mm

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791189244150.webp)

如下图显示轴距也是324mm

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791189083467.webp)

324/2=162

所以是

```text
FL:  0.162   0.162
FR:  0.162  -0.162
RL: -0.162   0.162
RR: -0.162  -0.162
```

```xml
        <!--front_left wheel link and joint-->
        <xmacro_block name="chassis_wheel_l" prefix="front_left" parent="chassis" pose="0.162 0.162 0 0 0 0"/>
        <!--front_right wheel link and joint-->
        <xmacro_block name="chassis_wheel_r" prefix="front_right" parent="chassis" pose="0.162 -0.162 0 0 0 0"/>
        <!--rear_left wheel link and joint-->
        <xmacro_block name="chassis_wheel_r" prefix="rear_left" parent="chassis" pose="-0.162 0.162 0 0 0 0"/>
        <!--rear_right wheel link and joint-->
        <xmacro_block name="chassis_wheel_l" prefix="rear_right" parent="chassis" pose="-0.162 -0.162 0 0 0 0"/>
```



2. 轮径

打开`/src/robot_description/resource/xmacro/reality_robot.sdf.xmacro`:

这是轮子的直径，不是半径，注意下。

如下图轮子直径是125mm

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791189090628.webp)


1. 底盘距离水平地面的高度chassis_height：

```xml
    <!-- 底盘距离水平地面的高度 -->
    <xmacro_define_value name="chassis_height" value="0.077" />
    <!-- gimbal_yaw 相对于底盘的 z 轴安装偏移 -->
    <xmacro_define_value name="gimbal_yaw_height" value="0.026" />
    <!-- gimbal_pitch 相对于 gimbal_yaw 的高度差 -->
    <xmacro_define_value name="gimbal_pitch_height" value="0.355" />
```

这里的几个参数，这里的`chassis_height`所描述的是：

```xml
<joint name="base_to_chassis" type="fixed">
    <pose>0 0 ${chassis_height} 0 0 0</pose>
    <parent>base_footprint</parent>
    <child>chassis</child>
</joint>
```

由于四个轮子的 joint 都是：

```xml
parent="chassis"
pose="... ... 0 0 0 0"
```

所以结构其实是：

```text
          chassis 原点
               ●──────── 底盘原点（北极熊这套规定为轮轴中心平面的中心点）
               │
               │ chassis_height
               │
地面 ──────────●──────────
          base_footprint
```

实际上北极熊这套认为规定chassis 坐标系的 z=0 就是 四个车轮轮轴中心所在的水平面

我们遵循北极熊的规定

base_footprint：地面
chassis：**轮轴中心平面**

实际上这个chassis确实不用是真的底盘，可以是空气，在底盘下方和底盘上方都可以。

所以此时，底盘高度就是轮子半径，也就是上面测量的轮子直径除以2。

此时，在底面上建一个参考基准面，再距离这个参考基准面上方距离轮子半径距离处再建一个平行的基准面。

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791189105313.webp)

对这个面进行草图绘制，然后把原点转实体引用，这个点就是轮轴中点，也是chassis中点。（这是北极熊规定的轮轴中点就是chassis中点）

我这里自然就是125mm/2=62.5mm

```xml
    <!-- 底盘距离水平地面的高度 -->
    <xmacro_define_value name="chassis_height" value="0.0625" />
```

还有个只影响**仿真**的地方需要修改，在`src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot/ysu2026_sentry_robot.def.xmacro`里面 include 的却还是北极熊公共轮子`<xmacro_includeuri="model://rmua19_standard_robot/chassis_wheel.def.xmacro" />`。

所以，真正的半径在`src/rmoss_gz_resources/resource/models/rmua19_standard_robot/chassis_wheel.def.xmacro`。

里面：

```xml
<xmacro_define_block name="chassis_wheel_l" params="prefix parent pose">

<xmacro_block
    name="inertia_cylinder"
    m="0.4"
    r="0.0758"
    l="0.0452"/>

<xmacro_block
    name="geometry_cylinder"
    r="0.0758"
    l="0.0452"/>

<xmacro_define_block name="chassis_wheel_r" params="prefix parent pose">

<xmacro_block
    name="inertia_cylinder"
    m="0.4"
    r="0.0758"
    l="0.0452"/>

<xmacro_block
    name="geometry_cylinder"
    r="0.0758"
    l="0.0452"/>
```

这里的`r="0.0758"`就是仿真轮子的碰撞半径。应该改成咱们轮子的半径62.5mm。

但我不建议直接改这个文件`rmua19_standard_robot/chassis_wheel.def.xmacro`，这个文件是公用的。

推荐复制一份到`src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot/chassis_wheel.def.xmacro`。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791277565471-19edbdb7.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791350215341-b1936f7f.webp)

修改掉下面这俩里的r半径，然后轮子厚度l也可以改一下。（注意把左右轮子的都修改了）

r = 圆柱半径
l = 圆柱沿自身轴线方向的长度

```xml
<xmacro_block name="inertia_cylinder"  m="0.4" r="0.0625" l="0.025"/>
<xmacro_block name="geometry_cylinder" r="0.0625" l="0.025"/>
```

把`ysu2026_sentry_robot.def.xmacro`里的`<xmacro_include uri="model://rmua19_standard_robot/chassis_wheel.def.xmacro" />`改成`<xmacro_include uri="model://ysu2026_sentry_robot/chassis_wheel.def.xmacro" />`。


1. 云台高度gimbal_yaw_height

```text
chassis
   ↓
gimbal_yaw_odom_joint      ← 大 Yaw
   ↓
gimbal_yaw_odom
   ↓
gimbal_pitch_odom_joint    ← fixed，只做辅助 frame
   ↓
gimbal_pitch_odom
   ↓
gimbal_yaw_joint           ← 小 Yaw
   ↓
gimbal_yaw
   ↓
gimbal_pitch_joint         ← 小 Pitch
   ↓
gimbal_pitch
```

```text
            gimbal_pitch
                 ●
                 │
                 │ gimbal_pitch_height
                 │
                 ●
          大 Yaw 旋转中心
                 │
                 │ gimbal_yaw_height
                 │
                 ●
          chassis 原点
          = 轮轴中心平面
```

```text
gimbal_yaw_height
= chassis → 大 Yaw

gimbal_pitch_height
= 大 Yaw → Pitch
```

```xml
        <joint name="gimbal_yaw_odom_joint" type="revolute">
        <pose relative_to='chassis'>
            0 0 ${gimbal_yaw_height} 0 0 0
        </pose>

        <joint name="gimbal_pitch_odom_joint" type="fixed">
            <pose relative_to='gimbal_yaw_odom'>0 0 ${gimbal_pitch_height} 0 0 0</pose>

        <joint name="gimbal_yaw_joint" type="revolute">
            <pose relative_to='gimbal_pitch_odom'>0 0 -${gimbal_pitch_height} 0 0 0
            </pose>
```

看这里可以知道，`gimbal_yaw_odom_joint`和`gimbal_yaw_joint`是重合的。（大小Yaw重合）

他描述的是

```text
chassis 坐标系原点
        │
        │ gimbal_yaw_height
        │
        ▼
云台 gimbal_yaw_odom_joint（大Yaw） 旋转轴原点 
```

也就是chassis 坐标系原点到 yaw 旋转轴中心的 Z 向距离。

如果云台相对于底盘有前后偏移，那么不能只填`gimbal_yaw_height`，应该填完整的pose。

看这个云台最中心的环与底盘原点之间只有dz，说明云台在车的最中央，无偏移。

经测量，gimbal_yaw_height为210.46mm

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791189106676.webp)

```xml
    <!-- gimbal_yaw 相对于底盘的 z 轴安装偏移 -->
    <xmacro_define_value name="gimbal_yaw_height" value="0.21046" />
```

除了改尺寸以外，由于咱们没有大小Yaw结构，而Gazebo又只给了小Yaw控制器，所以咱们把`gimbal_yaw_odom_joint`给固定死吧。

打开`/src/rmoss_gz_resources/resource/models/ysu2026_sentry_robot/ysu2026_sentry_robot.def.xmacro`把

```xml
        <joint name="gimbal_yaw_odom_joint" type="revolute">
            <pose relative_to='chassis'>0 0 ${gimbal_yaw_height} 0 0 0</pose>
            <parent>chassis</parent>
            <child>gimbal_yaw_odom</child>
            <axis>
                <xyz>0 0 1</xyz>
                <limit>
                    <effort>100</effort>
                    <velocity>50</velocity>
                    <lower>-1.79769e+308</lower>
                    <upper>1.79769e+308</upper>
                </limit>
            </axis>
        </joint>
```

改成

```xml
        <joint name="gimbal_yaw_odom_joint" type="fixed">
            <pose relative_to="chassis">0 0 ${gimbal_yaw_height} 0 0 0</pose>
            <parent>chassis</parent>
            <child>gimbal_yaw_odom</child>
        </joint>
```


1. 云台高度gimbal_pitch_height

```text
                    ● pitch 旋转轴
                    │
                    │ gimbal_pitch_height
                    │
                    ● yaw 旋转轴
                    │
                    │ gimbal_yaw_height
                    │
                    ● chassis 原点
                    │
                    │ chassis_height
                    │
────────────────────● 地面
```

```xml
<joint name="gimbal_pitch_odom_joint" type="fixed">
    <pose relative_to="gimbal_yaw_odom">
        0 0 ${gimbal_pitch_height} 0 0 0
    </pose>
```

```xml
<joint name="gimbal_pitch_joint" type="revolute">
    <pose relative_to="gimbal_yaw">
        0 0 ${gimbal_pitch_height} 0 0 0
    </pose>
```

这里大小pitch是重合的。

云台高度gimbal_pitch_height所表示是大 Yaw 轴中心 → Pitch 旋转轴中心的垂直高度差。

也就是 yaw 旋转轴中心 → pitch 旋转轴中心的垂直距离。

如下图云台高度gimbal_pitch_height是 96.41mm

![alt text](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/09/20/1791189237016.webp)

```xml
    <!-- gimbal_pitch 相对于 gimbal_yaw 的高度差 -->
    <xmacro_define_value name="gimbal_pitch_height" value="0.09641" />
```

