# 🛡️ Shadow-TLS + Snell + Surge 部署教程

## 文档说明

本文档用于记录 **Shadow-TLS + Snell + Surge** 的部署方式。

整体访问链路如下：

```text
Surge 客户端
    ↓
访问 VPS:8443
    ↓
Shadow-TLS 服务
    ↓
转发到 127.0.0.1:4143
    ↓
Snell 服务端
```

> 安全提示：本文档中的密码均使用占位符表示。实际部署时请自行生成并妥善保存，避免在公开聊天、截图或文档中泄露。

---

## 一、部署前提

部署前请确认服务器已经安装并配置好 Snell 服务端，并且 Snell 监听在本机端口：

```text
listen = 127.0.0.1:4143
```

也可以监听 `0.0.0.0:4143`，但更推荐使用 `127.0.0.1:4143`。这样 Snell 不直接暴露到公网，外部只访问 Shadow-TLS 的 8443 端口。

---

## 二、下载 Shadow-TLS

根据服务器架构选择对应版本。

### 1. x86 / AMD64 架构

```bash
wget https://github.com/ihciah/shadow-tls/releases/download/v0.2.23/shadow-tls-x86_64-unknown-linux-musl -O /usr/local/bin/shadow-tls
```

### 2. ARM64 / AArch64 架构

```bash
wget https://github.com/ihciah/shadow-tls/releases/download/v0.2.23/shadow-tls-aarch64-unknown-linux-musl -O /usr/local/bin/shadow-tls
```

---

## 三、赋予执行权限

```bash
chmod +x /usr/local/bin/shadow-tls
/usr/local/bin/shadow-tls --help
```

---

## 四、创建 systemd 服务文件

创建 `/etc/systemd/system/shadow-tls.service`：

```ini
[Unit]
Description=Shadow-TLS Server Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/shadow-tls --fastopen --v3 server --listen [::]:8443 --server 127.0.0.1:4143 --tls gateway.icloud.com --password <Shadow-TLS密码>
Restart=always
RestartSec=3
LimitNOFILE=1048576
Environment=MONOIO_FORCE_LEGACY_DRIVER=1

[Install]
WantedBy=multi-user.target
```

### 参数说明

| 参数 | 说明 |
| --- | --- |
| `--fastopen` | 启用 TCP Fast Open |
| `--v3 server` | 使用 Shadow-TLS v3 服务端模式 |
| `--listen [::]:8443` | 监听 8443 端口，支持 IPv4 / IPv6 |
| `--server 127.0.0.1:4143` | 转发到本机 Snell 服务端口 |
| `--tls gateway.icloud.com` | TLS 伪装域名 / SNI |
| `--password <Shadow-TLS密码>` | Shadow-TLS 连接密码 |
| `Restart=always` | 服务异常退出后自动重启 |
| `LimitNOFILE=1048576` | 提高文件描述符限制 |

---

## 五、重载并启动服务

```bash
systemctl daemon-reload
systemctl enable --now shadow-tls.service
systemctl status shadow-tls.service
```

正常情况下应显示 `active (running)`。

---

## 六、防火墙与安全组设置

服务器只需要对外放行 Shadow-TLS 监听端口 `TCP 8443`。

```bash
ufw allow 8443/tcp
ufw reload
```

不建议对公网开放 Snell 后端端口 `4143`，因为该端口只需要由本机 Shadow-TLS 访问。

---

## 七、Surge 节点配置

### Snell v4 写法

```text
Snell+TLS = snell, <VPS的IP或域名>, 8443, psk=<Snell密码>, version=4, shadow-tls-password=<Shadow-TLS密码>, shadow-tls-sni=gateway.icloud.com, shadow-tls-version=3
```

### Snell v5 写法

```text
Snell+TLS = snell, <VPS的IP或域名>, 8443, psk=<Snell密码>, version=5, shadow-tls-password=<Shadow-TLS密码>, shadow-tls-sni=gateway.icloud.com, shadow-tls-version=3
```

注意：

- `version=4` 或 `version=5` 必须与服务器端 Snell 版本一致。
- `shadow-tls-password` 必须与服务器端 `--password` 一致。
- `shadow-tls-sni` 必须与服务器端 `--tls` 一致。
- Surge 连接的是 Shadow-TLS 的端口 `8443`，不是 Snell 后端端口 `4143`。

---

## 八、生成密码建议

建议分别为 Snell 和 Shadow-TLS 生成不同密码：

```bash
openssl rand -base64 24
```

```text
Snell psk = <Snell密码>
Shadow-TLS password = <Shadow-TLS密码>
```

不要复用密码，也不要把真实密码写入公开文档。

---

## 九、常用排查命令

```bash
# 查看 Shadow-TLS 日志
journalctl -u shadow-tls.service -f

# 查看 Snell 日志
journalctl -u snell.service -f

# 查看端口监听
ss -lntp
```

应能看到类似结果：

```text
[::]:8443 或 0.0.0.0:8443    shadow-tls
127.0.0.1:4143               snell
```

---

## 十、检查清单

- [ ] Shadow-TLS 文件已下载到 `/usr/local/bin/shadow-tls`
- [ ] 已执行 `chmod +x /usr/local/bin/shadow-tls`
- [ ] Snell 服务端已正常运行
- [ ] Snell 监听端口为 `127.0.0.1:4143`
- [ ] Shadow-TLS 服务已正常运行
- [ ] VPS 防火墙 / 云安全组已放行 `8443/tcp`
- [ ] Surge 节点中的 Snell 版本号与服务端一致
- [ ] Surge 中的 `shadow-tls-password` 与服务端一致
- [ ] Surge 中的 `shadow-tls-sni` 与服务端 `--tls` 一致

---

## 十一、推荐最终结构

```text
公网访问：VPS:8443
Shadow-TLS：监听 8443
Snell：监听 127.0.0.1:4143
Surge：使用 Snell + Shadow-TLS 参数连接 8443
```

这样可以避免直接暴露 Snell 服务端口，同时通过 Shadow-TLS 提供 TLS 伪装。
