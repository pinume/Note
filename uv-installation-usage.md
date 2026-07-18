# UV: Installation & Usage

> 🦀 **uv** 是 Rust 编写的高性能 Python 包管理器，可一并替代 `pip` / `pipx` / `pyenv` / `virtualenv`。

<details>
<summary>📑 目录</summary>

- [安装 uv](#安装-uv)
- [Python 版本管理](#python-版本管理)
- [包管理（pip 替代）](#包管理pip-替代)
- [工具安装（pipx 替代）](#工具安装pipx-替代)
- [虚拟环境](#虚拟环境)
- [项目管理](#项目管理)
- [uv 自身维护](#uv-自身维护)
- [常用组合示例](#常用组合示例)

</details>

---

## 安装 uv

### macOS

<details>
<summary>✅ 方式一：官方脚本（推荐）</summary>

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

安装完成后重启终端，或手动加载环境变量：

```bash
source ~/.zshrc   # zsh（macOS 默认）
source ~/.bashrc  # bash
```

</details>

<details>
<summary>方式二：Homebrew</summary>

```bash
brew install uv
```

</details>

### Windows

<details>
<summary>✅ 方式一：官方脚本（PowerShell，推荐）</summary>

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

</details>

<details>
<summary>方式二：Scoop</summary>

```powershell
scoop install uv
```

</details>

<details>
<summary>方式三：winget</summary>

```powershell
winget install --id=astral-sh.uv -e
```

</details>

> 💡 安装后重启终端，使 `PATH` 生效。

### Linux（Debian / Ubuntu）

<details>
<summary>✅ 方式一：官方脚本（推荐）</summary>

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
```

</details>

<details>
<summary>方式二：pip 安装</summary>

```bash
pip install uv
```

</details>

### 验证安装

```bash
uv --version
```

---

## Python 版本管理

| 命令 | 作用 |
| --- | --- |
| `uv python list` | 列出所有可用版本（含可下载） |
| `uv python list --only-installed` | 只显示已安装版本 |
| `uv python install 3.13` | 安装指定主版本 |
| `uv python install 3.13.12` | 安装精确版本 |
| `uv python uninstall 3.13` | 卸载指定版本 |
| `uv python find` | 查找当前使用的 Python 路径 |

```bash
# 安装并固定到精确小版本
uv python install 3.13.12

# 列出已安装版本
uv python list --only-installed
```

---

## 包管理（pip 替代）

| 命令 | 作用 |
| --- | --- |
| `uv pip install <pkg>` | 安装包 |
| `uv pip install <pkg>==X.Y.Z` | 指定版本 |
| `uv pip install -r requirements.txt` | 按清单批量安装 |
| `uv pip install --upgrade <pkg>` | 升级包 |
| `uv pip uninstall <pkg>` | 卸载包 |
| `uv pip list` | 已安装列表 |
| `uv pip show <pkg>` | 查看包详情 |
| `uv pip freeze > requirements.txt` | 导出当前依赖 |

```bash
# 一次安装多个，固定版本
uv pip install "requests==2.31.0" "httpx>=0.27"

# 从清单还原环境
uv pip install -r requirements.txt
```

---

## 工具安装（pipx 替代）

| 命令 | 作用 |
| --- | --- |
| `uv tool install <tool>` | 全局安装命令行工具 |
| `uv tool list` | 列出已安装工具 |
| `uv tool upgrade <tool>` | 更新指定工具 |
| `uv tool upgrade --all` | 更新全部工具 |
| `uv tool uninstall <tool>` | 卸载工具 |
| `uvx <tool> ...` | 临时运行（不安装） |

```bash
# 常驻安装
uv tool install black
uv tool install ruff

# 临时运行，不污染全局
uvx ruff check .
uvx black .
```

---

## 虚拟环境

```bash
# 创建虚拟环境（默认目录 .venv）
uv venv

# 指定 Python 版本
uv venv --python 3.13

# 自定义目录名
uv venv myenv
```

**激活：**

```bash
source .venv/bin/activate        # macOS / Linux
.venv\Scripts\activate           # Windows
```

---

## 项目管理

| 命令 | 作用 |
| --- | --- |
| `uv init <name>` | 新建项目 |
| `uv add <pkg>` | 添加依赖 |
| `uv remove <pkg>` | 移除依赖 |
| `uv sync` | 同步依赖到虚拟环境 |
| `uv lock` | 锁定依赖版本 |
| `uv run <cmd>` | 在项目环境中运行命令 |

```bash
# 新建并进入项目
uv init myproject && cd myproject

# 添加依赖（支持版本约束）
uv add "requests>=2.31"

# 在项目环境里跑脚本，无需手动激活
uv run script.py
uv run python -c "import requests; print(requests.__version__)"
```

---

## uv 自身维护

```bash
# 查看当前版本
uv --version

# 更新 uv（独立安装）
uv self update

# 更新 uv（Homebrew 安装）
brew upgrade uv

# 缓存管理
uv cache dir       # 查看缓存目录
uv cache clean     # 清理下载缓存
```

---

## 常用组合示例

<details>
<summary>🚀 快速搭建项目环境</summary>

```bash
uv venv --python 3.13 \
  && source .venv/bin/activate \
  && uv pip install -r requirements.txt
```

</details>

<details>
<summary>🔄 检查并更新所有全局工具</summary>

```bash
uv tool upgrade --all
```

</details>

<details>
<summary>🐍 用指定 Python 版本运行脚本（无需激活环境）</summary>

```bash
uv run --python 3.13 script.py
```

</details>
