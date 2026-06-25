# YAOS — Selflare Docker 自动构建

将 [kavinsood/yaos](https://github.com/kavinsood/yaos)（Obsidian CRDT 实时同步服务端）通过 `selflare` 编译为 `workerd` 原生格式，自动构建成极简 Docker 镜像，推送到 GitHub Container Registry。

---

## 项目结构

```
.
├── .github/workflows/
│   └── build.yaml              # push / PR 时自动构建 + 烟雾测试 + 推镜像
├── server/                     # YAOS 服务端源码（Cloudflare Worker）
│   ├── src/                    # TypeScript 源码
│   ├── wrangler.toml           # Worker 配置（含 Durable Object 绑定）
│   ├── package.json
│   └── package-lock.json
└── README.md
```

---

## 工作原理

每次 push 到 `main` 或推送 `v*` 标签时，GitHub Actions 自动执行：

```
┌─── 环境准备 ──→ checkout → node setup → npm ci → typecheck
│
├─── Workerd 编译 ──→ selflare compile（JS → capnp）
│                    → selflare docker（生成 Dockerfile）
│
├─── 烟雾测试 ──→ 构建本地镜像 → 启动容器 → curl 验证 /api/capabilities → 清理
│
└─── 推送镜像 ──→ ghcr.io/owner/repo:latest + ghcr.io/owner/repo:${{ github.sha }}
```

> **PR 触发时**只执行前三个阶段（环境准备 → 编译 → 烟雾测试），不推送镜像。

---

## 本地部署

### docker-compose.yml

```yaml
services:
  yaos:
    image: ghcr.io/sinoserendipity/docker-yaos:latest
    container_name: yaos-server
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - ./storage/cache:/worker/cache
      - ./storage/do:/worker/do
      - ./storage/r2:/worker/r2
      - ./storage/kv:/worker/kv
      - ./storage/d1:/worker/d1
```

```bash
docker compose up -d
```

访问 `http://<你的服务器IP>:8080` 进入配置页面。

> **存储目录说明：** workerd 会把 Durable Object、Cache 等数据写入 `/worker/*`。只有挂载了卷的目录才会持久化，未挂载的目录使用镜像内的空目录，容器重启后数据消失。

### 纯 Docker

```bash
docker run -d --name yaos-server \
  -p 8080:8080 \
  -v ./storage/cache:/worker/cache \
  -v ./storage/do:/worker/do \
  ghcr.io/sinoserendipity/docker-yaos:latest
```

---

## 本地测试 Workflow（act）

需要在机器上安装 Docker 和 act：

```bash
# 在项目根目录执行
act -j build
```

act 会下载一个 Ubuntu 容器镜像，在本地模拟完整的 GitHub Actions 流程。烟雾测试也会在本地容器中运行。

---

## 技术栈

| 组件 | 作用 |
|------|------|
| [yaos](https://github.com/kavinsood/yaos) | Obsidian CRDT 实时同步服务器 |
| [selflare](https://github.com/JacobLinCool/selflare) | 将 Cloudflare Worker 编译为 workerd capnp 配置 |
| [workerd](https://github.com/cloudflare/workerd) | Cloudflare Workers 开源运行时 |
| [act](https://github.com/nektos/act) | 本地运行 GitHub Actions workflow |
| GHCR | GitHub Container Registry，无需额外配置即可托管镜像 |
