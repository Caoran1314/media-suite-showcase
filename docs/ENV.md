# 环境变量全表

> 全部通过容器启动时的 --env 注入，改完需重启容器。
> 标「必填」的项缺失时，面板顶部会出现红条（启动自检），对应功能不可用。

## 基础

| 变量 | 默认值 | 说明 |
|---|---|---|
| PORT | 8790 | 面板端口 |
| DATA_DIR | /data | 数据目录（订阅、事件、图片缓存） |
| ADMIN_PASSWORD | 空 | 面板口令，公网必须设置 |
| LOG_LEVEL | info | 日志级别 |

## 搜索源 pansou（必填）

| 变量 | 默认值 | 说明 |
|---|---|---|
| PANSOU_BASE_URL | 空 | 可选：自己部署的 pansou（留空用内置 TG115，需能访问 t.me） |
| TG115_PROXY | 空 | 可选：访问 t.me 用的 HTTP 代理（旁路由 / NAS 上的 mihomo，如 http://172.17.0.1:7890）；面板「设置 → 搜索」里也能填，保存后重启生效 |
| PANSOU_TIMEOUT_MS | 45000 | 单次搜索超时 |
| PANSOU_MAX_PAGES | 2 | 每个源最多翻页数 |
| PANSOU_PAGE_SIZE | 300 | 每页条数 |
| ALLOWED_NETDISKS | 115 | 接受的网盘类型：115 / 115,quark / all |
| SEARCH_BACKENDS | pansou | 启用的搜索适配器 |

> pansou 必须自建，且 CHANNELS / ENABLED_PLUGINS 要照官方 compose 抄，否则搜不到 115。
> 详见 docs/pansou-selfhost.md

## 筛选

| 变量 | 默认值 | 说明 |
|---|---|---|
| MIN_SCORE | 1000 | 候选最低分，面板「过滤强度」会覆盖 |
| PREFER_RESOLUTION | 2160p | 首选画质 |
| PREFER_COMPLETE_SERIES | true | 优先整季/合集 |
| REJECT_CAM | true | 剔除枪版 |

## 入库后端（二选一）

| 变量 | 默认值 | 说明 |
|---|---|---|
| TRANSFER_BACKEND | cms | cms（模式 A）或 115direct（模式 B） |

### 模式 A · CMS

| 变量 | 默认值 | 说明 |
|---|---|---|
| CMS_ENABLED | true | 是否启用 |
| CMS_BASE_URL | 空 | 例 http://10.0.0.5:9527（必填） |
| CMS_API_TOKEN | cloud_media_sync | CMS API token |
| CMS_TIMEOUT_MS | 60000 | 转存超时 |

### 模式 B · 直连 115

| 变量 | 默认值 | 说明 |
|---|---|---|
| DIRECT115_DRY_RUN | 1 | 1=只接收+规划，不移动文件；确认规划无误后设 0 才会真正整理 |
| U115_PENDING_CID | 空 | 待整理目录 cid；留空自动用 115 的「最近接收」 |

## 115

| 变量 | 默认值 | 说明 |
|---|---|---|
| U115_COOKIE_FILE | /data/cookie.txt | 115 cookie 文件路径（扫码后自动写入；/data 需持久化） |
| U115_ROUTING_ENABLED | true | 是否启用关键词分流 |
| U115_ROUTE_INTERVAL_MS | 60000 | 分流扫描间隔 |
| U115_RPS | 2.5 | 115 请求速率上限（防限流） |

## Emby（建议）

| 变量 | 默认值 | 说明 |
|---|---|---|
| EMBY_URL | 空 | 例 http://10.0.0.5:8096 |
| EMBY_API_KEY | 空 | Emby → 设置 → API 密钥 |
| EMBY_MATCH | 1 | 1=匹配库内条目；0=全部显示未入库 |

## TMDB（建议）

| 变量 | 默认值 | 说明 |
|---|---|---|
| TMDB_API_KEY | 空 | TMDB v3 API Key |
| TMDB_API_DOMAIN | https://api.tmdb.org | 可换镜像 |
| TMDB_IMG_DOMAIN | https://image.tmdb.org | 可换镜像 |
| TMDB_LANG | zh-CN | 元数据语言 |

## 巡检

| 变量 | 默认值 | 说明 |
|---|---|---|
| WATCH_ENABLED | true | 是否定时巡检 |
| WATCH_INTERVAL_MIN | 30 | 巡检间隔（分钟），面板可覆盖 |
| WATCH_TOP_N | 5 | 每部片每次推几个候选 |
| WATCH_REOFFER_MINUTES | 1440 | 同一候选多久后可再推 |

## 通知

| 变量 | 默认值 | 说明 |
|---|---|---|
| NOTIFY_TELEGRAM | false | TG 推送开关 |
| TG_BOT_TOKEN / TG_CHAT_ID | 空 | Bot token 与目标 |
| TG_INGEST | false | 接收 TG 指令（发片名即订阅） |
| NOTIFY_FEISHU | false | 飞书推送 |
| FEISHU_APP_ID / FEISHU_APP_SECRET | 空 | 飞书应用凭据 |
| FEISHU_WS | true | 飞书长连接 |
| NOTIFY_WEBHOOK / WEBHOOK_URL | false / 空 | 通用 webhook |

## MoviePilot（可选，仅导入订阅）

| 变量 | 默认值 | 说明 |
|---|---|---|
| MP_ENABLED | true | 是否读取 MP 订阅 |
| MP_BASE_URL | 空 | MP 地址 |
| MP_USERNAME / MP_PASSWORD | admin / 空 | MP 账号 |
| MP_TIMEOUT_MS | 20000 | 超时 |

## 授权 / 兑换

| 变量 | 默认值 | 说明 |
|---|---|---|
| LICENSE_API | 空 | 授权服务地址（面板下单、卡密兑换都走它） |
| LICENSE_SERVER | 空 | 心跳上报地址，一般与 LICENSE_API 相同 |
| LICENSE_ADMIN_TOKEN | 空 | 机器人调授权服务管理接口用 |
| LICENSE_ADMIN_CHATS | 空 | 允许用授权指令的 TG/飞书会话 |
| LICENSE_PUBLIC_KEY | 空 | 离线验签公钥；不填则读 data/license-public.pem |

## 搜索后端

| 变量 | 默认值 | 说明 |
|---|---|---|
| SEARCH_BACKENDS | pansou | 逗号分隔：pansou / tg115 |
| PANSOU_BASE_URL | 空 | 搜索源（可选），多个用逗号分隔 |
| TG115_CHANNELS | 内置 111 个 | tg115 采集的 TG 频道，逗号分隔 |
| TG115_CONCURRENCY | 12 | 并发抓取数 |
| TG115_TIMEOUT_MS | 8000 | 单频道超时 |
| TG115_BUDGET_MS | 28000 | 整轮预算，超时返回已抓到的部分 |
| TG115_CACHE_MIN | 30 | 结果缓存分钟数 |

## 开关（也可写进 data/local-config.json，见下）

| 变量 | 默认值 | 说明 |
|---|---|---|
| PRICING_ENABLED | 0 | 1 = 对外露出定价页与 landing 定价入口 |
| DIAG_ENABLED | 0 | 1 = 打开 /api/diag/* 排障接口（都需登录口令） |
| PRICING_EMAIL / PRICING_URL | 空 | 定价页展示的联系方式 |

## data/local-config.json

容器 env 不方便改的时候，把同名键写进 `DATA_DIR/local-config.json`，重启生效，优先级低于 env：

```json
{
  "LICENSE_API": "http://<NAS>:8791",
  "LICENSE_SERVER": "http://<NAS>:8791",
  "LICENSE_ADMIN_TOKEN": "……",
  "PANSOU_BASE_URL": "http://<NAS>:3200",
  "SEARCH_BACKENDS": "pansou,tg115",
  "PRICING_ENABLED": "1",
  "DIAG_ENABLED": "0"
}
```

## 数据备份

| 变量 | 默认值 | 说明 |
|---|---|---|
| BACKUP_ENABLED | 1 | 1 = 每天自动打包一次 DATA_DIR |
| BACKUP_KEEP | 7 | 保留最近几份 |
| BACKUP_DIR | DATA_DIR/backups | 备份目录；建议指到另一块盘 |

备份内容：订阅、事件、授权、卡密等状态文件（tar.gz，通常 <1MB）。
也可手动触发：`POST /api/backup/run`，列表：`GET /api/backup/list`。

## 敏感项放哪（避免容器重建后失效）

fnOS 的 docker inspect 会把 `*_API_KEY` / `*_TOKEN` / `*_PASSWORD` 遮蔽成 `[REDACTED]`，
用它做模板重建容器会把真值覆盖成假值。

因此下面这些值请写进 `DATA_DIR/local-config.json`（优先级高于 env，重建不受影响）：

```json
{
  "EMBY_API_KEY": "……",
  "TMDB_API_KEY": "……",
  "CMS_API_TOKEN": "……",
  "MP_PASSWORD": "……",
  "TG_BOT_TOKEN": "123456:AA……",
  "TG_INGEST_TOKEN": "123456:AA……",
  "TG_CHAT_ID": "你的 numeric id"
}
```

启动时 `config.js` 的 `applyLocalSecrets()` 会用它们覆盖 env 值。

## 身份与容器（local-config 里配）

| 键 | 说明 |
|---|---|
| DEV_MODE | 1 = 开发者模式：本机直接按永久版处理，隐藏购买/卡密/免费额度；用户部署不要设 |
| DOCKER_SHOW | 小程序里显示哪些容器，逗号分隔（默认 = 重启白名单 + media-suite） |
| DOCKER_RESTART_ALLOW | 允许点「重启」的容器白名单 |
| SERVICES | 服务健康探测列表（JSON 数组字符串）：[{"name":"面板","url":"http://127.0.0.1:8790/healthz"}] |
| BACKUP_ENABLED / BACKUP_DIR / BACKUP_KEEP | 每日备份开关、目录、保留份数 |

> DEV_MODE 是「一套代码两种配置」的开关：你是开发者就开，用户拿到的是默认（免费/付费）体验。
