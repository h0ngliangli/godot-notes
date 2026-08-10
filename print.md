# GDScript 打印相关函数总结

Godot 的 GDScript 提供了多个用于调试和日志输出的全局函数（主要位于 `@GlobalScope`，部分在 `@GDScript`）。

这些函数大多支持任意数量的参数（vararg），参数可以是任意类型，会自动转换为字符串。

## 基本打印函数

### `print(...)`
将所有参数转换为字符串后**直接拼接**（无任何分隔符），并在末尾添加换行符。  
输出到编辑器的 **Output** 面板和标准输出（stdout）。

```gdscript
print("Hello", "World", 123)   # 输出: HelloWorld123
print("a", "b", [1, 2, 3])     # 输出: ab[1, 2, 3]
```

### `prints(...)`
参数之间用**空格**分隔，末尾添加换行。最常用的多参数打印方式。

```gdscript
prints("A", "B", "C")  # 输出: A B C
```

### `printt(...)`
参数之间用**制表符（Tab）**分隔，末尾添加换行。适合简单对齐/表格输出。

```gdscript
printt("Name", "HP", "MP")
printt("Hero", 100, 50)
```

### `printraw(...)`
类似 `print`，但**不添加末尾换行符**。  
**重要**：只输出到操作系统终端（stdout），**不会**显示在编辑器的 Output 面板中。

适合需要连续输出（如进度条）的场景。

```gdscript
printraw("Loading")
printraw(".")
printraw(".")
# 终端显示: Loading..
```

## 格式化与条件打印

### `print_rich(...)`
类似 `print`，但支持 **BBCode** 格式化。  
在编辑器 Output 中完整支持；在终端中会转换为 ANSI 转义码（支持有限，颜色依赖终端主题）。

常用支持的标签：`b`、`i`、`u`、`s`、`color`、`bgcolor`、`fgcolor`、`code`、`url`、`center`、`right`、`indent` 等。

颜色支持命名颜色（如 `red`、`green`、`blue`、`yellow`、`orange`、`purple`、`cyan`、`white`、`black`、`gray` 等），不支持十六进制颜色值。

```gdscript
print_rich("[color=green][b]成功！[/b][/color]")
print_rich("[color=red]错误信息[/color]")
```

### `print_verbose(...)`
与 `print` 相同，但**仅在启用 verbose 模式时输出**。  
启用方式：项目设置 → Debug → Settings → stdout → Verbose stdout，或命令行参数 `--verbose`。

适合详细调试日志，默认不干扰正常运行输出。

## 错误与警告

### `printerr(...)`
类似 `print`，但输出到**标准错误流（stderr）**。  
大多数情况下更推荐使用 `push_error()`。

### `push_error(...)` / `push_warning(...)`
专门用于错误和警告消息：
- 显示在 **Debugger > Errors** 标签页
- 通常附带堆栈信息
- 不会暂停项目执行

推荐用于正式的错误/警告报告，而不是普通 `print`。

```gdscript
push_error("无法加载资源: ", path)
push_warning("此方法已弃用")
```

## 调试专用（@GDScript）

### `print_debug(...)`
类似 `print`，但会在输出后额外打印当前堆栈帧信息。  
仅在从编辑器运行或 debug 导出模式下有效。

### `print_stack()`
打印从当前位置开始的完整调用堆栈。  
同样仅在调试环境下有效。

## 其他相关

- `Node.print_tree()` / `Node.print_tree_pretty()`：打印当前节点及其子节点的场景树结构（pretty 版本使用 Unicode 树形字符，更美观）。

## 字符串格式化输出（与 print 结合使用）

GDScript 提供了强大的字符串格式化能力，常与 `print` / `prints` 等一起使用，实现更清晰、可控的输出。

主要有三种方式：

### 1. `%` 运算符（printf 风格，推荐用于数值格式化）

使用 `%` 运算符 + 格式说明符。多个值时必须传入**数组**。

```gdscript
# 基本用法
print("玩家: %s, 分数: %d" % ["Alice", 100])
# 输出: 玩家: Alice, 分数: 100

print("血量: %.1f%%" % 85.5)
# 输出: 血量: 85.5%   （%% 表示字面 %）
```

#### 常用格式说明符

| 说明符 | 含义 | 示例 |
|--------|------|------|
| `%s` | 任意类型（调用 `str()`） | `%s` % any |
| `%d` | 十进制整数（浮点会向下取整） | `%d` % 3.9 → 3 |
| `%f` | 浮点数 | `%f` % 3.14159 |
| `%x` / `%X` | 十六进制（小写/大写） | `%x` % 255 → ff |
| `%o` | 八进制 | `%o` % 8 → 10 |
| `%c` | 单个 Unicode 字符 | `%c` % 65 → A |
| `%v` | 向量（Vector2/3 等） | `%v` % Vector2(1, 2) |

#### 常用修饰符

- **宽度**：`%10d` → 右对齐，宽度至少 10
- **补零**：`%05d` → `00042`
- **精度**：`%.2f` → 保留 2 位小数
- **左对齐**：`%-10s`
- **正号**：`%+d` → 正数也显示 +
- **动态宽度/精度**：`%*.*f` % [宽度, 精度, 值]

```gdscript
print("%10.2f" % 3.14159)     # "      3.14"
print("%05d" % 42)            # "00042"
print("%-8s|%d" % ["Name", 1]) # "Name    |1"
print("%0*d" % [4, 7])         # "0007"  （动态宽度）
```

**注意**：格式字符串中的占位符数量必须与传入数组的元素数量严格匹配。

### 2. `String.format()` 方法（命名/索引占位符）

更适合使用**键名**或索引进行替换，可读性更好。

```gdscript
# 使用字典（推荐）
print("玩家: {name}, 分数: {score}".format({"name": "Alice", "score": 100}))

# 使用数组 + 索引
print("玩家: {0}, 分数: {1}".format(["Alice", 100]))

# 混合
print("玩家: {0}, 分数: {score}".format({"0": "Alice", "score": 100}))
```

`format()` 本身**不支持**数值的宽度/精度控制，需要嵌套 `%` 运算符：

```gdscript
var text = "分数: {score}".format({"score": "%.2f" % 98.765})
print(text)  # 分数: 98.77
```

### 3. 简单字符串拼接

```gdscript
print("玩家: " + name + ", 分数: " + str(score))
```

简单但可读性较差，且无法方便地控制数值格式。

### 使用建议

- **需要精确控制数字显示**（小数位数、补零、对齐）→ 优先用 `%` 运算符。
- **模板中有多个命名字段** → 用 `String.format()`。
- 日常快速调试 → `prints()` 或简单 `print()` 即可。
- 复杂输出可组合使用：先用 `%` 或 `format()` 生成字符串，再交给 `print` / `print_rich` 等。

## 使用建议

- 日常调试优先使用 `print` / `prints` / `printt`。
- 真正的错误用 `push_error`，警告用 `push_warning`。
- 需要颜色或样式时用 `print_rich`。
- 发布版本注意清理或使用 `print_verbose` / 条件判断，避免过多输出影响性能。
- 更复杂的格式化可结合 GDScript 字符串格式化（`%` 运算符或 `String.format()`）使用。

## 参考

- [官方 Output panel 文档](https://docs.godotengine.org/en/stable/tutorials/scripting/debug/output_panel.html)
- [GDScript 格式字符串官方教程](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_format_string.html)
- [@GlobalScope](https://docs.godotengine.org/en/stable/classes/class_@globalscope.html)
- [@GDScript](https://docs.godotengine.org/en/stable/classes/class_@gdscript.html)
