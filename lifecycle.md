# Godot 节点常用虚方法与生命周期总结

Godot 中几乎所有游戏逻辑都建立在 **Node** 的虚方法（virtual methods）之上。引擎会在特定时机自动调用这些方法，脚本只需重写（override）它们即可。

本文总结最常用的几个回调，覆盖生命周期、每帧处理、绘制和输入。

> 适用版本：Godot 4.x

## 1. 生命周期回调

节点从创建到销毁的完整顺序大致如下：

1. `_init()`（Object）
2. `_enter_tree()`
3. （所有子节点完成 `_enter_tree` 与 `_ready`）
4. `_ready()`
5. 运行中反复调用 `_process` / `_physics_process` / `_draw` / 输入回调
6. `_exit_tree()`

### `_init()`

```gdscript
func _init() -> void:
    # 脚本实例化时立即调用（比 _enter_tree 更早）
    # 此时节点尚未进入场景树，不能安全使用 get_node() 或访问其他节点
    pass
```

- 相当于构造函数。
- 适合初始化与场景树无关的成员变量。
- 子节点此时可能还不存在。

### `_enter_tree()`

```gdscript
func _enter_tree() -> void:
    # 节点进入场景树时调用（可能多次：移除后再添加会再次触发）
    pass
```

- 调用顺序：**自上而下**（父节点先于子节点）。
- 此时子节点可能尚未完全就绪。
- 适合做与树结构相关的早期设置（如添加信号连接）。

### `_ready()`

```gdscript
func _ready() -> void:
    # 节点及其所有子节点都已进入场景树后调用
    # 最常用的初始化位置！
    pass
```

- **只调用一次**（默认情况下）。移除再添加不会再次调用，除非使用 `request_ready()`。
- 调用顺序：**自下而上**（子节点先于父节点）。这保证了父节点可以安全访问已就绪的子节点。
- 推荐在这里做：
  - `get_node()` 查找子节点
  - 连接信号
  - 初始化游戏状态
  - 启动 Timer、Tween 等

`@onready` 变量也是在 `_ready` 之前自动赋值的。

### `_exit_tree()`

```gdscript
func _exit_tree() -> void:
    # 节点即将离开场景树时调用
    # 子节点的 _exit_tree 已经执行完毕
    pass
```

- 适合清理资源、断开信号、停止处理等。

## 2. 每帧处理回调

### `_process(delta)`

```gdscript
func _process(delta: float) -> void:
    # 每渲染帧调用一次（帧率依赖）
    # delta：距离上一帧的秒数
    position += velocity * delta
```

- 调用时机：每帧渲染前。
- `delta` 随帧率变化（低帧率时更大）。
- **始终用 `delta` 做时间相关计算**，保证速度与帧率无关。
- 启用/关闭：`set_process(true/false)` 或在 `_ready` 中默认开启（定义了方法就会自动启用）。

适合：动画、UI 更新、非物理移动、相机跟随等。

### `_physics_process(delta)`

```gdscript
func _physics_process(delta: float) -> void:
    # 固定物理帧率调用（默认 60 次/秒）
    # 与渲染帧率解耦
    velocity = move_and_slide()
```

- 调用时机：每次物理步进前。
- `delta` 基本固定（`1.0 / PhysicsServer.ticks_per_second`）。
- 启用/关闭：`set_physics_process(true/false)`。

适合：角色移动、碰撞检测、刚体控制、任何需要与物理引擎同步的逻辑。

**对比建议**：

| 场景 | 推荐使用 |
|------|----------|
| 物理移动 / 碰撞 | `_physics_process` |
| 纯视觉动画 / UI | `_process` |
| 需要稳定时间步的逻辑 | `_physics_process` |
| 依赖实际显示帧率的效果 | `_process` |

## 3. 绘制回调（CanvasItem）

`_draw()` 属于 **CanvasItem**（Node2D、Control、Sprite2D 等都继承自它）。

```gdscript
func _draw() -> void:
    # 只能在这里调用 draw_* 系列函数
    draw_circle(Vector2.ZERO, 50, Color.RED)
    draw_line(Vector2(-20, 0), Vector2(20, 0), Color.WHITE, 2.0)
```

- 引擎需要重绘时自动调用。
- **不要**在 `_process` 里直接画东西，而是调用 `queue_redraw()` 请求重绘。

```gdscript
func _process(delta: float) -> void:
    # 某些状态变化后
    queue_redraw()  # 请求下一帧调用 _draw
```

常用绘制函数：`draw_line`、`draw_rect`、`draw_circle`、`draw_polygon`、`draw_texture`、`draw_string` 等。

注意：只有节点可见且在场景树中时才会真正绘制。

## 4. 输入回调

### `_input(event)`

```gdscript
func _input(event: InputEvent) -> void:
    if event is InputEventKey and event.pressed:
        if event.keycode == KEY_ESCAPE:
            get_tree().quit()
            # 可选择消耗事件，阻止后续处理
```

- 所有输入事件最先到达这里（在 UI 和 `_unhandled_input` 之前）。
- 适合全局快捷键、调试热键等。

### `_unhandled_input(event)`

```gdscript
func _unhandled_input(event: InputEvent) -> void:
    # 只有未被 _input 或 GUI 消耗的事件才会到这里
    if event is InputEventMouseButton and event.pressed:
        # 处理游戏内点击
        pass
```

- 最常用的游戏逻辑输入入口（移动、攻击、交互等）。
- 当事件被 `get_viewport().set_input_as_handled()` 标记后，后续节点不会再收到。

### `_gui_input(event)`（Control 专用）

用于 UI 控件自身的交互（按钮点击、拖拽等）。

## 5. 调用顺序小结

**进入场景树时：**

```
父._enter_tree()
  子._enter_tree()
    孙._enter_tree()
    孙._ready()
  子._ready()
父._ready()
```

**离开场景树时：** 与上面相反，子先 `_exit_tree`。

**每帧：**
1. 物理步进 → 所有开启的 `_physics_process`
2. 空闲处理 → 所有开启的 `_process`
3. 输入处理
4. 绘制 → `_draw`

## 6. 使用建议与注意事项

- **初始化优先用 `_ready()`**，而不是 `_enter_tree()` 或 `_init()`（除非有特殊需求）。
- **永远用 `delta` 乘以速度/时间**，避免帧率变化导致行为不一致。
- 不需要每帧执行的逻辑，用 `Timer`、`Tween` 或信号，而不是在 `_process` 里写 `if` 计时。
- `_draw` 里只做绘制，不要做复杂计算（计算放在 `_process`，然后 `queue_redraw`）。
- 可以在运行时动态开关处理：`set_process(false)`、`set_physics_process(false)`。
- 对于只想在编辑器预览的逻辑，使用 `@tool` + 判断 `Engine.is_editor_hint()`。
- 所有虚方法都是可选的，只重写你需要的即可。

## 参考

- [Overridable functions](https://docs.godotengine.org/en/stable/tutorials/scripting/overridable_functions.html)
- [Idle and Physics Processing](https://docs.godotengine.org/en/stable/tutorials/scripting/idle_and_physics_processing.html)
- [Node 类文档](https://docs.godotengine.org/en/stable/classes/class_node.html)
- [CanvasItem 类文档](https://docs.godotengine.org/en/stable/classes/class_canvasitem.html)
- [Godot notifications](https://docs.godotengine.org/en/stable/tutorials/best_practices/godot_notifications.html)
