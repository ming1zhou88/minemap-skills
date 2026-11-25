---
name: minemap-framework-lifecycle
description: MineMap 在 React/Vue 等框架中的挂载、resize、异步请求与销毁。用于路由切换、重复挂载、底图重载、监听和资源泄漏排查。
metadata:
  minemap-version: 4.28.2
---

# MineMap Framework Lifecycle

## 实例所有权

组件持有一个 Map，放在 ref/浅引用中，避免 Vue 深层代理 GPU 对象。全局配置完成、有尺寸容器就绪后创建。绑定 React effect cleanup / Vue onBeforeUnmount 到同一销毁函数。

下面是框架无关模式；mapOptions 与 URL 由业务提供：

```javascript
function mountMap(container, mapOptions, geojsonUrl) {
  const map = new minemap.Map({ ...mapOptions, container });
  let disposed = false;
  let data;
  const request = new AbortController();
  const observer = new ResizeObserver(() => { if (!disposed) map.resize(); });
  observer.observe(container);
  function restoreOverlay() {
    if (disposed || !data || !map.isStyleLoaded()) return;
    if (!map.getSource("business")) map.addSource("business", { type: "geojson", data });
    if (!map.getLayer("business-points")) map.addLayer({
      id: "business-points", type: "circle", source: "business",
      paint: { "circle-radius": 5, "circle-color": "#22c7a9" }
    });
  }
  map.on("style.load", restoreOverlay);
  map.on("load", restoreOverlay);
  fetch(geojsonUrl, { signal: request.signal })
    .then(response => {
      if (!response.ok) throw new Error(`GeoJSON: ${response.status}`);
      return response.json();
    })
    .then(result => { if (!disposed) { data = result; restoreOverlay(); } })
    .catch(error => { if (!disposed && error.name !== "AbortError") console.error(error); });
  return () => {
    if (disposed) return;
    disposed = true;
    request.abort();
    observer.disconnect();
    map.off("style.load", restoreOverlay);
    map.off("load", restoreOverlay);
    map.remove();
  };
}
```

## 清理和恢复

- 清应用 RAF/interval/timeout、observer、fetch 和外部订阅，Map 不会代清业务任务。
- off 的 type、layerId 和函数引用与 on 一致；异步拾取/模型加载不能取消时，用 disposed 或请求序号丢弃过期结果。
- map.remove 前清外部 renderer、WindRenderer、插件、Marker/Popup 和应用图元。remove/destroy 是否同时释放几何和材质须按类核对，共享资源不能重复销毁。
- StrictMode 挂载→卸载→再挂载要求 cleanup 幂等。map.remove 后不再调用 Map 方法。
- load 用于初次加载，底图切换用 style.load 幂等补回业务 source/layer；keepUserInfo 可能已经保留，须判重。
- ResizeObserver 覆盖侧栏/标签页尺寸变化；隐藏容器显示后 resize。
- 本版本 triggerRepaint 同步调用渲染，不是 Mapbox 合帧请求；不在 render 回调中调用以免递归。已有主循环时先更新对象状态，需立即重绘时在回调外调用一次。
- style 切换期间暂缓数据/状态入口，恢复就绪后重放业务状态。

## 证据

- `source/api/map.js`：resize/remove/triggerRepaint/style 更新。
- [minemap-fundamentals](../minemap-fundamentals/SKILL.md)、[minemap-style-system](../minemap-style-system/SKILL.md)、[minemap-wind-field](../minemap-wind-field/SKILL.md)。
