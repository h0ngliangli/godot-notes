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

## 使用建议

- 日常调试优先使用 `print` / `prints` / `printt`。
- 真正的错误用 `push_error`，警告用 `push_warning`。
- 需要颜色或样式时用 `print_rich`。
- 发布版本注意清理或使用 `print_verbose` / 条件判断，避免过多输出影响性能。
- 更复杂的格式化可结合 GDScript 字符串格式化（`%` 运算符或 `String.format()`）使用。

## 参考

- [官方 Output panel 文档](https://docs.godotengine.org/en/stable/tutorials/scripting/debug/output_panel.html)
- [@GlobalScope](https://docs.godotengine.org/en/stable/classes/class_@globalscope.html)
- [@GDScript](https://docs.godotengine.org/en/stable/classes/class_@gdscript.html)
