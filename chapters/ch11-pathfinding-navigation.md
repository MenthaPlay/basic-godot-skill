# 第十一章：路径规划与导航系统

## Core Idea
本章把第十章“撞墙才转向”的局部避障升级为“知道怎么绕、也知道该往哪走”的全局导航：在 TileMap/TileSet 上配置 wander/patrol 导航图层生成导航网格，给移动单位挂 NavigationAgent2D 做路径规划与 RVO 动态避障，并把导航逻辑接入 FSM 的巡逻/索敌/攻击状态，让敌人能跨全图寻路、按状态切换可通行图层、互相避让。

## Frameworks Introduced
- **路径规划 vs 局部避障**：局部避障只看当前周围、反应式、易卡死；路径规划基于全图可通行区域、绕开多层障碍、路径平滑可控。
  - Use: 路径规划作为行动基础，局部避障作为移动修正补充；有 NavigationAgent 后删掉旧的 ShapeCast 避障，避免冲突。
- **Godot 导航三件套**：
  - NavigationServer2D：底层导航地图调度，修改在下一物理帧同步。
  - NavigationRegion2D / TileMap Navigation Layer：定义可通行区域与代价；格子地图由 TileSet 的 NavigationLayer 自动生成。
  - NavigationAgent2D：挂载在移动单位上，输出“建议”，不替单位移动。
- **导航图层（Navigation Layer）**：为不同行为给同一块地设置不同通行规则——patrol 只能走道路、wander/追击可走草地。
  - How: `set_navigation_layer_value(layer, bool)` 在状态 enter 时切换。
- **NavigationAgent2D 路径流程**：设置 `target_position` → `get_next_path_position()` 取下一路径点 → 更新朝向 → 计算期望速度 → 移动 → `is_navigation_finished()/is_target_reached()` 判断到达。
- **RVO 动态避障**：启用 Avoidance Enabled 后，把期望速度交给 `set_velocity()`，系统算出双方各让一步的“安全速度”并经 `velocity_computed` 信号回传。
  - Why: 两个单位规划同一条路时不会相撞卡死，像人群一起让路。
- **状态机 × 导航**：三个状态在 enter 时切换导航层、在 physics_update 里 set target + update_nav，接口一致即可复用到不同敌人。
- **等待 NavigationServer 就绪**：`_ready` 中暂停物理、`await 0.1s` 后再开启，避免导航图未构建导致路径失败。
- **攻击保持距离（kiting）**：离玩家 > min_dist 才追，≤ min_dist 停止并射击，避免贴脸。

## Key Concepts
- **静态避障**：不可通行 tile 不在导航网格里，路径根本不会经过——规划阶段就解决。
- **动态避障**：agent 之间通过 RVO 协调速度——移动阶段解决。
- **Navigation Layer 命名**：Project Settings → Layer Names → 2D Navigation 把 1 命名为 wander、2 命名 patrol。
- **八角形导航形状**：给 tile 编辑通行形状时避免方形边缘造成的路径抖动/贴墙。
- **Path PostProcessing = Edge Centered**：让路径更贴合网格地图。
- **Avoidance Radius**：避障范围；过小易撞、过大多绕。
- **velocity_computed 信号**：接住导航系统返回的安全速度并赋给 velocity。
- **is_target_reached / is_navigation_finished**：判到达/判全程结束。
- **Debug → Visible Navigation + Agent Debug Enabled**：可视化导航网格与单条规划路径。
- **接口一致性复用**：不同敌人只要把状态脚本的 enemy/nav_agent/search/weapon 变量绑对，脚本零修改复用。

## Mental Models
- Think of NavigationAgent2D as 导航 APP：只给路线建议，方向盘（移动代码）还是角色自己握。
- Think of RVO as 人群避让：不是一个人硬躲，而是双方各让一点速度。
- Think of 导航图层 as 同一张地图上的“限行规则”：巡逻只准走路（patrol），追击可以压草地（wander）。
- Think of NavigationServer as 后台调度中心：地图区域的修改下一帧统一同步，别在运行中立刻期待生效。
- Think of 静态避障 as 修路时就把墙修好：根本不让路径穿过墙，而不是角色到墙前才刹车。

## Anti-patterns
- **只做局部避障，没有全局导航**：敌人会进死胡同卡死；复杂地图必须路径规划 + 局部修正组合。
- **保留旧 AvoidComponent 又加 NavigationAgent**：两套避障打架；导航接管后删除射线避障。
- **在 _ready 立刻设目标**：导航图未构建，路径生成失败；先 await 一小段时间。
- **指望 Agent 自己移动**：它只给 next_path_position/safe velocity，必须自己 update_direction + move_and_slide。
- **开了 Avoidance 却不用 set_velocity**：动态避障不生效；期望速度必须先交给 Agent 处理。
- **所有状态共用同一导航层**：巡逻压草地、追击卡路都违反设计；按状态切换层。
- **不检查状态机引用**：enemy/nav_agent/search/weapon 没绑对，脚本“看起来一样”却不动；先核对 @export 绑定。
- **Radius 随意**：过小会重叠，过大会绕远；按单位体型调整。

## Code Examples
宿主导航移动接口：
```gdscript
@onready var navigation_agent_2d: NavigationAgent2D = $NavigationAgent2D

func _ready() -> void:
    navigation_agent_2d.velocity_computed.connect(on_velocity_computed)

func set_nav_to_target(target: Vector2) -> void:
    navigation_agent_2d.target_position = target

func on_velocity_computed(safe_velocity: Vector2) -> void:
    velocity = safe_velocity

func update_nav() -> void:
    if navigation_agent_2d.is_navigation_finished():
        return
    var next_pos = navigation_agent_2d.get_next_path_position()
    update_direction(next_pos)
    var new_velocity = transform.x * speed
    if navigation_agent_2d.avoidance_enabled:
        navigation_agent_2d.set_velocity(new_velocity)
    else:
        velocity = new_velocity
    move_and_slide()
```
- **What it demonstrates**: 取下一路径点 + 转向 + RVO 安全速度 + 实际移动。

巡逻状态按图层导航：
```gdscript
func enter() -> void:
    patrol_target = patrol_points.pick_random()
    nav_agent.set_navigation_layer_value(2, true)   # patrol=道路
    nav_agent.set_navigation_layer_value(1, false)  # wander=草地

func physics_update(_delta: float) -> void:
    enemy.set_nav_to_target(patrol_target)
    enemy.update_nav()
    if enemy.navigation_agent_2d.is_target_reached():
        patrol_target = patrol_points.pick_random()
    if search_component.can_see_player():
        transitioned.emit(self, "EnemyAttack")
```
- **What it demonstrates**: 进入状态切导航层，到达巡逻点换下一个点。

攻击状态保持距离：
```gdscript
const MIN_DIST := 150.0

func physics_update(_delta: float) -> void:
    if not search_component.can_see_player():
        transitioned.emit(self, "EnemyWander")
        return
    target_pos = search_component.player_ref.global_position
    if enemy.global_position.distance_to(target_pos) > MIN_DIST:
        enemy.set_nav_to_target(target_pos)
        enemy.update_nav()
    else:
        enemy.stop()
    weapon_component.target(target_pos)
    weapon_component.shoot(target_pos)
```
- **What it demonstrates**: 追到射程内就停，边射击边控制距离。

## Reference Tables
| 能力 | 局部避障 | 全局路径规划 |
|---|---|---|
| 视野 | 只看周围 | 全图可通行区域 |
| 成本 | 低 | 较高 |
| 能否绕死胡同 | 否 | 是 |
| 路径质量 | 僵硬 | 平滑可控 |
| 动态避障 | 简单 | RVO 多单位协调 |

| 常见问题 | 可能原因 | 修复 |
|---|---|---|
| 原地不动 | 没设 target_position / 路径未生成 | 检查 set_nav_to_target、等 0.1s |
| 路径点错乱 | 导航未初始化就设目标 | await 后再开物理 |
| 单位重叠 | Avoidance 未开或半径小 | 开 Avoidance、调大 Radius |
| 缺可通行区 | Tile 没绑 Navigation Layer | 回 TileSet 画导航形状 |

| 状态 | 导航层 | 目标 |
|---|---|---|
| Patrol | patrol(2)=true | 随机巡逻点 |
| Wander | wander(1)=true | 前方随机点 |
| Attack | wander(1)=true | 玩家位置（min_dist 停） |

## Worked Example
导弹攻击车导航改造：加 NavigationAgent2D（Edge Centered、Avoidance Enabled、Radius 40、Debug 开），删除 AvoidComponent。主脚本连 velocity_computed；`update_nav()` 取下一路径点转向，期望速度交给 Agent 获得安全速度再 move_and_slide。状态机里 Patrol 进入时只开 patrol 层沿道路随机巡逻；看到玩家 → EnemyAttack 切 wander 层追击，距玩家 ≤150 停下射击；目标丢失 → EnemyWander 在草地随机探索 5 秒，超时返回出生点附近后回 Patrol。Debug → Visible Navigation 可看到草地/道路网格与每条规划路径。

## Key Takeaways
1. 路径规划负责“全局怎么走”，RVO 动态避障负责“当下别撞人”。
2. TileMap 导航 = TileSet Navigation Layer + 每个 tile 的通行形状，自动生成导航网格。
3. NavigationAgent 只给建议，移动自己写：target_position → get_next_path_position → set_velocity → move_and_slide。
4. 开 Avoidance 必须走 set_velocity/velocity_computed 流程，动态避障才生效。
5. 导航图就绪前别设目标，await 一拍再启动状态机。
6. 不同行为用不同导航层（道路巡逻 vs 草地追击），在状态 enter 切换。
7. 状态脚本只依赖统一接口，换敌人类型只需重新绑定 @export 节点。

## Connects To
- **Ch 10**: 替代局部 ShapeCast 避障，保留视野感知。
- **Ch 9**: 导航逻辑注入 Patrol/Wander/Attack 状态。
- **Ch 12**: 程序化生成地图后同样要自动铺设导航图层。
- **Ch 8**: NavigationAgent/RVO/A* 概念在 Godot 落地。
