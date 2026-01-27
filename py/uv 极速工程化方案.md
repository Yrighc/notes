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

2.2 创建工程

## 2. 项目初始化 (Zero to Hero)

### 2.1 环境准备
MacOS / Linux 一键安装 uv：
```bash
curl -LsSf [https://astral-sh.github.io/uv/install.sh](https://astral-sh.github.io/uv/install.sh) | sh