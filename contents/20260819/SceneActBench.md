# SceneActBench

SceneActBench 主要用于评估 VLM Agent 是否能够根据视觉输入，真正理解并操作三维场景，而不仅仅是回答关于 3D 场景的问题。

论文将 3D 场景理解拆分为 5 类任务：

| Task           | 主要测试能力                 | 简单理解                                   |
| -------------- | ---------------------------- | ------------------------------------------ |
| Layout         | Spatial Grounding            | 根据图像恢复场景中物体的位置和朝向         |
| Camera         | Egocentric Spatial Reasoning | 根据场景和图像反推出相机的位置与朝向       |
| Articulated    | Kinematic Reasoning          | 根据物体运动视频恢复可动部件及其关节运动   |
| Reconstruction | Shape Imagination            | 根据多视角图像从零重建完整 3D 场景         |
| Dynamic        | Dynamic Reasoning            | 根据视频恢复场景布局以及多个物体的动态运动 |

------

## 1. Layout

### 任务目标

根据参考图像，将给定的 3D 物体资产放置到正确的空间位置，并恢复其朝向。

测试模型是否能够把：

```
2D 图像中的空间关系
        ↓
转化为
        ↓
3D 世界中的实际位置和方向
```

### 输入

- 一张或多张 reference image
- 场景中涉及的若干 3D object assets
- 所有物体初始时都被放置在统一位置，例如场景原点
- 原始物体的 GT pose 不提供给 Agent

### Agent 需要恢复

对于每个物体，需要估计：

```
位置：
x, y, z

朝向：
yaw
```

即大致恢复：

\[ (x_i, y_i, z_i, \theta_i) \]

### 示例

参考图像中：

```
         Desk

Sofa    Table    Chair

             Bed
```

初始 Blender 场景：

```
Sofa
Table
Chair
Desk
Bed

全部重叠在原点
```

Agent 需要把这些物体重新摆放成和参考图像对应的 3D layout。

### 测试的能力

主要测试：

> **Spatial Grounding**

也就是模型是否能够从视觉观察中恢复 metric-level 的三维空间关系。

它不仅要理解：

```
Chair 在 Table 右边
```

还需要进一步确定：

```
Chair 在什么位置？
距离 Table 多远？
朝哪个方向？
```

### 评价指标

主要使用：

```
ADD-S
```

比较预测物体表面和 GT 物体表面之间的几何距离。

越小越好。

------

## 2. Camera

### 任务目标

场景中的物体位置已经完全正确，Agent 需要根据 reference image 推断拍摄这张图像的相机位姿。

它与 Layout 是一个相反的问题。

Layout：

```
Camera 已知
↓
恢复 Object Pose
```

Camera：

```
Object Pose 已知
↓
恢复 Camera Pose
```

### 输入

- 已经正确构建好的 3D scene
- 一张 reference image
- Camera FOV
- 不提供 camera extrinsics

### Agent 需要恢复

完整的 6-DoF camera pose：

```
位置：
x, y, z

方向：
roll, pitch, yaw
```

也就是：

\[ T_{\text{camera}} \]

### 示例

如果 reference image 中：

```
Sofa 位于画面左边
Table 位于画面中心
Bed 位于画面右后方
```

Agent 需要推断：

```
Camera 应该站在哪里？
Camera 朝向哪个方向？
```

### 测试的能力

主要测试：

> **Egocentric Spatial Reasoning**

即模型是否能够根据自己“看到的画面”，反推出观察者在三维空间中的位置。

相比普通的：

```
left / right / front / behind
```

它要求的是更精确的 metric 3D reasoning。

### 评价指标

两个主要指标：

```
Position Error, PE
Angular Error, AE
```

其中：

- PE：相机位置误差，单位通常是米；
- AE：相机朝向误差，单位是角度。

越小越好。

------

## 3. Articulated

### 任务目标

根据一个可动对象的运动视频，推断：

```
哪些部件可以运动？
如何运动？
围绕哪里运动？
运动范围是多少？
```

然后真正修改 3D object，使其按照推断出的关节结构运动。

### 输入

主要包括：

- 一个处于 closed state 的 GLB 物体
- 一段大约 32 帧的 open → close / articulation reference video

但是不会提供：

```
Semantic Part Label
Joint
Joint Type
Joint Axis
Joint Pivot
Joint Range
```

模型只能看到匿名 mesh，例如：

```
Mesh_001
Mesh_002
Mesh_003
Mesh_004
```

### Agent 需要推断

可以拆成四步：

#### 1. Movable Part Discovery

判断：

```
哪些 mesh 是静态部件？
哪些 mesh 可以运动？
```

例如：

```
Cabinet Body → static
Door → movable
Drawer → movable
```

#### 2. Joint Type

判断运动类型，例如：

```
Door → Revolute Joint
Drawer → Prismatic Joint
```

#### 3. Joint Axis / Pivot

判断运动的位置和方向。

例如对于柜门：

```
Axis = vertical
Pivot = left edge of door
```

#### 4. Motion Range

估计运动范围，例如：

\[ \theta \in [0^\circ, 90^\circ] \]

或者：

```
Drawer translation:
0 → 0.4 m
```

### Agent 最终要做什么

并不是只回答：

```
“这是一个 revolute joint。”
```

而是需要真正操作 Blender 中的 mesh：

```
识别 movable part
        ↓
设置 rigid transformation
        ↓
生成完整运动过程
        ↓
输出多个 3D states
```

### 测试的能力

主要测试：

> **Kinematic Reasoning**

也就是模型能否理解物体的运动结构。

可以进一步拆成：

```
Part decomposition
        ↓
Movable-part discovery
        ↓
Joint type
        ↓
Joint axis / pivot
        ↓
Motion range
```

### 评价指标

主要使用：

```
MPE
Maximum Part Error
```

它不仅考虑：

```
预测出来的运动轨迹是否正确
```

还会惩罚：

```
没有识别出的 movable part
运动范围只恢复了一部分
```

因此它评价的是完整 articulation recovery，而不是单独的 joint classification。

------

## 4. Reconstruction

### 任务目标

给 Agent 多个房间视角，让 Agent 从一个完全空的 Blender scene 开始，重新构建整个 3D 场景。

这是五个任务中对几何生成要求最高的任务之一。

### 输入

- 大约 11 个 calibrated multi-view images
- Empty Blender scene

与 Layout 不同：

> **Reconstruction 不会提供现成的 furniture assets。**

因此 Agent 需要自己构建物体。

### Agent 需要完成

整个任务包含：

```
Object Discovery
        ↓
Object Counting
        ↓
Object Recognition
        ↓
Shape Understanding
        ↓
3D Geometry Construction
        ↓
Metric Scale Estimation
        ↓
Object Placement
        ↓
Appearance / Color Estimation
```

例如图像中存在：

```
Bed
Desk
Chair
Nightstand
Wardrobe
```

Agent 需要自己使用 Blender primitive / mesh operations 等构建这些物体。

### 测试的能力

论文称之为：

> **Shape Imagination**

因为输入图像通常只能看到物体的一部分。

模型必须根据有限观察推测：

```
背面是什么样？
完整几何结构是什么？
物体有多大？
各部件比例是多少？
```

### 与 Layout 的区别

这是最容易混淆的两个任务：

| Layout               | Reconstruction     |
| -------------------- | ------------------ |
| 提供现成 3D assets   | 不提供 assets      |
| 主要解决物体放在哪里 | 既要建模，又要摆放 |
| 几何已经确定         | 几何也需要恢复     |
| Spatial Grounding    | Shape Imagination  |

可以简单理解为：

```
Layout：
“这些家具应该摆在哪里？”

Reconstruction：
“这个房间里有什么，它们长什么样，以及应该摆在哪里？”
```

### 评价指标

使用：

```
F@5%
```

比较生成物体表面与 Ground Truth mesh 的几何一致程度。

------

## 5. Dynamic

### 任务目标

根据动态场景视频，恢复：

```
场景中有什么物体
+
物体初始在哪里
+
哪些物体会运动
+
每个物体怎么运动
```

最终生成一个完整的 animated 3D scene。

### 输入

- Dynamic reference video 中采样出来的 frames
- 对应场景中的 component GLB assets
- Camera 信息

### Agent 需要恢复

至少包括：

```
Static Scene Layout
        +
Initial Object Pose
        +
Moving Object Identification
        +
3D Motion Trajectory
        +
Rotation
        +
Keyframes
```

### 示例

一个交通场景可能包含：

```
Car A → 直线前进
Car B → 变道
Car C → 静止
Car D → 转弯
```

Agent 需要同时理解多个对象：

```
Who moves?
Where does it start?
In which direction?
How fast / how far?
Which objects remain static?
How do different trajectories relate?
```

### 最终输出

Agent 需要生成：

```
Animated GLB / Animated Blender Scene
```

也就是说，恢复的不只是某一帧的 3D scene，而是：

\[ Scene(t) \]

随时间变化的完整场景。

### 测试的能力

主要测试：

> **Dynamic Reasoning**

与 Articulated 不同：

```
Articulated
→ 关注单个物体内部的部件运动

Dynamic
→ 关注场景级多个对象的运动
```

例如：

```
Articulated：
柜门如何绕铰链打开？

Dynamic：
道路上的几辆车分别如何运动？
```

### 评价指标

主要包括：

#### MME — Maximum Mover Error

测试动态物体轨迹恢复是否正确。

重点惩罚：

```
遗漏 moving object
严重错误的 trajectory
```

#### LE — Layout Error

测试静态场景布局是否正确。

因此 Dynamic 同时评价：

```
Scene Layout
+
Object Motion
```

------

# 五个任务之间的关系

可以把 SceneActBench 看成逐渐增加难度的一组 3D reasoning task：

```
                    SceneActBench

                         │
          ┌──────────────┼──────────────┐
          │              │              │
       Static         Articulated     Dynamic
          │              │              │
     ┌────┼────┐         │              │
     │    │    │         │              │
  Layout Camera Reconstruction          │
```

从需要恢复的信息来看：

```
Layout
│
├── Object Position
└── Object Orientation


Camera
│
├── Camera Position
└── Camera Orientation


Articulated
│
├── Movable Part
├── Joint Type
├── Joint Axis
├── Joint Pivot
└── Motion Range


Reconstruction
│
├── Object Identity
├── Object Geometry
├── Object Scale
├── Object Pose
└── Scene Layout


Dynamic
│
├── Scene Layout
├── Moving Object
├── Initial Pose
├── Motion Direction
├── 3D Trajectory
└── Temporal Evolution
```

------

# 五个任务的核心区别

| Task               | 给定什么                  | 需要恢复什么                       |
| ------------------ | ------------------------- | ---------------------------------- |
| **Layout**         | 图像 + 已有 3D assets     | Object pose                        |
| **Camera**         | 图像 + 完整 3D scene      | Camera pose                        |
| **Articulated**    | 物体 + articulation video | Movable parts + joints + motion    |
| **Reconstruction** | Multi-view images         | 完整 object geometry + layout      |
| **Dynamic**        | Video + 3D assets         | Scene layout + object trajectories |

进一步可以记成：

```
Layout
→ 东西应该“放哪”

Camera
→ 我应该“站哪看”

Articulated
→ 东西内部“怎么动”

Reconstruction
→ 东西“长什么样 + 放哪”

Dynamic
→ 整个场景“随时间怎么动”
```

------

# SceneActBench 对 3D Understanding 的能力划分

论文实际上通过这五个 Task，把 3D understanding 拆成了五种相对独立的能力：

```
3D Scene Understanding
│
├── Spatial Grounding
│     └── Layout
│
├── Egocentric Spatial Reasoning
│     └── Camera
│
├── Kinematic Reasoning
│     └── Articulated
│
├── Shape Imagination
│     └── Reconstruction
│
└── Dynamic Reasoning
      └── Dynamic
```

这里比较重要的一点是：

> SceneActBench 不把“3D understanding”视为单一能力，而是分别测试空间、视角、几何、关节和动态等不同层面的三维理解能力。

对于关注 articulation / affordance / interaction 的研究，五个任务中最直接相关的是：

```
Articulated
```

其次是：

```
Dynamic
```

前者关注**物体内部的可运动结构**，后者关注**场景级对象的动态行为**。