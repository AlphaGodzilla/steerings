# Python 执行规约

适用于代理在本机运行 Python 代码、安装 Python 依赖与编写一次性脚本的场景。本文只规定运行方式与依赖获取，不涉及具体业务域、项目部署路径与 Python 代码风格；版本与环境细节以当前机器上 `uv` 的实际输出为准（本机基线：`uv 0.9.14`，安装于 `~/.local/bin/uv`）。全文依次为强制要求、运行方式、隔离与环境、项目边界、网络与索引。

## 规则索引

按索引定位小节，只读命中的部分，不必通读全文。

| 触发场景 | 读哪一节 |
| --- | --- |
| 不确定能否调用 `python`、`pip`，或该用什么安装依赖 | 强制要求 |
| 选择 `--with`、内联元数据还是从标准输入执行，或指定 Python 版本 | 运行方式 |
| 排查脚本环境位置、缓存清理，或为什么能导入宿主的包 | 隔离与环境 |
| 在已有 `pyproject.toml` 的项目里跑脚本，或改用 `uv tool install` / `uvx` | 项目边界 |
| 依赖拉取失败、需要镜像或离线执行 | 网络与索引 |

## 强制要求

- 运行 Python 代码必须使用 `uv run`；安装 Python 依赖必须使用 `uv add`（项目内）、`uv tool install`（命令行工具）或脚本内联元数据。
- 禁止直接调用 `python`、`python3`、`pip`、`pip3`、`python -m pip`、`python -m venv`；禁止向宿主任何解释器的 site-packages 安装包。
- 第三方依赖必须显式声明（脚本内联元数据或 `--with`）；禁止依赖宿主 site-packages 中恰好存在的包。
- 需要落盘的临时脚本必须写入 `/tmp`，再用 `uv run` 执行。

## 运行方式

1. **依赖简单、只跑一次**：用 `--with` 在命令中逐项声明依赖，例如 `uv run --with requests --with beautifulsoup4 /tmp/fetch.py`。
2. **依赖较多或需锁定版本**：在脚本头部写 PEP 723 内联元数据，然后执行 `uv run /tmp/report.py`。uv 会解析元数据并自行准备解释器与依赖，无需 `pip install`。

   ```python
   # /// script
   # requires-python = ">=3.12"
   # dependencies = [
   #     "pandas==2.2.2",
   #     "openai>=1.0.0",
   # ]
   # ///

   import pandas as pd
   ```

3. **不落盘的脚本**：`uv run -` 从标准输入读取脚本，内联元数据同样生效，例如 `printf '<脚本内容>' | uv run -`。
4. **需要特定 Python 版本**：在元数据中写 `requires-python`，或临时指定 `uv run --python 3.12 script.py`；本机缺失的版本由 uv 自动下载，也可用 `uv python install 3.12` 预装。

## 隔离与环境

- 带内联元数据或 `--with` 的脚本运行在独立环境中：环境位于 `~/.cache/uv`（用 `uv cache dir` 确认当前值），实测 `uv run --with six /tmp/s.py` 时脚本内 `sys.prefix` 指向缓存中的临时环境而非宿主解释器。
- 这些环境按依赖集合缓存复用，不随每次执行重建；缓存目录同时容纳下载的包与脚本环境，清理使用 `uv cache clean`（清空全部）或 `uv cache clean <包名>`（按包清理）。两种清理都只作用于 uv 缓存，不涉及宿主解释器与其 site-packages。
- 不带内联元数据且未使用 `--with` 时，`uv run script.py` 只是用 uv 发现的解释器直接运行脚本，不安装任何包；此时第三方导入只能命中该解释器已有的包，等价于隐式依赖宿主环境。这正是「强制要求」中“第三方依赖必须显式声明”的原因。

## 项目边界

- 当前目录或父目录存在 `pyproject.toml`（uv 项目或工作区）时，`uv run` 使用并同步项目环境，可能创建或更新 `uv.lock`。为一次性脚本争取隔离时必须加 `--no-project`：`uv run --no-project --with requests /tmp/fetch.py` 使用由 `--with` 填充的临时环境。
- 项目已确立 Python 工具链（自带 `.venv`、poetry、pdm、Makefile 目标或 CI 脚本）时，按其既有方式执行；改用 uv 改造项目前必须说明理由并向用户确认。
- 命令行工具长期复用使用 `uv tool install`（装入 `~/.local/share/uv/tools`），一次性调用命令行工具使用 `uvx <工具>`；两者都不修改宿主全局环境。

## 网络与索引

- 解析新依赖需要访问包索引，默认 `https://pypi.org/simple`，可用 `--default-index` 或 `UV_DEFAULT_INDEX` 指定镜像；依赖已在缓存中且不允许联网时使用 `--offline` 或 `UV_OFFLINE=1`。
- 索引不可达（如报 `Failed to fetch ... tunnel error`）时，禁止退回宿主 `pip` 安装；先核对网络、代理与镜像配置，并把结论告知用户后再决定后续动作。
