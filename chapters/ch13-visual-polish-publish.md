# 第十三章：视觉优化与发布

## Core Idea
本章把“可玩原型”打磨成“完整可发布游戏”：用 CanvasModulate/2D 光照/阴影/HDR+Glow 营造夜间氛围；搭建主菜单与 SoundManager Autoload 管理场景与声音；新增 LevelManager 统筹开场/胜利/失败/切关流程；让地形（道路快/草地慢）真正影响玩法；最后完成 Windows 导出发布。

## Frameworks Introduced
- **CanvasModulate 全画布调色**：给整个 2D 画布叠一层颜色滤镜，模拟昼夜/氛围。
  - Exception: CanvasLayer 内容独立渲染不受影响——HUD 天然保持明亮。
- **2D 光照体系**：PointLight2D（局部点光）、DirectionalLight2D（全场景平行光）、LightOccluder2D（遮挡物投阴影）。
- **特效防变暗两方法**：整组内容放 CanvasLayer（HUD）；单个精灵/粒子给 CanvasItemMaterial 设 Unshaded=true（爆炸、子弹、道具），不受光照与滤镜影响。
- **阴影两来源**：TileSet 的 Occlusion Layer 给石头/地形投影；LightOccluder2D 给动态角色投影。
- **HDR + Glow 发光**：Project Settings 开 HDR 2D → WorldEnvironment（Canvas 模式）开 Glow → 特效 modulate 调成 >1 的 RAW 颜色（如 1.5）触发光晕。
- **SoundManager Autoload**：BGM 与音效全局管理，跨场景切换不中断。
  - Why: 就近放 AudioStreamPlayer 会随场景卸载；单例常驻解决连续性。
- **主菜单流程**：场景用 preload 缓存，`get_tree().change_scene_to_packed()` 切换菜单/游戏。
- **LevelManager（关卡导演）**：控制 pause_level/start_level、开场打字动画、胜利/失败面板与切关；Process Mode = Always 保证暂停时仍能响应 UI。
  - Difficulty: `wave_number = current_level + 1`、`enemy_in_wave = current_level + 3`，敌人数量随关递增。
- **地形影响玩法（is_on_road）**：local_to_map + road_cells 判断坦克位置，道路 max_speed 250、草地 150。
- **关卡氛围循环色**：CanvasModulate 脚本按 `current_level` 从颜色数组取色，越界取模循环（清晨蓝/正午黄/傍晚橙/日落紫/深夜蓝）。
- **发布流程**：主场景设为菜单 → 安装 Export Templates → Project > Export 添加 Windows Desktop → 设 Export Path/Architecture/Icon/Embed PCK → Export Project → 压缩 zip 分发。

## Key Concepts
- **CanvasItemMaterial.Unshaded**：节点绕过全局光照/滤镜，直接原始颜色渲染。
- **HDR（>1 颜色）**：让亮区超阈值，为 Glow 提供输入。
- **WorldEnvironment（Canvas mode）**：2D 后期处理（Glow/Tonemap/Adjustments/Fog）总控。
- **change_scene_to_packed**：卸载当前场景并加载指定 PackedScene。
- **get_tree().paused / Process Mode=Always**：全局暂停；Always 节点暂停时仍运行。
- **RichTextLabel + BBCode + visible_ratio**：格式化文本并做打字机动画。
- **mouse_entered / pressed 信号**：菜单悬停与点击反馈。
- **AudioStreamPlayer finished 信号 + await**：播放完点击音再退出/切场景。
- **local_to_map 地形判定**：像素坐标 → 网格坐标，与 road_cells 求交集判断是否在道路。
- **Embed PCK**：资源打进 exe，单文件分发；Export with Debug 只用于测试。
- **Hotspot**：自定义鼠标图片的点击热点，64×64 准星设 (32,32) 居中。

## Mental Models
- Think of CanvasModulate as 整块屏幕上的彩色玻璃；CanvasLayer as 玻璃之外的独立画布。
- Think of Unshaded as “我自己负责这块画布”：聚光灯和滤镜都别管我。
- Think of Glow as “亮到溢出的光晕”：先开 HDR 让颜色超 1，再让后期去“发光”。
- Think of SoundManager as 永不退场的广播站：场景换台它继续放 BGM。
- Think of LevelManager as 关卡导演：不打仗，只管开场/暂停/喊开始/判胜败/切下一幕。
- Think of 地形速度 as 真实路况：走路 vs 公路，让玩家在选择路线上动脑。
- Think of 发布 as 装进礼物盒：导出 exe、写 README、压缩 zip，玩家无需 Godot 即可拆开玩。

## Anti-patterns
- **让特效跟着场景一起变暗**：爆炸/子弹/道具应加 Unshaded 材质或放 CanvasLayer，别被 CanvasModulate 拖暗。
- **BGM 放在场景节点**：切场景音乐即断；用 Autoload SoundManager。
- **场景切换每次运行时 load**：会卡顿；GameManager 里 preload 菜单与游戏场景。
- **暂停时 UI 也停**：LevelManager 忘记设 Process Mode = Always，开场/结算按钮点不动。
- **把胜负 UI 全塞 HUD**：职责混乱；结算归 LevelManager，HUD 只做实时状态。
- **忘记 HDR2D 就调 Glow**：颜色再亮也不发光；三件套缺一不可（HDR2D、WorldEnvironment Glow、超 1 颜色）。
- **地形只是贴图**：道路没有功能差异就浪费设计；用 is_on_road 调速。
- **直接导出不测试**：主场景、资源路径、运行报错必须先全流程验证。

## Code Examples
按关卡切换氛围色：
```gdscript
extends CanvasModulate

@export var map_colors: Array[Color]

func _ready() -> void:
    var index = Gamemanager.current_level - 1
    if index >= map_colors.size():
        index = Gamemanager.current_level % map_colors.size() - 1
    color = map_colors[index]
```
- **What it demonstrates**: 关卡编号驱动颜色，越界取模循环。

SoundManager：
```gdscript
const hover_sound = preload("res://assets/sound/hover.ogg")
const click_sound = preload("res://assets/sound/click.ogg")

func play_music() -> void:
    music.play()

func play_click() -> void:
    sound.stream = click_sound
    sound.play()

func play_hover() -> void:
    sound.stream = hover_sound
    sound.play()
```
- **What it demonstrates**: Autoload 单例集中放音，换场景 BGM 不断。

LevelManager 核心流程：
```gdscript
func pause_level() -> void:
    get_tree().paused = true

func intro_anim() -> void:
    control.show()
    continue_panel.show()
    animation_player.play("show")   # visible_ratio 0 → 1 打字机
    await animation_player.animation_finished
    continue_button.show()

func on_continue_pressed() -> void:
    Gamemanager.level_start.emit()
    continue_panel.hide()
    control.hide()
    get_tree().paused = false

func next_level() -> void:
    Gamemanager.next_level()
    Gamemanager.reset_score()
    get_tree().reload_current_scene()   # PCG 地图重生成
```
- **What it demonstrates**: 暂停/动画/信号/重载生成新关的完整导演流程。

地形速度：
```gdscript
func _physics_process(delta: float) -> void:
    if map.is_on_road(global_position):
        max_speed = 250
    else:
        max_speed = 150
    move(delta)
```
- **What it demonstrates**: 道路更快/草地更慢，让地图有策略价值。

## Reference Tables
| 视觉需求 | 方案 | 注意 |
|---|---|---|
| 夜间/氛围 | CanvasModulate | 不影响 CanvasLayer |
| 局部光 | PointLight2D | 移动端适量 |
| 平行光 | DirectionalLight2D | 全场景角度 |
| 地形阴影 | TileSet Occlusion Layer | 画遮挡形状 |
| 角色阴影 | LightOccluder2D | 动态跟随 |
| 特效发光 | HDR2D + Glow + >1 颜色 | 三件缺一不可 |
| 特效防变暗 | CanvasItemMaterial Unshaded | 单节点灵活 |

| 场景 | 用途 |
|---|---|
| Menu.tscn | 主菜单（Start/Quit） |
| game.tscn | 主游戏（每关重载生成） |
| SoundManager (Autoload) | BGM/音效全局 |
| LevelManager | 开场/胜负/切关导演 |

| 发布步骤 | 关键点 |
|---|---|
| 准备 | Main Scene=menu、保存、完整试玩 |
| 模板 | Editor → Manage Export Templates |
| 配置 | Export Path、x86_64、Embed PCK |
| 导出 | Export Project（测试勾 Debug） |
| 分发 | zip + README + 版本号 |

## Worked Example
完整关卡体验流：Menu 点 Start（悬停音/点击音）→ SoundManager 音量降为 -20 → GameManager.to_game() 切到 game.tscn。LevelManager._ready 暂停游戏并播放 “Level N / 几波几敌” 打字动画；玩家点 Continue → emit level_start（EnemyManager 按关数生成敌人）→ 取消暂停。玩家通关 → player_win → Timer 延迟显示 WinPanel → Next Level 调用 next_level() 重载场景并生成新的随机地图与更亮/暗氛围色；失败则 LossPanel → Back Menu。地形上道路 250px/s、草地 150px/s，爆炸带 HDR Glow。

## Key Takeaways
1. 氛围 = CanvasModulate + 2D 光照 + Occluder 阴影 + HDR/Glow；特效要 Unshaded 防变暗。
2. 跨场景声音交给 SoundManager Autoload，BGM 永不中断。
3. 流程交给 LevelManager（Process Mode Always），HUD 只做实时状态。
4. 关卡难度 = current_level 驱动波次与敌人数；重载当前场景即可生成新 PCG 关卡。
5. 让地形影响速度，地图才不只是装饰。
6. 发布前主场景设为菜单、全流程试玩；Embed PCK 单文件分发。
7. 版本号 + README + zip 是交给玩家的基本礼盒。

## Connects To
- **Ch 5-7**: 打磨 Flappy Bird/Battle Tank 的视觉与流程。
- **Ch 8**: AI 导演/DDA 理念在关卡节奏上的实践。
- **Ch 12**: PCG 地图 + 重载场景 = 无限关卡。
- **Ch 14/15**: AI 玩法章节将为作品加入现代技术体验。
