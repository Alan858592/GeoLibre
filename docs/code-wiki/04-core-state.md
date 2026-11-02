# 04 核心状态层 `@geolibre/core`

`packages/core` 是 GeoLibre 的领域内核：它定义所有跨模块共享的类型、项目文件格式、Zustand 全局 store，以及大量无副作用的纯函数工具。除 `react`/`zustand`/`zundo` 外，它不依赖任何 UI 或地图 SDK，因此可被 Web、桌面、Worker、Python 嵌入与测试复用。

- 包入口：`packages/core/src/index.ts`（barrel 导出）
- 包版本：`3.5.2`
- 运行依赖：`zustand`、`zundo`、`uuid`、`@maplibre/maplibre-gl-style-spec`
- 子入口：`@geolibre/core/worker-safe`（`src/worker-safe.ts`，供 Web Worker 使用、避免引入 React/DOM）

## 1. 类型系统

### 1.1 图层类型 `LayerType`

`types.ts:65` 的 `LAYER_TYPES` 是运行时数组，`LayerType` 由它派生，二者不会漂移：

```ts
geojson, raster, wms, wmts, xyz, vector-tiles, arcgis, pmtiles, mbtiles,
zarr, lidar, gaussian-splat, 3d-tiles, cog, flatgeobuf, geoparquet,
duckdb-query, deckgl-viz, video, image
```

共 20 种。每种图层由 `source`（SDK 无关的源描述）、`metadata`（引擎/插件私有元数据）、`style`（统一符号化）描述。

### 1.2 `GeoLibreLayer`

核心字段（`types.ts:1284`）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `string` | 稳定唯一 id，用作 MapLibre source/layer id 前缀 |
| `name` | `string` | 面板显示名 |
| `type` | `LayerType` | 20 种之一 |
| `source` | `Record<string, unknown>` | 引擎读取的源配置（URL、tiles、坐标、查询等） |
| `visible` | `boolean` | 可见性 |
| `opacity` | `number` | 图层整体透明度 |
| `style` | `LayerStyle` | 符号化（颜色、线宽、标注、挤出、分类…） |
| `metadata` | `Record<string, unknown>` | 引擎/插件私有状态（如 `rasterState`、`timeBinding`） |
| `geojson` | `FeatureCollection?` | 内联矢量数据 |
| `geoarrow` | `GeoArrowBatch?` | 可选的列式二进制表示，提升渲染与 Worker 分块性能 |
| `beforeId` | `string?` | 排序锚点（旧字段，面板排序以数组顺序为主） |
| `groupId` | `string?` | 所属图层组 |
| `sourcePath` | `string?` | 本地文件路径（桌面保存用） |
| `filterExpression` | `unknown[]?` | 持久化的 MapLibre 布尔过滤表达式 |
| `quickFilters` | `LayerQuickFilter[]?` | 面板化的快速过滤控件状态（同步时编译） |
| `timeFilter` / `embedFilter` | `unknown[]?` | 临时过滤（时间滑块 / iframe API），不持久化 |
| `capabilities` | `LayerCapabilities?` | 查询/创建/更新/删除/导出能力 |
| `editorTracking` | `EditorTrackingConfig?` | 编辑者追踪 |
| `fieldVisibility` | `Record<string, FieldVisibility>?` | 字段级可见性；导出/分享时物理移除 |
| `attributeForm` | `AttributeFormConfig?` | 属性表单设计器配置 |
| `popup` | `LayerPopupConfig?` | 弹窗/悬停卡片设计 |
| `descriptiveMetadata` | `LayerDescriptiveMetadata?` | 目录元数据（标题、摘要、许可、来源） |
| `joins` | `LayerJoin[]?` | 持久化属性连接（连接列物化进 `geojson`） |
| `virtualFields` | `LayerVirtualField[]?` | 表达式虚拟字段，在连接之后计算 |
| `connection` | `LayerConnection?` | 可刷新图层的连接策略 |

### 1.3 `LayerStyle`

`LayerStyle`（`types.ts:428`）是一份跨引擎的符号化描述，主要字段分组：

| 分组 | 字段 |
| --- | --- |
| 可见范围 | `minZoom`、`maxZoom` |
| 基础符号 | `fillColor`、`strokeColor`、`strokeWidth`、`strokeWidthUnit`、`fillOpacity`、`circleRadius` |
| 文本 | `textColor`、`textHaloColor`、`textHaloWidth`、`textSize`、`labels: LabelStyle` |
| 3D 挤出 | `extrusionEnabled`、`extrusionColor`、`extrusionHeightProperty`、`extrusionHeightScale`、`extrusionBase`、表达式版字段 |
| 真 3D | `elevation3dEnabled`、`elevation3dVerticalScale`、`elevation3dOffset` |
| 矢量配色方案 | `vectorStyleMode`（`single` / `graduated` / `categorized` / `rule-based` / `expression`）、`vectorStyleStops`、`vectorRules`、`vectorStyleExpression` |
| 比例符号 | `proportionalSizeEnabled` 及 Min/Max 值/半径 |
| 点渲染 | `pointRenderer`（`single` / `heatmap` / `cluster`）、`markerShape`、`markerColor`、`cluster*` |
| 填充图案 | `fillPattern`、`fillPatternSvg` |
| 线装饰 | `lineDecoration`、`lineDecoration*` |
| 几何生成器 | `geometryGenerator` 及类型/参数 |
| 图表 | `diagramType`、`diagramFields`、`diagramSize*` |
| 混合 | `blendMode` |

`LabelStyle`（`types.ts:264`）覆盖字段/表达式两种取值、字号、光晕、缩放范围、锚点/偏移/旋转、去重策略（`off` / `unique` / `concatenate`）、数字格式化、以及数据定义的 size/color/opacity/visibility/priority 表达式。

### 1.4 视图与偏好

```ts
interface MapViewState {
  center: [number, number];
  zoom: number;
  bearing: number;
  pitch: number;
  bbox?: [number, number, number, number];
}
```

`MapGridLayout`（`rows`、`cols`、`syncView`）与 `SecondaryMapView`（每个窗格的相机、引擎 `viewKind`、图层可见性覆盖）支撑分屏。

`ProjectPreferences` = `{ map, environmentVariables, geocoding }`。`DEFAULT_PROJECT_PREFERENCES` 定义默认缩略图范围、最小/最大 zoom、最大 pitch、投影（默认 `"globe"`）、椭球、比例尺单位、地理编码配置等。

### 1.5 项目文件类型 `GeoLibreProject`

`types.ts:2531` 与 `docs/project-format.md` 对应，关键字段包括：

`id`、`version`、`name`、`mapView`、`basemapStyleUrl`、`basemapVisible`、`basemapOpacity`、`blankBackgroundColor`、`layers`、`selectedLayerId`、`layerGroups`、`styles`（layer id → LayerStyle）、`preferences`、`plugins`、`legend`、`printLayout`、`storymap`、`models`、`processingHistory`、`obia`、`widgets`、`dashboardColumns`、`mapLayout`、`secondaryMapViews`、`primaryMapLabel`、`bookmarks`、`bookmarkGroups`、`styleLibrary`、`primaryRenderer`、`comments`、`interaction`、`metadata`。

格式版本 `PROJECT_VERSION = "0.2.0"`。旧版本（`0.1.0`）与 Python 包写出的文件仍可读取，`parseProject` 会做大量 normalize。

## 2. 全局 Store

### 2.1 创建

```ts
export const useAppStore = create<AppState>()(
  temporal(
    (set, get) => ({
      ...createProjectSlice(set, get),
      ...createMapViewSlice(set, get),
      ...createLayersSlice(set, get),
      ...createLayerGroupsSlice(set, get),
      ...createProjectContentSlice(set, get),
      ...createProcessingSlice(set, get),
      ...createLibrariesSlice(set, get),
      ...createSessionSlice(set, get),
      ...createCollaborationSlice(set, get),
      ...createCapabilitiesSlice(set, get),
      ...createUiSlice(set, get),
    }),
    createHistoryOptions(),
  ),
);
bindHistoryStore(useAppStore);
```

（`packages/core/src/store.ts:61`）

`temporal` 来自 `zundo`，为 store 增加 undo/redo。`bindHistoryStore` 让切片可以调用 `clearHistory` 而不产生循环依赖。

同时 store 订阅 `preferences.map.ellipsoidId`，同步到测量模块的模块级单例 `setActiveEllipsoidId`。

### 2.2 切片与关键动作

| 切片 | 关键状态 | 关键动作 |
| --- | --- | --- |
| `project` | `projectName`、`projectPath`、`isDirty`、`projectId` | `newProject`、`loadProject`、`saveProject`、`setProjectName` |
| `map-view` | `mapView`、`basemapStyleUrl`、`basemapVisible`、`basemapOpacity`、`mapLayout`、`secondaryMapViews`、`primaryRenderer`、`preferences` | `setMapView`、`setMapGrid`、`setSecondaryMapView`、`setPrimaryRenderer`、`setBasemapStyleUrl`、`setPreferences`、`applyPlanetaryBasemap` |
| `layers` | `layers`、`selectedLayerId`、样式剪贴板 | `addLayer`、`addGeoJsonLayer`、`addImageOverlayLayer`、`addTileLayer`、`removeLayer`、`updateLayer`、`setLayerVisibility`、`setLayerOpacity`、`setLayerStyle`、`reorderLayer`、`moveLayer`、`moveLayersRelative`、`setLayerFilterExpression`、`setLayerJoins`、`setLayerVirtualFields`、`copyLayerStyle`、`pasteLayerStyle` |
| `layer-groups` | `layerGroups` | 创建/重命名/删除组、移动组、组可见性与透明度 |
| `project-content` | `comments`、`widgets`、`storymap`、`printLayout`、`bookmarks`、`bookmarkGroups` | 评论增删/解决、Dashboard 部件、故事地图章节、打印块、书签 |
| `processing` | `models`、`processingHistory`、`obia` | 保存/重命名/删除模型、记录运行、OBIA 状态 |
| `libraries` | `projectStyleLibrary`、`layerLibrary`、`templateLibrary` | 样式/图层/模板库读写 |
| `session` | `selectedLayerId`、`identifyLayerId`、`identifyAll*`、`pointerCoords`、GPS 读数 | `selectLayer`、识别状态、指针坐标 |
| `collaboration` | `collaboration`（会话、参与者、模式、聊天） | `setCollaboration`、参与者更新 |
| `capabilities` | `deploymentCapabilities` | `setDeploymentCapabilities` |
| `ui` | 所有对话框/面板开关、处理对话框类型 | `openDialog`、`closeDialog`、`setProcessingDialog` |

`addGeoJsonLayer` 的签名（`store/layers-slice.ts:484`）：

```ts
addGeoJsonLayer(
  name: string,
  geojson: FeatureCollection,
  sourcePath?: string,
  beforeLayerId?: string | null,
  extra?: Partial<GeoLibreLayer>,
): string
```

返回新图层 id。所有 Add Data 路径最终都汇聚到这里（或 `addLayer` / `addTileLayer` / `addImageOverlayLayer`）。

`removeLayer`（`layers-slice.ts:260`）不仅从数组删除，还会清理：

- 图层组、SelectedLayerId
- 评论/图例/图例条目/Dashboard 部件中的引用
- 时间滑块绑定、协作层、Print Layout 图层引用
- 样式库中对图层的依赖

### 2.3 撤销 / 重做

`store/undo-history.ts`：

- `partializeHistory(state)`（`undo-history.ts:151`）只挑 `layers`、`layerGroups`、底图字段、`storymap`、`comments`、`widgets`、`legend`。
- `createHistoryOptions()` 提供 `limit`、`equality`、`handleSet` 等；`layers` 用逐元素 `Object.is` 比较，避免空数组重置记录步骤。
- 导出 `undo`、`redo`、`clearHistory`、`withoutHistory`、`registerProjectRestoreHistory`、`canUndoProjectRestore`、`canRedoProjectRestore`。
- `finishHistoryStep` 在每步完成后清理 SecondaryMapViews 和 Print Layout 中对已删除图层的引用。

### 2.4 选择器与性能

`layer-selectors.ts`：

```ts
useLayer(id)            // 单图层
useLayerIds()           // id 列表
useLayerSummaries()     // 轻量行
useLayersWhen(open)     // 面板打开时才订阅完整数组
NO_LAYERS               // 冻结空数组
```

`useAppCapability(privilege)`（`store.ts:114`）返回 `{ granted, reason }`，用于部署权限门控。

## 3. 项目生命周期

### 3.1 创建空项目

`createEmptyProject(options)`（`project.ts:113`）返回 `GeoLibreProject`，默认：

- `version: PROJECT_VERSION`
- 默认 `mapView`（`createDefaultMapView()`）
- 默认底图 `DEFAULT_BASEMAP`（OpenFreeMap Liberty）
- 空 `layers` / `layerGroups` / `styles`
- `DEFAULT_PROJECT_PREFERENCES`

### 3.2 序列化

| 函数 | 位置 | 说明 |
| --- | --- | --- |
| `serializeProject(project)` | `project.ts:277` | 基本 JSON 序列化 |
| `serializeProjectWithLayerCache(project, cache)` | `project.ts:324` | 复用图层序列化结果，减少大项目保存开销 |
| `serializeProjectWithLayerCacheAsync(...)` | `project.ts:364` | 大图层异步序列化，避免阻塞主线程 |
| `projectFromStore(state)` | `project.ts:2045` | 从 store 快照构造项目；过滤 session-only 图层；把 `layer.style` 收集到 `styles`；调用各 `normalize*` |

保存时会排除：

- `isSessionOnlyLayer` 为真的图层（例如运行时插件临时层）
- 空的 `legend` / `printLayout` / `storymap` / `models` / `comments` 等，保持默认项目的字节形状稳定
- 凭据字段（由桌面端 `splitProjectCredentials` 抽出到钥匙串）

### 3.3 解析

`parseProject(json)`（`project.ts:412`）是容错入口：

1. `JSON.parse` 失败抛错。
2. 校验 `version`（未知版本按兼容处理）。
3. `normalizeMapViewState`、`normalizeMapLayout`、`normalizeSecondaryMapViews`、`normalizePrimaryRenderer`、`normalizeLayerGroups`、`normalizeLegendConfig`、`normalizeStoryMap`、`normalizeModels`、`normalizeWidgets`、`normalizeBookmarks`、`normalizeProjectComments`、`normalizeProjectInteraction`。
4. 补全图层默认样式与缺失字段。
5. 过滤非法/未知图层类型与 URL（如插件清单只允许 https 白名单）。

### 3.4 应用到 Store

`applyProjectToStore(project)`（`project.ts:2457`）返回一个结果对象（可能包含警告），负责：

- `set` 项目身份与所有内容切片
- 恢复 `mapView`、底图、网格、副窗格、`primaryRenderer`
- 恢复插件状态（由 `usePlugins` 的 `restoreProjectState` 消费）
- 恢复处理模型、运行历史
- 触发 `clearHistory` 或历史恢复标记
- 处理凭据引用（`resolveProjectHeaderReferences`）

`detachProjectCopy`（`project.ts:2595`）用于"另存为副本"时重置 id/路径。

### 3.5 差异与恢复

- `project-diff.ts`：计算项目差异，供 Project History / 快照 diff 使用。
- `store/project-slice.ts` 维护最近项目、自动保存状态、恢复对话框。
- `layers-file.ts` / `layer-style-file.ts`：图层与样式单独导入导出。

## 4. 纯函数工具簇

### 4.1 样式与表达式

| 模块 | 关键导出 | 用途 |
| --- | --- | --- |
| `color-ramp.ts` | 色带解析/插值 | 分类、分级、栅格配色 |
| `vector-color.ts` | 颜色规范化/对比度 | 颜色选择器 |
| `expressions.ts` | MapLibre 表达式构造与校验 | 表达式编辑器、工具参数 |
| `style-layer-evaluator.ts` | `createFeatureStyleResolver` 的底层求值 | 在非 MapLibre 引擎复现样式 |
| `style-library.ts` | 样式库条目校验 | Style Manager |
| `layer-style-clipboard.ts` / `layer-style-file.ts` | 样式复制/粘贴/文件 | 跨图层复用 |
| `external-native-paint.ts` | 插件原生 paint 桥接 | 插件绘制的图层参与样式 |

### 4.2 数据与属性

| 模块 | 关键导出 | 用途 |
| --- | --- | --- |
| `joins.ts` | `LayerJoin` 类型与连接执行 | 属性连接 |
| `virtual-fields.ts` | 表达式虚拟字段 | 派生列 |
| `quick-filters.ts` / `layer-filters.ts` | 过滤控件模型与编译 | 图层过滤 |
| `attribute-form.ts` | 表单配置与校验 | 属性编辑 |
| `layer-descriptive-metadata.ts` | 元数据 schema | 图层 Metadata 对话框 |
| `visibility.ts` | `excludeHiddenFieldsFromGeojson/Project` | 导出时剔除隐藏字段 |
| `geojson-z.ts` | 保留 Z 值 | 3D 线/点 |
| `geoarrow.ts` | GeoArrow 批次 | 高性能矢量 |
| `label-number-format.ts` | 数字本地化 | 标注数字格式 |

### 4.3 引擎中立的地图描述

| 模块 | 关键导出 | 用途 |
| --- | --- | --- |
| `cesium-imagery.ts` | `basemapToCesiumImagery` | 把底图 URL 翻译成 Cesium 影像描述 |
| `cesium-ion.ts` | `createCesiumIonLayer`、`isCesiumOnlyLayer` | Ion 资产图层 |
| `cesium-kml.ts` | `createCesiumKmlLayer` | KML 图层描述 |
| `czml.ts` | `createCzmlLayer`、`parseCzml` | CZML |
| `three-d-tiles.ts` | 3D Tiles 请求头处理 | Google Photorealistic 等 |
| `ellipsoids.ts` | `setActiveEllipsoidId`、行星参数 | 月球/火星测量 |
| `regional-basemaps.ts` | 区域底图目录 | 中国大陆等 |
| `map-crs.ts` | CRS 解析 | 非 Web Mercator |
| `camera-altitude.ts` | 相机高度换算 | 状态栏/地形 |

### 4.4 平台与安全

| 模块 | 用途 |
| --- | --- |
| `runtime-env.ts` | `getRuntimeEnvironment`、`getCesiumIonToken`、`getMapboxAccessToken` 等；构建环境 + `window.__GEOLIBRE_RUNTIME_ENV__` 覆盖 |
| `credentials.ts` | 凭据字段识别与递归脱敏 |
| `project-credential-refs.ts` | 项目文件凭据 ↔ 设备存储的引用 |
| `header-references.ts` | `${ENV_VAR}` 请求头解析 |
| `deployment-capabilities.ts` | 部署权限枚举与校验 |
| `s3.ts` | S3 URL/签名辅助 |
| `paths.ts` / `platform.ts` | 路径与平台判定 |

## 5. 扩展 `@geolibre/core` 的检查清单

新增一个图层字段时，通常需要同步修改：

1. `types.ts` 的 `GeoLibreLayer` 与默认值（`layer-defaults.ts`）。
2. `project.ts` 的 `projectFromStore` / `parseProject` / `prepareLayerForSave`，决定是否持久化。
3. `store/undo-history.ts` 的 `partializeHistory`，决定是否进入撤销。
4. `layer-selectors.ts` 的 `LayerSummary`，如果面板需要展示。
5. `@geolibre/map` 的 `syncLayer` / 引擎适配器，决定如何渲染。
6. 项目格式文档 `docs/project-format.md`。
7. 对应测试（`tests/*.test.ts`，尤其是 `project-*.test.ts`、`undo-*.test.ts`）。

新增一个纯工具函数时，优先放在按职责命名的文件，并从 `index.ts` 导出；避免在 core 中引入 DOM、React 或地图 SDK。

下一章：[05 地图渲染层](05-map-rendering.md)。
