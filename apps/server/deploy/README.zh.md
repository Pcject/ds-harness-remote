# 自建部署示例（Server + coturn）

> **注意**：示例中的 `pcject/dsh-remote:v0.6.2` 是个人基于当前 WebRTC/TURN 版本构建的测试镜像，**未经过完整的检查与验证**，仅用于演示。正式部署请优先自行构建镜像（见「自行构建镜像」），或使用上游正式发布的镜像。

本目录提供一个最小可用的 Docker Compose 部署示例，用于运行 `pcject/dsh-remote:v0.6.2`（含 Server 端 WebRTC signaling / TURN credential 支持）和 coturn，实现 `LAN -> P2P -> TURN -> Relay` 四路连接。

如果在其它环境使用上游镜像或自己构建的版本，只需修改 `compose.yaml` 里的 `image`。

## 文件

| 文件 | 说明 |
| --- | --- |
| `compose.yaml` | Server + coturn 两个服务 |
| `env.example` | 环境变量模板，复制为 `.env` 后修改 |
| `README.zh.md` | 本说明 |

## 前置条件

- 一台有公网 IP 的 Linux 服务器，安装 Docker 与 Docker Compose v2
- 一个域名（例如 `remote.example.com`）解析到该服务器，用于 HTTPS / WSS
- 开放端口：

| 端口 | 协议 | 用途 |
| --- | --- | --- |
| 80 / 443 | TCP | 反向代理、HTTPS / WSS |
| 3478 | TCP + UDP | STUN / TURN |
| 49160–49200 | UDP | TURN relay 端口段 |

`8080` 只绑定在本机，由反向代理访问，不要直接暴露公网。

> coturn 使用 `network_mode: host`：TURN 需要真实源 IP 和 UDP relay 端口段，Docker 端口映射不可靠。该示例面向 Linux；macOS/Windows 的 Docker Desktop 不支持 host 网络。

## 自行构建镜像（推荐）

示例镜像为个人构建，未经完整检查与验证。建议从对应源码自行构建，再替换 `compose.yaml` 中的 `image`：

```bash
# 在仓库根目录执行：Dockerfile 需要仓库根作为构建上下文
git clone https://github.com/Pcject/ds-harness-remote.git
cd ds-harness-remote
# 切换到包含 WebRTC/TURN 支持的分支（PR 合并前用 feat/server-webrtc-turn）
git checkout feat/server-webrtc-turn

docker build -f apps/server/Dockerfile -t dsh-remote:local .
```

然后把 `compose.yaml` 中的：

```yaml
image: pcject/dsh-remote:v0.6.2
```

改为：

```yaml
image: dsh-remote:local
```

即可启动。上游正式镜像在 WebRTC/TURN PR 合并并发布前不包含该功能，不要直接替换成上游旧镜像。

## 快速启动

```bash
# 1. 复制示例文件到部署目录（或直接在本目录操作）
cp env.example .env
# 编辑 .env：账号密码、域名、公网 IP、TURN 密钥

# 2. 生成两个随机值
openssl rand -hex 32   # DSH_SERVER_TURN_SECRET
openssl rand -base64 24  # 也可以用来生成密码

# 3. 启动
docker compose --env-file .env up -d
docker compose ps
docker compose logs -f server
```

启动后浏览器访问 `https://remote.example.com`，用 `.env` 中的账号密码登录。Host 与客户端填写同一个 Server 地址和账号。

## 反向代理

Server 的登录、REST 和 Control WebSocket 都在 `127.0.0.1:8080`。用 Caddy 最简单：

```caddyfile
remote.example.com {
    reverse_proxy 127.0.0.1:8080
}
```

Caddy 会自动处理 HTTPS 和 WebSocket。使用 Nginx 时需要：

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 300s;
}
```

注意：

- `.env` 中的 `DSH_SERVER_PUBLIC_URL` 必须与浏览器实际访问的 origin 完全一致（含 `https://`、域名和非默认端口），否则会出现 Origin 校验失败或静态资源 403。
- 反向代理的 WebSocket 空闲超时要大于 75 秒。
- 如需按真实客户端 IP 限流，设置 `DSH_SERVER_TRUSTED_PROXY_IPS` 为代理在 Server 容器内看到的 peer IP（通常是 Docker 网关或代理容器 IP，可用 `docker network inspect` 查看）；不设置时所有客户端共用一个限流桶。

## 验证

```bash
# Server 健康检查（期望 {"status":"ok"}）
curl -fsS http://127.0.0.1:8080/healthz

# coturn 启动日志
docker compose logs -f coturn
```

可选：使用 coturn 自带工具验证短期 TURN credential：

```bash
SECRET='与 .env 中 DSH_SERVER_TURN_SECRET 相同'
USERNAME="$(( $(date +%s) + 300 )):test"
CREDENTIAL=$(printf %s "$USERNAME" | openssl dgst -binary -sha1 -hmac "$SECRET" | base64)
docker compose exec coturn turnutils_uclient -u "$USERNAME" -w "$CREDENTIAL" -p 3478 -y "$TURN_EXTERNAL_IP"
```

## 不要 TURN，只保留 LAN / P2P / Relay

1. 从 `compose.yaml` 删除 `coturn` 服务；
2. 在 `.env` 中清空 `DSH_SERVER_TURN_URLS`、`DSH_SERVER_TURN_SECRET`（以及可选的 `DSH_SERVER_STUN_URLS`）。

此时 Server 不宣告 `transport.turn`，客户端在 WebRTC 打不通时自动回退加密 Relay。注意：

- 同局域网或可直连时仍走 LAN / P2P；
- 跨 NAT 打不通时走 Relay，延迟和服务器带宽会受影响；
- 大文件上传在 Relay 路径受 20 MiB 限制，只有 LAN / 直连 P2P 是 1 GiB。

## 常见问题

- **客户端一直显示 Relay**：P2P 打洞失败会自动回退；检查 STUN 是否可达、NAT 类型、UDP 是否被网络禁用。
- **TURN 连不上**：优先检查 3478 TCP/UDP 与 relay 端口段是否放行；云服务器 1:1 NAT 必须设置 `--external-ip`；TURN 域名要能解析到该公网 IP。
- **登录/页面异常**：核对 `DSH_SERVER_PUBLIC_URL` 与访问地址完全一致。
- **所有客户端共用限流**：反向代理场景未设置 `DSH_SERVER_TRUSTED_PROXY_IPS`，或填错代理 IP。
- **coturn 反复重启**：确认 `DSH_SERVER_TURN_SECRET` 非空；compose 使用 `:?` 在缺失时直接报错。
