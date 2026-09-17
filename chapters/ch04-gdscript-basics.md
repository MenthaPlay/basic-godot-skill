# 第四章：GDScript编程基础与实践

## Core Idea
本章为节点“注入灵魂”：学会挂载脚本、用变量/类型/条件/函数/数组组织逻辑、处理输入，并通过信号让对象松耦合协作；最后用“准备→初始化→登场→主循环”的运行模型解释 Godot 代码的先后顺序。

## Frameworks Introduced
- **挂载脚本 = 创建节点的子类**：`extends Node2D` 让脚本继承节点全部能力，再扩展自定义变量与函数；Inspector 里能用的属性都能在代码中用。
  - When to use: 给某个节点增加行为逻辑时。
  - How: 编辑器附加脚本 → Godot 编译为运行时类 → 场景加载时节点成为自定义类的实例 → 自动调用 `_ready/_process` 等回调。
- **三个回调的分工**：`_ready()` 一次性初始化（子节点就绪后调用一次）；`_process(delta)` 每帧调用（视觉/输入逻辑）；`_physics_process(delta)` 固定 60 次/秒（物理、碰撞、移动）。
- **@export 与 @onready**：`@export` 把参数暴露到 Inspector 并存入 .tscn；`@onready` 在节点就绪后引用子节点，避免空引用。
  - How: 两者不能同时修饰同一变量；拆成“一个暴露参数 + 一个内部引用”。
- **信号（Signal）广播机制**：发送者 `emit` 广播，接收者提前 `connect` 并注册回调；谁感兴趣谁响应。
  - When to use: 按钮点击、碰撞、死亡等跨对象通知，避免直接互调造成的强耦合。
- **输入双通道**：`_input(event)` 事件驱动，适合点击/跳跃等一次性操作；`Input.is_action_pressed()` 主动查询，适合按住移动/拖拽。
- **Godot 运行生命周期**：SceneTree + 主场景解析 → 实例化触发 `_init()` → 节点入树后先赋 `@onready` → 子节点 `_ready` 先于父节点 → 进入主循环。

## Key Concepts
- **GDScript**：Godot 原生脚本语言，Python 风格缩进、与节点/信号深度集成。
- **脚本资源（.gd）**：Script 是 Resource 的一种，挂载后成为节点的扩展类。
- **变量作用域**：函数内局部变量、脚本顶部成员变量、Autoload 全局数据三层。
- **类型提示**：`var speed: int` / `:=` 推断；提高可读性、静态检查与编辑器补全。
- **Input Map**：在 Project Settings 定义动作名（如 click），再绑定按键/鼠标；代码里用动作名判断，不写死键位。
- **Group**：给节点贴“标签”（如 move），用于批量筛选与协作。
- **Array 常用 API**：`append/insert/erase/size/has/pick_random`。
- **`$` 节点路径**：`$Sprite2D` 引用当前节点子节点；路径变化后应用 `@onready var` 代替。
- **PackedScene / .tscn**：场景是可保存、可实例化、可继承的节点组合蓝图。
- **`_init()` 与 `_ready()`**：`_init` 创建时执行、子节点尚不存在；`_ready` 入树后执行、可安全访问子节点与信号。

## Mental Models
- Think of 节点 as 木偶、脚本 as 提线：编辑器摆好骨架，脚本决定“何时做什么”。
- Think of 挂载脚本 as 给汽车装自动驾驶：不改造发动机（内置节点功能），只加智能逻辑。
- Think of 信号系统 as 广播电台：发送者 emit 播放，接收者 connect 调频，互不相识也能协作。
- Think of 场景 as 蓝图/类：.tscn 实例化出无数独立个体。
- Think of 生命周期 as 一场演出：先搭舞台（加载资源）→ 演员后台化妆（_init）→ 上台站位（@onready、子级 _ready）→ 幕布拉开（主循环）。

## Anti-patterns
- **直接互调对方函数**（`player.take_damage()` 满天飞）：强耦合、改一处全乱；改用信号广播。
- **把键位写死在代码里**：应定义 Input Map 动作名，方便换键与手柄映射。
- **在 `_init` 里访问子节点**：子节点还没创建，必然空引用；初始化子节点引用放 `@onready`/`_ready`。
- **`@export` + `@onready` 同用**：报错或行为不稳定。
- **`$Sprite2D` 硬编码路径且无类型**：节点改名即失效；用带类型的 `@onready var sprite: Sprite2D = $Sprite2D`。
- **物理逻辑写在 `_process`**：帧率不稳定导致物理行为不一致；物理放 `_physics_process`。
- **旋转/移动不乘 delta**：速度随帧率漂移。

## Code Examples
帧率无关旋转 + 条件停止 + 函数封装：
```gdscript
extends Node2D

@export var player_speed: float = 2 * PI   # 每秒 1 圈
@onready var sprite_2d: Sprite2D = $Sprite2D

func _process(delta: float) -> void:
    rotation += player_speed * delta
    sprite_2d.rotation += player_speed * delta
    stop_rotate(3)

func stop_rotate(count: int) -> void:
    if rotation / (2 * PI) > count:
        print("rotate %s times" % count)
        set_process(false)
```
- **What it demonstrates**: @export 参数化、@onready 节点引用、delta 帧率无关、条件与函数封装。

信号解耦（Button 控制所有 move 分组角色）：
```gdscript
@onready var button: Button = $Button

func _ready() -> void:
    for child in get_children():
        if child.is_in_group("move"):
            button.pressed.connect(child.on_pressed)
```
- **What it demonstrates**: connect 信号 + Group 批量筛选，发信号者不认识接收者也行。

## Reference Tables
| 变量修饰符 | 行为 | 何时使用 |
|---|---|---|
| `@export` | Inspector 可见可改，存入场景 | 速度/血量/颜色等实例参数 |
| `@onready` | 入树后赋初值 | 引用子节点、依赖场景状态 |

| 输入方式 | 触发 | 适合 |
|---|---|---|
| `_input(event)` | 每次输入事件 | 点击、跳跃、一次性操作 |
| `Input.is_action_pressed()` | 每帧主动查询 | 持续移动、按住行为 |

| 生命周期函数 | 时机 | 用途 |
|---|---|---|
| `_init()` | 实例化时 | 基础数据初始化（不能访问子节点） |
| `_ready()` | 入树、子节点就绪后 | 初始化、连信号 |
| `_process(delta)` | 每帧 | 视觉/输入逻辑 |
| `_physics_process(delta)` | 固定频率 | 物理、重力、碰撞移动 |

## Worked Example
作者从“自动旋转图标”升级为“点击按钮控制旋转”：给所有 Player 加 move 分组并实现 `on_pressed()` 开关；在 level 场景放一个 Button，`_ready` 里遍历 `get_children()`，把 `button.pressed` 信号 connect 到每个 move 分组角色的 `on_pressed`。最终点击一次全员旋转、再点一次停止——两个场景互不感知对方内部实现，只通过信号协作。

## Key Takeaways
1. 脚本挂到节点 = 继承节点类并扩展；Inspector 属性都可代码访问。
2. `_ready` 初始化、`_process` 每帧、物理逻辑固定用 `_physics_process`。
3. 成员变量放脚本顶部；参数用 @export，节点引用用 @onready，两者不混用。
4. 输入动作走 Input Map + 动作名，不写死键位。
5. 跨对象通知用信号，避免直接互调造成的强耦合。
6. 生命周期顺序：_init → @onready → 子 _ready → 父 _ready → 主循环。

## Connects To
- **Ch 2**: OOP、事件驱动在 GDScript 落地。
- **Ch 3**: 挂载脚本是给第三章的场景/节点加行为。
- **Ch 5/6**: 变量、信号、组将在 Flappy Bird 与 Battle Tank 项目中大规模实战。
- **Ch 16**: 本章的类/继承/场景蓝图是 OOAD 与设计模式的起点。
