# OceanTemperatureAnalyzer 适用与限制

`source/index.js` 导出了 OceanTemperatureAnalyzer，实现位于 `source/webgl/object/OceanTemperatureAnalyzer.js`。公开方法包含 bind(map, volumeShell)、startDrawing、stopDrawing、clearTrajectory、getAnalysisResult、getTrajectory、destroy；构造参数有 sampleInterval、interpolationMode、onAnalysisComplete/onTrajectoryChange。

此类绑定 VolumeShell 的全球经纬 UV 与 CPU 数据缓存，不等于可对任意体数据、VolumeCloud 或模型温度执行通用查询。采样流程依赖私有 _volumeDataCache；没有缓存时返回 0，不能当成真实 0℃。分析假定 50 个深度层且含硬编码海洋深度表，time/depth/data 维度必须与资产对照，不宣称适用于任意深度轴。

当前 bind 注册匿名 click/dblclick 处理器，destroy 没有 off；重复 bind 会累加监听。等待纹理的轮询也没有超时/取消。一次绑定随整个 Map 生命周期结束仍需避免卸载中的分析回调；对需要重复挂载或中止任务的生产功能，应先修复该类的监听和等待取消机制，或在应用中自行管理命名事件与真实数据采样。不要推荐用 off("click") 清除所有地图监听，因为这会影响其他业务。

该类不是 addSceneComponent 入口。当前没有独立公开 demo 对主流程做验收；本参考仅记录源码证据和边界，不能用语法通过作为分析准确性证明。
