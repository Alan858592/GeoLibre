# 08 应用外壳 `apps/geolibre-desktop`

这是用户实际打开的 GeoLibre 应用：一个 React 19 + TypeScript 的单页应用，外层由 Tauri v2 托管。它同时承担：

- 全部界面（菜单、面板、对话框、地图画布）
- 所有前端 I/O 与格式解析
- 插件宿主与 UI 注册表
- Tauri 命令调用（文件、钥匙串、原生 HTTP、sidecar、Jupyter、Martin）
- Web / 桌面 / Android / iOS / Jupyter 嵌入的差异处理

包内脚本（`apps/geolibre-desktop/package.json`）：

| 脚本 | 作用 |
| --- | --- |
| `dev` | Vite 开发服务器；`predev` 先构建 `@geolibre/embed` 与 JupyterLite |
| `build` | `tsc -b && vite build` |
| `preview` | 预览生产构建 |
| `tauri` / `tauri:dev` | Tauri CLI / 开发模式 |
| `tauri:build` | 调用根 `scripts/tauri-build.mjs` 生成安装包 |
| `tauri:build:native-duckdb` | 启用原生 DuckDB 特性 |
| `tauri:build:mas` | Mac App Store 构建（`mas` feature） |

## 1. 启动流程

### 1.1 `src/main.tsx`

启动是一个**先决条件驱动的异步引导**，保证界面第一次绘制时语言、权限、凭据、原生传输都已就绪。

```text
main.tsx
 1. 安装兼容层：symbol-dispose、crypto.randomUUID、maplibre worker
 2. installEarthdataFetchAuth() / installGeocodingTokenProvider()
 3. installDiagnosticsCapture()（包裹 fetch/错误采集）
 4. 应用部署配置里的 app name
 5. loadDeploymentPolicy()：读取 deployment.json
      └─ 若提供 capabilities，渲染前写入 store
 6. 桌面端：
      - initializeNativeShareAuth()
      - initializeNativeCoordinateOpen()
      - initializeNativeProjectOpen()
      - 安装 ArcGIS / sidecar（仅 Windows）/ WMS identify / geocoding 原生 fetch
      - installNativeShareFetch() / GeoLens fetch
 7. installStaleChunkReload()（Web 重部署后的懒 chunk 404 恢复）
 8. startAnalytics()（仅托管 Web 且构建带 measurement id）
 9. resolveAuthGate()：Clerk / Auth0（仅托管 Web）
10. registerSW()：Web 构建注册 PWA；onNeedReload 被抑制，改为 controllerchange 时提示
11. 读取共享桌面设置（?settings= URL）→ applyTemporaryDesktopSettings
12. hydrateDesktopCredentials()：桌面钥匙串 → 内存
13. 并行等待：
      - import("./App")
      - import("./hooks/usePlugins")
      - 错误边界 chunk
      - 认证门 chunk
      - 原生传输就绪
      - nativeProjectOpenReady / nativeCoordinateOpenReady
      - i18nReady（含懒加载语言包）
      - desktopCredentialsReady
14. root.render(<StrictMode><I18nextProvider><AppErrorBoundary><App/></…>)
```

细节：

- **钥匙串等待屏**：桌面端若钥匙串解锁缓慢，300 ms 后显示 `KeychainWaitScreen`，避免白屏。
- **深链恢复**：`auth-return-url-boot` 必须在 `./i18n` 之前 import，因为两者都在模块加载期读取 query。
- **Service Worker**：`registerType: "autoUpdate"` + `skipWaiting` + `clientsClaim`；但 `onNeedReload` 为空，避免 `/demo/` 相对路径下的误重载。新构建通过 `offerReloadIfNewBuild()` 提示。
- **失败不静默**：原生 fetch 安装失败会 `console.error`，因为对应功能会退回不稳定的浏览器路径。

### 1.2 `src/App.tsx`

App 负责应用级持久化 hooks，然后渲染 `DesktopShell`：

| Hook | 作用 |
| --- | --- |
| `useLastBasemapPersistence` | 记住上次底图 |
| `useLastRendererPersistence` | 记住上次渲染引擎 |
| `useDesktopSettingsPersistence` | 桌面设置读写 |
| `useThemeScheme` / `useThemeMode` | 主题 |
| `useRecentProjectsPersistence` | 最近项目 |
| `useStartupProject` | 恢复启动项目；恢复期间不挂载 shell |
| `useStyleLibraryPersistence` / `useLayerLibraryPersistence` / `useTemplateLibraryPersistence` | 库持久化 |
| `useStartupLayerStyles` | 恢复上次图层样式 |
| `useRuntimeEnvironmentVariables` | 把设备设置投影到 `window.__GEOLIBRE_RUNTIME_ENV__` |
| `useStartupUpdateCheck` | 桌面更新检查 |
| `useProjectUrlLoader` / `useDataUrlLoader` / `useStacUrlLoader` | `?url=`、`?data=`、STAC 深链 |
| `useUndoRedoShortcuts` | Ctrl/Cmd+Z / Shift+Z |
| `useBeforeUnloadGuard` | 未保存提醒 |
| `useWhiteboxToolUrl` | `?tool=` 深链预选工具 |
| `useUiProfileBootstrap` | 首次引导与 UI Profile |

`DirectionProvider` 根据语言设置 RTL。

### 1.3 `DesktopShell`

`src/components/layout/DesktopShell.tsx`（约 1300 行）是应用总装。它负责：

| 区域 | 组件 |
| --- | --- |
| 顶部 | `TopToolbar`（菜单、工具、搜索、主题、账户） |
| 地图区 | `MapGrid` → 主画布 + 副窗格 + 各覆盖层 |
| 左侧/右侧 dock | `LayerPanel`、`StylePanel`、`PluginRightPanel`、`SharedSidebar`、`RightDockPortals` |
| 底部 | `StatusBar`、`BottomPanels`（属性表、时间轴、处理历史等） |
| 浮层 | 图例、评论、故事地图、知识卡片、导航、路线动画、飞行模拟、Sun、NetCDF 剖面 |
| 对话框 | Add Data、Settings、Print Layout、Share、Collaborate、Project History、Diagnostics、Manage Plugins、Onboarding… |
| 错误隔离 | `SectionErrorBoundary` / `SilentErrorBoundary` 包裹各个区块 |

地图挂载逻辑：

```text
primaryRenderer
  ├─ "maplibre" → <MapCanvas>（MapController）
  ├─ "mapbox"   → <PrimaryMapboxCanvas>（MapboxEngine）
  ├─ "cesium"   → <PrimaryCesiumCanvas>（CesiumEngine）
  └─ "arcgis"   → <PrimaryArcgisCanvas>（ArcgisEngine）
```

不支持的引擎组合下，Shell 会隐藏依赖 MapLibre 的功能（地图右键菜单、图例、评论、故事地图、地形设置、ML 面板、MapLibre 类型插件 API），并在 View 菜单中禁用相关项，而不是让它们静默失效。

Shell 还安装了大量 `hooks/desktop-shell/*` 子 hook，例如：

| Hook | 作用 |
| --- | --- |
| `useFileDrop` | 拖放文件导入 |
| `useDropStatus` | 拖放状态提示 |
| `useLayerImport` | 导入图层 |
| `useDataUrlFit` | 深链数据加载后自动 fit |
| `useUrlLoadErrorNotices` | URL 加载错误提示 |
| `useCollabShareLinkAutoOpen` | 分享链接自动打开协作 |
| `usePluginStateRestore` | 项目/渲染器切换后恢复插件 |
| `useRendererHandoff` | 引擎切换时的控件与插件交接 |
| `useRightPanelHost` | 右侧面板宿主 |
| `useTileProtocols` | 注册 MapLibre 自定义协议（MBTiles/PMTiles/COG/KML） |
| `useRasterFileHandlers` | 本地栅格处理 |
| `useMapControlLabels` | 控件文案翻译 |
| `usePanelResize` | 面板尺寸 |
| `useNativeProjectOpenListener` | 桌面文件关联打开 |
| `useWebglContextLossNotice` | WebGL 上下文丢失提示 |

## 2. 目录与模块职责

### 2.1 `src/components`

| 目录 | 内容 |
| --- | --- |
| `layout/` | Shell、工具栏、状态栏、地图网格、Add Data、设置、打印、分享、协作、项目历史、诊断、更新、引导、各主画布 |
| `panels/` | 图层、样式、属性表、SQL、Notebook、Dashboard、Python 控制台、NetCDF、Sun、路线动画、飞行模拟、插件右侧面板 |
| `comments/` | 评论地图叠加、评论工具、添加评论 |
| `storymap/` | 故事地图编辑栏、面板、演示模式 |
| `legend/` | 图例面板 |
| `navigation/` | 导航面板 |
| `processing/` | 处理对话框共享控件 |
| `selection/` | 选择工具 |
| `expressions/` | 表达式编辑器 |
| `command/` | 命令面板 |
| `auth/` | Clerk / Auth0 门 |
| `common/` | 通用错误边界、加载屏等 |

### 2.2 `src/hooks`

104 个 hook，按职责：

| 域 | 代表 hook |
| --- | --- |
| 启动/恢复 | `useStartupProject`、`useStartupLayerStyles`、`useProjectInteractionRestore`、`useScriptControlRestore`、`useTerrainRestore` |
| 项目 I/O | `useProjectFileActions`、`useProjectHistory`、`useProjectUrlLoader`、`useDataUrlLoader`、`useStacUrlLoader` |
| 插件 | `usePlugins`、`usePluginUiSurfaces`、`usePluginStateRestore`、`usePluginDeepLink` |
| 地图 | `useMapCapabilities`、`useMapPanelControl`、`useCesiumIonToken`、`useRasterViewportStretch`、`useExtentScreenOverlay` |
| 识别 | `useRasterIdentify`、`useGlobalRasterIdentify`、`useNetcdfIdentify`、`useCogSpectralIdentify` |
| 嵌入/协作 | `useEmbedApi`、`useEmbedBridge`、`useCollaboration`、`useJupyterRelay`、`useNotebookBridge` |
| UI | `useLayoutOptions`、`useIsMobileViewport`、`useRightPanels`、`useToolbarPanels`、`useThemeMode`、`useLanguage` |
| 数据加载 | `useOsmPbfLoader`、`useTilePackageLoader`、`useDataUrlLoader` |
| 快捷操作 | `useGlobalShortcuts`、`useUndoRedoShortcuts`、`useVoiceCommands`、`useCommandBridge` |

### 2.3 `src/lib`

503 个模块，是前端最厚的一层。按主题：

| 主题 | 代表模块 | 说明 |
| --- | --- | --- |
| DuckDB | `duckdb-wasm-bundles.ts`、`duckdb-vector-loader.ts`、`duckdb-processing.ts`、`duckdb-geometry.ts` | 初始化、扩展加载、矢量转换、SQL |
| 格式解析 | `kml.ts`、`gpx.ts`、`georss.ts`、`gpkg-reader.ts`、`delimited-text.ts`、`csv.ts`、`excel-workbook.ts`、`landxml.ts` | 各格式读取 |
| 文件 I/O | `file-io/`、`tauri-io.ts`、`native-*.ts`、`data-url.ts`、`abortable.ts` | 浏览器/桌面统一落盘 |
| 插件 | `external-plugins.ts`、`plugin-registry.ts`、`plugin-trust.ts`、`plugin-policy.ts`、`plugin-integrity.ts`、`plugin-blocklist.ts` | 安装、信任、完整性、策略 |
| 项目 | `project-url.ts`、`native-project-open.ts`、`autosave-status.ts`、`build-project-snapshot.ts` | 项目打开/保存/快照 |
| 处理 | `duckdb-processing.ts`、`gdal-loader.ts`、`georeference*.ts`、`cog-render-engine.ts`、`cereus-loader.ts`、`pyodide/` | 处理引擎与地理配准 |
| 数据服务 | `arcgis-*`、`stac-*`、`geolens-*`、`s3-*`、`mssql-*`、`martin.ts` | 外部服务集成 |
| 协作/嵌入 | `collab-client.ts`、`collab-protocol.ts`、`embed-api.ts`、`jupyter-relay.ts`、`jupyter.ts` | 实时协作与宿主通信 |
| 安全 | `credential-store.ts`、`credential-hydration.ts`、`desktop-settings-secrets.ts`、`plugin-credentials.ts` | 凭据存储与迁移 |
| 诊断 | `diagnostics.ts`、`issue-report.ts`、`layer-failure-notifier.ts`、`map-error-notification.ts` | 错误采集与提示 |
| 平台 | `is-tauri.ts`、`is-mobile.ts`、`build-flags.ts`、`mas-build.ts`、`tauri-io.ts` | 运行时判定 |
| 脚本/AI | `assistant/`、`scripting/`、`model-*.ts` | AI 助手、脚本控制、模型构建 |
| 地图辅助 | `map-click-modes.ts`、`live-url.ts`、`layer-refresh.ts`、`layer-swatch.ts`、`auto-legend.ts` | 交互与展示 |

### 2.4 `src/i18n`

- `index.ts`：初始化 i18next，英文内置，其余语言懒加载。
- `languages.ts`：语言列表与 RTL 判定。
- `locales/*.json`：21 个语言（ar、az、de、en、es、fa、fr、he、hi、id、it、ja、ka、ko、nl、pt、ru、th、tr、vi、zh）。
- `i18next.d.ts`：以 `en.json` 为键类型来源，新增字符串会得到类型检查。

规则：

- 所有用户可见字符串走 `t("namespace.key")`。
- 使用逻辑方向工具类，禁止物理方向类（`no-physical-tailwind` 规则）。
- 新增 key 先加 `en.json`，再补其他语言（可缺失，会回退英文）。

## 3. Tauri 壳层

### 3.1 工程结构

```text
src-tauri/
├── Cargo.toml            # features: default / native-duckdb / mas
├── build.rs
├── tauri.conf.json       # 窗口、CSP、bundle、devUrl
├── tauri.android.conf.json / tauri.ios.conf.json / tauri.mas.conf.json
├── capabilities/          # Tauri 权限
├── vendor/                # 打补丁的 dialog / geolocation
├── src/
│   ├── main.rs
│   ├── lib.rs             # 命令、窗口、深链、生命周期
│   ├── secure_store.rs
│   ├── native_duckdb.rs
│   ├── arcgis_http.rs
│   ├── plugin_http.rs
│   ├── remote_download.rs
│   ├── aws_credentials.rs / aws_sts.rs
│   ├── postgis_tiles.rs
│   ├── earth_engine_oauth.rs
│   └── http_body.rs
└── icons/ android/ ios/ mas/
```

### 3.2 Tauri 命令

`src-tauri/src/lib.rs` 的 `invoke_handler`（`lib.rs:500`）注册：

| 域 | 命令 |
| --- | --- |
| 窗口/深链 | `close_oauth_popups`、`take_pending_project_paths` |
| 文件 | `read_project_file`、`read_local_file`、`write_local_geojson_file`、`read_shapefile_siblings`、`pick_image_paths`、`read_selected_image`、`allow_raster_asset` |
| 原生矢量 | `native_duckdb::count_native_vector_file_features`、`load_native_vector_file` |
| 配置 | `read_admin_profile`、`read_deployment_policy`、`read_env_vars` |
| HTTP | `fetch_url_bytes`、`fetch_url_response`、`resolve_url_redirect`、`arcgis_http::fetch_arcgis_response` / `cancel_arcgis_request`、`plugin_http::plugin_http_request` / `cancel_plugin_http_request` |
| 下载 | `remote_download::download_remote_file` / `cancel_remote_download` / `pick_download_folder` / `take_cached_download` |
| AWS | `aws_credentials::aws_list_profiles` / `aws_resolve_credentials` / `aws_sso_login_start` / `aws_sso_login_poll` |
| 插件 | `install_external_plugin_archive`、`load_external_plugin_bundles` |
| 服务进程 | `ensure_martin_binary`、`start_martin_server`、`stop_martin_server`、`start_geolibre_sidecar`、`stop_geolibre_sidecar`、`start_jupyter_server`、`stop_jupyter_server` |
| MBTiles | `read_mbtiles_metadata`、`read_mbtiles_tile` |
| 凭据 | `secure_store::secure_store_get_many` / `secure_store_seal` / `secure_store_set` / `secure_store_delete` |
| PostGIS 瓦片 | `postgis_tiles::postgis_tiles_list_sources` / `source_info` / `tile` / `disconnect` |
| OAuth | `start_earth_engine_oauth`、`poll_earth_engine_oauth` |

`mas` feature 下，所有会下载或启动外部进程的命令（sidecar、Jupyter、Martin、uv、外部插件安装）被编译成返回错误的桩，因为 App Sandbox 与 App Store 2.5.2 禁止。

### 3.3 子进程

| 服务 | 默认端口 | 启动方式 | 说明 |
| --- | --- | --- | --- |
| FastAPI sidecar | `127.0.0.1:8765` | `start_geolibre_sidecar` | uv 管理的 Python 环境；每次启动生成 token |
| JupyterLab | `127.0.0.1:8766` | `start_jupyter_server` | Notebook 面板使用；token 拼到 iframe URL |
| Martin | 动态端口 | `start_martin_server` | PostGIS → 矢量瓦片；二进制按需下载 |

前端封装：

- `src/lib/sidecar.ts`：`startGeoLibreSidecar()`、`StaleSidecarError`、token 注入。
- `src/lib/jupyter.ts`：`getJupyterServer()`、启动/停止与监听。
- `src/lib/martin.ts`：目录、图层、瓦片 URL。

### 3.4 凭据存储

`src-tauri/src/secure_store.rs`：

- 服务名 `org.geolibre.desktop`，每个凭据一个条目。
- 使用操作系统钥匙串（macOS Keychain / Windows Credential Manager / Linux Secret Service）。
- `secure_store_get_many` 每个页面加载只允许读取一次（`reset_read_gate` 在页面加载时恢复）。
- `secure_store_seal` 在加载外部插件前关闭读取，插件无法窃取 token。
- 写入顺序：先写索引，再写条目，崩溃不会产生悬空引用。
- 启动时若索引存在但凭据缺失，删除索引并提示。

Web 端对应实现：

- `src/lib/credential-store.ts`：localStorage + 内存缓存。
- `src/lib/credential-hydration.ts`：桌面启动时把钥匙串值加载进内存，并迁移旧版本明文。
- `geolibre.desktopSettings` 中敏感字段留空；`geolibre.postgres.connectionIds` 只存非密钥顺序。

### 3.5 CSP 与安全

`tauri.conf.json` 的 CSP：

- `default-src 'self'`
- `connect-src` 允许 `https:`/`http:`、`ipc:`、`asset:`、WebSocket（含协作中继）
- `script-src` 精确列出版本化的 jsDelivr 路径（Pyodide、PGlite、ONNX Runtime、DuckDB、sql.js、ArcGIS、Mapbox、Google 登录）
- `worker-src blob: 'self'`
- `style-src 'self' 'unsafe-inline'`

Web 构建对应 `docker/nginx.conf`。两处 CSP **必须同步**，`tests/tauri-csp.test.ts` 会从构建配置推导并校验。

其他安全点：

- 文件访问只使用对话框选择的路径 + Tauri 作用域。
- 原生 HTTP 客户端 `guarded_http_client` 信任 OS 证书库，并支持 mTLS 环境变量（`GEOLIBRE_HTTP_CA_CERT`、`GEOLIBRE_HTTP_CLIENT_CERT`、`GEOLIBRE_HTTP_CLIENT_CERT_PASSWORD`）。
- 原生客户端会把客户端证书提供给任何请求它的 HTTPS 服务器，因此只在可信主机环境配置。

## 4. 多端差异

| 能力 | Web | 桌面 | Android/iOS | Jupyter 嵌入 |
| --- | --- | --- | --- | --- |
| 文件对话框 | 浏览器选择器 | Tauri dialog / fs | 原生文档选择 | 浏览器选择器 |
| 本地 MBTiles | sql.js 内存读取 | Tauri `rusqlite` | 同桌面（Tauri） | 不可用/受限 |
| 本地栅格 | 读取受限 | 原生读取 | 原生读取 | 受限 |
| sidecar | Docker `/sidecar` | 按需启动 | 不可用（隐藏） | 不可用 |
| Jupyter | JupyterLite | 外部 JupyterLab | 不可用 | 宿主 kernel |
| 钥匙串 | localStorage | OS 钥匙串 | OS 钥匙串 | localStorage |
| 认证门 | Clerk/Auth0 可选 | 无 | 无 | 无 |
| PWA | 是 | 否（离线打包） | 否 | 否 |
| 深链 | URL query | `org.geolibre.desktop:` + 文件关联 | 深链 | query |

移动端判定见 `src/lib/is-mobile.ts`：

- `isMobile()` 基于 UA + iPad 多触点判定。
- `isDesktopRuntime()` = `isTauri() && !isMobile()`。
- 移动端隐藏需要子进程的功能（sidecar、Martin、Jupyter、AI 分割），但浏览器 WASM 工具（Whitebox）仍可用。

## 5. 嵌入与 Jupyter 集成

### 5.1 iframe 嵌入

- 页面通过 `?embed=1`（以及 `?maponly`、`?panels=hidden`、`?layout=viewer` 等）进入嵌入布局；`useLayoutOptions` 解析。
- `useEmbedApi` 在应用侧实现 postMessage 协议，`useEmbedBridge` 处理父页面事件。
- `@geolibre/embed` 是给宿主页面用的类型化客户端，协议版本 `EMBED_API_VERSION = 2`（仍兼容 v1）。
- 嵌入只允许 `VITE_GEOLIBRE_EMBED_ORIGINS` 配置的来源驱动。

`GeoLibreEmbedClient` 的主要方法（`packages/embed/src/index.ts`）：

| 分组 | 方法 |
| --- | --- |
| 工厂函数 | `connect({ origin, timeoutMs?, requestTimeoutMs? })` → `GeoLibreEmbedClient` |
| 项目 | `loadProject(url)` |
| 视图 | `setView(target)`、`getViewport()` |
| 图层 | `listLayers()`、`setLayerVisibility`、`setFilter`、`addLayer(spec)`、`addData(url, options)` |
| 要素 | `highlightFeature`、`getLayerFeatures`、`getDrawnFeatures` |
| 引擎/工具 | `setRenderer`、`getRenderer`、`openTool(id, params)` |
| 导出 | `exportImage()` |
| 事件 | `on("ready" \| "ack" \| "projectLoaded" \| "selectionChanged" \| "rendererchange" \| "viewChanged" \| "toolCompleted" \| "serverFileWritten" \| "featuresChanged", listener)` |

`featuresChanged` 仅在部署授予 `export:data` 时发送，按图层约 250 ms 防抖；整项目加载用 `projectLoaded` 而非逐层事件。

### 5.2 Jupyter

- 桌面 Notebook 面板：内嵌外部 JupyterLab（`start_jupyter_server`），`useJupyterRelay` 把 map 命令转发给 Notebook kernel。
- Web：JupyterLite 站点（`scripts/build-jupyterlite.mjs`）。
- `python/` 包：anywidget 内嵌完整 GeoLibre 构建，`useNotebookBridge` / `useJupyterRelay` 处理 traitlets 消息。

## 6. 性能与可维护性

| 机制 | 说明 |
| --- | --- |
| 窄 store 订阅 | `useLayer` / `useLayerIds` / `useLayerSummaries`，避免全量图层重渲染 |
| 懒加载面板 | `desktopShellLazyPanels.ts`、`MountWhenOpened` |
| 懒加载插件 | `manualChunks` 每个 `maplibre-gl-*` 独立 chunk |
| Cesium 独立 chunk | 首次切到 3D 才下载 |
| Boot 预算 | `boot-budget.json` + `bootBundleBudgetPlugin` |
| 虚拟化属性表 | `@tanstack/react-virtual` |
| Web Worker | NetCDF、ATL03、project-diff、h5wasm、geojson-vt、区域分割 |
| 诊断 | `installDiagnosticsCapture` 捕获错误与网络响应，Diagnostics 对话框导出 |

Linux 桌面使用 WebKitGTK，低缩放平移时瓦片加载会掉帧；这是 WebView 引擎限制，`docs/architecture.md` 有完整分析与缓解建议（尚未实现）。

## 7. 修改应用外壳的检查清单

- 新面板：在 `components/panels/` 实现，使用 `useAppStore` 的切片与 `useLayer` 系列，必要时用 `MountWhenOpened`。
- 新对话框：在 `components/layout/` 实现，由 `ui` 切片打开；懒加载时加入 `desktopShellLazyPanels.ts`。
- 新持久化：区分"项目数据"（进 `projectFromStore`）与"设备偏好"（进 `useDesktopSettings`）。
- 新 Tauri 能力：在 Rust 增加命令 + `invoke_handler`，更新 `capabilities/` 权限，前端封装在 `lib/`。
- 新字符串：`en.json` + `t()`，使用逻辑方向类。
- 新外部资源：检查 CSP（桌面 + nginx）与 `tests/tauri-csp.test.ts`。
- 新渲染器依赖的功能：用 `useMapCapabilities()`，不要判断 `primaryRenderer`。

下一章：[09 Python 后端](09-backend.md)。
