# 10 Python 包、MCP 与多端分发

`python/` 是发布到 PyPI 的 `geolibre` 包，提供两种使用方式：

1. **Jupyter anywidget**：在 notebook 单元格里内嵌完整 GeoLibre 应用，并提供 leafmap / geemap 风格的 Python API。
2. **MCP server**：`geolibre-mcp` 以 stdio 或 HTTP 提供项目编写、导出与桌面实时控制工具。

包版本动态取自 `python/src/geolibre/__init__.py`（当前 `3.5.2`），Python >= 3.11。

## 1. 包结构

```text
python/
├── pyproject.toml
├── hatch_build.py               # 构建钩子：打包 dist-embed、扫描凭据
├── README.md
├── examples/                    # getting-started.ipynb 等
├── jupyter-config/
│   └── jupyter_server_config.d/geolibre.json
├── src/geolibre/
│   ├── __init__.py              # 公开 API、Jupyter 扩展入口
│   ├── geolibre.py              # Map / Layer / Feature widget
│   ├── project.py               # 项目构造与规范化
│   ├── authoring.py             # 对整份项目应用编辑
│   ├── basemaps.py              # 底图目录与解析
│   ├── color_ramp.py            # 色带
│   ├── legends.py               # 内置图例预设
│   ├── polyline.py              # Google Polyline 编解码
│   ├── pointcloud.py            # 点云标签写回
│   ├── spaceborne_lidar.py      # ICESat-2 / GEDI
│   ├── _server.py               # 本地静态服务器（iframe 用）
│   ├── _extension.py            # Jupyter Server 扩展
│   ├── dash_component.py        # DashMap
│   ├── _frontend.js             # anywidget 前端入口
│   ├── static/                  # 打包后的应用（git-ignored）
│   └── mcp/                     # MCP 服务器
└── tests/
```

### 1.1 构建

`npm run build:embed` 生成 `apps/geolibre-desktop/dist-embed`，`python/hatch_build.py` 把它打进 wheel 的 `geolibre/static/`。

- wheel 声明 `artifacts = ["src/geolibre/static/**"]`。
- sdist 强制包含 `scripts/credential-patterns.json`，供构建钩子扫描凭据。
- `jupyter-config/jupyter_server_config.d/geolibre.json` 安装到 `{sys.prefix}/etc/jupyter/...`，使 `pip install geolibre` 后扩展自动启用。
- 构建前扫描打包产物中的凭据模式，避免把开发者 token 带进发行版。

## 2. Jupyter widget

### 2.1 `Map`

`python/src/geolibre/geolibre.py:364`：

```python
class Map(anywidget.AnyWidget):
    ...
```

它持有：

- 一份 `.geolibre.json` 项目字典（`_project`）
- 一个 `_pending` 请求/响应队列（用于从 kernel 调用前端方法并等待结果）
- 事件订阅者（点击、选择变化、图层变化）
- 一个 iframe 基础 URL（本地静态服务器 / Jupyter Server 扩展 / Colab 代理）

数据同步是双向的：

```text
Python 修改 project → traitlets 同步 → JS 应用 parseProject → 地图更新
用户在地图操作 → JS 发送自定义消息 → Python 更新 project / 触发回调
```

### 2.2 Python API 分组

| 分组 | 方法 |
| --- | --- |
| 视图 | `get_view`、`get_center`、`get_bounds`、`fly_to`、`fit_bounds`、`zoom_to_bounds`、`zoom_to_layer` |
| 图层 | `add_geojson`、`add_gdf`、`add_kml`、`add_gpkg`、`add_cog`、`add_raster`、`add_tile_layer`、`add_wms_layer`、`add_wmts_layer`、`add_vector_tiles`、`add_pmtiles`、`add_zarr`、`add_lidar`、`add_3d_tiles`、`add_ee_layer`（可选） |
| 标记/可视化 | `add_marker`、`add_markers`、`add_circle_markers`、`add_marker_cluster`、`add_heatmap`、`add_polyline`、`add_legend`、`add_colorbar` |
| 图层控制 | `layers`、`layer_names`、`get_layer`、`find_layer`、`rename_layer`、`move_layer`、`duplicate_layer`、`show_layer`、`hide_layer`、`set_layer_visibility`、`set_layer_opacity`、`set_layer_filter`、`set_labels`、`set_popup`、`set_tooltip`、`clear_popup`、`set_layer_metadata` |
| 选择/识别 | `identify`、`get_features`、`get_selected_features`、`get_drawn_features`、`user_rois`、`on_click`、`on_selection_change`、`on_layer_change` |
| 处理 | `list_algorithms`、`run_algorithm`、`run_model_builder`、`list_whitebox_tools`、`run_whitebox_tool` |
| 故事地图/书签 | `set_story_map`、`add_story_chapter`、`remove_story_chapter`、`move_story_chapter`、`add_bookmark`、`remove_bookmark` |
| 插件 | `set_plugin_state` |
| 导出 | `to_image`、`to_html`、`describe` |
| 事件 | `on(event, callback)`、`request(method, ...)` |

`Feature` 与 `Layer` 是轻量数据类，用于返回类型与图层引用。

### 2.3 托管方式

`_server.py` 说明三种运行环境：

| 环境 | 应用来源 |
| --- | --- |
| 本地 Jupyter / VS Code | `ThreadingHTTPServer` 绑定 `127.0.0.1`，进程级单例 |
| Google Colab | Colab 端口代理 |
| JupyterHub / 远程 | Jupyter Server 扩展（`_extension.py`）托管静态资源 |

本地服务器还提供两个带不可猜测 token 的路径：

- `/_geolibre_local/<token>`：用户显式注册的本地文件（如 GeoTIFF）
- `/_geolibre_tiles/<token>/{z}/{x}/{y}.png`：Colab 安全的 XYZ 瓦片渲染

### 2.4 Dash

`dash_component.py` 的 `DashMap` 是可选封装（`geolibre[dash]`）。它以 iframe 形式嵌入同一个应用，并通过 `clickData` / `selectionData` / `viewData` 属性与 Dash 回调交互。

## 3. 项目构造与应用的分层

Python 侧刻意拆成两层，避免重复实现：

| 层 | 文件 | 职责 |
| --- | --- | --- |
| **构造层** | `project.py` | 构造项目片段：`build_empty_project`、`_layer_base`、`marker_style`、`popup_config`、`layer_metadata`、`redact_credentials`、`default_map_view`、规范化函数 |
| **应用层** | `authoring.py` | 把编辑应用到完整项目：`load_project`/`save_project`、`add_layer`/`remove_layer`/`update_layer`、`apply_style`、`set_popup`、`set_layer_metadata`、`build_choropleth_style`、`classify_layer`、`set_labels`、`set_view`、`fit_bounds`、`set_renderer`、`set_map_layout`、`secondary_panes`、`set_plugin_state`、`set_story_map`、书签 |

`Map` 和 MCP 工具都委托给 `authoring.py`，因此 Jupyter 里的编辑与 MCP 生成的项目语义一致。`python/tests/test_mcp_roundtrip.py` 会把 MCP 写出的项目交给应用侧的 `parseProject` 校验。

## 4. MCP server

### 4.1 定位

`python/src/geolibre/mcp/` 是一个 headless MCP 服务器，用于"用语言生成地图"：

- 无需浏览器、无需运行 GeoLibre、无需打包前端即可写 `.geolibre.json`。
- 可选 `show_map` 在支持的客户端内联预览（MCP Apps）。
- `live_*` 工具连接正在运行的 GeoLibre Desktop Notebook relay。

安装：

```bash
pip install "geolibre[mcp]"
```

运行：

```bash
geolibre-mcp --root ~/maps
```

- `--root` 可重复；环境变量 `GEOLIBRE_MCP_ROOTS`（`:` 分隔，Windows 用 `;`）。
- 未指定时工作区为当前目录。
- `--transport` 支持 `stdio`（默认）、`streamable-http`、`sse`。

### 4.2 模块分层

| 模块 | 职责 |
| --- | --- |
| `__init__.py` | `main()` 入口；缺 `mcp` extra 时给出可操作提示而不是 traceback |
| `server.py` | 唯一 import `mcp` SDK 的模块；`build_server(workspace)`（`:242`）注册全部工具 |
| `workspace.py` | `Workspace` 类：把所有路径限制在 roots 内；拒绝 symlink 逃逸；只写 `.json`/`.html`；默认不覆盖 |
| `project.py` | 复用 `geolibre.project` / `authoring` 的构造与应用函数 |
| `live.py` | 发现桌面 Notebook relay（connection file + pid + loopback 校验）、转发命令 |
| `network.py` | 预览的联网授权：只允许公共 IP、固定解析结果、限制体积与超时 |

### 4.3 工具分类（62 个）

| 分类 | 工具 |
| --- | --- |
| 项目生命周期 | `create_project`、`describe_project`、`list_catalog`、点云相关 `get_point_cloud_annotations` / `set_point_cloud_classes` / `prelabel_point_cloud` / `write_labeled_point_cloud` |
| 添加图层 | `add_geojson_layer`、`add_vector_layer`、`add_raster_layer`、`add_tile_layer`、`add_tiles_layer`、`add_ogc_layer`、`add_lidar_layer`、`add_spaceborne_lidar_layer`、`add_3d_tiles_layer`、`add_cesium_ion_layer`、`add_czml_layer`、`add_cesium_kml_layer` |
| 编辑 | `update_layer`、`remove_layer`、`style_layer`、`set_layer_popup`、`set_layer_metadata`、`classify_layer`、`list_layer_properties`、`set_layer_filter`、`set_labels` |
| 取景与装饰 | `set_view`、`set_basemap`、`set_renderer`、`set_map_layout`、`set_map_legend`、`add_legend`、`add_colorbar`、`add_swipe` |
| 插件/故事地图 | `set_plugin_state`、`set_story_map`、`add_story_chapter`、`move_story_chapter`、`remove_story_chapter`、`add_bookmark`、`remove_bookmark` |
| 桌面实时 | `live_status`、`live_list_layers`、`live_fly_to`、`live_fit_bounds`、`live_zoom_to_layer`、`live_set_basemap`、`live_add_geojson`、`live_remove_layer`、`live_set_visibility`、`live_set_opacity`、`live_set_style`、`live_list_algorithms`、`live_run_algorithm` |
| 导出 | `export_html` |
| 预览 | `show_map` 与 App-only 的 `get_map_preview` / 授权 / 取资源 / 关闭会话工具 |

### 4.4 工作区安全

`workspace.py`：

- 每个工具调用的路径先解析到 roots 内。
- 拒绝指向 roots 外的 symlink。
- 相对路径按第一个 root 解析。
- 写入只允许 `.json` 项目与 `.html` 导出，且不能是裸 `.json`。
- 已存在文件必须显式 `overwrite`。
- 编辑现有项目前检查它确实是项目文件，避免把 `package.json` 当项目改写。

### 4.5 Live relay

`live.py`：

- 从运行时目录/connection file 发现桌面 Jupyter 服务器。
- 只允许 loopback，校验进程存活。
- 把 `live_*` 调用转换为对 Notebook relay 的 HTTP 请求。
- 处理算法运行约 5 秒超时：超时后返回"未在时限内完成"，避免重复触发。

### 4.6 MCP App 预览

`apps/geolibre-mcp-app` 构建 `show-map.html`：

- 使用 `@geolibre/core` 与 `@geolibre/map/headless` 渲染只读地图。
- 只绘制 MapLibre 原生图层（内联 GeoJSON、栅格/WMS/WMTS/XYZ、矢量瓦片、图片叠加）。
- 其他图层（点云、3D Tiles、COG、ArcGIS、PMTiles、DuckDB、deck.gl、本地 MBTiles、video、Z 高程 GeoJSON）在状态栏说明"未显示"，而不是静默缺失。
- 网络请求由 Python MCP server 代取：浏览器不发请求，避免 CORS；仅允许公共 IP；每个资源 4 MiB、15 秒超时；重定向失败即拒绝。
- 授权面板一次性列出底图与可见图层的来源；未批准不发起请求。
- 预览只读，不写项目；会话 30 分钟空闲过期，最多 64 个并发。

构建：

```bash
npm run build:mcp-app
```

## 5. 其他分发物

### 5.1 R 包

R 包不在本仓库源码中，文档见 [`docs/r.md`](../r.md)，代码在独立仓库（`r.geolibre.app`）。它提供 RStudio / Quarto / R Markdown / Shiny 的 htmlwidgets 封装。

### 5.2 Chrome 扩展

`extensions/geolibre-chrome/`：

| 文件 | 作用 |
| --- | --- |
| `manifest.json` | Manifest V3 |
| `popup.html` / `popup.mjs` | 弹出面板 |
| `scanner.mjs` / `service-scanner.mjs` | 扫描页面中的数据链接 |
| `url-builder.mjs` | 生成 GeoLibre 打开 URL |
| `options.html` / `options.mjs` / `base-url-settings.mjs` | 配置自托管基址 |
| `PRIVACY.md` / `STORE_LISTING.md` | 商店材料 |

打包：`npm run build:chrome-extension`（`scripts/package-chrome-extension.mjs`）。

### 5.3 Agent Skill

`skills/geolibre/SKILL.md` + `references/` 教外部 AI agent（Claude Code、Claude Desktop 等）如何生成 GeoLibre 地图。它与应用内 AI Assistant 互补：

| | 应用内 Assistant | Agent Skill |
| --- | --- | --- |
| 运行位置 | 应用内 | 应用外的 agent |
| 作用对象 | 当前打开的地图 | 磁盘上的 `.geolibre.json` |
| 输出 | 可撤销的编辑 | 项目文件 / 独立 HTML |

### 5.4 MCP App

见 4.6。

## 6. 测试

| 测试 | 说明 |
| --- | --- |
| `python/tests/` | widget、项目构造、authoring、MCP、Dash、点云 |
| `python/tests/test_mcp_server.py` | 无 `mcp` extra 时 skip；CI 的 `publish-python.yml` 显式安装 `mcp` |
| 构建 | `cd python && pip install -e ".[dev]" && pytest` |

注意：`python/hatch_build.py` 在构建时会扫描凭据模式；如果 sdist → wheel 构建（没有 monorepo checkout），它会使用强制包含的 `credential-patterns.json`。

下一章：[11 Workers 与协作](11-workers.md)。
