# 🧩 Shadowsocks-rust ARM64 服务端部署教程

## 文档说明

本文档用于记录在 ARM64 架构 Linux 服务器上部署 **shadowsocks-rust** 服务端的流程，并通过 **systemd** 实现开机自启。

本文示例版本：

```text
shadowsocks-rust v1.24.0
架构：aarch64 / ARM64
```

适用场景：

- ARM64 / AArch64 架构 VPS
- 使用官方二进制包部署 shadowsocks-rust
- 使用 `ssserver` 作为服务端程序
- 使用 `ssservice` 生成安全密码
- 使用 systemd 管理服务

---

## 一、下载 ARM64 安装包

```bash
cd /tmp
wget https://github.com/shadowsocks/shadowsocks-rust/releases/download/v1.24.0/shadowsocks-v1.24.0.aarch64-unknown-linux-gnu.tar.xz
ls -lh shadowsocks-v1.24.0.aarch64-unknown-linux-gnu.tar.xz
```

---

## 二、解压安装包

```bash
tar -xf shadowsocks-v1.24.0.aarch64-unknown-linux-gnu.tar.xz
```

解压后的常用程序包括：

| 文件 | 说明 |
| --- | --- |
| `ssserver` | Shadowsocks 服务端程序 |
| `ssservice` | 辅助工具，常用于生成密钥 |

```bash
ls -lh
```

---

## 三、安装程序到系统路径

推荐使用 `install` 命令安装到 `/usr/local/bin`：

```bash
sudo install -m 755 ssserver  /usr/local/bin/ssserver
sudo install -m 755 ssservice /usr/local/bin/ssservice
```

- `install -m 755` 会复制文件并设置可执行权限。
- 安装后可以直接运行 `ssserver` 和 `ssservice`。

检查安装结果与版本：

```bash
which ssserver
which ssservice
ssserver --version
```

### 备选写法：`mv` + `chmod`

```bash
sudo mv ssserver  /usr/local/bin/ssserver
sudo mv ssservice /usr/local/bin/ssservice
sudo chmod 755 /usr/local/bin/ssserver /usr/local/bin/ssservice
```

`mv` 只负责移动文件，不会自动设置权限，因此需要额外执行 `chmod 755`。

---

## 四、生成服务端密码

使用 `ssservice` 生成指定加密方式的安全密码：

```bash
ssservice genkey -m "chacha20-ietf-poly1305"
```

保存输出结果，后续写入配置文件：

```json
"password": "YOUR_PASSWORD"
```

> 建议不要手写弱密码，也不要复用其他服务的密码。

---

## 五、创建配置文件

```bash
sudo mkdir -p /etc/shadowsocks-rust
sudo vim /etc/shadowsocks-rust/config.json
```

写入以下内容，并将 `YOUR_PASSWORD` 替换为上一步生成的密码：

```json
{
  "server": "::",
  "server_port": 8388,
  "password": "YOUR_PASSWORD",
  "method": "chacha20-ietf-poly1305",
  "timeout": 300,
  "mode": "tcp_and_udp"
}
```

### 配置说明

| 字段 | 说明 |
| --- | --- |
| `server` | 监听地址，`::` 表示监听 IPv6，通常同时兼容 IPv4 |
| `server_port` | 服务端监听端口，示例为 `8388` |
| `password` | Shadowsocks 连接密码 |
| `method` | 加密方式，示例为 `chacha20-ietf-poly1305` |
| `timeout` | 超时时间，单位为秒 |
| `mode` | `tcp_and_udp` 表示同时支持 TCP 和 UDP |

在 Vim 中输入 `:wq` 保存并退出。

---

## 六、手动启动测试

先手动启动一次，确认配置没有明显问题：

```bash
sudo ssserver -c /etc/shadowsocks-rust/config.json
```

如果没有报错，说明配置基本正确。测试结束后按 `Ctrl+C` 终止进程。

---

## 七、配置 systemd 开机自启

创建 `/etc/systemd/system/ssserver.service`：

```ini
[Unit]
Description=shadowsocks-rust Server Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ssserver -c /etc/shadowsocks-rust/config.json
Restart=always
RestartSec=3
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

### systemd 参数说明

| 配置项 | 说明 |
| --- | --- |
| `After=network-online.target` | 网络就绪后再启动服务 |
| `Wants=network-online.target` | 请求 systemd 等待网络在线 |
| `Type=simple` | 普通前台进程模式 |
| `ExecStart` | 指定 ssserver 启动命令和配置文件 |
| `Restart=always` | 服务异常退出后自动重启 |
| `RestartSec=3` | 退出后 3 秒再重启 |
| `LimitNOFILE=1048576` | 提高文件描述符限制 |

重新加载 systemd 并启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable ssserver
sudo systemctl start ssserver
sudo systemctl status ssserver
```

也可以合并执行：

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now ssserver
```

---

## 八、查看运行状态

```bash
# 查看服务状态
systemctl status ssserver --no-pager

# 查看实时日志
journalctl -u ssserver -f

# 查看端口监听
sudo ss -lntup | grep 8388
```

如果服务状态为 `active (running)`，且端口 `8388` 正在监听，通常说明服务已正常启动。

---

## 九、防火墙与安全组

如果使用 UFW，需要放行服务端口：

```bash
sudo ufw allow 8388/tcp
sudo ufw allow 8388/udp
sudo ufw reload
```

云服务器还需在云厂商安全组中放行：

```text
TCP 8388
UDP 8388
```

---

## 十、客户端连接参数

| 参数 | 值 |
| --- | --- |
| 服务器地址 | VPS 公网 IP 或域名 |
| 端口 | `8388` |
| 密码 | `/etc/shadowsocks-rust/config.json` 中的 `password` |
| 加密方式 | `chacha20-ietf-poly1305` |
| 传输协议 | TCP / UDP |

---

## 十一、常用管理命令

| 操作 | 命令 |
| --- | --- |
| 启动服务 | `systemctl start ssserver` |
| 停止服务 | `systemctl stop ssserver` |
| 重启服务 | `systemctl restart ssserver` |
| 查看状态 | `systemctl status ssserver --no-pager` |
| 查看日志 | `journalctl -u ssserver -f` |
| 查看端口 | `sudo ss -lntup \| grep 8388` |

---

## 十二、推荐目录结构

```text
/usr/local/bin/ssserver                    # Shadowsocks 服务端程序
/usr/local/bin/ssservice                   # 辅助工具，用于生成密钥
/etc/shadowsocks-rust/config.json          # 服务端配置文件
/etc/systemd/system/ssserver.service       # systemd 服务文件
```

---

## 十三、部署检查清单

- [ ] 已下载 ARM64 安装包
- [ ] 已解压安装包
- [ ] 已安装 `ssserver` 到 `/usr/local/bin/ssserver`
- [ ] 已安装 `ssservice` 到 `/usr/local/bin/ssservice`
- [ ] 已使用 `ssservice genkey` 生成安全密码
- [ ] 已创建 `/etc/shadowsocks-rust/config.json`
- [ ] 已替换配置文件中的 `YOUR_PASSWORD`
- [ ] 已手动执行 `ssserver -c` 测试成功
- [ ] 已创建 `/etc/systemd/system/ssserver.service`
- [ ] 已执行 `systemctl daemon-reload`
- [ ] 已设置开机自启
- [ ] 服务状态为 `active (running)`
- [ ] 防火墙和安全组已放行 `8388/tcp` 与 `8388/udp`

---

## 十四、附录 A：一键部署版（推荐）

下面的脚本可以直接复制执行，唯一需要提前替换的是 `YOUR_PASSWORD`：

```bash
cd /tmp
wget https://github.com/shadowsocks/shadowsocks-rust/releases/download/v1.24.0/shadowsocks-v1.24.0.aarch64-unknown-linux-gnu.tar.xz
tar -xf shadowsocks-v1.24.0.aarch64-unknown-linux-gnu.tar.xz

sudo install -m 755 ssserver  /usr/local/bin/ssserver
sudo install -m 755 ssservice /usr/local/bin/ssservice

sudo mkdir -p /etc/shadowsocks-rust
sudo tee /etc/shadowsocks-rust/config.json > /dev/null <<'EOF'
{
  "server": "::",
  "server_port": 8388,
  "password": "YOUR_PASSWORD",
  "method": "chacha20-ietf-poly1305",
  "timeout": 300,
  "mode": "tcp_and_udp"
}
EOF

sudo tee /etc/systemd/system/ssserver.service > /dev/null <<'EOF'
[Unit]
Description=shadowsocks-rust Server Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ssserver -c /etc/shadowsocks-rust/config.json
Restart=always
RestartSec=3
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now ssserver
systemctl status ssserver --no-pager
```

---

## 十五、附录 B：使用 `mv` 的一键备选版

```bash
cd /tmp
wget https://github.com/shadowsocks/shadowsocks-rust/releases/download/v1.24.0/shadowsocks-v1.24.0.aarch64-unknown-linux-gnu.tar.xz
tar -xf shadowsocks-v1.24.0.aarch64-unknown-linux-gnu.tar.xz

sudo mv ssserver  /usr/local/bin/ssserver
sudo mv ssservice /usr/local/bin/ssservice
sudo chmod 755 /usr/local/bin/ssserver /usr/local/bin/ssservice

sudo mkdir -p /etc/shadowsocks-rust
sudo tee /etc/shadowsocks-rust/config.json > /dev/null <<'EOF'
{
  "server": "::",
  "server_port": 8388,
  "password": "YOUR_PASSWORD",
  "method": "chacha20-ietf-poly1305",
  "timeout": 300,
  "mode": "tcp_and_udp"
}
EOF

sudo tee /etc/systemd/system/ssserver.service > /dev/null <<'EOF'
[Unit]
Description=shadowsocks-rust Server Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ssserver -c /etc/shadowsocks-rust/config.json
Restart=always
RestartSec=3
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now ssserver
systemctl status ssserver --no-pager
```

---

## 十六、注意事项

1. 本教程使用 ARM64 / AArch64 安装包。
2. AMD64 / x86_64 服务器需要下载对应架构的安装包。
3. `YOUR_PASSWORD` 必须替换为实际生成的强密码。
4. 服务无法启动时，优先执行 `journalctl -u ssserver -f` 查看日志。
5. 客户端无法连接时，依次检查：
   - VPS 安全组是否放行 `8388/tcp` 与 `8388/udp`。
   - 系统防火墙是否放行 `8388`。
   - 客户端加密方式是否与服务端一致。
   - 客户端密码是否与服务端一致。
   - 服务端口是否正在监听。
