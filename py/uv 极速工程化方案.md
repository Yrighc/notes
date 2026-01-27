# Python 极速工程化方案 (Powered by uv)

> **Version:** 1.0  
> **Toolchain:** [uv](https://github.com/astral-sh/uv) (by Astral)  
> **Philosophy:** Project-First, High Performance, Standards Compliant (PEP 621)

## 1. 核心理念

本方案旨在解决传统 Python 工程流中工具破碎（pip, venv, poetry, pyenv 各自为战）与依赖解析缓慢的问题。

**uv 的核心优势：**
* **All-in-One**：单一工具接管 Python 版本管理 (`pyenv`)、虚拟环境 (`venv`)、依赖解析 (`poetry`) 和包安装 (`pip`)。
* **Rust Native**：依赖解析与安装速度比常规工具快 10-100 倍。
* **Project-First**：基于项目声明自动下载对应的 Python Interpreter，无需全局安装。

---

## 2. 项目初始化 (Zero to Hero)

### 2.1 环境准备
MacOS / Linux 一键安装 uv：
```bash
curl -LsSf https://astral-sh.github.io/uv/install.sh | sh
```

### 2.2 创建工程

建议采用 **Src-Layout** 结构，避免 Import 路径污染。

```bash
# 1. 创建项目目录
mkdir my-service && cd my-service

# 2. 初始化 uv 项目 (生成 pyproject.toml)
uv init

# 3. [关键] 锁定 Python 版本
# uv 会自动下载并隔离安装该版本，不依赖系统 Python
uv python pin 3.12

# 4. 调整目录结构为 Src-Layout
mkdir -p src/app tests
mv hello.py src/app/main.py
```

### 2.3 依赖管理

不需要手动激活虚拟环境，`uv add` 会自动处理 `.venv`。

```bash
# 安装生产依赖
uv add fastapi uvicorn pydantic-settings

# 安装开发依赖 (Dev Group)
uv add --dev ruff mypy pytest
```

---

## 3. 标准目录结构

```text
my-service/
├── .venv/                   # [自动生成] 虚拟环境 (勿提交 Git)
├── .python-version          # [关键] 锁定当前项目的 Python 版本 (如 3.12.1)
├── uv.lock                  # [关键] 依赖锁文件 (必须提交 Git)
├── pyproject.toml           # 项目核心配置 (PEP 621)
├── src/                     # 源代码目录
│   └── app/
│       ├── __init__.py
│       └── main.py
├── tests/                   # 测试代码
├── Dockerfile               # 容器化构建文件
└── .gitignore
```

---

## 4. 常用命令速查 (Cheat Sheet)

| 场景         | 命令                        | 说明                                       |
| ---------- | ------------------------- | ---------------------------------------- |
| **初始化**    | `uv init`                 | 初始化当前目录                                  |
| **版本锁定**   | `uv python pin <version>` | 绑定特定 Python 版本 (如 3.11, 3.12)            |
| **同步环境**   | `uv sync`                 | **核心命令**。根据 lock 文件还原环境 (类似 npm install) |
| **添加依赖**   | `uv add <package>`        | 安装并更新 lock 文件                            |
| **添加开发依赖** | `uv add --dev <package>`  | 仅用于开发环境 (Linter, Tester)                 |
| **移除依赖**   | `uv remove <package>`     | 卸载并清理无用子依赖                               |
| **运行脚本**   | `uv run python main.py`   | 在虚拟环境中执行，无需 activate                     |
| **运行工具**   | `uv run pytest`           | 执行开发工具                                   |
| **查看树**    | `uv tree`                 | 查看依赖关系树                                  |
| **升级依赖**   | `uv lock --upgrade`       | 忽略 lock 文件，尝试升级所有包到允许的最新版                |

---

## 5. 工作流最佳实践

### 5.1 新成员入职 (Onboarding)

新同事 Clone 代码后，仅需一步即可开始工作，无需手动配置 Python 环境：

```bash
git clone <repo-url>
cd my-service
# 自动下载 Python runtime，创建 venv，安装所有依赖
uv sync
```

### 5.2 持续集成 (CI/CD)

在 GitHub Actions / GitLab CI 中，利用 `uv` 的缓存与极速安装。

```yaml
# GitHub Actions 示例片段
- name: Install uv
  uses: astral-sh/setup-uv@v1

- name: Install dependencies
  # --frozen: 严格使用 uv.lock，不更新版本
  run: uv sync --frozen
    
- name: Run Tests
  run: uv run pytest
```

### 5.3 容器化构建 (Docker)

使用 Multi-stage 构建，结合 `uv` 的缓存挂载，实现秒级构建。

**Dockerfile 示例：**

```dockerfile
# Stage 1: Builder
FROM python:3.12-slim-bookworm AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

WORKDIR /app

# 优化层级：先拷贝依赖文件
COPY pyproject.toml uv.lock .

# 安装依赖到系统路径 (在 Docker 里不需要 venv 隔离)
# --system: 安装到系统 Python 环境
# --no-dev: 不安装开发依赖
RUN uv sync --frozen --no-dev --system

# Stage 2: Runner (Distroless 或 Slim)
FROM python:3.12-slim-bookworm

WORKDIR /app

# 从 Builder 拷贝安装好的包 (位于 site-packages)
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# 拷贝源码
COPY src/ .

# 运行
CMD ["python", "-m", "app.main"]
```

---

## 6. 配置示例 (pyproject.toml)

```toml
[project]
name = "my-service"
version = "0.1.0"
description = "High performance Python service"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.109.0",
    "uvicorn>=0.27.0",
]

[tool.uv]
dev-dependencies = [
    "pytest>=8.0.0",
    "ruff>=0.3.0",
    "mypy>=1.8.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

# Ruff 配置 (Linter & Formatter)
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "UP"]
```
