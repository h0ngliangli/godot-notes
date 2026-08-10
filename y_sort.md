# Godot Y Sort 用法总结

Y Sort 用于根据节点的 **Y 坐标**自动调整绘制顺序，实现 2D 游戏中的深度感（近大远小、前后遮挡）。常用于俯视角或 3/4 视角游戏。

> **Godot 4 变化**：旧的 `YSort` 节点已移除。现在使用 `CanvasItem` 的 `y_sort_enabled` 属性（Node2D、Sprite2D、TileMap、TileMapLayer 等均可使用）。

## 基本原理

启用后，引擎会根据节点的全局 Y 位置对子节点进行排序：

- **Y 值更大**（更靠屏幕下方）→ 后绘制 → **显示在上层**
- **Y 值更小**（更靠屏幕上方）→ 先绘制 → **显示在下层**

这模拟了“越靠下的物体越近”的透视效果。

## 基本用法

在父节点上启用 `y_sort_enabled`：

```gdscript
# 在编辑器中勾选，或代码中设置
$YSortParent.y_sort_enabled = true
```

所有作为其子节点的 CanvasItem（Sprite2D、CharacterBody2D 等）会自动按 Y 排序。

### 示例场景结构

```
World (Node2D)          ← y_sort_enabled = true
├── Ground (TileMapLayer)
├── Trees (Node2D)
│   ├── Tree1
│   └── Tree2
├── Player (CharacterBody2D)
└── NPCs
```

## 重要注意事项

### 1. 与 z_index 的关系
节点**只在相同 z_index 下**才会进行 Y 排序。不同 z_index 的节点仍然按 z_index 优先级绘制。

建议：需要互相 Y 排序的对象保持相同的 `z_index`。

### 2. 嵌套行为
Y Sort 支持嵌套：

- 父节点启用 + 子节点禁用 → 子节点作为一个整体参与父节点的排序，其子节点保持相对顺序不变。
- 这允许你用空的 Node2D 分组管理对象，而不破坏整体排序。

### 3. 原点位置很关键
排序基于节点的 `position`（原点）。

**最佳实践**：将角色/物体的 Sprite 原点设置在**脚底**（视觉底部），这样排序才符合“站在地面上”的直觉。

```gdscript
# 在 Sprite2D 中调整 offset，或在场景中把 Sprite 往上移，让节点原点在脚部
```

### 4. TileMap / TileMapLayer 的特殊设置

1. 在 **TileMapLayer** 上启用 `y_sort_enabled`。
2. 在 **TileSet 编辑器**中，为每个需要排序的瓦片设置 **Y Sort Origin**（通常设到瓦片底部）。
   - 选中瓦片 → 右侧面板 → Rendering → Y Sort Origin
   - 例如 32×64 的树，可设为 `(0, 32)`（从中心向下偏移到根部）。

不设置 Y Sort Origin 时，默认使用瓦片中心，容易导致排序错误。

## 常见问题与排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 排序完全不生效 | 父节点未启用，或 z_index 不同 | 确保共同父节点启用，且 z_index 一致 |
| 角色总是在物体前面/后面 | 原点位置不对 | 把角色原点移到脚底，瓦片设置正确的 Y Sort Origin |
| 动态实例化的物体不排序 | 添加到错误的父节点 | 确保添加到启用了 y_sort_enabled 的父节点下 |
| 多层 TileMap 互相遮挡错误 | 每层都单独 Y 排序 | 合理设置各层的 Y Sort Origin 偏移，或调整层级结构 |

## 代码示例

```gdscript
# 启用 Y Sort
func _ready() -> void:
    y_sort_enabled = true

# 动态添加可排序物体
func spawn_tree(pos: Vector2) -> void:
    var tree = TreeScene.instantiate()
    tree.position = pos
    # 确保添加到启用了 y_sort 的父节点
    $YSortContainer.add_child(tree)
```

## 使用建议

- 只在需要深度排序的 2D 场景中启用（有性能开销）。
- 角色和可交互物体统一放在一个启用了 Y Sort 的父节点下。
- 地面层可单独处理（通常不需要与角色互相排序）。
- 结合 `z_index` 可以做更复杂的分层（如 UI 永远在最前）。

## 参考

- [CanvasItem.y_sort_enabled](https://docs.godotengine.org/en/stable/classes/class_canvasitem.html#class-canvasitem-property-y-sort-enabled)
- [Using TileMaps](https://docs.godotengine.org/en/stable/tutorials/2d/using_tilemaps.html)
