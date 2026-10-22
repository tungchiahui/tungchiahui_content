---
title: World-Model与移动机器人路径规划
---

## World Model（世界模型）与移动机器人路径规划

### 从“预测未来”到“选择动作”

**组会汇报｜论文调研与选题阶段**

> 核心问题：World Model（世界模型）到底是在“生成未来”，还是在“做规划”？

---

## 什么是 World Model（世界模型）？

### 一个适合当前学习式 World Model（世界模型）的概括

```text
真实观测 o_t
   │
   ▼
状态表示 / 状态估计
   │
   ▼
内部状态 z_t
   │  + 动作 a_t
   ▼
Dynamics Model（动力学模型）
   │
   ▼
未来状态 z_{t+1}, z_{t+2}, ...
```

World Model（世界模型）的核心通常可以概括为：

- **World State Representation（世界状态表示）**：把当前环境表示成可供预测的状态；
- **Action-conditioned Dynamics（动作条件动力学）**：预测执行动作后，世界如何变化。

根据任务需要，还可以额外加入：

- Decoder（解码器）：把内部状态还原成图像、深度、语义等；
- Renderer（渲染器）：从显式三维场景生成视觉观测；
- Reward / Value Head（奖励 / 价值预测头）；
- Task Head（任务预测头）。

**结论：World Model（世界模型）的共同点不是“生成视频”，而是“建立能够预测动作后果的内部世界”。**

> “状态表示 + 动力学预测 + 可选解码/任务头”适合作为分析框架，但不是所有论文都采用同一种结构。

### 结构实例：DINO-WM

![DINO-WM：观测编码、动作条件特征预测与规划代价](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/10/23/1792660599873-879baf21.webp)

读图重点：左侧 DINOv2 将观测编码成状态表示，中间动力学模型预测未来特征，右侧 Planning loss 比较预测特征与目标特征，并用于优化候选动作。图中展示了世界模型及使用它的规划流程。

图片来源：[DINO-WM 官方项目的模型结构图](https://dino-wm.github.io/mfiles/figs/model_arch.png)，对应[论文图 2](https://arxiv.org/html/2411.04983v2#S3.F2)。

---

## World Model（世界模型）不等于 Renderer（渲染器）、Simulator（模拟器）和 Planner（规划器）

```text
                 ┌─────────────────────────┐
Observation ─────▶│ World Model（世界模型） │
                 │                         │
                 │ 世界表示 + 动力学预测   │
                 └───────────┬─────────────┘
                             │
                      Future State
                       未来世界状态
                        ┌────┴────┐
                        │         │
                        ▼         ▼
             Renderer（渲染器）  Planner / Policy
                        │       （规划器 / 策略）
                        ▼         │
                   RGB / Depth    ▼
                               Action（动作）
```

### Renderer（渲染器）

解决的是：

> **“这个内部世界从某个视角看起来是什么样？”**

例如：

```text
3D Gaussian（三维高斯场景）
       │
       ▼
Gaussian Renderer（高斯渲染器）
       │
       ▼
RGB 图像
```

Renderer（渲染器）本身通常不负责时间演化，因此：

**Renderer（渲染器） ≠ World Model（世界模型）**

### Simulator（模拟器）

解决的是：

> **“执行动作以后，世界会怎样变化？”**

它既可以是：

- Physics Engine（物理引擎），如 PyBullet、MuJoCo；
- Learned Dynamics Model（学习式动力学模型），如 Transformer（变换器）、Diffusion Model（扩散模型）；
- 两者结合的 Hybrid Model（混合模型）。

### Planner（规划器） / Policy（策略）

解决的是：

> **“根据这些预测，我最后应该执行什么动作？”**

因此需要区分：

- **World Model Core（世界模型本体）**：负责表示和预测；
- **World-Model-Based System（基于世界模型的完整系统）**：可能把规划器、策略、解码器等一起画进去。

---

## World Model（世界模型）怎样参与决策？

```text
在线规划                           策略学习

候选动作                           World Model
  │                                  │
  ▼                                  ▼
World Model                       世界表示 / 想象经验
  │                                  │
  ▼                                  ▼
预测未来                           Policy（策略）
  │                                  │
  ▼                                  ▼
MPC / CEM                          Action（动作）
  │
  ▼
Action（动作）

NWM、DINO-WM                     GWM、X-MOBILITY
```

**World Model（世界模型）既可以在执行时预测候选动作的未来，也可以主要用于学习更好的 Policy（策略）。**

因此，**“使用 World Model（世界模型）”并不等于“部署时一定进行 MPC（模型预测控制）搜索”。**

### 在线规划实例：NWM 的候选轨迹评价

![NWM：候选轨迹对应的未来图像及目标匹配损失](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/10/23/1792660602301-b26052bc.webp)

读图重点：P1、P2、P3 是三个候选轨迹，右侧展示各自预测的末帧。NWM 用预测末帧与目标图像的 LPIPS 距离评分，图中 P3 的损失最低，为 0.369，因此获得优先选择。

图片来源：[Navigation World Models，论文图 7](https://arxiv.org/html/2412.03572v1#S4.F7)。

### 策略学习实例：GWM 的三维世界建模

![GWM：图像重建三维高斯场景、潜在动力学预测与未来场景解码](https://cdn.tungchiahui.cn/tungwebsite/assets/images/2026/10/23/1792660603436-7edbe0ce.webp)

读图重点：三维高斯用于表示场景，3D VAE 将其压缩为潜在特征，动作条件动力学模型预测未来，再解码得到未来高斯场景。

**怎样用于决策：** 在论文的模仿学习设置中，GWM 将扩散模型第一次去噪后的中间特征交给 BC-Transformer 或 Diffusion Policy，由策略输出动作序列。上图展示的是未来场景预测链路，策略模块没有画在图中。

**与 NWM 的区别：** 上面的 NWM 实例利用未来图像评价和选择候选轨迹；这里的 GWM 利用世界建模得到的特征支持策略学习，执行时由策略生成动作。论文还探索了将 GWM 作为学习式模拟器，预测状态和奖励以支持强化学习训练。

图片来源：[GWM 官方项目的方法图](https://gaussian-world-model.github.io/resources/method/pipeline.png)。策略接口参见[论文 §3.3](https://arxiv.org/html/2508.17600v1#S3.SS3)。

### 从 WM 到 WAM

近期的 WAM（World Action Model，世界动作模型）进一步把 **Future Prediction（未来预测）** 与 **Action Generation（动作生成）** 结合起来：

```text
Observation / Goal
（观测 / 目标）
       │
       ▼
WAM（世界动作模型）
    ┌──┴──┐
    ▼     ▼
未来世界   Action（动作）
```

| 类型 | 主要能力 |
|---|---|
| **WM（世界模型）** | 预测动作执行后的未来 |
| **VLA（视觉-语言-动作模型）** | 根据视觉和任务指令直接生成动作 |
| **WAM（世界动作模型）** | 将未来预测与动作生成联合建模 |

> **WAM 可以看作 World Model（世界模型）与机器人 Policy（策略）进一步融合的一种近期趋势，而不是简单的“WM + VLA”。**

例如 DreamZero 同时建模 Future Visual State（未来视觉状态）和 Robot Action（机器人动作）。

---

## 四篇论文中，决策真正使用什么？

| 论文 | 世界表示 | 预测什么 | 决策实际使用什么 | 执行时搜索动作？ |
|---|---|---|---|---|
| **NWM** | VAE Latent（潜在表示） | 未来视觉表示 | 解码后的预测图像与目标图像的视觉距离 | **是，MPC / CEM** |
| **DINO-WM** | DINOv2 Feature（特征） | 未来视觉特征 | **预测特征与目标特征之间的距离** | **是，MPC / CEM** |
| **GWM** | 3D Gaussian Latent（三维高斯潜在表示） | 未来三维场景 | **世界模型中间特征 → Policy（策略）** | **主要不是** |
| **X-MOBILITY** | Latent World State（潜在世界状态） | 未来潜在状态 | **当前世界状态 + Route（路线）→ Policy（策略）** | **不是主要方式** |

> 这说明：**World Model（世界模型）能生成什么**，和 **Planner / Policy（规划器 / 策略）真正使用什么进行决策**，是两个不同的问题。

---

## World Model（世界模型）与移动机器人路径规划

```text
       传统方法                 World Model（世界模型）

人工建立状态 / 地图 / 动力学      从数据学习世界表示与状态转移
         │                              │
         ▼                              ▼
     Planner（规划器）              预测动作后果
         │                              │
         ▼                              ▼
     Action（动作）              Planner / Policy
                                 （规划器 / 策略）
                                        │
                                        ▼
                                    Action（动作）
```

### 对移动机器人值得继续研究的问题

**1. 预测什么？**

- RGB 图像；
- Feature（特征）；
- BEV（鸟瞰图）；
- 3D Gaussian（三维高斯）；
- Latent State（潜在状态）。

**2. 怎样用于规划？**

- Online Planning（在线规划）：World Model Rollout（世界模型滚动预测） + MPC / CEM；
- Policy Learning（策略学习）：利用世界模型表示或想象经验训练 Policy（策略）。

**3. 真正的困难是什么？**

- 长时预测误差是否会累积；
- 如何保证几何、碰撞和可通行性约束；
- 什么表示既容易预测，又真正有利于规划。

> **对移动机器人而言，关键不一定是生成更真实的未来，而是学习“对规划真正有用”的未来表示。**

---

## 总结

### 当前认识

1. **World Model（世界模型）的核心是表示世界并预测动作后的未来，而不是单纯生成视频。**
2. **不同方法的关键差异，在于“预测什么表示”以及 Planner / Policy（规划器 / 策略）如何使用这种表示。**
3. **World Model（世界模型）不是路径规划器本身，而是为规划和决策提供可预测的内部世界。**

---

## 参考论文与官方项目

### 重点论文

1. **Navigation World Models（NWM）**  
   Amir Bar et al., CVPR 2025  
   - 论文：https://openaccess.thecvf.com/content/CVPR2025/papers/Bar_Navigation_World_Models_CVPR_2025_paper.pdf  
   - 官方代码：https://github.com/facebookresearch/nwm

2. **DINO-WM: World Models on Pre-trained Visual Features enable Zero-shot Planning**  
   Gaoyue Zhou et al., ICML 2025  
   - 论文：https://proceedings.mlr.press/v267/zhou25t.html  
   - 官方项目：https://dino-wm.github.io/

3. **GWM: Towards Scalable Gaussian World Models for Robotic Manipulation**  
   Guanxing Lu et al., ICCV 2025  
   - 论文：https://arxiv.org/abs/2508.17600  
   - 官方项目：https://gaussian-world-model.github.io/

4. **X-MOBILITY: End-To-End Generalizable Navigation via World Modeling**  
   Wei Liu et al., ICRA 2025  
   - 论文：https://arxiv.org/abs/2410.17491  
   - 官方项目：https://nvlabs.github.io/X-MOBILITY/

### 补充论文

1. **World Action Models are Zero-shot Policies（DreamZero）**  
   Seonghyeon Ye et al., 2026  
   - 论文：https://arxiv.org/abs/2602.15922  
   - 官方项目：https://dreamzero0.github.io/
