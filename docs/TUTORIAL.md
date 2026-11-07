# FilmShelf 使用教程

> FilmShelf 是 CMS（Cloud Media Sync）的运营面板：负责发现、订阅、候选、通知、用户与授权；
> **转存、整理、strm 生成、Emby 刷新都交给 CMS**。所以部署前你必须已经有一个能正常工作的 CMS。

---

## 一、准备清单

| 需要 | 说明 |
|---|---|
| 一台能跑 Docker 的 NAS / 服务器 | 群晖、威联通、fnOS、Linux 都行 |
| 一个 115 网盘账号 | 转存进你的网盘；账号由 CMS 持有，不需要给 FilmShelf |
| **一个能用的 CMS** | Cloud Media Sync 或兼容实现；必须已配好 115 授权、媒体目录、Emby |
| 一个数据目录 | 挂到容器 `/data`，存配置、日志；**必须持久化** |
| Emby | 配 `EMBY_URL` + `EMBY_API_KEY`，面板用它做媒体统计、海报和跳转播放 |
| 端口 | 面板默认 `8790` |

镜像：

```bash
docker pull caowenrui/media-suite:2.3.0   # 固定版本（推荐）
docker pull caowenrui/media-suite:latest  # 跟随最新
```

> 老教程里的 `caoran1314/media-suite` 已废弃，拉不到。

---

## 二、先把 CMS 准备好

FilmShelf 只负责给 CMS 发「转存这个分享」的请求，剩下都归 CMS。部署前确认：

1. CMS 已部署，能打开管理页；
2. CMS 里已配置 **115 账号**（它自己的授权；和 FilmShelf 无关）；
3. CMS 里已设置 **媒体目录 / strm 输出目录**；
4. CMS 已接入 **Emby**，能刷新媒体库；
5. 记下 CMS 地址，例如 `http://192.168.1.10:9527`；
6. 记下 CMS 的 API Token（Cloud Media Sync 默认 `cloud_media_sync`，以你的配置为准）。

> FilmShelf 调用 `POST /api/cloud/add_share_down?token=...`，body `{"url":"115分享链接"}`。
> 如果你的 CMS 是改过的版本，确认这个接口还在。

**先单独验证 CMS 能转存一单**，再装 FilmShelf；否则面板里所有转存都会失败，很难判断是哪一层的问题。

---

## 三、启动 FilmShelf

```yaml
services:
  media-suite:
    image: caowenrui/media-suite:2.3.0
    container_name: media-suite
    restart: unless-stopped
    ports:
      - "8790:8790"
    environment:
      TZ: Asia/Shanghai
      PORT: 8790
      DATA_DIR: /data
      PANEL_URL: http://192.168.1.10:8790

      # 搜索源：留空只用内置 TG115；有 pansou 就填
      # PANSOU_BASE_URL: http://192.168.1.10:3200
      SEARCH_BACKENDS: pansou,tg115

      # CMS（必填）
      CMS_ENABLED: "true"
      CMS_BASE_URL: http://192.168.1.10:9527
      CMS_API_TOKEN: cloud_media_sync

      # Emby（面板统计/播放跳转用）
      EMBY_URL: http://192.168.1.10:8096
      EMBY_API_KEY: ""

      # 巡检
      WATCH_ENABLED: "true"
      WATCH_INTERVAL_MIN: "30"

      # 面板口令（首次登录后强制修改）
      ADMIN_PASSWORD: "change-me"
    volumes:
      - ./data:/data
```

```bash
docker compose up -d
```

打开 `http://你的NAS:8790`。

---

## 四、首次配置

1. 用 `ADMIN_PASSWORD` 登录，按提示改密；
2. 进 **设置 → 入库**，确认 CMS 地址与 Token；
3. 进 **设置 → Emby**，填 Emby 地址与 API Key；
4. （可选）**设置 → 搜索**：填 pansou 地址；不填则只用内置 TG115；
5. （可选）**设置 → 通知**：Telegram / 飞书；
6. 点 **保存并自检**。

自检里如果「CMS 入库」是红的，说明 `CMS_BASE_URL` / `CMS_API_TOKEN` 不对，或者 CMS 本身没起来。

---

## 五、验证转存

1. 在发现页找一部片，点「转存」；
2. 面板提示「添加转存下载任务成功」即表示 CMS 已接单；
3. 去 CMS 的任务列表确认它开始下载/整理；
4. 等 CMS 生成 strm 并刷新 Emby 后，Emby 里能看到；
5. 回到面板，订阅/候选状态会随巡检更新。

如果第 2 步就失败：
- 400 / `needCms`：面板没配 CMS；
- CMS 返回 token 错误：核对 `CMS_API_TOKEN`；
- CMS 接单但任务一直 `status=1`：问题在 CMS 的 115 授权或 worker，不在 FilmShelf。

---

## 六、日常使用

- **订阅追更**：订阅页加入想追的剧，巡检每 30 分钟找新集，找到后推 TG/飞书；
- **候选**：点订阅卡片上的「候选」，按画质排序选一个回数字，面板调 CMS 转存；
- **用户/卡密**（付费版）：控制台里建用户、发卡密、设到期；
- **媒体库权限**：按用户勾选可见库（需要 Emby 配置）；
- **同播检测**：配置后可在概览看到异常播放。

---

## 七、升级 / 备份 / 回滚

```bash
# 升级
docker compose pull && docker compose up -d

# 备份（数据都在 ./data）
tar -czf media-suite-data-$(date +%F).tgz ./data

# 回滚：把 image 改回旧 tag，再 up -d
```

`./data` 里有 `local-config.json`（面板里改过的配置）和日志；迁移机器时一起带走。

---

## 八、常见问题

**Q：没有 CMS 能用吗？**
不能。2.3.0 起 CMS 是必需依赖，内置 115 直转已移除。

**Q：CMS 和 FilmShelf 都要配 115 吗？**
只有 CMS 需要。FilmShelf 不接触 115 账号。

**Q：转存成功但 Emby 看不到？**
看 CMS 的 strm 输出目录是否在 Emby 媒体库内，以及 CMS 是否真的刷新了 Emby。这属于 CMS 侧配置。

**Q：面板能播放吗？**
面板只做跳转，实际播放由 Emby 完成；播放地址指向 CMS 生成的 strm。

**Q：忘记密码？**
停容器，删 `./data/auth.json`（或改 `ADMIN_PASSWORD` 后重建），再启动会回到初始口令。
