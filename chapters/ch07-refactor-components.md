# 第七章：项目优化重构

## Core Idea
本章把第六章“能跑的坦克原型”重构成组件化、可扩展的架构：用“组合优于继承”把功能拆成乐高式组件；用 Collision Layer/Mask 取代组标签判断；用 GameManager 信号中枢统管全局；用多个 Manager 系统编排波次、道具与特效。

## Frameworks Introduced
- **组合优于继承（Composition over Inheritance）**：把可复用能力做成独立组件场景，按需挂载。
  - Why: Godot 场景不支持多重继承，层级越深越难改；“给坦克加雷达但不给玩家”在继承里很别扭，组合只加一个组件即可。
  - Component set: HurtBoxComponent（承伤）、HitBoxComponent（伤害）、HealthComponent（血量）、WeaponComponent（武器）、DetectComponent（探测）、TrailComponent（轨迹）。
- **碰撞层/遮罩（Layer & Mask）取代组标签**：Layer = 我是谁，Mask = 我检测谁；物理引擎从底层过滤无效碰撞。
  - Why: 组标签判断时引擎仍做了无效碰撞检测、逻辑散落且无法可视化；Layer/Mask 在 Inspector 中直观可调。
  - How: 玩家子弹 Layer 2 / Mask 1，敌方子弹 Layer 1 / Mask 2，双方子弹不与自己人碰撞。
- **承伤/伤害接口约定**：HurtBox `get_hurt(damage)` → 扣 armor 后 emit `get_damage`；HitBox 命中后调用目标的 `get_hurt`。
  - Key: `has_method()` 检查再调用，让碰撞只依赖“能力”不依赖“身份”，墙体没实现接口就自然免疫。
- **面向能力解码（面向对象设计变体）**：HitBox 不判断“是不是敌人”，只调用 `get_hurt`；谁想受伤谁实现接口。
- **血量组件（HealthComponent）**：纯逻辑 Node，管理 max/current HP，emit `health_changed(percent)` 与 `died`。
- **探测组件（DetectComponent）**：用 `has_overlapping_areas()`/`get_overlapping_areas()` 每帧轮询，而不是一次性 area_entered；配合 Mask 只探测玩家层。
- **武器组件（WeaponComponent）**：封装冷却、子弹生成、炮管 look_at、后座/火光动画与射击音效；带 `upgrade()` 缩短冷却。
- **Await 链（Await Chain）**：函数 A 内部有 await，调用 A 的函数 B 也必须 await；否则 A 未完成 B 就继续，波次/结算顺序会错乱。
- **call_deferred / set_deferred**：把“正在处理树/碰撞时”的树修改与属性修改推迟到安全时机（下一帧）。
- **管理器分层（Manager 模式）**：EnemyManager 管波次生成/路径/胜负统计；PickupManager 管掉落；VFXManager 管特效；GameManager 管全局信号/分数/重启。
- **Theme 资源复用**：StyleBoxTexture/LabelSettings/ProgressBar 样式存 .tres，多处复用保持 UI 统一。

## Key Concepts
- **get_parent() vs owner**：get_parent 是节点树父子；owner 是保存场景时的“户主”；组件用 owner 定位宿主。
- **Tween vs AnimationPlayer**：Tween 代码动态轻量（临时动画）；AnimationPlayer 编辑器可视化强大（成品动画）。
- **modulate + Color**：叠加颜色/透明度；`Color(1,1,1,0)` 透明白、保持原色仅改透明度。
- **area_entered vs body_entered**：另一方是 Area2D 触发前者；是 PhysicsBody2D 触发后者。
- **monitoring / monitorable / hit_multiple**：Area2D 是否监听、是否可被监听；子弹命中后 `set_deferred("monitoring", hit_multiple)` 控制单次/穿透。
- **输入函数路由**：`_input` 全局快捷键 → `_gui_input` 控件 → `_unhandled_input` 玩家控制 → `_input_event` 点选对象。
- **CanvasItem Shader**：GPU 逐像素程序；`uniform bool active` 由外部/动画控制，`fragment()` 用 flash_color 覆盖贴图实现受击闪白。
- **Line2D**：points 数组动态增删实现子弹拖尾。
- **PathFollow2D + Path2D**：改 `progress += speed*delta` 沿固定路径巡逻，适合入门移动敌人。
- **TileSet Terrain**：match sides/corners 自动地形，快速画连续道路。
- **多 TileMapLayer 地图**：Grass/Road/Border/Items/Tree 分层，职责分离；TileSet 的 Physics Layer 给大树/石头加碰撞。
- **enum Pickups {GUN, HEALTH}**：给道具类型起名，增强可读性。
- **AudioStreamRandomizer**：击中音效多个变体随机播放，避免重复感。
- **Camera2D**：跟随玩家、limits 限制范围、limit_smoothed/Position Smoothing 平滑。

## Mental Models
- Think of 组件 as 乐高积木：谁需要谁挂，零冗余、互不干扰。
- Think of Layer/Mask as 军事演习的识别色：红方只识别蓝、蓝方只识别红，友军永不误伤，且系统层面就不做无效检测。
- Think of 碰撞伤害 as “能力契约”而非“身份检查”：会受伤就实现 get_hurt，不会就自动忽略。
- Think of await 链 as 剧情先后顺序：A 导演没喊卡，B 不能冲上台；除非你故意要它后台跑。
- Think of call_deferred as “等安全了再动手”：不要在节点树正在变化时插入/删除/改属性。
- Think of 地图 as 分层施工图：草地/道路/边界/障碍/装饰各画一层，物理与导航才可控。

## Anti-patterns
- **把功能全堆进一个脚本**：加功能就牵一发动全身；先抽象成 HurtBox/Health/Weapon/Detect 等组件。
- **继承树越挖越深**：单继承无法满足“部分单位要雷达、部分不要”；用组合代替。
- **用 is_in_group 手动过滤碰撞**：无效碰撞照样发生、逻辑散落；改用 Layer/Mask 从底层过滤。
- **直接调用对方方法不检查**：`hurt_box.get_hurt()` 遇到墙体直接崩溃；先 `has_method()`。
- **await 链断裂**：生成波次的函数没被 await，敌人没生成完就开始结算。
- **在碰撞回调里立即改 monitoring / 树结构**：报错或行为不稳定；用 set_deferred / call_deferred。
- **把特效动画挂在即将销毁的对象里**：敌人一死动画即断；交给 VFXManager。
- **一次性音效永远同一个**：高频击中反馈单调刺耳；用 AudioStreamRandomizer。

## Code Examples
承伤组件（装甲减免 + 信号传递）：
```gdscript
extends Area2D

signal get_damage(damage)
@export var armor := 0

func get_hurt(damage: int) -> void:
    var final_damage = max(damage - armor, 0)
    get_damage.emit(final_damage)
```
- **What it demonstrates**: 组件只做“接收-减免-转发”，不关心自己是谁。

伤害组件（能力检查 + 单次/穿透）：
```gdscript
extends Area2D

signal hit
@export var damage := 1
@export var hit_multiple := false

func _ready() -> void:
    area_entered.connect(on_area_entered)

func apply_hit(hurt_box: Area2D) -> void:
    if hurt_box.has_method("get_hurt"):
        hurt_box.get_hurt(damage)
    set_deferred("monitoring", hit_multiple)
```
- **What it demonstrates**: 面向能力调用 + deferred 关闭重复命中。

角色组合接线（Player）：
```gdscript
func _ready() -> void:
    hurt_box_component.get_damage.connect(health_component.get_damage)
    health_component.health_changed.connect(on_health_changed)
    health_component.died.connect(on_died)

func on_died() -> void:
    Gamemanager.entity_died.emit(global_position, get_groups())
    Gamemanager.player_killed.emit()
    set_physics_process(false)
    hide()
    hurt_box_component.set_deferred("monitorable", false)
```
- **What it demonstrates**: 组件信号接在宿主角色里，一行代码把承伤与血量串起来。

敌方波次管理：
```gdscript
func spawn_waves() -> void:
    for i in wave_number:
        await spawn_wave()
        await wave_died

func spawn_wave() -> void:
    died_tank = 0
    for i in enemy_in_wave:
        call_deferred("spawn_enemy")
        await get_tree().create_timer(enemy_spawn_time).timeout
```
- **What it demonstrates**: await 链保证“一波全灭才刷下一波”。

## Reference Tables
| 组件 | 职责 | 挂载对象 |
|---|---|---|
| HurtBoxComponent | 承伤、护甲、转发伤害 | 玩家/敌人/炮塔 |
| HitBoxComponent | 攻击、单次/穿透 | 双方子弹 |
| HealthComponent | HP 与死亡信号 | 可被击毁对象 |
| WeaponComponent | 冷却/动画/音效/生成子弹 | 玩家、炮塔、坦克 |
| DetectComponent | 轮询索敌 | 敌人单位 |
| TrailComponent | 移动轨迹 | 移动单位 |

| Layer 命名 | 用途 | 典型 Mask |
|---|---|---|
| 1 PlayerHitBox | 玩家攻击 | 4 敌人承伤 |
| 2 PlayerHurtBox | 玩家承伤 | 3 敌弹 |
| 4 EnemyHurtBox | 敌人承伤 | 1 玩家子弹 |
| 6 PlayerBody | 玩家物理体 | 5/7/8 障碍/敌人/道具 |
| 7 EnemyBody | 敌人物理体 | 6 玩家 |

| 信号 | 触发场景 | 处理函数 |
|---|---|---|
| area_entered | 对方是 Area2D（HurtBox） | on_area_entered → apply_hit |
| body_entered | 对方是 PhysicsBody2D（墙） | 只播放 hit，不调用伤害 |

| 输入场景 | 推荐写法 |
|---|---|
| 暂停/截图/全局键 | `_input(event)` |
| 控件点击 | `_gui_input(event)` |
| 玩家移动/射击 | `_unhandled_input(event)` |
| 点击物体 | `_input_event()` |
| 持续按键 | `Input.is_action_pressed()` |
| 刚刚按下（跳/开火） | `Input.is_action_just_pressed()` |

## Worked Example
一发玩家子弹击中敌方炮塔的完整链路：子弹 HitBox（Layer 1）与炮塔 HurtBox（Layer 4）匹配触发 `area_entered` → HitBox `hit.emit()`（base bullet 转发 Gamemanager.bullet_hit 播放粒子与音效）→ `has_method("get_hurt")` 成立 → `get_hurt(damage)` 经 armor 减免后 emit `get_damage` → HealthComponent 扣血、emit `health_changed` 与 `died` → 炮塔闪烁并 queue_free，同时 entity_died 信号让 VFXManager 爆炸、PickupManager 概率掉道具、EnemyManager 更新计数、GameManager 判胜负。

## Key Takeaways
1. 先抽象组件再拼角色：Hurt/Hit/Health/Weapon/Detect 是可复用乐高块。
2. 碰撞关系用 Layer/Mask 矩阵表达，别用组标签在代码里过滤。
3. 伤害调用前 has_method，面向“能力”而非“身份”。
4. 含 await 的函数被调用处也要 await，保持 Await 链完整。
5. 树结构/碰撞属性修改尽量走 call_deferred/set_deferred。
6. 波次、道具、特效各自交给 Manager，避免主场景脚本膨胀。
7. 样式、TileSet、字体等存 .tres 资源复用，一处修改全项目生效。

## Connects To
- **Ch 6**: 本章重构第六章坦克 MVP。
- **Ch 8**: 组件化敌人为 AI 状态机、感知、导航打好载体。
- **Ch 10**: DetectComponent 将升级为完整视野感知。
- **Ch 12**: 多 TileMapLayer 地图为程序化生成提供结构基础。
- **Ch 16**: 组合优于继承、SRP、接口约定正是 SOLID 与设计模式的工程实践。
