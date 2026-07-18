# 📁 Yazi

> ⚡ **Yazi** 是一款基于异步 I/O、使用 Rust 编写的终端文件管理器。本页按 **Mac mini / Windows / Linux VPS** 三类环境整理安装、配置与常用操作。

## 页面内容

- **安装**：依赖、macOS、Windows、Ubuntu / Debian
- **使用**：启动退出、Shell Wrapper
- **快捷键**：导航、选择、文件操作、搜索、多标签与 Shell 集成
- **参考链接**：官网、文档、GitHub 与默认 keymap

---

## 一、安装

### 1. 前置依赖

| 类型 | 依赖 | 用途 |
| --- | --- | --- |
| 必装 | `file` | MIME 类型检测 |
| 可选增强 | `ffmpeg`、`7zip`、`jq`、`poppler`、`fd`、`ripgrep`、`fzf`、`zoxide`、`resvg`、`imagemagick` | 预览、搜索、跳转及格式处理 |
| 字体 | [Nerd Fonts](https://www.nerdfonts.com/) | 图标和符号显示 |

### 2. macOS（Homebrew）

**适用环境：Mac mini**

```bash
brew update
brew install yazi ffmpeg-full sevenzip jq poppler fd ripgrep fzf zoxide \
    resvg imagemagick-full font-symbols-only-nerd-font
brew link ffmpeg-full imagemagick-full -f --overwrite
```

### 3. Windows（Scoop）

已经使用 Scoop 时，可以直接执行：

```powershell
scoop install yazi
scoop install ffmpeg 7zip jq poppler fd ripgrep fzf zoxide resvg imagemagick
```

> ⚠️ **关键配置**
>
> Windows 上的 Yazi 需要 Git for Windows 自带的 `file.exe`。必须设置环境变量 `YAZI_FILE_ONE`，并将其指向对应路径。

| Git 安装方式 | `file.exe` 路径 |
| --- | --- |
| 官方安装包 | `C:\Program Files\Git\usr\bin\file.exe` |
| Scoop 安装 Git | `C:\Users\<用户名>\scoop\apps\git\current\usr\bin\file.exe` |

设置完成后，**重启终端**。

> ⚠️ 官方明确不建议通过 Scoop 或 Chocolatey 单独安装 `file`，因为相关版本可能无法正确处理 Unicode 文件名，例如 `oliver-sjöström.jpg`。

### 4. Ubuntu / Debian VPS

官方仓库通常不提供 Yazi，推荐以下两种安装方式。

#### 方式一：通过 Cargo 编译

**特点：获取最新版。**

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
cargo install --force yazi-build
```

该命令会同时安装：

- `yazi-fm`
- `yazi-cli`

#### 方式二：安装官方二进制

**特点：步骤更少。**

前往 [GitHub Releases](https://github.com/sxyazi/yazi/releases)，下载对应架构的 **musl** 版本压缩包，然后将以下两个文件放入 `/usr/local/bin/`：

- `yazi`
- `ya`

---

## 二、使用

### 1. 启动与退出

```bash
yazi
```

| 操作 | 作用 |
| --- | --- |
| `q` | 退出 Yazi |
| `F1` 或 `~` | 打开帮助菜单 |

### 2. Shell Wrapper

> 💡 **强烈推荐配置**
>
> 直接通过 `yazi` 启动时，退出后不会自动切换到浏览过的目录。配置 Shell Wrapper 后，可通过 `y` 启动，并在退出时同步切换目录。

将以下内容加入 `~/.zshrc` 或 `~/.bashrc`：

```bash
function y() {
    local tmp="$(mktemp -t "yazi-cwd.XXXXXX")" cwd
    command yazi "$@" --cwd-file="$tmp"
    IFS= read -r -d '' cwd < "$tmp"
    [ "$cwd" != "$PWD" ] && [ -d "$cwd" ] && builtin cd -- "$cwd"
    rm -f -- "$tmp"
}
```

配置后的行为：

| 操作 | 结果 |
| --- | --- |
| 使用 `y` 启动 | 启用目录同步功能 |
| 按 `q` 退出 | 切换到 Yazi 中最后浏览的目录 |
| 按 `Q` 退出 | 保留终端原目录 |

---

## 三、核心快捷键

> Yazi 默认采用 **Vim 风格**按键。以下内容按使用场景分类，可展开查看。

<details>
<summary>🧭 导航</summary>

| 按键 | 作用 |
| --- | --- |
| `h` `j` `k` `l` | 左 / 下 / 上 / 右 |
| `gg` | 跳到顶部 |
| `G` | 跳到底部 |
| `z` | 使用 fzf 跳转目录 |
| `Z` | 使用 zoxide 跳转目录 |

</details>

<details>
<summary>☑️ 选择</summary>

| 按键 | 作用 |
| --- | --- |
| `Space` | 选中或取消选中当前项 |
| `v` | 进入可视选择模式 |
| `Ctrl+a` | 全选 |
| `Esc` | 取消选择 |

</details>

<details>
<summary>📄 文件操作</summary>

| 按键 | 作用 |
| --- | --- |
| `o` / `Enter` | 打开 |
| `y` | 复制 |
| `x` | 剪切 |
| `p` | 粘贴 |
| `P` | 粘贴并覆盖 |
| `d` | 移到回收站 |
| `D` | 永久删除 |
| `a` | 新建文件；末尾加 `/` 表示新建目录 |
| `r` | 重命名 |
| `.` | 切换显示隐藏文件 |

</details>

<details>
<summary>🔎 搜索与过滤</summary>

| 按键 | 作用 |
| --- | --- |
| `f` | 过滤当前目录 |
| `/` | 查找 |
| `s` | 使用 `fd` 按文件名搜索 |
| `S` | 使用 `ripgrep` 按内容搜索 |

</details>

<details>
<summary>📋 复制路径</summary>

| 按键 | 作用 |
| --- | --- |
| `cc` | 复制文件完整路径 |
| `cd` | 复制目录路径 |
| `cf` | 复制文件名 |
| `cn` | 复制不含扩展名的文件名 |

</details>

<details>
<summary>🗂️ 多标签</summary>

| 按键 | 作用 |
| --- | --- |
| `tt` | 在当前目录新建标签 |
| `1` ～ `9` | 切换到第 N 个标签 |
| `[` / `]` | 切换到上一个 / 下一个标签 |
| `Ctrl+c` | 关闭当前标签 |

</details>

<details>
<summary>⌨️ Shell 集成</summary>

| 按键 | 作用 |
| --- | --- |
| `;` | 执行 Shell 命令，不阻塞 |
| `:` | 执行 Shell 命令，并阻塞至完成 |

</details>

---

## 四、参考链接

- **官网**：[Yazi](https://yazi-rs.github.io)
- **安装文档**：[Installation](https://yazi-rs.github.io/docs/installation)
- **快速开始**：[Quick Start](https://yazi-rs.github.io/docs/quick-start)
- **GitHub**：[sxyazi/yazi](https://github.com/sxyazi/yazi)
- **完整快捷键配置**：[keymap-default.toml](https://github.com/sxyazi/yazi/blob/shipped/yazi-config/preset/keymap-default.toml)
