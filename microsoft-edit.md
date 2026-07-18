# 📝 Microsoft Edit 安装与使用

> **Microsoft Edit** — 微软用 Rust 重写的现代 MS-DOS Editor，定位“终端里的 VS Code 替代品”，操作逻辑接近 VS Code（鼠标可用，`Ctrl+S` / `Ctrl+F` / `Ctrl+Z`），但跑在终端里。适合改配置文件、远程编辑小文件。Windows 11 24H2 起内置。

## Windows 安装

### 方式一：检查是否自带

Windows 11 24H2 之后预装。在 PowerShell 输入：

```powershell
edit
```

直接进入编辑器就是已有，不需要安装。

### 方式二：winget

```powershell
winget install Microsoft.Edit
```

### 方式三：Scoop

```powershell
scoop install main/msedit
```

Scoop 仓库的包名叫 `msedit`（避免和系统 `edit` 命令冲突），但装完命令仍是 `edit`。

> 💡 安装后重开一个 PowerShell 窗口让 `PATH` 生效。

## Linux 安装

官方没有 apt/dnf 包，主要有三种安装方式。

### 方式一：下载预编译二进制（推荐）

```bash
cd /tmp

# x86_64 机器
wget https://github.com/microsoft/edit/releases/latest/download/edit-1.2.1-x86_64-linux-gnu.tar.zst

# 解压需要 zstd
sudo apt install zstd -y  # Ubuntu/Debian
sudo dnf install zstd -y  # Fedora/RHEL

tar --use-compress-program=unzstd -xf edit-1.2.1-x86_64-linux-gnu.tar.zst
sudo mv edit /usr/local/bin/msedit
sudo chmod +x /usr/local/bin/msedit
```

根据机器架构选择对应文件名（前往 [releases 页面](https://github.com/microsoft/edit/releases/latest) 查看）：

- Oracle Cloud ARM 实例 → `aarch64-linux-gnu`
- 普通 x86 VPS → `x86_64-linux-gnu`
- musl 静态版（无 glibc 依赖）→ 选择 `-musl` 后缀

**注意**：Linux 下推荐用 `msedit` 而非 `edit`，避免和系统其他命令冲突。想用短命令可以添加 alias：

```bash
echo "alias edit='msedit'" >> ~/.bashrc
source ~/.bashrc
```

### 方式二：源码编译

需要 Rust nightly 工具链：

```bash
# 安装 Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
rustup install nightly

# 克隆并编译
git clone https://github.com/microsoft/edit.git
cd edit
cargo build --config .cargo/release-nightly.toml --release

# 安装到系统
sudo cp target/release/edit /usr/local/bin/msedit
```

编译大约 2–3 分钟，VPS 小机器可能更慢。

### 方式三：发行版包管理器

- **Arch / Manjaro**：`yay -S microsoft-edit`
- **Debian / Ubuntu / RHEL**：暂无官方源包，使用方式一或方式二
- 其他发行版查看 [Repology](https://repology.org/project/microsoft-edit/versions)

## 使用方法

### 基本命令

| 命令 | 说明 |
| --- | --- |
| `edit` | 空白打开 |
| `edit file.txt` | 打开已有文件 |
| `edit /etc/nginx/nginx.conf` | 通过绝对路径打开 |
| `edit new.txt` | 文件不存在时自动创建 |

Linux 上把 `edit` 换成 `msedit`（或添加 alias 后仍使用 `edit`）。

### 快捷键（VS Code 风格）

| 快捷键 | 功能 |
| --- | --- |
| `Ctrl+S` | 保存 |
| `Ctrl+O` | 打开文件 |
| `Ctrl+N` | 新建文件 |
| `Ctrl+W` | 关闭当前文件 |
| `Ctrl+Q` | 退出 |
| `Ctrl+F` | 查找 |
| `Ctrl+R` | 替换 |
| `Ctrl+Z` | 撤销 |
| `Ctrl+Y` | 重做 |
| `Ctrl+G` | 跳转到指定行 |
| `Alt` | 激活顶部菜单栏（File / Edit / View / Help） |

鼠标点击、拖选、滚轮全部可用。

## 常见问题

- **搜索/替换需要 ICU 库**：Linux 上一般已安装 `libicu`，Ubuntu/Debian 可通过 `sudo apt install libicu-dev` 补装。未安装时仍能使用编辑器，但搜索功能可能受限。
- **终端字体**：使用支持 UTF-8 和等宽字符的字体效果最好。Windows Terminal 默认的 Cascadia Code 没问题；Linux 上推荐 JetBrains Mono、Source Code Pro。
- **SSH 远程复制粘贴**：依赖本地终端的复制粘贴机制（例如 Windows Terminal 使用 `Ctrl+Shift+C` 或鼠标右键），不是编辑器自身处理。
- **大文件**：Edit 面向中小文件，几十 MB 以上的大文件可能会卡。处理日志或 dump 文件仍建议使用 `less` + `vim`。

## 与其他终端编辑器对比

| 编辑器 | 学习成本 | 鼠标 | 菜单栏 | 适合改配置 | 适合写代码 |
| --- | --- | --- | --- | --- | --- |
| Edit | 几乎零 | ✅ | ✅ | ⭐⭐⭐⭐⭐ | ⭐ |
| Nano | 低 | ❌ | ❌ | ⭐⭐⭐⭐ | ⭐⭐ |
| Vim | 高 | 有限 | ❌ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Helix | 高 | ✅ | ❌ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

## 实战场景示例

### Windows：修改 sing-box 配置

```powershell
edit "$env:USERPROFILE\sing-box\config.json"
```

### Linux VPS：远程修改 Nginx 配置

```bash
ssh ubuntu@vps
sudo msedit /etc/nginx/sites-available/default
```

比 `scp` 下载 → 本地修改 → `scp` 上传一整套操作省事得多，也不像 `vim` 那样需要记忆快捷键。

### Oracle Cloud IPv6 VPS：修改 sing-box 配置

```bash
ssh -6 ubuntu@instance-20260207-1353
sudo msedit /etc/sing-box/config.json
# Ctrl+S 保存，Ctrl+Q 退出
sudo systemctl restart sing-box
```

## 参考链接

- [GitHub 仓库](https://github.com/microsoft/edit)
- [最新 Release](https://github.com/microsoft/edit/releases/latest)
- [打包状态总览](https://repology.org/project/microsoft-edit/versions)
