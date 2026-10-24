---
title: "read-update-write控制周期"
---

> 本节回答：是谁在调 read？订阅回调和 update 是否处在同一个循环？period 为什么不能直接写成 0.01？

## 示例 1：从 main() 找到两个执行路径

打开源码阅读目录中的 `ros2_control/controller_manager/src/ros2_control_node.cpp`，按下面顺序阅读：

```text
rclcpp::init
  → 创建 Executor
  → 创建 ControllerManager
  → 创建控制线程 cm_thread
      → 取得当前时间与测得周期
      → cm->read
      → cm->update
      → cm->write
      → 等待下一轮
  → executor->spin
```

这里是该版本独立 ros2_control_node 的结构。Gazebo 后端的调用入口在仿真插件里，不能假设它也运行同一个独立主函数。

## 示例 2：逐轮推演命令生效

假设本轮 read 完成之后，ROS 回调才收到新目标：

```text
第 N 轮 read：读取旧状态
ROS 回调：保存新目标
第 N 轮 update：如果读到新快照，更新 command
第 N 轮 write：下发更新后的 command
第 N+1 轮 read：取得命令作用后的反馈
```

如果回调发生在本轮 update 之后，新目标就可能到下一轮 update 才被使用。多线程调度下，不应承诺每次消息都在固定的某一轮生效。

## 示例 3：比较频率

启动 mini 后，在另外两个终端分别执行：

```bash
ros2 topic hz /joint_states
ros2 topic pub --rate 2 /forward_position_controller/commands \
  std_msgs/msg/Float64MultiArray "{data: [0.2, -0.1]}"
```

### 预期现象

即使命令以 2 Hz 到达，状态仍可按更高频率发布。`ros2 topic hz` 测量的是接收端观察到的话题频率，不是控制线程每一轮的精确调度。

### time 与 period 的参数意义

`time` 是本轮调用使用的 ROS 时间；`period` 是测得的时间间隔。模拟硬件用 period 推进模型，差速里程计用时间差积分。

本次独立节点源码中 read、update、write 分别取得当前时间，不能据此假定三个 time 数值逐纳秒一致。计算积分应尊重传入的周期和算法需要。

Gazebo 使用仿真时间：暂停时仿真时间不前进，不能用宿主机已经等待几秒来推断仿真已经执行几秒。

## 关键函数说明

- `ControllerManager::read()` 把读状态任务交给资源管理器。
- `ControllerManager::update()` 调度 active 控制器，并处理相关管理任务。
- `ControllerManager::write()` 让硬件后端处理命令。
- Executor 负责 ROS 回调、服务和 Action 等执行，不等于实时控制循环。

## 为什么不要在 read 中 sleep

如果每轮预算约 10 ms，而串口读取阻塞 100 ms，整个链路就无法维持原目标周期。后面的通信实现会把接收放在独立线程，read 只取最新完整反馈。

源码入口：[ros2_control/controller_manager/src/ros2_control_node.cpp](https://github.com/ros-controls/ros2_control/blob/65d389596953cb4656c4216a3f0bc08a73230739/controller_manager/src/ros2_control_node.cpp)、[ros2_control/controller_manager/src/controller_manager.cpp](https://github.com/ros-controls/ros2_control/blob/65d389596953cb4656c4216a3f0bc08a73230739/controller_manager/src/controller_manager.cpp)。顺着函数调用追踪到 ResourceManager，而不是只停留在主函数中的三个名字。

---

[返回 Humble 总览](1500-0100-Humble.md) · 上一篇：[硬件与控制器生命周期](1500-0100-0900-硬件与控制器生命周期.md) · 下一篇：[状态广播与机器人显示](1500-0100-1100-状态广播与机器人显示.md)
