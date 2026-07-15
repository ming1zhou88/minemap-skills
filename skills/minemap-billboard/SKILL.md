---
name: minemap-billboard
description: MineMap 3D 公告板（Billboard）专题。覆盖 BillboardMaterial / BillboardGeometry / BillboardInstance 动画、拾取高亮、动态纹理更新与轨迹对象联动；不与 DOM 覆盖物 Marker 混淆。
---

# MineMap Billboard（3D 公告板）

## 概述

MineMap 的 Billboard 是**面向 3D 场景的 Primitive 体系下的公告板**，不是 DOM 覆盖物。

-   DOM 标注用 `minemap.Marker` / `minemap.Popup`（见 `minemap-marker-and-popup`）
-   3D 公告板用 `minemap.BillboardMaterial` + `minemap.Geometries.BillboardGeometry` + `new minemap.Primitive({...})`
-   大量同形态公告板用 `BillboardInstance` 场景组件

它和 sprite 图层（`addLayer({ type: "sprite" })`，见 `minemap-layer-system`）也不是同一回事：sprite 是 style-layer 路径，billboard 是 Primitive 路径。

## Quick Start：单公告板

对应 demo：`demo/html/Billboard.html`

```javascript
const material = new minemap.BillboardMaterial({
	position: [116.245, 40.063, 0],
	scale: [1000, 140, 1], // X 宽 / Y 高 / Z 缩放
	center: new minemap.Math.Vector2(0.5, 1), // 锚点 [0..1, 0..1]
	rotation: 0, // 弧度
	image: canvasElement // 或 imageUrl
});

const primitive = new minemap.Primitive({
	id: "billboard-1",
	geometry: new minemap.Geometries.BillboardGeometry({}),
	material
});

map.addPrimitive(primitive);

// 动态更新纹理
setInterval(() => {
	material.image = updateCanvas();
}, 100);

// 移除
map.removePrimitive(primitive);
```

## Quick Start：BillboardInstance 动画

对应 demo：`demo/html/BillboardInstanceAnimation.html`

`BillboardInstance` 是同一形状下"成百上千个公告板"的高效形式（共享几何 + 实例化），典型场景是车辆 / 人员 / 设备实时位置。

```javascript
const material = new minemap.BillboardMaterial({
	position: [119.447, 32.176, 0],
	scale: [800, 200, 1],
	center: new minemap.Math.Vector2(0.5, 1),
	rotation: 0,
	image: carIconImage
});

const geometry = new minemap.Geometries.BillboardGeometry({});

const billboardInstance = new minemap.BillboardInstance({
	name: "vehicles",
	geometry,
	material
});

// 通过 addInstance / setInstances 注入每个公告板的位置 / 状态
billboardInstance.addInstance({
	id: "car-1",
	position: [119.45, 32.18, 0],
	color: minemap.Color.WHITE.clone()
});

map.addPrimitive(billboardInstance);
```

## `BillboardMaterial` 公开参数

| 参数       | 说明                                     |
| ---------- | ---------------------------------------- |
| `position` | 经纬度+高度 `[lng, lat, alt]`            |
| `scale`    | `[X, Y, Z]`，X / Y 是世界米，Z 是缩放    |
| `center`   | `Vector2`，锚点 [0..1, 0..1]             |
| `rotation` | 弧度                                     |
| `image`    | `HTMLCanvasElement` / `HTMLImageElement` |
| `imageUrl` | 与 `image` 二选一；URL 形式              |
| `flipY`    | 翻转纹理 Y（参考 WebGL 纹理属性）        |
| `color`    | `Vector4` (r, g, b, a) 颜色板            |

> `image` 是元素，`imageUrl` 是路径。`material.image = newCanvas` 可以热替换纹理。

## `BillboardGeometry`

`new minemap.Geometries.BillboardGeometry({})` 不需要参数，公告板形状由 `BillboardMaterial.scale` / `center` 决定。

## `BillboardInstance` 关键行为

来源：demo `BillboardInstanceAnimation.html` 注释

1. **拾取高亮**：

    ```javascript
    billboardInstance.color = new minemap.Color(1.0, 0.0, 0.0, 0.3);
    ```

2. **动画回调**：

    ```javascript
    billboardInstance.animationCallbackFunction = (instance) => {
    	// 每帧拿 instance.position / instance.color 等
    };
    ```

3. **记录动画位置**（用于绘制实时轨迹）：

    ```javascript
    billboardInstance.recordAnimationPositions = true;
    const track = billboardInstance.animationPositions; // 数组
    ```

4. **取消高亮 / 复位**：

    ```javascript
    selectedBillboard.color = originalColor;
    selectedBillboard.animationCallbackFunction = null;
    selectedBillboard.recordAnimationPositions = false;
    ```

## Architecture Positioning

-   **单公告板** = `BillboardMaterial` + `BillboardGeometry` + `Primitive`（最小单元）
-   **实例化公告板** = `BillboardInstance`（场景组件，复用 `BillboardMaterial` + `BillboardGeometry`）
-   **DOM 标注** = `Marker` / `Popup`（不在 Primitive 体系内，不参与 3D 深度）
-   **sprite style-layer** = `addLayer({ type: "sprite" })`（贴地线 / 流动线，业务上不冲突）

## Common Patterns

### 模式 1：实时数据画板（数字 / 面积 / 评分）

对应 demo：`Billboard.html`

-   Canvas 画板按需重画
-   `material.image = canvas` 热替换
-   不重建 Primitive，不重建 Material，性能可控

### 模式 2：车辆 / 人员实时位置

对应 demo：`BillboardInstanceAnimation.html`

-   用 `BillboardInstance` 一次性放成百上千个图标
-   `animationCallbackFunction` 拿当前位置画圆形波纹 / 轨迹 / popup
-   `recordAnimationPositions = true` 开轨迹记录

### 模式 3：拾取 + 高亮

```javascript
map.on("click", (e) => {
	const picked = map.pickPrimitive(e.point);
	if (picked && picked.primitive === billboardInstance) {
		const instance = picked.detail; // instance 句柄
		// 高亮 / 弹窗
	}
});
```

## Strict Constraints

### 1. 别把 Billboard 当成 Marker

-   `BillboardMaterial` 是 3D Primitive，会参与 depth-test，会被遮挡
-   `Marker` 是 DOM，永远在最上层
-   业务上要"在 3D 场景里被建筑挡住"用 Billboard；"始终显示"用 Marker

### 2. `image` vs `imageUrl` 二选一

两者都传会冲突，源码会优先 `image`。

### 3. `scale` 是世界米

不要把它当作像素。`scale: [800, 200, 1]` 表示宽 800 米、高 200 米；缩放级别变化时看起来会更近 / 更远，这是正常的。

### 4. `recordAnimationPositions` 会持续累积

长时间开动画会越积越多，业务上要么在合适时机 `recordAnimationPositions = false`，要么定期清。

### 5. `BillboardInstance` 拾取粒度

拾取到的是**整个实例集合**，再 `picked.detail` 拿具体 instance；不要在每个 instance 上挂 `onClick`。

## Failure Cases

-   公告板不显示 → `image` / `imageUrl` 没加载完；或 `position` 高度低于地形被裁
-   公告板太大 / 太小 → `scale` 单位是米，不是像素
-   公告板永远在前面 → 误用了 `Marker` / `Popup` 路径
-   实例化公告板全部是同一个色 → 高亮 `color` 设错了字段（要在 instance 上而不是 material 上）
-   动画位置空 → 没设 `recordAnimationPositions = true` 或 `animationCallbackFunction` 之前就已经访问 `animationPositions`

## Demo References

-   `demo/html/Billboard.html`：单公告板 + Canvas 实时画板
-   `demo/html/BillboardInstanceAnimation.html`：车辆 / 设备级 BillboardInstance 动画 + 拾取 + 轨迹

## See Also

-   `minemap-marker-and-popup`：DOM 覆盖物 Marker / Popup
-   `minemap-primitives-and-materials`：Primitive 体系
-   `minemap-business-roaming-and-tracking`：轨迹 / 漫游
-   `minemap-layer-system`：`sprite` style-layer（不与 3D Billboard 混用）
