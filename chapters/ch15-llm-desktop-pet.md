# 第十五章：大模型打造 AI 桌面宠物

## Core Idea
本章把大语言模型接进 Godot：先梳理 LLM/多模态模型与提示词用法，再用 HTTPRequest 封装可复用的 LLMAPI 组件（系统提示、上下文、JSON 收发、信号返回），最后做出一个桌面常驻的小狐狸宠物——透明无边框窗口、多边形鼠标穿透、左键呼出聊天、右键拖动、Esc 退出，回答由大模型实时生成。

## Frameworks Introduced
- **LLMAPI 封装组件（extends HTTPRequest）**：集中处理密钥、对话历史、请求构造、响应解析与信号通知。
  - API: `call_llm(prompt)` → POST chat/completions；完成后 emit `request_finished(output)`。
  - Why: 任何场景/插件都能复用，UI 只连信号。
- **HTTPRequest 网络模型**：`request(url, headers, HTTPClient.METHOD_POST, body)`，完成后自动发 `request_completed`。
- **JSON.stringify / parse_string**：构造请求体、解析响应；Body = {model, messages, stream:false}。
- **对话上下文管理**：history 存 user/assistant 消息，每次请求带 system prompt + 最近 `history_count` 条，形成多轮记忆。
  - Anti: 无脑塞全部历史会超 token；用 slice(-N) 截取。
- **ConfigFile 密钥管理**：`res://config/api_config.cfg` 存 key，不进 git/源码；程序启动 load 后组 Authorization Bearer header。
- **System Prompt = 人设**：写清身份/性格/语气/限制（如“小狐狸桌面宠物，风趣幽默，≤200 字”），再生成对白。
- **编辑器插件（EditorPlugin）**：@tool 脚本 + plugin.cfg；_enter_tree 加控件（add_control_to_container）、_exit_tree 移除释放；可嵌入 LLMAPI 做成编辑器里的 AI 聊天助手。
- **桌面透明窗口配置**：Borderless + Transparent + Always On Top + Per-Pixel Transparency + Transparent Background；纹理 Nearest；透明 Clear Color。
- **鼠标穿透多边形（mouse_passthrough_polygon）**：窗口虽全屏透明，但操作系统仍会把整窗当可点；用 Polygon2D 指定“允许点击”的全局多边形，其余区域穿透到桌面。
  - Two polygons: Polygon2DPlayer（只框宠物）/ Polygon2DMenu（框宠物+聊天窗），随菜单开关切换并每物理帧更新。
- **窗口拖动（DisplayServer）**：右键按下记 `drag_offset = mouse - window_pos`，移动时 `window_set_position(mouse - offset)` 防抖动。
- **Tween 菜单动画 + visible_ratio 打字机**：Control scale 0↔1（TRANS_BACK），RichTextLabel visible_ratio 0→1 逐字显示。

## Key Concepts
- **LLM**：预测下一词的语言模型；大=参数多/数据广/通用任务/自注意力上下文。
- **多模态**：文本+图像+音频+视频联合理解生成（文生图/图问答/语音）。
- **Prompt 工作法**：先给大纲分章完善、设 persona、加长度/风格限制；输出统一 JSON 便于引擎接入。
- **HTTP Header**：Authorization: Bearer KEY + Content-Type: application/json。
- **request_finished 信号**：LLMAPI 向场景广播回答文本，解耦网络层与 UI。
- **DisplayServer**：引擎↔操作系统的显示/输入桥（窗口、鼠标、屏幕尺寸）。
- **get_window()**：当前 Window；宠物用全屏透明窗 + 穿透多边形。
- **Polygon2D.to_global**：局部顶点转屏幕/窗口全局坐标后再给 passthrough。
- **Sprite2D Hframes/Vframes + Frame**：精灵表切帧；AnimationPlayer 控制 frame 做 idle 动画（13→16）。
- **Area2D.input_event**：宠物身体可点击（Pickable=true）触发聊天开关。
- **Compatibility 渲染**：透明窗带黑底时可切换渲染器解决。
- **Input Map**: click=左键、drag=右键、quit=Esc。

## Mental Models
- Think of LLM as 一位懂游戏的超级助理：告诉它人设与任务，它替你写文案/代码/对白。
- Think of HTTPRequest as 打电话给模型：拨号（request）、等回音（request_completed）、拆信（JSON parse）。
- Think of system prompt as 给演员发剧本人设：不写人设，模型会“出戏”。
- Think of mouse_passthrough_polygon as 透明玻璃上的“可碰区”：只有划了区的地方接得住鼠标，其余直接穿到桌面。
- Think of drag_offset as 记住手抓在箱子的哪个位置：否则箱子会跳到你手上。
- Think of 桌面宠物 as 一个无边框透明小窗里的角色：不是传统游戏窗口。

## Anti-patterns
- **API Key 硬编码进脚本**：会随项目泄露；用 ConfigFile 且不提交。
- **无上下文记忆**：每次独立回答显得失忆；维护 history 并截断。
- **整段历史全发**：token 爆掉；history.slice(-N)。
- **网络请求不接信号**：不知道何时完成；连 request_completed → parse → emit。
- **直接 set window full clickable**：透明窗挡住桌面操作；必须 mouse_passthrough_polygon。
- **多边形用局部坐标不转全局**：窗口位置/宠物移动后区域错位；每帧 to_global 更新。
- **拖动让窗口跳到你鼠标位置**：没有 drag_offset；要记录并减去偏移。
- **界面/代码/密钥路径写死且不做异常**：请求失败 push_error、密钥读不到给提示。
- **用 Forward+ 渲染透明窗出现黑底**：可切 Compatibility 排查。

## Code Examples
LLMAPI 核心：
```gdscript
extends HTTPRequest
class_name LLMAPI

signal request_finished

var history: Array = []
var history_count: int = 3
var api_key: String = ""
var system_prompt: String = "你是一个精通Godot的助手，回答不超过200字"
var url = "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions"

func _ready() -> void:
    load_api_key()
    header = ["Authorization: Bearer " + api_key, "Content-Type: application/json"]
    request_completed.connect(on_request_completed)

func call_llm(prompt: String) -> void:
    history.append({"role": "user", "content": prompt})
    var messages = [{"role": "system", "content": system_prompt}]
    messages.append_array(history.slice(-history_count))
    var body = JSON.stringify({
        "model": "qwen-max",
        "messages": messages,
        "stream": false,
    })
    var result = request(url, header, HTTPClient.METHOD_POST, body)
    if result != OK:
        push_error("LLM 请求失败: %s" % result)

func on_request_completed(_result, _code, _headers, body) -> void:
    var response = JSON.parse_string(body.get_string_from_utf8())
    var output = response["choices"][0]["message"]["content"]
    history.append({"role": "assistant", "content": output})
    request_finished.emit(output)
```
- **What it demonstrates**: 完整“配置→组装→POST→解析→信号”闭环。

点击穿透更新：
```gdscript
func update_click_through() -> void:
    var pts: PackedVector2Array = current_polygon.polygon
    for i in range(pts.size()):
        pts[i] = current_polygon.to_global(pts[i])
    window.mouse_passthrough_polygon = pts

func _physics_process(_delta: float) -> void:
    update_click_through()
```
- **What it demonstrates**: 局部多边形 → 全局坐标 → 实时窗口穿透区。

拖动窗口：
```gdscript
func _input(event: InputEvent) -> void:
    if event.is_action_pressed("drag") and not dragging:
        dragging = true
        drag_offset = DisplayServer.mouse_get_position() - DisplayServer.window_get_position()
    elif event.is_action_released("drag") and dragging:
        dragging = false
    elif event is InputEventMouseMotion and dragging:
        DisplayServer.window_set_position(
            DisplayServer.mouse_get_position() - drag_offset)
```
- **What it demonstrates**: 右键拖窗的标准 offset 写法。

菜单与打字机：
```gdscript
func toggle_menu(show: bool) -> void:
    var tw = create_tween()
    if show:
        update_click_through()
        tw.tween_property(control, "scale", Vector2.ONE, 0.5) \
          .set_trans(Tween.TRANS_BACK).set_ease(Tween.EASE_IN_OUT)
        await tw.finished
        show_text("你好，我是小狐狸", 0.5)
    else:
        tw.tween_property(control, "scale", Vector2.ZERO, 0.5) \
          .set_trans(Tween.TRANS_BACK).set_ease(Tween.EASE_IN_OUT)
        await tw.finished
        current_polygon = polygons[0]
        update_click_through()

func show_text(text: String, time: float) -> void:
    rich_text_label.text = text
    var tw = create_tween()
    tw.tween_property(rich_text_label, "visible_ratio", 1.0, time).from(0)
```
- **What it demonstrates**: 缩放弹窗 + visible_ratio 打字效果 + 穿透区随状态切换。

## Reference Tables
| 窗口设置 | 值 |
|---|---|
| Borderless / Always On Top / Transparent | true |
| Per Pixel Transparency Allowed | true |
| Transparent Background | true |
| Default Texture Filter | Nearest |
| 初始窗口 | 1×1（启动后放大） |

| 输入 | 动作 | 用途 |
|---|---|---|
| 鼠标左键 | click | 切换聊天菜单 |
| 鼠标右键 | drag | 拖动宠物 |
| Esc | quit | 退出 |

| 组件 | 职责 |
|---|---|
| LLMAPI (HTTPRequest) | 密钥/历史/请求/解析/信号 |
| Polygon2D ×2 | 可点击区域（宠物/菜单） |
| Area2D | 宠物身体拾取 |
| AnimationPlayer | frame 序列 idle |
| RichTextLabel + LineEdit | 对话显示与输入 |

## Worked Example
运行流程：1×1 窗口启动 → `setup_window()` 放大到全屏并透明 → 小狐狸 idle 动画播放。左键点击宠物（Area2D input_event）→ show_menu 翻转 → 先切换 Polygon2DMenu 更新穿透区，再 Tween 弹开聊天窗并打字显示问候。输入问题回车 → `llmapi.call_llm`（带 system 人设 + 最近 3 轮历史）→ 输入框“让我想想”且不可编辑 → 模型返回 → `request_finished` → 恢复输入并打字显示回答。右键按住可拖动整个透明窗，鼠标穿透区每物理帧跟随，Esc 退出。

## Key Takeaways
1. LLM 接入 = HTTPRequest + JSON + Authorization Bearer + 信号返回，可封装成通用 LLMAPI 组件。
2. system prompt 是“人设”，history 截断保上下文又不爆 token；密钥走 ConfigFile。
3. 桌面宠物 = 透明无边框全屏窗 + mouse_passthrough_polygon 限定可点区。
4. 拖动窗口必须记录 drag_offset，否则鼠标一移窗口就跳。
5. Polygon 区域要转全局坐标并每帧刷新，才能跟随移动/菜单变化。
6. 编辑器插件 @tool/EditorPlugin 可复用 LLMAPI 做 AI 助手。
7. Prompt 先大纲后分章、控制长度与输出格式，生成内容才可落地。

## Connects To
- **Ch 8/14**: LLM 是 AI 的“语言表达”，与 RL“行动智慧”互补。
- **Ch 14**: 插件机制（addons/plugin.cfg/EditorPlugin）两章互通。
- **Ch 13**: 与主菜单/UI/音效集成可扩展成 NPC 或助手系统。
- **Ch 4**: HTTPRequest/信号/await 是 GDScript 网络异步基础。
