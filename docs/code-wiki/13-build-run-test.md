# 13 构建、运行与测试

## 1. 环境要求

| 依赖 | 版本 | 用途 |
| --- | --- | --- |
| Node.js | >= 22（`.nvmrc` 固定主版本） | 所有 JS workspace |
| npm | 随 Node | 仓库跟踪 `package-lock.json`，不要混用 pnpm/yarn |
| Rust | stable（`rust-toolchain.toml`） | Tauri 桌面/移动 |
| Python | >= 3.10 | sidecar、pre-commit |
| Python | >= 3.11 | `python/` 的 `geolibre` 包 |
| Linux 桌面构建 | `webkit2gtk`、`libayatana-appindicator` | Tauri v2 前置依赖 |
| Playwright Chromium | `npx playwright install chromium` | E2E |

## 2. 安装

```bash
git clone https://github.com/opengeos/GeoLibre.git
cd GeoLibre
npm install
```

`postinstall` 会运行 `scripts/apply-dependency-patches.mjs` 应用 `patches/` 中的补丁。

Python 侧按需：

```bash
pip install -e "backend/geolibre_server[test]"     # sidecar + 全量测试
pip install -e "backend/geolibre_server_api[test]" # 公共 API
cd python && pip install -e ".[dev, mcp]"          # Jupyter widget + MCP
```

> 安装 sidecar 时**必须**用 `[test]` 跑全量测试；只用 `[dev]` 会让矢量/栅格/SQL/ML 测试自行 skip，CI 显示绿色但未覆盖。

## 3. 运行

### 3.1 Web

```bash
npm run dev
# → http://localhost:5173
```

能力：

- 地图、本地矢量导入（DuckDB-WASM）
- 拖放 GeoTIFF/COG、Shapefile zip、KMZ、GeoJSON
- 所有 URL/服务型图层
- 浏览器内处理（Turf/Pyodide/Whitebox WASM）

不支持：文件对话框、本地 MBTiles、本地栅格文件读取、项目保存/打开（需要 Tauri）。

### 3.2 桌面

```bash
npm run tauri:dev
```

能力：桌面文件系统、钥匙串、原生 HTTP、本地 MBTiles、sidecar、Jupyter、Martin、深链、单实例。

### 3.3 Docker

```bash
docker build -t geolibre .
docker run --rm -p 8080:80 geolibre
# → http://localhost:8080
```

或完整自托管栈：

```bash
POSTGRES_PASSWORD=... docker compose up -d
```

| 服务 | 端口 | 说明 |
| --- | --- | --- |
| `geolibre-web` | 8080 | nginx + 前端 + `/sidecar` |
| `geolibre-server` | 127.0.0.1:8000 | 公共 API |
| `geolibre-collab` | 8787 | 协作中继 |
| `postgres` | 内部 | 数据库 |

sidecar 仅当部署策略授予 `processing:run` 或 `data:add` 且 `GEOLIBRE_DISABLE_SIDECAR != 1` 时启动。sidecar 读写路径受 `GEOLIBRE_CONVERSION_ROOTS`（默认 `/data`）限制：

```bash
docker run --rm -p 8080:80 -v "$PWD/data:/data" ghcr.io/opengeos/geolibre:latest
```

### 3.4 Android / iOS

同一 Tauri 工程，使用 `tauri.android.conf.json` / `tauri.ios.conf.json`。构建由 `.github/workflows/android.yml` 与 `ios.yml` 驱动；本地需安装对应 Tauri Mobile 工具链。

移动端限制：

- 不能启动 Python sidecar / Jupyter / Martin。
- 浏览器 WASM 工具仍可用。
- 原生定位通过打过补丁的 `tauri-plugin-geolocation`。

### 3.5 Jupyter

```bash
npm run build:embed              # dist-embed → python/src/geolibre/static/app
cd python && pip install -e .
```

在 notebook 中：

```python
from geolibre import Map
m = Map(center=(-100, 40), zoom=4)
m.add_geojson("https://example.com/data.geojson", name="Data")
m
```

JupyterLite：

```bash
npm run build:jupyterlite
```

### 3.6 MCP

```bash
pip install "geolibre[mcp]"
npm run build:mcp-app          # 若从源码运行，生成 show-map.html
geolibre-mcp --root ~/maps
```

### 3.7 Chrome 扩展

```bash
npm run build:chrome-extension
```

产物在 `extensions/geolibre-chrome`（打包脚本会生成可上传 zip）。

## 4. 构建目标

| 命令 | 输出 | 说明 |
| --- | --- | --- |
| `npm run build` | `apps/geolibre-desktop/dist/` | 标准 Web 构建（`tsc -b && vite build`） |
| `npm run lite:build` | `dist/`（可 `--outDir`） | `GEOLIBRE_DUCKDB_WASM_CDN=1`，并检查无文件超过 25 MiB |
| `npm run build:embed` | `dist-embed/` → `python/src/geolibre/static/app` | 相对 base、无 Service Worker、凭据扫描 |
| `npm run build:jupyterlite` | JupyterLite 站点 | Docker/文档演示 |
| `npm run build:mcp-app` | `apps/geolibre-mcp-app` 单文件 | MCP App 预览 |
| `npm run build:chrome-extension` | 扩展 zip | Chrome Web Store |
| `npm run tauri:build` | `target/release/bundle/` | 桌面安装包；Linux 默认 deb+rpm |
| `npm run tauri:build:native-duckdb` | 同 | 启用 Rust 原生 DuckDB |
| `npm run tauri:build:mas` | Mac App Store 包 | 仅 macOS；`mas` feature |
| `npm run msix:build` | MSIX | Windows Store（PowerShell） |
| `npm run portable:build` | 便携版 | Windows |
| `npm run bundle:report` | 包体报告 | 分析 chunk 体积 |

### 4.1 构建开关

| 变量 | 默认 | 作用 |
| --- | --- | --- |
| `GEOLIBRE_APP_BASE` | `/` | 应用基础路径；嵌入构建用 `./` |
| `GEOLIBRE_NO_EXTERNAL_CDN` | 未设置 | `1` 时移除所有 GeoLibre 控制的外部 CDN，并禁用依赖功能 |
| `GEOLIBRE_DUCKDB_WASM_CDN` | 未设置 | `1` 时 DuckDB-WASM 从 jsDelivr 加载 |
| `GEOLIBRE_PGLITE_CDN` | `1` | `0` 时把 PGlite/PostGIS 打包进构建（约 +22 MB） |
| `GEOLIBRE_CEREUS_CDN` | `1` | `0` 时把 CereusDB WASM 打包 |
| `GEOLIBRE_GDAL_CDN` | `1` | `0` 时禁用 gdal3.js 导出（而非打包） |
| `GEOLIBRE_EMBED` | 未设置 | `1` 生成嵌入构建（相对路径、无 SW） |
| `GEOLIBRE_STRIP_CREDENTIALS` | 未设置 | `1` 生成可分发的脱敏构建 |
| `GEOLIBRE_STORE_BUILD` | 未设置 | `1` 商店构建：关闭应用内更新 |
| `GEOLIBRE_MAS_BUILD` | 未设置 | `1` Mac App Store 构建 |
| `VITE_CROSS_ORIGIN_ISOLATION` | 未设置 | `1` 开发/预览服务器发送 COOP/COEP，启用 `threaded-simd` |

构建期还会把白名单环境变量写入 `__GEOLIBRE_BUILD_ENV__`，运行时可通过 `window.__GEOLIBRE_RUNTIME_ENV__` 覆盖（空值不覆盖构建值）。

### 4.2 运行时/部署变量（节选）

| 变量 | 作用 |
| --- | --- |
| `VITE_GEOLIBRE_SHARE_URL` / `GEOLIBRE_SHARE_URL` | 项目分享服务器；`off` 关闭 Share/Gallery |
| `VITE_GEOLIBRE_COLLAB_URL` / `GEOLIBRE_COLLAB_URL` | 协作中继 URL |
| `VITE_GEOLIBRE_APP_NAME` / `GEOLIBRE_APP_NAME` | 应用名（工具栏、标签页） |
| `VITE_GEOLIBRE_EMBED_ORIGINS` / `GEOLIBRE_EMBED_ORIGINS` | 允许驱动 iframe 的父页面源 |
| `VITE_GEOLIBRE_CAPABILITIES` | 旧版权限配置（`deployment.json` 优先） |
| `VITE_GEOLIBRE_PLUGIN_REGISTRY_URL` | 插件市场注册表 |
| `VITE_GEOLIBRE_VIEWER_URL` | 查看器源 |
| `VITE_GEOLENS_DEFAULT_URL` / `GEOLIBRE_GEOLENS_URL` | GeoLens 默认服务 |
| `VITE_CESIUM_TOKEN` / `CESIUM_TOKEN` | Cesium Ion token（也可在 Settings 运行时设置） |
| `VITE_MAPBOX_ACCESS_TOKEN` | Mapbox token |
| `VITE_DUCKDB_SPATIAL_EXTENSION_PATH` | 同源 spatial 扩展路径（离线） |
| `VITE_PYODIDE_INDEX_URL` | Pyodide 镜像 |
| `GEOLIBRE_DEPLOYMENT_FILE` | Docker 部署策略源文件 |
| `GEOLIBRE_SIDECAR_TOKEN` | sidecar 每会话 token（Docker 自动生成） |
| `GEOLIBRE_CONVERSION_ROOTS` | sidecar 允许的路径根 |
| `GEOLIBRE_POSTGIS_HOSTS` / `GEOLIBRE_MSSQL_HOSTS` | 数据库主机白名单 |
| `GEOLIBRE_DISABLE_SIDECAR` | `1` 时 Docker 不启动 sidecar |
| `GEOLIBRE_AI_URL` / `GEOLIBRE_AI_PROXY_URL` / `GEOLIBRE_AI_PROXY_TOKEN` | Docker AI 代理（三者必须同时设置） |
| `GEOLIBRE_HTTP_CA_CERT` / `GEOLIBRE_HTTP_CLIENT_CERT` / `GEOLIBRE_HTTP_CLIENT_CERT_PASSWORD` | 桌面原生 HTTP 的自定义 CA / mTLS |
| `GEOLIBRE_MCP_ROOTS` | MCP 工作区根（`:` 分隔） |

## 5. 测试

### 5.1 前端单元/组件

```bash
npm run test:frontend            # node --test tests/*.test.ts（tsx loader）
npm run test:frontend:coverage   # 覆盖率门禁
node --import tsx --test tests/<name>.test.ts
```

- 覆盖 `tests/` 下数百个 `*.test.ts`。
- 覆盖率下限：**78% lines / 78% branches / 63% functions**（`scripts/coverage-check.mjs`）。
- 覆盖率只统计被测试 import 的文件；首次为大模块写测试可能暂时拉低函数覆盖率，见 `docs/maintenance.md#coverage-floors`。

### 5.2 后端

```bash
npm run test:backend             # pytest backend/geolibre_server/tests
npm run test:backend:coverage    # --cov-fail-under=55
npm run test:docker              # pytest docker/tests
python -m pytest backend/geolibre_server/tests/test_x.py::test_y
cd python && pytest              # geolibre 包
```

### 5.3 Workers

```bash
npm run test:worker
# = typecheck viewer + collab + collab-node + tiles + ai-proxy
#   + collab-node 单测
```

### 5.4 Rust

```bash
npm run check:rust
# cargo check --manifest-path apps/geolibre-desktop/src-tauri/Cargo.toml
```

### 5.5 E2E

```bash
npx playwright install chromium
npm run test:e2e
npm run test:e2e:core       # core + core-engines
npm run test:e2e:features
npm run test:e2e:sso        # 需要 API + Keycloak
npm run test:e2e:preview    # 需要已部署站点
```

`playwright.config.ts`：

- 先 `npm run build`，再 `vite preview`，用 Playwright 驱动生产构建。
- 项目：`core`、`core-engines`（依赖 core，单 worker）、`features`。
- `enterprise-sso/` 与 `preview/` 有独立配置。
- 构建时清空 `CESIUM_TOKEN`/`VITE_CESIUM_TOKEN`，保证测的是无 token 路径。

### 5.6 类型检查与静态检查

```bash
npm run lint                 # eslint apps packages workers tests --max-warnings 242
npm run check:shared-deps    # 跨 workspace 依赖版本一致性
npm run typecheck:tests      # tsc over tests/，错误数基线
npm run typecheck:fast       # 构建 @geolibre/embed + tsc -b（不写 dist）
npm run typecheck            # 注意：等同 npm run build，会写 dist/
npm run i18n:tools:check     # 处理工具 i18n 目录是否最新
npm run gallery:check        # demo gallery 是否最新
npm run plugins:docs:check   # 插件参考是否最新
```

### 5.7 完整门禁

```bash
npm run ci          # lint + check:shared-deps + ci:frontend + typecheck:tests + ci:backend + check:rust
npm run ci:frontend # i18n/gallery/plugins 检查 + build + 覆盖率 + 未测试模块 + worker
npm run ci:backend  # 后端覆盖率 + docker 测试
```

## 6. CI 与发布

`.github/workflows/` 主要工作流：

| 工作流 | 作用 |
| --- | --- |
| `ci.yml` | PR/push 门禁；路径过滤决定运行哪些 job |
| `e2e-full.yml` | 全量 E2E |
| `pages.yml` / `web-deploy.yml` | GitHub Pages / Web 部署 |
| `deploy-viewer.yml` / `deploy-tiles.yml` / `deploy-collab.yml` | Workers 部署 |
| `publish-container.yml` | ghcr.io 镜像 |
| `publish-packages.yml` | `@geolibre/core` / `@geolibre/map` npm 发布 |
| `publish-python.yml` | PyPI `geolibre` |
| `publish-embed.yml` | `@geolibre/embed` 发布 |
| `release.yml` / `release-dry-run.yml` | 桌面安装包与 Release |
| `android.yml` / `ios.yml` | 移动端 |
| `mas-store.yml` / `msix-store.yml` | 商店构建 |
| `sync-download-mirror.yml` | 下载镜像同步 |
| `pr-preview.yml` / `pr-preview-deploy.yml` | PR 预览 |
| `studio-deploy.yml` | 文档/Studio |
| `claude*.yml` | AI 辅助审查/修复 |

CI 的路径过滤：

- `web`：几乎全部路径（前端测试读取 backend/python/docs/src-tauri）。
- `containers`：docker、collab-node、collab-core、schema 等。
- `backend`、`rust`、`identity`、`mssql` 各自按路径。

## 7. 预提交

`.pre-commit-config.yaml` 包含：

- ruff（Python）
- prettier/oxfmt（按 `.oxfmtrc.json`）
- 本地 `npm-build` 钩子：会编译整个应用，**很慢**，且可能触碰无关构建产物。

推荐只对改动文件运行：

```bash
pre-commit run --files <paths>
```

## 8. 常见问题

| 现象 | 原因/解决 |
| --- | --- |
| `npm run test:e2e` 启动失败 | 未安装 Chromium：`npx playwright install chromium` |
| 后端测试大量 skip | 未安装 `[test]` extra |
| 覆盖率突然下降 | 新测试 import 了大模块；查看 `docs/maintenance.md#coverage-floors` |
| `npm run typecheck` 产生 `dist/` | 它实际是完整 build；纯类型检查用 `typecheck:fast` |
| 桌面 Linux 低缩放平移掉帧 | WebKitGTK 限制，见 `docs/architecture.md` 性能章节 |
| Linux 桌面空白窗口 | CPU 缺少 AVX；见 `docs/getting-started.md#linux-desktop-a-blank-window-on-a-cpu-without-avx` |
| Docker 上传失败 | `geolibre-projects` 卷可能由 root 创建；按 README 修复所有权 |
| Cloudflare 部署被 25 MiB 限制拒绝 | 使用 `npm run lite:build` |
| 外部插件未加载 | 检查部署策略 `sideload`、插件黑名单、信任对话框 |
| sidecar 401 | token 不匹配（可能残留旧 sidecar）；重启桌面应用 |

## 9. 发布检查清单

- [ ] `npm run ci` 通过。
- [ ] 生成物已更新：`npm run i18n:tools`、`npm run gallery`、`npm run plugins:docs`。
- [ ] CSP 两处同步（`tauri.conf.json` + `docker/nginx.conf`）。
- [ ] 版本号（根 `package.json`、`packages/*/package.json`、`python/__init__.py`、Tauri `Cargo.toml`/`tauri.conf.json`）一致。
- [ ] 依赖升级检查 `docs/maintenance.md`（maplibre/cesium/wasm 镜像）。
- [ ] 凭据扫描：`node scripts/scan-credentials.mjs`（构建钩子也会跑）。
- [ ] 新增外部资源已加入 CSP。

下一章：[14 依赖关系](14-dependencies.md)。
