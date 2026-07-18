# ⚙️ mihomo Linux ARM64 部署教程

## 文档说明

本文档用于记录在 Linux 服务器上部署 **mihomo** 的流程。

本文示例使用版本：

```text
mihomo v1.19.0
架构：linux-arm64
```

适用场景：

- ARM64 / AArch64 架构服务器
- 使用 systemd 管理 mihomo 服务
- 配置目录固定为 `/etc/mihomo`
- 可执行文件路径固定为 `/usr/local/bin/mihomo`

---

## 一、下载二进制可执行文件

进入临时目录或当前工作目录后，下载 mihomo ARM64 版本：

```bash
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.19.0/mihomo-linux-arm64-v1.19.0.gz
```

下载完成后，可以查看文件是否存在：

```bash
ls -lh mihomo-linux-arm64-v1.19.0.gz
```

---

## 二、解压并移动到系统目录

将下载的压缩包解压，重命名为 `mihomo`，并移动到 `/usr/local/bin/`：

```bash
gzip -kd mihomo-linux-arm64-v1.19.0.gz
mv mihomo-linux-arm64-v1.19.0 /usr/local/bin/mihomo
```

赋予执行权限：

```bash
chmod +x /usr/local/bin/mihomo
```

检查是否可以正常运行：

```bash
/usr/local/bin/mihomo -v
```

---

## 三、创建配置目录与配置文件

创建 mihomo 配置目录：

```bash
mkdir -p /etc/mihomo
```

创建或编辑配置文件：

```bash
vim /etc/mihomo/config.yaml
```

配置文件路径为：

```text
/etc/mihomo/config.yaml
```

后续 systemd 服务会通过以下命令读取该目录下的配置文件：

```bash
/usr/local/bin/mihomo -d /etc/mihomo
```

---

## 四、创建 systemd 服务文件

创建服务文件：

```bash
vim /etc/systemd/system/mihomo.service
```

写入以下内容：

```ini
[Unit]
Description=mihomo Daemon, Another Clash Kernel.
After=network.target NetworkManager.service systemd-networkd.service iwd.service

[Service]
Type=simple
LimitNPROC=500
LimitNOFILE=1000000
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BIND_SERVICE CAP_SYS_TIME CAP_SYS_PTRACE CAP_DAC_READ_SEARCH CAP_DAC_OVERRIDE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BIND_SERVICE CAP_SYS_TIME CAP_SYS_PTRACE CAP_DAC_READ_SEARCH CAP_DAC_OVERRIDE
Restart=always
ExecStartPre=/usr/bin/sleep 1s
ExecStart=/usr/local/bin/mihomo -d /etc/mihomo
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

### 参数说明

| 配置项 | 说明 |
| --- | --- |
| `Description` | 服务描述 |
| `After=network.target ...` | 等待网络相关服务启动后再启动 mihomo |
| `Type=simple` | 普通前台进程模式 |
| `LimitNPROC=500` | 限制进程数量 |
| `LimitNOFILE=1000000` | 提高文件描述符限制 |
| `CapabilityBoundingSet` | 允许 mihomo 使用指定 Linux capabilities |
| `AmbientCapabilities` | 将指定 capabilities 传递给进程 |
| `Restart=always` | 服务异常退出后自动重启 |
| `ExecStartPre=/usr/bin/sleep 1s` | 启动前等待 1 秒 |
| `ExecStart=/usr/local/bin/mihomo -d /etc/mihomo` | 指定 mihomo 启动命令和配置目录 |
| `ExecReload=/bin/kill -HUP $MAINPID` | 使用 HUP 信号重新加载配置 |

---

## 五、重载并启动服务

```bash
systemctl daemon-reload
systemctl enable mihomo
systemctl start mihomo
```

也可以合并执行：

```bash
systemctl daemon-reload && systemctl enable --now mihomo
```

---

## 六、重新加载配置

修改 `/etc/mihomo/config.yaml` 后，可以执行：

```bash
systemctl reload mihomo
```

如果 reload 无效，可以重启服务：

```bash
systemctl restart mihomo
```

---

## 七、查看状态与日志

```bash
# 查看运行状态
systemctl status mihomo

# 查看日志末尾
journalctl -u mihomo -o cat -e

# 实时跟踪日志
journalctl -u mihomo -o cat -f
```

正常情况下应显示：

```text
active (running)
```

---

## 八、常用管理命令汇总

```bash
systemctl daemon-reload
systemctl enable mihomo
systemctl start mihomo
systemctl stop mihomo
systemctl restart mihomo
systemctl reload mihomo
systemctl status mihomo
journalctl -u mihomo -o cat -f
```

---

## 九、部署检查清单

- [ ] 已下载 `mihomo-linux-arm64-v1.19.0.gz`
- [ ] 已解压得到 `mihomo-linux-arm64-v1.19.0`
- [ ] 已移动到 `/usr/local/bin/mihomo`
- [ ] 已执行 `chmod +x /usr/local/bin/mihomo`
- [ ] 已创建 `/etc/mihomo/config.yaml`
- [ ] 已创建 `/etc/systemd/system/mihomo.service`
- [ ] 已执行 `systemctl daemon-reload`
- [ ] 已设置开机自启 `systemctl enable mihomo`
- [ ] 服务状态为 `active (running)`
- [ ] 日志中没有明显报错

---

## 十、推荐目录结构

```text
/usr/local/bin/mihomo                    # mihomo 可执行文件
/etc/mihomo/config.yaml                  # mihomo 主配置文件
/etc/systemd/system/mihomo.service       # systemd 服务文件
```

---

## 十一、注意事项

1. 本教程中的下载链接为 ARM64 架构版本，x86 / AMD64 服务器不能直接使用。
2. 如果服务器架构不同，需要下载对应架构的 mihomo 二进制文件。
3. 修改配置文件后，优先使用 `systemctl reload mihomo`，如未生效再使用 `systemctl restart mihomo`。
4. 如果服务启动失败，优先查看 `journalctl -u mihomo -o cat -e`。
5. 如果提示权限不足，确认 `/usr/local/bin/mihomo` 是否已经赋予执行权限。
