# 第十四章：强化学习实战

## Core Idea
本章把《Flappy Bird》改造成强化学习训练场：以 SB3（Stable-Baselines3）为 Python 端“大脑”，用 Godot RL Agents 插件把 Godot 场景变成 Gym 风格环境；通过射线传感器把状态编码给模型，模型输出“跳/不跳”离散动作，环境回传奖励并自动重启，训练出一只真正会飞的智能小鸟。

## Frameworks Introduced
- **强化学习五要素**：Agent（决策者）、Environment（反馈世界）、State（观测）、Action（可选行为）、Reward（评价反馈）、Policy（行为准则）。
  - Loop: 看状态 → 选动作 → 环境反馈新状态+奖励 → 更新策略 → 重复至收敛。
- **探索 vs 利用（Exploration/Exploitation）**：先随机试错找新路径，再走熟悉好路线；奖励函数不合理会学到“绕远路/原地横跳”。
- **Stable-Baselines3（SB3）**：PyTorch 强化学习库，开箱提供 PPO/DQN/A2C/TD3/SAC；训练环境需兼容 Gym。
  - PPO: 稳定、高效、对奖励设计不敏感，最适合新手项目。
- **Godot RL Agents 架构**：Godot=环境服务器，Python=大脑；状态→动作→奖励→新状态通过网络端口循环。
- **action_repeat**：一个动作重复执行的帧数；赛车可 >1 加速收敛，Flappy Bird 用 1 保证每帧可控。
- **Sensor 设计（归一化观测）**：多个 RayCast2D 感知障碍，`get_distance()` 返回“到碰撞点距离 / max_distance”（0~1），未碰撞返回 1.0；金币用 Area2D 相对高度差 / 800 归一化。
- **奖励塑形（Reward Shaping）**：存活每帧 +0.01、得分 +1、死亡 −5；用小幅生存奖励对抗“什么都不做”的局部最优。
- **动作空间**：discrete size 2 = {不跳, 跳}；`set_action` 从 Python 动作字典取值。
- **Godot 插件机制**：addons/ + plugin.cfg；编辑器插件继承 EditorPlugin + @tool，生命周期 _enter_tree/_exit_tree；运行时插件（RL Agents）在游戏中使用。
- **训练闭环参数**：net_arch 小网络起步（[16]）、learning_rate 0.001 起步、n_steps=128、tensorboard 观察、先 human 调试再 training。

## Key Concepts
- **状态编码**：6 条障碍射线距离 + 金币纵向差 + 归一化速度 = 17 维观测（书中示例）。
- **normalize 观测**：距离除以最大探测长度、速度除以 max_speed，数值稳定在合理范围，模型更好学。
- **done / reset**：一轮结束置 done=true，Godot 自动 reset 场景，训练无需人工干预。
- **Sync 节点**：Godot 与 Python 同步组件；control mode 有 human/training。
- **AIController2D**：控制代理；`get_obs/get_reward/get_action_space/set_action` 四个接口是实现核心。
- **RayCast2D collide_with_areas**：管道是 Area2D，传感器需开启 Areas 而非 Bodies。
- **SB3 MultiInputPolicy + Dict obs**：插件把观测包成字典 {"obs": [...]}。
- **TensorBoard**：`tensorboard --logdir logs` 看奖励/损失曲线。
- **PPO 超参**：ent_coef 熵系数、n_steps 更新频率、learning_rate 学习步长。
- **@tool + Engine.is_editor_hint()**：脚本在编辑器与运行时行为分流。

## Mental Models
- Think of RL as 训练小狗：做对了给零食（+reward），做错了不给/扣分，行为慢慢成形。
- Think of Godot 游戏 as 训练场，Python 模型 as 大脑，网络端口 as 神经：状态送出、动作送回。
- Think of Sensor as 小鸟的六根“触须”：探到墙距离归一化为 0~1，探不到就是 1（“前方安全”）。
- Think of 奖励设计 as 给 AI 立规矩：只说“活得久”它会偷懒，必须惩罚撞墙、奖励过洞。
- Think of 网络大小 as 学习能力：先小后大，能学会就不要加复杂度。
- Think of learning_rate as 步子大小：太大乱跳不收敛，太小挪不动。

## Anti-patterns
- **奖励只给最终结果**：稀疏奖励难学；拆成生存/得分/死亡的密集塑形。
- **不归一化观测**：距离/速度量纲悬殊，网络难收敛；一律除以最大范围。
- **raycast 忘了开 Areas**：管道是 Area2D，射线永远撞不到，AI 变成瞎子。
- **人类按键逻辑残留**：训练时不走 AI 动作分支，模型指令无效；区分 human 分支与 else 分支。
- **失败后不自动重置**：训练中断；done=true 后 Godot 必须 reset。
- **网络一开始堆太大**：难训练、易过拟合；从 [16] 这类小网络起步。
- **learning_rate 盲目默认**：loss 抖降低、奖励不涨适度调高或加步数。
- **没用 TensorBoard/日志就盲调**：先看曲线再动超参。
- **直接在游戏场景里跑大模型**：RL 训练分离到 Python；Godot 只做环境与执行。

## Code Examples
Sensor（归一化射线距离）：
```gdscript
extends RayCast2D
class_name Sensor

enum Direction { Horizontal, Vertical, Degree45 }
@export var direction: Direction
var max_distance: float

func _ready() -> void:
    match direction:
        Direction.Horizontal:
            max_distance = abs(target_position.x)
        Direction.Vertical:
            max_distance = abs(target_position.y)
        Direction.Degree45:
            max_distance = abs(target_position.x) * sqrt(2)

func get_distance() -> float:
    if is_colliding():
        return global_position.distance_to(get_collision_point()) / max_distance
    return 1.0
```
- **What it demonstrates**: 碰撞距离归一化，未碰撞=安全。

AIController 四接口：
```gdscript
extends AIController2D

var move_action: int = 0

func get_obs() -> Dictionary:
    var obs = _player.sensors.get_observation()
    obs.append(_player.velocity.y / _player.max_speed)
    return {"obs": obs}

func get_reward() -> float:
    return reward

func get_action_space() -> Dictionary:
    return {"move_action": {"size": 2, "action_type": "discrete"}}

func set_action(action: Dictionary) -> void:
    move_action = action["move_action"]
```
- **What it demonstrates**: 观测、奖励、动作空间、动作注入的标准接口。

奖励与回合（bird.gd 片段）：
```gdscript
func _physics_process(delta: float) -> void:
    if not is_dead:
        ai_controller.reward += 0.01          # 每帧生存小奖励
        velocity.y += GameManager.GRAVITY * delta
        if ai_controller.move_action == 1:    # AI 分支
            velocity.y = GameManager.JUMP_VELOCITY
        move_and_slide()

func on_game_over() -> void:
    ai_controller.reward -= 5
    ai_controller.done = true
    ai_controller.reset()

func on_get_score() -> void:
    ai_controller.reward += 1
```
- **What it demonstrates**: 死亡惩罚、得分奖励、回合结束自动重置。

SB3 训练骨架：
```python
from stable_baselines3 import PPO
model = PPO(
    "MultiInputPolicy", env,
    policy_kwargs={"net_arch": [16]},
    learning_rate=0.001,
    n_steps=128,
    verbose=2,
)
model.learn(total_timesteps=1_000_000)
model.save("model/ppo_model.zip")
```
- **What it demonstrates**: 小网络 + PPO 起步；等 Godot 端口握手后自动采集训练。

## Reference Tables
| RL 要素 | 迷宫示例 | Flappy Bird 示例 |
|---|---|---|
| Agent | 机器人 | 小鸟 + AIController2D |
| Environment | 迷宫 | Godot main.tscn |
| State | 格子坐标 (2,3) | 17 维传感器观测 |
| Action | 上下左右 | 跳 / 不跳 |
| Reward | 出口+10、碰墙-1 | 存活+0.01/帧、得分+1、死亡-5 |

| 参数 | 建议起点 | 调法 |
|---|---|---|
| net_arch | [16] | 行为单一就加神经元 |
| learning_rate | 0.001 / 0.0003 | loss 抖降、奖励不涨升 |
| n_steps | 128 | 更新频率 |
| action_repeat | Flappy=1 | 抖动就微调 |
| timesteps | 10万快速验证 → 100万正式 | 看 TensorBoard |

| 角色 | 职责 |
|---|---|
| Godot + RL Agents | 环境模拟：物理、传感器、奖励、回合 |
| Python SB3 | 学习：策略更新、模型保存 |
| TensorBoard | 可视化：奖励/损失曲线 |
| Sync 节点 | 双方握手同步 |

## Worked Example
训练一只小鸟：main.tscn 加 Sync（先 human 调试）与自动开始/自动重置逻辑；bird 挂 6 个 Sensor（前上/前下水平、上下垂直、两条 45°）+ SensorCoin + AIController2D 子类。运行 Godot 训练场景与 `python stable_baseline_example.py`，Python 在 11008 端口等待握手；每帧拿到 17 维 obs，PPO（net_arch=[16]、lr=0.001）输出离散动作；Bird 按动作跳或不跳，活 1 帧 +0.01、穿管道 +1、撞死 −5 并 done/reset。1~2 万步后小鸟学会躲第一个管道，TensorBoard 可看奖励爬升。

## Key Takeaways
1. RL = Agent/State/Action/Reward/Policy 五件套 + 试错更新循环。
2. Godot 只做环境与表现，训练交给 Python SB3/PPO，松耦合最稳。
3. 观测必须归一化：射线距离/最大距离、速度/max_speed。
4. 奖励塑形决定行为：稀疏奖励难学，分解成生存/得分/死亡。
5. AIController 四个接口（obs/reward/action_space/action）是接入关键。
6. 小网络、合理学习率、先 human 验证再 training、TensorBoard 看曲线。
7. 回合结束置 done 并自动 reset，训练才能无人工跑通。

## Connects To
- **Ch 5**: 改造 Flappy Bird 为训练环境。
- **Ch 8**: RL 概念落地成项目。
- **Ch 13**: 训练成果可接入完整游戏/主菜单流程。
- **Ch 15**: 大模型接口与 RL 同属“现代 AI 上桌”路线。
