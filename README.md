# musicgrabber-lzcapp

MusicGrabber（自托管音乐获取服务）的懒猫微服 LPK 打包。

**上游**: https://gitlab.com/g33kphr33k/musicgrabber （Unlicense）
**打包仓库**: 本仓库只存 LPK 配置与构建脚本，**不存储上游源码** —— GitHub Action 构建时按
`UPSTREAM.conf` 固定 commit 拉取上游源码，基于上游 Dockerfile 构建应用镜像并内嵌进 LPK。

## 结构

| 文件 | 说明 |
| --- | --- |
| `UPSTREAM.conf` | 上游仓库与固定 commit/tag（升级 = 改这里的 commit + bump `package.yml.version`） |
| `build.sh` | 构建前拉取上游源码到 `./upstream`（Docker 构建上下文） |
| `docker/Dockerfile` | 基于上游 Dockerfile（Bento4 编译 + Python 运行时 + 浏览器栈），仅 COPY 路径指向 `upstream/` |
| `lzc-build.yml` | LPK 构建配置（buildscript + embed 镜像） |
| `lzc-manifest.yml` | 应用清单：单服务 musicgrabber（embed 镜像，shm 2g） |
| `.github/lazycat-action.yml` | Action 配置：`version_source: git`，只发布喵喵商店 |

## 部署拓扑

- **musicgrabber**：上游 FastAPI 应用（8080），SQLite 任务库/设置持久化在 `/data`，
  音乐库 `/music`，Soulseek 完成目录占位 `/downloads`。按上游 compose 建议 `shm_size: 2g`。
- 无强制账号体系：默认单用户模式免密；在设置页添加第二个账号后切换为多用户登录模式。

## 升级上游

1. 改 `UPSTREAM.conf` 里的 `UPSTREAM_COMMIT`（和 `UPSTREAM_TAG`）
2. `package.yml` 的 `version` 同步为上游版本号
3. 提交推送，Action 自动构建发布；或 `gh workflow run lazycat.yml`

## License

- 上游代码: Unlicense（https://gitlab.com/g33kphr33k/musicgrabber）
- 打包配置: Unlicense
