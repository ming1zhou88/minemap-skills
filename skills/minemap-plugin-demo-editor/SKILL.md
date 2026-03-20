---
name: minemap-plugin-demo-editor
description: MineMap demo 在线编辑器规范。用于 Monaco 编辑、iframe 预览、WebGL 模式切换与示例试跑台。
---

# MineMap Plugin: Demo Editor Workbench

## Architecture Positioning

`demo/html/editor-map.html` 不是引擎运行时 API，而是 demo 体系里的“在线试跑台 / 示例编辑器”。

它的职责不是渲染地图业务，而是提供：

-   Monaco 编辑器
-   iframe 预览区
-   示例加载
-   WebGL1 / WebGL2 切换
-   左右分栏拖拽

因此这个 skill 的定位是“工具链 / demo 工位规范”，不是业务地图能力规范。

## Demo-backed Entry Surface

从 `editor-map.html` 可确认的核心组织包括：

-   `require(['vs/editor/editor.main'], ...)`
-   `monaco.editor.create(...)`
-   `$.get('./' + id + '.html')` 按 query 参数加载示例
-   `run()` 中用 iframe 重新写入 HTML
-   `localStorage` 记录 WebGL 模式
-   当选择 `webgl1` 时，对 `new minemap.Map({...})` 注入 `requestWebgl1: true`

## Recommended Workflow

1. 用 query 参数决定加载哪个 demo
2. Monaco 只维护源码文本
3. 运行时把文本写进独立 iframe
4. WebGL 模式切换放到 demo 注入层，不要改每个示例文件
5. 分屏拖拽时暂停 iframe 指针事件，避免地图吞鼠标

## Demo-backed Patterns

### 模式 1：示例文件按 `id` 动态装载

`editor-map.html` 通过 URL 的 `id` 参数加载具体 demo HTML。

这说明示例工作台最稳的组织方式不是把所有示例硬编码到页面里，而是：

-   编辑器壳页面固定
-   业务示例页面按 id 外部装载

这样更适合扩 demo 库。

### 模式 2：源码与预览彻底隔离

demo 不是直接执行编辑器 DOM 里的脚本，而是：

1. 读取编辑器内容
2. 创建 iframe
3. `document.write(adjustedHtml)`

这意味着示例执行环境和编辑器工作台是隔离的，便于重置、重跑和避免样式互相污染。

### 模式 3：统一注入 WebGL 模式

`editor-map.html` 用 `applyWebglMode(html)` 在运行前改写源码：

-   默认保持 WebGL2
-   当切到 `webgl1` 时，在 `new minemap.Map({` 后注入 `requestWebgl1: true`

这是 demo 工具层很好的做法，因为：

-   不需要改每个示例文件
-   可对同一份示例做后端兼容回归

### 模式 4：编辑器布局与地图交互解耦

分屏拖拽时，demo 做了两件关键事：

-   暂时禁用 iframe 指针事件
-   拖拽结束后恢复

否则地图会在拖拽过程中截获鼠标事件，导致分栏体验很差。

### 模式 5：工具壳只负责工作流，不负责业务逻辑

`editor-map.html` 并没有把 MineMap 业务逻辑直接写死在壳里，而是把职责限定为：

-   选 demo
-   编辑 demo
-   运行 demo
-   调整运行环境

这是后续扩更多 demo 或教程页面时最稳的方式。

## Strict Constraints

### 1. 这是 demo 工具，不是引擎 API

不要把 `editor-map.html` 当成业务项目要直接依赖的生产能力。它适合：

-   文档站
-   教学平台
-   内部示例工位
-   调试环境

### 2. WebGL 模式切换应做成注入层能力

demo 已经证明，把 `requestWebgl1: true` 的注入逻辑集中在编辑器壳层，比修改每个示例文件更可维护。

### 3. 运行区一定要隔离

直接在主页面执行 demo 代码，会让：

-   全局变量互相污染
-   样式串扰
-   重跑不可控

iframe 是这里的关键隔离层。

### 4. 拖拽分栏时要处理 iframe 交互冲突

只做 CSS 拉伸而不处理 pointer events，地图类 demo 很容易拖不动分隔条。

## Failure Cases

### 失败场景 1：把它当生产态在线编辑器方案

这个 demo 更适合教学和调试，不适合作为完整线上 SaaS 编辑平台直接照搬。

### 失败场景 2：切 WebGL 模式靠人工改源码

会导致示例文件越来越脏，也不利于批量回归测试。

### 失败场景 3：在主页面直接执行预览代码

一旦示例内部也操作全局对象、键盘事件或样式表，工作台本身会被污染。

## Performance Discipline

-   编辑器内容变更后可自动重跑，但要控制刷新频率
-   预览区重建 iframe 时要清空旧节点，避免残留实例
-   窗口尺寸变化后及时触发 Monaco `layout()`
-   示例库越来越大时，继续保持“壳页面 + 外部 demo 文件”模式

## Demo References

-   `demo/html/editor-map.html`
-   `demo/index.html`
-   `demo/src/config.js`
-   `demo/src/index.js`（旧版 CodeMirror 试跑页，可作为历史对照）

## See Also

-   `minemap-performance-and-backend`
-   `minemap-fundamentals`
-   `minemap-plugin-edit`
-   `minemap-plugin-echarts-integration`
