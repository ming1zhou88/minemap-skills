---
name: minemap-plugin-compare-swipe
description: MineMap 卷帘对比插件规范。用于双地图 Compare 卷帘、同步视角与插件销毁。
---

# MineMap Plugin: Compare Swipe

## Architecture Positioning

卷帘对比不是 MineMap 主包内核对象，而是 demo 依赖的外部 compare 插件。

对应 demo：

-   `demo/html/RollerShutterContrast.html`

真实接入方式是额外引入：

-   `minemap-compare.js`
-   `minemap-compare.css`

## Demo-backed Entry Surface

-   双 `new minemap.Map(...)`
-   `new Compare(mapA, mapB, isHorizontal, options, direction)`

## Demo-backed Patterns

### 模式 1：卷帘本质是双地图同步裁切

demo 里同时创建：

-   `map`
-   `mapCompare`

再交给 `Compare(...)` 组织裁切与同步。

### 模式 2：容器必须禁用文本选择

demo 明确给两个地图容器加了：

-   `user-select: none`

这是为了让拖动卷帘更平滑。

### 模式 3：销毁不只是 remove 地图对象

demo 里额外清理：

-   裁切样式
-   插件生成的垂直/水平 DOM

说明 compare 插件销毁要连 UI 痕迹一起清掉。

## Strict Constraints

### 1. 这是插件，不是内核 API

不要把 `Compare` 写成 MineMap 主包稳定导出能力。

### 2. 双地图底图和相机参数要尽量一致

否则卷帘边界两侧会明显错位。

## Failure Cases

-   只创建一个地图就想做卷帘
-   忽略插件 CSS / JS 依赖
-   销毁时不清理插件 DOM

## Demo References

-   `demo/html/RollerShutterContrast.html`

## See Also

-   `minemap-style-system`
-   `minemap-plugin-demo-editor`
