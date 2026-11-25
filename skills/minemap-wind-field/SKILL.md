---
name: minemap-wind-field
description: MineMap 规则网格风场和 GPU 粒子轨迹。用于 WindRenderer 数据转换、色带、粒子预算、启停和清理；体标量数据使用 volume-cloud。
metadata:
  minemap-version: 4.28.2
---

# MineMap Wind Field

## 公开入口

4.28 系列公开导出 WindRenderer；构造函数自动添加图元并启动模拟。在地图 load 后创建，不再调用 addLayer/addSceneComponent：

```javascript
const wind = new minemap.WindRenderer(map, {
  width: 2, height: 2, bbox: [110, 30, 111, 31],
  u: { array: [2, 3, 2, 3] }, v: { array: [1, 1, 2, 2] }
}, {
  particleCount: 2048, height: 100, frameRate: 30,
  colors: ["#22c7a9", "#2878bd"], domain: { min: 0, max: 10 }
});
wind.setOptions({ lineWidth: { min: 1, max: 2 }, displayRange: { min: 0, max: 8 } });
wind.setVisible(false);
// 页面卸载时，在 map.remove() 前执行：
wind.remove();
```

## 数据与运行约束

- 每轴至少 2 个点；U/V 分量用 `{ array }`，不是裸数组。支持小写 u/v 和大写 U/V。
- width/height/bbox 可替代为 dimensions.lon/lat 和 bounds；另给 lon/lat 轴数组时仍须提供边界，轴长度须匹配。
- 展平索引是 `y * width + x`。U/V 至少包含 width×height 个值；多层数据不会自动按高度播放，应用先选水平层。
- 数据方向和 flipY 一起验收。noDataValue 或非有限分量会置零，不能当作缺测区域掩膜。
- 依赖浮点纹理和浮点颜色附件支持；按实际构造能力检测判断，不能只检查 WebGL2。
- 实际粒子数是纹理边长的平方；particleCount 开方后向上取整。particlesTextureSize 优先。
- 改粒子预算重建 GPU 管线，改高度重建几何体，滑条更新应节流。
- domain 控制色带映射，displayRange 控制显示速度范围；线宽是像素范围，height/particleHeight 是米。
- stop 只停模拟，隐藏使用 setVisible(false)，setVisible(true) 会 start。
- 当前 painter 仅保存一个 windRenderer 引用，一个 map 管理一个活跃实例。换数据先 remove 再创建，不虚构 setData API。
- 引擎主循环已调用 update，不额外建立 RAF 或访问 painter。清理早于 map.remove，避免 remove 事件里访问已销毁的 GPU。

## 证据与相关技能

- `source/index.js`
- `source/renderer/effects/wind/WindRenderer.js`
- `source/renderer/effects/wind/WindField.js`
- `source/renderer/Painter.js`
- `demo/html/Wind.html`
- [minemap-volume-cloud](../minemap-volume-cloud/SKILL.md)、[minemap-framework-lifecycle](../minemap-framework-lifecycle/SKILL.md)、[minemap-performance-and-backend](../minemap-performance-and-backend/SKILL.md)。
