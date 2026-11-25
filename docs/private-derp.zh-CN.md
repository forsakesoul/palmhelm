# 远程连接与自建 DERP 中继

[English](private-derp.md)

iPhone 上的 Palmhelm 始终连接你 Mac 上的 Bridge。两端都在 Tailscale 里时，Tailscale 优先建立点对点连接；公司网络或蜂窝网络的 NAT 打不通时，会经 DERP 中继转发。默认用 Tailscale 官方中继，通常够用。如果官方中继离你远、延迟高，可以在附近的服务器上自建一个 DERP。

自建 DERP 只改善中继这条路，不保证能打洞成功；Palmhelm 里填的仍是 Mac 的地址，不用改成中继服务器。

## 连接路径与安全边界

```text
iPhone Palmhelm
    │ Bridge API 与事件流，经 Tailscale 承载
    ├── 优先：点对点连接 ────────────────────────┐
    └── 回退：DERP 转发 WireGuard 加密数据包 ─────┤
                                                ▼
                                           Mac Bridge
                                                │
                                         Herdr 中的 agent
```

- DERP 不是出口节点（Exit Node）：手机的其他上网流量不经过它。
- DERP 能看到连接来源、时间和流量大小等元数据，看不到内容：数据由 Tailscale 端到端加密。
- DERP 的 TLS 证书与 Palmhelm 自身的身份校验是两层。设备配对、令牌和 HTTPS 下的证书公钥固定照常生效。
- 换中继不用重新配对，也不要把 Palmhelm 的地址改成中继的公网 IP。
- 中继的公网 IP 可以从 DNS 查到。下文的 `-verify-clients` 只让你 Tailnet 里的设备使用中继，但挡不住扫描或 DDoS。

## 准备

以下步骤适用于一台全新的 Ubuntu 24.04 x86_64 服务器。服务器上已有 Tailscale、网站或同名 systemd 服务时，先检查现有配置，不要直接覆盖。

| 示例 | 换成 |
| --- | --- |
| `derp.example.com` | 你的 DERP 子域名 |
| `203.0.113.10` | 服务器公网 IPv4（这是文档专用示例地址） |
| `your-phone` | 你手机在 Tailscale 里的名称 |
| `private-relay` | 服务器的节点名 |
| 区域 `900` | 自定义区域编号，不要和已有配置冲突 |

不要把 Tailnet 策略、状态文件、证书私钥或配对凭据发到任何公开的地方。

1. 选一台 Mac 和手机访问延迟都低的服务器，两端分别测；Mac 上测的结果代替不了手机走蜂窝网络时的结果。
2. 给子域名加一条 A 记录，指向服务器公网 IPv4。
3. 检查系统、内存、端口占用和防火墙：

```bash
cat /etc/os-release
uname -m
free -h
sudo ss -lntup
sudo ufw status verbose
getent ahostsv4 derp.example.com
```

4. 在云防火墙放通 TCP 80、TCP 443、UDP 3478。80 用于自动申请和续期证书。
5. 系统防火墙开着的话，同样放通这三个端口，并保留现有的 SSH 规则。不要一键放通全部端口，也不要清空规则。

`ss` 显示在监听，不代表公网能访问；云防火墙要单独确认。

## 安装同一版本的 tailscale、tailscaled 与 derper

启用 `-verify-clients` 时，derper 和本机 tailscaled 必须来自同一版本。下面固定用 `v1.104.4`（本文验证用的版本）；以后升级时三个程序一起升。

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl golang-go
mkdir -p "$HOME/derp-build/bin"

TS_VERSION='v1.104.4'
env GOTOOLCHAIN=auto GOMAXPROCS=2 GOFLAGS='-p=1' GOBIN="$HOME/derp-build/bin" \
  go install \
  "tailscale.com/cmd/tailscale@$TS_VERSION" \
  "tailscale.com/cmd/tailscaled@$TS_VERSION" \
  "tailscale.com/cmd/derper@$TS_VERSION"
```

`GOTOOLCHAIN=auto` 会自动下载所需的 Go 版本；限制并发是为了适应小内存服务器。在中国大陆可加 `GOPROXY=https://goproxy.cn,direct`，模块校验仍然生效。

```bash
"$HOME/derp-build/bin/tailscale" version
"$HOME/derp-build/bin/derper" -version

sudo install -m 0755 "$HOME/derp-build/bin/tailscale" /usr/local/bin/tailscale
sudo install -m 0755 "$HOME/derp-build/bin/tailscaled" /usr/local/sbin/tailscaled
sudo install -m 0755 "$HOME/derp-build/bin/derper" /usr/local/bin/derper
```

从源码安装时，版本输出里出现 `ERR-BuildInfo` 不代表编译失败。

## 把服务器加入你的 Tailnet

下面的服务文件只用于还没有 tailscaled 服务的服务器：

```bash
sudo tee /etc/systemd/system/tailscaled.service > /dev/null <<'EOF'
[Unit]
Description=Tailscale node agent
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/tailscaled --state=/var/lib/tailscale/tailscaled.state --socket=/run/tailscale/tailscaled.sock --port=41641
Restart=on-failure
RestartSec=5
RuntimeDirectory=tailscale
StateDirectory=tailscale
StateDirectoryMode=0700
UMask=0077

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now tailscaled
sudo systemctl is-active tailscaled   # 应输出 active

sudo /usr/local/bin/tailscale up --hostname=private-relay --accept-dns=false --accept-routes=false
```

打开终端给出的登录链接，加入 Mac 和 iPhone 所在的 Tailnet；开了设备审批的话，在后台批准。登录链接不要发给别人。

```bash
/usr/local/bin/tailscale status
```

服务器必须能看到要使用中继的设备。用了标签或收紧过访问策略时，要确认它们互相可见。

## 启动带客户端验证的 DERP

```bash
id -u derper >/dev/null 2>&1 || sudo useradd --system --home-dir /var/lib/derper --shell /usr/sbin/nologin derper
sudo -u derper /usr/local/bin/tailscale status
```

如果 `derper` 用户读不到设备列表，先解决账号或 socket 权限，不要去掉 `-verify-clients`。

把 `derp.example.com` 换成你的域名：

```bash
sudo tee /etc/systemd/system/derper.service > /dev/null <<'EOF'
[Unit]
Description=Private Tailscale DERP relay
Wants=network-online.target
Requires=tailscaled.service
After=network-online.target tailscaled.service

[Service]
Type=simple
User=derper
Group=derper
StateDirectory=derper
StateDirectoryMode=0700
WorkingDirectory=/var/lib/derper
ExecStart=/usr/local/bin/derper -hostname=derp.example.com -a=:443 -http-port=80 -stun-port=3478 -certmode=letsencrypt -certdir=/var/lib/derper/certs -c=/var/lib/derper/derper.key -verify-clients -socket=/run/tailscale/tailscaled.sock -home=blank
Restart=on-failure
RestartSec=5
UMask=0077
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now derper
sudo systemctl is-active derper
sudo ss -lntup | grep -E ':(80|443|3478)\b'
```

证书和 DERP 节点密钥在 `/var/lib/derper`，不要外传。`-verify-clients` 按本机 tailscaled 看到的节点身份放行，不是 IP 白名单。

## 验证证书与公网入口

在服务器上：

```bash
curl --fail --show-error --silent --max-time 60 \
  --resolve derp.example.com:443:127.0.0.1 \
  --output /dev/null --write-out 'HTTP %{http_code}\n' \
  https://derp.example.com/derp/probe
```

应输出 `HTTP 200`，第一次访问可能会先申请证书。不要用 `curl -k` 绕过证书错误。

在 Mac 上去掉 `--resolve` 再跑一次，确认公网 443 可达。iPhone 关掉 Wi‑Fi，用 Safari 打开 `https://derp.example.com/robots.txt`，应显示 `User-agent: *` 和 `Disallow: /`。

这几步只验证了 HTTPS 入口，不代表 STUN、中继转发或手机到 Mac 的通路已经可用。

## 在 Tailscale 中加入自建区域

在 Tailscale 后台的访问控制 JSON 编辑器里，把下面的 `derpMap` 合并进顶层。**保留已有的 grants、ACL、SSH 等配置，不要整份替换。** 已经有 `derpMap` 的，把区域合并进去；`900` 已被占用就换一个编号，`RegionID` 一起改。

```json
"derpMap": {
  "OmitDefaultRegions": false,
  "Regions": {
    "900": {
      "RegionID": 900,
      "RegionCode": "private-relay",
      "RegionName": "Private DERP",
      "Nodes": [
        {
          "Name": "private-relay-1",
          "RegionID": 900,
          "HostName": "derp.example.com",
          "IPv4": "203.0.113.10",
          "DERPPort": 443,
          "STUNPort": 3478
        }
      ]
    }
  }
}
```

`OmitDefaultRegions: false` 保留官方中继作为后备。服务器没有公网 IPv6 就不要填 IPv6。客户端会按探测结果自动选择中继。不要把 DERP 放在普通 HTTP 反向代理或 CDN 后面。

## 分层验收

手机连着 Tailscale、走蜂窝网络，在 Mac 上执行：

```bash
tailscale netcheck
tailscale ping --until-direct=false --c=5 --timeout=3s your-phone
tailscale ping --tsmp --until-direct=false --c=3 --timeout=3s your-phone
```

| 看到 | 说明 |
| --- | --- |
| netcheck 列出自建区域及延迟 | 配置已下发、探测可达；不代表转发成功 |
| `via DERP(private-relay)` 的 pong | 已经过自建中继完成 Tailscale 层探测 |
| TSMP 有回复 | WireGuard 加密数据通路已打通 |
| iPhone 上 Palmhelm 显示已连接 | 应用连接已建立；发消息、状态更新和断线重连还要分别试 |

`tailscale ping` 默认会一直等到直连。已有 DERP pong 后出现 `direct connection not established`，不代表中继失败。

## 常见问题

| 症状 | 下一步 |
| --- | --- |
| 子域名解析不到 | 核对 DNS 记录拼写，区分记录还没生效和负缓存 |
| 服务器本机 HTTP 200，外网握手失败 | 检查云防火墙和系统防火墙 |
| 手机 Safari 能打开，`tailscale ping` 超时 | HTTPS 可达不等于 DERP 长连接可用；检查设备在线状态、区域配置、服务日志和节点可见性 |
| 日志出现 `not authorized`、`rejected` | 对照连接的设备身份和时间，检查 `-verify-clients` 与访问策略的可见性 |
| `failed to query local tailscaled` | 检查 tailscaled、socket、`derper` 用户权限和版本是否一致 |
| `missing server name`、`HTTP request to an HTTPS server` | 请求没带 TLS SNI，或用明文 HTTP 访问了 443；单凭这一行不能判断是你的设备被拒 |

```bash
sudo systemctl status tailscaled derper --no-pager
sudo -u derper /usr/local/bin/tailscale status
sudo journalctl -u derper --since "10 minutes ago" --no-pager -n 100
```

求助时只分享时间、错误类别和匿名化的节点名，不要贴完整日志。

### 与代理软件共存

- 本文验证时，Mac 上代理软件的 TUN 模式一直开着；不需要为了用中继关掉它。
- 代理规则里的 `DIRECT` 只决定出口，不代表绕过了 TUN，也不代表 DNS 一定返回本地结果。
- 需要排除时，Tailscale 的网段是 IPv4 `100.64.0.0/10`、IPv6 `fd7a:115c:a1e0::/48`；前者是共享 CGNAT 地址段，排除它可能影响其他用到这一网段的工具。
- 只对 UDP 目标端口 41641 设直连，覆盖不了所有 NAT 映射端口，也不会让路由器开放入站。

## 维护与回退

derper 与 tailscaled 一起升级，并记下版本和校验和；留意证书续期、服务状态和流量。小带宽服务器适合文字和状态流量，不适合传大文件。

回退时，先从 Tailscale 访问策略里只删掉自建区域，保留官方中继和其他策略；确认设备连接恢复后，再停服务：

```bash
sudo systemctl disable --now derper
```

之后按需删掉本次新增的云防火墙规则。不要清空防火墙、删除 SSH 规则，也不要在没确认其他用途前停掉 tailscaled。

## 参考

- [Tailscale 官方 derper 说明](https://github.com/tailscale/tailscale/tree/main/cmd/derper#derp)
- [Tailscale DERP 服务器文档](https://tailscale.com/kb/1232/derp-servers)

各项参数与策略格式请以所选版本的官方文档为准。
