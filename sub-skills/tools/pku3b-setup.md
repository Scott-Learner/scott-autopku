---
name: autopku-tool-pku3b-setup
description: PKU教学网工具 pku3b 的安装、配置和登录
---

# pku3b 工具配置

## 安装检查

```bash
PKU3B="$(command -v pku3b || true)"
test -n "$PKU3B" || echo "NOT_FOUND"
test -z "$PKU3B" || "$PKU3B" --version
```

## 版本要求

- 当前验证版本：`0.16.1`；最低使用 `0.16.1`
- 每次安装前查看 [sshwy/pku3b releases](https://github.com/sshwy/pku3b/releases) 的最新稳定版
- 不要继续安装旧版 `0.11.0`：`0.12.1` 才包含 macOS Blackboard TLS 修复，`0.14.0` 才支持课程内容列表/下载，`0.16.1` 修复了选课登录重定向

## 安装与升级

### macOS（推荐 Homebrew）

```bash
# 未安装
brew install sshwy/tools/pku3b

# 已通过 Homebrew 安装旧版
brew update
brew upgrade pku3b

pku3b --version
```

如果 `brew upgrade pku3b` 报告公式未安装，改用 `brew install sshwy/tools/pku3b`。安装后重新解析实际路径：

```bash
PKU3B="$(command -v pku3b || true)"
test -n "$PKU3B" || echo "NOT_FOUND"
test -z "$PKU3B" || "$PKU3B" --version
```

### macOS Apple Silicon 直接下载备选

只有 Homebrew 不可用时才使用直接下载。执行时先确认官方最新稳定版本；以下是当前验证过的 `0.16.1` 示例：

```bash
cd /tmp
curl -LO "https://github.com/sshwy/pku3b/releases/download/0.16.1/pku3b-0.16.1-aarch64-apple-darwin.tar.gz"
echo "bd7a4329c5ac688d8392e21728473b43f6ac40b72907f613de5618ab15919ac7  pku3b-0.16.1-aarch64-apple-darwin.tar.gz" | shasum -a 256 -c -
tar -xzf pku3b-0.16.1-aarch64-apple-darwin.tar.gz
chmod +x pku3b-0.16.1-aarch64-apple-darwin/pku3b
mkdir -p "$HOME/.local/bin"
install -m 755 pku3b-0.16.1-aarch64-apple-darwin/pku3b "$HOME/.local/bin/pku3b"
"$HOME/.local/bin/pku3b" --version
```

### 其他平台
从 [sshwy/pku3b releases](https://github.com/sshwy/pku3b/releases) 下载对应版本。

## 登录

`pku3b init` 只需执行一次，凭证持久化到 `~/Library/Application Support/org.sshwy.pku3b/cfg.toml`。

### 1. 检查是否已登录

```bash
pku3b a ls
```

正常返回作业列表 → Blackboard 登录可用，跳过步骤 2-3。命令失败时保留并读取原始错误；只有错误明确表明凭证缺失或账号密码无效时才执行步骤 2。网络、TLS、服务器和非 TTY 错误都不能改写成 `NOT_LOGGED_IN`。

### 2. 交互式登录（仅在未登录时）

```bash
pku3b init
```

`pku3b init` 检查 TTY，直接管道输入会报错 "input device is not a TTY"。使用当前运行环境提供的 PTY/交互终端；不要假定一定是 Kimi Code。需要账号、密码或验证码时让用户直接在该终端输入，不要生成 expect 脚本或明文凭证文件。

### 3. 验证

```bash
# 验证登录成功
pku3b a ls
```

## 手机令牌 OTP

`pku3b ct`、`pku3b cc` 和 `pku3b grades` 通过门户加载课程时可能要求手机令牌 OTP。此时：

1. 不要运行 `pku3b init`；作业命令成功已证明 Blackboard 登录有效。
2. 在 PTY/交互终端中重跑原命令，让用户直接输入 OTP。
3. 不要把 OTP 写入 `--otp-code` 命令参数、聊天、日志、脚本或文件；该参数虽然存在，但会扩大敏感信息暴露面。
4. 用户未完成 OTP 时，将依赖门户的课表、课件或成绩标为待认证，保留已经成功取得的作业和公告。

## 常用命令

```bash
# 作业
pku3b a -f ls                       # 强制刷新当前学期未完成作业
pku3b a -f ls -a                    # 强制刷新当前学期全部作业（含已完成）
pku3b a ls --all-term               # 所有学期作业（含已完成）
pku3b a download <ID> -d <dir>      # 下载附件
pku3b a submit <ID> <file>          # 提交作业

# 公告
pku3b ann -f ls                     # 强制刷新当前学期公告
pku3b ann show <ID>                 # 查看公告详情

# 课程内容/课件
pku3b cc -f ls
pku3b cc ls --course-title "课程名"
pku3b cc down <COURSE_ID:CONTENT_ID> -o <dir> --course-title "课程名"
# 如需另存条目描述，再追加：--output-desc <file>

# 课表与成绩（可能要求 OTP）
pku3b ct -f -r                      # 获取课表原始 JSON
pku3b grades -f                     # 当前学期成绩

# 选课（可选）
pku3b s -d major show               # 主修课程
```

## 踩坑记录

- `pku3b init` 需要交互式输入，直接管道输入不工作；使用当前运行环境提供的 PTY/交互终端
- `pku3b auth status/login` 命令不存在，正确命令是 `pku3b init`
- `pku3b s -d major show` 可能在某些账号返回 `302 Found`，作为可选步骤处理
- `The input device is not a TTY` 是交互终端问题，不等于凭证失效
- `pku3b 0.11.0` 在当前 macOS/教学网上可能出现 TLS `HandshakeFailure`；升级到 `0.16.1+`
- `pku3b cc ls` 的每个条目以 `course_id:content_id` 结尾；下载时必须传入完整复合 ID
