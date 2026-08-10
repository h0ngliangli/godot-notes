# GDScript 集合类型总结

GDScript 提供了几种主要的集合（Collections）类型，用于存储和管理数据。它们都是**按引用传递**的（修改会影响所有引用），需要独立副本时请使用 `duplicate()`。

## 1. Array（数组）

最常用的动态数组，可存储任意类型的元素（Variant）。

### 基本用法

```gdscript
var arr = []                    # 空数组
var arr2 = [1, "hello", true]   # 混合类型
var arr3: Array[int] = [1, 2, 3] # 类型化数组（Typed Array，Godot 4+）

print(arr2[0])     # 1
print(arr2[-1])    # true（负索引从末尾计数）
arr2.append(4.5)
arr2.push_back(5)  # 与 append 相同
arr2.insert(1, "world") # 在索引 1 插入
print(arr2)        # [1, "world", "hello", true, 4.5, 5]
print(arr2.size()) # 长度
```

### 常用方法

| 方法                                          | 说明                       |
| --------------------------------------------- | -------------------------- |
| `append(value)` / `push_back(value)`          | 末尾添加元素               |
| `push_front(value)`                           | 开头添加                   |
| `pop_back()` / `pop_front()`                  | 移除并返回末尾/开头元素    |
| `insert(index, value)`                        | 在指定位置插入             |
| `remove_at(index)`                            | 按索引删除                 |
| `erase(value)`                                | 删除第一个匹配值           |
| `clear()`                                     | 清空                       |
| `has(value)` / `find(value)`                  | 是否包含 / 查找索引        |
| `sort()` / `sort_custom(func)`                | 排序                       |
| `map(func)` / `filter(func)` / `reduce(func)` | 函数式操作                 |
| `slice(begin, end)`                           | 切片                       |
| `duplicate(deep=false)`                       | 复制（深/浅）              |
| `+` 运算符                                    | 拼接两个数组（返回新数组） |

**Typed Array** 示例：

```gdscript
var numbers: Array[int] = [10, 20, 30]
# numbers.append("str")  # 运行时报错
for n: int in numbers:
    print(n)
```

Typed Array 比未类型化 Array 迭代/修改更快，但不如 Packed\*Array 高效。

## 2. Dictionary（字典）

键值对集合，**保留插入顺序**。键可以是几乎任何类型（只要可哈希），值可以是任意类型。

### 基本用法

```gdscript
var dict = {}                              # 空字典
var points = {"White": 50, "Yellow": 75}   # 字面量
var typed: Dictionary[String, int] = {"a": 1, "b": 2}  # 类型化（Godot 4.4+）

print(points["White"])     # 50
print(points.White)        # 50（字符串键可用点语法）
points["Blue"] = 150       # 添加/修改
points.erase("Yellow")     # 删除
print(points.has("Blue"))  # true
print(points.get("Red", 0)) # 不存在时返回默认值
```

### 常用方法

| 方法                            | 说明                   |
| ------------------------------- | ---------------------- |
| `has(key)`                      | 是否包含键             |
| `get(key, default=null)`        | 获取值，不存在返回默认 |
| `erase(key)`                    | 删除键值对             |
| `keys()` / `values()`           | 返回所有键 / 值的数组  |
| `size()` / `is_empty()`         | 大小 / 是否为空        |
| `merge(other, overwrite=false)` | 合并另一个字典         |
| `duplicate(deep=false)`         | 复制                   |
| `clear()`                       | 清空                   |

**遍历**：

```gdscript
for key in dict:
    print(key, dict[key])

for key in dict.keys():
    ...
```

**模拟 Set（集合）**：用 Dictionary 只存键：

```gdscript
var set_like = {}
set_like["item1"] = true
set_like["item2"] = true
if set_like.has("item1"):
    ...
```

## 3. Packed\*Array（打包/紧凑数组）

专为**同质数据**设计的高性能数组，内存更紧凑，迭代和修改通常比 Typed Array 更快。

### 可用类型

- `PackedByteArray`
- `PackedInt32Array`
- `PackedInt64Array`
- `PackedFloat32Array`
- `PackedFloat64Array`
- `PackedStringArray`
- `PackedVector2Array`
- `PackedVector3Array`
- `PackedVector4Array`
- `PackedColorArray`

### 特点

- 只能存储特定类型的元素。
- 内存占用更少，适合大量数据（数千到上万元素）。
- 缺少 Array 的许多便利方法（如 `map`、`filter`）。
- 始终按引用传递。

```gdscript
var bytes = PackedByteArray([1, 2, 3])
var positions: PackedVector2Array = [Vector2(0,0), Vector2(1,1)]
bytes.append(4)
print(bytes.size())
```

**转换**：

```gdscript
var packed = PackedInt32Array([1, 2, 3])
var normal: Array = Array(packed)          # 转为普通 Array
var back = PackedInt32Array(normal)        # 转回
```

## 4. 对比与使用建议

| 类型                  | 灵活性         | 性能 | 内存 | 适用场景                      |
| --------------------- | -------------- | ---- | ---- | ----------------------------- |
| **Array**（未类型）   | 最高（可混合） | 较低 | 较高 | 通用、小数据、需要 map/filter |
| **Array[T]**（Typed） | 中             | 中   | 中   | 需要类型安全 + 便利方法       |
| **Packed\*Array**     | 低（固定类型） | 高   | 低   | 大量同类型数据、性能关键      |
| **Dictionary**        | 高（任意键值） | 中   | 中   | 键值映射、快速查找            |

**建议**：

- 日常开发优先用 `Array` 或 `Array[T]`。
- 大量数值/向量数据（如粒子位置、网格数据）用 `Packed*Array`。
- 需要快速按键查找用 `Dictionary`。
- 需要“集合”语义（去重）时用 Dictionary 模拟。
- 需要独立副本时务必调用 `duplicate()`（深拷贝用 `duplicate(true)`）。

## 参考

- [GDScript 基础 - 容器类型](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html)
- [Array 类参考](https://docs.godotengine.org/en/stable/classes/class_array.html)
- [Dictionary 类参考](https://docs.godotengine.org/en/stable/classes/class_dictionary.html)
- [静态类型（Typed Array / Dictionary）](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/static_typing.html)
