# uv 极速工程化方案
## 一、 核心思路：Project-First
在 `uv` 的体系下，我们不再把“Python 版本管理”和“项目依赖管理”分开。

- **以前**：先用 `pyenv` 装 Python，再用 `venv` 建环境，再用 `pip` 装包。
    
- **现在**：你只管描述你的项目需要什么，`uv` 会自动把 Python 解释器、虚拟环境、依赖包全搞定。
## 二、 从零创建工程 (Step-by-Step)
假设我们要创建一个名为 `fast-api-service` 的后端项目。
1. 初始化项目
`uv init` 会直接生成标准的 `pyproject.toml` 和 `.python-version` 文件。
```bash
# 创建目录
mkdir fast-api-service
cd fast-api-service

# 初始化 (你会发现目录下多了一个 .venv，且它是自动生成的！)
uv init
```
2. 锁定 Python 版本
这是 `uv` 最爽的地方。你不需要去官网下载 Python，直接指定版本，它会自动下载并绑定到当前项目。
```bash
# 强制项目使用 Python 3.12
uv python pin 3.12
```
3. 调整为 Src-Layout (工程化标配)
4. 