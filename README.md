# python-env-manager

> AI Skill —— Python 环境管理器，让 AI 再也不会静默污染你的系统 Python。

## 这是什么

`python-env-manager` 是一个 AI Skill。它会在 AI 执行任何 Python / pip / pytest 命令**之前**，自动扫描你系统上所有可用的 Python 环境（Conda 环境、项目内的 venv、系统全局 Python），然后用一份清晰的结构化菜单让你选择要用哪个环境。

选定之后，整个会话期间 AI 都会记住这个选择，所有 Python 操作全部走你指定的环境——**绝不静默用系统 Python**。

---

## 用它的好处

| 痛点 | 本 skill 如何解决 |
|------|-------------------|
| AI 经常静默用系统 Python，装一堆全局包导致依赖冲突 | 强制走环境选择流程，不选不让跑 |
| 电脑上有多个 conda 环境和 venv，不知道该用哪个 | 自动扫描全部环境，智能推荐最匹配的项目环境 |
| pip install 不知道装到哪里去了 | 始终使用绝对路径调用解释器，环境路径透明可见 |
| 每次都要手动 `conda activate`，繁琐易忘 | 选定一次，全会话复用，无需手动激活 |
| 不同项目依赖打架 | 支持独立选择每个项目的运行环境，互不干扰 |

**一句话总结**：把"我该用哪个 Python"这件事从"AI 猜"变成"你决定"，且只需决定一次。

---

## 用之前 vs 用之后

### 用之前（没有这个 skill）

```
用户：帮我跑一下这个脚本
AI：（静默）python script.py
     ↑ 这里的 python 到底是谁？
       是系统 Python 3.8？还是 conda base 的 3.11？
       装了个 requests 到底去了哪个 site-packages？
       不知道。可能跑通了，也可能炸了，看运气。
```

典型问题：
- AI 默认用 `python` 命令，实际指的系统 Python，频繁 `pip install` 污染全局环境
- 项目明明配了 `.venv`，AI 完全无视
- 不同用户、不同机器的环境千差万别，AI 猜不准
- pip install 的包找不到、版本冲突频频出现

### 用之后（加载了这个 skill）

```
用户：帮我跑一下这个脚本
AI：
  检测到即将执行 Python 操作。

  请选择执行环境：

  [A] Conda 环境：
      * base (Python 3.11.5)
      * my-project (Python 3.10.8)

  [B] venv 虚拟环境：
      * ./.venv (Python 3.12.0)
      * [+] 创建新 venv

  [C] 系统 Python：
      [!] Python 3.11.5 @ C:\Python311\

  推荐：[B] ./.venv —— 项目目录下存在 .venv，推荐使用项目专属环境。

用户：B .venv
AI：
  使用环境：.venv (venv)
  > ./.venv/Scripts/python.exe script.py
```

效果：
- 每一步都明明白白——用的是哪个 Python、装在哪个环境
- 系统 Python 不会被意外污染
- 项目 `requirements.txt` 的依赖精确安装在项目自己的 venv 里
- 切换项目时可以说"换环境"，立刻重新选择

---

## 怎么用

### 第一步：安装 Skill

将 `python-env-manager` 文件夹复制到对应 AI 编程助手的 skills 目录下即可。

#### Claude Code（Anthropic）

```bash
# 用户级（所有项目通用）
cp -r python-env-manager ~/.claude/skills/

# 项目级（跟随仓库，团队共享）
cp -r python-env-manager <你的项目>/.claude/skills/
```

#### Cursor

```bash
# 项目级
cp -r python-env-manager <你的项目>/.cursor/skills/
```

#### Windsurf

```bash
# 项目级
cp -r python-env-manager <你的项目>/.windsurf/skills/
```

#### Augment Code

```bash
# 用户级
cp -r python-env-manager ~/.augment/skills/

# 项目级
cp -r python-env-manager <你的项目>/.augment/skills/
```

#### Continue

```bash
# 将 SKILL.md 内容作为自定义 system prompt 添加到 Continue 配置中
# 编辑 ~/.continue/config.json，在 systemMessage 字段中引入本 skill 的指令
```

#### 其他 Agent

大多数支持 skill 机制的 Agent 遵循类似的目录结构。将 `python-env-manager` 放到对应 Agent 的 skills 目录即可：

```
用户级：~/.<agent-name>/skills/python-env-manager/
项目级：<你的项目>/.<agent-name>/skills/python-env-manager/
```

### 第二步：正常使用

什么都不用做。当你在对话中让 AI 执行任何 Python 操作时，skill 会自动触发：

- `帮我运行 main.py`
- `pip install numpy`
- `跑一下 pytest`
- `新建一个 requirements.txt`

AI 会先弹出环境选择菜单，你选一个，然后它就开始干活。

### 第三步：环境切换

会话中途想换环境？直接说：

- `换环境` 或 `切换环境` —— 重新扫描并选择
- `重置环境` —— 清除选择，下次 Python 操作再触发选择

### 使用技巧

| 场景 | 推荐做法 |
|------|---------|
| 新项目还没创建 venv | 选 `B [+] 创建新 venv`，一步建好隔离环境 |
| 项目有 conda 环境 | skill 会自动推荐匹配的 conda 环境名 |
| 想直接指定路径 | 直接输入解释器绝对路径，如 `/usr/bin/python3.12` |
| 选了系统 Python | 会有二次确认警告，防止误操作 |
| 想看看有哪些环境可用 | 说 `换环境`，会列出全部可用环境 |

---

## 环境要求

- 系统至少有一个可用的 Python 解释器（否则报错并停止）
- Conda 为可选依赖，没有也不影响使用（会跳过 Conda 扫描）
- 支持 Windows / Linux / macOS

---

## 文件结构

```
python-env-manager/
└── SKILL.md          # skill 定义文件（核心）
```
