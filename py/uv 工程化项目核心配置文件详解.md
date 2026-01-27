# uv 工程化项目核心配置文件详解

本文档整理了使用 `uv` 管理 Python 项目时最常见的配置文件，并提供了详细的配置示例与解释。

## 1. `pyproject.toml` (项目核心配置)

这是现代 Python 项目的心脏。`uv` 完全拥抱标准，使用此文件管理依赖、元数据、构建系统以及工具配置。

```toml
[project]
name = "my-uv-project"
version = "0.1.0"
description = "一个基于 uv 的极速 Python 工程示例"
readme = "README.md"
requires-python = ">=3.12"
license = { text = "MIT" }
authors = [
    { name = "Your Name", email = "your.email@example.com" }
]
# 核心依赖 (对应 requirements.txt)
dependencies = [
    "fastapi>=0.109.0",
    "pydantic>=2.6.0",
    "httpx>=0.26.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

# --- uv 专用配置 ---
[tool.uv]
# 开发依赖 (对应 requirements-dev.txt)
dev-dependencies = [
    "pytest>=8.0.0",
    "ruff>=0.2.0",
    "mypy>=1.8.0",
]

# --- Ruff 配置 (极速 Linter & Formatter) ---
[tool.ruff]
line-length = 88
target-version = "py312"
src = ["src"]

[tool.ruff.lint]
# 常用插件选择
# E: pycodestyle, F: Pyflakes, I: isort, N: pep8-naming, UP: pyupgrade
select = ["E", "F", "I", "N", "UP"]
ignore = ["D100"] # 忽略缺失的 docstring
fixable = ["ALL"]

[tool.ruff.lint.isort]
known-first-party = ["my_project"]

# --- MyPy 配置 (静态类型检查) ---
[tool.mypy]
python_version = "3.12"
strict = true  # 开启严格模式
warn_return_any = true
warn_unused_configs = true
show_error_codes = true
check_untyped_defs = true

[[tool.mypy.overrides]]
# 针对没有类型注解的第三方库关闭检查
module = ["some_untyped_library.*"]
ignore_missing_imports = true

# --- Pytest 配置 (单元测试) ---
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
# 默认参数：显示详细输出、显示本地变量、计算覆盖率
addopts = "-v --showlocals --cov=src --cov-report=term-missing"
asyncio_mode = "auto"
```

### 关键点解析
*   **`[project]`**: 定义符合 PEP 621 标准的项目元数据和运行时依赖。
*   **`[tool.uv]`**: `uv` 特有的配置区，主要用于定义**开发依赖** (`dev-dependencies`)。
*   **`[build-system]`**: 指定构建后端（如 `hatchling` 或 `setuptools`），`uv` 支持标准构建流程。

---

## 2. `.gitignore` (版本控制忽略规则)

针对 `uv` 和 Python 项目的推荐 Git 忽略配置。

```gitignore
# --- Python 核心 ---
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
share/python-wheels/
*.egg-info/
.installed.cfg
*.egg
MANIFEST

# --- uv 特性 ---
# uv 默认创建的虚拟环境目录 (必须忽略)
.venv/

# --- 测试与覆盖率 ---
htmlcov/
.tox/
.nox/
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
.hypothesis/
.pytest_cache/

# --- 类型检查与 Lint ---
.mypy_cache/
.dmypy.json
dmypy.json
.pyre/
.ruff_cache/

# --- IDE 与编辑器 ---
.vscode/
.idea/
*.swp
.DS_Store

# --- 环境变量 ---
.env
.env.*
!.env.example
```

---

## 3. `uv.lock` (依赖锁定文件)

*   **作用**: 类似于 `package-lock.json` 或 `Cargo.lock`。它精确记录了所有依赖（包括传递依赖）的确切版本、哈希值和来源。
*   **策略**: **必须提交到 Git**。
*   **内容示例**: (自动生成，通常不手动编辑)
    ```toml
    version = 1
    requires-python = ">=3.12"

    [[package]]
    name = "anyio"
    version = "4.2.0"
    source = { registry = "https://pypi.org/simple" }
    dependencies = [
        "idna>=2.8",
        "sniffio>=1.1",
    ]
    ...
    ```
*   **生成方式**: 运行 `uv lock` 或 `uv sync` 时自动创建/更新。

---

## 4. `.python-version` (Python 版本锁定)

*   **作用**: 声明项目所需的特定 Python 解释器版本。
*   **内容**: 仅包含版本号字符串。
    ```text
    3.12.1
    ```
*   **生成方式**: `uv python pin 3.12.1`
*   **使用场景**: 确保所有开发者和 CI 环境使用完全一致的 Python Patch 版本。如果只要求 Minor 版本（如 3.12），通常不需要此文件，依赖 `pyproject.toml` 中的 `requires-python` 即可。

---

## 5. `.env` (环境变量)

虽然 `uv` 不直接管理环境变量，但这是 Python 工程的标准配置。

*   **`.env`**: 存放本地敏感配置（API Key, 数据库密码）。**必须加入 .gitignore**。
*   **`.env.example`**: 存放配置模版（不含敏感值）。**应当提交到 Git**。

**示例 (.env.example)**:
```bash
# 数据库配置
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=

# 应用设置
DEBUG=True
SECRET_KEY=
```