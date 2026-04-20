---
name: minemap-global-configuration
description: MineMap 全局配置、key/solution、域名地址、sprite/fonts、minemapCDN 与启动前配置顺序。用于指导用户正确完成地图初始化前的配置，不再把 accessToken 作为主推荐路径。
---

# MineMap Global Configuration

## Quick Start

```javascript
// 先配全局，再创建第一个 Map
minemap.domainUrl = "https://minedata.cn";
minemap.dataDomainUrl = "https://minedata.cn";
minemap.serverDomainUrl = "https://minedata.cn";
minemap.spriteUrl = "https://minedata.cn/minemapapi/sprite/latest/sprite";
minemap.serviceUrl = "https://minedata.cn/service/";

minemap.key = "<替换成你的 key>";
minemap.solution = 11001;
window.minemapCDN = "https://minedata.cn/minemapapi/minemap-CDN";

const style = {
	version: 8,
	glyphs: "minemap://fonts/{fontstack}/{range}",
	sprite: "minemap://sprite/sprite",
	sources: {},
	layers: []
};

const map = new minemap.Map({
	container: "map",
	style,
	center: [116.46, 39.92],
	zoom: 12
});
```

## Architecture Positioning

### 1) 这些不是普通变量，而是 `source/index.js` 暴露的全局配置入口

MineMap 把一批启动配置直接挂在 `minemap` 顶层对象上。

当前源码能确认的主要入口有：

-   `minemap.key`
-   `minemap.solution`
-   `minemap.domainUrl`
-   `minemap.dataDomainUrl`
-   `minemap.serverDomainUrl`
-   `minemap.serviceUrl`
-   `minemap.spriteUrl`
-   `minemap.fontsUrl`
-   `minemap.dataVersion`

兼容入口仍然存在，但不再作为主推荐：

-   `minemap.appKey`
-   `minemap.accessToken`

另外，很多 3D / worker 相关 demo 还会显式设置：

-   `window.minemapCDN`

### 2) 推荐顺序：`key` + `solution`，不要再把 `accessToken` 当主方案

`source/util/urlTransformUtil.js` 的真实优先级是：

-   先用 `key`
-   没有 `key` 再用 `appKey`
-   再没有才退到 `accessToken`

也就是：`key > appKey > accessToken`

而仓库里的 demo，绝大多数也都在顶部配置：

-   `minemap.key = ...`
-   `minemap.solution = ...`

因此这套 skill 的推荐是：

-   新代码优先配置 `key`
-   有底图样式、平台方案时同时配置 `solution`
-   `appKey` 只作为兼容或某些旧服务/显式 URL 场景的补充
-   不再把 `accessToken` 作为默认推荐写法

### 3) 为什么一定要在创建 `Map` 之前配置

这些全局值会参与 MineMap 内部的 URL 组装、样式资源解析和服务请求拼接。

如果先 `new minemap.Map(...)`，再去补这些配置，常见后果是：

-   首批样式或数据请求已经发出
-   sprite / glyphs 已按旧配置解析
-   worker / 3D 资源已经按默认地址去拉取

所以正确顺序是：

1. 先配全局
2. 再创建 `Map`
3. 再进入 `load` / `style.load` 后续逻辑

### 4) 各配置项的真实作用边界

#### `minemap.key`

主推荐鉴权参数。MineMap 在内部拼装 URL 时，会优先把它写成 `key=...`。

#### `minemap.solution`

方案 ID。内部会按 `solu=...` 拼进请求参数。只要你的底图、样式、服务能力和某个方案绑定，就应该一起配。

#### `minemap.domainUrl`

静态资源根域配置。`source/index.js` 的 setter 不只是简单赋值，还会联动重算：

-   `SRC_URL`
-   `SERVICE_URL`

而且它会把传入地址拆成“域名 + 资源前缀”两部分，所以它既可以接纯域名，也能接带资源路径的地址。

#### `minemap.dataDomainUrl`

数据域配置。setter 会联动更新：

-   `API_URL`
-   `DYN_URL`
-   `MERGE_URL`
-   `OTHER_URL`

源码支持字符串，也支持数组。

适合：

-   切到私有化数据域
-   多数据域分流

#### `minemap.serverDomainUrl`

分析/服务域配置。setter 会联动更新 `SERVER_URL`，同样支持字符串或数组。

#### `minemap.serviceUrl`

直接覆盖服务根地址。很多 demo 都显式设置它，用来明确 style / service 请求走哪个服务域。

#### `minemap.spriteUrl`

雪碧图覆盖地址。`source/style/loadSprite.js` 表明，它会和 style 里的 `sprite` 一起参与解析，而且配置项优先级很高。

源码还支持：

-   字符串
-   数组

数组模式可用于多 sprite 源。

#### `minemap.fontsUrl`

字形资源覆盖地址。不是每个 demo 都配，但源码和 `VectorMap2.1.html` 表明这是合法入口，适合旧字体服务或私有字体服务迁移。

#### `minemap.dataVersion`

内部会转成请求参数 `version=...`。适合和数据发布版本联动。

#### `window.minemapCDN`

这不是 `minemap` 顶层 getter/setter，但对 3D / worker 资源非常关键。

源码里 `loaders.gl` worker 和 CDN 选项都会读取它。没配时，很多 3D demo 的 worker 资源路径就不稳定。

## Common Patterns

### 最常见的 demo 启动模板

仓库里大量 demo 都是这一段结构：

```javascript
minemap.domainUrl = "https://minedata.cn";
minemap.dataDomainUrl = "https://minedata.cn";
minemap.serverDomainUrl = "https://minedata.cn";
minemap.spriteUrl = "https://minedata.cn/minemapapi/sprite/latest/sprite";
minemap.serviceUrl = "https://minedata.cn/service/";
minemap.key = "...";
minemap.solution = 11001;
window.minemapCDN = "https://minedata.cn/minemapapi/minemap-CDN";
```

如果你是不知道“到底要配哪些字段”，先按这个最小模板起步，通常最稳。

### 纯 2D / 空底图场景

如果你自己传的是空 style：

```javascript
const style = {
	version: 8,
	glyphs: "minemap://fonts/{fontstack}/{range}",
	sprite: "minemap://sprite/sprite",
	sources: {},
	layers: []
};
```

那通常还是建议把下面几项先配好：

-   `key`
-   `solution`
-   `domainUrl`
-   `dataDomainUrl`
-   `serverDomainUrl`
-   `spriteUrl`

因为后面你很可能还是会加 source、layer、image、scene component。

### 3D / 模型 / 3D Tiles 场景

只要页面里有：

-   `3d-model`
-   `3d-tiles`
-   loaders.gl worker
-   部分高阶 3D 资源解析

就建议把 `window.minemapCDN` 一起配上。

这是 3D demo 中最常见的补充项之一。

### 多 sprite 源

`VectorGeojsonOverlap.html` 证明 `minemap.spriteUrl` 可以直接给数组：

```javascript
minemap.spriteUrl = ["https://host-a/sprite/sprite", "https://host-b/sprite/sprite"];
```

适合：

-   多风格 icon 合并
-   迁移阶段同时挂旧 sprite 和新 sprite

### 字体服务覆盖

`VectorMap2.1.html` 证明可以单独配置：

```javascript
minemap.fontsUrl = "https://your-host/fonts";
```

适合：

-   历史项目字体兼容
-   私有化字体服务
-   版本迁移时的 glyphs 定位修正

## 从 demo 里找配置的正确姿势

用户最容易卡住的，其实不是 `Map` 参数，而是“前面那段全局配置到底怎么写”。

建议直接看 demo 页面最上方的初始化段，通常就在创建 `Map` 之前。

优先参考这些页面：

-   `demo/html/3DTiles.html`
-   `demo/html/3DTilesLoaded.html`
-   `demo/html/3DModelViewer.html`
-   `demo/html/ExtrusionLightStripScanning.html`
-   `demo/html/VectorMap2.1.html`
-   `demo/html/3DTilesOpacity.html`

观察重点不是抄整页代码，而是看这几个字段：

-   `domainUrl`
-   `dataDomainUrl`
-   `serverDomainUrl`
-   `spriteUrl`
-   `serviceUrl`
-   `key`
-   `solution`
-   `window.minemapCDN`

### 关于 demo 里的 key

仓库里有两种情况：

1. 有些 demo 带了可运行示例 key
2. 有些 demo 明确写成 `<您的 key>` / `<your key>` 占位

正确理解是：

-   demo 顶部那段配置可以当“字段模板”
-   占位 key 必须自己替换
-   带示例 key 的页面可用于本地理解配置结构，但正式项目不要依赖仓库示例值

## Failure Cases

### 1) 先 `new Map()`，后补全局配置

这是最常见错误。

MineMap 的资源 URL 和样式资源解析很多都是启动早期就发生的。后补配置，通常来不及。

### 2) 还在把 `accessToken` 当主推荐方案

源码虽然还保留 `accessToken`，但当前真实优先级已经是：

-   `key`
-   `appKey`
-   `accessToken`

而且当前 demo 主流也不是这么写。

所以不要再把它当主教程入口。

### 3) 只配 `domainUrl`，不配 `dataDomainUrl` / `serverDomainUrl`

在私有化、分域部署、服务域拆分场景里，这会导致：

-   样式资源和数据资源不在同一套地址
-   某些请求走对了，某些请求还在走默认域

### 4) 3D 场景忘了 `window.minemapCDN`

很多 2D 页面不一定会立刻暴露这个问题，但 3D / worker 相关场景非常容易踩坑。

### 5) `spriteUrl` 和 style 内 `sprite` 配置混着改，却不知道谁生效

`loadSprite.js` 的逻辑说明：全局 `minemap.spriteUrl` 会介入 sprite 解析。

如果你全局已覆盖，就不要一边指望 style 里的 `sprite` 生效，一边再全局强覆盖，最后自己也分不清实际请求地址。

### 6) 在顶层全局配置里优先用 `appKey`

`appKey` 不是不能用，但它更适合：

-   历史项目兼容
-   某些手工拼接服务 URL 的场景
-   demo 中少数旧页面

主配置入口还是优先 `key`。

## Recommended Startup Recipe

如果你要给别人一段“先跑起来”的 MineMap 配置模板，优先给这套：

```javascript
minemap.domainUrl = "https://minedata.cn";
minemap.dataDomainUrl = "https://minedata.cn";
minemap.serverDomainUrl = "https://minedata.cn";
minemap.spriteUrl = "https://minedata.cn/minemapapi/sprite/latest/sprite";
minemap.serviceUrl = "https://minedata.cn/service/";
minemap.key = "<你的 key>";
minemap.solution = 11001;
window.minemapCDN = "https://minedata.cn/minemapapi/minemap-CDN";
```

然后再根据项目情况增量调整：

-   私有字体服务 → `fontsUrl`
-   多数据域 → `dataDomainUrl` 数组
-   多 sprite → `spriteUrl` 数组
-   数据版本控制 → `dataVersion`

## Demo References

-   `demo/html/3DTiles.html`
-   `demo/html/3DTilesLoaded.html`
-   `demo/html/3DTilesOpacity.html`
-   `demo/html/3DModelViewer.html`
-   `demo/html/ExtrusionLightStripScanning.html`
-   `demo/html/VectorGeojsonOverlap.html`
-   `demo/html/VectorMap2.1.html`

## See Also

-   `minemap-fundamentals`
-   `minemap-style-system`
-   `minemap-style-and-data`
-   `minemap-scene-components`
-   `minemap-performance-and-backend`
