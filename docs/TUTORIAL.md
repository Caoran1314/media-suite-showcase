# FilmShelf 使用教程（用户版）

> 这份教程只讲「用户怎么把它用起来」，不涉及任何卖家侧的东西。
> 全程只需要 **一个容器** + 你自己的 115 账号。

---

## 一、它是干什么的

一句话：**自动帮你把 115 网盘上的影视资源找出来、转存进自己的网盘、整理好并让 Emby 能直接看到。**

完整链路：

```
你输入片名
   ↓
它去多个搜索源找 115 分享链接（返回画质、体积、来源）
   ↓
你挑一个（点一下，或回个序号）
   ↓
它把分享转存进你的 115 网盘，并按「剧名/年份/季」建好目录
   ↓
Emby 刷新后就能在媒体库里看到
```

订阅模式：把想追的剧加进「我的订阅」，它会每 30 分钟自动找新集，找到就通知你。

---

## 二、准备

| 需要 | 说明 |
|---|---|
| 一台能跑 Docker 的 NAS / 服务器 | 群晖、威联通、fnOS、Linux 都行 |
| 一个 115 网盘账号 | 转存必须用你自己的账号（文件进你的网盘） |
| Emby（可选） | 有的话能显示媒体统计、海报，并入你的媒体库 |
| CMS（可选） | 你本来就在用 CMS 管理 115 的话，可以接上它；没有也能用 |

端口：面板默认 `8790`。

---

## 三、安装

### 方式 A：docker run

```bash
docker pull caoran1314/media-suite:latest

docker run -d --name media-suite --restart unless-stopped \
  -p 8790:8790 \
  -v /你的数据目录:/data \
  -e TZ=Asia/Shanghai \
  -e TRANSFER_BACKEND=auto \
  caoran1314/media-suite:latest
```

### 方式 B：docker-compose

```yaml
services:
  media-suite:
    image: caoran1314/media-suite:latest
    container_name: media-suite
    restart: unless-stopped
    ports:
      - "8790:8790"
    environment:
      TZ: Asia/Shanghai
      DATA_DIR: /data
      TRANSFER_BACKEND: auto
    volumes:
      - /你的数据目录:/data
```

```bash
docker compose up -d
```

装完打开 `http://你的NAS地址:8790`。

---

## 四、首次配置（5 步，5 分钟）

### 1. 登录（默认 admin / admin）

第一次打开用默认账号登录：

    用户名：admin
    密码：admin

登录后会直接跳「设置账号」页，**必须改成你自己的用户名和密码**才能继续用面板。

### 2. 授权 115（必须，推荐扫码）

```
设置 → 115 授权 →「用 115 App 扫码登录」
手机打开 115 → 扫一扫 → 确认登录 → 面板自动保存 Cookie
```

也可以继续用手工方式（备用）：

Cookie 怎么拿：

```
电脑浏览器打开 https://115.com 并登录
按 F12 → Network（网络）→ 随便点一个请求
找到 Request Headers 里的 Cookie，整段复制
（要包含 UID= 和 CID= 两部分）
```

保存时它会立刻拿这个 Cookie 去读你的 115 根目录验证，
成功会显示「已授权 根目录：手机相册、最近接收…」，失败会直接告诉你原因。

> 这个 Cookie 只要不退出登录就一直有效。换电脑/换网络一般也不用重新弄。

### 3. 选择转存方式

```
设置 → 入库方式 → TRANSFER_BACKEND 保持 auto 即可
```

- **你有 CMS**：填 `CMS 地址` + `CMS API Token`，转存交给 CMS 做（它会整理并生成 strm）
- **你没有 CMS**：什么都不填，走内置直转（它自己接收、识别、建目录、移动）

### 4. 接 Emby（可选但推荐）

```
设置 → EMBY → 地址 http://你的NAS:8096 + API Key
```

API Key 获取：Emby 后台 → 设置 → 高级 → API 密钥 → 新建。

接上之后：

- 概览页显示电影/剧集/单集数量
- 海报墙正常显示
- 新入库后自动刷新媒体库

### 5. 接 Telegram 或飞书（可选）

想用手机看通知、或在聊天里直接选片：

```
设置 → 通知 → Telegram
  bot token：找 @BotFather 建一个 bot 拿 token
  chat id：你的数字 ID（可以用 @userinfobot 查）
```

配好后在 TG 里：

```
搜 怪奇物语          → 返回 5 个候选
回 1                → 转存第 1 个
回 1.3              → 转存第 1 个候选里的第 3 个文件
回 1-4              → 转存第 1 个候选的第 1 到第 4 个文件
订阅                → 把刚才搜到的加入追更
状态                → 看当前订阅和服务状态
```

飞书同理：发片名 → 回候选 → 回序号。

---

## 五、日常使用

### 搜片并转存

```
1. 面板「发现」→ 输入片名 → 搜索
   或 TG 里发「搜 片名」
2. 看候选列表，每行带：
     画质（2160P / 1080P）、体积（GB）、来源（pansou / TG115）
3. 点「转存」，或 TG 里回序号
4. 等几秒到几分钟（看文件大小），事件页会出现转存结果
```

### 订阅与自动追更

```
面板「我的订阅」→ 新增（片名 / 年份 / 电影或剧集）
巡检默认 30 分钟一次：
  找到新集 → 推送给你 → 你回序号 → 自动入库
也可以在订阅详情里打开「自动追更」，找到就自动转，不用你回
```

### 事件页

```
记录每一次 搜索 / 订阅 / 转存
转存失败的记录后面有「重试」按钮，点一下重新提交
失败原因会写在右侧，比如「链接已失效」
```

### 手机上看

```
同一个地址用手机浏览器打开就是移动端布局
程序会自动适配，不需要装 App
```

---

## 六、可选功能

### 查看/重启容器

如果启动时挂了 Docker socket（`-v /var/run/docker.sock:/var/run/docker.sock`），

```
设置 → 容器：可以看到本系统相关容器是否在跑
只有配置里允许的容器才会出现「重启」按钮
```

没挂也没关系，这块会显示「未挂载（可选功能）」，其他功能不受影响。

### 自动备份

```
每天自动把 /data 打包到 /data/backups（保留 7 份）
想放别的盘：设置 BACKUP_DIR 环境变量指向另一块盘
手动备份：设置 → 数据备份 → 立即备份
```

### 媒体统计

接上 Emby 后，概览页会自动显示：电影数、剧集数、单集数，以及最近入库的海报。

---

## 七、升级

```bash
docker pull caoran1314/media-suite:latest
docker compose up -d          # 或 docker restart media-suite
```

数据都在 `/data` 目录里，升级不会丢。

---

## 八、常见问题

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 转存提示「登录超时」 | 115 Cookie 失效 | 设置 → 115 授权，重新粘贴一次 |
| 搜索一直是 0 条 | 搜索源不可用 / 关键词太泛 | 换更具体的片名；或联系部署方确认搜索源 |
| 概览没有媒体统计 | 没接 Emby 或 API Key 错 | 设置 → EMBY 重新填 |
| 转存成功但 Emby 里没有 | 入库方式没配对 / Emby 没刷新 | 看「设置 → 入库方式」，手动点一次刷新 |
| 事件里转存失败「链接已失效」 | 那个分享被原作者删了 | 点「重试」，或重新搜一个新链接 |
| 忘记口令 | — | 删掉 `/data/password.txt` 后重启容器，重新设置 |

---

## 九、命令速查

```bash
# 看日志
docker logs -f media-suite

# 重启
docker restart media-suite

# 看版本
docker inspect --format '{{.Config.Image}}' media-suite

# 备份数据目录（手动）
docker cp media-suite:/data ./media-suite-data-backup

# 升级
docker pull caoran1314/media-suite:latest && docker compose up -d
```
