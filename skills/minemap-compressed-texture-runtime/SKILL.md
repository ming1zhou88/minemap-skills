---
name: minemap-compressed-texture-runtime
description: "MineMap 压缩纹理运行时排查。适用于 KTX2、Basis、CRN、DDS/DXT、压缩纹理 mip 层级、WebGL1/WebGL2 纹理兼容、摩尔纹、黑纹理、TextureLoader 压缩资源，以及 S3M/glTF 压缩纹理问题定位。"
---

# MineMap 压缩纹理运行时

## 快速判断

压缩纹理通常由加载器创建，业务代码不应把它当普通图片手动拼装。排查渲染异常时，先看纹理来源和运行路径，再改材质：

```javascript
const texture = minemap.TextureLoader.load({
	url: "./assets/model-texture.ktx2"
});
```

如果模型出现远处摩尔纹、黑纹理、只显示 level 0 清晰度等问题，优先确认压缩纹理是否带完整 mip 链，以及是否触发 WebGL1 的 NPOT 限制。

## 架构位置

当前压缩纹理路径主要落在：

- `source/webgl/context/CompressedTexture.js`
- `source/third-party/loaders.gl/textures/src/lib/parsers/parse-basis.js`
- `source/third-party/loaders.gl/gltf/src/lib/parsers/parse-gltf.js`
- `source/third-party/s3m/S3MParser/S3ModelParser.js`
- `source/third-party/s3m/S3MTiles/Core/CRNTranscoder.js`

不要把 KTX2/Basis/CRN 当成 PNG/JPEG。它们携带 GPU 格式、mip 层级和转码约束。

## KTX2 mip 链规则

近期源码专门处理 KTX2 多级解析，用来降低摩尔纹并避免非法 mip 上传。

`CompressedTexture` 的关键规则是：

- 如果传入的 mip 链有效，并且能完整降到 `1x1`，保留压缩 mip 链。
- 如果 mip 链无效、不完整或尾部层级重复异常，只保留 level 0，解压成 RGBA 后重新生成 mipmap。

这样可以避免不完整压缩 mip 链在采样时导致黑纹理或 `INVALID_VALUE` 上传错误。

## WebGL1 兼容

WebGL1 对 NPOT 纹理更严格。非二次幂纹理如果还要求 mipmap filter，需要：

- 将 `wrapS` 与 `wrapT` 降为 `CLAMP_TO_EDGE`
- 将 mipmap filter 降为 `LINEAR` 或 `NEAREST`
- 将 `levels` 设为 `1`
- 禁止生成 mipmap

只在 WebGL1 复现的问题，先查 NPOT 尺寸和 sampler filter，不要直接归因到材质 shader。

## S3M 与 CRN

S3M 的 CRN 解码使用 `CRNTranscoder`，并依赖 WASM 资源。修改 S3M 材质代码前，先确认：

- `window.minemapCDN` 指向包含 CRN 转码资源的 CDN。
- CRN/WASM 网络请求成功。
- `S3MTilesLayer.update(...)` 已经通过 `CRNTranscoder.wasmReady` 门控。

S3M parser 会把 DXT、WebP、CRN、KTX2、CRN_DXT5 等压缩类型映射到内部像素格式。这个映射应留在 parser 层，不要在业务代码里复制一套。

## 排查清单

摩尔纹：

- 确认源 KTX2 是否真的包含 mip 链。
- 确认 `CompressedTexture._analyzeProvidedMipChain(...)` 是否会接受该链。
- 查看运行时是否已回退到 RGBA 后生成 mipmap。
- 确认各向异性和 min filter 没被业务层意外覆盖。

黑纹理：

- 查是否用 mipmap filter 采样了不完整 mip 链。
- 查 `compressedTexImage2D` / `compressedTexSubImage2D` 附近 WebGL 错误。
- 查 WebGL1 NPOT 限制。
- 查 S3M CRN/WASM CDN 资源是否缺失。

模糊：

- 确认 WebGL1 NPOT 降级没有把纹理强制成只保留 level 0。
- 查 `useMipmap`、`levels`、`minFilter` 和纹理尺寸。

## 禁用写法

- 不要为了修 KTX2 摩尔纹而全局关闭所有 mipmap。
- 不要盲目上传解析出的所有 KTX2 层级；必须先校验 mip 链形态。
- 不要在 WebGL1 路径里假设 WebGL2 的 immutable storage 能力可用。
- 不要在应用层另写一套临时 CRN/Basis parser。

## 源码依据

- `source/webgl/context/CompressedTexture.js`
- `source/third-party/loaders.gl/textures/src/lib/parsers/parse-basis.js`
- `source/third-party/loaders.gl/textures/dist/basis-worker.js`
- `source/third-party/loaders.gl/gltf/src/lib/parsers/parse-gltf.js`
- `source/third-party/s3m/S3MTiles/Core/CRNTranscoder.js`
- `source/third-party/s3m/S3MParser/S3ModelParser.js`

## 相关技能

- `minemap-material-system-and-shading`
- `minemap-performance-and-backend`
- `minemap-s3m-tiles`
- `minemap-model-runtime-debug`
