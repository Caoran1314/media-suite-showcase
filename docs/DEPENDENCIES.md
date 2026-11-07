# 依赖服务配置教程

media-suite 自己**不下载、不转存、不刮削**，它只做「找资源 + 调度」。
真正的搜索、转存、整理、播放由下面几个服务完成。本文把每个服务怎么配、哪个值填到哪里，一次讲清。

---

## 一、全局关系

```
       你（面板 / Telegram / 飞书）
                 │  加片名、回序号
                 ▼
        ┌───────────────────┐
        │    media-suite    │  ← 本项目（只有一个容器）
        │  搜索调度 / 推送   │
        └───┬───────────┬───┘
            │ 搜        │ 让它转存
            ▼           ▼
     ┌──────────┐  ┌──────────────────────┐
     │  pansou  │  │ Cloud Media Sync     │
     │ 聚合搜索  │  │ 转存115→整理→strm    │
     └──────────┘  └──────────┬───────────┘
                              │ 生成 strm / 302 直链
                              ▼
                       ┌──────────┐        ┌──────────┐
                       │   Emby   │        │   115    │
                       └──────────┘        └──────────┘

     （可选）MoviePilot —— 只作为「订阅来源」被读取，不参与执行
```

## 二、必需 / 可选一览

| 服务 | 是否必需 | 作用 | 本项目怎么用它 |
|---|---|---|---|
| **pansou** | 可选（建议） | 聚合搜索网盘分享；不填时用内置自研源 TG115 | `PANSOU_BASE_URL` |
| **Cloud Media Sync (CMS)** | 路线 A 必需；路线 B 不用 | 把 115 分享转存、自动整理命名、生成 strm、302 直链 | `CMS_BASE_URL` + `CMS_API_TOKEN` |
| **115 网盘** | ✅ 必需 | 资源落地的地方 | 路线 A 由 CMS 持有 cookie；路线 B 由本项目的 115 扫码持有 |
| **Emby / Jellyfin** | 建议 | 播放端 | 路线 A 读 CMS 生成的 strm；路线 B 需自行挂载 115 |
| **MoviePilot** | 可选 | 订阅中心（如果你习惯在 MP 里订阅） | 开启后从 MP 读订阅列表 |

> **最小可用组合（路线 A）**：`pansou` + `Cloud Media Sync` + `115`。
> **不装 CMS（路线 B）**：`pansou` + 115 + 一个 115→Emby 的挂载方案，部署步骤见 [TUTORIAL.md](TUTORIAL.md) 的「路线 B」。
> 不装 MoviePilot 也能用（订阅直接加在 media-suite 面板里）；不装 Emby 也能入库（只是没法看）。

---

## 三、pansou（搜索源）

### 3.1 部署

```bash
docker run -d --name pansou \
  -p 3200:8888 \
  --restart unless-stopped \
  ghcr.io/fish2018/pansou:latest
```

> 端口按你习惯改；部署好之后把地址填进 media-suite 的 `PANSOU_BASE_URL`。不部署也能用内置 TG115，但 TG115 需要能访问 t.me（大陆网络要代理），所以国内环境建议还是自建 pansou。

### 3.2 关键要求：必须能搜出 **115** 链接

pansou 是聚合器，不同实例/插件覆盖的网盘不同。实测中有些实例只返回夸克/阿里，**搜不到 115 就等于本项目没法用**。

验证方法（在能访问 pansou 的机器上）：

```bash
curl -s "http://<pansou地址>:<端口>/api/search" \
  -H 'Content-Type: application/json' \
  -d '{"kw":"沙丘2","res":"merge","src":"all"}' | head -c 300
```

返回里出现 `115.com` / `115cdn.com` 链接才算合格。

如果自建实例搜不到 115：
- 用社区里明确支持 115 的公开实例（稳定性看运气，别当唯一来源）
- 或在 pansou 里配置 115 相关的搜索插件/频道

### 3.3 填进 media-suite

```yaml
PANSOU_BASE_URL: http://<你的pansou>:3200    # 可选，建议填
```

---

## 四、Cloud Media Sync（路线 A：转存 + 整理 + strm）

> 本章只适用于**路线 A（有 CMS）**。没有 CMS、想用内置直转，请看 [TUTORIAL.md](TUTORIAL.md) 的「路线 B」；两条路线的区别见该文开头的对照表。

这是整条链路里配置最多的一环，但配好一次就长期稳定。

### 4.1 部署

```bash
docker run -d --name cms \
  -p 9527:9527 \
  -e PUID=1000 -e PGID=1000 -e TZ=Asia/Shanghai \
  -v /你的路径/cms/config:/config \
  -v /你的路径/cms/media:/media \
  -v /你的路径/cms/logs:/logs \
  --restart unless-stopped \
  imaliang/cloud-media-sync:0.5.0.1
```

首次打开 `http://<NAS>:9527`，默认账号 `admin/admin`，**第一件事改密码**。

### 4.2 必做配置（按顺序）

**① 绑定 115 账号**

```
设置 → 115 账号 → 扫码/OAuth 授权
```

授权成功后 CMS 自己持有 cookie，后续转存都用它。
（本项目不需要你的 115 cookie，除非你开了「关键词分流」。）

**② 媒体目录**

```
设置 → 基础设置 → 媒体目录 = /media
```

所有 strm、nfo、图片都会生成在这下面，之后 Emby 直接指向它。

**③ 分享转存**

```
设置 → 分享转存
  转存目录       /最近接收         ← 所有转存先进这里
  自动整理       开启
  整理后删除源文件  按需（建议开，省 115 空间）
```

**④ 自动整理（分类）**

```
设置 → 自动整理 → 分类规则（category.yaml）
```

示例（本项目实测可用）：

```yaml
movie:
  - name: 动画电影
    cid: "1234567890123456789"      # 115 里「动漫」目录的 cid
    genre_ids: "16"
  - name: 电影兜底
    cid: "3440173006499937751"      # 115 里「4K电影」目录的 cid
tv:
  - name: 动漫
    cid: "1234567890123456789"
    genre_ids: "16"
  - name: 剧集兜底
    cid: "1234567890123456789"
```

> `cid` 是 115 目录 ID：浏览器打开该目录，URL 里 `cid=xxxxx` 那串就是。

**⑤ 洗版规则（可选）**

```
设置 → 洗版 → 电影：替换模式，优先级 2160p REMUX > 2160p > 1080p
              剧集：跳过（避免误删连载）
```

**⑥ Webhook token（本项目要用）**

```
设置 → 接口/Webhook → token（默认 cloud_media_sync）
```

这就是 media-suite 里 `CMS_API_TOKEN` 要填的值。转存接口形如：

```
GET /api/cloud/add_share_down?token=<token>&url=<115分享链接>
```

### 4.3 目录规划建议（重要）

115 网盘侧建议先建好结构，`cid` 才不会乱：

```
/Movies影视
   /4K电影        ← movie 兜底
   /剧集          ← tv 兜底
   /最近接收      ← 转存入口（配置里的「转存目录」）
   /重复文件      ← 整理冲突时丢这里
```

### 4.4 填进 media-suite

```yaml
CMS_ENABLED: "true"
CMS_BASE_URL: http://<NAS>:9527
CMS_API_TOKEN: cloud_media_sync
```

---

## 五、Emby（播放端）

CMS 负责生成 strm，Emby 只负责「扫描 + 播放」。

### 5.1 添加媒体库

```
Emby 后台 → 媒体库 → 添加媒体库
  内容类型：电影 / 电视节目
  文件夹：  /media/Movies影视/4K电影  等（对准 CMS 的媒体目录）
  元数据语言：中文
```

### 5.2 让 strm 直连 115（302）

CMS 侧开启「strm 直链 / 302」，播放时 Emby 会 302 跳到 115 CDN，**不占本地带宽与空间**：

```
CMS → 设置 → strm / 直链 → 开启 302
```

### 5.3 触发刷新

CMS 整理完成后会自动通知 Emby 刷新（在 CMS 的「媒体服务器」里填 Emby 地址 + API Key）。
本项目只负责把文件丢进 CMS，不直接操作 Emby。

---

## 六、MoviePilot（可选：订阅来源）

只有你「习惯在 MoviePilot 里订阅」时才需要。开着它，media-suite 会优先读 MP 的订阅列表。

### 6.1 部署与初始化

```bash
docker run -d --name moviepilot-v2 \
  -p 3001:3000 \
  -v /你的路径/mp/config:/config \
  -v /你的路径/mp/media:/media \
  --restart unless-stopped \
  jxxghp/moviepilot-v2:latest
```

打开 `http://<NAS>:3001`，完成初始化向导，设置管理员密码。

### 6.2 给 media-suite 用的配置

```yaml
MP_ENABLED: "true"
MP_BASE_URL: http://<NAS>:3001
MP_USERNAME: admin
MP_PASSWORD: <你在MP里设的密码>
```

media-suite 只调用 `GET /api/v1/subscribe/` 读订阅，不会修改 MP 的任何内容。

> ⚠️ 不要在 MP 里装 `CloudStrm` / `P115StrmHelper` / `P115StrgmSub` / `CloudLinkMonitor` —— 它们会和 CMS 抢「转存/整理」的活，造成重复文件或互相删文件。

---

## 七、115 网盘准备

1. 开通会员（否则转存大文件会受限）
2. 建好目录结构（见 4.3），记下要用到的 `cid`
3. **不用**把 cookie 交给 media-suite，除非你要用「关键词分流」功能：
   ```yaml
   U115_ROUTING_ENABLED: "true"
   U115_COOKIE_FILE: /data/cookie.txt   # 把 115 网页 cookie 粘进去
   U115_PENDING_CID: "1234567890123456789"   # 待整理目录 cid
   ```

---

## 八、配置对照表（最常填错的几个值）

| media-suite 变量 | 从哪里拿 | 示例 |
|---|---|---|
| `PANSOU_BASE_URL` | pansou 地址（可选） | 空（用内置 TG115） |
| `CMS_BASE_URL` | CMS 的访问地址 | `http://<你的NAS-IP>:9527` |
| `CMS_API_TOKEN` | CMS → Webhook token | `cloud_media_sync` |
| `MP_BASE_URL` | MoviePilot 地址 | `http://<你的NAS-IP>:3001` |
| `MP_PASSWORD` | 你在 MP 设的管理员密码 | — |
| `U115_PENDING_CID` | 115「/最近接收」目录的 cid | `1234567890123456789` |
| `PANEL_URL` | 本项目对外地址（消息里的链接） | `https://ms.xxx.com:1273` |

---

## 九、连通性自检（配完先跑一遍）

```
① pansou 能搜出 115 链接     → curl /api/search，看返回里有没有 115.com
② CMS 能收转存               → 手动粘一个 115 分享链接到 CMS 的「分享转存」
③ CMS 整理完生成 strm        → 看 /media 下有没有同名 .strm
④ Emby 能播                  → 播一集，看是不是 302 到 115 CDN
⑤ media-suite 面板「健康」   → 底部四个灯全绿（分流/TG/CMS/pansou）
⑥ 面板「搜索候选」有结果     → 点一个「入库」，1~3 分钟出现在 Emby
```

逐个失败就从对应环节查：① pansou → ②③ CMS → ④ Emby。

---

## 十、常见坑

| 现象 | 原因 | 解决 |
|---|---|---|
| 搜不到 115，只有夸克/阿里 | pansou 实例覆盖面不够 | 换实例，或给 pansou 加 115 插件 |
| 转存报错 401/403 | `CMS_API_TOKEN` 不对 | 用 CMS 后台里那串 token |
| 转存成功但 Emby 没出现 | CMS 没整理 / Emby 没刷新 | 检查「自动整理」开关 + CMS 的媒体服务器配置 |
| 媒体库里一堆 `.strm` 却播不了 | 302 直连没开 | CMS → strm/直链 → 开启 |
| 同一部片子出现两份 | MP 的 strm 插件和 CMS 同时干活 | 卸掉 MP 的 CloudStrm/P115StrmHelper 类插件 |
| 面板显示「CMS 已连接」但入库失败 | CMS 里 115 授权过期 | CMS → 115 账号 → 重新授权 |
| 图片/海报全空 | 没配 `TMDB_API_KEY` 或容器访问不了外网 | 配 key + 容器代理 |


## 多个搜索源（可选）

`PANSOU_BASE_URL` 支持写多个地址（逗号或空格分隔），会**并行查询并合并去重**：

```bash
PANSOU_BASE_URL=http://<自建>:3200,https://<公共实例>
```

- 任一源可用即可正常搜索；单源失败只记日志，不影响整体
- 面板「设置」里能看到当前配置的源（`config.pansou` 为数组）
- 自建 pansou 建议保留，作为公网实例的兜底
