---
name: basic-godot-skill
description: "Godot 4.x 基础学习与 AI 编码知识库：《Godot游戏编程入门教程》（GDBook，肖老师）16 章提炼成可按需加载的 Agent Skill。面向初学者提供从节点/场景、GDScript、信号、碰撞到 FSM、游戏AI、导航、程序化地图、发布、强化学习与大模型集成的项目式学习路线；面向 AI 代理提供节点选择、代码范式、架构决策、反模式与调试规则，让 AI 编写或审查 Godot 项目时减少幻觉、按章节查证。Use when 学习 Godot、开发 2D 游戏、实现敌人 AI/寻路/程序化地图、接入 RL/LLM，或需要 Godot 代码与架构建议。"
---

<!-- argument-hint: [topic, framework name, 或章节号 ch01..ch16] -->

# Godot 基础技能（GDBook 16 章知识库）
**Skill**: `basic-godot-skill` | **原始资料**: 肖老师《Godot游戏编程入门教程》（https://gdbook.kidsgame.top，16 章电子预览版） | **Chapters**: 16 | **Generated**: 2026-09-02 | **Updated**: 2026-09-17

## 这个技能能做什么

### 给初学者：一条从零到完整项目的 Godot 学习路线

它不是术语词典，而是一套按学习顺序组织的 16 章路线：先建立游戏与引擎认知（Ch 1–3），再掌握 GDScript 与信号（Ch 4），随后通过《Flappy Bird》《Battle Tank》两个完整项目把知识串起来（Ch 5–6），再进入组件化重构、游戏 AI、状态机、感知避障、导航与程序化地图（Ch 7–12），最后完成视觉打磨与发布、强化学习与大模型集成、面向对象架构与代码风格（Ch 13–16）。适合：

- 按章节系统自学，每章都有核心概念、代码示例、反模式与实战示例；
- 做项目时快速回查节点选择、API 写法与设计取舍；
- 通过 cheatsheet 检查自己的实现是否符合 Godot 的惯用做法。

### 给 AI 代理：可加载、可查证的 Godot 工程知识库

把整本书塞进上下文既昂贵又容易失真。这个技能让 AI **按主题或章节按需加载**：无参数时读核心框架；问到某个主题时先查 Topic Index，再读对应章节。它的价值是：

- **减少 API 幻觉**：节点、信号、函数名与 Godot 4.x 写法有明确出处和章节；
- **提供工程判断而非零散答案**：碰撞层怎么分、什么时候用组合而不是继承、状态机该选哪种实现、AI 该用局部避障还是导航，都有决策规则；
- **支持代码生成与审查**：可基于章节里的范式生成 GDScript、组件、状态机与项目结构，并用反模式清单检查实现；
- **跨 Agent Skills 主机可用**：遵循 SKILL.md + chapters 的开放格式，Codex、Claude Code、GitHub Copilot CLI、Amp、Hermes 等兼容主机都可安装使用。

### 不适合用来做什么

- 不是 Godot 官方文档的替代；具体版本 API、属性名与行为请以目标 Godot 版本的官方文档为准。
- 不包含原书全文；章节是结构化提炼与代码示例。
- 不保证覆盖 3D、C#、移动端发布等原书预览版未深入的主题。

## How to Use This Skill

- **无参数** — 直接加载核心框架与速查。
- **带主题** — 问 `delta`、`信号`、`FSM`、`导航`、`程序化地图`、`强化学习` 等，先看 Topic Index 再读对应章节。
- **带章节** — 问 `ch05`、`ch09`，直接加载该章文件。
- **带任务** — 如“做 Flappy Bird”“让敌人巡逻追击”“调大模型对话”，读相关章节并按 cheatsheet 决策。

主题未出现在下方核心框架时，先读 Topic Index 对应章节再回答，不要凭印象编 Godot API。

---

## Core Frameworks & Mental Models

- **Learning by Doing + 教程陷阱**：跟随教程跑通后删除代码、凭记忆重写；能独立重建才算学会。做游戏 = 在无序中构建有序（Ch 1）。
- **游戏循环与 Delta Time**：世界靠“输入→更新→渲染”每帧运转；一切基于时间的位移都乘 delta，否则低帧设备变慢。物理逻辑放 `_physics_process`，视觉/轮询放 `_process`（Ch 2）。
- **节点 + 场景 = 舞台与积木**：节点是功能积木（类实例），场景是可保存/实例化/继承的节点组合（Packed Scene ≈ OOP 类）。子节点继承父节点空间变换；`$Child` 引用子节点，`@onready` 安全赋值（Ch 3）。
- **GDScript 心智**：脚本挂载=创建节点子类；Inspector 属性皆可代码访问。参数用 `@export`、节点引用用 `@onready`，两者不混用（Ch 4）。
- **信号解耦与 Signal Hub**：跨模块通知走 signal（定义→connect→emit），发送者不认识接收者；全局状态与胜负用 GameManager Autoload 单例，别让场景互相直调成蜘蛛网（Ch 4/5/7）。
- **组件组合优于继承**：功能拆成 HurtBox/HitBox/Health/Weapon/Detect/Nav 组件按需挂载；伤害用接口契约（has_method + get_hurt），碰撞关系用 Layer/Mask 矩阵而不是组标签过滤（Ch 6/7）。
- **FSM 管行为**：离散互斥行为用状态机；状态少 enum+match，状态多/复用升级 State Pattern（StateMachine + State 子节点 + transitioned 信号）。连续/叠加状态别硬套（Ch 8/9）。
- **AI 移动三件套**：感知（扇形 RayCast/Area 轮询）→ 局部避障（ShapeCast 法线）→ 全局导航（TileSet NavigationLayer + NavigationAgent2D + RVO 动态避障）；状态机按 patrol/wander/attack 切换导航层（Ch 10/11）。
- **PCG = 随机 + 约束**：地图生成按固定流水线（草地→安全区→边界→AStarGrid2D 路→free_cell 筛格→聚簇障碍→巡逻/炮塔点）；巡逻点从道路动态取，敌人/玩家从 map 取坐标，不写死（Ch 12）。
- **打磨与流程**：氛围用 CanvasModulate + 2D 光 + Occluder；特效防变暗用 CanvasLayer 或 Unshaded；发光需 HDR2D+Glow+颜色>1；BGM 用 SoundManager Autoload，关卡流程用 LevelManager（Process Mode Always）（Ch 13）。
- **现代 AI 接入**：RL 训练把 Godot 当环境、Python SB3/PPO 当大脑（归一化 obs、奖励塑形、AIController 四接口）；LLM 用 HTTPRequest 封装 LLMAPI 组件 + ConfigFile 密钥 + system prompt 人设 + history 上下文（Ch 14/15）。
- **OOAD 与工程化**：先分析对象与责任（类图/顺序图）再写码；设计→原型→反馈→重构循环；SOLID 与单例/观察者/工厂/状态四模式让项目可维护可扩展；按官方风格命名并统一目录（Ch 16）。

---

## Chapter Index

| # | Title | Key Frameworks |
|---|-------|----------------|
| [ch01](chapters/ch01-player-to-developer.md) | 从玩家到开发者 | 游戏四要素、心流、三阶段流程、纸面原型 |
| [ch02](chapters/ch02-2d-basics.md) | 2D 游戏开发技术基础 | 游戏循环、Delta Time、OOP、碰撞、事件、向量/Lerp、色彩 |
| [ch03](chapters/ch03-godot-quickstart.md) | Godot 引擎快速入门 | 节点+场景、Packed Scene、资源引用、AnimationPlayer |
| [ch04](chapters/ch04-gdscript-basics.md) | GDScript 编程基础与实践 | 挂载脚本、变量修饰符、输入、信号、生命周期 |
| [ch05](chapters/ch05-flappy-bird.md) | 入门项目实战 Flappy Bird | 模块化、信号总线、视差滚动、动态生成/回收 |
| [ch06](chapters/ch06-battle-tank.md) | 进阶项目实战 Battle Tank | MVP、Signal Hub、top_level、move_toward、VFX 解耦 |
| [ch07](chapters/ch07-refactor-components.md) | 项目优化重构 | 组合组件、Layer/Mask、伤害契约、Manager、await 链 |
| [ch08](chapters/ch08-game-ai-overview.md) | 游戏 AI 基础与演化 | 游戏 AI 哲学、FSM、感知避障、寻路、PCG、RL、LLM |
| [ch09](chapters/ch09-finite-state-machine.md) | 游戏中的有限状态机 | Simple FSM、State Pattern、状态机调度 |
| [ch10](chapters/ch10-perception-avoidance.md) | 感知系统与避障 | 扇形视野、鸭子类型、ShapeCast 避障、索敌、追踪弹 |
| [ch11](chapters/ch11-pathfinding-navigation.md) | 路径规划与导航系统 | NavigationLayer、NavigationAgent2D、RVO、状态×导航 |
| [ch12](chapters/ch12-procedural-map.md) | 程序化地图生成 | PCG 流水线、AStarGrid2D、free_cell、PathGenerator |
| [ch13](chapters/ch13-visual-polish-publish.md) | 视觉优化与发布 | CanvasModulate、光/阴影/Glow、LevelManager、导出 |
| [ch14](chapters/ch14-reinforcement-learning.md) | 强化学习实战 | RL 五要素、SB3/PPO、Godot RL Agents、奖励塑形 |
| [ch15](chapters/ch15-llm-desktop-pet.md) | 大模型打造 AI 桌面宠物 | LLMAPI、HTTPRequest、穿透窗、桌面宠物 |
| [ch16](chapters/ch16-oop-design-style.md) | 面向对象设计与编程风格 | OOAD/UML、SOLID、设计模式、GDScript 风格、目录 |

## Topic Index

- **2D 光照/阴影/Glow** → ch13
- **AI 敌人行为** → ch08, ch10
- **AnimationPlayer / Tween** → ch03, ch07
- **Autoload 单例** → ch05, ch13
- **避障（ShapeCast / RVO）** → ch08, ch10, ch11
- **碰撞（Layer/Mask/节点选择）** → ch02, ch07, cheatsheet
- **场景复用/实例化/继承** → ch03, ch06, ch12
- **程序化地图 PCG** → ch08, ch12
- **Delta Time / 帧率无关** → ch02, cheatsheet
- **GDScript 语法与风格** → ch04, ch16
- **信号 / 事件驱动** → ch02, ch04, ch05
- **组件化设计** → ch07, ch16
- **有限状态机（FSM/状态模式）** → ch08, ch09, ch16
- **伤害接口 HurtBox/HitBox** → ch07
- **寻路导航 NavigationAgent** → ch08, ch11
- **路径巡逻** → ch06(PathFollow2D), ch09, ch11
- **强化学习 RL** → ch08, ch14
- **大语言模型 LLM** → ch08, ch15
- **入口/菜单/关卡流程** → ch13
- **鼠标穿透桌面窗口** → ch15
- **输入映射与处理** → ch04, ch07, cheatsheet
- **资源系统（res:// / .tres / TileSet）** → ch03, ch07, ch12
- **视差滚动** → ch05
- **射击/子弹/追踪弹** → ch06, ch10
- **像素级调试（print/Remote/断点）** → ch06, ch11
- **自动生成单位/波次** → ch07, ch12

## Supporting Files

- [glossary.md](glossary.md) — 全书关键术语速查
- [patterns.md](patterns.md) — 实战模式/技术清单
- [cheatsheet.md](cheatsheet.md) — 决策表与经验规则

---

## Scope & Limits

- 覆盖网站 gdbook.kidsgame.top 公开的 16 章电子预览版内容（Godot 4.x、2D、GDScript 为主）。
- 章节为结构化提炼与代码示例，不是原书全文；涉及具体版本号（如 Godot 4.6 的 Parallax2D）时以目标版本官方文档为准。
- 生成的技能文件内容为二次创作摘要，不用于替代原书；原始书稿与插图的版权归原作者/权利人，商用与再分发请遵循原站及出版社条款。
- 实战落地请结合项目版本与官方文档核对 API。
