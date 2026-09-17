# Cheatsheet

## 物理节点选择（先问：动还是静？谁控制？）
| 需求 | 节点 |
|---|---|
| 静止地面/墙 | StaticBody2D |
| 物理引擎全托管 | RigidBody2D |
| 代码控制移动+碰撞 | CharacterBody2D |
| 只感应不阻挡 | Area2D |
| 定义形状 | CollisionShape2D（挂在上面之一） |

## 回调分工
| 想做的事 | 放哪 |
|---|---|
| 一次性初始化/连信号 | `_ready()` |
| 重力/速度/碰撞/移动 | `_physics_process(delta)` |
| 视觉/轮询输入/UI 非物理 | `_process(delta)` |
| 实例化时数据初始化 | `_init()`（不能访问子节点） |

## 输入决策
| 场景 | 写法 |
|---|---|
| 玩家持续移动 | `Input.get_vector()` / `is_action_pressed` |
| 玩家点击/跳跃一次性 | `is_action_just_pressed` / `_unhandled_input` |
| UI 按钮 | pressed.connect + `_gui_input` |
| 全局快捷键 | `_input` |
| 点选世界对象 | `_input_event` |

## 跨对象通信决策树
1. 两个节点父子/强关联可直接访问？ → 函数调用即可。
2. 事件要 2+ 模块响应或模块互不相识？ → Signal Hub（GameManager 定义）。
3. 发送者触发、单点订阅？ → 本地信号 connect/emit。
4. 全局唯一状态/流程？ → Autoload 单例。

## Layer/Mask 经验矩阵
| 单位 | Layer | Mask 检测 | 效果 |
|---|---|---|---|
| 玩家弹 | 1 | 4,5 | 只伤敌承伤+墙 |
| 玩家体 | 6 | 5,7,8 | 撞墙/敌/拾道具 |
| 敌弹 | 3 | 2 | 只伤玩家承伤 |
| 敌 HurtBox | 4 | 1 | 接收玩家弹 |
| 障碍 | 5 | 6,7 | 挡双方物理体 |

## FSM 取舍
| 情况 | 方案 |
|---|---|
| 状态 ≤3、原型 | enum + match 单脚本 |
| 状态 >5、多人复用、AI 复杂 | StateMachine + State 子节点 |
| 连续/叠加状态（移动+中毒） | HFSM/行为树，别硬套平面 FSM |

## 敌人移动能力搭配
| 目标 | 方案 |
|---|---|
| 巡固定路线 | Path2D + PathFollow2D progress |
| 轻量绕石头 | ShapeCast2D 法线转向 |
| 全图绕墙寻路 | TileSet NavigationLayer + NavigationAgent2D |
| 多个敌人互让 | NavigationAgent avoidance + set_velocity |
| 追到射程停 | Attack 状态 min_dist 判断 stop() |

## 设计架构速查
| 原则 | 一句话检查 |
|---|---|
| SRP | 这个脚本能只说清“我做一件事”吗？ |
| OCP | 加新功能能不碰旧代码吗？ |
| LSP | 子类替换父类会崩吗？ |
| ISP | 炮塔被迫实现了 move() 吗？ |
| DIP | 高层知道具体 res:// 路径吗？ |
| 组合优于继承 | 是否该把能力拆成组件再拼？ |

## 常见“气味”与对策
| 看到 | 对策 |
|---|---|
| 一堆布尔开关控行为 | FSM/状态模式 |
| 到处 `if is_in_group` 过滤碰撞 | Layer/Mask |
| 死亡节点挂动画 | VFXManager 解耦 |
| 对象越积越多 | screen_exited queue_free / 容器清空 |
| UI 硬编码坐标 | 容器+锚点+CanvasLayer |
| 帧率影响速度 | 检查是否都乘 delta |
| BGM 切场景断 | SoundManager Autoload |
| 碰撞回调里改树 | set_deferred/call_deferred |

## Godot 坐标/方向铁律
- 2D 原点左上，Y 向下为正。
- 子节点 position 相对父；跨层级对齐用 global_position。
- “前方”= 本地 X 轴：`transform.x`；贴图默认朝右再 look_at。
- 动态发射物设 `top_level = true`，别让子弹被父节点旋转“甩弯”。
- 斜向移动前 normalize，否则快 √2 倍。

## 资源/路径
| 前缀 | 用途 | 说明 |
|---|---|---|
| res:// | 项目资源 | 只读资源引用 |
| user:// | 用户数据 | 存档/设置 |
| .tscn | 场景 | 实例化复用 |
| .tres | 资源数据 | TileSet/Theme/LabelSettings 共享 |

## GDScript 风格 30 秒版
- 变量/函数 snake_case；类 PascalCase；常量 SCREAMING_SNAKE。
- 类型提示：`var speed: float`；函数返回 `-> void`。
- 空值检查：`is_instance_valid()` / `!= null`，别盲信节点存在。
- 函数 ≤30 行、单行 ≤80、早期返回减嵌套。
- 脚本顺序：extends → signals → const → @export → var/@onready → 生命周期 → 逻辑 → 私有辅助。

## RL/LLM 快速起步
| 事项 | 默认 |
|---|---|
| RL 算法 | PPO，net_arch=[16]，lr≈0.001 |
| 观测 | 归一化 0~1 |
| 奖励 | 存活小正、得分正、死亡大负 |
| 先跑 | Sync=human 手测，再 training |
| LLM | HTTPRequest 封装 + ConfigFile 密钥 + JSON |
| LLM 上下文 | system prompt + history.slice(-N) |
