---
title: "回溯rosbag2"
---

### 概述
![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1228.webp)

方式1一边采集数据，一边生成地图信息。

方式2是将采集数据和生成地图信息分割开来了，方式2做到了解耦合，所以更灵活一些。用同一套数据，可能用不同的算法处理，总之，非常灵活。

留存的过程咱们也可以叫做序列化。（转化为磁盘文件）

留存一般叫录制（序列化）。

读取一般叫回放(反序列化)。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1229.webp)

留存时也可以将文件进行分卷，也就是每个文件最大能占多大的大小，如果超过该大小，就新建一个文件继续留存。（类似于压缩文件的分卷）

这样的话，存的数据太大，我们一次性打开太慢，就可以分段打开。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1230.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1231.webp)

需要依赖于rosbag2_cpp或者rosbag2_py

然后还要依赖于geometry_msgs，这个是因为我们要序列化的数据是这个包下的速度指令。

```bash
ros2 pkg create cpp02_rosbag --build-type ament_cmake --dependencies rclcpp rosbag2_cpp geometry_msgs --node-name cpp01_writer
```

### rosbag2的命令工具
![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1232.webp)

一般ROSBAG2的使用有命令行工具和编码两种使用方式，**命令行工具功能比较齐全，够用。**

查看帮助文档ros2 bag -h

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1233.webp)

主要有6个指令。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1234.webp)![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1235.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1236.webp)

可以看record的详细用法。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1237.webp)

record是用来将消息序列化的，但实际上比这个复杂。(录制)

play是用来反序列化消息的，但实际上比这个复杂。(回放)

```text
ROS 2 Message
    ↓ serialization
serialized bytes
    ↓ rosbag2 + timestamp + topic metadata + storage plugin
bag
```

序列化只是录制过程中的一个环节。

record 实际上包含：

```text
发现 Topic
    ↓
建立 Subscription
    ↓
接收序列化消息
    ↓
记录时间戳
    ↓
交给 Storage Plugin
    ↓
写入磁盘
```

```text
读取 bag
    ↓
按照记录时间调度消息
    ↓
创建 Publisher
    ↓
按照 QoS 发布
    ↓
ROS 2 其他节点重新收到消息
```

这一点非常重要，因为后面所有 QoS 问题，**本质上就是 Recorder 是订阅者、Player 是发布者**。

info是用来输出bag文件的相关信息的。比如有多少条消息，录制起始时间和终止时间以及持续时间。

reindex是重建bag文件，可以根据现有 bag 数据重新构建 metadata。
例如异常退出导致 metadata.yaml 缺失或损坏时，它很有价值。

list是输出rosbag2中可用的插件(高阶应用)。

convert我们可以用这个给bag重新读取再写出一个新的 bag，也可以把多个bag文件合并成一个文件。
可以用来：
- 合并 bag
- 拆分 bag
- 重新选择话题
- 改 storage
- 压缩/解压
- 修改序列化格式

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1238.webp)

打开小乌龟节点与键盘控制小乌龟节点。

我们用键盘控制小乌龟，然后把速度指令通过rosbag2给序列化。

然后我们关掉两个节点，再重启小乌龟节点，然后这次不通过键盘控制，而是通过play bag文件让小乌龟运动。

record指令后面要跟一个话题组成的列表，但是在咱们下面的操作中，只用到了一个话题。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1239.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1240.webp)

然后再用output，把序列化后的文件写出到一个磁盘目录中去。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1241.webp)

cd进想保存bag的目录，然输入record指令，后面跟话题名称，然后-o +bag文件名，这里也可以不重新命名bag文件，这样会用默认的名字，默认的名称是年月日命名的。

这样就已经开始录制了。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1242.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1243.webp)

按Ctrl + C进行结束，结束有个提示，说正在将消息写入bag，需要一段时间。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1244.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1245.webp)

```text
my_bag/
├── metadata.yaml
└── my_bag_0.db3
```
其中：

```text
metadata.yaml
    → bag 的描述信息
    → 录了哪些 topic
    → topic 类型
    → 消息数量
    → duration
    → storage id
    → bag 分片信息
    ...

*.db3
    → SQLite3 storage plugin 真正保存的消息数据
```

db3是SQLite数据库，这个是移动端(比如手机)常用的数据库，但其他平台其实也可以用。
所以 .db3 才是主要的数据载体。

这个数据库就存储了录制的数据。

Humble 默认用 SQLite3 storage plugin 保存 bag。

而且存储插件是可替换的。你本机究竟有哪些插件，不应该靠猜，应该直接：

```text
ros2 bag list storage
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/1792764659701-0064e3a3.webp)

Humble 的 CLI 会根据系统实际注册的 writer 插件决定 -s/--storage 可以选择什么。[ROS 2 Humble rosbag2 README](https://github.com/ros2/rosbag2/blob/humble/README.md)

### rosbag2中的QoS

前面已经知道，rosbag2 在录制和回放时并不是单纯地“读写文件”：

```text
录制：Publisher → rosbag2 Recorder（Subscriber）

回放：rosbag2 Player（Publisher）→ Subscriber
```

因此，只要 rosbag2 参与 ROS 2 通信，就同样要遵守 ROS 2 的 QoS（Quality of Service）规则。

ROS 2 的 QoS 策略很多，例如 History、Depth、Reliability、Durability、Deadline、Liveliness 等。

对于 rosbag2 的录制和回放，首先最需要理解的是：

- Reliability：`Reliable` 和 `Best Effort`
- Durability：`Transient Local` 和 `Volatile`

官方文档：

- [ROS 2 Humble：Quality of Service settings](https://github.com/ros2/ros2_documentation/blob/humble/source/Concepts/Intermediate/About-Quality-of-Service-Settings.rst)
- [ROS 2 Humble：Overriding QoS Policies for Recording and Playback](https://github.com/ros2/ros2_documentation/blob/humble/source/How-To-Guides/Overriding-QoS-Policies-For-Recording-And-Playback.rst)

#### Request / Offered 模型

ROS 2 中，Publisher 和 Subscriber 的 QoS 并不是“必须完全相同”才能通信。

它采用的是 Request / Offered 模型：

```text
Publisher
    ↓
Offered QoS（提供的服务质量）

Subscriber
    ↓
Requested QoS（请求的服务质量）
```

只有 Publisher 提供的 QoS 能满足 Subscriber 请求的 QoS，两者才能正常通信。

所以不能简单理解成：

```text
Publisher QoS == Subscriber QoS
```

而应该理解成：

```text
Publisher 提供的能力
        ↓
能不能满足 Subscriber 的要求？
```

这个规则对普通 ROS 2 节点成立，对 rosbag2 也一样成立。

录制时：

```text
原节点 Publisher
        ↓
rosbag2 Recorder Subscriber
```

回放时：

```text
rosbag2 Player Publisher
        ↓
算法节点 Subscriber
```

#### Reliability

Reliability 决定消息传输过程中，对“消息是否必须尽量送达”的要求。

**Reliable** 可以理解为：

```text
尽量保证消息送达
如果底层发现消息丢失，可以进行重传等处理
```

**Best Effort** 可以理解为：

```text
尽最大努力发送
如果某条消息丢失，不要求一定重传
更加关注最新的数据
```

很多高频传感器数据使用的 Sensor Data QoS Profile 会采用 `Best Effort`。对于这类数据，相比于等待旧数据重传，系统通常更关心最新的传感器数据。

Reliability 的兼容关系如下：

| Publisher Offered | Subscriber Requested | 是否兼容 |
|---|---|---|
| Best Effort | Best Effort | ✅ |
| Best Effort | Reliable | ❌ |
| Reliable | Best Effort | ✅ |
| Reliable | Reliable | ✅ |

最容易混淆的是下面两种情况：

```text
Best Effort Publisher
        ↓
Reliable Subscriber

❌ 不兼容
```

原因是 Subscriber 要求“可靠传输”，但 Publisher 只提供 Best Effort，无法满足 Subscriber 的要求。

反过来：

```text
Reliable Publisher
        ↓
Best Effort Subscriber

✅ 兼容
```

因为 Publisher 提供的能力已经能够满足 Subscriber 较低的要求。

因此，`Reliable` 并不是简单意义上的“比 Best Effort 更高级，所以任何时候都应该选 Reliable”。

QoS 的核心是：**双方是否兼容，以及这种通信策略是否适合当前数据。**

#### Durability

Durability 主要决定：**一个 Subscriber 后加入通信时，能不能收到 Publisher 之前已经发布过的数据。**

**Volatile** 可以理解为：

```text
只接收建立通信之后的新消息
不保留之前已经发布的历史消息
```

例如：

```text
Publisher：A  B  C

                Subscriber 在这里启动

Publisher：          D  E
```

如果 Subscriber 使用 Volatile，它通常只关心：

```text
D  E
```

而不会要求得到之前的：

```text
A  B  C
```

**Transient Local** 则允许 Publisher 为后加入的 Subscriber 保留一定的历史样本。

可以把它理解成：

```text
Publisher
   │
   ├── 发布消息
   │
   └── 在本地保留历史样本
             ↓
       Late Joiner Subscriber
```

它的行为和 ROS 1 中 latched publisher 的概念有一定相似之处。

Durability 的兼容关系如下：

| Publisher Offered | Subscriber Requested | 是否兼容 |
|---|---|---|
| Volatile | Volatile | ✅ |
| Volatile | Transient Local | ❌ |
| Transient Local | Volatile | ✅ |
| Transient Local | Transient Local | ✅ |

其中：

```text
Volatile Publisher
        ↓
Transient Local Subscriber

❌ 不兼容
```

因为 Subscriber 要求能够获得 Transient Local 的历史数据能力，但 Publisher 并没有提供这种能力。

而：

```text
Transient Local Publisher
        ↓
Volatile Subscriber

✅ 可以通信
```

但 Volatile Subscriber 并不会因为 Publisher 是 Transient Local，就自动要求得到建立通信之前的历史数据。

#### /tf_static为什么特殊

`/tf_static` 是理解 Durability 很典型的例子。

静态 TF 的特点是：

```text
变换关系基本不会变化
        ↓
没有必要像动态 /tf 一样持续高频发布
```

例如机器人中的某个固定传感器变换：

```text
base_link
    ↓
laser_link
```

这个静态关系可能在系统启动时就已经发布。

假设：

```text
10:00:00
Static TF Publisher 发布 /tf_static

10:00:20
rosbag2 Recorder 才启动
```

如果 Recorder 只能接收启动之后的新数据，而 `/tf_static` 后面又没有重新发布，那么这个 bag 就可能缺少关键的静态坐标变换。

因此 `/tf_static` 使用 `Transient Local` Durability，使后加入的 Subscriber 也有机会获得之前保留的静态 TF 数据。

这也是为什么录制 SLAM、定位和导航数据时，不能只确认“bag 里面有没有 `/tf`”，还需要关注 `/tf_static` 是否被正确录制。

#### rosbag2会自动适配QoS

理解 QoS 以后，也不要走到另一个极端：

```text
是不是录每一个 Topic 都必须手动写 QoS？
```

通常不是。

ROS 2 Humble 的 rosbag2 在录制时会根据发现到的 Publisher 尝试调整 Recorder 的 Requested QoS；在回放时，也会尝试保留原 Topic 原先 Offered 的 QoS。

因此大多数普通 Topic 不需要手动指定 QoS。

但是在一些需要明确控制行为的场景中，例如：

- 关键传感器 Topic
- `/tf_static`
- 同一个 Topic 存在多个 Publisher，并且 QoS 不完全一致
- 默认自动适配无法满足实际录制或回放需求
- 希望实验中的 QoS 行为固定、可复现

就可以使用 rosbag2 的 QoS Override。

#### 查看Topic实际使用的QoS

不要只根据源码、Topic 名字或者经验猜测实际 QoS。

系统真正运行起来后，可以使用：

```bash
ros2 topic info -v /topic_name
```

查看该 Topic 当前 Publisher 和 Subscriber 实际使用的 QoS。

例如重点观察：

```text
Reliability
Durability
History
Depth
```

尤其在实车中，最终的 rosbag2 配置应该以**实际运行时的 Topic 和 QoS**为准。

#### rosbag2 QoS Override

rosbag2 可以通过 YAML 文件为特定 Topic 覆盖 QoS。

例如：

```yaml
/tf_static:
  reliability: reliable
  durability: transient_local
```

如果某个传感器 Topic 实际使用 Best Effort，也可以按实际情况配置：

```yaml
/sensor_points:
  reliability: best_effort
  durability: volatile
```

这里的 `/sensor_points` 只是示例。实际工程中应当先运行系统，再通过 `ros2 topic info -v` 确认真实 Topic 和 QoS。

录制时使用：

```bash
ros2 bag record \
  --qos-profile-overrides-path qos_overrides.yaml \
  /topic1 /topic2
```

此时 QoS Override 影响的是：

```text
rosbag2 Recorder
        ↓
Subscription QoS
```

回放时使用：

```bash
ros2 bag play \
  --qos-profile-overrides-path qos_overrides.yaml \
  my_bag
```

此时 QoS Override 影响的是：

```text
rosbag2 Player
        ↓
Publisher QoS
```

所以同一个参数在 record 和 play 中的作用对象并不相同。

最终可以把这一节的关系整理成：

```text
录制

原 Publisher
    │ Offered QoS
    ↓
rosbag2 Recorder
    Requested QoS


回放

rosbag2 Player
    │ Offered QoS
    ↓
算法 Subscriber
    Requested QoS
```

理解这个模型以后，后面处理传感器数据、`/tf_static` 以及 rosbag2 的 QoS Override，就不再只是记配置参数，而是在判断：

> **Publisher 提供的 QoS，能不能满足 Subscriber 请求的 QoS。**

### rosbag2 C++案例分析及框架搭建
![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1246.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1247.webp)

```cmake
add_executable(demo01_writer src/demo01_writer.cpp)
ament_target_dependencies(
  demo01_writer
  "rclcpp"
  "rosbag2_cpp"
  "geometry_msgs"
)

add_executable(demo02_reader src/demo02_reader.cpp)
ament_target_dependencies(
  demo02_reader
  "rclcpp"
  "rosbag2_cpp"
  "geometry_msgs"
)

install(TARGETS 
  demo01_writer
  demo02_reader
  DESTINATION lib/${PROJECT_NAME})
```

### rosbag2 C++ 录制数据
```cpp
/* 
  需求：录制 turtle_teleop_key 节点发布的速度指令。
  步骤：
    1.包含头文件；
    2.初始化 ROS 客户端；
    3.定义节点类；
      3-1.创建写出对象指针；
      3-2.设置写出的目标文件；
      3-3.写出消息。
    4.调用 spin 函数，并传入对象指针；
    5.释放资源。

 */
// 1.包含头文件；
#include "rclcpp/rclcpp.hpp"
#include "rosbag2_cpp/writer.hpp"
#include "geometry_msgs/msg/twist.hpp"

using std::placeholders::_1;

// 3.定义节点类；
class SimpleBagRecorder : public rclcpp::Node
{
public:
  SimpleBagRecorder()
  : Node("simple_bag_recorder")
  {
    // 3-1.创建写出对象指针；
    writer_ = std::make_unique<rosbag2_cpp::Writer>();
    // 3-2.设置写出的目标文件；(目录为ws目录)
    writer_->open("src/cpp02_rosbag/my_bag");
    subscription_ = create_subscription<geometry_msgs::msg::Twist>(
      "/turtle1/cmd_vel", 10, std::bind(&SimpleBagRecorder::topic_callback, this, _1));
  }

private:
  void topic_callback(std::shared_ptr<rclcpp::SerializedMessage> msg) const
  {
    rclcpp::Time time_stamp = this->now();
    // 3-3.写出消息。
    RCLCPP_INFO(this->get_logger(),"数据写出... ...");
    writer_->write(msg, "/turtle1/cmd_vel", "geometry_msgs/msg/Twist", time_stamp);
  }

  rclcpp::Subscription<geometry_msgs::msg::Twist>::SharedPtr subscription_;
  std::unique_ptr<rosbag2_cpp::Writer> writer_;
};

int main(int argc, char * argv[])
{
  // 2.初始化 ROS 客户端；
  rclcpp::init(argc, argv);
  // 4.调用 spin 函数，并传入对象指针；
  rclcpp::spin(std::make_shared<SimpleBagRecorder>());
  // 5.释放资源。
  rclcpp::shutdown();
  return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1248.webp)

这是一个相对目录，目录位置是工作空间目录。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1249.webp)

写数据之前，要先创建一个订阅方，订阅方要建立一个回调函数。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1250.webp)

回调函数入口参数，消息类型用write函数的入口参数。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1251.webp)

这里要用斜杠代替冒号，因为入口是string类型。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1252.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1253.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1254.webp)

创建乌龟节点。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1255.webp)

启动writer节点

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1256.webp)

运行小乌龟

运行一会儿后，关掉所有节点。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1257.webp)

成功生成了文件

验证录制文件是否成功。

在验证前，先创建一个乌龟节点，不用创建控制节点。

```cpp
ros2 bag play src/cpp02_rosbag/my_bag
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1258.webp)

可以看到乌龟正常走了。回放成功！

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1259.webp)

如果已经生成了一遍my_bag，再想生成新的会显示不能覆盖。

解决方案，可以把my_bag设置为动态的，加个时间戳或者直接按功能命名。

### rosbag2 C++ 读取数据
```cpp
/* 
  需求：读取 bag 文件数据。
  步骤：
    1.包含头文件；
    2.初始化 ROS 客户端；
    3.定义节点类；
      3-1.创建读取对象指针；
      3-2.设置读取的目标文件；
      3-3.读消息；
      3-4.关闭文件。
    4.调用 spin 函数，并传入对象指针；
    5.释放资源。

 */
 // 1.包含头文件；
 #include "rclcpp/rclcpp.hpp"
 #include "rosbag2_cpp/reader.hpp"
 #include "geometry_msgs/msg/twist.hpp"
 // 3.定义节点类；
class SimpleBagPlayer: public rclcpp::Node {
public:
    SimpleBagPlayer():Node("simple_bag_player"){
        // 3-1.创建读取对象指针；
        reader_ = std::make_unique<rosbag2_cpp::Reader>();
        // 3-2.设置读取的目标文件；
        reader_->open("src/cpp02_rosbag/my_bag");
        // 3-3.读消息；
        while (reader_->has_next())
        {
            auto twist = reader_->read_next<geometry_msgs::msg::Twist>();
            RCLCPP_INFO(this->get_logger(),"线速度:%.2f, 角速度: %.2f",twist.linear.x, twist.angular.z);
        }
        // 3-4.关闭文件。
        reader_->close();
    }
private:
    std::unique_ptr<rosbag2_cpp::Reader> reader_;

};

int main(int argc, char const *argv[]){
    // 2.初始化 ROS 客户端；
    rclcpp::init(argc,argv);
    // 4.调用 spin 函数，并传入对象指针；
    rclcpp::spin(std::make_shared<SimpleBagPlayer>());
    // 5.释放资源。
    rclcpp::shutdown();
    return 0;
}
```

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1260.webp)

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1261.webp)

编译

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1262.webp)

这显示能读出来几条信息。能读出来8条。

![](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2023/12/30/image1263.webp)

正好8条
