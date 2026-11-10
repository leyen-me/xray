# Xray REALITY 一体化 VPN 节点

> 一套可直接复用的自建翻墙节点方案：**Xray VLESS + REALITY + XTLS Vision**，
> 用 nginx `ssl_preread` 做 SNI 分流，让「节点」和「HTTPS 订阅」共用 443，
> 附带客户端分流规则、订阅服务、流量统计与自动化运维。
>
> 特点：单文件静态二进制、不依赖内核模块、1 核 1G 可跑、抗主动探测。

---

## 目录

- [1. 它解决什么问题](#1-它解决什么问题)
- [2. 架构](#2-架构)
- [3. 端口与组件](#3-端口与组件)
- [4. 快速开始](#4-快速开始)
- [5. 手动部署](#5-手动部署)
- [6. 订阅服务与流量上报](#6-订阅服务与流量上报)
- [7. 客户端配置与分流](#7-客户端配置与分流)
- [8. 自动化运维](#8-自动化运维)
- [9. 流量与计费 FAQ](#9-流量与计费-faq)
- [10. 分流的正确姿势](#10-分流的正确姿势)
- [11. 安全清单](#11-安全清单)
- [12. 排障速查](#12-排障速查)
- [13. 已知限制 / Roadmap](#13-已知限制--roadmap)
- [14. License](#14-license)

---

## 1. 它解决什么问题

自建代理的常见痛点，以及本方案的选择：

| 痛点 | 本方案 |
|------|--------|
| 传统协议（VMess/SS）易被主动探测和封锁 | **VLESS + REALITY**：借用真实网站的 TLS 握手，无自签证书特征 |
| WireGuard / OpenVPN 在老内核要 DKMS 编译 | Xray 是**静态二进制**，不碰内核模块，3.10 内核也能跑 |
| 订阅地址用明文 HTTP + 奇怪端口，容易被墙/被扫 | 订阅和节点**共用 443**，nginx 按 **SNI** 分流，全程 TLS |
| 伪装目标选错，越用越慢 | 部署时**实测握手耗时**挑 dest，并定时复测 |
| 订阅服务被扫描器慢连接卡死 | 多线程 + 连接超时 + 看门狗自动重启 |
| 不知道流量用到哪了 | `vnstat` + 计费周期对齐 + 订阅头显示已用/剩余 |

**不适用**：需要「住宅 IP 伪装」的场景。机房 IP 的 ASN/`hosting` 标记改不了，
本方案只能解决加密与抗封锁，不解决「被网站判定为机房 IP」。

---

## 2. 架构

```text
                         ┌──────────────────────────── VPS ────────────────────────────┐
                         │                                                             │
  Client                 │   nginx stream :443  (ssl_preread，只看 SNI，不解密)         │
  (Mihomo /              │        │                                                    │
   Shadowrocket)         │        ├─ SNI = <DOMAIN>  ──▶ 127.0.0.1:8443  Caddy         │
        │                │        │                         └─ TLS(LE 证书) + 反代     │
        │  443/tcp       │        │                            └─ 127.0.0.1:34817     │
        │  VLESS+REALITY │        │                               sub_server.py        │
        └───────────────▶│        │                                                    │
                         │        └─ 其他 SNI        ──▶ 127.0.0.1:8444  Xray REALITY │
                         │                                       │                    │
                         │                                       ▼                    │
                         │                                  freedom (UseIPv4)         │
                         │                                       │                    │
                         └───────────────────────────────────────┼────────────────────┘
                                                                 ▼
                                                              Internet
```

要点：

- 对外**只暴露 443**（外加 80 做 ACME）。
- nginx 只看 TLS ClientHello 里的 SNI：
  - 命中订阅域名 → 转给 Caddy，走正常 HTTPS 网站；
  - 其他 SNI（客户端伪装成 `www.bing.com`）→ 转给 Xray REALITY。
- 被动探测者直接访问 `https://<IP>/` 看到的是 nginx 转发的真实目标站点证书，
  与正常网站无异。
- Xray 只监听 `127.0.0.1:8444`，公网无法直连，减少攻击面。

> 进阶：如果不想用 nginx，也可以让 Xray 直接监听 `0.0.0.0:443`，
> 但那样订阅就没法和节点共用 443，需要单独端口（更容易被扫）。

---

## 3. 端口与组件

| 端口 | 监听 | 用途 | 是否对公网 |
|------|------|------|-----------|
| 443/tcp | nginx stream | 节点 + 订阅统一入口（SNI 分流） | ✅ |
| 80/tcp | Caddy | ACME HTTP-01 签证书 | ✅ |
| 8443/tcp | Caddy | HTTPS 订阅（可只监听内网） | ⚠️ 建议仅本机/反代 |
| 8444/tcp | Xray | REALITY 入站 | ❌ 仅 127.0.0.1 |
| 34817/tcp | sub_server.py | 订阅内容（token 路径） | ❌ 仅 127.0.0.1 |
| 22/tcp | sshd | 运维 | ✅（建议限制来源） |

| 组件 | 作用 |
|------|------|
| `xray` | VLESS + REALITY 入站，`freedom` 出站 |
| `nginx` | `stream` 模块按 SNI 把 443 分给 Caddy / Xray |
| `caddy` | 自动申请 Let's Encrypt 证书，反代订阅服务 |
| `sub_server.py` | 多线程 HTTP 订阅服务，输出 Clash YAML + 流量头 |
| `vnstat` | 网卡流量统计，供计费周期使用 |
| systemd timers | dest 复测、订阅看门狗、IP 同步、流量同步 |

### 仓库结构

```text
xray-reality-stack/
├── README.md
├── deploy-xray-reality.sh                     # 一键部署（会生成新密钥）
├── config/
│   ├── xray-server.json          # 服务端配置模板
│   ├── nginx-stream.conf         # 443 SNI 分流
│   ├── Caddyfile                 # 订阅 HTTPS
│   ├── 99-xray-tune.conf         # sysctl 调优
│   └── clash.yaml.tpl            # 客户端订阅模板
├── scripts/
│   ├── sub_server.py             # 订阅服务（多线程）
│   ├── sub-watchdog.sh           # 订阅看门狗
│   ├── xray-retune.sh            # 伪装目标复测/切换
│   ├── endpoint-sync.sh          # 公网 IP → DDNS/订阅
│   └── traffic-sync.sh           # 流量统计
└── systemd/
    ├── sub.service
    ├── sub-watch.service / .timer
    ├── xray-retune.service / .timer
    ├── endpoint-sync.service / .timer
    └── traffic-sync.service / .timer
```

> README 只描述原理与步骤；脚本需从原机器复制到对应目录后再提交。
> 提交前务必确认 `config/` 和 `scripts/` 里没有真实 UUID / 私钥 / token。

---

## 4. 快速开始

```bash
# 1) 上传仓库
scp -r xray-reality-stack root@<SERVER_IP>:/root/

# 2) 一键部署（自动测速挑伪装目标）
cd /root/xray-reality-stack
chmod +x deploy-xray-reality.sh
sudo ./deploy-xray-reality.sh

# 3) 去云厂商安全组放行 443/tcp（和 80/tcp，若用域名订阅）
```

脚本完成后会打印：

```text
协议        : VLESS + REALITY (XTLS Vision)
伪装目标    : www.bing.com (自动测速选出)
节点端口    : 443/tcp
订阅链接    : https://<DOMAIN>/sub/<SUB_TOKEN>/clash.yaml
```

> ⚠️ 部署脚本会**重新生成** UUID / 密钥 / shortId，会破坏已有客户端配置。
> 已经上线后只想换伪装目标，请用 `xray-retune.sh`，不要重跑 `deploy-xray-reality.sh`。

---

## 5. 手动部署

### 5.1 依赖

```bash
# Debian/Ubuntu
apt update && apt install -y curl openssl python3 vnstat nginx caddy ufw
# CentOS/RHEL
yum install -y curl openssl python3 nginx
```

需要：`curl`、`openssl`、`python3`、`vnstat`、`nginx`（含 stream 模块）、`caddy`。

### 5.2 安装 Xray

```bash
curl -fsSL -o /tmp/install-release.sh \
  https://github.com/XTLS/Xray-install/raw/main/install-release.sh
bash /tmp/install-release.sh install
xray version   # 建议 v26.x
```

### 5.3 生成凭据

```bash
xray x25519          # 输出 PrivateKey / Password(PublicKey) / Hash32
xray uuid            # UUID
openssl rand -hex 8  # shortId
openssl rand -hex 16 # 订阅 token
```

`xray x25519` 三行的含义（v26 起）：

| 输出 | 用途 |
|------|------|
| `PrivateKey` | 服务端 `realitySettings.privateKey`，**绝密** |
| `Password (PublicKey)` | 客户端公钥（Mihomo 里叫 `public-key`） |
| `Hash32` | 仅校验用，**哪边都不要填** |

### 5.4 服务端配置

`/usr/local/etc/xray/config.json`：

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "listen": "127.0.0.1",
      "port": 8444,
      "protocol": "vless",
      "settings": {
        "clients": [{ "id": "<UUID>", "flow": "xtls-rprx-vision" }],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.bing.com:443",
          "xver": 0,
          "serverNames": ["www.bing.com"],
          "privateKey": "<PRIVATE_KEY>",
          "shortIds": ["<SHORT_ID>"]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"],
        "routeOnly": false
      }
    }
  ],
  "outbounds": [
    { "protocol": "freedom", "tag": "direct", "settings": { "domainStrategy": "UseIPv4" } },
    { "protocol": "blackhole", "tag": "block" }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [{ "type": "field", "ip": ["geoip:private"], "outboundTag": "block" }]
  }
}
```

```bash
xray run -test -config /usr/local/etc/xray/config.json
systemctl enable --now xray
```

### 5.5 nginx SNI 分流

`/etc/nginx/nginx.conf` 末尾追加：

```nginx
stream {
    map $ssl_preread_server_name $backend {
        <DOMAIN>   127.0.0.1:8443;   # 订阅走 Caddy
        default    127.0.0.1:8444;   # 其余走 Xray REALITY
    }

    server {
        listen 443;
        listen [::]:443;
        ssl_preread on;
        proxy_connect_timeout 5s;
        proxy_timeout 10m;
        proxy_pass $backend;
    }
}
```

```bash
nginx -t && systemctl reload nginx
```

> nginx 只做 TCP 转发，不终止 TLS，所以不需要给它配证书。

### 5.6 Caddy（订阅 HTTPS）

`/etc/caddy/Caddyfile`：

```caddyfile
{
    https_port 8443
    http_port 80
}

<DOMAIN>:8443 {
    encode gzip
    header {
        Cache-Control "no-store"
        X-Content-Type-Options "nosniff"
    }
    reverse_proxy 127.0.0.1:34817
}
```

Caddy 会自动用 80 端口完成 ACME，签发 Let's Encrypt 证书。

### 5.7 内核与网络调优

`/etc/sysctl.d/99-xray-tune.conf`：

```ini
# PMTU 黑洞探测：跨境链路「连上但极慢/时断时续」的自愈
net.ipv4.tcp_mtu_probing = 1

# TCP 缓冲
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576
net.ipv4.tcp_rmem = 4096 131072 16777216
net.ipv4.tcp_wmem = 4096 16384 16777216

# BBR + fq
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# 队列
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192
net.core.netdev_max_backlog = 8192
```

```bash
sysctl --system
```

### 5.8 防火墙

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
```

> **先 allow 再 enable**，否则会把自己 SSH 断掉。
> ufw 和云厂商安全组是两层，都要放行。

### 5.9 端到端自测

```bash
cat > /tmp/client.json <<'EOF'
{
  "log": { "loglevel": "warning" },
  "inbounds": [{ "listen": "127.0.0.1", "port": 10808, "protocol": "socks",
                 "settings": { "udp": false } }],
  "outbounds": [{
    "protocol": "vless",
    "settings": { "vnext": [{ "address": "127.0.0.1", "port": 443,
      "users": [{ "id": "<UUID>", "encryption": "none", "flow": "xtls-rprx-vision" }] }] },
    "streamSettings": { "network": "tcp", "security": "reality",
      "realitySettings": { "serverName": "www.bing.com", "fingerprint": "chrome",
        "password": "<PUBLIC_KEY>", "shortId": "<SHORT_ID>", "spiderX": "/" } }
  }]
}
EOF

xray run -config /tmp/client.json >/tmp/client.log 2>&1 &
sleep 3
curl -s --socks5-hostname 127.0.0.1:10808 https://api.ipify.org; echo
# 应返回服务器公网 IP
```

---

## 6. 订阅服务与流量上报

`sub_server.py`（多线程，避免慢连接卡死）：

- 只监听 `127.0.0.1:34817`，公网访问统一由 Caddy 反代；
- 路径 `/sub/<SUB_TOKEN>/clash.yaml`，token 不对一律 404；
- 15 秒连接超时，扫描器慢连接不会拖垮服务；
- 读取 `/var/lib/traffic/usage.json`，通过响应头告诉客户端用量：

```http
subscription-userinfo: upload=<rx>; download=<tx>; total=<quota>; expire=<ts>
profile-update-interval: 6
```

systemd 单元 `sub.service`：

```ini
[Unit]
Description=Clash Subscription Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/etc/sub_server.py
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### 流量统计（`traffic-sync.sh`）

- 用 `vnstat` 的累计计数器做**增量**统计，避免网卡计数器回绕；
- 按**计费周期**（不是自然月）重置，`/etc/traffic.conf` 可配：

```ini
TOTAL_BYTES=1099511627776   # 月额度（字节）
IFACE=eth0
CYCLE_RESET_DAY=25          # 每月 25 号
CYCLE_RESET_HOUR_UTC=0
BASE_BYTES=0                # 历史补偿：vnstat 安装前的已用量
BASE_UNTIL_UTC=
ALERT_CMD=                  # 80/90/100% 时执行的命令，消息作为 $1
```

- 达到 80% / 90% / 100% 时各触发一次告警（同一个周期内不重复）。

> 新机器部署时 **`BASE_BYTES` 必须是 0**。它只是本机用来对齐服务商面板的
> 一次性补偿项，写死会导致流量显示虚高。

---

## 7. 客户端配置与分流

生成的 `clash.yaml` 关键部分（Mihomo / Clash Verge Rev / Shadowrocket 通用）：

```yaml
mixed-port: 7890
mode: rule
ipv6: false
geodata-mode: true

dns:
  enable: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  respect-rules: true
  nameserver:
    - https://1.1.1.1/dns-query#Proxy
    - https://8.8.8.8/dns-query#Proxy
  proxy-server-nameserver:
    - https://223.5.5.5/dns-query
    - https://120.53.53.53/dns-query
  nameserver-policy:
    "geosite:private,cn": [https://223.5.5.5/dns-query, https://120.53.53.53/dns-query]

proxies:
  - name: "🇺🇸 <City>-Reality"
    type: vless
    server: <SERVER_IP>
    port: 443
    uuid: <UUID>
    network: tcp
    tls: true
    udp: true
    flow: xtls-rprx-vision
    servername: www.bing.com
    client-fingerprint: chrome
    reality-opts:
      public-key: <PUBLIC_KEY>
      short-id: <SHORT_ID>

rules:
  - GEOSITE,private,DIRECT
  - GEOSITE,category-ads-all,REJECT
  - DOMAIN,<DOMAIN>,DIRECT          # 订阅域名不绕代理，避免死循环
  - DST-PORT,3478,Proxy             # WebRTC/STUN 强制走代理，防 IP 泄露
  - GEOSITE,cn,DIRECT
  - GEOIP,CN,DIRECT,no-resolve
  - IP-CIDR6,::/0,REJECT,no-resolve # 防止 IPv6 绕过
  - MATCH,Proxy
```

**使用**：导入订阅链接 → 更新 → 选节点 → 规则模式 → 开 TUN。

**字段对照（最容易填错的地方）**：

| 概念 | 服务端 | 客户端 | 备注 |
|------|--------|--------|------|
| x25519 私钥 | `privateKey` | — | 绝密 |
| x25519 公钥 | — | `password` / `public-key` | 不是 Hash32 |
| UUID | `clients[].id` | `users[].id` | 双方一致 |
| shortId | `shortIds[]` | `shortId` | 属于列表之一 |
| SNI | `serverNames[]` | `serverName` | 必须一致 |
| 流控 | `flow` | `flow` | `xtls-rprx-vision` |
| 伪装目标 | `dest` | — | 用域名，别固定 IP |

---

## 8. 自动化运维

| Timer | 频率 | 作用 |
|-------|------|------|
| `xray-retune.timer` | 每周日 04:17 | 复测伪装目标，发现限速则自动切换 |
| `sub-watch.timer` | 每 2 分钟 | 订阅健康检查，连续失败 3 次重启 `sub` |
| `endpoint-sync.timer` | 每 5 分钟 | 公网 IP 变化时同步 DDNS 和订阅里的 `server` |
| `traffic-sync.timer` | 每 5 分钟 | 统计流量并写入 `usage.json` |

### 伪装目标复测（`xray-retune.sh`）

```bash
xray-retune.sh --check   # 只测速报告，不改配置
xray-retune.sh           # 测速并择优切换
xray-retune.sh --force   # 强制切到最优
```

原理：对每个候选跑「3 次预热 + 12 次 REALITY 握手采样」，比较**首段/末段均值**。
只看「能不能连」会误判——某些 CDN 边缘前几次很快，连续握手后限速到 10 倍延迟。

| 候选 dest | 实测 |
|-----------|------|
| `www.bing.com` | ✅ 稳定 30~34ms |
| `www.apple.com` | ⚠️ 首 46ms → 连打后 559ms（限速） |
| `www.yahoo.com` | ❌ 320ms（太远） |
| `www.microsoft.com` | ❌ 握手失败 |
| `www.cloudflare.com` | ❌ 握手失败 |

> `xray-retune.sh` **不会**重新生成密钥，只改 `dest` + `serverNames`。
> 绝不能把 `deploy-xray-reality.sh` 挂定时任务。

### 订阅看门狗（`sub-watchdog.sh`）

订阅服务「进程 active、端口 LISTEN，但完全连不上」是典型故障：
单线程 HTTP 服务被扫描器的慢连接占死，`accept` 不再响应。
修复是改多线程 + 超时；看门狗负责兜底：

```bash
cat /var/log/sub-watch.log
systemctl list-timers | grep sub-watch
```

### 日志位置

```text
/var/log/xray/error.log
/var/log/nginx/error.log
/var/log/sub-watch.log
/var/log/xray-retune.log
/var/log/endpoint-sync.log
/var/log/traffic-sync.log
```

---

## 9. 流量与计费 FAQ

### Q1：为什么入站和出站几乎一样大？我又没上传什么东西

因为 VPN 是**两段传输**。你下载 1GB，服务器是：

```text
源站 ──1GB──▶ 服务器 ──1GB──▶ 你的设备
         入站            出站
```

同一份数据在服务器网卡上被计量两次，所以 `入站 ≈ 出站 ≈ 你的实际下载量`。
「入站大」不是别人上传，而是**服务器替你去源站下载**。

### Q2：出站显示得比入站大很多，正常吗？

不正常——纯中转不可能出站是入站的两倍。常见原因：

1. 面板把「总流量 = 入 + 出」显示成了出站（计费口径）；
2. 统计里加了历史补偿（本项目的 `BASE_BYTES` 会并到 `download` 里）；
3. 计费周期与自然月不一致。

判断标准：**看服务器网卡的真实计数器**（`vnstat` / `/proc/net/dev`），
入出应该基本相等。若真出现单向大很多，才需要排查异常。

### Q3：有没有「单向传输」的 VPN？

**技术上没有。** 转发必然一进一出，TCP 还要回 ACK，UDP/QUIC 也有回程。

**计费上有。** 服务商可以：

| 计费方式 | 说明 |
|----------|------|
| 双向计费 | 入 + 出都算，看 1GB 视频扣 2GB 额度 |
| 单向计费 | 只算出站，或取 `MAX(入, 出)`，看 1GB 只扣约 1GB |

以 DMIT 为例：Pro / EB 系列双向计费；T1 系列按 `MAX(IN, OUT)` 单向计费。
买之前一定看清套餐是哪种。

### Q4：我这套做分流了吗？

**服务端没有做**。服务端路由只有一条：拦截 `geoip:private`，其余全部
`freedom` 直出——它只是转发，不区分国内国外。真正的分流在**客户端规则**里
（`GEOSITE,cn,DIRECT` / `GEOIP,CN,DIRECT`）。

如果客户端也没配规则，就是全局代理，国内流量也会绕道服务器，
在「双向计费」下等于白烧一倍额度。

### Q5：为什么服务器有 IPv6，却不给客户端发 v6 节点？

服务器有 v6 ≠ 用户网络有 v6。运营商覆盖率不足时，加了也是连不上。
另外 v6 还可能绕过代理造成泄露，所以客户端配置里直接 `IP-CIDR6,::/0,REJECT`。

---

## 10. 分流的正确姿势

- **服务端分流不影响省流量**：流量已经过了隧道，怎么路由都算在服务器网卡上。
- **要省额度，必须在客户端分流**：国内 + 局域网直连，只有需要的域名走代理。
- **规则顺序**：先 `GEOSITE`（域名）再 `GEOIP`（IP），并对 IP 规则加 `no-resolve`，
  避免每个新域名都触发一次真实 DNS 解析，导致首包卡 2~5 秒。
- **DNS 不要用直连 `1.1.1.1/8.8.8.8`**：被干扰时丢包严重。用
  「国内 DoH 直连 + 国外 DoH 走代理」。
- **防泄露**：开 TUN + 严格路由 + DNS 劫持，关浏览器自带 DoH 和 QUIC，
  WebRTC 走代理或禁用。

---

## 11. 安全清单

- [ ] `privateKey`、UUID、`SUB_TOKEN`、DDNS token 只存服务器本地，权限 `600`；
- [ ] `.gitignore` 排除 `*.token`、`config.json`、`clash.yaml`、`*.log`；
- [ ] 订阅链接里的 token **等同密码**，不要发群、不要截图；
- [ ] 使用 HTTPS 订阅（本项目已用 Caddy + Let's Encrypt）；
- [ ] ufw 默认拒绝入站，只放行必要端口；
- [ ] 云安全组与 ufw 两层都配置；
- [ ] 建议后续加：SSH 密钥登录、禁用密码、fail2ban；
- [ ] 定期轮换 UUID / token（改完记得更新订阅）；
- [ ] 遵守所在地区法律法规，仅用于合法用途。

---

## 12. 排障速查

| 现象 | 排查方向 |
|------|----------|
| 日志 `handshake did not complete successfully` | dest 不兼容，或 password/shortId/SNI/flow 填错 |
| 报 `invalid "password"` | 公钥填错（多半填了 Hash32） |
| 刚用还行、越用越慢 | dest 被限速，跑 `xray-retune.sh` 换目标 |
| 网页首次打开卡几秒 | 客户端 DNS/规则问题，检查 DoH 与规则顺序 |
| 能连但没流量 | 看 routing / freedom 出站 / 云安全组 |
| 完全连不上 | 443 是否放行；ufw 是否误拦；时间误差是否 > 30s |
| 订阅突然超时，本机 curl 也超时 | 订阅服务被慢连接卡死，`systemctl restart sub`，确认是多线程版 |
| 节点 IP 变了 | 等 `endpoint-sync.timer`，或手动跑 `endpoint-sync.sh` |
| 流量显示和面板对不上 | 核对 `CYCLE_RESET_DAY` 和 `BASE_BYTES` |

```bash
systemctl status xray nginx caddy sub
tail -f /var/log/xray/error.log
ss -lntp | grep -E ':443|:8443|:8444|:34817'
ufw status verbose
systemctl list-timers
```

---

## 13. 已知限制 / Roadmap

- [ ] 订阅仍是「token 路径保密」，可再加 Basic Auth / 限速；
- [ ] 机房 IP 无法伪装住宅，需要住宅代理做落地才能根治风控；
- [ ] 中转与落地未分离（单机同时承担，峰值会互相抢带宽）；
- [ ] 未启用后量子签名 `mldsa65`（为兼容 Mihomo / Shadowrocket 暂不启用）；
- [ ] 无 IPv6 节点（取决于用户网络，见 FAQ Q5）；
- [ ] 建议补 SSH 密钥登录 + fail2ban。

---

## 14. License

MIT。仅供学习与合法用途，请遵守当地法律法规。
