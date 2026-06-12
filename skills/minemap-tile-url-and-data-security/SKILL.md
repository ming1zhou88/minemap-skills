---
name: minemap-tile-url-and-data-security
description: "MineMap 瓦片请求地址模板、内部协议归一化、鉴权参数与数据安全边界。适用于配置 raster/vector/terrain tiles，处理 {z}/{x}/{y}、{z}/{r}/{c}/{z}_{x}_{y}、{TileMatrix}/{TileCol}/{TileRow}、WMTS/TMS/WMS、cloudservice/minemapdata/mineservice 等地址，以及说明数据加密接入要求但不能暴露加解密算法。"
---

# MineMap 瓦片地址与数据安全

## 快速写法

MineMap demo 中最常见的静态瓦片地址是 `{z}/{r}/{c}/{z}_{x}_{y}`：

```javascript
map.addSource("satellite", {
	type: "raster",
	tiles: ["https://staticdata-b.minedata.cn:14434/satellite/{z}/{r}/{c}/{z}_{x}_{y}.jpg"],
	tileSize: 256
});
```

地形也使用同一套分目录规则：

```javascript
map.addSource("dem", {
	type: "raster",
	tiles: ["https://staticdata-b.minedata.cn:14434/dem/{z}/{r}/{c}/{z}_{x}_{y}.terrain"],
	tileSize: 512
});
```

如果服务是普通 XYZ，可继续使用 `{z}/{x}/{y}`；不要为了统一格式强行加 `{r}/{c}`，服务端路径必须和真实切片目录一致。

## 模板替换规则

`CanonicalTileID#url(...)` 支持这些占位符：

- `{z}`：瓦片级别。
- `{x}` / `{y}`：XYZ 坐标，`scheme: "xyz"` 时 y 不翻转。
- `{r}` / `{c}`：MineMap 常用分级目录，低级别为 `R0/C0`，`z > 6` 后按当前瓦片坐标分段生成。
- `{z}_{x}_{y}`：常见文件名组合，通常和 `{z}/{r}/{c}/` 目录配合。
- `{m}` / `{n}`：按 `x / 16`、`y / 16` 取整的老式分片目录。
- `{quadkey}`：四叉树 key。
- `{bbox}`：按当前投影生成 bbox。
- `{crs}`：`LATLON` 为 `EPSG:4326`，`MERCATOR` 为 `EPSG:3857`。
- `{TileMatrix}` / `{TileCol}` / `{TileRow}`：WMTS 风格占位符，也可在普通 URL 模板里替换。

`tmsUrl(...)` 会翻转 y，并额外支持 `{bbox-epsg-3857}`。`wmtsUrl(...)` 用 `capabilities` 填充 `{TileMatrixSet}`、`{TileMatrix}`、`{TileCol}`、`{TileRow}` 以及维度参数。

## `{r}/{c}` 目录规则

`{r}` 和 `{c}` 不是外部服务随意给的字符串，而是源码内置计算：

- `z <= 6` 时固定为 `R0` / `C0`。
- `z > 6` 时，分段大小为 `1 << (z - 5)`。
- `{c}` 为 `C${Math.floor(x / 分段大小)}`。
- `{r}` 为 `R${Math.floor(y / 分段大小)}`。

因此 `https://.../{z}/{r}/{c}/{z}_{x}_{y}.pbf` 在 z/x/y 已知时会变成类似：

```text
https://.../9/R12/C26/9_420_193.pbf
```

排查 404 时，先确认服务端目录是否按同一规则切片，再确认 `scheme` 是否误设成 `tms`。

## 内部协议与鉴权参数

MineMap 支持若干内部协议，业务代码可以配置逻辑地址，由 `urlTransformUtil` 统一归一化：

- `minemap://...`：基于 `SRC_URL` 解析资源。
- `minemapdata://...`、`minemapdatad://...`、`minemapdatam://...`、`minemapdatao://...`：基于不同数据域名解析。
- `cloudservice://...`：基于 `SERVER_URL` 解析云服务路径，常见于 `cloudservice://mvt-layer/merge/{z}/{r}/{c}/{z}_{x}_{y}.pbf`。
- `mineservice://...`：基于 `SERVICE_URL` 解析服务路径。
- `mineserver://...`：基于 `SERVER_URL` 解析。

`normalizeTileURL(...)` 会在地址包含 `auth=true` 或使用 MineMap 内部协议时进入归一化流程。开启访问令牌要求后，`makeAPIURL(...)` 会按配置追加 `key`、`appKey` 或 `token`，并可追加 `solu`、`version`。

不要在业务层自己拼接一套鉴权参数优先级；统一走引擎配置和 URL 归一化入口。

## 数据加密说明

涉及加密数据时，只写接入边界，不写算法细节：

- 可以说明“该数据由内部安全链路处理，业务侧只配置授权地址、key/token、solution/version 等公开接入参数”。
- 可以说明“客户端不得自行实现、复刻或推断加解密过程；加密、解密、密钥派生、密钥轮换等均属于内部保密实现”。
- 可以说明“排查时只检查请求地址、鉴权参数、服务响应状态、CORS、数据版本、瓦片坐标模板是否正确”。
- 不要写加密算法名称、模式、密钥格式、密钥来源、解密伪代码、逆向步骤、抓包还原过程或任何可复现内部实现的描述。

如果用户要求“解释加密算法”或“补解密代码”，应明确拒绝暴露内部实现，并改为提供安全接入检查清单。

## 常见错误

- 把 `{r}` / `{c}` 当成可手填常量，导致高层级瓦片目录错位。
- 服务是 TMS，却按 XYZ 解释 y，或者反过来。
- 在 WMTS 地址里忘记使用 `{TileMatrix}` / `{TileCol}` / `{TileRow}`。
- 复制 Mapbox 风格地址时只保留 `{z}/{x}/{y}`，漏掉 MineMap 服务实际需要的 `{r}/{c}` 目录。
- 自己在业务代码里拼 `key/appKey/token/solu/version`，和全局配置产生重复或冲突。
- 为了排查加密数据而尝试还原内部算法。排查应停在接入参数、请求链路和服务响应层。

## 源码与 demo 依据

- `source/data-source/tile/TileID.js`
- `source/util/urlTransformUtil.js`
- `source/config/MinemapConfig.js`
- `demo/html/TransformRequest.html`
- `demo/html/3DPick.html`
- `demo/html/S3M.html`
- `demo/html/VectorMapAdaptTerrain.html`

## 相关技能

- `minemap-global-configuration`
- `minemap-style-and-data`
- `minemap-style-v8-differences`
- `minemap-terrain-and-analysis`
