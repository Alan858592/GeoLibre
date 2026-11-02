# 09 Python 后端

仓库包含两个**完全独立**的 Python 服务，职责与信任边界不同：

| 服务 | 目录 | 面向 | 默认绑定 | 是否可选 |
| --- | --- | --- | --- | --- |
| 处理 sidecar | `backend/geolibre_server` | 单用户本机处理 | `127.0.0.1:8765` | 是（浏览器 WASM 可替代大部分） |
| 公共 Server API | `backend/geolibre_server_api` | 多用户项目分享/身份 | `0.0.0.0:8000` | 是（仅分享、协作、组织功能需要） |

两者都不参与地图渲染，也不托管前端。

---

## 第一部分：处理 sidecar

### 1. 定位

`backend/geolibre_server` 是桌面应用按需启动的 FastAPI 服务，为以下功能提供 Python 引擎：

- 格式转换（GeoParquet、COG、FlatGeobuf、GPKG、Shapefile、PMTiles、CSV）
- 栅格工具（rasterio / numpy / contourpy）
- 可选 GeoPandas 矢量引擎
- Apache Sedona SQL（SedonaDB）
- PostGIS / SQL Server 可编辑图层读写
- Whitebox 工具箱（可选替代引擎）
- 点云标注写回
- AI 分割代理（SamGeo/SAM3）
- OBIA 原生分割与测量

包配置：`backend/geolibre_server/pyproject.toml`

- 包名 `geolibre-server`，版本 `1.0.0`
- Python >= 3.10
- 基础依赖：`fastapi>=0.115`、`uvicorn[standard]>=0.32`
- entry point：`geolibre-server = geolibre_server.app.main:run`
- FastAPI 应用标题版本：`0.8.0`（`app/main.py:64`）

### 2. 入口与应用装配

`geolibre_server/app/main.py`：

```python
app = FastAPI(title="GeoLibre Server", version="0.8.0")
```

| 中间件 | 作用 |
| --- | --- |
| `require_sidecar_token` | 校验 `X-GeoLibre-Token` 或 `Authorization: Bearer`；`/health` 与 OPTIONS 豁免；常量时间比较（按 bytes） |
| `TrustedHostMiddleware` | 只允许 `localhost`、`127.0.0.1`、`testserver`，防 DNS rebinding |
| `CORSMiddleware` | 只允许 `http://localhost:5173`、`http://127.0.0.1:5173`、`tauri://localhost`、`http://tauri.localhost` |

全局端点：

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/health` | 健康检查（免 token） |
| POST | `/shutdown` | 优雅退出；先停止 samgeo 子进程，再发 SIGINT |
| GET | `/algorithms` | 占位算法目录 |
| POST | `/run` | 未实现（501），保留给未来统一调度 |

路由注册（`main.py:121`）：

```python
app.include_router(whitebox_router)
app.include_router(conversion_router)
app.include_router(raster_router)
app.include_router(vector_router)
app.include_router(postgis_router)
app.include_router(mssql_router)
app.include_router(sql_router)
app.include_router(pointcloud_router)
app.include_router(ml_router)
app.include_router(obia_router)
```

### 3. 路由与端点

| 路由前缀 | 文件 | 端点 | 引擎 |
| --- | --- | --- | --- |
| `/whitebox` | `whitebox.py` | `GET /status`、`GET /tools`、`GET /tools/{id}`、`POST /run`、`GET /jobs/{id}`、`GET /output` | `whitebox-workflows` |
| `/conversion` | `conversion.py` | `GET /status`、`POST /vector-layers`、`/vector-to-vector`、`/vector-to-geoparquet`、`/vector-to-flatgeobuf`、`/vector-to-shapefile`、`/vector-to-geopackage`、`/csv-to-geoparquet`、`/vector-to-pmtiles`、`/raster-to-cog`、`GET /jobs/{id}` | duckdb / rio-cogeo / freestiler / GDAL |
| `/raster` | `raster.py` | `GET /status`、`POST /run` | rasterio / numpy / contourpy |
| `/vector` | `vector.py` | `GET /status`、`POST /run`、`POST /write` | GeoPandas / Shapely |
| `/sql` | `sql.py` | `GET /status`、`POST /run` | Apache Sedona（SedonaDB） |
| `/postgis` | `postgis.py` | `GET /status`、`POST /tables`、`/read`、`/write` | psycopg |
| `/mssql` | `mssql.py` | `GET /status`、`POST /connect`、`/disconnect`、`/tables`、`/read`、`/write` | pyodbc / azure-identity / shapely / pyproj |
| `/pointcloud` | `pointcloud.py` | `GET /status`、`POST /apply-labels` | laspy / lazrs |
| `/ml` | `ml.py` | `GET /status`、`GET /models`、`POST /segment/automatic`、`/segment/predict`、`/segment/text` | 反向代理到 `samgeo-api` |
| `/obia` | `obia.py` | `GET /status`、`POST /segment`、`/measure`、`GET /jobs/{id}/files/{name}`、`POST /jobs/{id}/cancel` | scikit-image / scipy / rasterio |

转换与栅格任务共享同一 job store、runtime 与路径白名单（`raster.py` 复用 `conversion.py` 的 helper）。

### 4. 安全管理

| 机制 | 实现 |
| --- | --- |
| 每会话 token | 桌面端 Rust 生成；Docker entrypoint 生成；未设置时跳过（仅本地开发/pytest） |
| 路径白名单 | `GEOLIBRE_CONVERSION_ROOTS`（`os.pathsep` 分隔）；读写都必须位于这些根下 |
| 数据库白名单 | `GEOLIBRE_POSTGIS_HOSTS`、`GEOLIBRE_MSSQL_HOSTS`（精确 host[:port] 匹配，`*` 表示任意） |
| Host 限制 | `TrustedHostMiddleware` 只允许回环 |
| CORS 限制 | 只允许 Vite 与 Tauri 源 |
| token 比较 | `hmac.compare_digest`（按 UTF-8 bytes，避免非 ASCII 头导致 500） |
| Docker 默认 | sidecar 通过 nginx `/sidecar` 同源代理；端口不对外暴露 |

Mac App Store 构建不启动 sidecar（App Sandbox 禁止子进程），前端通过 `IS_MAS_BUILD` 提前给出提示。

### 5. 可选 extras

`pyproject.toml` 的 `[project.optional-dependencies]`：

| extra | 内容 | 启用功能 |
| --- | --- | --- |
| `whitebox` | `whitebox-workflows>=2.0.2` | Whitebox 服务端引擎 |
| `conversion` | `duckdb`、`rio-cogeo`、`freestiler` | 格式转换、COG |
| `vector` | `geopandas`、`shapely` | 投影感知矢量工具 |
| `raster` | `rasterio`、`numpy`、`contourpy` | 栅格工具 |
| `postgis` | `psycopg[binary]` | PostGIS 可编辑图层 |
| `mssql` | `pyodbc`、`azure-identity`、`shapely`、`pyproj` | SQL Server / Azure SQL |
| `sedona` | `apache-sedona[db]`、`geopandas`、`shapely` | Sedona SQL |
| `ml` | `httpx` | AI 分割代理 |
| `obia` | `scikit-image`、`scipy`、`rasterio`、`numpy` | OBIA |
| `pointcloud` | `laspy[lazrs]`、`numpy` | 点云标签写回 |
| `notebook` | `jupyterlab`、`jupyter-server`、`pip` | 桌面 Notebook 面板 |
| `test` | 上述全部 + pytest | 完整测试 |

缺失 extra 时：

- `/vector/status` 报告不可用，前端回退 Turf.js。
- `/sql/status` 报告不可用，前端回退 CereusDB。
- 转换/栅格的部分端点返回明确的 501/400，而不是崩溃。

### 6. 与前端共享的几何代码

`backend/geolibre_server/geolibre_server/vector_ops.py`：

- 无框架依赖的 GeoPandas/Shapely 操作模块。
- sidecar 的 `/vector/run` 调用它。
- 浏览器 Pyodide 也调用同一文件（由 `vite-plugins/copy-vector-ops.ts` 复制进前端），保证两个引擎结果一致。

`sedona_ops.py` 同理封装 SedonaDB 的 SQL 执行，供 `/sql/run` 与浏览器 CereusDB 路径参考。

### 7. 运行

```bash
# 本地开发（无 token 校验，回环绑定）
pip install -e "backend/geolibre_server[test]"
geolibre-server            # uvicorn 127.0.0.1:8765, reload=True

# 只跑测试
python -m pytest backend/geolibre_server/tests
```

桌面端不直接运行它；Tauri 的 `start_geolibre_sidecar` 会准备 uv 管理的环境并按需启动。Docker 镜像内由 `docker/entrypoint.sh` 根据部署策略决定是否启动，并在 nginx 挂到 `/sidecar`。

---

## 第二部分：公共 Server API

### 8. 定位

`backend/geolibre_server_api` 是 GeoLibre 的**多用户**参考服务，实现 `docs/server-api.md` 描述的协议，用于：

- 项目云端保存、版本、可见性、Fork、活动日志
- 账号、会话、个人 API token
- OAuth 2.1（PKCE）授权服务器
- 组织、成员、邀请、组
- 企业单点登录（OIDC、可信代理头、SCIM 2.0）
- 对象存储（本地目录或 S3）

它**不是**桌面处理 sidecar 的替代品，也不执行地理处理。

### 9. 模块

| 模块 | 作用 |
| --- | --- |
| `main.py` | 应用工厂 `build_app()`、中间件、全部项目/组织路由、限流与错误处理 |
| `auth.py` | 账号、登录、token、会话、OAuth 授权/令牌、PKCE |
| `auth_models.py` | SQLAlchemy 账号/会话/token 模型 |
| `project_models.py` | 项目、版本、活动模型 |
| `org_models.py` | 组织、成员、邀请、组模型 |
| `enterprise_models.py` | 安全策略、身份提供商、SCIM token 模型 |
| `enterprise_admin.py` | 组织安全策略、IdP、SCIM token 管理端点 |
| `scim.py` | SCIM 2.0 Users/Groups/ServiceProviderConfig/ResourceTypes/Schemas |
| `oidc.py` | OIDC 发现、授权码、token 校验、JWKS |
| `proxy_identity.py` | 可信代理头身份 |
| `projects.py` | 项目活动记录、匿名访问聚合、保留期 |
| `egress.py` | 出站网络限制（SSRF 防护） |
| `policy.py` | 组织策略、IP 允许列表等 |

### 10. 认证与授权

| 体系 | 端点 | 说明 |
| --- | --- | --- |
| 账号 | `POST /api/accounts`、`GET/PATCH /api/account`、`POST /api/account/password`、`GET /api/users/me` | 注册、资料、密码 |
| 登录 | `POST /api/auth/token`、`DELETE /api/auth/token` | 换取/撤销访问 token |
| 会话 | `GET /api/auth/sessions`、`DELETE /api/auth/sessions/{id}`、`POST /api/auth/sessions/revoke-others` | 设备会话管理 |
| OAuth | `GET/POST /oauth/authorize`、`GET /oauth/sso/callback`、`POST /oauth/token`、`POST /oauth/revoke` | PKCE 公共客户端；作用域 `read:projects`、`write:projects`、`share:public`、`manage:sessions` |
| 组织 | `/api/organizations`、`/members`、`/invitations`、`/projects` | 组织 CRUD、成员、邀请 |
| 组 | `/api/groups` 等 | 组与共享 |
| 企业 | `/api/organizations/{id}/security-policy`、`/identity-provider`、`/scim-tokens` | 安全策略、IdP、SCIM token |
| SCIM | `/ServiceProviderConfig`、`/ResourceTypes`、`/Schemas`、`/Users`、`/Groups` | 标准 SCIM 2.0 |
| 项目 | `POST/GET /api/projects`、`GET/PATCH/DELETE /api/projects/{id}`、`GET /api/projects/{id}/versions`、`PUT /api/projects/{id}/content`、`GET /api/users/{username}/projects`、`/api/projects/{id}/activity` | 项目 CRUD、版本、内容、Fork、活动 |
| 健康 | `GET /health` | — |

Token 类型：

- **访问 token**：短期（默认 600 秒），可刷新。
- **刷新 token**：默认 30 天；轮换时旧 token 立即失效。
- **个人 API token**：长期，受作用域限制。
- **管理授权**（`manage:sessions`）：仅 5 分钟，不可刷新。

### 11. 存储与数据库

| 配置 | 说明 |
| --- | --- |
| `GEOLIBRE_DATABASE_URL` | SQLAlchemy URL；默认 SQLite；生产建议 PostgreSQL（`postgresql+psycopg://…`） |
| `GEOLIBRE_STORAGE_PATH` | 本地对象目录，默认 `./data` |
| `GEOLIBRE_STORAGE=s3` | 使用 S3 兼容存储 |
| `GEOLIBRE_S3_BUCKET` / `GEOLIBRE_S3_ENDPOINT` / `GEOLIBRE_S3_REGION` | S3 配置 |
| `GEOLIBRE_PUBLIC_URL` | API 外部 URL，同时是 OAuth issuer |
| `GEOLIBRE_VIEWER_URL` | 查看器源 |
| `GEOLIBRE_CORS_ORIGINS` | 允许的浏览器源（OAuth 路由不继承通配符） |
| `GEOLIBRE_OAUTH_CLIENTS` | 公共客户端注册 JSON |
| `GEOLIBRE_MAX_PROJECT_BYTES` / `GEOLIBRE_MAX_THUMBNAIL_BYTES` | 上传上限 |
| `GEOLIBRE_HOST` / `GEOLIBRE_PORT` | 绑定地址，默认 `0.0.0.0:8000` |

启动时会为旧数据库补建 OAuth 查询/过期索引；在已有数据的库上建索引可能持有写锁，升级时应先启动单个实例。

### 12. 安全与运维

- OAuth `/oauth/authorize`、`/oauth/token`、`/api/auth/token`、`/api/accounts` 需要外层限流代理；Compose 中 API 端口绑定到 `127.0.0.1` 以防绕过。
- OIDC 用户提供的 IdP URL 受 `egress.py` 的出站限制：默认拒绝回环/私有/链路本地/CGNAT/多播/保留地址；`GEOLIBRE_OIDC_ALLOWED_NETWORKS` 可放行内网 IdP。
- 自定义 CA：`GEOLIBRE_OIDC_CA_BUNDLE`。
- 可信代理：`GEOLIBRE_TRUSTED_PROXIES` + `GEOLIBRE_PROXY_AUTH` + 头名配置。
- 活动日志默认保留 90 天（`GEOLIBRE_ACTIVITY_RETENTION_DAYS`），匿名 open/fetch 按天聚合。
- 项目可见性：private / unlisted / public；公开项目可被匿名读取与 Fork。

### 13. 运行

```bash
cd backend/geolibre_server_api
pip install -e ".[test]"
geolibre-server-api            # 默认 0.0.0.0:8000
python -m pytest               # 测试
```

生产推荐 `docker-compose.yml`：

```bash
POSTGRES_PASSWORD=... docker compose up -d
```

它会启动：

| 服务 | 端口 | 说明 |
| --- | --- | --- |
| `geolibre-web` | 8080 | nginx + 前端构建 + `/sidecar` |
| `geolibre-server` | 127.0.0.1:8000 | 公共 API |
| `geolibre-collab` | 8787 | Node 协作中继 |
| `postgres` | 内部 | 17-alpine |

### 14. 测试

| 测试 | 命令 |
| --- | --- |
| sidecar | `npm run test:backend`（`python -m pytest backend/geolibre_server/tests`） |
| sidecar 覆盖率 | `npm run test:backend:coverage`（下限 55%） |
| Docker 策略 | `npm run test:docker`（`python -m pytest docker/tests`） |
| 公共 API | `backend/geolibre_server_api/tests/`（CI 的 identity 任务） |

> 注意：只安装 `[dev]` 而不安装 `[test]` 时，矢量/栅格/SQL/ML 测试会自行 skip，CI 会显示绿色但实际未覆盖。CI 使用 `pip install -e "backend/geolibre_server[test]"`。

下一章：[10 Python 包与 MCP](10-python-and-mcp.md)。
