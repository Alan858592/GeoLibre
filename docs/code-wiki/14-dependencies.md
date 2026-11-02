# 14 依赖关系

## 1. 内部依赖图

```mermaid
graph TD
  App[apps/geolibre-desktop] --> Plugins[@geolibre/plugins]
  App --> Map[@geolibre/map]
  App --> Core[@geolibre/core]
  App --> UI[@geolibre/ui]
  App --> Processing[@geolibre/processing]
  App --> Embed[@geolibre/embed]
  MCPApp[apps/geolibre-mcp-app] --> Map
  MCPApp --> Core
  MCPApp --> UI
  Plugins --> Map
  Plugins --> Core
  Map --> Core
  Processing --> Core
  CollabWorker[workers/collab] --> CollabCore[@geolibre/collab-core]
  CollabNode[workers/collab-node] --> CollabCore
```

> `@geolibre/ui`、`@geolibre/embed`、`@geolibre/collab-core` 没有内部依赖：它们刻意保持可独立复用（UI 原语、iframe 客户端、协作协议）。

| 包 | 依赖的内部包 | 说明 |
| --- | --- | --- |
| `@geolibre/core` | 无 | 领域内核；只依赖 zustand/zundo/uuid/style-spec |
| `@geolibre/map` | `@geolibre/core` | 读取类型与纯函数；不反向依赖 plugins |
| `@geolibre/plugins` | `@geolibre/core`、`@geolibre/map` | 插件通过 MapEngine 操作地图 |
| `@geolibre/ui` | 无 | 纯 UI 原语 |
| `@geolibre/processing` | `@geolibre/core` | 只读类型；DuckDB 能力由宿主注入 |
| `@geolibre/embed` | 无 | 运行时无依赖，构建后发布 |
| `@geolibre/collab-core` | 无 | 纯协议/校验，供 Worker 与前端共享 |
| `geolibre-desktop` | 上述全部 | 组合层 |
| `geolibre-mcp-app` | core/map/ui | 只读预览 |
| `geolibre-collab-worker` / `geolibre-collab-node` | collab-core | 同一协议两个运行时 |

### 1.1 依赖方向的意义

- **core 不依赖任何上层**：测试、Worker、Python 嵌入都能只拿类型与纯函数。
- **map 不知道 plugins**：引擎是基础设施，插件是消费者。
- **processing 不知道 map**：算法只接收 `GeoLibreLayer` 与 `ProcessingContext`，结果通过回调交给宿主。
- **collab-core 不知道 UI**：Cloudflare 与 Node 两个中继共用同一套校验与权限。
- **apps 是唯一的组合层**：所有跨包编排都在 `apps/geolibre-desktop`。

## 2. 各包的外部依赖

### 2.1 `@geolibre/core`

| 依赖 | 用途 |
| --- | --- |
| `zustand` | 全局 store |
| `zundo` | 撤销/重做 |
| `uuid` | 生成 id |
| `@maplibre/maplibre-gl-style-spec` | 表达式/过滤校验（不引入地图引擎） |

设计约束：core 不依赖 React、DOM、MapLibre、Cesium、deck.gl。

### 2.2 `@geolibre/map`

| 依赖 | 用途 |
| --- | --- |
| `maplibre-gl` | 默认 2D 引擎 |
| `mapbox-gl` | 可选 2D 引擎 |
| `cesium` / `@cesium/engine` / `@cesium/widgets` | 3D 地球 |
| `@maplibre/geojson-vt` / `@maplibre/vt-pbf` | 客户端矢量瓦片 |
| `pmtiles` | PMTiles 归档 |
| `cog-tiler-wasm` | COG 分块 |
| `geotiff` / `geotiff-geokeys-to-proj4` | GeoTIFF 与 CRS |
| `copc` / `laz-perf` | 点云解码 |
| `zarrita` / `zarr-cesium` | Zarr |
| `proj4` | 投影 |
| `satellite.js` | 卫星轨道（太阳/星空相关） |
| `@protomaps/basemaps` | Protomaps 底图 |
| `maplibre-gl-layer-control` | 图层控件 |
| `@turf/*` | 几何辅助（bbox、buffer、centroid…） |
| `react` / `react-dom` | Canvas 组件 |
| `fast-xml-parser` | WMS/样式 XML |

`@geolibre/map` 同时发布 headless 子入口，供 `@geolibre/mcp-app` 与外部宿主使用。

### 2.3 `@geolibre/plugins`

依赖极多（60+），按其用途分组：

| 组 | 代表包 |
| --- | --- |
| 地图控件 | `maplibre-gl-*`（3d-tiles、lidar、splat、raster、vector、swipe、time-slider、components、basemap-control、directions、streetview、overture-maps、planetary-computer、earth-engine…） |
| deck.gl | `@deck.gl/core`、`layers`、`geo-layers`、`aggregation-layers`、`mesh-layers`、`mapbox`、`maplibre` |
| 栅格/科学 | `@developmentseed/deck.gl-geotiff`、`deck.gl-raster`、`morecantile`、`cog-tiler-wasm`、`geotiff`、`@carbonplan/zarr-layer`、`zarrita`、`h5wasm`、`netcdfjs`、`icechunk-js`、`@carbonplan/zarr-layer` |
| 矢量/格式 | `shpjs`、`pmtiles`、`pbf`、`@mapbox/vector-tile`、`h3-js`、`s2js`、`a5-js`、`dggal`、`webdggrid`、`mgrs`、`ngeohash`、`open-location-code-typescript` |
| 3D | `three`、`@dvt3d/maplibre-three-plugin`、`@loaders.gl/i3s`、`lerc` |
| 编辑 | `@geoman-io/maplibre-geoman-free`、`@turf/union` |
| 其他 | `jspdf`、`fflate`、`mapillary-js`、`whitebox-wasm` |

### 2.4 `@geolibre/processing`

| 依赖 | 用途 |
| --- | --- |
| 25+ `@turf/*` 子包 | 矢量工具 |
| `geolibre-wasm` | Whitebox WASI 二进制 |
| `@bjorn3/browser_wasi_shim` | WASI 运行时 |
| `duckdb-wasm`（由宿主注入能力） | SQL/DGGS |
| `geotiff` / `geotiff-geokeys-to-proj4` | 栅格读取与 CRS |
| `onnxruntime-web` | 目标检测 |
| `proj4` | 投影 |
| `a5-js` / `dggal` / `s2js` | DGGS |
| `fflate` | 压缩包 |
| `@geolibre/core` | 类型 |

### 2.5 `apps/geolibre-desktop`

在内部包之外，应用直接依赖：

| 类别 | 包 |
| --- | --- |
| React 生态 | `react`、`react-dom`、`react-i18next`、`i18next`、`@tanstack/react-virtual` |
| AI | `openai`、`@anthropic-ai/sdk`、`@google/genai`、`@aws-sdk/client-bedrock`、`@modelcontextprotocol/sdk`、`@strands-agents/sdk` |
| 认证 | `@clerk/react`、`@auth0/auth0-react` |
| 浏览器计算 | `@duckdb/duckdb-wasm`、`@electric-sql/pglite`、`@electric-sql/pglite-postgis`、`@cereusdb/standard`、`gdal3.js`、`sql.js`、`apache-arrow` |
| 文件格式 | `@e965/xlsx`、`shpjs`、`fflate`、`exifr`、`h3-js`、`geotiff`、`proj4`、`@osmix/*` |
| 地图 | `maplibre-gl` 及其插件、`cesium`、`three` |
| 导出 | `jspdf`、`html2canvas-pro`、`svgcanvas`、`qrcode.react` |
| Tauri | `@tauri-apps/api`、`plugin-dialog`、`plugin-fs`、`plugin-http`、`plugin-deep-link`、`plugin-geolocation`、`plugin-opener` |
| 安全 | `dompurify`、`zod` |

### 2.6 Workers

| Worker | 依赖 |
| --- | --- |
| `viewer` | 无 |
| `tiles` | `upng-js` |
| `collab` | `@geolibre/collab-core` |
| `collab-node` | `@geolibre/collab-core`、`ws` |
| `ai-proxy` | 无（只使用 Cloudflare 运行时） |

### 2.7 Python

| 包 | 依赖 |
| --- | --- |
| `geolibre-server` | `fastapi`、`uvicorn`（其余为 extras） |
| `geolibre-server-api` | FastAPI、SQLAlchemy、`psycopg`（可选）、S3 SDK（可选） |
| `geolibre` | `anywidget`、`traitlets`、`jupyter-ui-poll`；`mcp`、`geopandas`、`rasterio`、`xarray`、`laspy`、`h5py` 等为 extras |

## 3. 版本约束与覆盖

### 3.1 根 `package.json` 的 `overrides`

| Override | 原因 |
| --- | --- |
| `@deck.gl/geo-layers.a5-js: ^1.0.0` | deck.gl 9.4 仍依赖 a5-js 0.10，GeoLibre 使用 A5 v1 |
| `@loaders.gl/compression.fflate: ^0.8.3` | 统一 fflate |
| `brace-expansion`、`fast-uri`、`filelist`、`sweepline-intersections` | 安全/兼容修复 |
| `maplibre-gl: ^6.15.0` | 锁定主引擎版本 |
| `miniflare.undici` | Workers 测试兼容 |
| `texture-compressor.image-size` | 兼容 |
| `three: ^0.186.1` | 只允许一份 three（`@dvt3d/maplibre-three-plugin` 声明 ^0.178） |
| `uuid: ^14.0.0` | 统一 UUID |
| `eslint-plugin-jsx-a11y.eslint` | 共享 ESLint 实例 |

### 3.2 共享依赖一致性

`npm run check:shared-deps`（`scripts/check-shared-deps.mjs`）比较：

- `apps/geolibre-desktop/package.json`
- `packages/plugins/package.json`

同一包在任意依赖段（dependencies/dev/peer/optional）中范围不同即失败。原因是 npm 无法 hoist 单份副本时会嵌套第二份，导致：

- 包体翻倍；
- 手工镜像的上游私有字段检查的是另一份；
- Dependabot 每个 manifest 单独开 PR，容易漏。

修复方式：给两个 manifest 相同范围，并在同一 PR 刷新 `package-lock.json`。

### 3.3 单份 `three`

- 上游 `maplibre-gl-3d-tiles >= 0.5.11`、`maplibre-gl-splat >= 0.2.9`、`maplibre-gl-components >= 0.31.2` 把 three 声明为 peer。
- 根 `overrides` 把 three 钉到应用范围。
- 升级后运行 `npm ls three`，应只列一个主线程版本。
- `mapillary-js` 的 dist 内嵌 0.134 的 three，在独立 worker/上下文运行，仍会有一条 "Multiple instances of Three.js" 警告。

### 3.4 单份 `a5-js`

- GeoLibre 全局使用 A5 v1（`a5-js` 1.x）。
- deck.gl 的 `A5Layer` 声明 0.10，`overrides` 强制到 1.x（它只调用 v1 未变的 API）。
- DuckDB 的 `a5` 社区扩展在当前 DuckDB（1.5.1）仍是 v0 索引；`a5-tools.ts` 对返回 ID 调用 `migrate()`。升级到 DuckDB >= 1.5.6 后应移除迁移。

### 3.5 补丁

`patches/@cogeotiff+core+9.5.0.patch` 由 `postinstall` 应用。补丁文件记录适用版本；升级到不匹配版本时 `npm install` 失败。

## 4. 发布布局

### 4.1 npm 包

`.github/workflows/publish-packages.yml` 在 Release 时发布：

- `@geolibre/core`
- `@geolibre/map`
- `@geolibre/embed`

特殊约定：

- 仓库内的 `main`/`types`/`exports` 指向 **TypeScript 源码**，因为 monorepo 的 Vite/tsc/tsx 直接解析 `src`。
- npm tarball 必须发布 `dist`，但 npm 忽略 `publishConfig` 下的入口字段（npm/cli#7586）。
- 因此 `scripts/prepare-npm-package.mjs` 在 `npm publish` 前把 `publishConfig` 的入口提升到顶层，并把 `"*"` 的 `@geolibre/core` 依赖钉到发布版本。
- `tsdown --format esm --dts` 产出 `.mjs` + `.d.mts`，不是 `.d.ts`；`tests/prepare-npm-package.test.ts` 校验发布路径与产物一致。

### 4.2 PyPI

`publish-python.yml` 构建 wheel/sdist：

- `python/hatch_build.py` 打包 `dist-embed` 与 `dist-mcp-app`。
- 构建前扫描凭据。
- 显式安装 `mcp`，否则 `test_mcp_server.py` 会 skip。

### 4.3 容器

`publish-container.yml` 发布 `ghcr.io/opengeos/geolibre`：

- PR 构建但不推送。
- push 到 `main`、版本 tag、手动触发时推送。

## 5. 跨文件同步点

这些点“不同步也不会报错”，必须人工维护：

| 同步点 | 文件 | 校验 |
| --- | --- | --- |
| CSP `script-src` | `apps/geolibre-desktop/src-tauri/tauri.conf.json` + `docker/nginx.conf` | `tests/tauri-csp.test.ts` |
| 共享依赖范围 | app + plugins `package.json` | `npm run check:shared-deps` |
| Whitebox 菜单目录 | `src/lib/whitebox-menu-catalog.ts` | `node scripts/gen-whitebox-menu-catalog.mjs` |
| 处理工具 i18n | `i18n/locales/*.json` 工具条目 | `npm run i18n:tools:check` |
| 插件参考 | `docs/plugin-api.md` / registry | `npm run plugins:docs:check` |
| Demo gallery | `docs/gallery.md` / JSON | `npm run gallery:check` |
| sidecar lockfile | `backend/geolibre_server/uv.lock` | CI `uv lock --check` |
| 版本号 | 根/app/packages/python/Cargo/tauri.conf | 发布脚本 |
| 生成物 | `whitebox-catalog-snapshot.json`、品牌图标、下载镜像 | 各生成脚本 |
| 上游镜像常量 | 见 `docs/maintenance.md` 依赖升级章节 | `tests/upstream-contracts.test.ts` |

### 5.1 上游私有契约镜像

以下包的上游私有字段/常量被手工镜像，升级后必须按 `docs/maintenance.md` 检查：

- `maplibre-gl`（内部类/控件）
- `maplibre-gl-*` 控件
- `geolibre-wasm`（`MAX_VECTOR_PMTILES_ZOOM`、参数名分类）
- `geotiff`（`actualizedFields` 私有 map）
- `@tauri-apps/plugin-http`
- `@carbonplan/zarr-layer`（`createShaderProgram` 的 uniform 查找）

`tests/upstream-contracts.test.ts` 会检查**已安装**包的形状；语义问题仍需运行前端测试与人工验证。

## 6. 导入边界（ESLint）

`eslint.config.mjs` 加载 `eslint-rules/` 下的本地规则：

| 规则 | 作用 |
| --- | --- |
| `local/no-cross-engine-imports` | `packages/map/src` 中引擎文件不得 import 另一引擎 |
| `local/no-renderer-kind-checks` | 禁止按引擎名分支（`primaryRenderer === "cesium"`）；须用 capability |
| `local/no-physical-tailwind` | 禁止 `ml-`/`left-` 等物理方向类，强制逻辑属性 |

例外需要显式注释：

```ts
// eslint-disable-next-line local/no-renderer-kind-checks -- Mapbox keeps its own persisted style URL
```

删除规则或放宽范围前，先确认对应不变量仍有其他保障。

## 7. 依赖升级检查清单

升级以下任一包时，先读 `docs/maintenance.md` 对应章节，再运行前端测试：

- [ ] `maplibre-gl` / 所有 `maplibre-gl-*`
- [ ] `cesium` / `@cesium/*`
- [ ] `geolibre-wasm`
- [ ] `@tauri-apps/plugin-http`
- [ ] `geotiff`
- [ ] `@carbonplan/zarr-layer`（精确版本，无 `^`）
- [ ] `three`（检查 `npm ls three`）
- [ ] `a5-js`（检查 DuckDB a5 扩展索引版本）
- [ ] `@duckdb/duckdb-wasm`（CSP 路径 + spatial 扩展）
- [ ] `@electric-sql/pglite*`（CSP 路径）
- [ ] `onnxruntime-web`（CSP 路径）
- [ ] `zod` / `i18next`（类型与语言包）
- [ ] `patch-package` 补丁适用版本

升级后：

1. 两个 manifest 的范围保持一致。
2. 同一 PR 更新 `package-lock.json`。
3. 若 CDN 脚本路径变化，同时更新桌面与 nginx 的 CSP。
4. 运行 `npm run ci:frontend` 与 E2E 引擎测试。
5. 若涉及生成文件，重新生成并提交。

## 8. Python extras 矩阵

| extra | 影响的前端功能 | 缺失时行为 |
| --- | --- | --- |
| `vector` | Vector 工具 sidecar 引擎 | 回退 Turf/Pyodide |
| `raster` | Raster 工具全集 | 仅客户端子集 |
| `conversion` | 转换工具（GeoParquet/COG） | 回退 DuckDB-WASM |
| `sedona` | SQL Workspace Sedona 引擎 | 回退 CereusDB |
| `postgis` | PostGIS 可编辑图层 | 只读/不可用 |
| `mssql` | SQL Server 可编辑图层 | 不可用 |
| `ml` | AI 分割代理 | 不可用 |
| `obia` | OBIA 原生分割 | 客户端实现 |
| `pointcloud` | 点云标签写回 | 不可用 |
| `notebook` | 桌面 Notebook 面板 | Jupyter 不可用 |
| `whitebox` | sidecar Whitebox 引擎 | 使用浏览器 WASM |
| `mcp` | MCP 服务器 | `geolibre-mcp` 不可用 |
| `spaceborne` | ICESat-2 / GEDI | 不可用 |
| `dash` | `DashMap` | 不可用 |
| `s3`（API） | S3 存储 | 本地目录 |

下一章：[15 扩展指南](15-extension-guide.md)。
