---
title: "Joint-Trajectory-Controller"
---

> 轨迹是时间轴上的关节目标，不只是一次位置数组赋值。

## 示例 1：加载并切换控制器

保持自己的双关节实例运行，另一终端 source 学习工作区：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_control_jazzy_ws/install/setup.bash
```

```bash
ros2 run controller_manager spawner trajectory_controller --inactive \
  --param-file $(ros2 pkg prefix jazzy_control_hardware)/share/jazzy_control_hardware/config/controllers.yaml
ros2 control switch_controllers --deactivate forward_position_controller \
  --activate trajectory_controller --strict
ros2 control list_controllers
```

spawner 使用 --inactive 完成加载与配置；switch 在同一个请求里释放旧接口并激活新控制器。不要让两个普通控制器争用同一 position 命令接口。

## 示例 2：Action 发送目标

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
Goal finished with status: SUCCEEDED
```

成功表示符合配置中的轨迹和目标容差，不表示设备没有误差。教学硬件有跟随延迟，因此配置了允许误差与额外到达时间。

## 示例 3：Topic 发送两段轨迹

```bash
ros2 topic pub --once /trajectory_controller/joint_trajectory \
  trajectory_msgs/msg/JointTrajectory \
  "{joint_names: [joint1, joint2], points: [
    {positions: [0.3, -0.2], time_from_start: {sec: 2}},
    {positions: [0.0, 0.0], time_from_start: {sec: 5}}
  ]}"
```

time_from_start 是相对整条轨迹起点，第二点表示开始后 5 秒，不是再等 5 秒。时间应递增，数组顺序与 joint_names 一致。

Action 提供接受、反馈和结果；Topic 发布成功不能证明执行完成。指定绝对 header 时间时，要确保发送端与控制器使用相同时间域；仿真中尤其要检查 use_sim_time。

## 参数与恢复

command_interfaces 决定输出类型，state_interfaces 决定所需反馈，constraints 定义轨迹误差与目标误差。换成 velocity 或 effort 时，还要核对对应的接口组合要求。

```bash
ros2 control switch_controllers --deactivate trajectory_controller \
  --activate forward_position_controller --strict
```

源码：[ros2_controllers/joint_trajectory_controller/src/joint_trajectory_controller.cpp](https://github.com/ros-controls/ros2_controllers/blob/aacd842600a09d556b983ac3d53a0983e9ebcbb1/joint_trajectory_controller/src/joint_trajectory_controller.cpp)。沿目标回调、轨迹缓冲、采样和容差判断阅读。

---

[返回 Jazzy 总览](/wiki/2023-12-30-ros2-tutorial/1500-0200-jazzy) · 上一篇：[命令转发与控制模式](/wiki/2023-12-30-ros2-tutorial/1500-0200-1200-ming-ling-zhuan-fa-yu-kong-zhi-mo-shi) · 下一篇：[Harmonic机械臂物理仿真](/wiki/2023-12-30-ros2-tutorial/1500-0200-1400-harmonic-ji-xie-bi-wu-li-fang-zhen)
