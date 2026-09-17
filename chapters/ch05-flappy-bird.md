# 第五章：入门项目实战《Flappy Bird》

## Core Idea
本章通过完整复刻《Flappy Bird》，把前四章的节点、场景、脚本、信号串成一条可运行主线：以“模块拆分 + 场景组合”构建五层架构，以 GameManager 单例做全局信号总线，最终打通“开始 → 得分 → 结束 → 重开”的完整游戏循环。

## Frameworks Introduced
- **模块划分三原则**：功能单一（一个模块只做一类事）、结构独立（松耦合，可单独拔插）、层次清晰（底层对象/中层逻辑控制/顶层 UI 与管理）。
  - When to use: 动手写场景前先画出模块职责表。
- **全局信号总线（GameManager Autoload）**：把跨模块事件集中定义成信号，各场景只连接与自己相关的信号。
  - When to use: 游戏开始、得分、结束等需要 2+ 模块协作的状态变化。
  - How: 单例里 `signal GameStart / GameOver / UpdateScore`；HUD 按钮/小鸟/主场景 connect；Pipes 等触发者只管 emit，不知道谁在听。
- **场景即模块**：每个功能（Bird/Pipes/HUD/背景/主场景）独立成 .tscn，主场景用实例化组合它们。
- **场景继承**：用通用背景模板 bg.tscn 继承出 sky.tscn（Motion Scale 0.2）与 ground.tscn（1.0），共享滚动代码、差异只留贴图与参数。
- **视差滚动（Parallax）**：ParallaxBackground + ParallaxLayer，近景快、远景慢，Mirroring 实现无缝循环。
- **游戏主循环整合**：Timer 定时生成 + Marker2D 出生点 + 动态 instantiate + queue_free 清理 + 状态信号切换。
- **信号三步骤**：定义（signal）→ 连接（connect）→ 发射（emit），配合 await 可等待动画/计时器完成。

## Key Concepts
- **CharacterBody2D + velocity + move_and_slide**：完全由代码控制的物理移动角色。
- **Area2D + body_entered**：感应区域，不阻挡物体，适合碰撞得分/死亡触发。
- **StaticBody2D / RigidBody2D**：静止基石 vs 物理引擎接管。
- **AnimatedSprite2D（SpriteFrames）**：逐帧序列动画；AnimationPlayer：关键帧属性动画。
- **CPUParticles2D / GPUParticles2D**：CPU 少量简单粒子 vs GPU 海量粒子。
- **CanvasLayer + Control 容器**：UI 独立渲染层；Margin/VBox 容器自动排版，避免硬编码坐标。
- **LabelSettings 资源复用**：字体/字号/颜色/描边存成 .tres 供多处复用。
- **VisibleOnScreenNotifier2D**：screen_entered/screen_exited 信号，判断对象是否离开屏幕以销毁。
- **Autoload（单例）**：Project Settings > Autoload 注册，任何脚本可直接访问 GameManager。
- **Parallax2D（4.6 新节点）**：合并旧双节点；有 Camera2D 用 Scroll Scale，无则用 Autoscroll，都要设 Repeat Size 实现无限滚动。

## Mental Models
- Think of GameManager as 导演/总指挥：全局数据归它管，各模块只向它汇报，不互相找。
- Think of 模块协作 as “相忘于江湖，却感应于信号”：小鸟不知道 HUD 存在，只发“我撞了”。
- Think of 2D 横版飞行 as 坐标系欺骗：小鸟位置不动，世界向左移，让玩家产生前进错觉。
- Think of UI 布局 as 磁铁/相框：容器自动吸附排列，锚点决定相对父控件的自适应行为。
- Think of 场景继承 as 模板派生：父场景共享代码，子场景改参数与贴图即可。

## Anti-patterns
- **强耦合直调**：小鸟直接调用 HUD 更新分数 → 一改全乱；全部改走 GameManager 信号。
- **UI 硬编码坐标**：不同分辨率下排版错乱；用 CanvasLayer + 容器 + 锚点。
- **管道只生成不清理**：对象无限累积导致卡顿；离屏用 VisibleOnScreenNotifier2D 触发 queue_free，重开时清空 PipesGroup。
- **金币未等动画直接消失**：收集反馈生硬；用 `await animation_player.animation_finished` 再删节点。
- **在 _process 写物理**：应把重力、碰撞与移动放进物理回调或 move_and_slide 流程，保证稳定。
- **把鸟做成不会死的静态精灵**：必须同时有碰撞形状 + 状态变量（is_dead）+ 分组（bird），别让非玩家物体触发逻辑。

## Code Examples
障碍模块：碰撞→信号、离屏→清理、金币→动画后删除：
```gdscript
extends Node2D

const SPEED = -150
var passed = false

@onready var coin: Area2D = $Coin
@onready var pipe_bottom: Area2D = $PipeBottom
@onready var pipe_top: Area2D = $PipeTop
@onready var animation_player: AnimationPlayer = $Coin/AnimationPlayer
@onready var notifier: VisibleOnScreenNotifier2D = $VisibleOnScreenNotifier2D

func _ready() -> void:
    pipe_bottom.body_entered.connect(on_pipe_body_entered)
    pipe_top.body_entered.connect(on_pipe_body_entered)
    coin.body_entered.connect(on_coin_body_entered)
    notifier.screen_exited.connect(queue_free)

func on_pipe_body_entered(body: Node2D) -> void:
    if body.is_in_group("bird") and not body.is_dead:
        GameManager.GameOver.emit()

func on_coin_body_entered(body: Node2D) -> void:
    if body.is_in_group("bird") and not passed:
        passed = true
        GameManager.UpdateScore.emit()
        animation_player.play("coin")
        await animation_player.animation_finished
        coin.queue_free()

func _process(delta: float) -> void:
    position.x += delta * SPEED
```
- **What it demonstrates**: 内建信号→自定义信号传递、离屏自动回收、await 延迟删除。

主场景动态生成管道：
```gdscript
const GAP = 180
var pipes_scene: PackedScene = preload("res://pipes/pipes.tscn")
@onready var timer: Timer = $Timer
@onready var spawn_point: Marker2D = $SpawnPoint
@onready var pipes_group: Node2D = $PipesGroup

func new_pipes() -> void:
    var pipes = pipes_scene.instantiate()
    var pos_y = randf_range(spawn_point.position.y - GAP, spawn_point.position.y + GAP)
    pipes.global_position = Vector2(spawn_point.position.x, pos_y)
    pipes_group.add_child(pipes)
```
- **What it demonstrates**: PackedScene preload + instantiate + Marker2D 出生点 + 随机高度 + 容器管理。

## Reference Tables
| 物理节点 | 行为 | 控制 | 示例 |
|---|---|---|---|
| StaticBody2D | 静止 | 无需控制 | 地面/墙 |
| RigidBody2D | 受力运动 | 物理引擎 | 掉落物 |
| CharacterBody2D | 可移动+碰撞 | 代码控制 | 小鸟/玩家 |
| Area2D | 感应不阻挡 | 信号 | 得分/死亡触发 |

| 协作需求 | 触发方 | 响应方 |
|---|---|---|
| 游戏开始 | HUD 按钮 emit GameStart | Bird 激活、Timer 启动 |
| 得分 | Pipes 金币 emit UpdateScore | HUD 更新、音效播放 |
| 游戏结束 | Bird/Pipes emit GameOver | 全模块停止、UI 显示 |

| 动画方案 | 节点 | 适用 |
|---|---|---|
| 逐帧动画 | AnimatedSprite2D | 鸟翅膀、金币旋转 |
| 关键帧动画 | AnimationPlayer | 金币放大淡出、UI 动效 |

## Worked Example
完整信号流：主场景放 Sky/Ground/Bird/HUD + SpawnPoint/Timer/Ceil/Floor/PipesGroup。玩家点 Start → `GameManager.GameStart.emit()` → 小鸟 `is_dead=false` 并开启粒子、Timer 每 2 秒 instantiate 一组 Pipes。小鸟穿金币 → Coin 触发 `UpdateScore` → GameManager.score+1、HUD 刷新文本、音效播放。撞管道或边界 → `GameOver.emit()` → Timer 停、小鸟死亡、Pipes 清空重开。每个模块只知道自己关心的事件，主流程由信号串联。

## Key Takeaways
1. 动手前先画模块职责与协作表：功能单一、结构独立、层次清晰。
2. 跨模块状态用 GameManager 单例 + 信号总线，绝不让场景互相直调。
3. 动态生成对象必须配套离屏/重置清理，否则迟早卡死。
4. UI 放 CanvasLayer，布局用容器和锚点，不写死坐标。
5. 背景滚动用 ParallaxLayer 的 Motion Scale 分层，Mirroring 无缝循环。
6. await 信号可以优雅地“等动画播完再删除节点”。
7. 场景继承让“同逻辑不同外观”的模块只写一次代码。

## Connects To
- **Ch 3/4**: 节点/场景/信号/GDScript 全部在此实战。
- **Ch 6**: 单信号总线升级为更大项目的组件化架构。
- **Ch 9**: is_dead 这类状态开关将发展为正式的状态机。
- **Ch 14**: 本章《Flappy Bird》项目将作为强化学习训练环境被改造。
