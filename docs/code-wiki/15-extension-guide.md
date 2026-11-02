# 15 扩展指南

本章给出常见扩展任务的落点清单，避免"改了一处、漏了三处"。

## 1. 新增一种图层类型

假设新增 `my-layer`。

### 1.1 核心层

1. `packages/core/src/types.ts`
   - 在 `LAYER_TYPES` 中加入 `"my-layer"`（`LayerType` 自动派生）。
   - 如需新字段，扩展 `GeoLibreLayer` / `LayerStyle`。
   - 在 `DEFAULT_LAYER_STYLE` 中补默认值（`layer-defaults.ts`）。
2. `packages/core/src/project.ts`
   - `prepareLayerForSave` / `projectFromStore`：决定哪些字段持久化、哪些是运行时。
   - `parseProject`：校验与 normalize（坏的导入不应进入 store）。
3. `packages/core/src/layers-file.ts` / `layer-style-file.ts`：若支持单独导入导出。
4. 若图层需要新的源描述，添加引擎中立的辅助（如 `my-layer.ts`）。

### 1.2 地图层

1. `packages/map/src/layer-kind.ts`：把 `"my-layer"` 映射到 `LayerKind`（native/plugin/unsupported）。
2. `packages/map/src/maplibre-layer-kinds.ts`：MapLibre 的支持方式。
3. 其他引擎：
   - `cesium-layer-sync.ts` / `packages/map/src/cesium-*`
   - `mapbox-layers.ts`
   - `arcgis-layers.ts`
4. 若由 MapLibre 直接渲染：在 `layer-sync.ts` 的 `syncLayer` switch 增加 case，并命名 source/layer id 辅助函数。
5. 若由插件渲染：在 `packages/plugins` 对应控件中实现，并在 `registerExternalNativeLayer` 注册。

### 1.3 应用层

1. `apps/geolibre-desktop/src/components/layout/AddDataDialog.tsx`：入口与对话框。
2. `src/lib/add-data-renderer.ts`：不同引擎下的选项过滤。
3. `src/components/panels/StylePanel.tsx` / `style-panel/`：可编辑样式。
4. `src/lib/layer-swatch.ts`：图层列表图标。
5. `src/lib/auto-legend.ts`：图例条目。
6. `src/lib/layer-refresh.ts`：是否支持自动刷新。
7. `src/lib/file-dialog-filters.ts`：文件过滤器。

### 1.4 测试与文档

- `tests/layer-support-matrix.test.ts`
- `tests/add-tile-layer.test.ts` 或新文件
- 项目往返：`tests/project-*.test.ts`
- `docs/project-format.md`、`docs/data-formats.md`
- E2E：`e2e/*.spec.ts`（如需要）

---

## 2. 新增插件

### 2.1 内置插件

1. 在 `packages/plugins/src/plugins/<id>.ts` 实现：

```ts
export const myPlugin: GeoLibrePlugin = {
  id: "my-plugin",
  name: "My Plugin",
  version: "1.0.0",
  engines: ["maplibre", "cesium"],   // 按需
  activate(app) {
    const control = new MyControl(app);
    app.addMapControl(control, "top-right");
    return () => app.removeMapControl(control);
  },
  deactivate(app) {
    /* 清理控件、事件、面板注册 */
  },
};
```

2. 在 `packages/plugins/src/index.ts` 导出。
3. 在 `apps/geolibre-desktop/src/hooks/usePlugins.ts` 的 `BUILT_IN_PLUGINS` 注册（顺序 = 菜单顺序）。
4. 需要稳定 id 时加入 `plugin-ids.ts`；需要 viewer 模式时加入 `viewer-plugins.ts`。
5. 实现 `getProjectState` / `applyProjectState`（若插件有项目状态）。
6. 若支持多引擎，分支使用 `app.getMapRenderer()` + 能力查询 / `getCesiumScene()` / `getArcgisControlMap()`。
7. 若注册右侧面板/菜单/浮窗/Assistant 工具，确保停用时由宿主自动回收或手动清理。
8. 添加测试 `tests/plugin-*.test.ts`；运行 `npm run plugins:docs` 更新参考。

### 2.2 外部插件

外部插件是独立 JS bundle + `plugin.json`：

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "main": "plugin.js",
  "engines": ["maplibre"]
}
```

打包为 zip 后从 Manage Plugins 安装；或部署到 `public/plugins/<id>/` 随构建烘焙。注意：

- 安装受部署策略、黑名单、完整性 pin 约束。
- 插件不能读取已保存凭据（加载前 `secure_store_seal`）。
- 插件只能写声明的 URL 参数。
- 外部插件不应强制 `activeByDefault`。

---

## 3. 新增地理处理工具

1. `packages/processing/src/<area>-tools.ts` 定义 `ProcessingAlgorithm`（参数、默认值、`visibleWhen`、`group`）。
2. 实现 `run(ctx)`：
   - 从 `ctx.parameters` 读取，
   - 使用 `ctx.log`，
   - 通过 `ctx.addResultLayer(name, geojson, options)` 返回结果，
   - 尊重 `ctx.signal`。
3. 注册到 `VECTOR_TOOLS` / `RASTER_TOOLS` / `STATISTICS_TOOLS` 等数组。
4. 如需 Worker：加入 `vector-tool-runner.ts` 的 `WORKER_VECTOR_TOOL_IDS`，并确保无 DOM/SDK 依赖。
5. 如需 sidecar：标记 `supportsSidecar` / `requiresSidecar`，在 `backend/geolibre_server/geolibre_server/app/*.py` 增加端点，并在 `sidecar-client.ts` 封装。
6. 若用 DuckDB：通过 `ctx.duckdb` 能力，不要直接 import duckdb-wasm。
7. 若结果需要特定样式：`ResultLayerOptions`。
8. UI：`VectorToolsDialog` / `RasterToolsDialog` / `ProcessingDialog` 会自动读取注册表；如需自定义面板，在 `components/processing/` 实现。
9. 文档：`docs/user-guide/processing.md`。
10. i18n：`npm run i18n:tools` 生成工具目录；不要手改生成文件。
11. 测试：`tests/<tool>.test.ts`。

---

## 4. 新增渲染引擎（高成本）

新增第五个引擎需要完成：

1. `MapEngine` 实现类（新文件 `<engine>-engine.ts`），实现全部成员；不支持的返回 `null`/`false`/no-op。
2. `MapEngineCapabilities` 对象（frozen），列出真实能力。
3. Canvas 组件（`<Engine>Canvas.tsx`），负责 `init`、订阅 store、销毁。
4. `rendererCapabilities` 增加分支。
5. `layer-kind.ts` 与新引擎的 `*_SUPPORTED_LAYER_KINDS`。
6. 图层同步适配器（`<engine>-layer-sync.ts` 或等价）。
7. 底图/地形/相机/CRS/捕获适配。
8. `MapGrid` / `DesktopShell` 的挂载分支。
9. 插件 `engines` 声明与能力分支。
10. `tests/layer-support-matrix.test.ts` 与引擎专属测试。
11. ESLint `no-cross-engine-imports` 的文件命名分类（若前缀新引擎，需要更新规则）。

优先考虑：能否通过现有引擎的能力组合实现，而不是新增引擎。

---

## 5. 新增面板或对话框

### 面板

1. `apps/geolibre-desktop/src/components/panels/<Name>.tsx`。
2. 用 `useLayer` / `useLayerIds` / `useLayerSummaries` 做窄订阅。
3. 若面板较重，使用 `MountWhenOpened` 或懒加载。
4. 在 `DesktopShell` / `RightDockPortals` / `useRightPanels` 注册。
5. 若为插件面板，通过 `app.registerRightPanel` 而不是直接改 Shell。

### 对话框

1. `components/layout/<Name>Dialog.tsx`。
2. 在 `store/ui-slice.ts` 增加开关字段与 action。
3. 在 `DesktopShell` 中挂载；若为大型处理对话框，加入 `desktopShellLazyPanels.ts`。
4. 使用 `@geolibre/ui` 的 Dialog/Input/Select 等。

---

## 6. 新增 Tauri 命令

1. `apps/geolibre-desktop/src-tauri/src/<module>.rs` 实现 `#[tauri::command]`。
2. 在 `lib.rs` 的 `generate_handler![]` 注册。
3. 在 `src-tauri/capabilities/` 增加权限（如需要）。
4. `mas` 构建下若命令会启动外部进程或下载代码，提供返回错误的桩。
5. 前端封装到 `src/lib/<name>.ts`，不要在各组件里直接 `invoke`。
6. 添加 Rust 侧测试或前端 mock 测试；运行 `npm run check:rust`。
7. 若涉及网络，优先复用 `guarded_http_client`（OS 信任库 + mTLS）。

---

## 7. 新增 sidecar 端点

1. 在 `backend/geolibre_server/geolibre_server/app/<router>.py` 增加路由。
2. 若涉及文件路径，使用现有的路径白名单校验（`GEOLIBRE_CONVERSION_ROOTS`）。
3. 若涉及数据库，使用 `db_common.py` 的主机白名单。
4. 若为长任务，复用 job store 与 `GET /jobs/{id}`。
5. 在 `packages/processing/src/sidecar-client.ts` 增加客户端方法。
6. 前端探测 `/status`，不可用时降级。
7. 添加 pytest；更新 `docs/server-api.md` 或 sidecar 文档。

---

## 8. 新增 MCP 工具

1. 在 `python/src/geolibre/mcp/server.py` 的 `build_server` 内定义工具。
2. 复用 `authoring.py`，不要在 MCP 层重复项目逻辑。
3. 所有路径经 `Workspace` 校验。
4. 写操作限制 `.json`/`.html`，默认不覆盖。
5. 若涉及网络，使用 `network.py` 的授权与公共 IP 校验。
6. 若需要预览，更新 `apps/geolibre-mcp-app`。
7. 添加 `python/tests/test_mcp_*.py`；无 SDK 时 skip 的处理保持。

---

## 9. 新增用户可见字符串

1. 在 `apps/geolibre-desktop/src/i18n/locales/en.json` 添加 key。
2. 代码中使用 `t("namespace.key")`（React）或 `i18n.t(...)`（非 React）。
3. 使用逻辑方向工具类（`ms-`/`me-`/`text-start`…）。
4. 运行类型检查（`i18next.d.ts` 会校验 key）。
5. 其他语言可后补，缺失回退英文。
6. 若为处理工具名/描述，运行 `npm run i18n:tools`。

---

## 10. 新增格式导入

1. 在 `src/lib/<format>.ts` 实现解析/读取。
2. 统一输出 `FeatureCollection` 或源描述。
3. 在 `AddDataDialog` 增加入口与文件过滤器。
4. 在 `useFileDrop` 识别扩展名。
5. 在 `file-dialog-filters.ts` 增加过滤器。
6. 若需要 DuckDB，使用 `duckdb-vector-loader.ts` 的既有路径。
7. 若需要 CRS，使用 `datum-shift.ts` / `epsg-proj4.ts`。
8. 添加测试 fixture 与 `tests/<format>.test.ts`。
9. 更新 `docs/data-formats.md` 与 `docs/user-guide/adding-data.md`。

---

## 11. 代码约定速查

| 约定 | 说明 |
| --- | --- |
| 不直接改 `node_modules` | MapLibre 控件样式覆盖写在 `src/index.css` |
| 不直接操作地图 | 改 store，让 `syncLayers` 应用 |
| 不按引擎名分支 | 用 `useMapCapabilities` / `rendererCapabilities` |
| 引擎文件不跨 import | 共享逻辑放中立模块 |
| 不读整个 `import.meta.env` | 用 `getBuildEnvironment()` / `getRuntimeEnvironment()` |
| 不新增未列入 `BUILD_ENV_KEYS` 的构建变量 | 会被 `pruneBuildEnv` 删除 |
| 不在组件里 `invoke` | 封装到 `lib/` |
| 不手改生成文件 | 用脚本重新生成 |
| 不提交 `main` | 分支 + PR |
| 不加无消费者的 capability | 每个 flag 都要有真实分支 |

---

## 12. 调试技巧

| 问题 | 工具 |
| --- | --- |
| 图层未显示 | 查看 Diagnostics、`map-diagnostic` 事件、`MapEngine.getRenderStatus()` |
| 样式不生效 | 检查 `vector-style.ts` 编译、图层 id、`beforeId` 排序 |
| Cesium 不同步 | 检查 `cesium-layer-sync.ts`、`cesium-camera.ts` 的容差 |
| DuckDB 报错 | `ensureSpatialExtension` 是否失败、扩展路径、CORS |
| sidecar 不可用 | `GET /status`、token、stderr、`uv` 环境 |
| 插件未激活 | `engines`、部署策略、黑名单、信任对话框 |
| E2E 不稳定 | 单 worker 跑 `core-engines`；检查 token 清空 |
| 包体超预算 | `npm run bundle:report`、`boot-budget.json` |
| 权限被拒 | `useAppCapability` 的 `reason`、`deployment.json` |

---

## 13. 提交前自检

- [ ] `npm run lint`
- [ ] `npm run check:shared-deps`
- [ ] `npm run typecheck:fast`
- [ ] `npm run typecheck:tests`
- [ ] `npm run test:frontend`
- [ ] 涉及后端：`npm run test:backend`
- [ ] 涉及 Rust：`npm run check:rust`
- [ ] 涉及生成物：`npm run i18n:tools:check`、`npm run gallery:check`、`npm run plugins:docs:check`
- [ ] 涉及 CSP：两处同时更新
- [ ] 涉及依赖：两个 manifest 同范围 + lockfile
- [ ] 文档：更新对应 `docs/` 页面
- [ ] `pre-commit run --files <改动文件>`

返回 [Code Wiki 首页](README.md)。
