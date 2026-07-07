---
name: python-env-manager
description: 在你执行python命令前触发该skills。扫描系统上可用的 Python 环境（conda、venv、系统 Python），向用户展示结构化选择菜单，并在整个会话中跟踪所选环境。不要在未经过此选择流程的情况下静默使用系统 Python。
---

# Python 环境管理器

## 目的

在执行任何 Python 命令之前，此 skill 拦截并强制执行显式的环境选择流程。它发现系统上所有可用的 Python 环境，以结构化菜单展示给用户，记录用户的选择，并将后续所有 Python 操作定向到所选环境，在会话剩余时间内持续生效。

## 触发时机

当即将执行以下任何操作时，立即触发此 skill：

- 执行 `.py` 文件（`python xxx.py`、`python3 xxx.py`）
- 包管理命令（`pip install`、`pip uninstall`、`pip list` 等）
- 测试运行器调用（`pytest`、`python -m pytest`）
- 任何 `python -c "..."` 或 `python -m <module>` 命令
- Jupyter notebook 相关操作
- 创建或修改 Python 项目配置文件（`requirements.txt`、`setup.py`、`pyproject.toml`）
- 任何需要通过 Python 解释器执行的命令

## 核心规则

1. **绝不跳过选择流程。** 当尚未选定环境时，拦截每个 Python 命令并在继续之前执行完整流程。
2. **记住选择结果。** 一旦选定环境，在同一会话中的后续所有 Python 操作中复用该环境，不再重复询问。
3. **使用绝对路径。** 环境激活后，所有 Python 命令必须使用解释器的绝对路径，而非依赖系统 `PATH`。

## 状态机

维护以下四个内部状态之一：

| 状态       | 含义                     | 进入条件                     |
|------------|--------------------------|------------------------------|
| `IDLE`     | 尚未触发环境选择         | skill 刚加载                  |
| `SCANNING` | 正在扫描可用环境         | 首次触发，或用户要求切换环境 |
| `ASKING`   | 扫描完成，等待用户选择   | SCANNING 完成                 |
| `ACTIVE`   | 环境已选定，直接使用     | 用户做出有效选择             |

状态转换：

- `IDLE` →（首次触发）→ `SCANNING`
- `SCANNING` →（扫描完成）→ `ASKING`
- `ASKING` →（用户选择）→ `ACTIVE`
- 任意状态 →（用户说"切换环境"/"换环境"）→ `SCANNING`
- 任意状态 →（用户说"重置环境"）→ `IDLE`

---

## SCANNING 阶段

进入 `SCANNING` 状态后，执行以下扫描。三个顶层分类（Conda / venv / 系统 Python）可以并行执行，但每个分类内部的子步骤必须按顺序执行。

### 扫描一：Conda 环境

执行：

```bash
conda env list
```

解析输出，提取环境名称和路径。输出示例：

```
base                 *  /home/user/miniconda3
data-science            /home/user/miniconda3/envs/data-science
web-dev                 /home/user/miniconda3/envs/web-dev
```

提取字段：
- `env_name`：环境名称（如 `base`、`data-science`）
- `env_path`：环境路径

对每个 conda 环境，获取其 Python 版本：

```bash
{env_path}/bin/python --version      # Linux/Mac
{env_path}\python.exe --version      # Windows
```

将 `python_version`（如 `3.11.5`）追加到每个条目。

如果 `conda` 命令不存在或执行失败，设置 `conda_available = false`。

### 扫描二：venv 虚拟环境

使用 `search_file` 工具在工作区目录中搜索 `**/pyvenv.cfg`（每个 venv 根目录包含此文件）。

对每个发现的 `pyvenv.cfg`，将其父目录作为 venv 路径。然后获取 Python 版本：

```bash
{venv_path}/bin/python --version         # Linux/Mac
{venv_path}\Scripts\python.exe --version # Windows
```

提取字段：
- `venv_path`：venv 目录路径
- `python_version`：Python 版本

如果未找到任何 venv，设置 `venvs = []`（空列表）。

### 扫描三：系统 Python

执行：

```bash
python --version
```

同时获取路径：

- Windows：`where python`
- Linux/Mac：`which python`

提取字段：
- `sys_python_version`：版本字符串
- `sys_python_path`：解释器路径

如果 `python` 命令不存在，立即报告错误并停止：

> 未检测到系统 Python，请先安装 Python。

### 扫描结果汇总

扫描完成后，构建以下内部数据结构。从 `conda env list` 中的 base 环境路径推导 `conda_prefix`（去掉末尾的 `/base` 或 `\base`）：

```
conda_available: true/false
conda_prefix: "conda 安装根目录"  （如 /home/user/miniconda3、C:\Users\xxx\miniconda3）
conda_envs: [{name, path, python_version}, ...]

venvs: [{path, python_version}, ...]

sys_python: {version, path}
```

然后立即转入 `ASKING` 阶段。

---

## ASKING 阶段

进入 `ASKING` 后，按以下顺序执行：

1. 根据扫描结果构建选择菜单
2. 使用推荐逻辑确定推荐项
3. 展示菜单并等待用户回复

### 菜单格式

```
检测到即将执行 Python 操作。

请选择执行环境：

[A] Conda 环境：
    * base (Python 3.11.5)
    * data-science (Python 3.10.8)

[B] venv 虚拟环境：
    * ./.venv (Python 3.12.0)
    * [+] 创建新 venv

[C] 系统 Python：
    [!] Python 3.11.5 @ C:\Python311\

推荐：[A] data-science —— 检测到与项目名匹配的 conda 环境。

请选择环境（如：A data-science / B 1 / C / 或自定义路径）：
```

### 各分组展示规则

**[A] Conda：**
- 若 `conda_available = true`：列出所有 conda 环境，格式为 `* {env_name} (Python {version})`
- 若 `conda_available = false`：显示 `[X] Conda 未安装或不可用`

**[B] venv：**
- 列出所有发现的 venv，格式为 `* {相对路径} (Python {version})`
- 始终在末尾追加 `* [+] 创建新 venv`
- 若列表为空，仅显示 `* [+] 创建新 venv`

**[C] 系统 Python：**
- 始终显示 `[!] Python {version} @ {path}`
- 若系统 Python 不可用：`[X] 未检测到系统 Python`
- 当用户选择系统 Python 时，显示额外警告：
  > [!] 使用系统 Python 可能导致依赖冲突。建议使用 venv 或 conda 隔离环境。确认使用系统 Python？（输入 yes 确认）

### 推荐逻辑

按以下优先级依次检查确定推荐项（匹配第一个后停止）：

1. **项目专属 venv**：如果发现的 venv 路径中包含工作区根目录名称，推荐该 venv。
   - 提示语：*"项目目录下存在 .venv，推荐使用项目专属环境。"*

2. **匹配的 conda 环境**：如果 conda 环境名包含工作区根目录名称（不区分大小写），推荐该 conda 环境。
   - 提示语：*"检测到与项目名匹配的 conda 环境。"*

3. **非 base 的 conda 环境**：如果存在非 base 的 conda 环境，推荐第一个非 base 环境。
   - 提示语：*"推荐使用 conda 环境，避免污染系统 Python。"*

4. **系统 Python**：兜底方案。
   - 提示语：*"未找到项目专属环境，使用系统 Python。建议创建 venv 隔离项目依赖。"*

### 用户输入解析

按以下规则解析用户回复：

| 输入示例                      | 解析结果                         |
|-------------------------------|----------------------------------|
| `A data-science`              | conda 环境 `data-science`        |
| `A base`                      | conda 环境 `base`                |
| `B 1`                         | venv 列表中第 1 个               |
| `B .venv`                     | 路径为 `.venv` 的 venv           |
| `B 创建新venv` / `B new`      | 创建新 venv                      |
| `C`                           | 系统 Python                      |
| `/usr/bin/python3.12`         | 自定义绝对路径                   |
| `my-conda-env`                | 自定义 conda 环境名              |

**验证规则：**
- 所选路径必须存在且可执行。
- 若路径不存在，回复 `路径 {path} 不存在，请重新选择。`，停留在 `ASKING`。
- 若 conda 环境名不在列表中但 conda 可用，验证 `{conda_prefix}/envs/{name}` 是否存在，存在则接受。
- 若 conda 环境名不在列表中且 conda 不可用，回复 `Conda 不可用，无法验证环境 "{name}"。`，停留在 `ASKING`。
- 若输入无法解析，回复 `无法识别输入，请按格式重新选择。`，停留在 `ASKING`。

解析成功后，将选择保存为 `active_env`，转入 `ACTIVE`。

### 创建新 venv 子流程

当用户选择创建新 venv 时：

1. 询问 venv 名称（默认：`.venv`）
2. 确定用于创建 venv 的 Python 解释器：
   - 若 conda 可用：优先询问是否使用 conda base 环境的 Python
   - 否则：使用系统 Python
3. 执行创建命令：

   ```bash
   {selected_python} -m venv {venv_name}
   ```

4. 验证创建结果：检查 `{venv_name}/pyvenv.cfg` 是否存在。
5. 将新创建的 venv 设为 `active_env`，转入 `ACTIVE`。

---

## ACTIVE 阶段

进入 `ACTIVE` 状态后，`active_env` 已设置。后续所有 Python 操作直接使用此环境，**不再询问用户**。

### active_env 数据结构

```
active_env = {
    "type": "conda" | "venv" | "system" | "custom",
    "name": "环境名称",
    "python_path": "解释器绝对路径"
}
```

所有 pip 操作通过 `{python_path} -m pip` 调用；无需单独的 `pip_path` 字段。

### 按类型激活环境

**Conda 环境：**

`python_path` 为 `{conda_prefix}/envs/{env_name}/bin/python`（Linux/Mac）或 `{conda_prefix}\envs\{env_name}\python.exe`（Windows）。

```bash
{conda_prefix}/envs/{env_name}/bin/python script.py
{conda_prefix}/envs/{env_name}/bin/python -m pip install requests
```

**venv 环境：**

`python_path` 为 `{venv_path}/bin/python`（Linux/Mac）或 `{venv_path}\Scripts\python.exe`（Windows）。

```bash
{venv_path}/bin/python script.py
{venv_path}/bin/python -m pip install requests
```

**系统 Python：**

```bash
python script.py
python -m pip install requests
```

**自定义路径：**

直接使用用户提供的绝对路径。使用前需验证：

1. 路径指向存在的文件。
2. 文件是可执行文件（非目录）。
3. 若验证失败，回复 `路径 {path} 无效，请确认输入的是 Python 解释器的完整路径。`，返回 `ASKING`。

```bash
{user_specified_path} script.py
{user_specified_path} -m pip install requests
```

### 会话级别记忆

- `active_env` 在会话中一旦设置便持久保留，不再改变。
- 将 `active_env` 作为内部上下文变量维护，在所有后续工具调用中引用。
- 新会话开始时（skill 重新加载），`active_env` 重置为 null，状态回到 `IDLE`。
- 后续所有 Python 操作（`pip install`、`pytest`、`python -c` 等）均使用 `active_env.python_path`。
- 不再向用户重新确认。

### 环境切换

如果用户在会话过程中说"切换环境"、"换环境"：

1. 清除 `active_env`。
2. 进入 `SCANNING` 重新扫描和选择。

如果用户说"重置环境"：

1. 清除 `active_env`。
2. 回到 `IDLE` 状态（下次 Python 操作时触发 `SCANNING`）。

### 执行提示

在每次 Python 命令前，简要提示当前使用的环境（仅展示信息，无需确认）：

```
使用环境：data-science (conda)
> python script.py
```

---

## 错误处理

| 场景                               | 处理方式                                                         |
|------------------------------------|------------------------------------------------------------------|
| `conda` 命令不存在                 | [A] 分组标注为不可用，跳过 conda 扫描                             |
| `conda env list` 执行失败          | 同 conda 不存在                                                  |
| 无任何 venv 目录                   | [B] 分组仅显示"创建新 venv"                                       |
| 系统 `python` 命令不存在           | 立即停止："未检测到系统 Python，请先安装 Python。"                |
| venv 的 python 路径不存在          | 将该 venv 从选项列表中排除                                        |
| 扫描命令超时或返回异常             | 标记该项为不可用，继续其他扫描                                    |
| 所选环境路径在执行时已不存在       | "环境路径 {path} 已不存在。"返回 ASKING                           |
| 用户输入无法解析                   | "无法识别输入，请重新选择。"停留在 ASKING                          |
| SCANNING 阶段被中断                | 下次触发时从 SCANNING 重新开始                                    |
| conda 环境的 Python 可执行文件损坏 | 将该 conda 环境从选项列表中排除                                   |
| 自定义路径不是有效的 Python 解释器 | "路径 {path} 无效。"                                              |
| 创建 venv 失败                     | 报告具体错误（磁盘空间、权限等），询问是否重试或选择其他环境      |

## 跨平台适配

**路径分隔符判断：**
- 通过系统环境判断：`%OS%` 包含 "Windows" 则为 Windows，否则为 Unix-like。
- Windows：使用 `\` 和 `.exe` 后缀。
- Linux/Mac：使用 `/` 和无后缀。

**命令适配：**
- Windows：`where python`
- Linux/Mac：`which python`

**Python 可执行文件：**
- Windows：`python.exe`
- Linux/Mac：`python` 或 `python3`（优先 `python3`）
