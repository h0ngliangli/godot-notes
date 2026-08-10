# GDScript 函数规范总结（含 Lambda）

GDScript 中的函数是一等公民，可通过 `Callable` 传递。支持类型注解、默认参数、Lambda 等。

## 1. 普通函数定义

```gdscript
func function_name(param1: Type, param2: Type = default_value) -> ReturnType:
    # 函数体
    return value
```

- 使用 `func` 关键字定义。
- 参数之间用逗号分隔。
- 无显式 `self`（实例方法自动绑定）。
- 如果不指定返回类型，默认可返回任意值或 `null`。
- 指定返回类型后，所有代码路径必须返回匹配类型（`void` 除外）。
- 单行函数可写在同一行：`func square(x): return x * x`

### 示例
```gdscript
func add(a: int, b: int = 0) -> int:
    return a + b

func greet(name: String = "World") -> void:
    print("Hello, ", name)
```

## 2. 类型注解

- 参数类型：`param: int`
- 返回类型：`-> float` 或 `-> void`
- 可使用基础类型、自定义类、`Variant`、数组等。
- 有助于编辑器检查、自动补全和性能优化。

```gdscript
func process_data(data: Array[int]) -> Dictionary:
    return {"count": data.size()}
```

## 3. 默认参数与可变参数

- 默认参数必须放在参数列表末尾。
- Godot 4.5+ 支持可变参数（rest）：
```gdscript
func sum(a, b=0, ...args):
    var total = a + b
    for arg in args:
        total += arg
    return total
```

## 4. 静态函数

```gdscript
static func utility(a: int, b: int) -> int:
    return a + b
```

- 使用 `static func`。
- 不能访问实例变量或 `self`。
- 通过类名调用：`MyClass.utility(1, 2)`

还有静态构造函数：`static func _static_init():`

## 5. Lambda（匿名函数）

Lambda 创建独立的 `Callable`，不绑定到类实例。

### 基本语法
```gdscript
var lambda = func (x):
    print(x)

lambda.call(42)  # 必须用 .call() 调用
```

### 带类型和返回值
```gdscript
var multiply := func (x: int) -> int:
    return x * 2

print(multiply.call(5))  # 10
```

**注意**：Lambda 中返回值必须显式写 `return`（不能省略）。

### 命名 Lambda（便于调试）
```gdscript
var named = func my_lambda(x: int) -> void:
    print(x)
```
调试器中会显示名称，而非 `<anonymous lambda>`。

### 捕获局部变量
Lambda 会捕获创建时的局部环境（按值捕获）：
```gdscript
var x = 42
var lambda = func ():
    print(x)  # 打印 42

x = 100
lambda.call()  # 仍然打印 42
```

- 对 Array、Dictionary、Object 的内容修改会反映（引用语义）。
- 不能重新赋值捕获的局部变量（会产生警告）。

### 常见用法
```gdscript
# 信号连接
button.pressed.connect(func(): print("Clicked!"))

# 数组操作
var filtered = array.filter(func(item): return item > 10)
var mapped = array.map(func(x): return x * 2)
```

## 6. Callable 相关

函数引用会自动生成 `Callable`：
```gdscript
var callable = my_method          # 或 Callable(self, "my_method")
callable.call(arg1, arg2)
callable.callv([arg1, arg2])      # 用数组传参
```

### 常用方法
- `call(...)` / `callv(Array)`
- `bind(...)`：绑定额外参数
- `unbind(count)`：解绑参数
- `is_valid()` / `is_custom()` / `get_method()`
- `call_deferred(...)`：延迟调用

```gdscript
# 绑定示例
var bound = print.bind("Prefix:")
bound.call("Hello")  # 输出 Prefix: Hello

# 信号中绑定
signal_name.connect(my_func.bind(extra_arg))
```

## 7. 其他规范与注意事项

- **虚函数 / 抽象函数**（Godot 4.5+）：可用 `@abstract func name()` 声明抽象方法。
- **重写**：子类可重写父类方法，使用 `super()` 或 `super.method()` 调用父类。
- **构造函数**：`_init()`，可带参数，调用父类用 `super(...)`。
- **内置虚方法**：`_ready()`、`_process(delta)`、`_physics_process(delta)` 等。
- 函数名应使用 snake_case。
- 避免在迭代集合时修改集合结构。
- Lambda 不能声明为 `static`。

## 使用建议

- 日常方法：优先使用命名 `func` + 类型注解。
- 临时回调 / 高阶函数：使用 Lambda。
- 需要传递函数时：用 `Callable` + `bind`。
- 调试时给 Lambda 命名，便于堆栈查看。

## 参考
- [GDScript 基础 - 函数](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html)
- [Callable](https://docs.godotengine.org/en/stable/classes/class_callable.html)
- [静态类型](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/static_typing.html)
