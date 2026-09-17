# 第十二章：程序化地图生成

## Core Idea
本章用代码替代人工绘制，把《Battle Tank》地图改造成“每次运行都是新关卡”：先铺草地、圈安全区、画边界，再用 PathGenerator + AStarGrid2D 自动生成连接敌人与玩家出生点的道路，最后在空闲网格上聚簇布置障碍/装饰，并自动产出巡逻点与炮塔位，让敌人、玩家与 Manager 全部从地图对象取数据。

## Frameworks Introduced
- **程序化地图生成（PCG）流程**：初始化关键坐标 → 填充草地 → 出生点安全区 → 边界 → 随机道路 → 筛空闲格 → 布置障碍/装饰 → 生成巡逻点 → 生成炮塔点。
  - Why: 提升重玩性、节省关卡成本、支持动态世界；挑战是可控性、可调试性与规则设计。
- **随机点 + 寻路连接法**：生成随机途经点 → 最近邻排序 → 加“L 形拐点”保证纵横路 → AStarGrid2D 逐段连线 → set_cells_terrain_connect 自动地形绘制。
- **安全区（Safe Area）**：以敌我出生点为中心外扩两层，禁止任何障碍/炮塔占用，防止出生即卡死或被堵门。
- **空闲网格筛选（free_cell）**：草地全部格子 − 道路 − 敌我安全区 = 合法放置池；每个占用后从池中 erase，避免重叠。
- **聚簇填充（set_item）**：随机中心格 + 以 prob 概率向 8 邻格扩散，形成自然的石头群/树丛；`cannot_nav` 决定是否同步铺不可导航草地。
- **守卫出生公平性**：炮塔候选点必须“紧邻道路”且距玩家出生点 > 400px，避免开局即被压制。
- **逻辑对象 new() vs 场景对象 instantiate()**：纯计算（PathGenerator）继承 RefCounted 用 new()；要渲染/参与生命周期的游戏实体用 .tscn + instantiate() 进场景树。
- **类职责封装**：路径随机/排序/拐点逻辑抽到独立 PathGenerator 类，主地图脚本只编排步骤。

## Key Concepts
- **TileMapLayer 编程 API**：`set_cell`、`clear`、`get_used_cells(_by_id)`、`get_surrounding_cells`、`map_to_local`、`local_to_map`、`set_cells_terrain_connect`。
- **source_id / atlas 坐标**：set_cell 三参数 = 网格坐标 + 图块资源编号 + 图集内坐标（如草地 (0,0)，不可导航草地 (0,1)）。
- **AStarGrid2D**：为网格定制的 A*；设置 region/cell_size/offset、`diagonal_mode = DIAGONAL_MODE_NEVER`（只走横竖），`get_id_path(from, to)` 返回路径格数组。
- **最近邻 + L 形拐点**：先排随机点顺序，再给每段加 (x1,y2)/(x2,y1) 之一作为拐点，把斜线道路变横竖路。
- **distance_squared_to**：比 distance 快（免开方），排序足够用。
- **不可导航草地 + 树叠层**：边界树先铺非导航草地垫底，避免透明树透出下层可通行视觉。
- **map_to_local 出生坐标**：巡逻点/炮塔点返回像素坐标供单位放置；local_to_map 把场景 Marker2D 转成网格坐标。
- **按随机比例混合敌人**：`randf < enemy_tank_ratio` 选坦克或导弹车，并统一设置 speed/map/出生点。
- **max_try 防死循环**：炮塔候选尝试上限 100 次，找不到合法空位就放弃该点。

## Mental Models
- Think of 程序化生成 as 炼金：随机种子是火星，规则炉子是设计约束，产出是“每次不同但都能玩”的关卡。
- Think of 地图图层 as 分层施工图：先草地打底，再道路连通，边界封闭，障碍/装饰最后摆。
- Think of PathGenerator as 只算不演的工具人：RefCounted 逻辑类，不进场景树。
- Think of free_cell as 一张“已售座位表”：每放一样东西就划掉一格，绝不让两件东西挤同一格。
- Think of 安全区 as 出生点的“免打扰半径”：炮塔/石头都不能挡在门口。
- Think of AStarGrid2D 禁斜线 as 只能横竖走的城市街道：适合坦克地图，不会抄斜线近路穿墙。

## Anti-patterns
- **无约束纯随机**：地图杂乱、道路不连通、出生被堵；坚持固定流程 + 合法性筛选。
- **把复杂生成逻辑全堆主脚本**：应抽 PathGenerator 等独立类，职责清晰、可复用可测试。
- **该用 new() 却继承 Node**：纯计算对象进场景树白白消耗生命周期；用 RefCounted + new()。
- **set_cell 参数搞混**：网格坐标 ≠ 像素坐标，source_id ≠ atlas 坐标；先用 local_to_map/map_to_local 转换。
- **障碍直接压路/压出生点**：先算 free_cell，摆完即 erase。
- **巡逻点写死在编辑器**：地图每次不同，巡逻点必须从 road 图层动态抽。
- **炮塔紧贴玩家出生点**：开局被压制体验极差；强制距离检查。
- **随机抽点无重复控制**：多个单位挤同一个点；抽完 erase + max_try。

## Code Examples
铺草地 + 安全区：
```gdscript
func setup_grass() -> void:
    grass.clear()
    for x in range(map_size.x):
        for y in range(map_size.y):
            grass.set_cell(Vector2i(x, y), grass_id, nav_grass_atlas)

func setup_safe_area() -> void:
    enemy_safe_area.append(enemy_start)
    enemy_safe_area.append_array(grass.get_surrounding_cells(enemy_start))
    for cell in grass.get_surrounding_cells(enemy_start):
        enemy_safe_area.append_array(grass.get_surrounding_cells(cell))
    # Player 同理
```
- **What it demonstrates**: 双层循环铺底 + 出生点外扩两圈锁定禁布置区。

PathGenerator 随机点排序（最近邻 + L 拐点）：
```gdscript
func generate_random_points() -> Array[Vector2i]:
    var nearest_start = _start_pos
    for i in range(_points_num):
        var nearest_point = find_nearest_point(nearest_start, _random_points)
        var mid_points = [
            Vector2i(nearest_start.x, nearest_point.y),
            Vector2i(nearest_point.x, nearest_start.y),
        ]
        _path_points.append(mid_points.pick_random())
        _path_points.append(nearest_point)
        nearest_start = nearest_point
        _random_points.erase(nearest_point)
    return _path_points
```
- **What it demonstrates**: 贪心最近邻串点，插入 L 形拐点保证横竖路。

AStar 连线并自动地形画路：
```gdscript
func get_random_path() -> void:
    var path_points: Array[Vector2i] = path_generator.generate_random_points()
    var path_start = enemy_start
    for point in path_points:
        path_cell.append_array(astargrid.get_id_path(path_start, point))
        path_start = point
    path_cell.append_array(astargrid.get_id_path(path_start, player_start))
    road.set_cells_terrain_connect(path_cell, 0, 0)
```
- **What it demonstrates**: 分段寻路拼接 + 自动地形连接绘制。

通用聚簇填充：
```gdscript
func set_item(tile: TileMapLayer, prob: float, item_id: int,
              item_atlases: Array, cannot_nav: bool) -> void:
    var cell = free_cell.pick_random()
    if cannot_nav:
        grass.set_cell(cell, grass_id, nonav_grass_atlas)
    tile.set_cell(cell, item_id, item_atlases.pick_random())
    free_cell.erase(cell)
    for near_cell in grass.get_surrounding_cells(cell):
        if randf_range(0, 1) < prob and near_cell in free_cell:
            if cannot_nav:
                grass.set_cell(near_cell, grass_id, nonav_grass_atlas)
            tile.set_cell(near_cell, item_id, item_atlases.pick_random())
            free_cell.erase(near_cell)
```
- **What it demonstrates**: 一个函数同时处理石头（挡路）与树（装饰），簇状扩散且不重叠。

## Reference Tables
| 方法 | 作用 |
|---|---|
| set_cell | 网格坐标画指定 source_id + atlas 图块 |
| clear | 清空图层 |
| get_used_cells(_by_id) | 取已绘制格/特定图块格 |
| get_surrounding_cells | 取 8 邻格 |
| local_to_map / map_to_local | 像素坐标 ↔ 网格坐标 |
| set_cells_terrain_connect | 自动地形批量连图块 |

| PCG 方法 | 应用 |
|---|---|
| 随机点+路径 | 塔防/坦克道路（本章） |
| 细胞自动机 | 洞穴/地牢 |
| BSP 分割 | Roguelike 房间 |
| Perlin 噪声 | 开放世界地形 |
| WFC | 像素/拼贴约束生成 |

| 步骤 | 产出 | 关键约束 |
|---|---|---|
| 草地 | 可通行底图 | 全图填充 |
| 安全区 | 出生免打扰区 | 外扩两层 |
| 边界 | 封闭战场 | 树 + 非导航草地 |
| 道路 | 敌我连通路线 | 随机点 + A* |
| 障碍/装饰 | 石头群/树丛 | free_cell 防重叠 |
| 巡逻点 | 敌人航点 | 从 road 格随机抽 |
| 炮塔点 | 防守位 | 道路旁、距玩家 >400 |

## Worked Example
一次完整生成：脚本读 Points 下的 EnemyStart/PlayerStart/TopLeft/BottomRight 并 local_to_map → 45×22 草地铺满 → 以敌我出生点各外扩两圈加入安全区 → 地图外圈铺树边界 → PathGenerator 在 TopLeft..BottomRight 生成 6 个随机点，最近邻排序后每段加 L 拐点 → AStarGrid2D（禁斜线）把 enemy_start→点1→…→player_start 连成路径，road.set_cells_terrain_connect 画出道路 → 从草地格剔除道路与安全区得到 free_cell → set_item 放 8 块石头（转非导航草地）、5 棵树 → 从 road 格随机抽 5 个巡逻点 map_to_local → 在路边 free_cell 放炮塔且距玩家 >400。EnemyManager 从 map 取炮塔点/出生点并 spawn，敌人巡逻状态从 map.get_patrol_points() 取点。

## Key Takeaways
1. PCG = 随机 + 约束：固定流程 + 合法性筛选是质量底线。
2. 图块三要素别搞混：网格坐标、source_id、atlas 坐标。
3. 出生点先锁安全区，障碍/炮塔绝不压路不堵门。
4. AStarGrid2D + DIAGONAL_MODE_NEVER + 最近邻随机点，是坦克地图道路生成的轻量组合。
5. free_cell 边放边删，聚簇扩散用 get_surrounding_cells + 概率。
6. 纯逻辑用 RefCounted + new()，游戏实体才用 instantiate() 进场景树。
7. 动态地图要求玩家/敌人/Manager 都从地图对象取坐标，不写死。

## Connects To
- **Ch 8**: PCG 概念与 Godot TileMap 编程落地。
- **Ch 11**: 道路自动生成后需保证导航层仍正确（非导航草地/石头阻挡）。
- **Ch 13**: 生成地图同样需要关卡管理、UI 与发布衔接。
- **Ch 16**: PathGenerator 单职责类是 SRP 与对象设计的范例。
