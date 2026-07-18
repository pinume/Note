# 🔐 sing-box 部署教程（AnyTLS）

本文提供 sing-box 1.13.x 部署 AnyTLS 服务端的完整流程，适用于专门运行代理、没有伪装站需求的 VPS。

---

## 0. 关于 AnyTLS 与伪装

AnyTLS 协议自身已具备较强的抗主动探测能力：

- 真实 TLS 握手，可配合 uTLS 模拟 Chrome 指纹。
- 内置 `padding_scheme` 数据包长度填充，对抗机器学习流量分析。
- 认证失败直接断开，不返回任何特征响应。

因此本教程**不部署伪装站点**，由 sing-box 独占 443 端口。如果未来需要在同一域名挂网站，可参考 Trojan + WebSocket 教程添加 Nginx 分流。

> 注意：AnyTLS 协议自 sing-box 1.12.0 开始支持，1.13.x 完全兼容。

---

## 1. 架构

```text
Client → :443 sing-box（AnyTLS + TLS 终止）
              └─ 直连出站
```

**特点：**

- 单进程，无 Nginx，无伪装站。
- sing-box 直接读取证书。
- 对外端口为 `443`。

---

## 2. 准备证书

证书放置路径：

```text
/etc/ssl/cloudflare/929413105.cc.pem
/etc/ssl/cloudflare/929413105.cc.key
```

确保 sing-box 进程对证书有读取权限：

```bash
sudo chmod 644 /etc/ssl/cloudflare/929413105.cc.pem
sudo chmod 600 /etc/ssl/cloudflare/929413105.cc.key
sudo chown root:root /etc/ssl/cloudflare/929413105.cc.*
```

> Cloudflare Origin Certificate 主要用于 Cloudflare CDN 回源，不适合 AnyTLS 直连场景。客户端直连 VPS 时，推荐使用 Let's Encrypt / ZeroSSL 等公开 CA 证书。否则需要配置 `insecure: true` / `skip-cert-verify: true`、跳过证书验证，或手动导入 Cloudflare Origin CA。

---

## 3. 生成密码

推荐使用 16–32 位强随机字符串，密码字段可以是任意字符串，无需 Base64 编码。

```bash
openssl rand -base64 24
# 或
uuidgen | tr -d '-'
```

保存生成的字符串，服务端与客户端必须完全一致。

---

## 4. sing-box 服务端配置

文件：`/etc/sing-box/config.json`

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "inbounds": [
    {
      "type": "anytls",
      "tag": "anytls-in",
      "listen": "::",
      "listen_port": 443,
      "users": [
        {
          "name": "user1",
          "password": "your-anytls-password-here"
        }
      ],
      "padding_scheme": [],
      "tls": {
        "enabled": true,
        "server_name": "929413105.cc",
        "alpn": ["h2", "http/1.1"],
        "certificate_path": "/etc/ssl/cloudflare/929413105.cc.pem",
        "key_path": "/etc/ssl/cloudflare/929413105.cc.key"
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ]
}
```

### 4.1 关键字段说明

| 字段 | 说明 |
| --- | --- |
| `type: anytls` | 固定值，声明协议 |
| `listen: "::"` | 监听所有 IPv6 地址；Linux 双栈通常同时接受 IPv4。若 IPv4 无法连接，可改为 `0.0.0.0` 或检查 `net.ipv6.bindv6only` |
| `listen_port: 443` | 对外端口 |
| `users[].name` | 用户标识，会显示在日志中，可自定义 |
| `users[].password` | 任意字符串，客户端必须一致 |
| `padding_scheme: []` | 空数组表示使用官方默认填充策略 |
| `tls.server_name` | 必须与证书签发域名一致 |
| `tls.alpn` | AnyTLS 使用 TCP，`["h2", "http/1.1"]` 是稳妥写法 |

### 4.2 `padding_scheme` 说明

`padding_scheme` 控制不同阶段的数据包长度填充策略，用于对抗机器学习流量分析。空数组表示启用 sing-box 内置默认策略：

```text
stop=8
0=30-30
1=100-400
2=400-500,c,500-1000,c,500-1000,c,500-1000,c,500-1000
3=9-9,500-1000
4=500-1000
5=500-1000
6=500-1000
7=500-1000
```

绝大多数场景保持空数组即可。除非有特殊流量伪装需求，否则不建议自定义。

---

## 5. systemd 服务

如果通过包管理器（`apt install sing-box`）安装，systemd unit 已自带，可跳过本节。

手动部署时创建 `/etc/systemd/system/sing-box.service`：

```ini
[Unit]
Description=sing-box service
Documentation=https://sing-box.sagernet.org
After=network.target nss-lookup.target

[Service]
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_NET_RAW CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_NET_RAW CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
ExecStart=/usr/local/bin/sing-box -D /var/lib/sing-box -C /etc/sing-box run
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10s
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable sing-box
```

> ⚠️ **重要**：AnyTLS 与 TCP Fast Open 不兼容。systemd 服务中不要添加 `tcp_fast_open`，inbound 配置中也不要添加 `tcp_fast_open: true`。TFO 创建的懒连接会与 AnyTLS 的 SOCKS 包装器冲突，导致握手失败。

---

## 6. 验证与启动

```bash
# 校验配置语法
sudo sing-box check -c /etc/sing-box/config.json

# 启动 / 重启
sudo systemctl restart sing-box
sudo systemctl status sing-box

# 跟踪日志
sudo journalctl -u sing-box -f
```

### 6.1 端口检查

```bash
sudo ss -tlnp | grep ':443\b'
```

应看到 sing-box 在 `*:443` 或 `[::]:443` 监听。

### 6.2 TLS 握手测试

```bash
echo | openssl s_client -servername 929413105.cc -connect 929413105.cc:443 2>/dev/null | openssl x509 -noout -subject -dates
```

应输出证书的 Subject 和有效期。如果失败，检查证书路径权限与 `server_name` 配置。

---

## 7. 客户端配置

### 7.1 sing-box 客户端（JSON）

```json
{
  "outbounds": [
    {
      "type": "anytls",
      "tag": "anytls-out",
      "server": "929413105.cc",
      "server_port": 443,
      "password": "your-anytls-password-here",
      "idle_session_check_interval": "30s",
      "idle_session_timeout": "30s",
      "min_idle_session": 0,
      "tls": {
        "enabled": true,
        "server_name": "929413105.cc",
        "alpn": ["h2", "http/1.1"],
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        }
      }
    }
  ]
}
```

要点：

- `utls.fingerprint` 推荐使用 `chrome`，也可选择 `firefox`、`safari`、`ios`、`edge`、`random` 等。
- 客户端建议开启 uTLS，服务端无需相应配置。

### 7.2 Mihomo / Clash Meta（YAML）

```yaml
proxies:
  - name: "AnyTLS-929413105"
    type: anytls
    server: 929413105.cc
    port: 443
    password: "your-anytls-password-here"
    udp: true
    idle-session-check-interval: 30
    idle-session-timeout: 30
    min-idle-session: 0
    sni: 929413105.cc
    client-fingerprint: chrome
    alpn:
      - h2
      - http/1.1
```

> 如果继续使用 Cloudflare Origin Certificate，客户端需要开启 `skip-cert-verify: true`，但不建议长期使用。更推荐更换为 Let's Encrypt / ZeroSSL 证书。

### 7.3 AnyTLS URI

部分客户端（如 v2rayN v7.14.3+、Mihomo Party）支持 AnyTLS URI：

```text
anytls://your-anytls-password-here@929413105.cc:443?sni=929413105.cc#AnyTLS-929413105
```

---

## 8. 常见问题

**`TLS handshake: context deadline exceeded`？**

通常是客户端配置错误。检查 `server_name` / `sni` 是否与证书域名一致、密码是否完全一致，以及防火墙是否放行 443。

**`TLS handshake: EOF`？**

通常是客户端 uTLS 指纹与服务端 TLS 库不兼容，或 ALPN 协商失败。尝试切换 uTLS 指纹，或临时关闭 uTLS 测试。

**`unknown user password: fallback disabled`？**

密码不匹配且未配置 fallback。AnyTLS inbound 不支持 fallback 字段，密码错误时会直接断开。

**启动时提示 schema 校验失败？**

sing-box 1.13 的配置校验更严格。检查字段拼写，并确认未添加 AnyTLS 不支持的 `transport`、`fallback` 等字段。

**浏览器访问 `https://929413105.cc` 会显示什么？**

TLS 握手成功后连接会断开，浏览器通常显示连接重置或空白。这是 AnyTLS 的正常行为，不需要伪装站。

**能否与 Trojan / VLESS 共用 443 端口？**

不能。AnyTLS 接管整个 443 TCP 流量。如需多协议共用，需要在前方使用 Nginx 按 SNI 或 ALPN 分流。通常建议使用一台 AnyTLS 专用服务器。

---

## 9. AnyTLS 与 Trojan + WebSocket 对比

| 维度 | AnyTLS | Trojan + WebSocket |
| --- | --- | --- |
| 协议成熟度 | 较新（2024–2025），sing-box 1.12+ | 成熟，广泛支持 |
| 伪装站需求 | 不需要 | 强烈推荐 |
| CDN 中转 | 不支持 | 支持（WS over TLS） |
| 抗主动探测 | 强，认证失败静默断开 | 需依赖 fallback 伪装站 |
| 抗流量分析 | 强，内置 padding | 较弱，WS over TLS 有特征 |
| TCP Fast Open | 不兼容 | 兼容 |
| 客户端支持广度 | sing-box / Mihomo / v2rayN / Stash | 几乎全平台 |
| 推荐场景 | 直连 VPS / 专用代理机 | 需要 CDN 中转 / 同域名挂网站 |

---

## 10. 参考链接

- [sing-box AnyTLS inbound 文档](https://sing-box.sagernet.org/configuration/inbound/anytls/)
- [AnyTLS 协议参考实现](https://github.com/anytls/anytls-go)
- [Mihomo AnyTLS 配置](https://wiki.metacubex.one)
