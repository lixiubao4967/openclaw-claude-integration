# OpenClaw + Claude Code：长期自动化 AI 写代码系统

## 核心理念

三者不是竞品，是不同层级的产品，应各归其位：

| 层级 | 产品 | 角色 | 核心能力 |
|------|------|------|----------|
| **执行层** | Claude Code | 最强 coding worker | 读 repo、改代码、跑测试、修失败、产出 PR |
| **底盘层** | OpenManus | 开源通用代理框架 | 研究/实验/快速原型，适合学术探索 |
| **编排层** | OpenClaw | orchestrator + control plane | 任务接入、调度、状态管理、长期运行、多渠道通知 |

**一句话总结：Claude Code 是"工人"，OpenManus 是"底盘"，OpenClaw 是"组织系统"。**

---

## 架构设计

### 端到端流水线

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Telegram   │    │   OpenClaw   │    │  Claude Code │    │    CI/CD     │
│   下发需求    │ ──→│   编排调度    │ ──→│ 写码+测试    │ ──→│  自动部署    │
│              │    │              │    │ +commit+push │    │              │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
     你发消息          拆任务/派发          执行编码            检测到 push
                      状态管理             跑测试             构建 + 部署
                      失败通知             提交推送            发布上线
```

**完整流程：**

```
你在 Telegram 发消息："给用户列表页加个搜索功能"
  │
  ▼
OpenClaw（编排层）
  ├── 理解需求，拆分任务
  ├── 调用 claude_code_start 插件工具
  └── 管理任务状态，失败时通知你
  │
  ▼
Claude Code（执行层，容器内运行）
  ├── 读取 repo 代码
  ├── 实现功能
  ├── 运行 npm test 确认通过
  ├── git commit + git push
  └── 🔒 凭据通过容器挂载 ~/.claude/ 获取，编排层不接触
  │
  ▼
CI/CD（验收层）
  ├── 检测到代码 push
  ├── 运行 lint / typecheck / test / build
  ├── 通过 → 自动部署
  └── 失败 → 通知（可触发 OpenClaw 回修）
```

### 凭据隔离（借鉴 n8n 模式）

参考 [awesome-openclaw-usecases](https://github.com/hesamsheikh/awesome-openclaw-usecases) 中 n8n Workflow Orchestration 案例：**凭据不进 agent，每个任务独立隔离**。

- OpenClaw 不持有 Claude 凭据，只负责编排
- Claude Code 的 Team 账号凭据仅通过容器挂载传递
- Git 推送凭据（SSH key / token）在 VM 内管理，不暴露给编排层

---

## 4 个核心判断

1. **Claude Code** 优势是执行深度，不是长期组织 — 最强代码工人
2. **OpenManus** 优势是开源实验场，不是生产控制面 — 适合研究通用代理
3. **OpenClaw** 优势是把 agent 长期组织起来，不是也能写代码 — 外层编排、持久运行、渠道接入
4. 能跑长任务的系统，核心不是更聪明，而是**更会推进** — 每轮结束知道下一步是什么

---

## 成本策略

使用 Claude **Team 包年包月账号**作为 Claude Code 的认证方式：

| 对比项 | API Key 按量付费 | Team 包月账号 |
|--------|-----------------|--------------|
| 计费方式 | 按 token 消耗 | 固定月费 |
| 成本可控性 | 难预估，任务多时可能爆 | 固定，不受任务量影响 |
| 适合场景 | 偶尔调用 | 长期高频自动化 |
| 配置方式 | 设置 ANTHROPIC_API_KEY | OAuth 登录，凭据存 ~/.claude/.credentials.json |

**核心优势**：OpenClaw 只负责轻量编排（几乎不消耗 token），重编码任务全部走 Claude Code + Team 额度，成本固定可控。

---

## 环境搭建：OrbStack 容器部署（详细步骤）

整个系统运行在 OrbStack 的容器/虚拟机中，与宿主机隔离。

### 第一步：准备 OrbStack 环境

```bash
# 确认 OrbStack 已安装并运行
orb version

# 创建一个 Ubuntu 虚拟机用于部署（推荐用 VM 而非纯容器，方便安装 daemon）
orb create ubuntu openclaw-worker

# 进入虚拟机
orb shell openclaw-worker
```

> 如果已有 OrbStack 中的 Linux 环境，跳过创建步骤，直接 `orb shell <name>` 进入即可。

### 第二步：在 VM 中安装基础依赖

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装必要工具
sudo apt install -y curl git build-essential

# 安装 Node.js（OpenClaw 插件需要）
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# 安装 Docker（OrbStack VM 中用 Docker 即可）
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker

# 验证
node --version   # 应 >= 20.x
docker --version
git --version
```

### 第三步：安装 OpenClaw

```bash
# 安装 OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash

# 初始化并启动 daemon（后台常驻进程）
openclaw onboard --install-daemon

# 验证安装
openclaw status
# 应看到 daemon 运行中的状态
```

### 第四步：安装 Claude Code 并登录 Team 账号

```bash
# 安装 Claude Code（官方推荐方式）
curl -fsSL https://claude.ai/install.sh | bash

# ⚠️ 安装后可能提示 ~/.local/bin 不在 PATH 中，需要手动添加：
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# 验证安装
claude --version

# 首次启动，进行 OAuth 登录
claude

# 登录流程：
# 1. 终端会显示一个 URL，复制到浏览器打开
# 2. 用公司 Team 账号登录
# 3. 授权后回到终端，会自动完成认证
# 4. 凭据自动保存到 ~/.claude/.credentials.json

# 验证登录状态
claude --version
claude "echo hello"  # 简单测试，确认能正常响应
```

> **重要**：确保用 Team 账号登录，不要用个人账号。Team 账号的额度是包月的，不会产生额外费用。

### 第五步：安装容器化 Claude Code 插件

```bash
# 安装插件
openclaw plugins install @13rac1/openclaw-plugin-claude-code

# 拉取 Claude Code 容器镜像
docker pull ghcr.io/13rac1/openclaw-claude-code:latest

# 验证镜像
docker images | grep openclaw-claude-code
```

### 第六步：配置插件

编辑 OpenClaw 配置文件：

```bash
# 配置文件位置
# 如果文件不存在，创建它
mkdir -p ~/.openclaw
nano ~/.openclaw/openclaw.json
```

在已有配置的 `plugins` 部分中，添加 `allow` 白名单和 `claude-code` 的 `config` 块：

```json
{
  "plugins": {
    "allow": ["claude-code", "telegram"],
    "entries": {
      "claude-code": {
        "enabled": true,
        "config": {
          "image": "ghcr.io/13rac1/openclaw-claude-code:latest",
          "runtime": "docker",
          "startupTimeout": 30,
          "idleTimeout": 300,
          "memory": "1g",
          "cpus": "2.0",
          "network": "bridge",
          "sessionsDir": "~/.openclaw/claude-sessions",
          "workspacesDir": "~/.openclaw/workspaces"
        }
      }
    }
  }
}
```

**配置说明：**

| 字段 | 说明 | 建议值 |
|------|------|--------|
| `runtime` | 容器运行时，OrbStack VM 中用 docker | `"docker"` |
| `startupTimeout` | 容器启动超时（秒） | `30` |
| `idleTimeout` | 空闲会话自动清理时间（秒） | `300`（5 分钟） |
| `memory` | 单个容器内存上限 | `"1g"`（复杂任务可调到 `"2g"`） |
| `cpus` | 单个容器 CPU 核心数 | `"2.0"` |
| `network` | 网络模式，bridge 允许容器访问外网（拉依赖等） | `"bridge"` |
| `sessionsDir` | 会话持久化目录 | `"~/.openclaw/claude-sessions"` |
| `workspacesDir` | 工作区目录，存放 clone 的 repo | `"~/.openclaw/workspaces"` |

### 第七步：配置 webhook 通知（可选，v0.5 再做）

```json
{
  "hooks": {
    "enabled": true,
    "token": "your-secret-token"
  }
}
```

### 第八步：验证完整链路

```bash
# 1. 确认 OpenClaw daemon 运行中
openclaw status

# 2. 确认插件已加载
openclaw plugins list
# 应看到 claude-code 插件状态为 enabled

# 3. 确认 Claude Code 认证有效
ls ~/.claude/.credentials.json
# 文件应存在

# 4. 手动测试一次任务下发（见下方 v0.1 实施步骤）
```

---

## 渐进式落地路线（详细实施）

### v0.1 — Telegram 下发任务，Claude Code 写码（已完成）

**目标**：验证 Telegram → OpenClaw → Claude Code → 产出代码 链路能通。

**准备测试 repo：**

```bash
mkdir -p ~/.openclaw/workspaces/test-project
cd ~/.openclaw/workspaces/test-project
git init
npm init -y

# ⚠️ VM 全新环境需要配置 git 身份（否则无法 commit）
git config --global user.email "you@example.com"
git config --global user.name "Your Name"

# 创建待实现的文件和测试
cat > index.js << 'EOF'
// TODO: 实现一个函数，接收数组，返回去重后的数组
function uniqueArray(arr) {
  // 请实现
}

module.exports = { uniqueArray };
EOF

cat > index.test.js << 'EOF'
const { uniqueArray } = require('./index');

test('去重基本数组', () => {
  expect(uniqueArray([1, 2, 2, 3])).toEqual([1, 2, 3]);
});

test('空数组', () => {
  expect(uniqueArray([])).toEqual([]);
});

test('无重复', () => {
  expect(uniqueArray([1, 2, 3])).toEqual([1, 2, 3]);
});
EOF

npm install --save-dev jest
# npx json 首次运行会提示安装，输入 y 确认
npx json -I -f package.json -e 'this.scripts.test="jest"'
git add -A && git commit -m "init: test project for openclaw integration"
```

**下发任务：**

> **重要**：OpenClaw 没有 `openclaw run` 命令。插件工具（如 `claude_code_start`）是由 OpenClaw agent 内部调用的，你只需向 agent 发送自然语言消息。

在 Telegram 中给 OpenClaw bot 发消息：

> 请使用 claude_code_start 工具，在 /home/xiubao_li/.openclaw/workspaces/test-project 中实现 index.js 的 uniqueArray 函数，使 npm test 通过。

也可以通过 CLI 发：

```bash
openclaw agent --agent main --message "请使用 claude_code_start 工具，在 /home/xiubao_li/.openclaw/workspaces/test-project 中实现 index.js 的 uniqueArray 函数，使 npm test 通过。"
```

**v0.1 通过标准**：Claude Code 成功修改代码，`npm test` 全部通过。**已验证通过。**

---

### v0.2 — Claude Code 自动 commit + push

**目标**：Claude Code 完成编码后自动提交并推送代码，触发 CI/CD。

**前置：给测试项目配置 Git 远程仓库**

```bash
cd ~/.openclaw/workspaces/test-project

# 在 GitHub 上创建一个测试仓库，然后关联
git remote add origin git@github.com:<your-username>/test-project.git

# 配置 SSH key（如果还没有）
ssh-keygen -t ed25519 -C "openclaw-worker"
cat ~/.ssh/id_ed25519.pub
# 将公钥添加到 GitHub（Settings → SSH keys）

# 推送初始代码
git push -u origin master
```

**下发任务时要求 commit + push：**

在 Telegram 中发：

> 在 /home/xiubao_li/.openclaw/workspaces/test-project 中添加一个 flattenArray 函数，将嵌套数组展平为一维数组。添加对应的测试。完成后运行 npm test 确认通过，然后 git commit 并 git push。

Claude Code 会自动：
1. 实现 flattenArray 函数
2. 写测试
3. 运行 `npm test`
4. `git add` + `git commit` + `git push`

**v0.2 通过标准**：Claude Code push 后，GitHub 仓库中能看到新的 commit。

---

### v0.3 — 接入 CI/CD 自动部署

**目标**：Claude Code push 代码后，CI/CD 自动运行测试和部署。

**GitHub Actions 示例 `.github/workflows/ci.yml`：**

```yaml
name: CI/CD

on:
  push:
    branches: [master, main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm install
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: success()
    steps:
      - uses: actions/checkout@v4
      # 根据你的部署方式配置，例如：
      # - 部署到服务器：rsync / ssh
      # - 部署到云平台：AWS / Vercel / Cloudflare
      # - 构建 Docker 镜像：docker build + push
      - run: echo "部署步骤，根据实际项目配置"
```

**完整链路验证：**

```
Telegram 发任务
  → OpenClaw 派发
    → Claude Code 写码 + npm test + commit + push
      → GitHub Actions 检测到 push
        → 运行 lint + test
          → 通过 → 自动部署
          → 失败 → 通知（下一步 v0.4 处理）
```

**v0.3 通过标准**：Claude Code push 后，GitHub Actions 自动运行并部署成功。

---

### v0.4 — CI 失败时自动回修

**目标**：CI/CD 失败时通知 OpenClaw，OpenClaw 让 Claude Code 回修，最多 3 轮。

**方案 A：通过 GitHub Actions 回调 OpenClaw**

在 CI 失败时通过 Telegram bot 通知，或直接调用 OpenClaw Gateway API 触发回修：

```yaml
# 在 ci.yml 的 test job 中添加失败处理
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm install
      - run: npm test
      - name: 通知 CI 失败
        if: failure()
        run: |
          curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
            -H 'Content-Type: application/json' \
            -d '{
              "chat_id": "${{ secrets.TELEGRAM_CHAT_ID }}",
              "text": "CI 失败！请检查并修复。\n仓库: ${{ github.repository }}\n提交: ${{ github.sha }}\n日志: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }'
        env:
          TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
```

**方案 B：OpenClaw cron 轮询 CI 状态**

```bash
# 配置 OpenClaw 定时检查最近的 CI 状态
openclaw cron add --schedule "*/5 * * * *" \
  --agent main \
  --message "检查 GitHub Actions 最近的运行状态，如果有失败的，分析失败原因并使用 claude_code_start 修复代码。"
```

**方案 C：手动通知回修（最简单）**

CI 失败后，你在 Telegram 中告诉 bot：

> CI 失败了，错误信息是 [粘贴错误]。请在 /home/xiubao_li/.openclaw/workspaces/test-project 中修复并重新 push。

**v0.4 通过标准**：CI 失败后，通过某种方式触发 Claude Code 自动修复并重新 push，CI 最终通过。

---

### v0.5 — 实际项目接入

**目标**：将一个真实项目接入这套流水线。

**步骤：**

```bash
# 1. Clone 你的实际项目到 workspaces
cd ~/.openclaw/workspaces
git clone git@github.com:<your-org>/<your-project>.git
cd <your-project>

# 2. 确认项目有测试和 CI 配置
cat package.json | grep -A5 '"scripts"'
ls .github/workflows/

# 3. 通过 Telegram 下发真实任务
```

在 Telegram 中发实际需求：

> 在 /home/xiubao_li/.openclaw/workspaces/<your-project> 中，给用户列表页添加搜索功能。搜索应该支持按用户名和邮箱模糊匹配。完成后运行测试，确认通过后 commit 并 push。

**v0.5 通过标准**：真实项目的功能需求通过 Telegram → OpenClaw → Claude Code → CI/CD 完整流水线交付。

---

## 关键原则

- **各层各司其职** — OpenClaw 编排调度，Claude Code 写码测试提交，CI/CD 验收部署
- **CI/CD 是验收层** — 不需要额外验收脚本，CI 通过 = 可部署，CI 失败 = 需回修
- **凭据不进 agent** — Claude 凭据通过容器挂载传递，Git SSH key 在 VM 内管理，编排层不接触
- **Telegram 是控制面板** — 下发需求、接收通知、触发回修，全在 Telegram 里完成
- **先通后优** — v0.1 跑通链路，v0.2 加 push，v0.3 接 CI/CD，逐步完善

---

## 安全加固（基于 OpenClaw 已知安全事件）

> 以下内容参考 [The Hidden Danger of OpenClaw](https://github.com/affaan-m/everything-claude-code/blob/main/the-openclaw-guide.md)（Everything Claude Code 系列 Part 3）中记录的真实安全事件和行业安全专家建议。

### 为什么必须重视安全

OpenClaw 生态已发生多起严重安全事件：

| 事件 | 时间 | 影响 |
|------|------|------|
| **Moltbook 数据库泄露** | 2026.1.31 | 149 万条记录暴露，含 3.2 万+ AI agent API key，根因：Supabase 未配 RLS |
| **ClawdHub 技能市场污染** | 2026.1 | ~20% 技能（800+）含恶意代码（AMOS 恶意软件、反向 shell、凭据窃取） |
| **CVE-2026-25253（CVSS 8.8）** | 2026.1 | 控制面板 WebSocket 自动泄露认证 token，4.2 万暴露实例 |

CrowdStrike 评价：OpenClaw 的架构"将 prompt injection 转变为全面入侵的推手"。

### 加固清单（按优先级）

#### 1. 版本与补丁

```bash
# 确保 OpenClaw >= 2026.1.29（修复 CVE-2026-25253）
openclaw --version

# 如果版本过旧，更新
curl -fsSL https://openclaw.ai/install.sh | bash
```

#### 2. 禁用不必要的渠道接入

OpenClaw 的每个接入渠道（Telegram、Discord、X、WhatsApp、邮件、浏览器）都是一个攻击面。**只开启你实际使用的渠道**。

```bash
# 查看当前启用的渠道
openclaw config list

# 禁用不需要的渠道（示例）
openclaw config set channels.telegram.enabled false
openclaw config set channels.discord.enabled false
openclaw config set channels.x.enabled false
openclaw config set channels.whatsapp.enabled false
openclaw config set channels.email.enabled false
```

**原则：如果只通过 CLI/API 下发任务，不需要开启任何外部渠道。**

#### 3. 不使用未审计的 ClawdHub 技能

ClawdHub 市场约 20% 的技能含有恶意代码，包括：
- 隐藏的 prompt injection 指令
- 凭据外泄载荷
- 反向 shell

**规则：**
- 不安装 ClawdHub 上的第三方技能，除非逐行审计过源码
- 只使用本项目中明确列出的插件（`openclaw-plugin-claude-code`）
- 如果需要新技能，从 GitHub 源码安装，不从市场安装

```bash
# 查看已安装的技能/插件
openclaw plugins list

# 移除可疑的技能
openclaw plugins uninstall <plugin-name>
```

#### 4. 容器级隔离（已在架构中实现）

我们的方案使用 `openclaw-plugin-claude-code` 容器化插件，天然具备以下安全特性：

```
安全特性                          状态
─────────────────────────────────────
容器隔离（每个任务独立容器）          ✅ 已配置
--cap-drop ALL（丢弃所有 Linux 能力） ✅ 插件默认
内存/CPU/PID 限制                    ✅ openclaw.json 中配置
tmpfs /tmp（nosuid）                 ✅ 插件默认
rootless 运行（如果用 Podman）       ⚠️ OrbStack 用 Docker，建议额外加固
```

**额外加固 Docker 运行时：**

```bash
# 创建专用用户运行 OpenClaw（不用 root）
sudo useradd -m -s /bin/bash openclaw-runner
sudo usermod -aG docker openclaw-runner

# 切换到专用用户
su - openclaw-runner

# 后续所有操作都在 openclaw-runner 用户下进行
```

#### 5. 凭据保护

```bash
# 确保凭据文件权限正确（仅 owner 可读）
chmod 600 ~/.claude/.credentials.json
chmod 700 ~/.claude/

# 确认凭据不在 git 中
echo ".credentials.json" >> ~/.claude/.gitignore

# 检查环境变量中没有硬编码的 API key
env | grep -i "api\|key\|token\|secret"
# 如果有不需要的，从 ~/.bashrc 或 ~/.zshrc 中移除
```

#### 6. 网络隔离

```bash
# 如果 Claude Code 容器不需要访问外网（已预装依赖），使用 none 网络
# 在 openclaw.json 中修改：
# "network": "none"

# 如果需要访问外网（拉依赖等），保持 bridge 但限制出站
# 可通过 iptables 限制容器只能访问必要的域名：
# - api.anthropic.com（Claude API）
# - registry.npmjs.org（npm 依赖）
# - github.com（git 操作）
```

#### 7. 审计日志

```bash
# 创建日志目录
mkdir -p ~/.openclaw/logs

# 在 openclaw.json 中开启审计日志
# 添加到配置：
# "logging": {
#   "level": "info",
#   "file": "~/.openclaw/logs/audit.log",
#   "rotate": true,
#   "maxSize": "50m"
# }

# 定期检查日志中的异常行为
tail -f ~/.openclaw/logs/audit.log

# 关注的异常模式：
# - 非预期的外部网络请求
# - 尝试读取 ~/.claude/.credentials.json 以外的凭据文件
# - 容器逃逸相关的 syscall 失败
```

#### 8. 定期安全检查脚本

创建 `scripts/security_check.sh`：

```bash
#!/bin/bash
# security_check.sh - 定期运行的安全检查
set -euo pipefail

echo "========== OpenClaw 安全检查 =========="

# 1. 版本检查
echo "[1/6] 版本检查..."
OPENCLAW_VERSION=$(openclaw --version 2>/dev/null || echo "未安装")
echo "  OpenClaw: $OPENCLAW_VERSION"
echo "  ⚠️ 确保 >= 2026.1.29（CVE-2026-25253 修复版本）"

# 2. 凭据文件权限
echo "[2/6] 凭据权限检查..."
if [ -f ~/.claude/.credentials.json ]; then
  PERMS=$(stat -c '%a' ~/.claude/.credentials.json 2>/dev/null || stat -f '%Lp' ~/.claude/.credentials.json)
  if [ "$PERMS" = "600" ]; then
    echo "  ✅ ~/.claude/.credentials.json 权限正确 (600)"
  else
    echo "  ❌ ~/.claude/.credentials.json 权限为 $PERMS，应为 600"
    echo "  修复: chmod 600 ~/.claude/.credentials.json"
  fi
else
  echo "  ⚠️ 凭据文件不存在"
fi

# 3. 已安装插件审计
echo "[3/6] 插件审计..."
openclaw plugins list 2>/dev/null || echo "  无法获取插件列表"
echo "  ⚠️ 确认所有插件来源可信，不使用未审计的 ClawdHub 技能"

# 4. 启用的渠道检查
echo "[4/6] 渠道检查..."
echo "  ⚠️ 手动确认：只开启了实际使用的渠道"
echo "  运行 openclaw config list 查看"

# 5. 容器运行状态
echo "[5/6] 容器检查..."
RUNNING_CONTAINERS=$(docker ps --filter "ancestor=ghcr.io/13rac1/openclaw-claude-code" --format "{{.ID}} {{.Status}}" 2>/dev/null)
if [ -n "$RUNNING_CONTAINERS" ]; then
  echo "  运行中的 Claude Code 容器:"
  echo "$RUNNING_CONTAINERS" | while read line; do echo "    $line"; done
else
  echo "  ✅ 没有运行中的 Claude Code 容器"
fi

# 6. 环境变量泄露检查
echo "[6/6] 环境变量检查..."
LEAKED=$(env | grep -iE "api.?key|secret|token|password" | grep -v "^_=" || true)
if [ -n "$LEAKED" ]; then
  echo "  ⚠️ 发现可能敏感的环境变量:"
  echo "$LEAKED" | while read line; do
    VAR_NAME=$(echo "$line" | cut -d= -f1)
    echo "    $VAR_NAME=****"
  done
else
  echo "  ✅ 未发现明文敏感环境变量"
fi

echo "========== 检查完毕 =========="
```

```bash
chmod +x scripts/security_check.sh

# 建议：部署后立即运行一次
./scripts/security_check.sh

# 建议：加入 cron 每天自动检查
# crontab -e
# 0 9 * * * /path/to/scripts/security_check.sh >> ~/.openclaw/logs/security.log 2>&1
```

### 安全架构对比

我们的方案 vs 原文推荐的 "MiniClaw" 理念：

| 安全维度 | 原始 OpenClaw | MiniClaw（原文推荐） | 本项目方案 |
|----------|-------------|-------------------|-----------|
| 接入点 | 7+ 渠道 | 仅 SSH | 仅 CLI/API（按需开渠道） |
| 执行环境 | 宿主机，权限宽泛 | 容器化，受限 | OrbStack VM + 容器双层隔离 |
| 技能来源 | ClawdHub（未审计） | 手动审计，仅本地 | 仅用已审计插件，不用 ClawdHub |
| 凭据管理 | 可能分散 | 集中管理 | 容器挂载，编排层不接触 |
| 网络暴露 | 多端口多服务 | SSH only（Tailscale） | OrbStack 内网，按需开放 |
| 爆炸半径 | agent 可访问的一切 | 沙盒限于项目目录 | 容器 --cap-drop ALL + 资源限制 |

**本项目的额外优势**：OrbStack VM 本身就是一层隔离，即使容器被突破，攻击者仍在 VM 内，无法直接触及宿主 macOS。

---

## 项目文件结构

```
openclaw-claude-integration/
├── README.md                        # 本文件（架构设计 + 搭建指南 + 落地路线）
├── scripts/
│   └── security_check.sh           # 安全检查脚本（建议每日 cron 运行）
├── .github/
│   └── workflows/
│       └── ci.yml                   # CI/CD 配置（v0.3 阶段创建）
└── .openclaw/
    └── openclaw.json                # OpenClaw 插件配置（在 VM 中）
```

> 注意：这套方案不需要额外的编排脚本或任务队列文件。OpenClaw agent 本身就是编排器，通过 Telegram 交互即可完成所有调度。CI/CD 就是验收层。

---

## 参考资源

- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw 官网](https://openclaw.ai/)
- [OpenClaw Claude Code Skill 插件](https://github.com/Enderfga/openclaw-claude-code-skill)
- [OpenClaw Claude Code 容器化插件](https://github.com/13rac1/openclaw-plugin-claude-code)
- [OpenClaw 完整指南 - Milvus Blog](https://milvus.io/blog/openclaw-formerly-clawdbot-moltbot-explained-a-complete-guide-to-the-autonomous-ai-agent.md)
- [The Hidden Danger of OpenClaw（安全风险深度分析）](https://github.com/affaan-m/everything-claude-code/blob/main/the-openclaw-guide.md) — 本项目安全加固章节的核心参考
- [Setup OpenClaw with Claude & Gemini](https://vertu.com/ai-tools/the-ultimate-guide-setting-up-openclaw-with-claude-code-and-gemini-3-pro/)
- [Awesome OpenClaw Use Cases](https://github.com/hesamsheikh/awesome-openclaw-usecases) — n8n 凭据隔离模式参考
