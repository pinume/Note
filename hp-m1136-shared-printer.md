# 🖨️ 共享打印机故障排查笔记（HP M1136）

> 环境：北国电器益庄店 · 共享打印机 HP LaserJet Professional M1136 MFP<br>
> 主机 IP：`192.168.165.98`

## 一、先快速判断问题在哪一层

| 现象 | 大概原因 |
| --- | --- |
| ping 不通主机 IP | 主机关机 / 睡眠 / IP 变了 |
| ping 通，但打印报 `0x00000709` | Windows 安全更新导致 RPC 兼容性问题 |
| ping 通，记事本打印报“RPC 服务器不可用” | 本机 RPC 或 Print Spooler 服务异常 |
| 其他电脑能打印，只有这台不行 | 问题在这台客户端，不是主机 |
| 所有电脑都不能打印 | 问题在主机或打印机本身 |
| WPS 报“无法启动打印作业” | 先用记事本测试，能打印就是 WPS 的问题 |

## 二、按顺序排查

### 第 1 步：确认网络和主机

- `Win+R` → `cmd` → `ping 192.168.165.98`
- 不通 → 前往主机检查是否开机、睡眠或 IP 发生变化

### 第 2 步：判断是本机还是主机的问题

- 使用 **记事本** 打印一段文字测试
- 能打印 → WPS 问题（重启 WPS，或导出 PDF 后再打印）
- 不能打印 → 继续下一步

### 第 3 步：重启 Print Spooler

- `Win+R` → `services.msc`
- 找到 **Print Spooler** → 右键“重新启动”
- 将启动类型改为“自动”

### 第 4 步：清理卡住的打印任务

1. 停止 Print Spooler 服务。
2. `Win+R` → `C:\Windows\System32\spool\PRINTERS`。
3. 删除文件夹中的所有文件。
4. 重新启动 Print Spooler。

### 第 5 步：检查 RPC 相关服务

出现“RPC 服务器不可用”时，确认下列服务都在运行，且启动类型为自动：

- Remote Procedure Call (RPC)
- RPC Endpoint Mapper
- DCOM Server Process Launcher
- Print Spooler

### 第 6 步：出现 `0x00000709` 时修改注册表

路径：

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Print
```

- 新建 **DWORD（32 位）值**：`RpcAuthnLevelPrivacyEnabled`
- 数值：`0`
- 重启电脑后生效
- 建议客户端和主机两边都添加

### 第 7 步：删除并重新连接共享打印机

1. 设置 → 蓝牙和其他设备 → 打印机和扫描仪 → 删除 M1136。
2. `Win+R` → 输入 `\\192.168.165.98` → 回车。
3. 双击共享的 M1136 重新安装。

### 第 8 步：兜底重启

重启本机。RPC / Spooler 类问题通常可以通过重启解决。

## 三、常用命令速查

```text
hostname                              查看本机名
ping 192.168.165.98                   测试主机连通
\\192.168.165.98                      访问主机共享
services.msc                          打开服务管理
regedit                               打开注册表
C:\Windows\System32\spool\PRINTERS    打印队列文件夹
```

## 四、长期建议

- 主机和客户端都添加 `RpcAuthnLevelPrivacyEnabled = 0` 注册表项，避免 Windows 更新后复发。
- 主机不要随意关机或睡眠，最好由 IT 固定 IP。
- 如果共享打印长期不稳定，可考虑将打印机从主机解绑，直接连接路由器并使用 IP 打印。
