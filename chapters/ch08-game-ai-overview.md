# 第八章：游戏AI基础与演化

## Core Idea
本章建立游戏 AI 的完整认知：游戏 AI 与通用 AI 的根本区别是“让机器表演”而非“让机器思考”；随后梳理从吃豆人到 LLM 的演进，掌握随机决策、FSM、感知避障、寻路、PCG、RL、LLM 七类核心技术，并用《吃豆人》幽灵 AI 说明“简单规则 + 差异化设计 = 深度体验”。

## Frameworks Introduced
- **游戏 AI 设计哲学**：服务体验优先于追求智能——AI 应“足够聪明但不至于绝对击败玩家”。
  - 拟人化缺陷：会犹豫（给玩家预警）、会犯错（让玩家能反制）、会沟通（用台词暗示意图）。
  - Success: 可解释、可调节、可预测，让玩家有掌控感，而不是黑盒最强对手。
- **四支柱定义 AI**：感知、推理、学习、行动；游戏 AI 只借用表现需要的部分。
- **随机概率决策**：用随机数打破行为周期感；配合权重（狂暴 80% 攻击 / 20% 咆哮）和情境限定，避免过度随机失去逻辑。
- **有限状态机（FSM）**：状态 + 转换；每个状态只做一件事，含 Enter/Update/Exit；复杂时用分层状态机（HFSM）或节点式 FSM。
- **感知与避障**：视野锥 + 视线遮挡 + 警觉值（延迟识别）；射线预判 + 转向力实现平滑避障；NavigationAgent 内置 RVO。
- **寻路导航**：A* 启发式搜索 + 导航网格（NavMesh）抽象地形 + 路径平滑；NavigationServer/NavigationRegion/NavigationAgent/AStar2D 是 Godot 四层工具。
- **程序化内容生成（PCG）**：随机种子 + 规则约束 = 海量变化；噪声、细胞自动机、WFC、房间模块拼接是四大方法；核心是“随机性与约束性的平衡”。
- **强化学习（RL）**：Agent 通过 State-Action-Reward 试错学策略；适合离线训练、QA 探路、复杂策略；混合 AI（行为树框架 + 局部 ML 决策）是务实路线。
- **大语言模型（LLM）**：给 NPC 语言表达；用 Prompt 写“人设”、RAG 注入设定知识、情感状态机联动表情动作。
- **Pac-Man 差异化 AI**：四幽灵各用不同目标算法（直接追击/前方 4 格伏击/基于 Blinky 双倍矢量/距离阈值折返），配合三态切换产生战术深度。

## Key Concepts
- **通用 AI vs 游戏 AI**：追求真实准确 vs 追求好玩可控；实时性要求完全相反。
- **A*（启发式搜索）**：用启发估计引导搜索，求近似最短路径，行业标准。
- **NavMesh / NavigationRegion / NavigationAgent / NavigationLink**：可行走区域抽象、区域节点、逐帧代理、跨区连接。
- **FastNoiseLite**：Perlin/Simplex/Cellular 噪声，用于地形高度图与 TileMap 阈值划分。
- **细胞自动机 / WFC / Room Prefabs**：洞穴迭代生成、约束传播坍缩、模块拼接。
- **Godot RL Agents**：把场景包装成 Gym 风格环境；训练通常在 Python，推理可走 ONNX Runtime 或 HTTP/Socket 服务。
- **RVO（Reciprocal Velocity Obstacles）**：NavigationAgent 内置的多智能体避障。
- **RandomNumberGenerator**：Godot 随机 API（randf/randi_range/rand_weighted/randfn）。
- **AnimationTree State Machine**：Godot 内置的可视化 FSM，常用于动画与 Boss 阶段。
- **LimboAI**：HSM + Behavior Tree 融合的 Godot 专业 AI 插件。
- **Chase / Scatter / Frightened**：吃豆人幽灵追逐、分散、恐慌三态。

## Mental Models
- Think of 游戏 AI as 导演与演员的合体：按脚本演，又保留即兴，最终目的是制造“合适的对手”。
- Think of 工业 AI as “让机器思考”，游戏 AI as “让机器表演”。
- Think of FSM as 大脑的开关盒：巡逻→发现→追击→攻击，状态清晰可调。
- Think of 感知 as 眼睛、避障 as 双腿：先看到（视野锥+射线），再走顺（转向力/导航）。
- Think of PCG as 炼金术：给算法随机种子与规则炉子，产出无限但受控的世界。
- Think of RL as 迷宫中的孩子：状态感知、动作选择、奖励修正，长期累积最大化。
- Think of 最强 AI 不一定是好 AI：完美的上帝视角狙击手会剥夺玩家生存空间。

## Anti-patterns
- **追求“最强/最聪明”对手**：计算完美的 AI 是设计灾难；要拟人化缺陷与可预测破绽。
- **随机不加约束**：行为无逻辑自相矛盾，玩家觉得不公平；随机应限定在合理状态与权重内。
- **FSM 状态无限膨胀**：状态与转换几何级增长成“意大利面”；用 HFSM 分层或组件状态机。
- **每帧对每个 NPC 做全量感知检测**：CPU 爆炸；降低刷新频率、空间划分、批处理。
- **动态环境还依赖烘焙 NavMesh 旧路径**：路径被堵就撞墙；需定期重寻路或结合局部避障。
- **纯随机 PCG 无约束**：关卡杂乱无章、不可通关；坚持“算法 + 人工润色”。
- **把 RL 模型直接当黑盒进游戏**：不可控、难调试；用离线训练 + 混合 AI + QA Bot。
- **NPC 胡说八道**：LLM 直接上线不设防；用 RAG 注入世界观 + Prompt 人设 + 过滤。

## Code Examples
Godot 随机概率决策（权重攻击）：
```gdscript
var rng = RandomNumberGenerator.new()
rng.randomize()

func decide() -> void:
    if rng.randf() < 0.8:
        attack()
    else:
        roar()
```
- **What it demonstrates**: 用 randf 实现 80/20 权重决策。

视野 + 视线校验：
```gdscript
func can_see(target: Node2D) -> bool:
    if not detect_area.has_overlapping_bodies():
        return false
    ray_cast_2d.target_position = ray_cast_2d.to_local(target.global_position)
    ray_cast_2d.force_raycast_update()
    return ray_cast_2d.get_collider() == target
```
- **What it demonstrates**: Area 范围检测负责“在范围内”，RayCast 负责“没被墙挡”。

## Reference Tables
| 维度 | 通用 AI | 游戏 AI |
|---|---|---|
| 目标 | 真实智能/准确 | 表现智能/娱乐性 |
| 技术 | 数据驱动泛化 | 规则驱动可调 |
| 成功标准 | 是否智能 | 是否好玩 |
| 实时性 | 可容忍延迟 | 每帧必须快 |
| 复杂度 | 模型庞大 | 控制复杂度 |

| 技术 | 优点 | 局限 |
|---|---|---|
| 随机决策 | 简单、不可预测 | 过度随机失逻辑 |
| FSM | 直观高效 | 状态爆炸 |
| 感知避障 | 逼真、不撞墙 | 性能/调参 |
| 寻路 A*+NavMesh | 高效自然 | 动态环境弱 |
| PCG | 海量内容 | 质量控制难 |
| RL | 复杂策略强 | 黑盒、训练贵 |
| LLM | 对话自由 | 一致性/成本 |

| 吃豆人幽灵 | 追逐策略 | 散点 |
|---|---|---|
| Blinky | 直接追玩家 | 右上 |
| Pinky | 玩家前方 4 格 | 左上 |
| Inky | Blinky→前方2格点 双倍矢量 | 右下 |
| Clyde | >8 格追、<8 格放弃 | 左下 |

## Worked Example
《吃豆人》幽灵状态机：开局进 Scatter（分散 7s 回各自角落）→ Chase（追击 20s）交替 4 轮；吃能量豆立即切 Frightened，全幽灵变蓝减速 50%、路口伪随机转向、闪烁提示后恢复；Blinky 剩 20 粒豆进入 Cruise Elroy 提速且无视 Scatter。路口决策用贪心：计算每个可选方向下一格到目标的直线距离选最近，平局按 上>左>下>右 打破——没有 A*，但节奏与差异让玩家几十年都玩不腻。

## Key Takeaways
1. 游戏 AI 的目标是“最合适的演员”而非“最强的对手”。
2. FSM 是行为骨架，HFSM/节点式状态机解决复杂度爆炸。
3. 感知 = 范围 + 视线 + 延迟警觉；避障 = 射线预判 + 转向力/RVO。
4. 寻路用 A* + 导航网格；动态环境要配合重寻路与局部避障。
5. PCG 的关键是约束下的随机，纯随机不是程序化设计。
6. RL/LLM 先离线/外部服务化，Godot 只做推理与表现，避免黑盒失控。
7. 差异化的简单规则（四幽灵）远比单一聪明算法更有深度。

## Connects To
- **Ch 9**: FSM 将从概念落到坦克状态机代码。
- **Ch 10**: 视野感知、避障组件实战。
- **Ch 11**: NavigationAgent/A* 寻路实战。
- **Ch 12**: PCG 程序化地图实战。
- **Ch 14/15**: RL 训练 Flappy Bird、LLM 驱动桌面宠物落地。
