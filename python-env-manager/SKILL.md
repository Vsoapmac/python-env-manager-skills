---
name: python-env-manager
description: Automatically used when the agent is about to execute Python-related commands such as "pip install" / "pip3 install" / "python -m " / "python -u" / "python3 -m" / "python3 -u"—this skill must be loaded before running any Python command. Also used when the user says "安装xxxx" (install a Python third-party package) / "运行xxxx" (run Python code). Subagents must NOT load this skill.
---

<SUBAGENT-STOP>
If you are a subagent dispatched to perform a specific task, skip this skill. Subagents have no interactive interface and cannot present the Python environment selection menu to the user, so loading this skill is pointless.
</SUBAGENT-STOP>

## Trigger Conditions

Triggered when about to execute any of the following commands: `python`, `python3`, `pip`, `pip3`, `pytest`, `python -m`, `python -c`.

## Core Rules

1. **Never skip.** Before executing a Python command for the first time, environment selection must be completed.
2. **Remember the choice.** After selection, set `active_env`; all subsequent Python commands in this session use it directly without asking again.
3. **Switch environments.** When the user says "换环境" (change environment) or "切换环境" (switch environment), clear `active_env` and re-enter the selection flow.

## Scanning Environments

On first trigger (or when the user switches environments), scan the following three categories:

**Conda:** Run `conda env list`, and for each environment get `{env_path}/python --version`. Skip if conda is unavailable.

**Venv:** Search the workspace for `**/pyvenv.cfg`; the parent directory is the venv path. Get the Python version the same way.

**System Python:** Run `python --version` and `which python` (Windows: `where python`). If none exists, stop and report an error.

## Recommendation and Menu

Recommendation priority: `.venv` under the project > conda environment matching the project name > non-base conda environment > system Python.

Menu format:

```
Please select a Python environment:

[A] Conda environments:
    * base (Python 3.11.5)
    * myproject (Python 3.10.8)

[B] venv virtual environments:
    * .\.venv (Python 3.12.0)

[C] System Python:
    Python 3.11.5 @ C:\Python311\

Recommended: [B] .\.venv —— .venv exists in the project directory
```

Mark unavailable items as `[X] xxx unavailable`. Annotate the recommended item with the reason for recommendation.

## Input Parsing

| Input | Meaning |
|------|------|
| `A <name>` | The conda environment with that name |
| `B <index/path>` | The Nth venv, or the venv at the given path |
| `C` | System Python |
| `/abs/path/python` | Custom interpreter path |

Verify the path exists and is executable; if invalid, prompt for re-selection.

## Activation and Usage

After selection, save `active_env = {type, name, python_path}`. Subsequent commands use absolute paths:

- conda: `{conda_prefix}/envs/{name}/python`
- venv: `{venv_path}/Scripts/python` (Win) or `{venv_path}/bin/python`
- system: `python`
- custom: the user-specified path

For pip, always use `{python_path} -m pip`. Briefly indicate the current environment before each execution.
