---
name: autopku-task-sync-notices
description: 同步PKU课程课件、作业和通知，为每门课程创建目录结构和摘要
---

# 任务：同步课程通知

在 AutoPku 桌面端中，如果用户只说“同步通知”“看看有什么作业”“DDL”等短请求，必须先执行本任务的 pku3b 读取步骤。不要只基于旧会话、旧摘要或目录里的缓存文件回答。如果本轮当前终端里没有实际调用这些 pku3b 命令，就不能声称完成同步，必须先调用或报告具体失败命令和原因。

## 执行步骤

### 1. 检查并安装 pku3b

```bash
PKU3B="$(command -v pku3b || true)"
test -n "$PKU3B" || echo "NOT_FOUND"
test -z "$PKU3B" || "$PKU3B" --version
```

如未安装或低于 `0.16.1`，按 `tools/pku3b-setup.md` 安装或升级。不要继续使用旧版 `0.11.0`。

### 2. 登录教学网

先运行 `pku3b a ls`。只有错误明确表明凭证缺失或账号密码无效时，才在 PTY/交互终端运行 `pku3b init`；需要账号、密码或验证码时让用户直接在终端输入。不要生成 expect 脚本或明文凭证文件。

`pku3b ct`、`pku3b cc` 和 `pku3b grades` 可能单独要求手机令牌 OTP。`The input device is not a TTY` 表示需要交互终端，不等于凭证失效；在 PTY 中重跑原命令，不要改跑 `pku3b init`，也不要把 OTP 放进命令参数或日志。

### 3. 获取数据

先获取不依赖门户 OTP 的 Blackboard 数据：

```bash
pku3b a -f ls -a > /tmp/pku_assignments_raw.txt
pku3b ann -f ls > /tmp/pku_announcements_raw.txt
```

再在支持 PTY 的终端中获取当前课表和课程内容；如果出现 OTP 提示，由用户直接在该终端输入：

```bash
pku3b ct -f -r > /tmp/pku_course_table_raw.json
pku3b cc -f ls > /tmp/pku_course_contents_raw.txt
```

选课查询只作为可选核对，不得让它的失败阻断作业、公告或课件同步：

```bash
pku3b s -d major show > /tmp/major_courses.txt 2>/dev/null || true
```

如果用户没有完成 OTP，继续处理已成功取得的作业和公告，并在报告中明确标记“课表/课程内容待认证”。此时不能声称课件已同步。

### 4. 解析数据

引用 `sub-skills/tools/data-parser.md`，先清除 ANSI 控制码，再分别解析并保存：

- `/tmp/pku_assignments.json`：`course`、`assignment`、`status`、`id`
- `/tmp/pku_announcements.json`：`course`、`title`、`id`
- `/tmp/pku_course_contents.json`：`course`、`kind`、`title`、`attachment_count`、`ccid`

课程集合取上述三类数据与可选选课结果的并集。不要只使用 `pku3b s -d major show`，因为它可能遗漏“未选上但教学网已有内容”的课程。

### 5. 并行处理课程

为主 skill 提供 agent 配置，由主 skill 统一创建：

```python
agent_configs = []
for course in courses:
    agent_configs.append({
        "name": f"{course}-agent",
        "task": f"""
你是课程 "{course}" 的专属 agent。

工作目录：{base_dir}/{course}/

任务：
1. 创建目录：作业/、通知/、资料/
2. 读取 /tmp/pku_assignments.json，筛选 course="{course}" 的作业
3. 读取 /tmp/pku_announcements.json 和 /tmp/pku_course_contents.json，筛选 course="{course}" 的公告和课程内容
4. 下载作业附件（如有）：pku3b a download <ID> -d {course}/作业/
5. 对 `kind="File"` 或 `attachment_count>0` 的课程内容，使用完整复合 ID 下载：
   pku3b cc down <COURSE_ID:CONTENT_ID> -o {course}/资料/ --course-title "{course}"
   需要保存条目描述时再添加：--output-desc <安全文件名>.txt
6. 对每条公告运行 pku3b ann show <ID>，保存详情到 {course}/通知/
7. 生成 {course}/通知摘要.md：
   - 统计：总作业数、待交、已完成、逾期
   - 待完成作业（🔴紧急<1天 🟡临期<7天 🟢正常）
   - 逾期列表、已完成列表
   - 公告列表和详情文件
   - 课程内容与下载文件列表
   - 任何因 OTP 或命令错误未完成的同步项

返回：找到X个作业、Y条公告、Z项课程内容，下载N个文件，另有M项待认证/失败
"""
    })

# 由主 skill 根据 RUNTIME 环境统一创建 agents
# create_agents(agent_configs)
```

### 6. 生成汇总报告

```python
from datetime import datetime

report = f"""# PKU 课程通知汇总
生成时间：{datetime.now():%Y-%m-%d %H:%M}

## 统计
- 课程数：{len(courses)}
- 总作业：{len(assignments)}
- 待交：{sum(1 for a in assignments if '已完成' not in a['status'])}
- 公告：{len(announcements)}
- 课程内容：{len(course_contents)}
- 同步完整性：{sync_status}

*由 AutoPku 自动生成*
"""

with open(f'{base_dir}/通知摘要汇总.md', 'w') as f:
    f.write(report)
```

## 输出

- `test/通知摘要汇总.md`
- `test/{course}/通知摘要.md`
- `test/{course}/作业/`（附件）
- `test/{course}/通知/`（公告详情）
- `test/{course}/资料/`（课程内容文件和附件）
