# 第六章：进阶项目实战《Battle Tank》

## Core Idea
本章用俯视角坦克射击游戏《Battle Tank》演示中大型 2D 项目的工程方法：模块化拆解 + 迭代式开发，先做出“移动-射击-反馈”MVP；用 GameManager 信号集线器协调全局状态；每个角色/子弹/特效独立成场景，最终由主场景组装并联调。

## Frameworks Introduced
- **模块化设计**：把系统拆成玩家控制、AI 敌人、子弹、地图、生命值、全局管理、UI、视听八个独立低耦合模块。
  - How: “一个功能、一个场景”；场景之间不直接互相找，通过分组/信号/管理器协作。
- **迭代式开发（MVP 先行）**：阶段一先做核心玩法（移动-射击-碰撞-HP-胜负），阶段二加 AI/导航/程序化地图，阶段三再做光照菜单打磨。
  - When to use: 任何中大型游戏项目；每一阶段都要有“可运行可测试”的里程碑。
- **Signal Hub（信号集线器）**：把所有跨模块信号集中在 GameManager 单例，作为广播中心。
  - When to use: 玩家死亡、敌人阵亡、胜负判定等需要多个分散模块响应的事件。
  - Contrast: 父子等可直接访问的模块用节点本地信号即可，别把一切塞进 Hub。
- **组件发现模式（分组查询）**：`get_tree().get_first_node_in_group("Player")` 让敌人自主发现玩家。
  - When to use: 动态生成的敌人数量多、层级变化大时，替代 @export 手动引用。
- **解耦视觉特效（VFXManager）**：爆炸动画不挂在敌人节点上，而是由独立管理器监听 `enemy_killed` 信号，在敌人销毁后也能完整播放。
  - How: manager 收到信号 → instantiate 动画场景 → 设坐标 → play → `await animation_finished` → queue_free。
- **伤害接口约定（reduce_health）**：子弹碰撞后调用目标的 `reduce_health()`；防御式调用前用 `has_method()` 检查。
- **平滑操控（move_toward / rotate_toward）**：用线性趋近让速度与朝向渐变，替代瞬间加减速。
- **调试三板斧**：print 快速验证 → Remote 场景树查运行时层级与变量 → 断点单步追踪调用链。

## Key Concepts
- **top_level = true**：把动态生成的子弹提升为世界级节点，避免继承炮塔旋转产生“甩鞭子”轨迹。
- **transform.x**：节点本地 X 轴方向向量；旋转后自动变化，是“沿自己朝向移动”的标准写法。
- **global_position vs position**：跨父级对齐用全局坐标；子节点内部偏移用局部坐标。
- **look_at()**：让节点 X 轴正方向指向目标；贴图应默认朝右或预先旋转 90°。
- **Input.get_vector()**：根据四个动作返回自动归一化的八方向向量。
- **get_viewport_rect().size + clamp**：动态获取屏幕尺寸并限制坐标，防止角色跑出窗口。
- **TileSet / TileMapLayer**：先定义瓦片仓库（大小 64×64），再用画笔/矩形工具铺地图；关卡 = 地图 + 玩法逻辑。
- **Area2D 根节点选择**：无需物理移动但需被击中检测的对象（炮塔、子弹、玩家）适合用 Area2D。
- **@export var player / bullet_scene**：在 Inspector 中拖拽赋值场景引用或玩家节点。
- **CanvasLayer + Control 容器 HUD**：血条 ProgressBar、击杀 Label、胜负 PanelContainer + Timer 延时提示。
- **Autoload 注册**：Project Settings → Autoload 添加 GameManager，全局脚本可访问。

## Mental Models
- Think of 模块化设计 as 盖房子先定承重墙和管线：换窗户不用拆整栋楼。
- Think of 迭代式开发 as 施工顺序：先让第一层能住人，再刷墙装灯，别等精装完才验收。
- Think of GameManager as 总导演：各模块只向它汇报，不互相串门，避免“蜘蛛网代码”。
- Think of 场景 as 独立零件：在 Game 主场景里像组装精密机器一样拼装，可单独测试。
- Think of 敌人自动索敌 as 寻人广播：只要目标在 Player 组，无论层级在哪都能瞬间锁定。
- Think of 特效 as 外包演出：敌人死了（信号发出）立刻交给 VFXManager 上台表演，不受尸体销毁影响。

## Anti-patterns
- **场景复制后逻辑重复**：两份子弹脚本各自维护飞行逻辑；同逻辑差异小时用场景继承，父场景改一次全同步。
- **模块直接交叉引用**：Enemy 直调 UI、Player 直调 GameManager 之外的对象，形成蜘蛛网；走 Signal Hub。
- **把动画节点挂在会销毁的对象上**：敌人被 queue_free 动画立刻中断；交给外部 VFXManager。
- **用 position 对齐跨父级对象**：父子层级不同坐标参考系不一致，导致炮塔朝向偏移；对齐用 global_position。
- **忘记 top_level**：子弹成为旋转父节点的子节点后轨迹弯曲。
- **让子弹无差别伤害一切**：碰撞后应检查 Group + has_method，再调用接口，避免误伤石头报错。
- **硬编码窗口边界**：用 get_viewport_rect 动态获取，适配不同分辨率。

## Code Examples
玩家平滑移动 + 炮管瞄准 + 发射：
```gdscript
func move(delta: float) -> void:
    direction = Input.get_vector("left", "right", "up", "down")
    if direction != Vector2.ZERO:
        rotation = rotate_toward(rotation, direction.angle(), 2 * PI * delta)
        speed = move_toward(speed, max_speed, max_speed * delta)
    else:
        speed = move_toward(speed, 0, 2 * max_speed * delta)
    position += transform.x * speed * delta
    position = position.clamp(Vector2.ZERO, get_viewport_rect().size)

func target() -> void:
    gun.look_at(get_global_mouse_position())

func shoot() -> void:
    if Input.is_action_just_pressed("shoot"):
        shoot_sound.play()
        var bullet = bullet_scene.instantiate()
        bullet.global_position = marker_2d.global_position
        bullet.look_at(get_global_mouse_position())
        bullet.top_level = true
        add_child(bullet)
```
- **What it demonstrates**: Input.get_vector、rotate_toward/move_toward 惯性、transform.x 朝向移动、Marker2D 炮口与 top_level 发射。

敌人追踪与开火：
```gdscript
@onready var player = get_tree().get_first_node_in_group("Player")

func _ready() -> void:
    timer.timeout.connect(on_time_out)
    timer.start(1)
    Gamemanager.player_killed.connect(on_player_killed)

func _process(_delta: float) -> void:
    if player:
        gun.look_at(player.global_position)

func on_time_out() -> void:
    shoot()
    timer.start(randf_range(1, 3))
```
- **What it demonstrates**: 组查询索敌 + look_at + Timer 随机开火节奏。

伤害连锁（玩家受击）：
```gdscript
# player_bullet.gd
func on_area_entered(area: Area2D) -> void:
    if area.is_in_group("Enemy") and area.has_method("reduce_health"):
        area.reduce_health()
        queue_free()
```
- **What it demonstrates**: Group + has_method 防御式伤害接口调用。

## Reference Tables
| 模块 | 核心职责 | 技术要点 |
|---|---|---|
| 玩家控制 | 输入→平滑移动、瞄准、射击 | Input.get_vector / look_at / move_toward |
| AI 敌人 | 索敌与攻击 | group 查询 / look_at / Timer |
| 子弹系统 | 运动、碰撞、回收 | transform.x / top_level / queue_free |
| 生命值 | HP 与伤害传递 | reduce_health 接口 + 信号 |
| 全局管理 | 流程与状态 | GameManager Autoload + Signal Hub |
| UI | 血条/计分/结算 | CanvasLayer + Control 容器 |
| 视听 | 爆炸与音效 | VFXManager + AudioStreamPlayer |

| 方案 | 优点 | 缺点 |
|---|---|---|
| 场景复制 | 简单直观 | 代码冗余，改逻辑要改两份 |
| 场景继承 | 逻辑统一、改一处全同步 | 耦合较高，特化差异易困惑 |
| 通用场景+参数 | 轻量 | 扩展性不足 |

| 调试手段 | 适用 | 交互性 |
|---|---|---|
| print() | 验证信号/数值 | 弱 |
| Remote 场景树 | 检查层级、改运行值 | 强 |
| 断点调试 | 复杂逻辑调用链 | 极强 |

## Worked Example
敌方子弹击中玩家的完整连锁：子弹 `area_entered` 触发 → 判断 `area.is_in_group("Player")` → 调用 `player.reduce_health()` → 玩家 HP-1 并 `emit update_health_ui(health)` → HUD 收到信号按 `100*health/10` 缩短 ProgressBar → 若 HP≤0 则 `emit player_killed` → 玩家 `set_process(false)` 停摆、敌人 Timer 停止、HUD 延时显示 "You Lost"；子弹自身 queue_free。

## Key Takeaways
1. 大项目先拆模块再定迭代阶段，每个阶段都留一个可运行里程碑。
2. 跨模块通信一律走 GameManager 信号总线，避免蜘蛛网耦合。
3. 动态生成物（子弹/敌人）要用 top_level 或归属世界容器，避免继承父节点旋转。
4. 对齐跨父级对象用 global_position，沿自身朝向移动用 transform.x。
5. 敌人索敌用 get_first_node_in_group，别为每个敌人手动绑玩家引用。
6. 会销毁的节点不背特效，交给 VFXManager 解耦播放。
7. 碰撞伤害前先查 Group 与 has_method，保证系统可扩展。

## Connects To
- **Ch 5**: 从单信号总线升级为模块化 + Signal Hub 架构。
- **Ch 7**: 本章 MVP 将接受重构（组合优于继承、组件化）。
- **Ch 8-12**: 敌人 AI 状态机、感知、导航、程序化地图都在本章项目上展开。
- **Ch 13**: 视觉优化、主菜单与发布继续打磨本章成品。
