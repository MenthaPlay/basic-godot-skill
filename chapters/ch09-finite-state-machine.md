# 第九章：游戏中的有限状态机

## Core Idea
本章把“行为切换”系统化为有限状态机：一个对象同时只能处于一个状态，事件/条件触发状态转换。先用枚举 + match 的状态变量实现坦克“巡逻/追击”AI，再升级为“状态模式”——每个状态独立成节点与脚本，由 StateMachine 统一调度，适合中大型项目与可复用 AI。

## Frameworks Introduced
- **有限状态机（FSM）四要素**：State（行为模式）、Transition（切换路径）、Event/Condition（触发开关）、Action（进入/退出/停留时执行的逻辑）。
- **为什么用 FSM**：避免“布尔地狱”和“条件混战”——每类行为一个布尔变量会互相冲突、判断重复、难以扩展与调试；FSM 让状态互斥且明确。
  - When to use: 角色动作、敌人 AI、游戏流程（菜单/暂停/运行）、任务生命周期等有离散互斥阶段的对象。
  - When NOT: 物理模拟、动画混合等连续系统，以及需要多状态叠加（移动+中毒+隐身）时；叠加需求考虑 HFSM/行为树/其他架构。
- **状态变量 FSM（Simple FSM）**：`enum + match` 单脚本实现。
  - Use when: 状态少（≤3 个原型）、逻辑简单、无需复用。
  - Trade-off: 脚本随状态膨胀成“巨型类”，新增状态要改多处。
- **状态模式（State Pattern）**：状态机节点（StateMachine）+ 状态基类（State.gd）+ 具体状态节点（EnemyPatrol/EnemyAttack）。
  - Use when: 状态 >5、逻辑复杂、需要在多个对象间复用状态（通用巡逻）。
  - How: StateMachine 把 State 子节点收集成字典、调用当前状态 `physics_update`；状态间不互相认识，只 `transitioned.emit(self, "NewStateName")` 请求切换。
- **宿主方法下沉**：把移动、转向、受伤等角色能力留在宿主 Enemy（update_direction/move），状态只负责“决策”，避免每个状态重复实现。
- **方向与到达判断**：`global_position.direction_to(target)` + `distance_to < 阈值` + `transform.x * speed` + `move_and_slide()` 是 Godot 追踪/巡逻的标配。

## Key Concepts
- **State / Transition / Action**：行为模式 / 路径 / 触发后动作。
- **互斥状态**：每一时刻只有一个活动状态，保证动画与逻辑不重叠。
- **Enter / Exit / Physics_update**：状态基类统一生命周期接口。
- **transitioned 信号**：状态向 StateMachine 发出切换请求（带 self 与目标状态名），只有当前状态能触发。
- **HFSM**：分层状态机，把战斗等大状态内部再拆子状态，解决状态爆炸。
- **Behavior Tree / GOAP**：FSM 之外的高级行为架构（条件模糊/动态权重时使用）。
- **state diagram**：用图描述状态与转换，先画图再写代码。
- **布尔地狱**：多个布尔开关组合导致“又在跳又在蹲”式的冲突状态。

## Mental Models
- Think of FSM as 行为交通指挥员：同一时刻只放行一条行为车道，明确何时因何转向哪条道。
- Think of 状态变量实现 as 一张表格 + 分支：简单直白，但表一大就乱。
- Think of 状态模式 as 每个状态一位演员，导演（StateMachine）喊谁上台谁上台；演员之间不认识，只向导演申请换人。
- Think of 状态生命周期 as 上台（enter）→ 演出（update）→ 下台（exit）：新状态只在上台时拿数据、下台时交数据。
- Think of 状态机 not as 万能药：连续系统与状态叠加场景别硬套。

## Anti-patterns
- **布尔状态叠开关**（is_jumping + is_running + is_attacking）：状态冲突、调试噩梦；改用枚举或状态对象。
- **每帧重复判断“能不能 X”**：把条件收敛进状态与转换，而不是散落各处。
- **新状态只加变量到处补判断**：应新增枚举值/状态节点，不改旧状态内部逻辑。
- **状态之间互相调用/直接 new**：状态只发 transitioned 信号，切换由 StateMachine 统一做。
- **在状态脚本里重复写角色移动**：共享方法放宿主，状态专注决策。
- **状态少还上状态模式**：增加脚本数量与复杂度，违反“简单问题简单解决”。
- **强行 FSM**：连续/叠加型行为用 HFSM 或行为树，别硬塞进平面状态机。

## Code Examples
状态变量 FSM（敌方坦克巡逻/追击核心）：
```gdscript
enum FSMState { Patrol, Chase }
var cur_state = FSMState.Patrol

func update_patrol() -> void:
    update_direction()          # global_position.direction_to(target_pos)
    move_to_target()            # velocity = transform.x * speed; move_and_slide()
    if near_point():
        find_next_point()
    if find_player():
        cur_state = FSMState.Chase

func update_chase() -> void:
    if find_player():
        target_pos = detect_component.get_player_pos()
        update_direction()
        move_to_target()
        weapon_component.target(target_pos)
        weapon_component.shoot(target_pos)
    else:
        cur_state = FSMState.Patrol

func _physics_process(_delta: float) -> void:
    match cur_state:
        FSMState.Patrol:
            update_patrol()
        FSMState.Chase:
            update_chase()
```
- **What it demonstrates**: enum + match 的简单 FSM 骨架。

状态模式——基类与状态机：
```gdscript
# State.gd
extends Node
class_name State
signal transitioned
func enter(): pass
func exit(): pass
func physics_update(_delta: float): pass
```
```gdscript
# StateMachine.gd
extends Node
@export var initial_state: State
var current_state: State
var states: Dictionary = {}

func _ready() -> void:
    for child in get_children():
        if child is State:
            states[child.name] = child
            child.transitioned.connect(on_child_transition)
    if initial_state:
        initial_state.enter()
        current_state = initial_state

func on_child_transition(state: State, new_state_name: String) -> void:
    if state != current_state:
        return
    var new_state = states.get(new_state_name)
    if not new_state:
        return
    current_state.exit()
    new_state.enter()
    current_state = new_state

func _physics_process(delta: float) -> void:
    if current_state:
        current_state.physics_update(delta)
```
- **What it demonstrates**: 收集子状态、只让当前状态切换、统一生命周期。

具体状态切换：
```gdscript
# EnemyPatrol.gd
extends State
@export var enemy: Node2D
@export var detect_component: Area2D

func physics_update(_delta: float) -> void:
    enemy.update_direction(target_pos)
    enemy.move()
    if near_point():
        find_next_point()
    if detect_component.find_player():
        transitioned.emit(self, "EnemyAttack")
```
- **What it demonstrates**: 状态只做决策，移动复用宿主方法，切换走信号。

## Reference Tables
| 维度 | 状态变量 FSM | 状态模式 |
|---|---|---|
| 机制 | enum + match/if | 节点 + 独立脚本 |
| 结构 | 单脚本集中 | 一状态一脚本一节点 |
| 适用 | 状态少、原型 | 复杂 AI、中大型项目 |
| 可维护性 | 差（巨型类） | 优（解耦） |
| 扩展 | 需改多处判断 | 加节点即可 |
| 复用 | 难 | 状态脚本可共享 |
| 上手 | 极低 | 中等 |

| FSM 应用 | 状态示例 |
|---|---|
| 角色控制 | Idle → Run → Jump |
| 敌人 AI | Patrol → Chase → Attack |
| 游戏流程 | 主菜单/加载/进行/暂停 |
| 任务系统 | 未接取 → 进行中 → 已交付 |

## Worked Example
状态模式版敌方坦克：EnemyTank（CharacterBody2D）挂 StateMachine 子节点，StateMachine 下挂 EnemyPatrol 与 EnemyAttack。初始状态设 EnemyPatrol；`_ready` 里 StateMachine 把两个状态收进字典并连接 transitioned。EnemyPatrol 每物理帧调用宿主 `update_direction/move`，接近巡逻点就取下一个 Marker2D 巡逻点；探测组件发现玩家 → `transitioned.emit(self, "EnemyAttack")` → StateMachine 退出 Patrol、进入 Attack；Attack 每帧追玩家并让 WeaponComponent 瞄准开火；玩家脱离探测范围 → 发信号切回 EnemyPatrol。

## Key Takeaways
1. FSM = 互斥状态 + 明确转换 + 对应动作，先画状态图再编码。
2. 状态少用 enum+match；状态多/要复用就升级状态模式。
3. 状态模式里状态之间不直连，统一走 transitioned 信号与 StateMachine。
4. 角色共享能力放宿主对象，状态脚本专注决策。
5. 连续/叠加行为别硬套 FSM，必要时用 HFSM 或行为树。
6. 巡逻 AI 的标配：direction_to 转向 + distance_to 判到达 + move_and_slide 移动。

## Connects To
- **Ch 7**: 组件化宿主为状态模式提供 Enemy 能力接口。
- **Ch 8**: FSM 从概念落为代码。
- **Ch 10/11**: 巡逻/追击状态将接入视野感知、避障与导航系统。
- **Ch 16**: 状态模式是本章设计的经典 GoF 模式之一。
