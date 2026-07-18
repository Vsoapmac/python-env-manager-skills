---
name: python-env-manager
description: 在智能体即将执行 "pip install" / "pip3 install" / "python -m "/ "python -u" / "python3 -m" /"python3 -u" 等python相关命令时自动使用--在运行任何python命令时必须加载此技能。或者用户说"安装xxxx"(安装python第三方库)/"运行xxxx"(运行python代码)时使用。子agent不要加载此技能。
---

<SUBAGENT-STOP>
如果你是作为子智能体被分派来执行特定任务的，跳过此技能。子智能体没有交互界面，无法向用户展示 Python 环境选择菜单，加载此技能无意义。
</SUBAGENT-STOP>

## 触发条件

当即将执行以下命令时触发：`python`、`python3`、`pip`、`pip3`、`pytest`、`python -m`、`python -c`。

## 核心规则

1. **绝不跳过。** 首次执行 Python 命令前，必须先完成环境选择。
2. **记住选择。** 选定后设置 `active_env`，该会话内后续所有 Python 命令直接使用，不再询问。
3. **切换环境。** 用户说"换环境"或"切换环境"时，清除 `active_env`，重新进入选择流程。

## 扫描环境

首次触发（或用户切换环境）时，扫描以下三类：

**Conda：** 执行 `conda env list`，对每个环境获取 `{env_path}/python --version`。若 conda 不可用则跳过。

**Venv：** 搜索工作区 `**/pyvenv.cfg`，父目录即为 venv 路径，同样获取 Python 版本。

**系统 Python：** 执行 `python --version` 和 `which python`（Windows: `where python`）。若不存在则停止并报错。

## 推荐与菜单

推荐优先级：项目下 `.venv` > 项目名匹配的 conda 环境 > 非 base conda 环境 > 系统 Python。

菜单格式：

```
请选择 Python 环境：

[A] Conda 环境：
    * base (Python 3.11.5)
    * myproject (Python 3.10.8)

[B] venv 虚拟环境：
    * .\.venv (Python 3.12.0)

[C] 系统 Python：
    Python 3.11.5 @ C:\Python311\

推荐：[B] .\.venv —— 项目目录下存在 .venv
```

不可用项标注 `[X] xxx 不可用`。推荐项标注推荐理由。

## 输入解析

| 输入 | 含义 |
|------|------|
| `A <name>` | 对应名称的 conda 环境 |
| `B <序号/路径>` | 第 N 个 venv 或指定路径的 venv |
| `C` | 系统 Python |
| `/abs/path/python` | 自定义解释器路径 |

验证路径存在且可执行，无效则提示重新选择。

## 激活使用

选定后保存 `active_env = {type, name, python_path}`。后续命令使用绝对路径：

- conda: `{conda_prefix}/envs/{name}/python`
- venv: `{venv_path}/Scripts/python`（Win）或 `{venv_path}/bin/python`
- 系统: `python`
- 自定义: 用户指定路径

pip 统一用 `{python_path} -m pip`。每次执行前简要提示当前环境。
