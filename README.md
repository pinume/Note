# Note · 技术笔记

个人技术笔记仓库，当前收录 **11 份笔记**，主要记录 Linux/VPS 服务部署、代理网络配置、终端工具使用和常见故障排查。

**快速导航：** [代理与网络](#代理与网络) · [开发与终端工具](#开发与终端工具) · [资源收藏](#资源收藏) · [故障排查](#故障排查)

## 文档导航

### 代理与网络

| 文档 | 内容 |
| --- | --- |
| [Realm 完整配置教程](realm.md) | 使用 Realm 搭建 VPS 流量中转，包含多落地、UDP 与故障排查 |
| [mihomo Linux ARM64 部署教程](mihomo-linux-arm64.md) | 在 ARM64 Linux 上安装 mihomo 并配置 systemd 服务 |
| [Shadow-TLS + Snell + Surge 部署教程](shadow-tls-snell-surge.md) | 部署 Shadow-TLS 与 Snell，并配置 Surge 客户端 |
| [Shadowsocks-rust ARM64 服务端部署教程](shadowsocks-rust-arm64.md) | 安装 Shadowsocks-rust、配置服务端与 systemd |
| [sing-box AnyTLS 部署教程](sing-box-anytls.md) | 配置 AnyTLS 服务端、证书、客户端和常见问题 |
| [sing-box Trojan + WebSocket + OpenList](sing-box-trojan-websocket-openlist.md) | 使用 Nginx 分流或 sing-box fallback 搭建伪装站 |

### 开发与终端工具

| 文档 | 内容 |
| --- | --- |
| [uv 安装与使用](uv-installation-usage.md) | Python 版本、依赖、工具、虚拟环境和项目管理 |
| [Yazi](yazi.md) | Yazi 在 macOS、Windows、Linux 上的安装、Shell Wrapper 和快捷键 |
| [Microsoft Edit 安装与使用](microsoft-edit.md) | Microsoft Edit 的跨平台安装、快捷键和使用场景 |

### 资源收藏

| 文档 | 内容 |
| --- | --- |
| [GitHub 收藏整理](github.md) | 分类整理 142 个 Star、48 个 Following 与 14 个推荐项目 |

### 故障排查

| 文档 | 内容 |
| --- | --- |
| [HP M1136 共享打印机故障排查](hp-m1136-shared-printer.md) | Windows 共享打印、RPC、Print Spooler 和错误码排查 |

## 使用说明

- 文档中的域名、密码和路径示例需要按实际环境替换。
- 执行安装或部署命令前，请确认系统架构、软件版本和防火墙规则。
- 涉及版本号的下载地址可能随软件更新而变化，建议同时查阅项目官方文档和 Release 页面。

## 维护约定

- 新增笔记时，将其加入本页最接近的分类。
- 文件名使用简洁、可读的英文小写名称，单词之间使用连字符。
- 命令、路径、配置字段和快捷键使用行内代码格式。
