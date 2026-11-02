# 01 项目总览

## 1. 项目定位

GeoLibre 是一个"一次编写、多端运行"的云原生 GIS 平台。它把传统桌面 GIS 的图层管理、符号化、属性表、地理处理与空间 SQL，搬到了浏览器与 WebView 中，同时通过 Tauri 获得桌面级的文件系统与系统集成能力。

三条产品主线决定了代码形态：

1. **本地优先（local-first）**。矢量转换、空间 SQL、地理处理默认全部在浏览器内通过 WebAssembly 完成，数据不上传服务器。
2. **渲染引擎可替换**。项目状态与地图 SDK 解耦，同一个 `.geolibre` 项目可用 MapLibre、Mapbox、Cesium 或 ArcGIS 打开，也可以在同一窗口分屏混用。
3. **能力可选、渐进增强**。Python sidecar、Pyodide、CDN 引擎、协作中继、插件都是可选层；缺失时对应功能降级或隐藏，而不是整页失败。

## 2. 功能域一览

| 功能域 | 主要实现位置 | 说明 |
| --- | --- | --- |
| 项目与图层模型 | `packages/core` | `.geolibre` 项目文件、20 种图层类型、样式、分组、书签、故事地图 |
| 地图渲染 | `packages/map` | 四个 `MapEngine` 实现、图层同步、识别/选择、地形、CRS、截图 |
| 数据接入 | `apps/geolibre-desktop/src/lib`、`packages/plugins` | 文件拖放、Add Data 对话框、URL/服务、云存储、STAC、OGC、ArcGIS |
| 格式转换 | DuckDB-WASM Spatial、`shpjs`、`sql.js`、`fflate` | GeoParquet、Shapefile、GPKG、KML/KMZ、FlatGeobuf、GML、DXF 等 |
| 矢量化处理 | `packages/processing` | Turf.js 工具、Pyodide GeoPandas、sidecar GeoPandas 三种引擎 |
| 栅格处理 | `packages/processing` + sidecar | 栅格工具、COG 读写、光谱指数、地形分析 |
| Whitebox 工具 | `geolibre-wasm` + worker 池 | 1000+ WASM 工具，可选 sidecar 作为替代引擎 |
| 空间 SQL | `apps/geolibre-desktop/src/lib` + sidecar | DuckDB、PGlite/PostGIS、Apache Sedona（CereusDB/SedonaDB） |
| 插件系统 | `packages/plugins` + `usePlugins.ts` | 70 个内置插件、外部 zip/manifest 插件、插件市场 |
| 协作 | `packages/collab-core`、`workers/collab` | 基于 Durable Object / Node 的会话中继、快照、评论、聊天 |
| 分享与嵌入 | `backend/geolibre_server_api`、`@geolibre/embed` | 项目云端存储、OAuth、iframe 嵌入 API |
| 桌面/移动 | `apps/geolibre-desktop/src-tauri` | 文件、钥匙串、原生 HTTP、本地 MBTiles、Martin、Jupyter、sidecar |
| Jupyter | `python/src/geolibre` | anywidget 内嵌应用 + leafmap 风格 API |
| AI | `apps/geolibre-desktop/src/lib/assistant`、`workers/ai-proxy` | 应用内 AI 助手、MCP、Agent Skill |

## 3. 技术栈

### 3.1 前端与渲染

| 技术 | 角色 | 关键位置 |
| --- | --- | --- |
| React 19 + TypeScript | UI 框架 | `apps/geolibre-desktop/src` |
| Vite（Rolldown） | 开发服务器与构建 | `apps/geolibre-desktop/vite.config.ts` |
| Zustand + zundo | 全局状态与撤销/重做 | `packages/core/src/store.ts` |
| MapLibre GL JS | 默认 2D 渲染引擎 | `packages/map/src/map-controller.ts` |
| Mapbox GL JS | 可选 2D 引擎（Style Spec 兼容） | `packages/map/src/mapbox-engine.ts` |
| CesiumJS | 可选 3D 地球 | `packages/map/src/cesium-engine.ts` |
| ArcGIS Maps SDK for JS | 可选 2D/3D 引擎 | `packages/map/src/arcgis-engine.ts` |
| deck.gl | 栅格、点云、3D 可视化叠层 | `packages/plugins/src/plugins/deckgl-viz`、`shared-deck-overlay.ts` |
| Tailwind CSS v4 + Radix UI | 设计系统 | `packages/ui` |
| react-i18next | 国际化（21 个语言包） | `apps/geolibre-desktop/src/i18n/locales` |

### 3.2 浏览器内计算引擎

| 引擎 | 用途 | 加载方式 |
| --- | --- | --- |
| DuckDB-WASM + Spatial | 矢量格式转换、SQL Workspace | 默认打包；`npm run lite:build` 可切到 jsDelivr |
| PGlite + PostGIS | 浏览器内 PostGIS SQL | 默认 jsDelivr CDN，首次使用后由 Service Worker 缓存 |
| CereusDB（SedonaDB WASM） | 浏览器内 Apache Sedona SQL | 动态 import，独立 chunk |
| Pyodide | 浏览器内 CPython + GeoPandas/Shapely | 首次使用时从 CDN 下载 |
| `geolibre-wasm` | Whitebox Next Gen Rust → WASI | Web Worker 池 |
| `cog-tiler-wasm` / `geotiff` / `zarr-cesium` / `copc` / `laz-perf` | 栅格、Zarr、点云解码 | 按需加载 |
| `gdal3.js` | 客户端 GeoTIFF/COG 导出（地理配准） | 仅 CDN，不打包 |
| `onnxruntime-web` | 目标检测 / 分割 | 动态加载 |

### 3.3 后端与基础设施

| 技术 | 角色 | 位置 |
| --- | --- | --- |
| Tauri v2（Rust） | 桌面/移动壳层 | `apps/geolibre-desktop/src-tauri` |
| FastAPI + uvicorn | 本地处理 sidecar | `backend/geolibre_server` |
| FastAPI + SQLAlchemy | 公共项目/身份 API | `backend/geolibre_server_api` |
| Cloudflare Workers + Durable Objects | 协作中继、瓦片代理、AI 代理、查看器 | `workers/` |
| Node.js + ws + SQLite | 自托管协作中继 | `workers/collab-node` |
| nginx | Docker 静态托管与 sidecar 反向代理 | `docker/nginx.conf` |
| Docker Compose | 一体化自托管栈 | `docker-compose.yml` |
| GitHub Actions | CI、E2E、多平台发布 | `.github/workflows/` |

### 3.4 Python 生态

| 包/技术 | 角色 |
| --- | --- |
| anywidget + traitlets | Jupyter 内嵌 widget（`python/src/geolibre/geolibre.py`） |
| hatchling + 自定义构建钩子 | wheel 打包（`python/hatch_build.py`） |
| MCP Python SDK | `geolibre-mcp` stdio/HTTP 服务器（可选 extra `geolibre[mcp]`） |
| GeoPandas / Shapely / rasterio / rio-cogeo | sidecar 的矢量化、栅格化、转换能力 |
| Apache Sedona（SedonaDB） | sidecar 的 Sedona SQL 引擎 |
| laspy / scikit-image / scipy | 点云标注、OBIA |

## 4. 关键设计原则

### 4.1 Store 驱动，单向数据流

```text
用户操作 → Zustand action → 新的 GeoLibreLayer[] → MapEngine.syncLayers → SDK 源/图层
```

UI 永远不直接调用 `map.addSource` / `map.addLayer`。唯一的例外是插件通过受控的 `GeoLibreAppAPI` 写入 store，或者通过 `MapEngine` 的公开方法操作地图。详见 [02 系统架构](02-architecture.md) 与 [04 核心状态层](04-core-state.md)。

### 4.2 引擎中立（renderer-neutral）

`GeoLibreLayer`、`MapViewState`、`LayerStyle` 等类型不包含任何 SDK 对象。四个引擎各自实现 `MapEngine` 接口（`packages/map/src/map-engine.ts`）。是否支持某功能通过 `MapEngineCapabilities` 判断，而不是比较引擎名称。`eslint-rules/no-renderer-kind-checks.mjs` 会拦截 `primaryRenderer === "cesium"` 这类写法。

### 4.3 引擎隔离

`packages/map/src` 中：

- 以引擎名开头的文件（`arcgis-*`、`cesium-*`、`mapbox-*`，以及 MapLibre 的 `layer-sync`、`map-controller`、`MapCanvas`、`SecondaryMapCanvas`）只能 import 中立模块与自身引擎文件。
- 两引擎共享的代码放入按职责命名的中立模块，如 `pmtiles-archive.ts`、`protocol-tiles.ts`、`gl-style-compiler.ts`、`feature-style.ts`。
- 由 `eslint-rules/no-cross-engine-imports.mjs` 强制。

### 4.4 渐进降级

| 缺失的能力 | 降级行为 |
| --- | --- |
| Python sidecar | 矢量工具退回 Turf.js；栅格核心工具有客户端实现；Sedona 退回 CereusDB |
| `vector` extra | 矢量对话框通过 `/vector/status` 探测后使用客户端引擎 |
| Cesium Ion token | 地球仍可绘制项目底图；仅缺 World Terrain 与部分影像回退 |
| Cesium 不支持某图层 | 图层保留在项目中并标记 "2D only"，切回 2D 时恢复 |
| CDN 不可用 | `GEOLIBRE_NO_EXTERNAL_CDN=1` 可移除外部引用；部分功能按设计禁用 |
| Service Worker | 桌面与 Jupyter 嵌入构建不注册，功能不受影响 |
| 插件引擎不匹配 | 插件声明 `engines`，不匹配时不激活 |

### 4.5 凭据最小暴露

- 桌面端敏感值存 OS 钥匙串（`secure_store_*`），Web 端存 localStorage。
- 项目文件中的凭据在保存时走 "keep/strip" 提示，默认不落盘明文。
- 图层请求头使用 `${ENV_VAR}` 间接引用，不保存真实密钥。
- `secure_store_seal` 在加载外部插件前关闭读取通道，插件无法读取已保存 token。

## 5. 术语表

| 术语 | 含义 |
| --- | --- |
| **Project / 项目** | 一个 `.geolibre`（兼容 `.geolibre.json`）JSON 文件，包含图层、视图、偏好等 |
| **Layer / 图层** | `GeoLibreLayer` 记录；`type` 属于 20 种 `LayerType` 之一 |
| **Layer Group / 图层组** | Layers 面板中的文件夹，可嵌套，带可见性与透明度 |
| **MapEngine** | 渲染引擎的统一接口，四个实现：MapLibre、Mapbox、Cesium、ArcGIS |
| **Primary renderer** | 主地图区使用的引擎，store 字段 `primaryRenderer` |
| **SecondaryMapView** | 分屏/网格中的附加地图窗格，可独立选择引擎与相机 |
| **Capability** | 引擎能力标志（`styleSpec`、`picking`、`terrain` 等）或部署权限（`project:edit` 等） |
| **Sidecar** | 本地 FastAPI 处理服务，默认 `127.0.0.1:8765` |
| **Plugin** | 实现 `GeoLibrePlugin` 的模块，可内置或外部加载 |
| **Workspace / SQL Workspace** | 空间 SQL 面板，支持 DuckDB / PostGIS / Sedona |
| **Story Map** | 滚动驱动的叙事地图，章节含相机、图层可见性、文字 |
| **UI Profile** | Beginner / Intermediate / Advanced / Custom 的界面裁剪配置 |
| **Deployment policy** | `deployment.json`，部署方可授予的权限、品牌、界面与凭据策略 |

## 6. 版本与发布

- 根 `package.json` 版本：`3.5.2`；`package-lock.json` 锁定依赖。
- 项目文件格式版本：`PROJECT_VERSION = "0.2.0"`（`packages/core/src/types.ts:58`）。
- 发布渠道：Web（`web.geolibre.app`）、GitHub Pages demo、Docker/ghcr.io、Microsoft Store、Mac App Store、App Store、Google Play、Chrome Web Store、PyPI（`geolibre`）、conda-forge、AUR/COPR/Flatpak/Homebrew 等。发布工作流位于 `.github/workflows/`。

下一章：[02 系统架构](02-architecture.md)。
