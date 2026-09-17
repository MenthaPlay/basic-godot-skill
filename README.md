# Godot 基础技能 · basic-godot-skill

> 把《Godot游戏编程入门教程》（GDBook，肖老师）16 章公开内容提炼成一个可按需加载的 **Agent Skill**：既能带初学者从零走完 Godot 4.x 2D 游戏开发路线，也能让 AI 代理在写 Godot 代码时查得到节点、范式、架构决策与反模式。

## 这是什么

`basic-godot-skill` 是一个遵循 [Agent Skills](https://github.com/agentskills/agentskills) 开放格式的技能包：

- **对人**：一套 16 章的学习与速查资料，覆盖从“游戏是什么”到“发布一个完整 2D 游戏”，再到游戏 AI、程序化地图、强化学习与大模型接入。
- **对 AI**：一个可被 Codex、Claude Code、GitHub Copilot CLI、Amp、Hermes Agent 等主机加载的知识库。AI 只在需要时读取对应章节，而不是把整本书塞进上下文。
- **对项目**：一套 Godot 工程决策清单——节点怎么选、信号怎么连、碰撞层怎么分、状态机怎么拆、什么时候用组合而不是继承。

章节是结构化提炼与代码示例，**不是原书全文**。

## 为什么值得用

### 给初学者：少走弯路的学习路线

很多 Godot 教程的问题是：知识点零散、跳步、只教“怎么做”不教“为什么”。这个技能按真实开发顺序组织：

1. 先建立对游戏、引擎、节点与场景的整体认知；
2. 再学 GDScript、信号、输入、碰撞与资源系统；
3. 用《Flappy Bird》和《Battle Tank》两个完整项目把零散知识串起来；
4. 进入组件化重构、FSM、感知、避障、导航与程序化地图；
5. 最后完成视觉优化、关卡流程、发布，并接触强化学习和大模型。

每一章都包含：核心思想、框架与关键概念、心智模型、反模式、代码示例、参考表和实战示例。你可以按顺序学，也可以在项目卡住时按主题直接查。

### 给 AI 代理：可查证的 Godot 工程知识库

直接问 AI “Godot 敌人 AI 怎么写”，答案往往正确但不完整：它能给出一个状态机，却不知道你的项目是组件式架构，也不知道导航层该怎么配。这个技能提供的是**有章节出处、有决策规则的工程知识**：

- 节点/API/信号名有明确章节来源，减少版本幻觉；
- 提供“什么时候用 Area2D、什么时候用 CharacterBody2D”“局部避障和 NavigationAgent2D 怎么配合”这类决策依据；
- 自带反模式清单，可用来审查 AI 生成的代码；
- 按需加载章节，控制上下文成本；
- 兼容 Agent Skills 标准，可跨主机安装与复用。

## 内容结构

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 技能入口：核心框架、16 章索引、主题索引、使用方式 |
| `chapters/ch01–ch16` | 16 章结构化摘要：概念、代码、反模式、实战示例 |
| `glossary.md` | 全书关键术语表（节点、模式、API） |
| `patterns.md` | 20+ 实战模式：用途、做法、取舍 |
| `cheatsheet.md` | 决策速查：节点选择、碰撞层、FSM、导航、代码风格 |

## 16 章学习地图

| 阶段 | 章节 | 你会掌握 |
|---|---|---|
| 建立认知 | Ch 1–3 | 游戏构成、开发流程、2D 原理、Godot 编辑器/节点/场景/资源 |
| 编程基础 | Ch 4 | GDScript、变量与类型、输入、信号、运行时生命周期 |
| 两个完整项目 | Ch 5–6 | Flappy Bird：模块化+信号总线；Battle Tank：MVP+模块协作 |
| 架构与 AI | Ch 7–12 | 组件化、碰撞层、FSM、视野与避障、导航寻路、程序化地图 |
| 打磨与前沿 | Ch 13–16 | 光照/UI/关卡/发布、强化学习、大模型桌面宠物、OOAD/SOLID/风格 |

## 安装

### 方式一：skills CLI（发布后可用）

```bash
npx skills add MenthaPlay/basic-godot-skill
```

### 方式二：手动克隆到对应主机的技能目录

```bash
# Codex / Copilot / Amp 通用
git clone https://github.com/MenthaPlay/basic-godot-skill.git ~/.agents/skills/basic-godot-skill

# Claude Code
git clone https://github.com/MenthaPlay/basic-godot-skill.git ~/.claude/skills/basic-godot-skill

# Hermes Agent
git clone https://github.com/MenthaPlay/basic-godot-skill.git \
  "${HERMES_HOME:-$HOME/.hermes}/skills/productivity/basic-godot-skill"
```

也可以把技能文件夹放在项目的 `.agents/skills/` 下作为项目级技能使用。

## 使用

```text
# 加载核心框架
basic-godot-skill

# 按主题查询
basic-godot-skill 敌人巡逻与追击状态机怎么写？
basic-godot-skill 局部避障和 NavigationAgent2D 怎么配合？
basic-godot-skill 程序化地图生成流程是什么？

# 按章节深入
basic-godot-skill ch05
basic-godot-skill ch09
```

AI 会先看主题索引，再读取对应章节后回答；涉及具体版本 API 时，应再结合 Godot 官方文档核对。

## 来源与生成方式

- 原始资料：肖老师《Godot游戏编程入门教程》公开电子版（https://gdbook.kidsgame.top），共 16 章。
- 生成工具：[book-to-skill](https://github.com/virgiliojr94/book-to-skill)，按“结构优先”的方式提炼章节框架、代码范式与反模式。
- 所有章节均为二次创作摘要，不包含原书全文；若原书内容更新，应以原站为准。

## 版权与许可

- 本仓库的**整理结构、索引、原创注解与工具性文档**采用 MIT License，见 [LICENSE](LICENSE)。
- 章节摘要基于原书公开预览整理，**原始书稿、插图与相关内容的版权归原作者/权利人所有**。本仓库不声称拥有这些内容的版权，详见 [NOTICE.md](NOTICE.md)。
- 如果你是权利人并希望调整署名、限制内容或要求下架，请提 Issue 联系处理。

## 贡献

欢迎提交 Issue 或 PR 修正 API 错误、补充 Godot 版本差异、改进示例或增加新的实战模式。提交前请确保内容可验证，并注明对应的 Godot 版本。

## 免责声明

本项目为学习资料整理，与 Godot 基金会及原书作者/出版社无官方关联。Godot 是 Godot Engine contributors 的商标；相关名称与内容版权归各自权利人所有。
