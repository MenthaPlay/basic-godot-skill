# 第三章：Godot游戏引擎快速入门

## Core Idea
本章带读者从安装配置 Godot 到做出第一个“会旋转的图标”场景，建立两个最核心的引擎心智模型：节点是功能积木、场景是可复用的节点组合；同时掌握资源路径与引用复用机制。

## Frameworks Introduced
- **节点 + 场景 = 舞台与演员**：场景是节点树组织的可保存/可实例化的模块；节点是最小功能单位，父子关系构成场景树。
  - When to use: 设计任何游戏元素时，先想“这个对象应该由哪些节点组合”，而不是写一个巨大脚本。
  - How: 每个场景一个根节点，功能拆到子节点；子场景通过 Instance Child Scene 复用，就像 OOP 的类与实例。
- **三类节点心智**：2D 节点（蓝色）负责 2D 世界、3D 节点（红色）、UI/Control 节点（绿色）。
- **场景复用（Packed Scene）**：`.tscn` 保存的是节点树结构；实例化只是建立引用，不复制内容，修改原场景所有实例自动更新。
- **资源统一引用**：所有图片/音频/场景/脚本都是 Resource 对象，默认在多个节点间共享引用。
- **动画插值制作**：AnimationPlayer 只需在时间轴上放关键帧，Godot 自动补间。
- **编辑器四区分工**：左侧结构、右侧属性、中间舞台、底部调试。

## Key Concepts
- **Node / Scene**：节点是最基本单位；场景是节点组成的树，保存为 `.tscn`。
- **Node2D**：一切 2D 节点的父类，提供 position/rotation/scale（Transform），可作容器根节点。
- **Sprite2D**：继承 Node2D，用 Texture 显示图像。
- **AnimationPlayer**：继承 Node 的时间轴属性控制器，控制任何节点属性随时间变化。
- **继承链**：Object → Node → Node2D → Sprite2D；每个编辑器节点都是某个类的实例。
- **project.godot**：项目入口配置文件；含它的目录才是有效的 Godot 项目。
- **res:// 与 user://**：res:// 指项目资源路径（只读资源）；user:// 是运行时用户数据（存档/设置）。
- **Renderer 模式**：Forward+（桌面高质量）、Mobile（手机）、Compatibility（低端/网页）。
- **Make Unique / Local to Scene / Save As**：控制资源独立副本、实例局部资源、保存为 .tres 的三种方式。

## Mental Models
- Think of 场景 as 舞台、节点 as 演员：演员有不同能力（节点类型），组合成完整节目（功能模块）。
- Think of 场景文件 as 女娲造人的“模型”/OOP 类：实例化就是批量造出独立个体。
- Think of 资源 as 共享的道具：同一个纹理被多个精灵引用，不复制多份；改一处全项目生效。
- Think of 子节点位置 as 相对父节点：移动父节点，所有子节点一起动。
- Think of Godot 开发 as “分而治之”：把游戏拆成可复用场景，再组合出关卡。

## Anti-patterns
- **把一切塞进一个场景**：关卡场景变成巨复杂节点树，难以维护；敌人、UI、玩家都应独立成场景再实例化。
- **使用中文/空格/特殊符号的路径**：部分系统下 Godot 读取资源失败；项目名、文件名用清晰英文。
- **多个项目嵌套在同一个文件夹**：每个 Godot 项目必须独立文件夹且创建时为空。
- **忘记资源共享特性就复制节点**：默认改一个节点上的共享资源会影响所有使用者；需独立副本时用 Make Unique / Local to Scene。
- **直接在压缩包内运行 Godot**：务必先解压再启动，否则项目管理器无法正常读取资源。

## Code Examples
场景结构（不是代码，是树）：
```
Player (Node2D)          # 掌管位置与旋转
├── Sprite2D             # 显示图像
└── AnimationPlayer      # 导演：时间轴上安排动作
```
- **What it demonstrates**: Node2D 容器 + Sprite2D 视觉 + AnimationPlayer 控制，是 Godot 对象的最小分工样板。

## Reference Tables
| 区域 | 面板 | 用途 |
|---|---|---|
| 顶部 | 工具栏/菜单 | 运行、保存、设置、调试 |
| 中央 | 2D/3D/Script | 编辑场景或代码 |
| 左侧 | Scene / FileSystem | 节点树与资源管理 |
| 右侧 | Inspector | 编辑选中节点属性、绑信号 |
| 底部 | Output / Debugger | 输出与调试 |

| 路径前缀 | 含义 | 示例 |
|---|---|---|
| `res://` | 项目资源路径 | `res://assets/icon.svg` |
| `user://` | 运行时用户数据 | 存档、设置文件 |

| 资源类型 | 后缀 | 用途 |
|---|---|---|
| 图像 | .png/.svg/.webp | 精灵、UI、背景 |
| 音频 | .wav/.ogg/.mp3 | 音效与 BGM |
| 场景 | .tscn | 可运行/可实例化模块 |
| 脚本 | .gd | GDScript 逻辑 |
| 字体 | .ttf/.otf | 文本样式 |

## Worked Example
第一个动画场景的完整路径：新建项目 Demo（Renderer 默认 Forward+）→ 创建 2D 根节点 Node2D 改名 Player → 添加子节点 Sprite2D 并把 icon.svg 拖入 Texture → 添加 AnimationPlayer → 新建动画 rotate → 在 Sprite2D 的 Rotation Degrees 属性上打关键帧（0 秒为 0°、1 秒为 360°）→ 设置 AutoPlay + Loop → 运行当前场景。观察结果：图标绕父节点公转且自身旋转，验证子节点继承父节点空间变换。

## Key Takeaways
1. 场景是节点树，节点是功能积木；子场景可实例化复用，等同 OOP 类与实例。
2. 编辑器口诀：左侧结构、右侧属性、中间舞台、底部调试。
3. 子节点位置相对于父节点，移动父节点整组动。
4. 资源默认共享引用，需要独立副本时用 Make Unique / Local to Scene。
5. 所有运行时资源走 res://，动态存档走 user://。
6. 动画 = 关键帧 + 自动补间，AnimationPlayer 是时间轴导演。

## Connects To
- **Ch 2**: Node2D/Sprite2D/AnimationPlayer 对应坐标、精灵与插值概念。
- **Ch 4**: “节点都是类的实例”将进入 GDScript 类、继承与脚本挂载。
- **Ch 5**: 用本章场景/资源技能开始完整项目《Flappy Bird》。
