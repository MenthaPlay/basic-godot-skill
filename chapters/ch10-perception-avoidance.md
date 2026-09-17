# 第十章：感知系统与避障

## Core Idea
本章把“看到就追、消失就忘、遇墙就卡”的敌人升级为“会潜行博弈的智能猎手”：用 RayCast2D 实现带视角限制的扇形视野；用 ShapeCast2D + 法线向量实现局部动态避障；新增 EnemyWander 索敌状态提供“目标丢失后的短期记忆搜索”；最后用组件 + 场景继承构建可击落的自动追踪导弹与远程压制单位导弹攻击车。

## Frameworks Introduced
- **扇形视野感知（SearchComponent）**：RayCast2D 射线 + 角度阈值 = 有方向限制的视觉。
  - How: 射线每帧 `look_at(player)`；`transform.get_rotation()` 绝对值 < π/2 且射线命中 Player 分组时返回 true。
  - Why: 替代 360° Area2D 全向感知，让玩家可从背后潜行、从视野盲区接近。
- **鸭子类型 / 统一感知接口**：SearchComponent 与 DetectComponent 底层实现完全不同，但都提供 `can_see_player()` 与 `player_ref`，宿主只看行为不看类型。
  - Why: 从“为特定节点写代码”进化为“为功能接口写代码”，换感知组件不必改状态机。
- **ShapeCast2D 动态避障（AvoidComponent）**：把圆形状向前投射 50px，像盲杖探测“即将经过的路径”。
  - How: `is_colliding()` 发现障碍 → `get_collision_normal()` 取墙面法线 → 新方向 = `owner.transform.x + normal * force`，让单位沿墙“滑”过去而不是卡死。
  - Limit: 局部避障不是完整寻路；长距离绕行仍需导航系统。
- **索敌状态（EnemyWander）**：目标脱离视野后进入“最后可见区域搜索”，超时再回巡逻。
  - How: 进入时生成“前方 distance±noise 随机搜索点”并启动 10s Timer；搜索中再次看到玩家立刻切 EnemyAttack；Timer 到点切 EnemyPatrol。
  - Why: 消灭“看见就追、消失即忘”的机械感，给玩家施加持续压力。
- **自动追踪导弹**：导弹每帧用 DetectComponent 探测玩家，`rotate_toward` 平滑转向目标，再沿 transform.x 前进。
  - Key: 命中玩家触发伤害并销毁；自身带 HurtBox+Health 可被玩家击落；Collision Mask 只检测玩家层以穿越障碍。
- **导弹攻击车（远程压制单位）**：360° 感知 + MisLaunchComponent（双发射口、冷却、音效）+ 状态机 Patrol/Wander/Attack + 避障；速度慢、血厚，攻击间隔长、导弹可被击毁作为平衡。
- **组件化兵种对比**：炮塔（固定高反应）、坦克（近距追击）、导弹车（移动+远程区域压制）。

## Key Concepts
- **RayCast2D**：看不见的探针；Target Position 定方向长度、Collision Mask 定检测层、get_collider/is_colliding 读结果。
- **ShapeCast2D**：形状投射，比射线更贴“体积感”；get_collision_normal 返回碰撞面法线。
- **法线向量（normal）**：垂直碰撞表面的单位向量；本意是“别往这走”，加到前进方向就产生绕行。
- **class_name Player**：给脚本注册全局类名，使别的脚本能做类型标注。
- **鸭子类型（Duck Typing）**：走起来像鸭子、叫起来像鸭子就当作鸭子——有 `can_see_player()` 就能用。
- **索敌（Wander）**：目标丢失后的区域搜索状态，属于“短期记忆”。
- **rotate_toward**：角度插值平滑转向，避免瞬间转身。
- **Debug → Visible Collision Shapes**：可视化碰撞/探测/避障区域，联调必备。
- **组件化继承复用**：EnemyMissile 继承 bullet_base.tscn 再组合 Detect/Hurt/Health，避免重写移动与碰撞。

## Mental Models
- Think of RayCast2D as 敌人的目光：只看得见正前方、足够远、没被墙挡住的东西。
- Think of ShapeCast2D as 盲人的盲杖：不断探测前路，碰到障碍就调整步伐方向。
- Think of 碰撞法线 as 墙在“推你回去”：把原本前进方向 + 法线×力度，即可沿墙滑动。
- Think of Duck Typing as “会叫会走就当鸭子”：换眼睛（雷达/视线）不必改大脑（状态机）。
- Think of EnemyWander as 警卫丢了目标后的“最后目击地搜索”：不会立刻当没事发生。
- Think of 导弹 as 一次性的小型智能体：自己探测、自己转向、可被打爆。

## Anti-patterns
- **360° 全向感知用在所有敌人**：没有潜行空间与背后偷袭玩法；需要方向性视野时用射线+夹角。
- **敌人直接撞墙卡死**：没有前方探测；用 ShapeCast2D 局部避障，即使不导航也能顺畅绕行。
- **丢目标立刻切回巡逻**：机械无张力；插入 EnemyWander 搜索状态。
- **换感知实现就改状态机**：两种组件方法名不统一会破坏宿主逻辑；统一 `can_see_player` 接口。
- **让玩家/敌人子弹命中一切**：Mask 分层——导弹只检测玩家层，才能穿越障碍物。
- **追踪弹无敌不可反制**：导弹可被击落、攻击间隔长，才能保持玩法公平。
- **每帧硬转向**：瞬时转角生硬；用 rotate_toward 平滑。

## Code Examples
扇形视野组件：
```gdscript
extends RayCast2D

var player_ref: Player

func _ready() -> void:
    player_ref = get_tree().get_first_node_in_group("Player")

func _process(_delta: float) -> void:
    if player_ref:
        look_at(player_ref.global_position)

func can_see_player() -> bool:
    return abs(transform.get_rotation()) < PI / 2 and player_detected()

func player_detected() -> bool:
    var obj = get_collider()
    return obj != null and obj.is_in_group("Player")
```
- **What it demonstrates**: 射线 + 夹角限制的扇形视野。

避障组件：
```gdscript
extends ShapeCast2D

@export var force: float = 2

func obstackes_detected() -> bool:
    return is_colliding()

func get_new_direction() -> Vector2:
    var normal_vec = get_collision_normal(0)
    return owner.transform.x + normal_vec * force
```
- **What it demonstrates**: 法线向量修正前进方向，产生沿墙滑动。

自动追踪导弹：
```gdscript
func _process(delta: float) -> void:
    if detect_component.can_see_player():
        update_direction(detect_component.player_ref.global_position, delta)
    move_forward(delta)

func update_direction(target_pos: Vector2, delta: float) -> void:
    var direction = global_position.direction_to(target_pos)
    rotation = rotate_toward(rotation, direction.angle(), 2 * PI * delta)

func move_forward(delta: float) -> void:
    position += transform.x * speed * delta
```
- **What it demonstrates**: 探测 → 平滑转向 → 前进的追踪弹循环。

索敌状态：
```gdscript
func enter() -> void:
    end_wander = false
    find_next_point()
    timer.start(10)

func physics_update(_delta: float) -> void:
    if search_component.can_see_player():
        transitioned.emit(self, "EnemyAttack")
    elif end_wander:
        transitioned.emit(self, "EnemyPatrol")
    else:
        enemy.update_direction(target_pos)
        enemy.move()
        if enemy.global_position.distance_to(target_pos) < 5:
            find_next_point()
```
- **What it demonstrates**: 搜索状态优先响应“再次看到”，超时才回巡逻。

## Reference Tables
| 单位 | 感知 | 攻击 | 移动 | 弱点 |
|---|---|---|---|---|
| 炮塔 | 360° 圆域 | 炮弹 | 无 | 固定 |
| 敌方坦克 | 扇形视野 | 炮弹近距 | 巡逻/追击 | 需接近 |
| 导弹车 | 360° | 追踪导弹 | 巡逻+避障 | 间隔长、导弹可击毁 |

| 机制 | 节点 | 关键调用 |
|---|---|---|
| 视线探测 | RayCast2D | get_collider / is_colliding |
| 体积探测 | ShapeCast2D | get_collision_normal |
| 全向感知 | Area2D | has_overlapping_bodies |
| 平滑转向 | Node2D | rotate_toward |
| 平滑移动 | CharacterBody2D | move_and_slide |

## Worked Example
敌方坦克完整智能链：Patrol 沿 Marker2D 巡逻；SearchComponent（射线长 350、Mask 6）在前方 90° 内发现玩家 → EnemyAttack 追击射击；玩家绕到背后，扇形视野失效 → 攻击态切 EnemyWander：以“前方 200px + ±50 随机点”为搜索目标移动 10 秒；期间再看到玩家立即回到 EnemyAttack，找不到则回 Patrol。途中 ShapeCast2D 探到石头 → 用墙面法线修正方向沿边滑过，不再卡死。

## Key Takeaways
1. 方向性视野 = 射线长度 + 夹角阈值；给潜行留出战术空间。
2. 不同感知实现统一 `can_see_player()` 接口，宿主逻辑零改动。
3. 局部避障用 ShapeCast + 法线向量合成新方向，轻量且见效快。
4. 目标丢失别立刻放弃，加索敌状态模拟“最后目击搜索”。
5. 追踪弹 = 探测 + rotate_toward + transform.x 前进；自身可带 Hurt/Health 形成反制。
6. 新兵种通过组件组合与场景继承快速装配，只改参数与感知/攻击模块。

## Connects To
- **Ch 8**: 感知与避障概念落地。
- **Ch 9**: 新增 EnemyWander 扩展 FSM 状态机。
- **Ch 11**: 局部避障将补上导航系统做全局寻路。
- **Ch 12**: 地图中的石头/树正是 AvoidComponent 的 Mask 5 障碍层。
