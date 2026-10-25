---
title: "read-update-write控制周期"
---

> read、update、write 的名字容易记，难点是调用线程、实际周期和时钟来源。

## 示例 1：沿 main() 阅读两个执行路径

打开 ros2_control_node.cpp：

```text
创建 Executor 和 ControllerManager
  → 创建控制线程
      → 计算时间与测得周期
      → read → update → write
      → 安排下一轮
  → Executor spin，处理消息与服务回调
```

订阅回调保存目标，update 在控制线程取快照。服务请求也不应直接把正在 update 的对象随意销毁。

## 示例 2：逐轮推演

```text
N 轮 read：读当前反馈
回调接收新目标
N 轮 update：若取得新快照，就计算新命令
N 轮 write：交给设备
N+1 轮 read：取得新的设备反馈
```

若消息在 update 之后到达，就可能下一轮才被使用。不要承诺所有消息都在一个固定周期内生效。

## 示例 3：比较命令频率与反馈频率

```bash
ros2 topic hz /joint_states
```

另一个终端以较低频率发送：

```bash
ros2 topic pub --rate 2 /forward_position_controller/commands \
  std_msgs/msg/Float64MultiArray "{data: [0.2, -0.1]}"
```

反馈可比命令到达更频繁。topic hz 测量的是接收端话题频率，不是实时线程的精确调度。

### time、period 与 update_rate

update_rate 是目标频率。period 是实际测得的时间间隔，算法应使用它积分或计算步长。time 属于调用所采用的时钟，不能把 ROS 时间、单调时间和墙上时间混在一起相减。

仿真使用 /clock：暂停世界时，仿真时间可能不前进。通信失联期限适合用单调时间检查，不能让暂停仿真掩盖真实设备已经失联。

## Gazebo 的调用入口

Harmonic 插件在仿真更新过程中调度框架，不应再额外启动同一机器人的独立 ros2_control_node。两种运行后端共享接口契约，调度入口不同。

源码：[ros2_control/controller_manager/src/ros2_control_node.cpp](https://github.com/ros-controls/ros2_control/blob/b7f1a5d420443c7a079f4c81e80d5e39759acf4c/controller_manager/src/ros2_control_node.cpp)、[gz_ros2_control/gz_ros2_control/src/gz_ros2_control_plugin.cpp](https://github.com/ros-controls/gz_ros2_control/blob/89ca76ed20842377f879c711c756d444c5b7c20b/gz_ros2_control/src/gz_ros2_control_plugin.cpp)。

---

[返回 Jazzy 总览](/wiki/2023-12-30-ros2-tutorial/1500-0200-jazzy) · 上一篇：[硬件与控制器生命周期](/wiki/2023-12-30-ros2-tutorial/1500-0200-0900-ying-jian-yu-kong-zhi-qi-sheng-ming-zhou-qi) · 下一篇：[状态广播与机器人显示](/wiki/2023-12-30-ros2-tutorial/1500-0200-1100-zhuang-tai-guang-bo-yu-ji-qi-ren-xian-shi)
