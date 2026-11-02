# 11 Workers 与协作

`workers/` 托管 GeoLibre 的五个边缘/服务端组件。它们都不是地图渲染所必需，但为公开站点、行星底图、协作与 AI 提供支撑。

| Worker | 域名（默认） | 运行时 | 作用 |
| --- | --- | --- | --- |
| `viewer` | `viewer.geolibre.app` | Cloudflare Worker | 反向代理 GitHub Pages demo |
| `tiles` | `tiles.geolibre.app` | Cloudflare Worker | 行星底图 CORS/重投影 + 多源数据代理 |
| `collab` | `collab.geolibre.app` | Worker + Durable Object | 协作会话中继 |
| `collab-node` | 自托管 `:8787` | Node.js + ws + SQLite | 协作中继的自托管版本 |
| `ai-proxy` | 部署方指定 | Cloudflare Worker | OpenAI 兼容 AI 端点 + 搜索代理 |

## 1. `workers/viewer`

文件：`src/index.ts`、`src/proxy.ts`。

背景：

- `viewer.geolibre.app` 是旧别名。
- 实际构建位于 `https://geolibre.app/demo`（GitHub Pages）。
- demo 内含约 32 MiB 的 DuckDB WASM，超过 Cloudflare Workers/Pages 每资源 25 MiB 限制，因此不能直接托管。

行为：

```text
viewer.geolibre.app/<path>?<query>
  → https://geolibre.app/demo/<path>?<query>
```

安全：

- 仅跟随 `https://geolibre.app/demo/…` 前缀内的重定向。
- 跨源或越界重定向一律拒绝，避免成为开放代理。

部署：`.github/workflows/deploy-viewer.yml`。

## 2. `workers/tiles`

文件：`src/index.ts`、`allowlisted-fetch.ts`、`earthdata.ts`、`odp.ts`、`overpass-query.ts`、`reproject.ts`。

### 2.1 行星底图

| 路由 | 作用 |
| --- | --- |
| `/opm/<dataset>/<z>/<x>/<y>.png` | 代理 OpenPlanetaryMap 的 S3 栅格（Mars、Moon） |
| `/wms/<dataset>/<z>/<x>/<y>.png` | 请求 USGS Astrogeology WMS（EPSG:4326），重投影为 Web Mercator，编码 PNG 并加 CORS |

为什么需要：

- MapLibre 用 `fetch()` 取瓦片，受 CORS 限制；OPM 的 S3 桶不返回 `Access-Control-Allow-Origin`，浏览器会渲染黑图。
- USGS 行星 WMS 只提供 EPSG:4326，MapLibre 不能直接消费。

实现要点：

- `wms/<dataset>` 的 `z/x/y` 用整数约束。
- 每块瓦片只计算一次，之后由 Cloudflare 边缘缓存。
- `reproject.ts` 负责等距圆柱 → Web Mercator 的像素重采样。
- `upng-js` 负责 PNG 编解码。

### 2.2 其他受控代理

tiles Worker 同时为插件提供一批**严格 allowlist** 的上游代理，避免浏览器 CORS 与密钥暴露：

| 类别 | 路径示例 |
| --- | --- |
| 卫星/对地观测 | Celestrak、NASA FIRMS、Earthdata 下载、OpenAerialMap 元数据 |
| 航空/船舶 | adsb.lol、ADSBDB、OpenSky、AIS |
| 摄像头 | Calgary / Austin / Ontario / NSW / Caltrans CCTV 目录与帧 |
| 开放数据 | CKAN 搜索、Socrata、Source Cooperative、GitHub raw、Overpass |
| 影像/瓦片 | PMTiles、STAC、地球引擎代理 |
| 发射/天文 | Launch Library、Celestrak |
| ODP | Ocean Data Platform |

所有上游都通过 `allowlisted-fetch.ts` 校验，不接受任意 URL。`overpass-query.ts` 校验 Overpass QL，防止滥用。

部署：`.github/workflows/deploy-tiles.yml`。

## 3. 协作体系

### 3.1 `@geolibre/collab-core`

共享核心，`packages/collab-core/src`：

| 模块 | 内容 |
| --- | --- |
| `protocol.ts` | 角色、模式、参与者、光标、聊天、快照、全部客户端/服务端消息类型 |
| `session.ts` | 会话状态、权限、快照合并、邀请、踢人、层级锁等纯逻辑 |
| `comment-validate.ts` | 评论/回复校验（长度、锚点、作者、频率、数量上限） |
| `identity.ts` | 可选签名身份 token（HMAC） |
| `internal/` | 内部实现细节 |

角色与模式：

```ts
type CollaborationRole = "host" | "guest";
type CollaborationMode = "view-only" | "co-edit";
```

消息（`protocol.ts`）分为客户端 → 服务端与服务端 → 客户端：

| 方向 | 消息 |
| --- | --- |
| C→S | `join`、`client-snapshot`、`presence`（光标/视图）、`set-mode`、`set-participant-mode`、`chat-send`、`comment-mutation`、`mint-invite`、`revoke-invite`、`set-session-config`、`kick-participant`、`block-participant`、`set-layer-locks` |
| S→C | `welcome`（含快照、参与者、presence、聊天、rev）、`snapshot`、`presence`、`participant-*`、`chat`、`comment-*`、`mode`、`invite-*`、`kicked`、`error` 等 |

设计要点：

- **快照 + 增量**：host 上传完整项目快照，guest 加入时获得最新快照；之后双方按 revision 应用编辑。
- **回声抑制**：客户端忽略自己刚发出的变更（`rev` 与服务端广播比对），避免反馈循环。
- **撤销交互**：本地撤销只回退本地编辑，不撤销他人改动。
- **权限**：host 可改模式、踢人、锁图层、铸造邀请；guest 权限受 host 设置限制。
- **大小限制**：`MAX_SNAPSHOT_BYTES` 默认 1,000,000；评论、聊天、参与者数量都有上限。
- **身份可选**：`isIdentityConfigured(secret)` 为真时，加入者需签名 token；否则匿名。

### 3.2 `workers/collab`（Cloudflare）

文件：`src/index.ts`、`src/session.ts`、`src/protocol.ts`、`src/comment-validate.ts`。

路由：

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| POST | `/sessions` | 创建会话，返回 8 位会话码 + host token |
| GET | `/sessions/:id/ws` | WebSocket 升级到 Durable Object |
| GET | `/sessions/:id/log` | host-only 会话日志 |
| DELETE | `/sessions/:id/log` | host-only 清空日志 |

实现：

- 会话码使用无歧义 base32 字母表（去掉 `0/1/O/I`），长度 8。
- `idFromName(code)` 把同一会话的所有参与者路由到同一 Durable Object。
- Durable Object `CollabSession` 持有参与者、快照、聊天、评论、锁、速率限制。
- CORS 允许 `GET/POST/DELETE/OPTIONS`。
- 每个客户端有速率限制（`MAX_RATE_LIMIT_KEYS = 10_000` 防内存膨胀）。

部署：`.github/workflows/deploy-collab.yml`。

### 3.3 `workers/collab-node`（自托管）

文件：`src/server.ts`、`src/store.ts`。

- 使用 Node `http` + `ws`，同一套 `@geolibre/collab-core` 策略。
- SQLite 持久化（`COLLAB_DB_PATH`）。
- 端点：`POST /sessions`、`GET /sessions/:id/ws`、`GET /health`。
- 环境变量：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `PORT` | `8787` | HTTP/WS 端口 |
| `HOST` | `0.0.0.0` | 监听地址 |
| `COLLAB_DB_PATH` | `./data/collab.sqlite` | SQLite 文件 |
| `COLLAB_MAX_SNAPSHOT_BYTES` | `1000000` | 快照上限 |
| `COLLAB_IDLE_TTL_MS` | `7200000` | 最后一人离开后会话存活时间 |

Docker：`workers/collab-node/Dockerfile`；Compose 服务 `geolibre-collab`。TLS 由入口反向代理终止，浏览器使用 `wss://`。

冒烟测试：

```bash
node workers/collab-node/scripts/smoke.mjs http://127.0.0.1:8787
```

### 3.4 前端接入

| 模块 | 作用 |
| --- | --- |
| `apps/geolibre-desktop/src/lib/collab-client.ts` | WebSocket 客户端、重连、消息编解码 |
| `src/lib/collab-protocol.ts` | 应用侧消息封装 |
| `src/hooks/useCollaboration.ts` | React 状态与 store 同步 |
| `src/components/layout/CollaborateDialog.tsx` | 创建/加入会话 |
| `src/components/layout/RemoteCursorsOverlay.tsx` | 远端光标 |
| `src/components/layout/CollaborationStatusBadge.tsx` | 状态 |
| `src/components/layout/CollaborationParticipantRow.tsx` | 参与者列表 |
| `src/lib/collaboration-layers.ts` / `collaboration-project.ts` | 图层/项目级合并 |

服务地址来自 `VITE_GEOLIBRE_COLLAB_URL`（Web 构建）或设备设置；未配置时功能关闭。项目文件不会携带协作会话，会话是运行时的。

## 4. `workers/ai-proxy`

文件：`src/index.ts`。

### 4.1 路由

| 路径 | 作用 |
| --- | --- |
| `GET /health` | 健康检查 |
| `POST /v1/chat/completions` | OpenAI 兼容聊天补全，流式 |
| `POST /search` | 通过 Anthropic-messages 兼容上游做 GPT 原生网页搜索 |
| `POST /tavily` | Tavily 搜索（显式选择时） |
| `POST /systemone` | TypeSafe System One 快速路径 |

### 4.2 设计

- 通过 Cloudflare AI Gateway Unified API 路由到 OpenAI / Anthropic / Gemini / Workers AI，**前端不携带供应商密钥**。
- 需要服务端实例 token（`GEOLIBRE_AI_PROXY_TOKEN`）才能使用。
- 防护：模型白名单、请求体上限、输出 token 上限、每客户端速率限制（所有路由共享一个预算）。
- 超时：聊天上游头 300 秒；Tavily 搜索头 60 秒。
- 可选搜索密钥：`SEARCH_MESSAGES_URL` + `SEARCH_MESSAGES_API_KEY`；`TAVILY_API_KEY`；`JEV_API_KEY`。未配置时对应路由返回 503，聊天仍可用。

## 5. 部署与配置

| 组件 | 部署方式 | 工作流 |
| --- | --- | --- |
| viewer | `wrangler deploy` | `deploy-viewer.yml` |
| tiles | `wrangler deploy` | `deploy-tiles.yml` |
| collab | `wrangler deploy`（Durable Object） | `deploy-collab.yml` |
| collab-node | Docker 镜像 / Compose | `ci.yml` 构建 + 测试 |
| ai-proxy | `wrangler deploy` + secrets | 手动/wrangler |
| 主 Web 应用 | GitHub Pages / Cloudflare Pages / Docker | `pages.yml`、`web-deploy.yml`、`publish-container.yml` |

前端通过构建/运行时变量连接：

- `VITE_GEOLIBRE_SHARE_URL` / `GEOLIBRE_SHARE_URL`：公共 Server API
- `VITE_GEOLIBRE_COLLAB_URL` / `GEOLIBRE_COLLAB_URL`：协作中继
- `GEOLIBRE_AI_URL` / `GEOLIBRE_AI_PROXY_URL` / `GEOLIBRE_AI_PROXY_TOKEN`：AI 代理（Docker 下必须同时设置，且 `GEOLIBRE_AI_URL` 必须是同源 `/ai`）

## 6. 测试

| 组件 | 测试 |
| --- | --- |
| collab-core | `tests/collab-*.test.ts` |
| collab worker | `npm run typecheck -w geolibre-collab-worker` |
| collab-node | `npm test -w geolibre-collab-node` + `scripts/smoke.mjs` |
| tiles | `npm run typecheck -w geolibre-tiles-worker` |
| viewer | `npm run typecheck -w geolibre-viewer-worker` |
| ai-proxy | `npm run typecheck -w geolibre-ai-proxy-worker`，`tests/ai-proxy-*.test.ts` |
| 全量 | `npm run test:worker` |

下一章：[12 数据流与格式](12-data-flow.md)。
