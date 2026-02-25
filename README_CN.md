# openclaw-ops `v1.1.0`

[English](README.md) | [中文](README_CN.md)

一个教 AI Agent 如何运维 [OpenClaw](https://openclaw.ai) Gateway 的技能。

适用于**任何有 shell 访问权限的 Agent** — Claude Code、Codex、OpenClaw、Pi 或任何运行在 OpenClaw Gateway 同一台机器上的 AI Agent。

支持 **Linux (systemd)** 和 **macOS (launchd)**。

## 架构

```
┌─────────────────────────────────────────────────┐
│                 服务器 / Mac                      │
│                                                  │
│  ┌──────────────┐       ┌────────────────────┐  │
│  │ 主 OpenClaw   │◄─────│  Rescue Agent      │  │
│  │   Gateway     │ 修复 │  (Claude Code /    │  │
│  │              │       │   备用 OC /        │  │
│  │  • Agents    │       │   任何 AI Agent)   │  │
│  │  • Channels  │       │                    │  │
│  │  • Sessions  │       │  + openclaw-ops    │  │
│  │  • Cron jobs │       │    技能已安装      │  │
│  └──────┬───────┘       └────────┬───────────┘  │
│         │                        │               │
│         │ systemd / launchd      │ shell 访问    │
│         │                        │               │
└─────────┼────────────────────────┼───────────────┘
          │                        │
          ▼                        ▼
   通过 Discord/Telegram          当主 Agent 挂了
   等渠道服务用户                  你连接 Rescue Agent
```

**主 OpenClaw Gateway** — 你的主要 AI Agent 系统，处理日常操作：聊天频道、定时任务、会话等。

**Rescue Agent** — 一个独立的 Agent（Claude Code、备用 OpenClaw 实例或其他有 shell 访问权限的 AI Agent），安装了本技能。和主 Gateway 运行在同一台机器上。它的唯一职责：主 Gateway 出问题时修复它，以及执行运维健康检查。

**本技能** — 教会 Rescue Agent 该跑什么命令、如何解读输出、按什么步骤诊断和修复。

## 两个核心场景

### 🔴 救援：主 Gateway 挂了

主 OpenClaw 崩溃、配置损坏或无法启动。你连接到 Rescue Agent 让它修复。

```
你: "OpenClaw 挂了，帮我看看"

Rescue Agent: 检查 systemctl 状态 → 读取崩溃日志 → 发现 ENOMEM →
回复 "内存不足，Node 进程被杀。内存使用率 94%。
需要我释放一些内存然后重启吗？"
```

### 🟢 健康检查：主 Gateway 运行中

主 OpenClaw 正常运行，你想确认运行状态、升级或清理。

```
你: "检查一下 OpenClaw 的运行状况"

Rescue Agent: 运行 openclaw doctor → 报告状态、孤立文件数量、
频道连接情况、磁盘使用 → 提供修复建议
```

## 如何远程连接 Rescue Agent

主 OpenClaw 挂了之后，你无法通过 Discord/Telegram 和它对话。你需要其他方式连接服务器上的 Rescue Agent。

### 方案 1：通过 SSH 连接 Claude Code（推荐）

SSH 登录服务器，直接运行 Claude Code：

```bash
ssh user@your-server
claude  # 启动 Claude Code，技能自动可用
```

移动端可以用任何 SSH 客户端（Termius、Blink 等）。

### 方案 2：Claude Code Remote（VS Code）

使用 VS Code Remote SSH：

1. VS Code → Remote-SSH → 连接到服务器
2. 打开终端 → `claude`
3. Rescue Agent 拥有 shell 访问和本技能

### 方案 3：备用 OpenClaw 实例

运行第二个 OpenClaw 实例作为 Rescue Agent，使用不同的频道（比如主 Agent 用 Discord，Rescue 用 Telegram）：

```bash
openclaw daemon install --name openclaw-rescue
```

这样主 Agent 无响应时，你通过 Telegram 联系 Rescue Agent。

### 方案 4：tmux + SSH

在服务器上保持一个 tmux 会话运行 Claude Code：

```bash
# 在服务器上（一次性设置）
tmux new -s rescue
claude

# 之后从任何地方通过 SSH 连接
ssh user@your-server
tmux attach -t rescue
```

## 安装

### 在对话中让 Agent 安装

直接在对话里告诉你的 Agent：

```
> 从 https://github.com/dinstein/openclaw-ops-skill 安装 openclaw-ops 技能
```

Agent 会自动下载 `SKILL.md` 并放到正确的技能目录。

### 通过 ClawHub

```bash
clawhub install openclaw-ops
```

### 手动安装

将 `SKILL.md` 复制到你的 Agent 技能目录：

```bash
# Claude Code
mkdir -p ~/.claude/skills/openclaw-ops
cp SKILL.md ~/.claude/skills/openclaw-ops/SKILL.md

# OpenClaw
mkdir -p ~/.openclaw/workspace/skills/openclaw-ops
cp SKILL.md ~/.openclaw/workspace/skills/openclaw-ops/SKILL.md

# 其他 Agent — 放到你的 Agent 读取技能文件的目录
```

## 功能覆盖

| 模块 | 场景 | 说明 |
|------|------|------|
| 崩溃诊断 | 🔴 救援 | 读取日志、定位根因 |
| 配置修复 | 🔴 救援 | JSON 修复、Schema 校验、常见错误 |
| 服务重启 | 🔴 救援 | 修复根因后安全重启 |
| 资源检查 | 🔴 救援 | 磁盘、内存、Node.js、依赖 |
| 健康检查 | 🟢 运维 | `openclaw doctor`、服务状态 |
| 更新升级 | 🟢 运维 | 版本检查、安全升级、回滚 |
| 磁盘清理 | 🟢 运维 | 孤立 transcript、会话管理 |
| 备份 | 🟢 运维 | 配置、agents、workspace 备份 |
| Tailscale 检查 | 🟢 运维 | 反向代理验证 |

## 工作原理

这是一个**纯文档技能** — 无脚本、无外部依赖、无框架绑定。安装到任何能读取 Markdown 并执行 shell 命令的 Agent 中即可。它教会 Agent：

1. **检查什么** — 每种场景对应的正确命令
2. **如何解读** — 输出含义和常见错误模式
3. **怎么修复** — 带安全护栏的逐步修复流程
4. **如何验证** — 每个操作后的确认步骤

## 安全规则

- 修改前必须先查日志
- 编辑配置前必须备份
- 编辑后必须校验 JSON
- 不打印敏感信息（env 文件）
- 不删除 workspace 文件，除非用户确认
- 重启后必须验证服务状态

## 许可证

MIT
