# Glossary

**AABB** — 轴对齐包围盒，最简单快速的矩形碰撞检测（Ch 2）
**Action（动作空间）** — RL 中智能体可执行行为的集合，如 Flappy Bird 的 {不跳, 跳}（Ch 14）
**AIController2D** — Godot RL Agents 的控制代理节点，实现 get_obs/get_reward/get_action_space/set_action（Ch 14）
**Area2D** — 区域感应节点，可检测进入/离开但不阻挡物体（Ch 2）
**AnimationPlayer** — 时间轴关键帧动画控制器（Ch 3）
**AnimatedSprite2D** — 序列帧动画节点，管理 SpriteFrames（Ch 5）
**Autoload（自动加载/单例）** — 项目设置注册、全局唯一可访问的节点，如 GameManager/SoundManager（Ch 5/13）
**AStarGrid2D** — 网格专用 A* 寻路类，region/diagonal_mode/get_id_path（Ch 12）
**await 链** — 含 await 的函数必须被上层 await，否则异步顺序错乱（Ch 7）
**@export** — 暴露到 Inspector 的参数修饰符（Ch 4）
**@onready** — 节点就绪后赋值子节点引用的修饰符（Ch 4）
**行为树（Behavior Tree）** — 2000s 主流模块化 NPC 行为架构（Ch 8）
**C# / C++ / GDScript** — Godot 支持的语言，入门推荐 GDScript（Ch 4）
**CanvasLayer** — 独立画布渲染层，UI/特效不受 CanvasModulate 影响（Ch 5/13）
**CanvasItemMaterial / Unshaded** — 跳过光照与滤镜的材质模式（Ch 13）
**CanvasModulate** — 全画布颜色滤镜，模拟昼夜氛围（Ch 13）
**CharacterBody2D** — 代码控制移动+碰撞的角色体，自带 move_and_slide（Ch 2/5）
**CollisionShape2D** — 定义碰撞形状的组件（Ch 2）
**碰撞层/遮罩（Layer & Mask）** — 我是谁 / 我检测谁，物理引擎底层过滤碰撞（Ch 7）
**组合优于继承** — has-a 比 is-a 更灵活；功能组件按需挂载（Ch 7/16）
**Control 容器** — UI 自动布局系统（Margin/VBox/HBox/GridContainer）（Ch 5）
**DetectComponent** — 探测组件，get_overlapping_* 轮询索敌（Ch 7/10）
**Delta Time（Δt）** — 帧间真实时间差，帧率无关移动的基础（Ch 2）
**鸭子类型（Duck Typing）** — 只要提供 can_see_player() 就用，不看类型（Ch 10）
**DIP 依赖反转** — 上层依赖抽象接口而非具体实现（@export 注入）（Ch 16）
**enum** — 枚举命名常量组，Simple FSM 的状态集合（Ch 9）
**FSM（有限状态机）** — 状态+转换+动作的行为建模；实现有状态变量与状态模式（Ch 8/9）
**游戏循环（Game Loop）** — 输入→更新→渲染循环，一帧一次（Ch 2）
**GameManager** — 全局信号/分数/流程中枢（Autoload）（Ch 5/7）
**Godot RL Agents** — Godot-Python RL 通信插件（Ch 14）
**HDR 2D + Glow** — 高动态范围与发光后期特效（Ch 13）
**HealthComponent** — 血量管理组件，emit health_changed/died（Ch 7）
**HurtBox / HitBox 接口** — 承伤 get_hurt 与攻击 apply_hit 契约，has_method 检查（Ch 7）
**HTTPRequest** — Godot HTTP 客户端节点，request_completed 回调（Ch 15）
**HUD** — 抬头显示界面（Ch 5/6）
**Input Map** — 动作名↔按键映射，代码不写死键位（Ch 4）
**Input.get_vector** — 多方向输入合成归一化向量（Ch 6）
**instantiate()** — 场景文件实例化，加入场景树参与生命周期（Ch 5/12）
**Lerp / move_toward / rotate_toward** — 平滑过渡与趋近函数（Ch 2/6）
**LightOccluder2D / Occlusion Layer** — 2D 阴影遮挡定义（Ch 13）
**Line2D** — 动态点数组画轨迹线（Ch 7）
**LLM（大语言模型）** — 预测性语言生成模型；游戏里做对话/设计/代码（Ch 8/15）
**LLMAPI 组件** — HTTPRequest 封装：密钥/历史/请求/解析/信号（Ch 15）
**Marker2D** — 出生点/炮口等可视化坐标标记（Ch 5）
**move_and_slide** — CharacterBody2D 移动方法，自动处理碰撞滑动（Ch 2/6）
**mouse_passthrough_polygon** — 桌面窗口可点击穿透多边形（Ch 15）
**NavigationAgent2D** — 路径规划代理：target_position/get_next_path_position/RVO（Ch 8/11）
**NavigationLayer（导航图层）** — 行为级通行规则，如 patrol=道路、wander=草地（Ch 11）
**new() vs instantiate()** — RefCounted 逻辑类 vs 场景实体（Ch 12）
**对象池/延迟回收** — VisibleOnScreenNotifier2D+queue_free 及时清理动态物（Ch 5）
**PCG（程序化内容生成）** — 算法生成地图/内容，随机+约束（Ch 8/12）
**PPO** — Proximal Policy Optimization，稳定易用的 RL 算法（Ch 14）
**PointLight2D / DirectionalLight2D** — 2D 点光/平行光（Ch 13）
**Polygon2D** — 多边形；桌面宠物用它限定鼠标区域（Ch 15）
**project.godot** — 项目入口配置文件（Ch 3）
**RandomNumberGenerator** — Godot 随机类：randf/randi_range/rand_weighted/randfn（Ch 8）
**RayCast2D** — 射线检测：视线、传感器、探路（Ch 10/14）
**RefCounted** — 轻量引用计数逻辑基类（Ch 12）
**res:// / user://** — 项目资源路径 / 运行时用户数据路径（Ch 3）
**reward shaping（奖励塑形）** — 密集奖励设计：存活/得分/死亡（Ch 14）
**RVO** — Reciprocal Velocity Obstacles 多智能体互相让行（Ch 8/11）
**Scene（场景）** — 节点树组成的可复用模块，保存 .tscn（Ch 3）
**Shader（CanvasItem）** — GPU 像素着色器，uniform + fragment 实现闪白等（Ch 7）
**ShapeCast2D** — 形状投射探测，get_collision_normal 求法线避障（Ch 10）
**Signal Hub（信号总线）** — 全局集中信号（Ch 6/7）
**SKILL/Agent Skills** — 结构化技能：SKILL.md + chapters + 索引（全书）
**Sprite2D** — 显示单张图像的 2D 节点（Ch 3）
**SRP（单一职责）** — 一个类只负责一件事（Ch 16）
**State 基类** — enter/exit/physics_update + transitioned 信号（Ch 9）
**Tween** — 代码式属性补间动画（Ch 7）
**TileSet / TileMapLayer** — 瓦片仓库与图层；set_cell 编程铺图（Ch 3/12）
**top_level** — 子节点提升为世界级，摆脱父节点变换（Ch 6）
**transform.x** — 本地 X 轴方向向量，沿朝向移动的标准写法（Ch 6）
**Unshaded** — 见 CanvasItemMaterial（Ch 13）
**Vector2** — 2D 向量：位置/方向/速度；标准化防斜向加速（Ch 2）
**velocity_computed** — NavigationAgent 动态避障安全速度信号（Ch 11）
**VisibleOnScreenNotifier2D** — 屏幕进出信号（Ch 5）
**WFC / 细胞自动机 / Perlin Noise / BSP** — PCG 常见算法（Ch 8/12）
