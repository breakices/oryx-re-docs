# Oryx-RE 部署

单机 docker-compose 部署。包含 FastAPI 后端、PostgreSQL、ClickHouse、nginx 静态前端、host 上的 sandbox docker 镜像。

数据分两层：
- **业务数据**（user / session / message / memory / bg_task ...）走 pg
- **Trace 数据**（每 turn 的完整事件 + 历史快照）走 CH，90 天 TTL，列式压缩
- 本地 dev：不设 `DATABASE_URL` fallback SQLite，不设 `ORYX_TRACE_BACKEND` fallback file

## 前置

- Linux host（macOS 也能跑，但 docker.sock 行为略有差异）
- Docker Engine + Compose v2
- Host 上对 `/var/lib/oryx/`（或自定义路径）有读写权限
- 域名 DNS 已指向本机 + TLS 证书（fullchain.pem + privkey.pem 两文件）
- DeepSeek API key；如要 web 检索还需 Tavily key

## 首次部署

```bash
# 1. 拉代码
git clone <repo> oryx-re && cd oryx-re

# 2. 准备环境变量
cp .env.example .env
$EDITOR .env
#    至少填:
#      DEEPSEEK_API_KEY          — LLM 调用
#      DEFAULT_USER_PASSWORD     — 不填会用代码默认值 + 启动 WARN
#      ORYX_PG_PASSWORD          — postgres 密码（强口令）
#      ORYX_CH_PASSWORD          — clickhouse 密码（强口令）
#      ORYX_CERT_FULLCHAIN       — host 上证书 bundle 绝对路径
#      ORYX_CERT_KEY             — host 上私钥绝对路径
#      ORYX_CORS_ORIGINS         — https://yourdomain.tld

# 3. 准备 host 数据目录（compose 里 bind-mount 的源）
sudo mkdir -p /var/lib/oryx/workspaces /var/lib/oryx/uv_cache \
              /var/lib/oryx/pg /var/lib/oryx/ch
sudo chown -R $USER:$USER /var/lib/oryx

# 4. 确认证书文件存在（.env 里直接写两个绝对路径，文件名任意）
ls -l \
  "$(grep '^ORYX_CERT_FULLCHAIN=' .env | cut -d= -f2)" \
  "$(grep '^ORYX_CERT_KEY=' .env | cut -d= -f2)"

# 5. 构建 sandbox base image（一次性，~5-10min）
docker buildx build --tag oryx/sandbox:latest backend/sandbox-image

# 6. 起服务
docker compose up -d

# 7. 拿首个 admin 邀请码（一次性，注册完即作废）
docker compose logs backend | grep bootstrap_invite=
```

浏览器打开 `https://yourdomain.tld/`，切到「注册」tab，邮箱 + 强密码 + 上一步拿到的邀请码完成首个 admin 账户。后续邀请码在「账户与配置」→「邀请码」tab 里生成。

> ⚠️ 默认 admin 账户 (`DEFAULT_USER_EMAIL` 指定的邮箱) 也会自动创建，密码 = `DEFAULT_USER_PASSWORD`。如果你在 `.env` 里没显式设密码，会用代码默认值 `oryx-dev-2026` —— 启动日志会打一条 `auth.default_password_in_use` WARN 提醒。**对外暴露之前必须改掉**。

## `.env` 关键配置

| Key | 必填 | 说明 |
|---|---|---|
| `DEEPSEEK_API_KEY` | ✅ | LLM 调用；用户也可以在自己的「账户与配置」里覆盖私钥（admin 限定） |
| `DEEPSEEK_MODEL` |  | 默认 `deepseek-v4-pro` |
| `DEEPSEEK_LIGHT_MODEL` |  | 默认 `deepseek-chat`；用于 triage / 记忆排序 / 内容摘要 |
| `TAVILY_API_KEY` |  | 留空则 web 工具返回 `disabled=true`；agent 会如实告知用户「无法联网」 |
| `AUTH_ENABLED` |  | 默认 `true`；本地单用户调试可置 `false` 跳过登录 |
| `DEFAULT_USER_EMAIL` | ✅ | 内置 admin 账户邮箱 |
| `DEFAULT_USER_PASSWORD` | ✅ | 内置 admin 密码；不设 = 用代码默认值（不安全） |
| `BOOTSTRAP_INVITE` |  | 首启动若无 invite 时生成的一次性 admin 码；`auto` = 随机，或指定 6 位 A-Z+2-9 |
| `ORYX_HOST_WORKSPACE_ROOT` |  | 默认 `/var/lib/oryx/workspaces`；host = container 路径（沙箱挂载必需） |
| `ORYX_HOST_UV_CACHE` |  | 默认 `/var/lib/oryx/uv_cache`；同上 |
| `ORYX_PG_DB` |  | 默认 `oryx` |
| `ORYX_PG_USER` |  | 默认 `oryx` |
| `ORYX_PG_PASSWORD` | ✅ | postgres 密码；强口令 |
| `ORYX_HOST_PG_DATA` |  | 默认 `/var/lib/oryx/pg`；postgres 数据 host 路径 |
| `DATABASE_URL` |  | compose 自动注入 `postgresql+asyncpg://...`；本地 dev 留空 fallback sqlite |
| `ORYX_CH_DB` |  | 默认 `oryx` |
| `ORYX_CH_USER` |  | 默认 `oryx` |
| `ORYX_CH_PASSWORD` | ✅ | clickhouse 密码；强口令 |
| `ORYX_HOST_CH_DATA` |  | 默认 `/var/lib/oryx/ch`；clickhouse 数据 host 路径 |
| `ORYX_TRACE_BACKEND` |  | `clickhouse`（compose 自动）or `file`（本地 dev 默认） |
| `ORYX_CERT_FULLCHAIN` | ✅ | Host 上证书 bundle 绝对路径（含中间 chain 的 PEM）；CA 给啥文件名都行 |
| `ORYX_CERT_KEY` | ✅ | Host 上私钥绝对路径；同上 |
| `ORYX_CORS_ORIGINS` | ✅ | 浏览器访问的 URL origin，逗号分隔；要和地址栏完全一致（含 https）。例：`https://app.example.com` |

### 金融 Provider 配置

金融模式默认可用；provider 独立开关。未启用或缺 key 时工具会返回 `disabled` / `missing_config` / `degraded`，不会阻断本地或单机部署。

| Key | 默认 | 说明 |
|---|---|---|
| `FINANCE_PROVIDER_YFINANCE_ENABLED` | `true` | Yahoo Finance 行情；无需 key |
| `FINANCE_PROVIDER_WEB_ENABLED` | `true` | 复用 Tavily web 检索；需要 `TAVILY_API_KEY` 才 ready |
| `FINANCE_PROVIDER_POLYMARKET_ENABLED` | `true` | Polymarket Gamma API；无需 key |
| `FINANCE_PROVIDER_FTSHARE_ENABLED` | `false` | FTShare/market.ft.tech；无需 key，建议验证后再开 |
| `FINANCE_PROVIDER_MX_ENABLED` | `false` | EastMoney/MX；需要 `FINANCE_MX_API_KEY` |
| `FINANCE_PROVIDER_ALPHAPAI_ENABLED` | `false` | AlphaPai；需要 `FINANCE_ALPHAPAI_API_KEY`，可选 `FINANCE_ALPHAPAI_BASE_URL` |
| `FINANCE_PROVIDER_GANGTISE_ENABLED` | `false` | Gangtise；需要 `FINANCE_GANGTISE_ACCESS_KEY` + `FINANCE_GANGTISE_SECRET_KEY` |
| `FINANCE_PROVIDER_IFIND_ENABLED` | `false` | iFinD SDK；需要 `IFIND_USERNAME`、`IFIND_PASSWORD`、`IFIND_SDK_PATH` |

iFinD 在多 pod 下依赖 Redis/state lock 串行同一账号调用；没有 Redis 时只适合本地/单机进程内锁。

## Trace 查询（ClickHouse）

每个 chat turn 一行进 `oryx.turn_traces`。一些常用查询：

```bash
# 最近 20 个 turn,看 token / 工具用量
docker compose exec clickhouse clickhouse-client --user oryx --password "$ORYX_CH_PASSWORD" -q "
  SELECT ts, user_id, status, model, n_iters, n_tools, total_tokens, duration_ms
  FROM oryx.turn_traces
  ORDER BY ts DESC LIMIT 20
  FORMAT Pretty
"

# 单个 session 完整 turn 链
docker compose exec clickhouse clickhouse-client --user oryx --password "$ORYX_CH_PASSWORD" -q "
  SELECT ts, turn_id, status, n_tools, total_tokens
  FROM oryx.turn_traces
  WHERE session_id = 's-xxxxxxxx'
  ORDER BY ts FORMAT PrettyCompact
"

# 取某 trace 的完整 events 流(展开 JSON)
docker compose exec clickhouse clickhouse-client --user oryx --password "$ORYX_CH_PASSWORD" -q "
  SELECT events_json FROM oryx.turn_traces WHERE trace_id = 'xxxxxxxx'
"
```

90 天 TTL 自动清理（`TTL ts + INTERVAL 90 DAY` 在 schema 里）。长期保留特定 trace 单独 `INSERT INTO archive_table SELECT ...`。

**失败回退**: 如果 CH 临时 down，每条 trace 自动落回 `/.traces/<sid>/*.json`（FileSink fallback），且后端日志打 `trace.ch_emit_failed` WARN。CH 恢复后人工对那段时间的文件做一次性回灌即可（直接 `clickhouse-client --query "INSERT INTO oryx.turn_traces FORMAT JSONEachRow" < file.json` 之类，按需要 ad-hoc）。

## 沙箱镜像

Backend 容器通过 bind-mounted docker.sock 调 host 的 docker daemon 拉沙箱跑代码。镜像不在 compose 里 build，必须事先在 host 上构建好：

```bash
docker buildx build --tag oryx/sandbox:latest backend/sandbox-image
```

镜像内预装 `numpy / scipy / pandas / matplotlib / pytest / torch-cpu`。要给沙箱加额外包（用户代码可能要用的），在 host 上预热到 uv cache：

```bash
UV_CACHE_DIR=/var/lib/oryx/uv_cache uv pip install --target /tmp/_discard \
    sympy scikit-learn torchvision
rm -rf /tmp/_discard
```

任何放进 uv_cache 的包，沙箱里都能 `--offline` 装上。沙箱内 `network=none`，未在缓存里的包安装会立刻失败。

## 首次登录后

1. 「账户与配置」→「邀请码」生成更多 invite 给其他用户
2. 配置自己的 DeepSeek key（覆盖 server 默认）→ 「配置」tab
3. 「账户」tab 可查看身份 / 登出

普通用户走「注册」+ 邀请码加入；他们看不到 admin 的 sessions（数据隔离做在 `current_user` 依赖里）。

## 关闭沙箱（不需要代码执行）

如果不打算让 agent 跑代码，沙箱可关：`.env` 里加 `SANDBOX_ENABLED=false`。这样：

- `run_sandbox` 工具调用直接返回 `disabled=true`
- backend 不再依赖 docker.sock —— 但 compose 里 socket mount 留着无害
- 也不需要预 build sandbox image

Web 检索 / 文件操作 / sub-agent 全部正常。

## TLS / 域名

Frontend 容器内的 nginx 直接监听 443，证书从 host bind-mount 进去（`${ORYX_CERT_DIR}` → `/etc/nginx/certs:ro`）。80 端口自动 301 跳 443。

容器内 nginx 期望的文件名：

```
/etc/nginx/certs/fullchain.pem
/etc/nginx/certs/privkey.pem
```

Let's Encrypt 路径（`/etc/letsencrypt/live/<domain>/`）天然匹配；其他 CA 在该目录里 `ln -s <你的名字>.crt fullchain.pem`、`ln -s <你的名字>.key privkey.pem` 就行。

**证书更新**：

```bash
# 直接热重载 nginx（不重启容器；cert 文件已 mount，nginx reload 重读）
docker compose exec frontend nginx -s reload
```

或者 `docker compose restart frontend`，但 reload 不中断现有 SSE 连接。

**端口暴露**：

- `80` / `443` — frontend 容器（公网）
- `127.0.0.1:8000` — backend 容器（仅本机，不对外）。frontend 通过 docker network 反代到 `http://backend:8000/`，公网不需要直连

**附加安全头**（nginx.conf 已开）：

- `Strict-Transport-Security max-age=31536000; includeSubDomains` — 强制浏览器一年内只走 HTTPS
- `X-Frame-Options: DENY`、`X-Content-Type-Options: nosniff`、`Referrer-Policy: strict-origin-when-cross-origin`
- CSP 暂未启用（workspace HTML 预览用 `iframe srcDoc + sandbox=""`，CSP 配错会 break，等用户场景稳定后再加）

**HTTP/2 已启用**（`listen 443 ssl; http2 on;`）。SSE 在 HTTP/2 下走 multiplex，避开浏览器同源 6 连接限制 —— 多个并发 sub-agent / bg drawer 同时订阅没问题。

**速率限制**：`/api/auth/login` 和 `/api/auth/signup` 走 nginx `limit_req_zone`，单 IP 10 req/min + burst 5，超限返回 429。其他 `/api/*`（含 `me` / `logout` / chat / SSE）不限。调整在 `nginx.conf` 顶部的 `limit_req_zone ... rate=10r/m;`。

## 日常运维

**更新部署**：
```bash
git pull
docker buildx build --tag oryx/sandbox:latest backend/sandbox-image   # 仅当 sandbox-image/ 变了
docker compose build
docker compose up -d
```

**查日志**：
```bash
docker compose logs -f backend
docker compose logs -f frontend
```

**备份**：
- DB：`docker run --rm -v oryx-data:/d alpine tar czf - -C /d . > backup-$(date +%F).tar.gz`
- Workspaces：`tar czf workspaces-$(date +%F).tar.gz /var/lib/oryx/workspaces`
- traces (调试用，可丢)：`oryx-traces` volume

**清理沙箱遗留**：
```bash
docker ps -a --filter "name=oryx-sandbox-" --format "{{.ID}}" | xargs -r docker rm -f
```

## 故障排查

| 现象 | 原因 / 排查 |
|---|---|
| 上传文件 413 | `nginx.conf` 默认 `client_max_body_size 50m`；超大文件改这里 |
| nginx 起不来 / `cannot load certificate` | 证书目录里没 `fullchain.pem` 或 `privkey.pem`；symlink 一下 |
| 浏览器报 `NET::ERR_CERT_*` | 证书域名与访问 URL 不一致 / 证书过期 / chain 不完整（fullchain 应含中间证书） |
| 前端 console 报 CORS | `ORYX_CORS_ORIGINS` 没写或不匹配地址栏 origin（含 scheme + host + port） |
| `discover_skills()` 返回空 | backend image 漏了 `COPY skills` —— 重新 `docker compose build backend` |
| `run_sandbox` 报 `docker: command not found` | backend image 里缺 docker CLI —— 重新 build |
| `run_sandbox` 报 `Cannot connect to docker daemon` | docker.sock 没 mount，或 host daemon 没启 |
| sandbox 跑起来但找不到包 | 包不在 uv_cache 里；预热（见「沙箱镜像」） |
| 注册时 invite code 用过了 | `one_shot` 类型只能用一次；admin 在「邀请码」tab 生成新码 |
| 忘了 bootstrap invite | `docker compose logs backend | grep bootstrap_invite=`；若日志已 rotate，进 DB 看 `invite_codes` 表，或直接进 backend 容器 `python -m app.scripts ...`（目前没写工具，按需要再加） |
| 前端首屏一直转圈 | `/api/auth/me` 401；浏览器 console 看请求 |
| Sub-agent 跑超时 | 默认 `SUB_AGENT_MAX_RUNTIME_S=1800` (30min)；在 `.env` 加项覆盖 |
| 占满磁盘 | sandbox 容器 / 镜像 / `.traces/` —— `docker system prune` + 清 `oryx-traces` volume |

## 性能 / 容量参考

| 资源 | 默认上限 | 调节 |
|---|---|---|
| 单沙箱 CPU | 0.25 core | `SANDBOX_MAX_CPU` |
| 单沙箱 内存 | 256 MB | `SANDBOX_MAX_MEM_MB` |
| 单沙箱 wallclock | 30s | `SANDBOX_MAX_TIMEOUT_S` |
| 全局并发沙箱 | 8 | `SANDBOX_MAX_CONCURRENT` |
| Sub-agent 上限运行时间 | 30 min | `SUB_AGENT_MAX_RUNTIME_S` |
| Web fetch 单页 inline | 30k 字符（超出落盘） | `WEB_FETCH_INLINE_MAX` |
| Compact 软触发 | 800k tokens | 代码里硬编码，需要 PR |

Agent 请求超 cap 的资源会被自动 clamp（返回里 `clamped: {...}` 字段告诉它），不会 reject。
