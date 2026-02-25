# openclaw-ops `v1.0.0`

一个教 AI Agent 如何运维 [OpenClaw](https://openclaw.ai) Gateway 的技能。

适用于**任何有 shell 访问权限的 Agent** — Claude Code、Codex、OpenClaw、Pi 或任何运行在 OpenClaw Gateway 同一台机器上的 AI 编程代理。

支持 **Linux (systemd)** 和 **macOS (launchd)**。

[English](README.md) | [中文](README_CN.md)

## 安装

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

| 模块 | 说明 |
|------|------|
| **状态检查** | `openclaw doctor`、服务状态、端口、进程 |
| **配置修复** | JSON 语法修复、常见配置错误、Schema 校验 |
| **重启与恢复** | 崩溃诊断、安全重启、根因分析 |
| **日志诊断** | journalctl / 日志文件模式匹配、常见错误识别 |
| **服务初始化** | 从零配置 systemd / launchd 常驻服务 |
| **更新升级** | 版本检查、升级、回滚、升级前后对比 |
| **环境检查** | Node.js、依赖完整性、端口冲突、Tailscale |
| **备份恢复** | 关键文件备份与恢复流程 |
| **会话清理** | 孤立 transcript 清理、磁盘管理 |
| **定时健康检查** | 自动化监控 cron 模板 |

## 使用示例

安装后，你可以用自然语言让 Agent 执行运维任务：

### 快速健康检查

```
> 检查一下 OpenClaw 是否正常

Agent 执行: openclaw doctor, systemctl status, 端口检查
Agent 回复: "Gateway 运行中，所有频道已连接，发现 3 个孤立 transcript — 需要我清理吗？"
```

### 诊断崩溃

```
> OpenClaw 没响应了，帮我看看怎么回事

Agent 执行: systemctl status（发现 failed），检查 journalctl 日志
Agent 回复: "Gateway 15 分钟前因 ENOMEM 崩溃，服务器内存使用率 94%。我可以在释放内存后重启 — 需要我检查是什么在占用内存吗？"
```

### 修复配置

```
> 我改了 openclaw.json 之后启动不了了

Agent 执行: JSON 校验（发现第 42 行多了个逗号），备份当前文件，修复语法
Agent 回复: "第 42 行有个多余的逗号，已修复并通过校验。需要我重启 Gateway 吗？"
```

### 新机器部署

```
> 在这台机器上把 OpenClaw 配置成常驻服务

Agent 检测操作系统（Linux/macOS），创建服务文件（systemd unit 或 launchd plist），
配置环境变量文件，启用自动启动，启动服务，验证健康状态。
```

### 安全升级

```
> 升级 OpenClaw 到最新版本

Agent 检查当前与最新版本，备份配置，执行 npm update，
重启服务，对比升级前后 doctor 输出，报告新增警告。
```

### 备份

```
> 在改动之前帮我备份 OpenClaw 配置

Agent 创建带时间戳的备份，包含 openclaw.json、env、agents/、devices/、
workspace/ 和服务配置。报告备份位置。
```

### Tailscale 代理问题

```
> 通过 Tailscale 访问不了 Dashboard

Agent 检查: tailscale status, tailscale serve status, localhost 连通性
Agent 回复: "Tailscale Serve 没有配置。需要我设置 HTTPS 代理到 localhost:18789 吗？"
```

### 定时监控

```
> 设置每 6 小时自动健康检查

Agent 创建 OpenClaw cron job 或系统 crontab，定期运行
openclaw doctor 并在发现问题时报告。
```

## 工作原理

这是一个**纯文档技能** — 无脚本、无外部依赖、无框架绑定。安装到任何能读取 Markdown 并执行 shell 命令的 Agent 中即可。它教会 Agent：

1. **检查什么** — 每种场景对应的正确命令和文件
2. **如何解读** — 输出含义和常见错误模式
3. **怎么修复** — 带安全护栏的逐步修复流程
4. **如何验证** — 每个操作后的确认步骤

## 安全规则

技能内置以下安全规则：

- 修改前必须先查日志
- 编辑配置前必须备份
- 编辑后必须校验 JSON
- 不打印敏感信息（env 文件）
- 不删除 workspace 文件，除非用户确认
- 重启后必须验证服务状态

## 版本管理

遵循 [语义化版本](https://semver.org/lang/zh-CN/)：

- **MAJOR** — 技能结构或安全规则的破坏性变更
- **MINOR** — 新模块、新平台支持、新命令
- **PATCH** — 修复、错别字、描述优化

当前版本：`1.0.0`

## 许可证

MIT
