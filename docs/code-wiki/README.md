# GeoLibre Code Wiki

本文档集是对 [GeoLibre](https://geolibre.app) 仓库的完整代码导读，面向需要阅读、修改或扩展该项目的开发者。内容基于仓库源码（版本 `3.5.2`）整理，覆盖整体架构、模块职责、关键类与函数、依赖关系、运行与构建方式。

## 这是什么项目

GeoLibre 是一个免费、开源、轻量、云原生的 GIS 平台，用于可视化、探索与分析地理空间数据。同一套 React + TypeScript 应用以五种形态分发：

| 形态 | 入口 | 说明 |
| --- | --- | --- |
| Web 应用 | `npm run dev` / `npm run build` | 由 Vite 构建，可托管为静态站点或 Docker/nginx |
| 桌面应用 | `apps/geolibre-desktop/src-tauri` | Tauri v2 壳层，提供文件系统、本地 MBTiles、原生 HTTP、钥匙串等能力 |
| Android / iOS | 同一 Tauri 工程 | 通过 Tauri Mobile 打包 |
| Jupyter | `python/`（`geolibre` 包） | anywidget 内嵌完整应用，提供 leafmap 风格 Python API |
| iframe 嵌入 | `@geolibre/embed` + `?embed=1` | 带类型的 postMessage 嵌入客户端 |

核心能力包括：四种渲染引擎（MapLibre GL JS 默认 / Mapbox GL JS / CesiumJS / ArcGIS Maps SDK）、20 种图层类型、DuckDB-WASM Spatial 数据转换、1000+ 浏览器端 WebAssembly 地理处理工具、可选 Python FastAPI 处理 sidecar、插件系统、协作与分享、故事地图、打印布局等。

## 文档导航

| 文档 | 内容 |
| --- | --- |
| [01 项目总览](01-overview.md) | 定位、核心概念、技术栈、设计原则、术语表 |
| [02 系统架构](02-architecture.md) | 分层架构、单向数据流、渲染引擎模型、运行时拓扑、启动时序 |
| [03 仓库结构](03-repository-layout.md) | monorepo 各 workspace 的职责与目录地图 |
| [04 核心状态层](04-core-state.md) | `@geolibre/core`：Zustand store、切片、类型、项目格式 |
| [05 地图渲染层](05-map-rendering.md) | `@geolibre/map`：`MapEngine`、四引擎、图层同步、样式引擎 |
| [06 UI 与地理处理](06-ui-and-processing.md) | `@geolibre/ui`、`@geolibre/processing`、WASM 工具链 |
| [07 插件体系](07-plugins.md) | 插件接口、生命周期、`PluginManager`、内置插件目录 |
| [08 应用外壳](08-desktop-app.md) | `apps/geolibre-desktop`、`DesktopShell`、hooks/lib、Tauri 命令 |
| [09 Python 后端](09-backend.md) | 本地 sidecar 与公共 Server API |
| [10 Python 包与 MCP](10-python-and-mcp.md) | Jupyter widget、`geolibre` Python API、MCP server、MCP App |
| [11 Workers 与协作](11-workers.md) | 边缘 Worker、协作协议、`collab-node` 自托管中继 |
| [12 数据流与格式](12-data-flow.md) | 数据接入、格式支持矩阵、项目持久化、分享与离线 |
| [13 构建、运行与测试](13-build-run-test.md) | 环境准备、常用命令、构建目标、测试与 CI |
| [14 依赖关系](14-dependencies.md) | 内部依赖图、外部依赖分类、版本约束与跨文件同步点 |
| [15 扩展指南](15-extension-guide.md) | 新增图层类型 / 插件 / 处理工具 / 引擎的落点清单 |

## 快速开始

```bash
git clone https://github.com/opengeos/GeoLibre.git
cd GeoLibre
npm install          # Node.js 22+，npm workspaces 一次装齐所有 JS workspace
npm run dev          # Web 开发服务器 → http://localhost:5173
npm run tauri:dev    # 桌面应用（文件对话框、本地 MBTiles、本地栅格读取需要它）
npm run build        # 生产 Web 构建 → apps/geolibre-desktop/dist/
npm run tauri:build  # 桌面安装包 → apps/geolibre-desktop/src-tauri/target/release/bundle/
```

完整命令、环境变量与测试矩阵见 [13 构建、运行与测试](13-build-run-test.md)。

## 一分钟架构速览

```mermaid
flowchart LR
  UI[React UI<br/>apps/geolibre-desktop] --> Store[Zustand Store<br/>@geolibre/core]
  Store --> Engine[MapEngine 接口<br/>@geolibre/map]
  Engine --> ML[MapLibre GL JS]
  Engine --> MB[Mapbox GL JS]
  Engine --> CZ[CesiumJS]
  Engine --> AG[ArcGIS Maps SDK]
  Plugins[内置/外部插件<br/>@geolibre/plugins] --> Store
  Plugins --> Engine
  UI --> Tauri[Tauri 命令]
  Tauri --> FS[文件系统 / 钥匙串 / 原生 HTTP]
  Proc[处理工具<br/>@geolibre/processing] --> Store
  Proc -. 可选 .-> Sidecar[FastAPI Sidecar<br/>127.0.0.1:8765]
  Sidecar -. 可选 .-> Pyodide[浏览器内 Pyodide]
```

三条必须记住的规则：

1. **Store 是唯一事实来源**。UI 不直接修改 MapLibre；所有图层状态写入 `useAppStore`，由引擎的 `syncLayers` 收敛到地图。
2. **能力驱动，而非引擎名驱动**。是否显示某个功能查询 `MapEngineCapabilities`，不要写 `primaryRenderer === "cesium"`（有 ESLint 规则强制）。
3. **引擎之间不得互相 import**。`packages/map/src` 中引擎专用文件只能引用中立模块与自身；由 `eslint-rules/no-cross-engine-imports.mjs` 强制。

## 关键符号速查

| 符号 | 位置 | 说明 | 详见 |
| --- | --- | --- | --- |
| `useAppStore` | `packages/core/src/store.ts:61` | 全局 Zustand store（11 个切片 + zundo） | [04](04-core-state.md) |
| `GeoLibreLayer` | `packages/core/src/types.ts:1284` | 图层领域模型（20 种类型） | [04](04-core-state.md) |
| `projectFromStore` / `parseProject` / `applyProjectToStore` | `packages/core/src/project.ts:2045/412/2457` | 项目保存/解析/应用 | [04](04-core-state.md) |
| `useLayer` / `useLayerIds` / `useLayerSummaries` | `packages/core/src/layer-selectors.ts:155/166/179` | 图层窄订阅 | [04](04-core-state.md) |
| `MapEngine` / `MapEngineCapabilities` | `packages/map/src/map-engine.ts:45/271` | 渲染器中立接口与能力 | [05](05-map-rendering.md) |
| `MapController` | `packages/map/src/map-controller.ts:229` | MapLibre 引擎实现 | [05](05-map-rendering.md) |
| `CesiumEngine` / `MapboxEngine` / `ArcgisEngine` | `packages/map/src/cesium-engine.ts:303`、`mapbox-engine.ts:188`、`arcgis-engine.ts:577` | 其他三个引擎 | [05](05-map-rendering.md) |
| `syncLayer` / `createLayerSync` | `packages/map/src/layer-sync.ts:481`、`headless.ts:32` | 图层同步（完整/headless） | [05](05-map-rendering.md) |
| `createFeatureStyleResolver` | `packages/map/src/feature-style.ts:175` | 跨引擎逐要素样式求值 | [05](05-map-rendering.md) |
| `PluginManager` | `packages/plugins/src/plugin-manager.ts:29` | 插件注册/激活/恢复/回滚 | [07](07-plugins.md) |
| `GeoLibreAppAPI` | `packages/plugins/src/types.ts:678` | 插件可用的宿主 API | [07](07-plugins.md) |
| `createAppAPI` | `apps/geolibre-desktop/src/lib/app-api.ts:258` | 构造宿主 API | [08](08-desktop-app.md) |
| `DesktopShell` | `apps/geolibre-desktop/src/components/layout/DesktopShell.tsx:188` | 应用总装 | [08](08-desktop-app.md) |
| `runAlgorithmCapture` / `runModel` | `packages/processing/src/runner.ts:46/104` | 单算法/模型运行 | [06](06-ui-and-processing.md) |
| `runWasmToolInBackground` / `runParallelWasmTools` | `packages/processing/src/wasm-tool-runner.ts:423/446` | Whitebox Worker 池 | [06](06-ui-and-processing.md) |
| `resolveExecutionMode` | `packages/processing/src/wasm-capabilities.ts:163` | 三档 WASM 执行模式 | [06](06-ui-and-processing.md) |
| `ensureSpatialExtension` | `apps/geolibre-desktop/src/lib/duckdb-vector-loader.ts:201` | DuckDB Spatial 只加载一次 | [12](12-data-flow.md) |
| `Map`（Python） | `python/src/geolibre/geolibre.py:364` | Jupyter anywidget | [10](10-python-and-mcp.md) |
| `build_server` / `Workspace` | `python/src/geolibre/mcp/server.py:242`、`workspace.py:34` | MCP 工具与路径沙箱 | [10](10-python-and-mcp.md) |
| `connect` / `GeoLibreEmbedClient` | `packages/embed/src/index.ts` | iframe 嵌入客户端 | [08](08-desktop-app.md) |
| `CollabSession` / `ClientMessage` | `workers/collab/src/session.ts`、`packages/collab-core/src/protocol.ts` | 协作中继与协议 | [11](11-workers.md) |

## 建议阅读路线

| 目标 | 路线 |
| --- | --- |
| 第一次接触项目 | [01](01-overview.md) → [02](02-architecture.md) → [03](03-repository-layout.md) → [13](13-build-run-test.md) |
| 改图层/渲染 | [04](04-core-state.md) → [05](05-map-rendering.md) → [14](14-dependencies.md) |
| 写插件 | [07](07-plugins.md) → [08](08-desktop-app.md) → [15](15-extension-guide.md) |
| 加处理工具 | [06](06-ui-and-processing.md) → [09](09-backend.md) → [15](15-extension-guide.md) |
| 做桌面/移动集成 | [08](08-desktop-app.md) → [09](09-backend.md) → [10](10-python-and-mcp.md) |
| 做分享/协作 | [09](09-backend.md) → [11](11-workers.md) → [12](12-data-flow.md) |
| 数据格式与项目文件 | [04](04-core-state.md) → [12](12-data-flow.md) |

## 文档约定

- 代码路径均相对于仓库根目录；行号来自当前 `3.5.2` 源码，版本变化后可能漂移，以符号名为准。
- `docs/code-wiki/` 不在 `mkdocs.yml` 的导航中，因此不会发布到公开文档站，仅作为仓库内的开发者 Wiki。
- 若本 Wiki 与 `docs/architecture.md`、`docs/project-format.md` 等官方参考冲突，以源码与官方参考为准，并请更新本 Wiki。

## 参考入口

- 用户文档：[docs/architecture.md](../architecture.md)（仓库官方架构参考）
- 项目文件格式：[docs/project-format.md](../project-format.md)
- 插件 API：[docs/plugin-api.md](../plugin-api.md)
- 服务端 API：[docs/server-api.md](../server-api.md)
- 维护约定：[docs/maintenance.md](../maintenance.md)
- 贡献指南：[docs/contributing.md](../contributing.md)

> 本 Wiki 以源码为准。若与 `docs/` 下的用户文档冲突，以代码与 `docs/architecture.md` 为准。
