# Patterns

## Game Loop + Delta Time
**When to use**: 任何基于时间的移动、计时、物理逻辑。
**How**: 所有位移/旋转/进度乘以 delta；`_process` 处理视觉与输入，`_physics_process` 处理物理与碰撞。
**Trade-offs**: 必须每处一致使用；漏乘导致低帧设备变慢。

## 场景组合与场景继承
**When to use**: 组织游戏对象与复用。
**How**: 一个功能一个 .tscn，主场景实例化组合；同逻辑不同外观用 New Inherited Scene 继承（bg → sky/ground、bullet → player/enemy bullet）。
**Trade-offs**: 继承省重复但耦合深；组合灵活但场景多。

## Autoload 单例（GameManager / SoundManager）
**When to use**: 全局状态、跨模块事件、BGM 跨场景连续播放。
**How**: Project Settings → Autoload 注册；集中定义信号与公共函数（score/restart/to_game/play_music）。
**Trade-offs**: 全局可访问易滥用成“上帝对象”；只放真正的全局能力。

## Signal Hub 信号总线
**When to use**: 玩家死亡/得分/胜负等 2+ 分散模块要响应的事件。
**How**: GameManager 定义 signal；触发者 emit，订阅者 connect；父子强关联用小范围本地信号，别全塞 Hub。
**Trade-offs**: 集中易查，但信号过多难追踪；配 Remote 调试与命名规范。

## HurtBox / HitBox 伤害契约
**When to use**: 子弹/攻击命中与承伤解耦。
**How**: HitBox 命中后 `has_method("get_hurt")` → `get_hurt(damage)`；HurtBox 扣 armor 后 emit 给 HealthComponent。
**Trade-offs**: 能力契约灵活；无文档时接口易失联。

## Collision Layer/Mask 矩阵
**When to use**: 需要对象间选择性碰撞且想可视化配置。
**How**: 项目设置命名 2D Physics 层；Layer=我是谁，Mask=检测谁；同阵营互相不可见。
**Trade-offs**: 层规划错误难查；32 层上限需统一登记。

## 组件化角色（Composition）
**When to use**: 角色能力多变、复用度高（Health/Hurt/Weapon/Detect/Trail/Nav）。
**How**: 功能独立成组件场景，宿主 _ready 连接组件信号，能力方法留宿主。
**Trade-offs**: 文件多、绑定繁琐；换来扩展性与替换性。

## Simple FSM vs State Pattern
**When to use**: 离散互斥行为切换。
**How**: 状态少用 enum+match；状态多/复用用 StateMachine 收集 State 子节点，状态发 transitioned 请求切换。
**Trade-offs**: Simple 快但会膨胀；State Pattern 清晰但脚本多。

## 视野感知（扇形视线）
**When to use**: 敌人不应 360° 全知、要留潜行空间。
**How**: RayCast2D look_at 玩家 + `abs(rotation) < π/2` + 命中判定；不同实现统一 `can_see_player()` 接口。
**Trade-offs**: 每帧射线有开销；探测频率/角度需调参。

## 局部避障（ShapeCast + 法线）
**When to use**: 轻量绕障、无导航时的补充。
**How**: 前方投射形状，`is_colliding()` 时新方向 = `transform.x + normal * force`。
**Trade-offs**: 只管局部，可能进死胡同；全局复杂地图交给 NavigationAgent。

## 全局导航 + RVO
**When to use**: 复杂地图跨点移动与多单位避让。
**How**: TileSet NavigationLayer 建网格；NavigationAgent2D 设 target，`get_next_path_position()` 转向，开 avoidance 走 set_velocity/velocity_computed。
**Trade-offs**: 导航图构建有一帧延迟；动态环境需重规划。

## PCG 地图流水线
**When to use**: 关卡需随机且可玩。
**How**: 关键点→草地→安全区→边界→PathGenerator+AStarGrid2D 路→free_cell 筛格→聚簇障碍→巡逻/炮塔点；单位与 Manager 从 map 取数据。
**Trade-offs**: 结果多样但质量需规则约束；调试用固定种子。

## 归一化 RL 传感器
**When to use**: 强化学习状态输入。
**How**: 射线距离除以 max_distance、速度除以 max_speed、金币差除以屏幕常量，输出 0~1；多射线 + 速度 = obs。
**Trade-offs**: 归一化稳训练但丢绝对信息；传感器布局要覆盖关键状态。

## Reward Shaping
**When to use**: RL 目标稀疏难学。
**How**: 每帧存活小正奖、关键事件大正奖、失败大惩罚；回合 done + auto reset。
**Trade-offs**: 奖励设计偏差会学出“原地横跳”；观察曲线再调。

## LLMAPI 网络封装
**When to use**: Godot 需要调用大模型/任意 HTTP API。
**How**: extends HTTPRequest；ConfigFile 读 key；组装 system+history messages；JSON.stringify POST；request_completed 解析后 emit 信号。
**Trade-offs**: 依赖外网与密钥管理；离线可换本地 Ollama/localhost。

## 桌面宠物穿透窗
**When to use**: 透明常驻窗口不挡桌面操作。
**How**: Borderless/Transparent/Always On Top + Polygon2D 转全局坐标赋 mouse_passthrough_polygon；每物理帧刷新；DisplayServer 拖窗。
**Trade-offs**: 多边形区域需随 UI 状态切换；平台差异需测试。

## 动态对象回收
**When to use**: 子弹/管道/敌人持续生成。
**How**: VisibleOnScreenNotifier2D.screen_exited → queue_free；重开清空容器；`await animation_finished` 后再删特效节点。
**Trade-offs**: 高频生成仍有实例化开销；重负载可用对象池。

## 输入分层
**When to use**: 选择正确的输入处理位置。
**How**: 全局键 `_input`；UI `_gui_input`；玩家控制 `_unhandled_input`；对象点选 `_input_event`；按住轮询 `Input.is_action_pressed`。
**Trade-offs**: 放错层会与 UI 抢事件；明确各层职责。

## 特效渲染隔离
**When to use**: CanvasModulate/光照下特效应保持高亮。
**How**: 整组放 CanvasLayer；单个精灵/粒子用 CanvasItemMaterial Unshaded；发光用 HDR2D + Glow + modulate>1。
**Trade-offs**: CanvasLayer 分层粗；Unshaded 每节点设材质略繁琐。

## 调试三板斧
**When to use**: 行为不符预期。
**How**: print 快速验证 → Remote 场景树看层级/改运行值 → 断点单步；导航另开 Debug Visible Navigation。
**Trade-offs**: print 最快但弱；断点最强但打断运行节奏。

## 树修改安全化
**When to use**: 碰撞回调/死亡链中要改树或属性。
**How**: queue_free 延迟释放；`set_deferred("monitoring", ...)`、`call_deferred("spawn")` 推迟到安全帧。
**Trade-offs**: 延迟语义要习惯；立刻读结果会读到旧值。

## Autotile / Terrain
**When to use**: 道路/河流等需要连续自动拼接。
**How**: TileSet Terrain Sets 设 match sides/corners，按 tile 标记边，再 set_cells_terrain_connect / Path 绘制。
**Trade-offs**: 初配繁琐；画起来极快且无接缝错误。
