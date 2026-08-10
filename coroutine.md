# Godot 协程（Coroutine）总结

本文总结 Godot 4 GDScript 中协程的概念、原理、常见陷阱与推荐实践。

---

## 1. 概念

协程（Coroutine）是一种**可以挂起（suspend）并在之后恢复（resume）**的执行流。

在 GDScript 中：

- 当函数内部出现 `await` 关键字时，该函数就变成了协程。
- 协程遇到 `await` 时会**暂停当前执行**并让出控制权：
  - 如果调用者是**普通调用**（没有使用 `await`），控制权会立刻返回给调用者，调用者可以继续执行后续代码。
  - 如果调用者使用了 `await func()`，则调用者自身也会保持挂起状态，控制权继续向上传递，最终通常交还给引擎。
- 等待的信号发出或操作完成后，协程从暂停的地方继续往下执行。

**与线程的关键区别：**

| 特性 | 协程 | 线程 |
|------|------|------|
| 执行模型 | 单线程、协作式 | 真正并行（受 GIL/系统调度影响） |
| 数据安全 | 无需锁（同一时刻只有一个在跑） | 需要同步机制 |
| 开销 | 极低 | 较高 |
| 适用场景 | 异步等待、时序逻辑、动画 | CPU 密集型计算 |

Godot 的协程运行在**主线程**上，属于协作式多任务，不会真正并行执行。

---

## 2. 基本语法与功能

### 2.1 最常见写法

```gdscript
func example() -> void:
	print("开始")
	await get_tree().create_timer(1.0).timeout   # 等待 1 秒
	print("1 秒后继续")
	await some_signal                            # 等待信号
	print("信号触发后继续")
```

### 2.2 可以 await 的对象

- 信号（Signal）：`await $Button.pressed`
- `SceneTreeTimer`：`await get_tree().create_timer(2.0).timeout`
- 其他协程函数：`await another_async_func()`
- 部分引擎对象返回的状态（如 Tween、HTTPRequest 等）

### 2.3 函数一旦包含 await，就成为协程

即使你不主动 `await` 调用它，它也会以协程方式运行。

---

## 3. 原理：引擎如何与协程交互

这是理解协程行为最关键的部分。

### 3.1 引擎对 `_ready` / `_process` 的调用方式

引擎主循环对节点回调是**普通调用**，**不会** `await`：

```gdscript
# 引擎内部伪代码（概念示意）
while true:
	# 等待下一帧
	_process(delta)   # 普通调用，没有 await
```

### 3.2 普通调用 vs await 调用的区别

| 调用方式 | 行为 | 结果 |
|---------|------|------|
| `func()`（普通调用） | 函数内部碰到 `await` 后**立刻返回** | 协程挂起在后台，调用者继续执行 |
| `await func()` | 调用者自己也挂起，**等待**整个协程结束 | 严格串行 |

因为引擎使用的是普通调用：

1. `_process` 或 `_ready` 执行到 `await` 时立刻返回控制权给引擎。
2. 引擎认为这次调用已经结束，继续推进主循环。
3. 下一帧（或后续时机）引擎可以再次调用该函数，从而启动**新的**协程实例。
4. 之前被挂起的协程仍然活着，等待条件满足后恢复执行。

这就是「协程满天飞」的根本原因。

### 3.3 `_ready` 中的 while true 循环为什么是安全的

```gdscript
func _ready() -> void:
	while true:
		await get_tree().create_timer(3.0).timeout
		# 做事情
```

执行过程：

1. 引擎调用一次 `_ready()`。
2. 碰到第一个 `await`，当前调用挂起并返回。
3. 引擎认为 `_ready` 已结束，节点进入 ready 状态，开始正常调用 `_process`。
4. 定时器到期后，**同一个协程**恢复执行，做事情，再次进入 `await`，再次挂起。
5. 如此循环。永远只有**一个**由 `_ready` 启动的协程在运行。

注意：后续的恢复**不是**引擎再次调用 `_ready()`，而是继续执行当初启动的那个协程。

---

## 4. 经典陷阱：在 `_process` 中直接 await

### 错误示例

```gdscript
func _process(delta: float) -> void:
	await get_tree().create_timer(3.0).timeout
	hp += 10
	printt("hp:", hp)
```

### 发生了什么

- 每一帧都会启动一个新的协程。
- 大约 3 秒后，之前启动的协程开始依次完成。
- 同时新的帧还在不断创建更多协程。
- 结果：第一次等待生效后，后续几乎每帧都在打印，造成「协程满天飞」。

**根本原因**：`await` 只挂起当前这一次调用，**不会阻止**引擎在下一帧再次调用 `_process`。

---

## 5. 推荐写法与实例

### 5.1 周期性任务：`_ready` + while true（简单直接）

```gdscript
extends Node2D

const MAX_HP = 100
var hp = 10

func _ready() -> void:
	print("hp: %d / %d" % [hp, MAX_HP])
	while true:
		await get_tree().create_timer(3.0).timeout
		hp += 10
		printt("hp:", hp, "max hp:", MAX_HP)
```

### 5.2 使用 delta 累计（不依赖 await）

```gdscript
var time_accum := 0.0

func _process(delta: float) -> void:
	time_accum += delta
	if time_accum >= 3.0:
		time_accum = 0.0
		hp += 10
		printt("hp:", hp)
```

### 5.3 使用 Timer 节点（更符合 Godot 习惯）

```gdscript
var regen_timer: Timer

func _ready() -> void:
	regen_timer = Timer.new()
	regen_timer.wait_time = 3.0
	regen_timer.timeout.connect(_on_regen)
	add_child(regen_timer)
	regen_timer.start()

func _on_regen() -> void:
	hp += 10
	printt("hp:", hp)
```

需要暂停时直接调用 `regen_timer.stop()` 即可。

### 5.4 使用 Tween（适合动画和一次性延迟）

```gdscript
var tween: Tween

func play_effect() -> void:
	if tween:
		tween.kill()          # 取消上一个
	tween = create_tween()
	tween.tween_property($Sprite, "modulate:a", 0.0, 1.0)
```

---

## 6. 如何管理多个并发协程

### 6.1 原则

**最好的管理是尽量不产生不必要的并发。**

### 6.2 布尔锁（防重入）—— 最常用

```gdscript
var is_busy := false

func do_action() -> void:
	if is_busy:
		return
	is_busy = true

	await get_tree().create_timer(2.0).timeout
	# 执行逻辑

	is_busy = false
```

### 6.3 支持中途取消（标志位检查）

协程没有强制取消 API，最可靠的方式是在每个 `await` 之后检查标志：

```gdscript
var should_continue := true

func long_task() -> void:
	await get_tree().create_timer(1.0).timeout
	if not should_continue:
		return

	await get_tree().create_timer(1.0).timeout
	if not should_continue:
		return

	# 继续后续逻辑

func cancel() -> void:
	should_continue = false
```

### 6.4 使用 GDScriptFunctionState（进阶）

```gdscript
var active_task: GDScriptFunctionState

func start_task() -> void:
	if active_task and active_task.is_valid():
		return          # 已有任务在跑
	active_task = do_work()   # do_work 是包含 await 的函数

func do_work() -> void:
	await get_tree().create_timer(3.0).timeout
	# ...
```

### 6.5 优先使用引擎自带对象

| 需求 | 推荐方案 | 优点 |
|------|----------|------|
| 周期性执行 | `Timer` 节点 | 可 start/stop/paused |
| 补间 / 延迟 | `Tween` | 可 kill 上一个 |
| 复杂时间线 | `AnimationPlayer` | 自带播放控制 |

---

## 7. 注意事项与常见坑

1. **节点被释放后恢复会报错**  
   如果节点在协程挂起期间被 `queue_free()`，下次恢复时可能出现错误。建议在关键 `await` 后检查：

   ```gdscript
   await get_tree().create_timer(1.0).timeout
   if not is_instance_valid(self) or not is_inside_tree():
       return
   ```

2. **不要在 `_process` / `_physics_process` 中无防护地使用 await**  
   会导致大量并发协程，造成性能与逻辑问题。

3. **协程仍然是主线程**  
   长时间的同步计算（即使放在协程里）仍然会卡住游戏。CPU 密集型任务应使用 `WorkerThreadPool`。

4. **信号连接与协程的生命周期**  
   使用 `await signal` 时，确保信号的发射者在等待期间仍然有效。

---

## 8. 总结

| 场景 | 推荐做法 |
|------|----------|
| 周期性逻辑（回血、冷却等） | `_ready` 中 `while true + await`，或 `Timer` 节点 |
| 防止重复触发 | 布尔锁 `is_busy` |
| 需要中途打断 | 标志位 + 每个 `await` 后检查 |
| 动画 / 一次性延迟 | 优先使用 `Tween`，需要时 `kill()` |
| 真正需要多任务并发 | 自己维护任务列表 + 取消标志 |

**核心记忆点：**

- 引擎对 `_process` / `_ready` 是**普通调用**，不会等待协程结束。
- `await` 只挂起当前协程实例，不会阻止引擎再次调用同一函数。
- 管理并发的最好方式是**从源头控制创建数量**，并用标志位或引擎对象（Timer/Tween）管理生命周期。

---

*基于实际调试与官方行为整理，适用于 Godot 4.x。*
