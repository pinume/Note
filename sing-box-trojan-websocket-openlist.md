# 🔐 sing-box 部署教程（Trojan + WebSocket + OpenList 伪装）

本文提供两种架构方案，**任选其一**部署即可。

---

## 1. 方案 A：Nginx 监听 443，WebSocket 分流给 sing-box

### 1.1 架构图

```text
Client → :443 Nginx（TLS 终止）
              ├─ WS 路径 /your-ws-path → sing-box:8443（Trojan）
              └─ 其他路径               → OpenList:5244（伪装站）
```

**特点：**

- 伪装完整，浏览器直接访问域名可以看到真实 OpenList 网页。
- 证书由 Nginx 管理，sing-box 不读取证书文件。

### 1.2 Nginx 配置

文件：`/etc/nginx/sites-available/929413105.cc.conf`

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name 929413105.cc;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name 929413105.cc;

    ssl_certificate     /etc/ssl/cloudflare/929413105.cc.pem;
    ssl_certificate_key /etc/ssl/cloudflare/929413105.cc.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # WebDAV / 上传需要较大的 body
    client_max_body_size 20G;

    # WebSocket 路径优先匹配，转给 sing-box
    location /your-ws-path {
        if ($http_upgrade != "websocket") {
            return 404;
        }
        proxy_pass http://127.0.0.1:8443;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }

    # 其他请求全部转给 OpenList
    location / {
        proxy_pass http://127.0.0.1:5244;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # OpenList 自身也有 WebSocket（文件上传进度等）
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

启用站点：

```bash
sudo ln -s /etc/nginx/sites-available/929413105.cc.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 1.3 sing-box 配置

文件：`/etc/sing-box/config.json`

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "inbounds": [
    {
      "type": "trojan",
      "tag": "trojan-ws-in",
      "listen": "127.0.0.1",
      "listen_port": 8443,
      "users": [
        {
          "name": "user1",
          "password": "your-trojan-password-here"
        }
      ],
      "transport": {
        "type": "ws",
        "path": "/your-ws-path"
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

**要点：**

- TLS 由 Nginx 处理，sing-box inbound **不配置** `tls` 字段。
- `listen: "127.0.0.1"` 表示只监听本地，仅 Nginx 可以转入流量。
- WebSocket 路径必须与 Nginx 中的 `location` 完全一致。

---

## 2. 方案 B：sing-box 直接占用 443，fallback 给 OpenList

### 2.1 架构图

```text
Client → :443 sing-box（TLS 终止）
              ├─ Trojan 流量（WS 路径 /your-ws-path）
              └─ 非 Trojan / 密码错误 → fallback → OpenList:5244（伪装站）
```

**特点：**

- 架构简单，只有一个对外进程，无需 Nginx。
- 性能略好，减少一跳本地转发。
- 未来添加 Reality / Hysteria2 时，可直接在 sing-box 增加 inbound。
- 缺点是 `X-Forwarded-For` 等请求头丢失，OpenList 日志中客户端 IP 均为 `127.0.0.1`。

### 2.2 sing-box 配置

文件：`/etc/sing-box/config.json`

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "inbounds": [
    {
      "type": "trojan",
      "tag": "trojan-in",
      "listen": "::",
      "listen_port": 443,
      "users": [
        {
          "name": "user1",
          "password": "your-trojan-password-here"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "929413105.cc",
        "alpn": ["h2", "http/1.1"],
        "certificate_path": "/etc/ssl/cloudflare/929413105.cc.pem",
        "key_path": "/etc/ssl/cloudflare/929413105.cc.key"
      },
      "transport": {
        "type": "ws",
        "path": "/your-ws-path"
      },
      "fallback": {
        "server": "127.0.0.1",
        "server_port": 5244
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

**要点：**

- `listen: "::"` 监听所有 IPv6 地址，Linux 双栈默认通常会同时接受 IPv4。
- sing-box 自行读取证书文件，确保进程有读取权限（`chmod 644` 或将文件所有者改为 sing-box 运行用户）。
- `fallback` 在 Trojan 密码错误、非 WebSocket 请求、WebSocket 路径不匹配时触发，将请求透传给 OpenList。

---

## 3. 部署与验证（两个方案通用）

### 3.1 sing-box systemd 服务

如果通过包管理器安装 sing-box（`apt install sing-box` 或官方 deb），systemd unit 已自带，可跳过本节。

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

### 3.2 验证配置并启动

```bash
# 校验 sing-box 配置语法
sudo sing-box check -c /etc/sing-box/config.json

# 方案 A：校验 Nginx 配置
sudo nginx -t

# 启动 / 重启
sudo systemctl restart sing-box
sudo systemctl status sing-box

# 跟踪日志
sudo journalctl -u sing-box -f
```

### 3.3 端口检查

- 方案 A：`80`（Nginx）、`443`（Nginx）、`8443`（sing-box，本地）、`5244`（OpenList，本地）。
- 方案 B：`443`（sing-box）、`5244`（OpenList，本地）。

```bash
sudo ss -tlnp | grep -E ':(80|443|8443|5244)\b'
```

### 3.4 联通性测试

| 测试项 | 方案 A | 方案 B |
| --- | --- | --- |
| 浏览器访问 `https://929413105.cc` | 显示 OpenList 界面 | 显示 OpenList 界面 |
| 客户端 Trojan 连接 | 经过 Nginx → sing-box | 经过 sing-box 自身 |
| `curl -I https://929413105.cc` | 200 OK，Server: nginx | 200 OK，无明显 sing-box 特征 |

---

## 4. OpenList 伪装站加固建议

1. 不要让 OpenList 处于空白状态。进入 `/@manage` 配置至少一个存储，使首页看起来在真实使用。
2. 修改站点标题与 favicon，避免保留默认外观。
3. 为管理后台 `/@manage` 设置强密码。方案 A 还可以通过 Nginx 添加 IP 白名单。
4. 不要将 OpenList 端口暴露到公网，仅绑定 `127.0.0.1:5244`。

---

## 5. 两种方案对比

| 维度 | 方案 A（Nginx 443） | 方案 B（sing-box 443） |
| --- | --- | --- |
| 证书管理 | Nginx | sing-box |
| 浏览器直访域名 | 真实网页 | fallback 后也是真实网页 |
| 客户端 IP 真实性 | Nginx 可注入 `X-Forwarded-For` | OpenList 看到 `127.0.0.1` |
| 同 IP 运行其他网站 | 容易，添加 `server` 块 | 较麻烦，全部经过 fallback |
| 添加 Reality | 无法直接添加，443 被 Nginx 占用 | 容易，添加 inbound 即可 |
| 添加 Hysteria2（UDP） | 不冲突，可以添加 | 不冲突，可以添加 |
| 修改 WebSocket 路径 | 修改 Nginx 与 sing-box 两处 | 只修改 sing-box |
| 进程数 | Nginx + sing-box | 仅 sing-box |
| 推荐场景 | 同 IP 多站点 / 需要真实客户端 IP | 极简部署 / 未来扩展 Reality |

---

## 6. 版本注意事项（sing-box 1.13.11）

本文配置按 1.12+ 新格式编写，可在 1.13.x 上使用。

### 6.1 ALPN 字段更严格

1.13 对 ALPN 协商更严格。方案 B 中的 `["h2", "http/1.1"]` 是为 WebSocket 兼容性保留的稳妥写法。

如果只使用 WebSocket 传输，可以精简为：

```json
"alpn": ["http/1.1"]
```

### 6.2 fallback 字段与替代方案

1.13 中 Trojan inbound 的 `fallback` 字段仍受支持。更现代的方式是在 inbound 上开启 `sniff`，再用 `route.rules` 将非 Trojan 流量路由到指向 OpenList 的 direct outbound。该方式更灵活，但配置也更复杂。

单 Trojan + 单伪装站场景继续使用 `fallback` 即可。

### 6.3 配置校验更严格

粘贴配置后首先执行：

```bash
sudo sing-box check -c /etc/sing-box/config.json
```

字段拼写、类型或未知字段问题会立即显示。

---

## 7. 常见问题

**浏览器访问域名提示证书错误？**

检查证书类型。如果使用 Cloudflare Origin Certificate，必须通过 CDN 访问才受信任；直连场景应使用 Let's Encrypt。

**客户端无法连接，日志显示 TLS handshake 失败？**

检查 `server_name` 是否与证书域名一致，并确认 sing-box 进程能够读取证书文件。

**客户端连接成功但代理不通？**

检查 WebSocket 路径是否一致。方案 A 还应检查 Nginx `location` 拼写，并查看 sing-box 日志。

**方案 B 中浏览器访问域名时 fallback 未生效？**

检查 OpenList 是否监听 `127.0.0.1:5244`，使用 `docker ps` 查看容器状态，并执行 `curl http://127.0.0.1:5244` 本地测试。

---

## 8. 参考链接

- [sing-box 官方文档](https://sing-box.sagernet.org)
- [OpenList 项目](https://github.com/OpenListTeam/OpenList)
- [OpenList 文档](https://doc.oplist.org)
