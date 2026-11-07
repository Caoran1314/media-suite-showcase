# FilmShelf

**自建 Emby 运营面板**：发现 → 订阅 → CMS 转存 → 入库 → 通知。  
用户、卡密、到期、媒体库权限、同播检测，也都在同一个面板里。

[![Version](https://img.shields.io/badge/version-2.3.1-3ddc97?style=flat-square)](CHANGELOG.md)
[![License](https://img.shields.io/badge/license-商业授权-blue?style=flat-square)](LICENSE)
![Docker](https://img.shields.io/badge/docker-caowenrui%2Fmedia--suite-2496ed?style=flat-square)

> 2.3.0 起 **Cloud Media Sync（CMS）是必需依赖**：转存、整理、strm 生成、Emby 刷新都由 CMS 完成。  
> 没有 CMS 时面板能登录和浏览，但无法转存入库。

![概览](docs/screenshots/01-overview.png)

## 它做什么

- **说个片名**：面板、Telegram Bot、飞书都能加片
- **自动找源**：pansou 多实例 + TG115 频道，按画质/体积排序
- **回个序号**：候选交给 CMS 转存、识别、重命名、写 strm、刷新 Emby
- **订阅追更**：剧集自动补新集，Telegram / 飞书推送
- **运营**：本地用户、卡密、到期、媒体库权限、同播检测
- **授权**：免费版可用；永久版 ¥66，一次买断

## 部署

```yaml
services:
  media-suite:
    image: caowenrui/media-suite:2.3.1
    container_name: media-suite
    restart: unless-stopped
    ports: ["8790:8790"]
    environment:
      TZ: Asia/Shanghai
      DATA_DIR: /data
      CMS_ENABLED: "true"
      CMS_BASE_URL: http://你的CMS地址:9527
      CMS_API_TOKEN: 你的CMS令牌
      EMBY_URL: http://你的Emby地址:8096
      EMBY_API_KEY: 你的EmbyKey
    volumes:
      - ./data:/data
```

```bash
docker compose up -d
```

打开 `http://你的NAS:8790`，首次 `admin/admin` 登录并改密，然后到 **设置 → 入库与 Emby** 填 CMS 与 Emby。

## 文档

- [部署教程](docs/TUTORIAL.md)
- [环境变量](docs/ENV.md)
- [依赖与架构](docs/DEPENDENCIES.md)
- [更新日志](CHANGELOG.md)

## 许可

商业授权，仅限自用部署。禁止复制、转售、二次开发、反向工程。详见 [LICENSE](LICENSE)。
