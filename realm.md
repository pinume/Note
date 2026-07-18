# 🌐 Realm 完整配置教程

> 适用场景：在中转 VPS 上部署 Realm，把流量转发到落地机（Trojan / sing-box / Shadowsocks / Hysteria 等任意协议）。

---

## 一、架构理解

```text
你的设备（Surge / sing-box）
    │  连接：中转机 IP:5000
    ▼
中转 VPS  ←  本教程要部署的
    │  Realm 把字节流原样转发
    ▼
落地 VPS（Trojan / sing-box / 任意协议）
    │
    ▼
出网
```

**核心原则**：

- Realm 只在**中转机**上部署。
- **落地机不需要任何改动**。
- **客户端只改 server 和 port**，其他参数（密码、SNI、ws-path、Host）保持原样。

---

## 二、安装 Realm

### 2.1 确认 CPU 架构

```bash
uname -m
```

| 输出 | 下载文件名 |
| --- | --- |
| `x86_64` | `realm-x86_64-unknown-linux-gnu.tar.gz` |
| `aarch64` | `realm-aarch64-unknown-linux-gnu.tar.gz` |

### 2.2 下载并安装

```bash
cd /tmp

# 将 v2.9.3 改为最新版本号
wget https://github.com/zhboner/realm/releases/download/v2.9.3/realm-x86_64-unknown-linux-gnu.tar.gz
tar xzf realm-x86_64-unknown-linux-gnu.tar.gz
sudo mv realm /usr/local/bin/realm
sudo chmod +x /usr/local/bin/realm

# 验证
realm -v
```

---

## 三、防火墙放行

### 3.1 系统层防火墙

```bash
# Ubuntu/Debian（UFW）
sudo ufw allow 5000/tcp
sudo ufw allow 5000/udp

# 或 iptables
sudo iptables -I INPUT -p tcp --dport 5000 -j ACCEPT
sudo iptables -I INPUT -p udp --dport 5000 -j ACCEPT
sudo netfilter-persistent save  # Debian/Ubuntu 持久化
```

### 3.2 云厂商控制台防火墙

| 厂商 | 操作位置 |
| --- | --- |
| **Oracle Cloud** | Networking → VCN → Security Lists → Add Ingress Rule（**必做，否则一定连不上**） |
| **AWS** | EC2 → Security Groups → Inbound Rules |
| **GCP** | VPC → Firewall Rules |
| **DMIT / Misaka / 一般 KVM** | 通常无需操作 |

每个监听端口都要放行 **TCP + UDP**。

---

## 四、编写配置文件

```bash
sudo mkdir -p /etc/realm
sudo nano /etc/realm/config.toml
```

按需求选择下面一个模板。

### 模板 A：单落地（最常用）

```toml
[log]
level = "warn"
output = "/var/log/realm.log"

[network]
no_tcp = false
use_udp = true
tcp_timeout = 5
udp_timeout = 30
tcp_keepalive = 15

[[endpoints]]
listen = "0.0.0.0:5000"
remote = "your-oracle-domain.com:443"
```

### 模板 B：多落地

```toml
[log]
level = "warn"
output = "/var/log/realm.log"

[network]
use_udp = true
udp_timeout = 30

# Oracle Trojan
[[endpoints]]
listen = "0.0.0.0:5000"
remote = "oracle.example.com:443"

# DMIT 备用落地
[[endpoints]]
listen = "0.0.0.0:5001"
remote = "dmit.example.com:443"

# Misaka Shadowsocks
[[endpoints]]
listen = "0.0.0.0:5002"
remote = "misaka.example.com:8388"
```

### 模板 C：纯 UDP 落地

适用于 Hysteria2 / TUIC / WireGuard。

```toml
[network]
no_tcp = true        # 关闭 TCP 以节省资源
use_udp = true
udp_timeout = 120    # QUIC 长连接，适当调大

[[endpoints]]
listen = "0.0.0.0:5000"
remote = "你的Hysteria落地:443"
```

### 模板 D：负载均衡

```toml
[network]
use_udp = true

[[endpoints]]
listen = "0.0.0.0:5000"
remote = "落地A:443"
extra_remotes = ["落地B:443", "落地C:443"]
balance = "roundrobin: 4, 2, 1"   # 权重 4:2:1
```

可选算法：`roundrobin`（按权重轮询）、`iphash`（同一客户端 IP 总是使用同一落地）。

---

## 五、关键配置参数说明

| 参数 | 含义 | 推荐值 |
| --- | --- | --- |
| `listen` | 中转机监听地址 | `0.0.0.0:端口`（IPv4）或 `[::]:端口`（双栈） |
| `remote` | 落地机地址，可用 IP 或域名 | 优先使用域名，落地 IP 变更时无需修改配置 |
| `use_udp` | 是否转发 UDP | 协议涉及 UDP 时开启 |
| `no_tcp` | 关闭 TCP 转发 | 纯 UDP 协议（Hysteria）开启 |
| `tcp_timeout` | 连接落地的超时时间 | `5` 秒 |
| `udp_timeout` | UDP 关联超时时间 | TCP 类协议 `30`，QUIC 类协议 `120` |
| `tcp_keepalive` | TCP 保活间隔 | `15` |

---

## 六、启动测试

```bash
# 前台运行并查看输出
realm -c /etc/realm/config.toml
```

终端会停住，请不要关闭。打开新窗口或在本地客户端测试连接。测试通过后按 `Ctrl+C` 停止，再配置系统服务。

---

## 七、配置 systemd 服务

创建 `/etc/systemd/system/realm.service`：

```ini
[Unit]
Description=Realm relay service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/realm -c /etc/realm/config.toml
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

启用并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable realm
sudo systemctl start realm
sudo systemctl status realm
```

看到绿色 `active (running)` 即表示成功。

---

## 八、客户端配置

把原来直连落地的节点复制一份，**只改两个字段**：

| 字段 | 原值（直连落地） | 改为（走中转） |
| --- | --- | --- |
| `server` | `oracle.example.com` | `中转机IP或域名` |
| `port` | `443` | `5000` |

**绝对不要修改的字段**：

- 密码 / UUID
- SNI（伪装域名）
- WebSocket Host header
- ws-path
- TLS verify、加密方式
- Reality public-key、shortId

> **为什么不能改？** TLS 握手和协议握手是端到端加密的，Realm 看不到内容。落地机收到时仍按原 SNI / Host 处理，修改后会导致匹配失败。

---

## 九、运维常用命令

```bash
# 实时日志
sudo journalctl -u realm -f

# Realm 自己写入的日志
sudo tail -f /var/log/realm.log

# 修改配置后重启
sudo systemctl restart realm

# 停止 / 启动
sudo systemctl stop realm
sudo systemctl start realm

# 查看 Realm 是否在监听端口
sudo ss -tlnp | grep realm
sudo ss -ulnp | grep realm

# 查看服务状态
sudo systemctl status realm
```

---

## 十、常见故障排查

### 故障 1：客户端无法连接中转机

按顺序排查：

```bash
# 1. Realm 在运行吗？
sudo systemctl status realm

# 2. 端口在监听吗？
sudo ss -tlnp | grep 5000

# 3. 防火墙放行了吗？（包括云控制台）
sudo iptables -L INPUT -n | grep 5000

# 4. 在本地电脑测试端口
nc -zv 中转机IP 5000
# 或
telnet 中转机IP 5000
```

### 故障 2：TCP 正常但 UDP 不通

- 是否设置了 `use_udp = true`？
- 防火墙是否放行 UDP？
- 云控制台的 Security List 是否添加 UDP？

### 故障 3：TLS 握手失败或节点连接后不通

通常是客户端的 SNI / Host 修改错误，请恢复为落地节点的原值。

### 故障 4：中转后延迟反而变高

- 中转机到落地的线路本身较慢：更换中转机。
- 中转机 CPU 跑满：升级配置或更换性能更好的中转。
- 中间路径丢包：使用 `mtr 落地IP` 检查具体跳点。

### 故障 5：修改配置文件后未生效

```bash
sudo systemctl restart realm
```

---

## 十一、多协议落地速查表

| 落地协议 | 配置要点 |
| --- | --- |
| Trojan / Trojan+WS | 默认配置即可 |
| Shadowsocks（仅 TCP） | 默认配置即可 |
| Shadowsocks（TCP+UDP） | `use_udp = true` |
| VLESS / VMess + WS | 默认配置即可 |
| VLESS + Reality | 默认配置即可，客户端 SNI 不变 |
| **Hysteria2 / TUIC** | `no_tcp = true`、`use_udp = true`、`udp_timeout = 120` |
| WireGuard | `no_tcp = true`、`use_udp = true` |
| SSH 端口转发 | 默认配置即可 |

---

## 十二、最小化心智模型

记住三件事即可：

1. **Realm 是透明水管**：只看 IP 和端口，不看内容。
2. **落地和客户端的协议参数全部不动**：只修改 server 和 port。
3. **修改配置**：编辑 `/etc/realm/config.toml`，然后执行 `systemctl restart realm`。
