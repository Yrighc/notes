# uv 项目 .gitignore 示例

```gitignore
# --- Python 核心 ---
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
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
# uv 默认创建的虚拟环境目录
.venv/
# uv 可能会生成的 Python 版本说明文件
# .python-version

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
cover/

# --- 类型检查与 Lint ---
.mypy_cache/
.dmypy.json
dmypy.json
.pyre/
.phry/
.ruff_cache/

# --- IDE 与编辑器 ---
.vscode/
.idea/
*.swp
*.swo
.DS_Store

# --- 环境变量与私密信息 ---
.env
.venv
env.bak
venv.bak
```
