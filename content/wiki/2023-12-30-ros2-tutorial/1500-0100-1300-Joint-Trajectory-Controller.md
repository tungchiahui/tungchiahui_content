---
title: "Joint-Trajectory-Controller"
---

> 轨迹控制器在时间轴上执行目标点。本节沿用自己的双关节工程，不再创建另一套模型。

## 示例 1：切换到轨迹控制器

保持 mini.launch.py 运行。YAML 中已经登记 trajectory_controller：

```bash
ros2 control load_controller trajectory_controller --set-state inactive
ros2 control switch_controllers \
  --deactivate forward_position_controller \
  --activate trajectory_controller --strict
ros2 control list_controllers
ros2 action list -t
```

### 执行顺序

load 创建并配置实例，处于 inactive；switch 在一次切换请求中释放旧控制器接口，再让新控制器取得接口。不要让两个控制器同时争用 joint1/position。

## 示例 2：通过 Action 发送完整目标

```bash
ros2 action send_goal /trajectory_controller/follow_joint_trajectory \
  control_msgs/action/FollowJointTrajectory \
  "{trajectory: {joint_names: [joint1, joint2], points: [
    {positions: [0.2, 0.1], time_from_start: {sec: 3}}
  ]}}" --feedback
```

### 成功时的示例输出

```text
error_code: 0
error_string: Goal successfully reached!
Goal finished with status: SUCCEEDED
```

MiniSystem 的一阶响应会滞后，配置中允许一定轨迹误差和额外到达时间。目标成功表示符合当前配置的成功条件，不表示真实设备没有误差。

## 示例 3：通过 Topic 发送两段轨迹

```bash
ros2 topic pub --once /trajectory_controller/joint_trajectory \
  trajectory_msgs/msg/JointTrajectory \
  "{joint_names: [joint1, joint2], points: [
    {positions: [0.3, -0.2], time_from_start: {sec: 2}},
    {positions: [0.0, 0.0], time_from_start: {sec: 5}}
  ]}"
```

### 时间参数说明

time_from_start 相对于整条轨迹起点，而不是相对于上一个点。上例第二段到达时间是开始后 5 秒，第二段时长约为 3 秒。点的时间应严格递增。

header.stamp 为零时使用控制器当前接收与执行逻辑安排起点；指定绝对时间时必须保证时间域一致。在仿真中尤其不要混用墙上时间和仿真时间。

### Topic 与 Action 的区别

Action 提供目标接受、执行反馈与最终结果，适合需要监控任务是否完成的场景。Topic 可以直接替换轨迹，但发布成功并不是动作执行成功。

### 关键配置说明

- joints：关节及数组顺序。
- command_interfaces：本例为 position。
- state_interfaces：本例读取 position、velocity。
- trajectory tolerance：运行中允许的跟踪误差。
- goal tolerance：结束时允许的误差。
- goal_time：允许额外达到目标的时间。

不同命令接口要求不同反馈组合，不能把本例 position 配置原样用于 effort 控制。

## 恢复命令转发器

```bash
ros2 control switch_controllers --deactivate trajectory_controller \
  --activate forward_position_controller --strict
```

源码：[ros2_controllers/joint_trajectory_controller/src/joint_trajectory_controller.cpp](https://github.com/ros-controls/ros2_controllers/blob/65db73628ebac1948833f0fbdc60973c74d6cd0e/joint_trajectory_controller/src/joint_trajectory_controller.cpp)、[control_msgs/control_msgs/action/FollowJointTrajectory.action](https://github.com/ros-controls/control_msgs/blob/0a1d341f0960defc40a4b146b671dad74a429f4d/control_msgs/action/FollowJointTrajectory.action)。沿目标回调、轨迹缓冲、采样与容差检查阅读；不要把整段 update 当成只有一次数组赋值。

---

[返回 Humble 总览](/wiki/2023-12-30-ros2-tutorial/1500-0100-humble) · 上一篇：[命令转发与常用控制接口](/wiki/2023-12-30-ros2-tutorial/1500-0100-1200-ming-ling-zhuan-fa-yu-chang-yong-kong-zhi-jie-kou) · 下一篇：[机械臂物理仿真](/wiki/2023-12-30-ros2-tutorial/1500-0100-1400-ji-xie-bi-wu-li-fang-zhen)
