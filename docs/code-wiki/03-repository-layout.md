# 03 仓库结构

## 1. Monorepo 总览

根 `package.json` 声明三个 npm workspaces：

```json
"workspaces": ["apps/*", "packages/*", "workers/*"]
```

一次根目录 `npm install` 会链接所有 JS workspace。仓库还有两个非 npm 组件：

- `backend/geolibre_server` — Python FastAPI 本地处理 sidecar
- `python/` — `geolibre` PyPI 包（Jupyter anywidget + MCP）

```text
GeoLibre/
├── apps/
│   ├── geolibre-desktop/        # 主应用：React + Tauri v2 + Vite
│   └── geolibre-mcp-app/        # MCP App 视图（show_map 的聊天内地图）
├── packages/
│   ├── core/                    # @geolibre/core：类型、store、项目格式
│   ├── map/                     # @geolibre/map：四引擎与图层同步
│   ├── ui/                      # @geolibre/ui：shadcn 风格原语
│   ├── processing/              # @geolibre/processing：客户端算法注册表
│   ├── plugins/                 # @geolibre/plugins：插件 API + 内置插件
│   ├── embed/                   # @geolibre/embed：iframe 嵌入客户端
│   └── collab-core/             # @geolibre/collab-core：协作协议与校验
├── workers/
│   ├── viewer/                  # viewer.geolibre.app 反向代理
│   ├── tiles/                   # tiles.geolibre.app 行星底图/重投影瓦片
│   ├── collab/                  # collab.geolibre.app（Durable Object 中继）
│   ├── collab-node/             # 自托管 Node 协作中继
│   └── ai-proxy/                # AI 代理 Worker
├── backend/
│   ├── geolibre_server/         # 本地处理 sidecar（FastAPI）
│   └── geolibre_server_api/     # 公共项目/身份 API（FastAPI）
├── python/                      # geolibre Jupyter 包 + geolibre-mcp
├── e2e/                         # Playwright 端到端测试
├── tests/                       # 前端 node:test 单测
├── docs/                        # 文档站（含本 Wiki）
├── schema/                      # deployment.schema.json
├── extensions/geolibre-chrome/  # Chrome 扩展：从网页发现数据集
├── packaging/                   # AUR / COPR / MSIX / portable / winget / linux
├── scripts/                     # 构建、代码生成、校验脚本
├── eslint-rules/                # 自定义 ESLint 规则
├── patches/                     # patch-package 补丁
├── docker/                      # nginx、entrypoint、策略脚本
├── skills/geolibre/             # Agent Skill（SKILL.md）
├── apps/geolibre-desktop/public/ # 静态资源、内置 drop-in 插件、Pyodide worker
├── Dockerfile
├── docker-compose.yml
├── vite 配置位于 apps/geolibre-desktop/vite.config.ts
├── mkdocs.yml
└── package.json
```

## 2. 各 Workspace 职责与规模

| Workspace | 包名 | 源文件数（约） | 职责 |
| --- | --- | --- | --- |
| `apps/geolibre-desktop` | `geolibre-desktop` | 1077 | 应用外壳、所有 UI、Tauri、前端 I/O、集成所有包 |
| `apps/geolibre-mcp-app` | `geolibre-mcp-app` | 9 | MCP Apps 的单文件地图视图 |
| `packages/core` | `@geolibre/core` | 93 | 领域类型、store、项目 JSON、纯函数工具 |
| `packages/map` | `@geolibre/map` | 149 | 四引擎、图层同步、样式编译、识别/选择 |
| `packages/ui` | `@geolibre/ui` | 20 | Button/Dialog/Dropdown/Slider/… 与主题 |
| `packages/processing` | `@geolibre/processing` | 49 | 矢量/栅格/统计/网络/拓扑算法与 WASM 运行器 |
| `packages/plugins` | `@geolibre/plugins` | 299 | 插件接口、`PluginManager`、70 个内置插件 |
| `packages/embed` | `@geolibre/embed` | 4 | 无依赖 iframe 客户端 |
| `packages/collab-core` | `@geolibre/collab-core` | 8 | 协作线协议、权限、评论校验、身份 token |
| `workers/viewer` | `geolibre-viewer-worker` | 5 | 反向代理 GitHub Pages demo |
| `workers/tiles` | `geolibre-tiles-worker` | 9 | 行星底图 CORS 代理 + WGS84→WebMercator 重投影 |
| `workers/collab` | `geolibre-collab-worker` | 7 | Durable Object 协作中继 |
| `workers/collab-node` | `geolibre-collab-node` | 6 | Node 版协作中继（自托管） |
| `workers/ai-proxy` | `geolibre-ai-proxy-worker` | 4 | AI 搜索/对话代理 |
| `backend/geolibre_server` | `geolibre-server` | 44 | 本地 FastAPI sidecar（转换/栅格/矢量/SQL/点云/ML） |
| `backend/geolibre_server_api` | — | 13 模块 | 公共项目分享与身份 API |
| `python/src/geolibre` | `geolibre` | 20 | Jupyter widget、Python API、MCP、Dash 组件 |

> 文件数统计包含 `.ts/.tsx/.py/.rs/.mjs/.json/.toml`，用于量级参考。

## 3. `apps/geolibre-desktop` 内部结构

```text
apps/geolibre-desktop/
├── index.html
├── package.json
├── vite.config.ts              # 构建、PWA、CDN 开关、代码分割、boot 预算
├── vite-plugins/               # 自定义 Vite 插件（Cesium 资产、vector_ops 拷贝等）
├── jupyterlite/                # JupyterLite 站点构建
├── public/
│   ├── plugins/<id>/           # 部署方 drop-in 插件目录（可烘焙进构建）
│   ├── pyodide/                # Pyodide classic worker
│   ├── basemaps-assets/        # 行星底图资产
│   ├── deployment.json         # 默认部署策略
│   └── whitebox-catalog-snapshot.json
├── src/
│   ├── main.tsx                # 启动引导
│   ├── App.tsx                 # 持久化 hooks + DesktopShell
│   ├── components/
│   │   ├── layout/             # DesktopShell、TopToolbar、StatusBar、对话框
│   │   ├── panels/             # Layers、Style、AttributeTable、SQL、Notebook、Dashboard…
│   │   ├── comments/           # 评论与地图叠加
│   │   ├── storymap/           # 故事地图编辑/演示
│   │   ├── legend/             # 图例
│   │   ├── navigation/         # 导航面板
│   │   ├── processing/         # 处理对话框共享组件
│   │   ├── selection/          # 选择工具
│   │   ├── expressions/        # 表达式编辑器
│   │   ├── command/            # 命令面板
│   │   └── auth/               # Clerk / Auth0 门
│   ├── hooks/                  # 104 个 hook（启动、项目、插件、地图能力…）
│   ├── lib/                    # 503 个模块（I/O、格式、集成、助手、脚本）
│   ├── i18n/                   # i18next 初始化、语言列表、21 个 locale
│   ├── workers/                # NetCDF / ATL03 / project-diff / h5wasm worker
│   └── types/                  # 全局类型声明
└── src-tauri/                  # Rust 桌面壳
    ├── src/
    │   ├── main.rs
    │   ├── lib.rs              # 全部 Tauri 命令与窗口/深链/生命周期
    │   ├── secure_store.rs     # OS 钥匙串
    │   ├── native_duckdb.rs    # 可选原生 DuckDB
    │   ├── arcgis_http.rs      # ArcGIS 原生 HTTP
    │   ├── plugin_http.rs      # 插件受控 HTTP
    │   ├── remote_download.rs  # 远程下载
    │   ├── aws_credentials.rs / aws_sts.rs
    │   ├── postgis_tiles.rs    # PostGIS → Martin
    │   ├── earth_engine_oauth.rs
    │   └── http_body.rs
    ├── capabilities/           # Tauri 权限声明
    ├── vendor/                 # 打补丁的 dialog / geolocation 插件
    ├── android/ ios/ mas/      # 平台工程
    ├── tauri.conf.json         # 主配置（CSP、窗口、bundle）
    └── Cargo.toml
```

### 3.1 `src/lib` 主题分区

| 子目录 / 文件组 | 代表模块 | 用途 |
| --- | --- | --- |
| `lib/file-io/` | 文件读写、格式嗅探 | 浏览器/桌面统一的文件落盘 |
| `lib/assistant/` | AI 助手工具、模型提供商 | 应用内对话、工具调用 |
| `lib/scripting/` | 脚本控制、Python 控制台桥 | 脚本化 API |
| `lib/navigation/` | 导航、路线 | 导航面板 |
| `lib/obia/` | OBIA 工作台 | 对象分割与分类 |
| `lib/pyodide/` | Pyodide worker 封装 | 浏览器内 GeoPandas |
| `duckdb-*.ts` | DuckDB 初始化/加载/几何 | 本地矢量转换与 SQL |
| `georeference*.ts`、`gdal-loader.ts` | 地理配准 | 客户端 GeoTIFF/COG 导出 |
| `kml*.ts`、`gpx.ts`、`georss.ts` | XML 地理格式 | 导入/导出 |
| `external-plugins.ts`、`plugin-*.ts` | 插件安装/信任/凭据 | 外部插件生命周期 |
| `collab-client.ts`、`collab-protocol.ts` | 协作客户端 | WebSocket 同步 |
| `deployment-policy.ts`、`deployment-gates.ts` | 部署策略 | 权限与品牌 |
| `diagnostics.ts`、`issue-report.ts` | 诊断 | 问题报告 |
| `tauri-io.ts`、`native-*.ts` | 桌面 I/O | 原生命令封装 |

### 3.2 `src/hooks` 关键分组

| 分组 | 代表 hook | 作用 |
| --- | --- | --- |
| 启动/持久化 | `useStartupProject`、`useDesktopSettings`、`useRecentProjectsPersistence`、`useLastRendererPersistence` | 恢复上次会话 |
| 插件 | `usePlugins`、`usePluginUiSurfaces`、`usePluginStateRestore` | 注册与激活插件 |
| 地图 | `useMapCapabilities`、`useMapPanelControl`、`useTerrainRestore`、`useCesiumIonToken` | 引擎能力与地图状态 |
| 项目 | `useProjectFileActions`、`useProjectHistory`、`useProjectInteractionRestore` | 打开/保存/撤销 |
| 数据 | `useDataUrlLoader`、`useProjectUrlLoader`、`useStacUrlLoader`、`useOsmPbfLoader` | 深链与外部数据 |
| 嵌入/协作 | `useEmbedApi`、`useEmbedBridge`、`useCollaboration`、`useJupyterRelay` | iframe、协作、笔记本桥 |
| 处理 | `useGlobalRasterIdentify`、`useRasterIdentify`、`useCogSpectralIdentify` | 栅格识别 |

### 3.3 `src/components` 关键文件

| 文件 | 作用 |
| --- | --- |
| `layout/DesktopShell.tsx` | 应用总装：地图、面板、工具栏、对话框、插件宿主（1306 行） |
| `layout/TopToolbar.tsx` | 菜单栏与工具按钮 |
| `layout/StatusBar.tsx` | 坐标、缩放、高程、GPS、渲染引擎状态 |
| `layout/MapGrid.tsx` | 单地图 / 多窗格网格布局 |
| `layout/AddDataDialog.tsx` | 统一数据接入入口 |
| `panels/LayerPanel.tsx` | 图层树、分组、可见性、拖放排序 |
| `panels/StylePanel.tsx` | 符号化、分类/分级/规则/表达式 |
| `panels/AttributeTable.tsx` | 虚拟化属性表、编辑、筛选、统计 |
| `panels/SqlWorkspacePanel.tsx` | DuckDB / PostGIS / Sedona SQL |
| `panels/NotebookPanel.tsx` | 内嵌 Jupyter（桌面）/ JupyterLite（Web） |
| `layout/PrintLayoutDialog.tsx` | 打印布局与图集 |
| `storymap/` | 故事地图编辑与演示 |
| `layout/desktopShellLazyPanels.ts` | 懒加载处理对话框的聚合入口 |

## 4. `packages/core` 文件地图

| 类别 | 文件 | 内容 |
| --- | --- | --- |
| 类型 | `types.ts` | `GeoLibreLayer`、`LayerStyle`、`MapViewState`、`ProjectPreferences`、`LAYER_TYPES` |
| 项目 | `project.ts` | `createEmptyProject`、`serializeProject`、`parseProject`、`projectFromStore`、`applyProjectToStore`、各 normalize 函数 |
| Store | `store.ts`、`store/*.ts` | Zustand store 与 11 个切片、撤销历史 |
| 图层工具 | `layer-defaults.ts`、`layer-groups.ts`、`layer-selectors.ts`、`layer-filters.ts`、`quick-filters.ts`、`joins.ts`、`virtual-fields.ts` | 图层默认值、分组、选择器、过滤、连接 |
| 样式 | `color-ramp.ts`、`vector-color.ts`、`expressions.ts`、`style-layer-evaluator.ts`、`style-library.ts` | 色带、颜色、MapLibre 表达式、样式求值 |
| 渲染辅助 | `cesium-imagery.ts`、`cesium-ion.ts`、`cesium-kml.ts`、`czml.ts`、`three-d-tiles.ts`、`ellipsoids.ts`、`regional-basemaps.ts` | 引擎中立的地图数据描述 |
| 数据 | `geojson-z.ts`、`geoarrow.ts`、`attribute-form.ts`、`layer-descriptive-metadata.ts` | GeoJSON Z 处理、GeoArrow、属性表单 |
| 平台 | `runtime-env.ts`、`paths.ts`、`platform.ts`、`s3.ts`、`routing.ts` | 运行时环境、路径、S3 |
| 安全 | `credentials.ts`、`project-credential-refs.ts`、`header-references.ts`、`visibility.ts` | 凭据脱敏、项目凭据引用、请求头变量、字段隐藏 |
| 内容 | `storymap-*.ts`、`print-layout-config.ts`、`diagram.ts`、`popup.ts` | 故事地图、打印布局、图表、弹窗配置 |

## 5. `packages/map` 文件地图

| 前缀/组 | 文件示例 | 作用 |
| --- | --- | --- |
| 入口/画布 | `index.ts`、`MapCanvas.tsx`、`SecondaryMapCanvas.tsx`、`CesiumCanvas.tsx`、`MapboxCanvas.tsx`、`ArcgisCanvas.tsx`、`headless.ts` | React 画布与 headless 入口 |
| 引擎 | `map-controller.ts`、`cesium-engine.ts`、`mapbox-engine.ts`、`arcgis-engine.ts`、`map-engine.ts`、`renderer-capabilities.ts` | `MapEngine` 四实现 + 能力表 |
| 图层同步 | `layer-sync.ts`、`cesium-layer-sync.ts`、`arcgis-layers.ts`、`mapbox-layers.ts`、`layer-kind.ts`、`maplibre-layer-kinds.ts` | 类型分派与同步 |
| 样式 | `gl-style-compiler.ts`、`feature-style.ts`、`vector-style.ts`、`mapbox-style.ts`、`mapbox-style-import.ts`、`mapbox-style-export.ts`、`qml-import.ts`、`qml-export.ts`、`sld-import.ts`、`sld-export.ts`、`style-mapper.ts` | 样式编译与互转 |
| 栅格 | `cog-imagery.ts`、`cog-dem-source.ts`、`raster-color-adjustments.ts`、`raster-identify.ts`、`cesium-cog-imagery.ts`、`arcgis-cog-imagery.ts` | COG/栅格渲染 |
| 点云/3D | `cesium-point-cloud.ts`、`cesium-point-cloud-stream.ts`、`cesium-tileset-style.ts`、`deck-viz` 相关 | 点云、3D Tiles、splat |
| 瓦片 | `pmtiles-archive.ts`、`protocol-tiles.ts`、`pmtiles-layer.ts`、`wms-proxy.ts`、`tile-request-headers.ts` | 自定义协议、PMTiles、WMS |
| 交互 | `map-feature-selection.ts`、`feature-selection.ts`、`identify-*.ts`、`feature-popup.ts`、`markers.ts`、`photo-popup.ts` | 选择、识别、弹窗、标记 |
| 相机/CRS | `cesium-camera.ts`、`map-crs.ts`、`active-map-crs.ts`、`epsg-proj4.ts`、`map-bounds.ts` | 相机换算与投影 |
| 其他 | `map-capture.ts`、`map-resize.ts`、`globe-fit-bounds.ts`、`layer-control-host.ts`、`basemap-style.ts` | 截图、尺寸、地球自适应、图层控件 |

## 6. `packages/plugins` 结构

```text
packages/plugins/src/
├── index.ts                    # 全部公开导出
├── types.ts                    # GeoLibrePlugin / GeoLibreAppAPI / 各种注册表类型
├── plugin-manager.ts           # PluginManager 生命周期
├── plugin-ids.ts               # 稳定插件 id 常量
├── viewer-plugins.ts           # viewer 模式可用插件集合
├── right-panel-registry.ts     # 右侧面板注册
├── toolbar-menu-registry.ts    # 工具栏菜单贡献
├── menu-contribution-registry.ts
├── floating-panel-registry.ts
├── assistant-tool-registry.ts
├── control-panels.ts
├── panel-dom.ts / panel-title.ts
├── localized-error.ts
└── plugins/                    # 约 213 个文件：插件实现 + 共享辅助
```

内置插件按用途分组（详见 [07 插件体系](07-plugins.md)）：

- 基础：图层控制、底图控制、OSM/CARTO、标注、尺寸、GeoEditor
- 数据目录/服务：STAC、Socrata、CKAN、ArcGIS Portal/Hub、US Federal/State/Local GIS、NASA Earthdata、USGS、EnviroAtlas、FEMA、National Map、GeoLens、Overture、S3、Source Cooperative、HuggingFace、Planetary Computer、Earth Engine
- 栅格/影像：3D Tiles、Raster、COG、LiDAR、Splat、Zarr、时间滑块、Timelapse、Sentinel-2
- 分析/工具：Sun、Clouds、Precipitation、Effects、Flight Simulator、Route Animation、God's Eye View、Elevation Profile、Swipe、Directions、Reverse Geocode、GeoAgent、SamGeo、点云标注
- 网格系统：H3、S2、A5、DGGRID、DGGAL、OLC、Geohash、Tilecode、MGRS
- 可视化：deck.gl Viz、Components、Graticule

## 7. `workers` 与后端

| 目录 | 运行时 | 入口 | 部署 |
| --- | --- | --- | --- |
| `workers/viewer` | Cloudflare Worker | `src/index.ts` → `proxyViewerRequest` | `deploy-viewer.yml` |
| `workers/tiles` | Cloudflare Worker | `src/index.ts` | `deploy-tiles.yml` |
| `workers/collab` | Worker + Durable Object | `src/index.ts` + `session.ts` | `deploy-collab.yml` |
| `workers/collab-node` | Node `http` + `ws` + SQLite | `src/server.ts` | Docker / `docker-compose.yml` |
| `workers/ai-proxy` | Cloudflare Worker | `src/index.ts` | wrangler |
| `backend/geolibre_server` | Python 3.10+ FastAPI | `geolibre_server/app/main.py` | 桌面按需 / Docker `/sidecar` |
| `backend/geolibre_server_api` | Python FastAPI + SQLAlchemy | `geolibre_server_api/main.py` | Docker Compose / 自托管 |

## 8. 根目录重要文件

| 文件 | 作用 |
| --- | --- |
| `package.json` | workspace、脚本、依赖 override、Node >= 22 |
| `package-lock.json` | 锁文件（使用 npm，不用 pnpm/yarn） |
| `.nvmrc` | Node 主版本 |
| `tsconfig.base.json` | 严格 TS、`noEmit`、`moduleResolution: bundler` |
| `eslint.config.mjs` | ESLint 10 flat config，含本地规则 |
| `ruff.toml` | Python lint/format 配置 |
| `rust-toolchain.toml` | Tauri 的 Rust 工具链 |
| `Dockerfile` | 多阶段 Web 镜像（Node 构建 → nginx） |
| `docker-compose.yml` | web + server-api + collab + postgres |
| `docker/nginx.conf` | 静态托管、SPA fallback、`/sidecar` 代理、CSP |
| `docker/entrypoint.sh` | 校验部署策略、生成 nginx 守卫、启动 sidecar |
| `schema/deployment.schema.json` | `deployment.json` 的 JSON Schema |
| `mkdocs.yml` | 文档站导航 |
| `playwright.config.ts` | E2E 项目配置 |
| `boot-budget.json` | 入口 JS 预算 |
| `.pre-commit-config.yaml` | 预提交钩子（含本地 `npm-build`） |

## 9. 生成文件与不要手改的文件

以下文件由脚本生成，修改源后需重新生成并在同一 PR 提交：

| 生成物 | 生成命令 |
| --- | --- |
| Whitebox 菜单目录 | `scripts/gen-whitebox-menu-catalog.mjs` |
| 处理工具 i18n 目录 | `npm run i18n:tools` |
| 插件参考文档 | `npm run plugins:docs` |
| Demo gallery | `npm run gallery` |
| `skills/geolibre/` | 由仓库脚本/维护流程同步 |

`apps/geolibre-desktop/public/whitebox-catalog-snapshot.json`、`docs/user-guide/processing*` 中的工具列表也与此相关。

下一章：[04 核心状态层](04-core-state.md)。
