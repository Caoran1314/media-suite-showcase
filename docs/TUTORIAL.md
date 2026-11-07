# FilmShelf 使用教程（用户版）

> 这份教程只讲「用户怎么把它用起来」，不涉及卖家侧的东西。
> 全程只需要 **一个容器** + 你自己的 115 账号 + 一个转存方案（下面二选一）。

---

## 〇、先选路线（30 秒）

FilmShelf 不自己存文件，它负责「找片 → 转存 → 整理 → 让 Emby 看到」。**转存/整理这一步有两种做法，部署前先选一种**：

| | 路线 A：有 CMS（推荐） | 路线 B：没有 CMS（内置直转） |
|---|---|---|
| 额外要装什么 | Cloud Media Sync（或兼容 CMS） | 什么都不用装 |
| 谁负责转存 | CMS | FilmShelf 自己调 115 接口 |
| 谁负责整理/生成 strm | CMS | FilmShelf 只整理 115 目录，**不生成 strm** |
| Emby 怎么看到片 | CMS 生成 strm → Emby 扫描 | 需要你自己把 115 挂进 Emby（CloudDrive2 / Alist / rclone） |
| 适合谁 | 想装完就能在 Emby 看 | 已经有 115→Emby 挂载，或只想自动整理网盘 |
| 成熟度 | 稳定 | 实验性（半自动） |

一句话：

- **想「装完就能在 Emby 播放」→ 走 A。**
- **已经有 115 挂载、只缺自动转存和整理 → 走 B。**

两条路线的容器、登录、115 授权、通知都是一样的，只有「转存方式」和「Emby 怎么看片」不同。下面分开写。

---

## 一、两条路线都要做的准备

| 需要 | 说明 |
|---|---|
| 一台能跑 Docker 的 NAS / 服务器 | 群晖、威联通、fnOS、Linux 都行 |
| 一个 115 网盘账号 | 转存必须用你自己的账号（文件进你的网盘） |
| 一个数据目录 | 挂到容器 `/data`，存配置、Cookie、日志；**必须持久化** |
| Emby（可选） | 路线 A 由 CMS 刷新；路线 B 需要你自行挂载 115 |
| 端口 | 面板默认 `8790` |

镜像：

```bash
docker pull caowenrui/media-suite:2.1.0   # 固定版本（推荐）
docker pull caowenrui/media-suite:latest  # 跟随最新
```

> 以下示例统一用 `caowenrui/media-suite:2.1.0`。老教程里的 `caoran1314/media-suite` 是错的，拉不到。

---

# 路线 A：有 CMS

适用：你已经在用 Cloud Media Sync（或兼容它的服务）管理 115，想让它继续负责转存、整理、strm 和 Emby 刷新。

## A1. 先把 CMS 准备好

FilmShelf 只给 CMS 发一个「转存这个分享」的请求，剩下都归 CMS。所以 CMS 那边必须先能用：

1. CMS 已部署，能打开它的管理页；
2. CMS 里已配置好 **115 账号**（它自己的授权，和 FilmShelf 的扫码是两回事）；
3. CMS 里已设置好 **媒体目录 / strm 输出目录**；
4. CMS 已接入 **Emby**，能刷新媒体库；
5. 记下 CMS 的访问地址，例如 `http://192.168.1.10:9527`；
6. 记下 CMS 的 API Token（Cloud Media Sync 默认是 `cloud_media_sync`，以你的 CMS 配置为准）。

> FilmShelf 调用的是 CMS 的 `POST /api/cloud/add_share_down?token=...`，body 为 `{"url": "115分享链接"}`。如果你的 CMS 是改过的版本，确认这个接口还在。

## A2. 启动 FilmShelf（路线 A 的 compose）

```yaml
services:
  media-suite:
    image: caowenrui/media-suite:2.1.0
    container_name: media-suite
    restart: unless-stopped
    ports:
      - "8790:8790"
    environment:
      TZ: Asia/Shanghai
      DATA_DIR: /data
      TRANSFER_BACKEND: cms
      CMS_BASE_URL: http://192.168.1.10:9527   # 改成你的 CMS 地址
      CMS_API_TOKEN: cloud_media_sync          # 改成你的 CMS Token
    volumes:
      - ./data:/data
```

```bash
docker compose up -d
```

要点：

- `CMS_BASE_URL` 必须是**容器里能访问到**的地址，不要写 `127.0.0.1` / `localhost`；
- 不想写死在 compose 里，也可以启动后在面板「设置 → 入库」填，效果一样；
- `TRANSFER_BACKEND` 显式写 `cms` 最稳，避免 `auto` 在 CMS 没填好时偷偷走内置直转。

## A3. 首次配置

1. 打开 `http://你的NAS:8790`，用默认 `admin / admin` 登录；
2. 按提示改掉用户名和密码；
3. 「设置 → 115 授权」扫码登录（**必须做**：转存前 FilmShelf 要用它校验分享是否有效、判断是文件还是目录）；
4. 「设置 → 入库」确认 CMS 地址 + Token 已填，保存；
5. （可选）「设置 → EMBY」填 Emby 地址和 API Key，概览页就有媒体统计和海报。

## A4. 验证（3 步）

1. 面板「发现」搜一部片，点一个候选的「转存」；
2. 事件页出现「已提交转存」→ 去 CMS 的任务/记录里能看到这条分享；
3. 等 CMS 整理完、生成 strm 后，Emby 刷新，媒体库里出现这部片。

三步都通，路线 A 就配好了。

## A5. 路线 A 常见坑

| 现象 | 原因 | 处理 |
|---|---|---|
| 转存报「CMS 不可达」 | 容器里访问不到 CMS 地址 | 把 `CMS_BASE_URL` 改成 NAS 的局域网 IP；确认端口通 |
| 转存报 401 / token 错误 | `CMS_API_TOKEN` 和 CMS 配置不一致 | 两边改成同一个值，重启容器 |
| 转存成功但 Emby 没片 | CMS 的媒体目录和 Emby 库路径不一致 | 在 CMS 里核对目录映射，重新生成 strm |
| 提示分享失效 | FilmShelf 的 115 Cookie 过期 | 重新扫码授权 |
| 明明有 CMS 却走了直转 | `TRANSFER_BACKEND=auto`，且 CMS 未配置成功 | 显式改成 `cms` |

---

# 路线 B：没有 CMS（内置直转，实验性）

适用：你没有 CMS，或者不想为了转存再装一套 CMS。FilmShelf 自己完成「接收分享 → 识别 → 在 115 里建目录 → 移动/重命名」。

**必须知道的能力边界（很重要）：**

- ✅ 会把分享转存进你的 115；
- ✅ 会按 TMDB 识别，整理到 `Movies影视/4K电影/...` 或 `Movies影视/剧集/...`；
- ✅ 会自动建目录、移动、重命名；
- ❌ **不会生成 `.strm` 文件**（当前版本还没接这条链路）；
- ❌ 不会通知 Emby 刷新。

所以路线 B 想在海报墙上看到片，**你必须自己让 Emby 能访问 115**，见 B5。

## B1. 启动 FilmShelf（路线 B 的 compose）

```yaml
services:
  media-suite:
    image: caowenrui/media-suite:2.1.0
    container_name: media-suite
    restart: unless-stopped
    ports:
      - "8790:8790"
    environment:
      TZ: Asia/Shanghai
      DATA_DIR: /data
      TRANSFER_BACKEND: 115direct
      DIRECT115_DRY_RUN: "0"        # 先别急着 0，见 B4
      # U115_PENDING_CID: ""        # 可留空，默认用 115 的「最近接收」
    volumes:
      - ./data:/data
```

```bash
docker compose up -d
```

> 注意后端的正确取值是 **`115direct`**（不是 `direct115`）。写错会回退到 CMS，然后报 CMS 不可达。

## B2. 授权 115（必须）

「设置 → 115 授权」→ 用 115 App 扫码 → 面板把 Cookie 存到 `/data/cookie.txt`。

也可以用备用方式手工粘 Cookie（要包含 `UID=` 和 `CID=`），保存时会立刻读一次 115 根目录验证。

> 没有这个 Cookie，内置直转直接不可用（面板会回退到 CMS）。

## B3. 先跑一次 dry-run（强烈建议）

`DIRECT115_DRY_RUN` 默认是 `1`：**只接收分享、做规划，不移动文件**。先用默认值跑一部片，看事件页/日志里的规划结果：

```
[direct115] 规划完成 { name, season, episode, target: {...}, dryRun: true }
```

确认片名、季集、目标目录都对，再改成 `DIRECT115_DRY_RUN=0` 重启容器，才会真正移动。

## B4. 它会整理成什么样

115 里的目标结构（`libraryRoot` 默认 `Movies影视`）：

```
Movies影视/
  4K电影/
    D-沙丘-2021/沙丘.2021.mkv
  剧集/
    剧名-2024/Season 01/剧名.S01E01.mkv
```

- 待整理目录默认是 115 的「最近接收」（`U115_PENDING_CID` 可改）；
- 目录/文件名会清洗 115 不接受的字符（`:` `/` `*` 等）；
- 115 的重命名接口偶尔报「服务器开小差了」，移动成功就算入库成功，文件名可能保持原样，可手动改。

## B5. 让 Emby 看到（路线 B 的关键）

FilmShelf 只把文件整理进 115，**不会写 strm**。Emby 要看到，选一种：

1. **挂载 115（推荐）**：在 NAS 上用 CloudDrive2 / Alist / rclone 把 115 的 `Movies影视` 挂成本地目录，Emby 媒体库指向这个挂载点；
2. **自己生成 strm**：用其他工具/脚本把 115 直链写成 `.strm`，放到 Emby 能扫的目录；
3. **以后接 CMS**：想要自动 strm，直接切回路线 A。

> 如果你没有 115 挂载，也没有 strm 方案，路线 B 只能帮你把网盘整理干净，Emby 里是看不到片的——这不是 bug，是当前版本的边界。

## B6. 验证（3 步）

1. `DIRECT115_DRY_RUN=1` 下转存一部片，事件/日志出现 `dryRun: true` 和规划路径；
2. 改成 `0` 重启，再转存一部，去 115 里确认文件出现在 `Movies影视/...`；
3. 到 Emby 的挂载点/媒体库刷新，确认能看到这部片（没挂载则这一步会失败，属预期）。

## B7. 路线 B 常见坑

| 现象 | 原因 | 处理 |
|---|---|---|
| 转存「成功」但 115 里没动 | `DIRECT115_DRY_RUN` 还是默认 `1` | 改成 `0` 并重启容器 |
| 报 CMS 不可达 | `TRANSFER_BACKEND` 写成了 `direct115` 等错值 | 改成 `115direct` |
| 转存报没有 Cookie | 没扫码 / `/data/cookie.txt` 丢了 | 重新扫码；确认 `/data` 已持久化 |
| 文件在 115 里，Emby 没有 | 没挂载 115，也没有 strm | 见 B5 |
| 115 接口偶发失败 | 风控/频率 | 放慢操作，隔一会儿再试；必要时重新扫码 |
| 名字被改得奇怪 | 115 不允许的字符被清洗 | 正常现象，可手动重命名 |

---

## 二、两条路线对比（一张表）

| 项目 | 路线 A（CMS） | 路线 B（内置直转） |
|---|---|---|
| 需要 CMS | 是 | 否 |
| `TRANSFER_BACKEND` | `cms` | `115direct` |
| 需要 115 Cookie | 是（校验分享） | 是（转存本身） |
| 自动建目录/移动 | CMS 做 | FilmShelf 做 |
| 生成 strm | CMS 做 | 不生成 |
| Emby 刷新 | CMS 做 | 不刷新 |
| 装完就能看 | ✅ | ❌ 需自行挂载 115 |
| 推荐场景 | 大多数用户 | 已有 115 挂载 / 只整理网盘 |

---

## 三、通用：登录、日常使用、升级

### 登录

第一次 `admin / admin`，登录后必须改用户名和密码。

### 搜片并转存

面板「发现」输入片名 → 选候选 → 「转存」。路线 A 会进 CMS，路线 B 会进 115 的待整理目录并移动。

也可以在 Telegram 里：`搜 怪奇物语` → 回序号 `1` 转存。

### 订阅与自动追更

订阅后每 30 分钟巡检一次，发现新集会通知；已入库的不会重复。

### 升级

```bash
docker compose pull
docker compose up -d
```

数据都在 `/data`，升级不会丢。升级前建议备份一份 `/data`。

### 备份

```bash
tar -czf media-suite-data-$(date +%F).tgz -C /你的路径/media-suite data
```

### 命令速查

```bash
docker logs -f media-suite     # 看日志
docker restart media-suite     # 重启
docker compose pull && docker compose up -d   # 升级
```

---

## 四、常见问题

**Q：到底要不要装 CMS？**
A：想装完就能在 Emby 看，装（路线 A）。已经有 115 挂载、只需要自动转存整理，不装（路线 B）。

**Q：路线 B 为什么不生成 strm？**
A：当前版本还没接这条链路，只做「转存 + 整理」。需要自动 strm 请用路线 A，或自行挂载 115。

**Q：115 扫码和 CMS 的 115 授权是一回事吗？**
A：不是。FilmShelf 的扫码只给它自己用（校验分享/直转），CMS 的 115 授权是 CMS 自己用的，两边各配各的。

**Q：`auto` 是什么？**
A：自动选择：CMS 配好就用 CMS，否则用内置直转。排查问题时建议显式写 `cms` 或 `115direct`，避免它自己切。

**Q：数据会上传吗？**
A：不会。所有数据在你自己 NAS 的 `/data`，对外只访问 115、TMDB、pansou 和你配置的 CMS/Emby。

**Q：能放在公网吗？**
A：可以，但请套反向代理 + HTTPS，并改掉默认密码；面板本身有登录鉴权。

---

## 五、相关文档

- 依赖服务怎么装、参数怎么填：[DEPENDENCIES.md](DEPENDENCIES.md)
- 环境变量总表：[ENV.md](ENV.md)
- 更新记录：[CHANGELOG.md](../CHANGELOG.md)
