# uv 使用笔记

`uv` 是 Astral 开发的 Python 包与项目管理工具，可以覆盖 `pip`、`venv`、`pipx`、依赖锁定以及 Python 版本管理等常见场景。

## 一、安装与检查

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# 查看版本
uv --version

# 查看帮助
uv --help
uv add --help

# 更新 uv
uv self update
```

## 二、创建项目

```bash
# 在当前目录初始化项目
uv init

# 创建新项目目录
uv init my-project

# 创建库项目
uv init --lib my-library

# 创建最小项目，只生成 pyproject.toml
uv init --bare
```

常见项目文件：

- `pyproject.toml`：项目配置和直接依赖
- `uv.lock`：锁定完整、精确的依赖版本，通常应提交到 Git
- `.venv/`：项目虚拟环境，通常不提交到 Git
- `.python-version`：项目默认使用的 Python 版本

## 三、管理 Python 版本

```bash
# 查看可安装和已安装的 Python
uv python list

# 安装 Python
uv python install 3.12

# 同时安装多个版本
uv python install 3.11 3.12

# 为当前项目指定 Python 版本
uv python pin 3.12

# 查找 Python 解释器
uv python find 3.12

# 卸载由 uv 管理的 Python
uv python uninstall 3.11
```

## 四、管理项目依赖

```bash
# 添加依赖
uv add requests

# 指定版本或版本范围
uv add "django>=5,<6"

# 添加开发依赖
uv add --dev pytest ruff

# 添加可选依赖
uv add --optional postgres psycopg

# 从 Git 仓库添加依赖
uv add git+https://github.com/user/repo.git

# 删除依赖
uv remove requests

# 删除开发依赖
uv remove --dev pytest
```

`uv add` 会更新 `pyproject.toml`、`uv.lock` 和项目环境。

## 五、锁定与同步环境

```bash
# 根据 pyproject.toml 更新锁文件
uv lock

# 检查锁文件是否需要更新，不修改文件
uv lock --check

# 按锁文件创建或同步虚拟环境
uv sync

# 严格同步，删除锁文件中不存在的包
uv sync --exact

# 同步时不安装开发依赖
uv sync --no-dev

# 同步指定的可选依赖组
uv sync --extra postgres

# 同步全部可选依赖
uv sync --all-extras
```

一般不需要手动创建或激活虚拟环境；`uv run` 会自动确保环境存在且已同步。

## 六、运行项目命令和脚本

```bash
# 运行 Python 文件
uv run python main.py

# .py 文件可省略 python
uv run main.py

# 运行项目环境中的命令
uv run pytest
uv run ruff check .

# 临时附加依赖后运行，不写入项目依赖
uv run --with requests python script.py

# 使用指定 Python 版本运行
uv run --python 3.12 main.py
```

如果脚本包含 PEP 723 内联依赖元数据，`uv run script.py` 会自动为它准备隔离环境。

## 七、`uv run` 与 `uvx`

```bash
# 使用当前项目环境运行
uv run pytest

# 在隔离的工具环境中运行
uvx ruff check .

# uvx 完全等价于
uv tool run ruff check .
```

选择原则：

- 工具属于当前项目、版本需要跟随项目锁定：使用 `uv run`
- 临时使用某个命令行工具、不想加入项目依赖：使用 `uvx`
- 希望长期安装并直接从 `PATH` 调用：使用 `uv tool install`

```bash
# 运行指定版本
uvx ruff@0.12.0 check .

# 命令名与包名不一致时指定来源包
uvx --from httpie http

# 为工具临时增加额外依赖
uvx --with mkdocs-material mkdocs serve
```

## 八、长期安装命令行工具

```bash
# 安装工具
uv tool install ruff

# 查看已安装工具
uv tool list

# 升级指定工具
uv tool upgrade ruff

# 升级全部工具
uv tool upgrade --all

# 卸载工具
uv tool uninstall ruff

# 把工具目录加入 PATH
uv tool update-shell
```

工具被安装在各自独立的环境中，不会成为当前项目可导入的 Python 依赖。

## 九、查看和导出依赖

```bash
# 查看项目依赖树
uv tree

# 检查过期包
uv tree --outdated

# 将锁文件导出为 requirements.txt
uv export --format requirements-txt --output-file requirements.txt

# 导出时不包含开发依赖
uv export --no-dev --format requirements-txt --output-file requirements.txt
```

## 十、pip 兼容模式

迁移旧项目或没有 `pyproject.toml` 时，可以使用 `uv pip`：

```bash
# 创建虚拟环境
uv venv

# 指定 Python 创建环境
uv venv --python 3.12

# 安装包
uv pip install requests

# 按 requirements.txt 安装
uv pip install -r requirements.txt

# 严格同步 requirements.txt
uv pip sync requirements.txt

# 查看已安装包
uv pip list

# 查看依赖树
uv pip tree

# 卸载包
uv pip uninstall requests

# 冻结当前环境
uv pip freeze
```

`uv pip install` 只操作环境，不会像 `uv add` 一样修改 `pyproject.toml` 或项目锁文件。新项目通常优先使用 `uv add` 和 `uv sync`。

## 十一、日常工作流

### 新建项目

```bash
uv init demo
cd demo
uv python pin 3.12
uv add requests
uv add --dev pytest ruff
uv run pytest
```

### 拉取已有项目

```bash
git clone <repo-url>
cd <repo>
uv sync
uv run pytest
```

### 运行单文件脚本

```bash
uv run script.py

# 脚本临时需要某个依赖
uv run --with requests script.py
```

### 临时运行工具

```bash
uvx ruff check .
uvx black .
uvx httpie https://example.com
```

## 十二、常用命令速查

| 目的 | 命令 |
| --- | --- |
| 初始化项目 | `uv init` |
| 添加依赖 | `uv add <package>` |
| 添加开发依赖 | `uv add --dev <package>` |
| 删除依赖 | `uv remove <package>` |
| 更新锁文件 | `uv lock` |
| 同步环境 | `uv sync` |
| 运行项目命令 | `uv run <command>` |
| 临时运行工具 | `uvx <tool>` |
| 安装全局工具 | `uv tool install <tool>` |
| 查看依赖树 | `uv tree` |
| 安装 Python | `uv python install <version>` |
| 固定项目 Python | `uv python pin <version>` |
| 创建虚拟环境 | `uv venv` |
| pip 兼容安装 | `uv pip install <package>` |

## 参考资料

- [uv 官方文档](https://docs.astral.sh/uv/)
- [CLI 命令参考](https://docs.astral.sh/uv/reference/cli/)
- [项目管理指南](https://docs.astral.sh/uv/guides/projects/)
- [工具运行指南](https://docs.astral.sh/uv/guides/tools/)
