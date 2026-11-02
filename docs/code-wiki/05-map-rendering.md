# 05 地图渲染层 `@geolibre/map`

`packages/map` 是渲染内核与渲染器中立的地图抽象层。它既不包含全局 store 的业务动作，也不包含应用面板，而是提供：

- `MapEngine` 统一接口与四个实现
- React 画布组件（主地图、分屏、Cesium、Mapbox、ArcGIS）
- 由 `GeoLibreLayer[]` 驱动的图层同步与样式编译
- 识别、选择、弹窗、标记、截图、CRS、地形等地图能力
- 可独立复用的 headless 入口

包入口：`packages/map/src/index.ts`。子入口（`package.json` 的 `exports`）包括：

| 子路径 | 文件 | 用途 |
| --- | --- | --- |
| `@geolibre/map` | `src/index.ts` | 完整 React/引擎入口 |
| `@geolibre/map/headless` | `src/headless.ts` | 无 React、无 store 的图层同步 |
| `@geolibre/map/derived-geometry` | `src/derived-geometry.ts` | 几何生成器（质心、包围盒、凸包、缓冲区） |
| `@geolibre/map/pmtiles-layer` | `src/pmtiles-layer.ts` | PMTiles 图层 id/元数据辅助 |
| `@geolibre/map/style-import` / `style-export` | `src/style-import.ts` / `mapbox-style-export.ts` | Mapbox 样式导入/导出 |
| `@geolibre/map/style-layer-ids` | `src/style-layer-ids.ts` | 样式图层 id |
| `@geolibre/map/tile-request-headers` | `src/tile-request-headers.ts` | 图层请求头 |
| `@geolibre/map/layer-labels` | `src/layer-labels.ts` | 标注辅助 |
| `@geolibre/map/map-capture` | `src/map-capture.ts` | 画布捕获 |
| `@geolibre/map/arcgis-sdk` | `src/arcgis-sdk.ts` | ArcGIS SDK 加载 |
| `@geolibre/map/zarr-source` | `src/zarr-source.ts` | Zarr store 解析 |
| `@geolibre/map/epsg-proj4` | `src/epsg-proj4.ts` | EPSG 与 proj4 定义 |
| `@geolibre/map/arcgis-control-adapters` | `src/arcgis-control-adapters.ts` | 插件控件适配 |

## 1. `MapEngine` 接口

定义于 `packages/map/src/map-engine.ts:45`。它是所有面板、菜单和插件访问地图的唯一抽象面。

### 1.1 生命周期与相机

| 成员 | 说明 |
| --- | --- |
| `kind` | `"maplibre" \| "mapbox" \| "cesium" \| "arcgis"`，仅诊断/遥测 |
| `capabilities` | 只读能力对象，UI 门控依据 |
| `destroy()` | 释放 GPU 资源，可重复调用 |
| `applyView(view)` | 无动画设置相机；异步引擎返回 Promise |
| `easeToView(view)` | 短缓动 |
| `readView()` | 返回引擎中立的 `MapViewState` |
| `readCameraAltitude()` | 相机离地高度（米），无法计算返回 `null` |
| `flyTo(camera)` | 部分相机字段动画 |
| `flyToView(location)` / `applyStoryChapterCamera(...)` / `stopStoryCamera()` | 故事地图相机 |
| `zoomIn` / `zoomOut` / `resetNorth` / `resetNorthPitch` / `resetPitch` | 标准相机操作 |
| `fitBounds(bounds)` / `fitLayer(layer)` | 缩放到范围/图层 |
| `readProjection()` / `applyMapPreferences(preferences)` | 投影与约束 |
| `readCameraAltitude`、`isCameraMoving`、`onCameraMove`、`onCameraIdle` | 状态与事件 |

### 1.2 图层

```ts
syncLayers(layers: GeoLibreLayer[]): void
waitAndSyncLayers(layers: GeoLibreLayer[]): void
getLayerGeoJson(layerId: string): Promise<FeatureCollection | null>
getLayerRasterSource(layerId: string): Record<string, unknown> | null
```

`syncLayers` 接受完整数组（自底向上），负责新增、更新、删除与重排。

### 1.3 底图 / 故事 / 拾取

| 成员 | 说明 |
| --- | --- |
| `setBasemapVisible` / `setBasemapOpacity` / `setStyle(url)` / `setBlankBackgroundColor` | 底图控制 |
| `getBasemapStyleLayerIds()` | 区分底图样式层与项目图层 |
| `setStoryLayerOpacity(layerId, opacity, durationMs)` / `restoreLayerStyles()` | 故事播放的临时淡入淡出，不修改存储透明度 |
| `identifyFeatures(lngLat, layerId?)` | 返回 `IdentifiedFeature[]` |
| `highlightFeature(layer, featureId, options?)` / `clearFeatureHighlight()` | 高亮 |
| `showSearchResult(geometry)` | 临时搜索结果标记（返回清理函数） |
| `startManualPlacement(lngLat, options)` | 可拖动图钉 |
| `drawExtent(options)` | 两角范围绘制 |
| `getViewBounds()` / `getMapCrs?()` / `showExtent(extent)` | 范围与 CRS |
| `getRenderSurface()` / `getRenderStatus()` / `captureImage()` | 截图与渲染状态 |
| `onMapClick` / `suspendNavigation` / `stopCamera` | 交互控制 |
| `addControl` / `removeControl` / `setBuiltInControlVisible` / `setBuiltInControlPosition` | 控件宿主 |
| `isTerrainEnabled` / `setTerrainEnabled` / `setTerrainExaggeration` / `setTerrainCogSource` / `setTerrainIonAssetId` | 地形 |
| `getMap()` | 返回底层 MapLibre map（非 MapLibre 为 `null`） |

### 1.4 能力对象

`MapEngineCapabilities` 是 UI 判断"这个引擎能否做 X"的唯一依据。每个 flag 都有实际消费者：

| Flag | 消费者示例 |
| --- | --- |
| `styleSpec` | Style Panel 的颜色/paint 编辑、样式导入导出 |
| `nativeMapInstance` | AI 目标检测、Segmentation、直接操作 MapLibre 的插件 |
| `customLayers` | deck.gl 叠层、MapLibre CustomLayerInterface |
| `deckOverlay` | Deck.gl Viz、glTF、DuckDB 查询结果、LiDAR |
| `terrain` | 地形菜单 |
| `picking` | Identify |
| `onMapDrawing` | 手动放置、栅格子集框 |
| `domControls` | 插件控件挂载 |
| `screenOverlays` | 面板自己的 SVG 范围框 |
| `flatProjection` | globe-only 功能判断 `preferences.map.projection` |
| `terrainSource` | Terrain source 控件 |
| `ionTerrain` | Cesium Ion 地形字段 |
| `nativeZarr` | 引擎自绘 Zarr，绕过 `maplibre-gl-zarr` 控件 |
| `nativeDataSources` | KML/KMZ、CZML、Ion 资产原生加载 |
| `deferredEngineReady` | 首次加载完成后才发布引擎 |
| `measureTool` | 内置测量控件 |
| `controlLayerPanels` | 栅格/PMTiles 面板控件 |
| `supportedLayerKinds` | 每种图层的渲染方式 |

`MAPLIBRE_CAPABILITIES` 是 `Object.freeze` 的常量（防止外部插件写入），`rendererCapabilities(name, projection)` 用于引擎尚未挂载时按名称取能力。

## 2. 四个引擎实现

### 2.1 MapLibre — `MapController`

- 文件：`packages/map/src/map-controller.ts:229`
- 工厂：`createMapController()`（`map-controller.ts:3102`）
- 特点：默认引擎，能力最全；`getMap()` 返回真实 `maplibregl.Map`
- 负责：
  - `init(container, options)` 创建 Map、控件、地形源、CRS
  - `syncLayers` 的 diff 与 `syncLayer`
  - 相机 store 回声抑制（`markStoreEcho` / `applyStoreView`）
  - 视图约束（`constrainMapView`、`mapBoundsForPreferences`）
  - 内置控件：导航、全屏、指南针、定位、地球、地形、比例尺、attribution、logo、maptoolkit logo、图层控制
  - 底图透明度、空白底图背景色、样式代际（`styleGeneration`）
  - CRS 切换与 `crsGeneration`

### 2.2 Mapbox — `MapboxEngine`

- 文件：`packages/map/src/mapbox-engine.ts:188`
- 与 MapLibre 共享 Style Spec 表面，但：
  - 使用 Mapbox GL JS 的样式与 token
  - 通过 `@deck.gl/mapbox` 的 `MapboxOverlay` 承载 deck.gl
  - 不暴露 MapLibre map（`getMap()` 返回 `null`）
  - 支持 Mapbox Standard Style 与图层导入（`mapbox-standard-style.ts`、`mapbox-style-import.ts`）
  - 样式导入/导出路径独立：`mapbox-style-export.ts`

### 2.3 Cesium — `CesiumEngine`

- 文件：`packages/map/src/cesium-engine.ts:303`
- 能力：`styleSpec: false`、`nativeMapInstance: false`、`nativeZarr: true`、`nativeDataSources: true`、`ionTerrain: true`、`deferredEngineReady: true`
- 范围：
  - 通过 `basemapToCesiumImagery` + `applyBasemapImagery` 绘制底图
  - `CesiumLayerSync` 把 store 图层映射为 `ImageryLayer`、`GeoJsonDataSource`、`Cesium3DTileset`、point primitive、CZML、KML 等
  - `CesiumPointCloudStreamer` 按视图流式加载 COPC/EPT
  - `MapLibreDrape` 让瓦片矢量图层通过隐藏的单瓦片 MapLibre 渲染后贴到地球
  - `ProtocolImageryProvider` 复用 MapLibre 自定义协议
  - `CesiumControlHost` 让插件控件在无 MapLibre 时仍可绘制
  - 提供 `getCesiumScene()` 给环境类插件（Sun、Effects、Flight Simulator）
- 相机换算在 `cesium-camera.ts`：以**地面分辨率**（米/像素）对齐 MapLibre 的 zoom/pitch/bearing 与 Cesium 的 range/pitch/heading。

### 2.4 ArcGIS — `ArcgisEngine`

- 文件：`packages/map/src/arcgis-engine.ts:577`
- 能力：依赖 `arcgis-sdk.ts` 动态加载 SDK；2D `MapView` 可承载 deck.gl 叠层，3D `SceneView` 不可
- 范围：
  - FeatureServer / MapServer / VectorTileServer / ImageServer 图层
  - 服务端 identify、附件、关联记录、域、 contingent values、编辑写回
  - `arcgis-control-host.ts` 为插件控件提供录制式地图 façade
  - `arcgis-cog-imagery.ts`、`arcgis-cog-terrain.ts`、`arcgis-zarr.ts` 支持 COG/Zarr
  - `arcgis-reprojected-tiles.ts`、`arcgis-template-tiles.ts` 处理瓦片投影与模板

### 2.5 引擎选择与画布

| 组件 | 文件 | 何时使用 |
| --- | --- | --- |
| `MapCanvas` | `MapCanvas.tsx` | MapLibre 主地图/副窗格 |
| `SecondaryMapCanvas` | `SecondaryMapCanvas.tsx` | MapLibre 分屏窗格 |
| `PrimaryMapboxCanvas` | app 的 `components/layout/PrimaryMapboxCanvas.tsx` | Mapbox 主地图 |
| `PrimaryCesiumCanvas` / `CesiumCanvas` | app / `CesiumCanvas.tsx` | Cesium 主地图 / 副窗格 |
| `PrimaryArcgisCanvas` / `ArcgisCanvas` | app / `ArcgisCanvas.tsx` | ArcGIS 主地图 / 副窗格 |
| `MapGrid` | app 的 `components/layout/MapGrid.tsx` | 1×1 直接渲染主地图；更大网格渲染副窗格 |

每个副窗格通过 `viewId` 区分角色：有 `viewId` 时读写 `secondaryMapViews` 记录（自有相机与可见性覆盖）；无 `viewId` 时代表主地图，直接读写共享 `mapView`。

## 3. 图层同步管线

### 3.1 入口

```text
useAppStore.layers 变更
  → 画布订阅
    → MapEngine.syncLayers(layers)
      → MapController.syncLayers（diff）
        → syncLayer(map, layer, beforeId)   // layer-sync.ts:481
```

`MapCanvas` 同时处理识别、弹窗、选择、照片、栅格识别等交互，但不在渲染路径中直接构建图层。

### 3.2 `syncLayer` 分派

`syncLayer` 首先排除两类记录：

1. **外部原生图层**（`isExternalNativeLayer`）：插件已经通过 `registerExternalNativeLayer` 注册了 MapLibre 图层，这里只做镜像。
2. **占位图层**（`isPlaceholderLayer`）：由插件稍后填充。

然后 `classifyLayer(layer)` 得到 `LayerKind`，按 `MAPLIBRE_SUPPORTED_LAYER_KINDS` 分派：

| Kind | MapLibre 处理 |
| --- | --- |
| `geojson` | 内联 GeoJSON 源；超过 **50,000** 要素（`LARGE_VECTOR_FEATURE_THRESHOLD`）时切到 geojson-vt 客户端瓦片 |
| `raster-tiles` | XYZ/WMS/WMTS 栅格源 |
| `vector-tiles` | 矢量瓦片源（非 Web Mercator CRS 下跳过） |
| `tile-archive` | MBTiles 由本地协议读取；PMTiles 由插件控件 |
| `video` / `image` | 视频/图片叠加 |
| `arcgis` / `zarr` / `lidar` / `gaussian-splat` / `3d-tiles` / `cog` / `vector-file` / `duckdb-query` / `deckgl-viz` | 由对应插件控件绘制（"plugin"） |

> 注：`LayerKind` 是比 `LayerType` 更粗的分类（多个类型可映射到同一 kind），见 `layer-kind.ts`。

### 3.3 GeoJSON 两条路径

| 路径 | 触发 | 实现 |
| --- | --- | --- |
| 内联 source | `featureCount <= 50,000` 且非自定义 CRS | `syncGeoJsonLayer`：`map.addSource(id, { type: "geojson", data })` + 渲染层 |
| geojson-vt 瓦片 | 超过阈值，或需要大量点 | `syncGeoJsonVtLayer`：`geojson-vt` worker 建索引，自定义协议提供 `{z}/{x}/{y}` 瓦片，源类型为 `vector` |

两条路径复用相同的图层 id 命名，便于样式/可见性/过滤统一更新。3D Z 值图层（`elevation3dEnabled` 且数据确有 Z）交给 deck.gl overlay 绘制，MapLibre 端移除，避免重复绘制。

### 3.4 样式编译

| 模块 | 作用 |
| --- | --- |
| `vector-style.ts` | 把 `LayerStyle` 编译为 MapLibre 图层 paint/layout（单值/分级/分类/规则/表达式） |
| `feature-style.ts` | `createFeatureStyleResolver(style)`（`feature-style.ts:175`）在非 MapLibre 引擎（尤其 Cesium）逐要素复现相同样式 |
| `gl-style-compiler.ts` | 把 Mapbox 样式图层编译为可跨引擎使用的计划（`compileMapboxLayer`） |
| `label-style.ts` / `layer-labels.ts` | 标注文本、去重、数值格式 |
| `fill-patterns.ts` | 图案填充 sprite |
| `line-decorations.ts` | 箭头/三角/圆点等线装饰 |
| `derived-geometry.ts` | 几何生成器（质心、包围盒、凸包、缓冲） |
| `diagram.ts`（core） + deck.gl overlay | 饼图/柱状图符号 |
| `layer-blend-modes.ts` | 混合模式 |

### 3.5 过滤与可见性

同步时按以下顺序组合过滤：

1. 图层自身的 `filterExpression`
2. `quickFilters` 编译结果（`compileQuickFilters`）
3. 规则可见性（`ruleBasedVisibilityFilter`）
4. 时间滑块 `timeFilter`
5. iframe `embedFilter`
6. 要素选择/高亮（独立高亮源，不改原图层）

图层可见性还要叠加：图层 `visible`、所属图层组链的可见性、当前窗格的可见性覆盖（`secondaryMapViews`）。

### 3.6 排序

`syncLayers` 接受自底向上的数组。新增图层默认落在顶部；`moveLayer` / `moveLayersRelative` 会改变数组顺序，同步时从"第一个位置变化的图层"开始重建尾栈，避免 MapLibre 保留旧位置。排序锚点还可由 `beforeId` 与样式图层 id 计算（`externalExtrusionLayerId`、`mbtiles*`、`vectorTile*` 等辅助函数）。

## 4. 识别、选择与弹窗

| 能力 | 文件 | 说明 |
| --- | --- | --- |
| 单图层识别 | `feature-pick-index.ts`、`identify-sources.ts` | 查询 MapLibre 要素、WMS GetFeatureInfo、DuckDB、像素识别 |
| 全图层识别 | `identify-all.ts`、`identify-all-popup.ts` | 分组弹窗、去重、栅格像素 |
| 要素选择 | `map-feature-selection.ts`、`feature-selection.ts` | 点选/框选/多边形选择，Shift/Ctrl 修饰键 |
| 高亮 | `map-selection.ts`、`feature-selection.ts` | 独立高亮 source，不改变图层数据 |
| 弹窗 | `feature-popup.ts`、`photo-popup.ts`、`popup`（core 类型） | 属性卡片、照片预览、最大宽度与自定义字段 |
| 编辑动作 | `identify-edit-actions.ts` | 识别结果上的"编辑几何/编辑属性"按钮 |
| 搜索 | `gl-search-result.ts` | 搜索结果标记与清理 |
| ArcGIS 识别 | `arcgis-identify.ts` | 服务端 identify（避免客户端几何缺失） |

## 5. 底图、地形、CRS、捕获

### 5.1 底图

- `basemap-style.ts`：解析 `basemapStyleUrl`，处理 `geolibre://` 哨兵样式（行星、离线、区域底图）与空白底图。
- `layer-control-host.ts`：MapLibre/Mapbox 共享的图层控件宿主，负责底图行、图层行、位置与回调。
- `mapbox-style.ts` / `mapbox-standard-style.ts`：Mapbox 专有样式。
- Cesium 端：`cesium-basemap.ts` + core 的 `basemapToCesiumImagery()`。

### 5.2 地形

- MapLibre：`terrain-control.ts`、`cog-dem-source.ts`（COG 作为 DEM）、`TERRAIN_SETTINGS_EVENT`。
- Cesium：`cesium-terrarium.ts` 读取 Terrarium 编码瓦片、`cesium-cog-imagery.ts`、Ion terrain asset。
- ArcGIS：`arcgis-cog-terrain.ts`。

### 5.3 CRS

`map-crs.ts` / `active-map-crs.ts` / `epsg-proj4.ts`：

- 只有 MapLibre 支持非 Web Mercator 的自定义 CRS；其他引擎返回 `null`。
- 切换 CRS 会重建样式（`createMapCrsStyle`）并注册投影。
- 地图 CRS 变化触发 `MAP_CRS_CHANGE_EVENT`（`@geolibre/core`），让面板同步范围框。

### 5.4 捕获与打印

| 文件 | 作用 |
| --- | --- |
| `map-capture.ts` | `getRenderSurface`、`captureImage`、`imageBlobToDataUrl` |
| `map-resize.ts` | ResizeObserver 调度与面板缩放事件 |
| `globe-fit-bounds.ts` | 地球模式下的 fitBounds 修正 |
| `map-diagnostic.ts` | 图层加载失败/瓦片错误诊断事件 |

截图支持不依赖 MapLibre 的渲染色（`MapRenderSurface`），供 Print Layout、视频录制、分享缩略图使用。

## 6. Headless 入口

`@geolibre/map/headless` 面向非 GeoLibre 宿主或脚本：

```ts
import { createLayerSync, installMapTransformCompat } from "@geolibre/map/headless";

const sync = createLayerSync(map);
sync.sync(layers);   // 自底向上
sync.dispose();      // 移除所有本 sync 添加的图层
```

特点：

- 不引入 React、Zustand、Cesium、控件。
- 使用与完整应用相同的 `syncLayer()`，但只重排受管图层，不计算样式锚点。
- 在创建 `Map` 前安装 `map.transform` 兼容层，满足 `@deck.gl/mapbox` 等第三方库。

## 7. 测试与扩展检查

关键测试：

- `tests/layer-support-matrix.test.ts`：校验四个引擎的 `supportedLayerKinds` 与 `classifyLayer`。
- `tests/cesium-*.test.ts`：相机、图层同步、点云、瓦片样式。
- `tests/arcgis-*.test.ts`：服务图层、编辑、识别。
- `tests/mapbox-*.test.ts`：样式、图层、overlay。
- `tests/feature-style.test.ts`：跨引擎样式一致性。

新增一个引擎能力或图层 kind 时：

1. 更新 `MapEngine` / `MapEngineCapabilities`（若需要新 flag）。
2. 更新 `layer-kind.ts` 与四个 `*_SUPPORTED_LAYER_KINDS` 表。
3. 在对应引擎的 sync 中实现，或标记 `"plugin"` 并让插件负责。
4. 更新 `rendererCapabilities` 与 UI 门控（用能力，不用引擎名）。
5. 若跨引擎共享逻辑，放入中立模块，不要相互 import。
6. 补充 `layer-support-matrix` 与引擎测试。

下一章：[06 UI 与地理处理](06-ui-and-processing.md)。
