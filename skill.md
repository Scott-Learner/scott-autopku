---
name: autopku
description: AutoPku - 自动获取PKU课程通知、完成作业、撰写笔记
---

# AutoPku

自动处理北京大学课程相关任务：同步通知、完成作业、撰写笔记。

## 配置说明

### 本地配置（推荐）

**文件位置**: `.claude/settings.local.json`（项目根目录）

```json
{
  "permissions": {
    "allow": ["Skill(update-config)", "Bash(*)"],
    "deny": ["Bash(rm:*)", "Bash(rm -rf:*)"],
    "defaultMode": "bypassPermissions"
  },
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "USER_TYPE": "ant"
  }
}
```

本地配置仅在本项目目录生效，不会影响其他目录的 Claude Code 行为。

**Kimi Code CLI 用户**: 建议设置环境变量 `KIMI_CODE_CLI=1` 以启用 Kimi Agent Team 支持（或通过 `which kimi` 自动检测）。

### 全局配置（可选）

**文件位置**: `~/.claude/settings.json`

内容同上。全局配置会在**所有目录**生效，每次打开 Claude Code 都会显示 bypass 模式警告。除非你只在本机使用 AutoPku，否则不推荐。

### 配置项说明

| 配置项 | 说明 |
|--------|------|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | 启用 Agent Team 并行处理功能 |
| `USER_TYPE` | 用户类型标识（ant 为开发测试账号） |
| `permissions.allow` | 允许使用的工具 |
| `permissions.deny` | 禁止使用的工具（安全保护） |
| `defaultMode` | 默认权限模式 |

## 使用方式

直接告诉我要做什么：

在 AutoPku 桌面端中，这个 skill 应被视为已预加载。用户只输入“同步通知”“看看有什么作业”“课程/选课/DDL”等短请求时，不要先凭记忆或旧摘要回答；第一步必须先运行只读 `pku3b` 命令获取当前教学网数据。如果本轮当前终端里没有实际调用这些 `pku3b` 命令，就不能声称完成同步，必须先调用或报告具体失败命令和原因。

### 内容生成入口 Gate（强制）

在引用任何知识处理或内容生成类 task skill 之前，先检查：**用户本轮输入是否明确要求执行该项具体任务？**

| Task skill | 允许引用的条件 |
|------------|----------------|
| `sub-skills/tasks/do-homework.md` | 用户明确要求完成或解答某项作业 |
| `sub-skills/tasks/write-notes.md` | 用户明确要求撰写或生成课程笔记 |
| `sub-skills/tasks/write-paper.md` | 用户明确要求撰写或生成课程论文 |
| `sub-skills/tasks/make-slides.md` | 用户明确要求制作或生成幻灯片、PPT |

如果条件不成立，**不得读取、引用、加载或通过 Agent 调用上述 task skills，也不得开始其任何准备阶段**。同步结果中出现未完成作业、课件、论文要求或汇报要求，只是状态信息，不构成内容生成授权；“你自己看着办”“都整理一下”“把能处理的都处理了”等笼统授权也不构成对具体生成任务的明确请求。

对于“同步通知”“看看有什么作业”“课程/选课/DDL”等只读请求，只能路由到 `sub-skills/tasks/sync-notices.md`。如果用户意图不明确，先询问用户希望执行哪一项具体生成任务并停止，不得预先加载生成类 task skill。用户给出明确请求后，只路由到其明确指定的 task skill，并继续遵守该 skill 内部的选择、确认和提交 gate。

| 用户意图示例 | 执行的任务 |
|-------------|-----------|
| "同步课程通知" / "看看有什么作业" | 同步当前课程的课件、通知和作业 |
| "完成量子力学的第五次作业" | 完成指定课程作业（解析→解答→渲染→询问→提交） |
| "给逻辑导论写笔记" | 从课件提取数学核心内容撰写笔记 |
| "写马原课程论文" / "生成论文" | 生成课程论文（大纲→正文→渲染→询问） |
| "给XX课做个汇报PPT" / "生成幻灯片" | 基于课件/笔记/论文生成汇报幻灯片（Typst PDF） |

## 执行架构

### 1. 环境检测

执行时自动检测运行环境：

```python
import os
import shutil

if os.environ.get("CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS"):
    RUNTIME = "claude"      # 使用 Agent() tool + SendMessage()
elif os.environ.get("CODEX") == "1":
    RUNTIME = "codex"       # 使用 native subagents
elif os.environ.get("KIMI_CODE_CLI") == "1" or os.environ.get("KIMI") == "1" or shutil.which("kimi"):
    RUNTIME = "kimi"        # 使用 Kimi Agent() + TaskList/TaskOutput
else:
    RUNTIME = "serial"      # 串行执行
```

### 2. 任务路由

根据用户意图，先通过“内容生成入口 Gate”，再引用符合条件的 task skill：

```
用户: "同步课程通知"
  ↓
引用: sub-skills/tasks/sync-notices.md
  ↓
执行同步流程

用户: "完成作业"
  ↓
引用: sub-skills/tasks/do-homework.md
  ↓
执行作业流程（含用户确认）

用户: "写笔记"
  ↓
引用: sub-skills/tasks/write-notes.md
  ↓
执行笔记撰写流程

用户: "写论文" / "生成课程论文"
  ↓
  引用: sub-skills/tasks/write-paper.md
  ↓
执行论文生成流程（大纲→正文→渲染→询问）

用户: "制作幻灯片" / "生成汇报PPT"
  ↓
  引用: sub-skills/tasks/make-slides.md
  ↓
执行幻灯片生成流程（提取→大纲→生成→渲染→询问）
```

### 3. Agent 创建

主 skill 根据 RUNTIME 创建 agents：

```python
# Claude Code 环境
for config in agent_configs:
    Agent({
        "name": config["name"],
        "prompt": config["task"],
        "description": config.get("description", "")
    })

# Kimi Code CLI 环境
for config in agent_configs:
    Agent({
        "description": config.get("description", ""),
        "prompt": config["task"],
        "subagent_type": config.get("subagent_type", "coder")
    })

# Codex 环境
# 使用自然语言描述并行 subagent 任务

# 其他环境
# 串行执行
```

## 核心依赖: pku3b

### 安装

```bash
# 检查是否已安装，并固定本轮实际路径
PKU3B="$(command -v pku3b || true)"
test -n "$PKU3B" || echo "NOT_FOUND"
test -z "$PKU3B" || "$PKU3B" --version
```

**版本要求**：
- 当前验证版本：`pku3b 0.16.1`；最低使用 `0.16.1`
- 执行时先检查 [sshwy/pku3b releases](https://github.com/sshwy/pku3b/releases) 的最新稳定版，不要继续安装文档中曾使用的 `0.11.0`
- macOS 优先使用 `brew install sshwy/tools/pku3b`；已安装旧版时使用 Homebrew 升级
- 具体安装、升级和直接下载方式见 `sub-skills/tools/pku3b-setup.md`

**重要**:
- 正确仓库: `sshwy/pku3b` (原 `yang-er/pku3b` 已失效)
- `0.12.1` 修复了 macOS Blackboard TLS；`0.14.0` 新增课程内容 `cc`；`0.16.1` 修复选课登录重定向

### 登录

`pku3b init` 只在初始化执行一次。后续所有命令自动读取 `cfg.toml` 中的凭证。

**步骤**：

```bash
# 1. 检查 Blackboard 登录是否可用
pku3b a ls
```

如果正常返回作业列表，跳过步骤 2-3，直接进入任务。如果命令失败，先读取原始错误；只有错误明确表明凭证缺失或账号密码无效时才执行步骤 2。网络、TLS、服务器或非 TTY 错误都不能标记为 `NOT_LOGGED_IN`。即使 `ct`、`cc` 或 `grades` 随后要求手机令牌 OTP，也不代表账号密码失效。

如果错误明确表明凭证不存在或账号密码无效，执行步骤 2：

```bash
# 2. 在当前运行环境的 PTY/交互终端里登录
pku3b init
```

```bash
# 3. 验证登录成功
pku3b a ls
```

**交互认证边界**：
- `pku3b init`、`pku3b ct`、`pku3b cc` 和 `pku3b grades` 都可能需要交互式输入
- `The input device is not a TTY` 表示当前命令缺少交互终端，不等于登录失效；改用 PTY/交互终端重跑原命令，不要因此重新初始化凭证
- 手机令牌 OTP、密码和验证码必须由用户直接在交互终端输入；不要放进命令参数、聊天、日志、expect 脚本或明文文件
- 用户暂时无法完成 OTP 时，保留已经成功取得的作业/公告结果，将课表、课件或成绩标记为“认证待完成”，不得声称完整同步

### 常用命令

```bash
# 作业
pku3b a -f ls                           # 强制刷新当前学期未完成作业
pku3b a -f ls -a                        # 强制刷新当前学期全部作业（含已完成）
pku3b a ls --all-term                   # 所有学期作业（含已完成，带 ANSI 颜色码）
pku3b a download <ID> -d <dir>          # 下载附件
pku3b a submit <ID> <file>              # 提交作业

# 公告
pku3b ann -f ls                         # 强制刷新当前学期公告
pku3b ann show <ID>                     # 查看公告详情

# 课程内容/课件 (v0.14.0+，可能要求 OTP)
pku3b cc -f ls                          # 列出当前学期课程内容
pku3b cc down <COURSE_ID:CONTENT_ID> -o <dir> --course-title "<course>"
# 如需另存条目描述，再追加：--output-desc <file>

# 课表与成绩（可能要求 OTP）
pku3b ct -f -r                          # 获取当前课表原始 JSON
pku3b grades -f                         # 查看当前学期成绩；仅在用户明确要求时读取

# 选课（可选核对）
pku3b s -d major show                   # 主修课程（可能返回302，作为可选）
```

## Task Skills 索引

| Task | 文件 | 说明 |
|------|------|------|
| 同步通知 | `tasks/sync-notices.md` | 获取课件/作业/公告，生成摘要，并行处理课程 |
| 完成作业 | `tasks/do-homework.md` | 解析PDF→解答→渲染→询问用户→提交 |
| 撰写笔记 | `tasks/write-notes.md` | 从课件提取数学核心，去除噪声 |
| 撰写论文 | `tasks/write-paper.md` | 从大纲到正文到编译，支持LaTeX/Word双模式 |
| 生成幻灯片 | `tasks/make-slides.md` | 基于pkusli模板生成Typst汇报幻灯片 |

## Tool Skills 索引

| Tool | 文件 | 说明 |
|------|------|------|
| pku3b配置 | `tools/pku3b-setup.md` | 安装、登录、命令参考 |
| 数据解析 | `tools/data-parser.md` | ANSI颜色码处理、正则提取 |
| PDF读取 | `tools/pdf-reader.md` | PyMuPDF/pdfplumber 代码示例 |
| 图片处理 | `tools/image-handler.md` | 论文配图搜索、绘制与LaTeX插入 |
| Agent模板 | `tools/agent-helpers.md` | Coordinator/Parser/Solver/Writer/Submitter Prompts |
| 幻灯片渲染 | `tools/slide-renderer.md` | pkusli模板获取、main.typ组装、typst编译 |

## Runtime Skills 索引

| Runtime | 文件 | 说明 |
|---------|------|------|
| 环境检测 | `runtime/_detect.md` | 自动检测 Claude/Codex/Kimi/Fallback |
| Agent 创建 | `runtime/create-agent.md` | 统一的跨平台 Agent 创建接口 |
| Claude Team | `runtime/claude-team.md` | Claude Code Agent Team 语法 |
| Codex Subagent | `runtime/codex-subagent.md` | Codex native subagent 语法 |
| Kimi Team | `runtime/kimi-team.md` | Kimi Code CLI Agent Team 语法 |

## 关键踩坑记录

### 1. 课程筛选遗漏

**问题**: 选课系统显示"未选上"但作业系统有记录（如"学术英语写作"）。

**解决**: 从作业系统提取所有课程，对比选课系统，标注不一致状态。

### 2. 附件下载问题

| 情况 | 说明 | 示例 |
|------|------|------|
| 无附件 | 提交类作业无下载内容 | 操作系统（实验班）|
| 教学网无作业 | 使用Canvas/微信群 | 哲学导论、逻辑导论 |
| 体育课 | 通常无在线作业 | 太极拳 |

**检查**: `grep "课程名" /tmp/pku_assignments_raw.txt | grep "附件"`

### 3. 数据格式问题

- **列表命令无 `--json` 参数**: `a`、`ann`、`cc` 输出纯文本并可能带 ANSI 颜色码，需先清理再解析
- **`pku3b a ls -a` 不完整**: 只显示有作业的课程，可用 `pku3b s -d major show` 做可选核对
- **`--all-term` 返回所有历史学期**: 需筛选当前学期
- **课程内容 ID 是复合 ID**: `pku3b cc ls` 每行末尾的 `course_id:content_id` 必须整体传给 `pku3b cc down`
- **OTP 与登录失败不同**: 非 TTY/OTP 提示应在交互终端继续原命令，不能据此运行 `pku3b init`

### 4. 用户确认（重要）

完成作业前**必须**:
1. 列出所有待交作业
2. 使用 `AskUserQuestion` 让用户选择
3. 二次确认后才创建 Agent Team
4. 渲染完成后询问是否提交

**禁止**: 自动选择最新作业、未经确认直接提交

## 输出规范

### 同步通知输出

```
test/
├── 通知摘要汇总.md
├── {course1}/
│   ├── 作业/           # 下载的附件
│   ├── 通知/           # 公告详情
│   ├── 资料/           # 课程资料
│   └── 通知摘要.md     # 该课程摘要
└── {course2}/
    └── ...
```

### 完成作业输出

```
{course}/
├── 作业/
│   ├── Homework202605.pdf          # 原题
│   ├── homework_parsed.json        # 解析结果
│   ├── answers.json                # 解答数据
│   ├── Homework202605_answer.md    # Markdown答案
│   └── Homework202605_answer.pdf   # PDF答案
└── 提交/                           # 最终提交文件
    └── Homework202605_answer.pdf
```

### 生成幻灯片输出

```
{course}/
└── slides/
    ├── main.typ              # Typst 源文件
    ├── slides.pdf            # 最终 PDF
    ├── outline.md            # 幻灯片大纲
    └── figures/              # 图片目录
        └── ...
```

## 安全规则

- 从不回显密码到日志
- 不自动提交作业（必须用户确认）
- 不自动选择作业（必须用户选择）
- 安全处理课程名中的特殊字符

## 参考

- 原完整 skill 备份: `archive/archived-skill.md`
- pku3b 仓库: https://github.com/sshwy/pku3b
