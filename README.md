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

### 四层模型

```
┌──────────────────────────────────────────────────┐
│  第一层：任务入口（OpenClaw）                       │
│  Feishu / Slack / Discord / Issue / Cron 汇聚需求  │
├──────────────────────────────────────────────────┤
│  第二层：编排层（OpenClaw）                         │
│  拆任务、记状态、决定重试或升级人工                    │
├──────────────────────────────────────────────────┤
│  第三层：执行层（Claude Code）                      │
│  专注读 repo + 改码 + 验证 + 产出 diff              │
├──────────────────────────────────────────────────┤
│  第四层：验收与续航（独立判断）                       │
│  lint / typecheck / test，"写代码"和"判定完成"必须拆开 │
└──────────────────────────────────────────────────┘
```

### 凭据隔离模型（借鉴 n8n 模式）

参考 [awesome-openclaw-usecases](https://github.com/hesamsheikh/awesome-openclaw-usecases) 中 n8n Workflow Orchestration 案例的核心思想：**凭据不进 agent，webhook 双向通信，每个任务独立隔离**。

```
┌─────────────────────────────────────────────────┐
│  OpenClaw（编排层）                               │
│  接收需求 → 拆分任务 → 调用插件工具触发执行          │
│  ⚠️ 不持有任何 Claude 凭据                        │
├─────────────────────────────────────────────────┤
│  claude-code 插件（执行层，容器隔离）               │
│  claude_code_start → 返回 job_id                 │
│  claude_code_status → 轮询状态                    │
│  claude_code_output → 获取结果                    │
│  🔒 凭据仅通过容器挂载 ~/.claude/ 获取             │
├─────────────────────────────────────────────────┤
│  验收层（独立脚本，不在 Claude Code 内部）           │
│  lint + typecheck + test → pass/fail             │
├─────────────────────────────────────────────────┤
│  通知层                                          │
│  webhook 回调 → Slack / 飞书 通知结果              │
└─────────────────────────────────────────────────┘
```

### 最小闭环流程

```
需求输入 → 任务拆分 → Claude Code 写代码 → 自动 review/verify
  ↑                                                    ↓
  ←── 不通过：回修（≤N轮） ←←←←←←←←←←←←←←←←←←←←←←←←←←
  ←── 多轮失败：通知人工介入
  ──→ 通过：进入下一任务
```

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

写入以下内容：

```json
{
  "plugins": {
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

### v0.1 — 手动下发单任务，跑通链路

**目标**：验证 OpenClaw → Claude Code → 产出代码 这条路能通。

**准备一个测试 repo：**

```bash
# 在 workspaces 目录下创建一个测试项目
mkdir -p ~/.openclaw/workspaces
cd ~/.openclaw/workspaces
mkdir test-project && cd test-project
git init
npm init -y

# 创建一个简单的文件让 Claude Code 去修改
cat > index.js << 'EOF'
// TODO: 实现一个函数，接收数组，返回去重后的数组
function uniqueArray(arr) {
  // 请实现
}

module.exports = { uniqueArray };
EOF

# 创建测试文件
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
npx json -I -f package.json -e 'this.scripts.test="jest"'
git add -A && git commit -m "init: test project for openclaw integration"
```

**通过 OpenClaw 下发任务：**

```bash
# 使用 claude_code_start 工具下发编码任务
# OpenClaw 会创建一个容器，在其中运行 Claude Code
openclaw run --plugin claude-code --tool claude_code_start \
  --input '{
    "workspace": "~/.openclaw/workspaces/test-project",
    "prompt": "请实现 index.js 中的 uniqueArray 函数，使所有测试通过。运行 npm test 验证。",
    "allowed_tools": ["read", "edit", "bash"]
  }'

# 会返回一个 job_id，记录下来
# 例如：job_id = "abc123"
```

**查看任务状态：**

```bash
# 查看状态
openclaw run --plugin claude-code --tool claude_code_status \
  --input '{"job_id": "abc123"}'

# 查看输出
openclaw run --plugin claude-code --tool claude_code_output \
  --input '{"job_id": "abc123"}'
```

**手动验证结果：**

```bash
cd ~/.openclaw/workspaces/test-project
npm test          # 测试是否通过
git diff          # 查看 Claude Code 做了什么改动
git log --oneline # 查看是否有新 commit
```

**v0.1 通过标准**：Claude Code 成功修改了代码，`npm test` 全部通过。

---

### v0.2 — 加自动验收脚本

**目标**：用独立脚本自动判断任务是否完成，不依赖 Claude Code 的自我判断。

**创建验收脚本 `scripts/verify.sh`：**

```bash
mkdir -p scripts
cat > scripts/verify.sh << 'SCRIPT'
#!/bin/bash
# verify.sh - 独立于 Claude Code 的验收门
# 用法: ./scripts/verify.sh <project-dir>
# 返回: 0=通过, 1=失败

set -euo pipefail

PROJECT_DIR="${1:-.}"
cd "$PROJECT_DIR"

echo "========== 验收开始 =========="

# 第一关：代码风格检查（如果配置了 lint）
if grep -q '"lint"' package.json 2>/dev/null; then
  echo "[1/3] 运行 lint..."
  npm run lint || { echo "❌ lint 失败"; exit 1; }
  echo "✅ lint 通过"
else
  echo "[1/3] 跳过 lint（未配置）"
fi

# 第二关：类型检查（如果配置了 typecheck）
if grep -q '"typecheck"' package.json 2>/dev/null; then
  echo "[2/3] 运行 typecheck..."
  npm run typecheck || { echo "❌ typecheck 失败"; exit 1; }
  echo "✅ typecheck 通过"
else
  echo "[2/3] 跳过 typecheck（未配置）"
fi

# 第三关：测试
if grep -q '"test"' package.json 2>/dev/null; then
  echo "[3/3] 运行测试..."
  npm test || { echo "❌ 测试失败"; exit 1; }
  echo "✅ 测试通过"
else
  echo "[3/3] ⚠️ 未配置测试，跳过"
fi

echo "========== 验收通过 =========="
exit 0
SCRIPT
chmod +x scripts/verify.sh
```

**在 OpenClaw 编排中调用验收：**

```python
# orchestrate_v02.py - v0.2 编排脚本
import subprocess
import json

def claude_code_execute(workspace, prompt):
    """调用 claude-code 插件执行任务，返回 job_id"""
    result = subprocess.run([
        "openclaw", "run",
        "--plugin", "claude-code",
        "--tool", "claude_code_start",
        "--input", json.dumps({
            "workspace": workspace,
            "prompt": prompt,
            "allowed_tools": ["read", "edit", "bash"]
        })
    ], capture_output=True, text=True)
    return json.loads(result.stdout).get("job_id")

def wait_for_completion(job_id, timeout=300):
    """轮询等待任务完成"""
    import time
    for _ in range(timeout // 5):
        result = subprocess.run([
            "openclaw", "run",
            "--plugin", "claude-code",
            "--tool", "claude_code_status",
            "--input", json.dumps({"job_id": job_id})
        ], capture_output=True, text=True)
        status = json.loads(result.stdout)
        if status.get("state") in ("completed", "failed"):
            return status
        time.sleep(5)
    return {"state": "timeout"}

def run_verification(project_dir):
    """运行独立验收脚本"""
    result = subprocess.run(
        ["./scripts/verify.sh", project_dir],
        capture_output=True, text=True
    )
    return result.returncode == 0, result.stdout

# ---- 主流程 ----
workspace = "~/.openclaw/workspaces/test-project"
task_prompt = "请实现 index.js 中的 uniqueArray 函数，使所有测试通过。"

# 1. 下发任务
job_id = claude_code_execute(workspace, task_prompt)
print(f"任务已下发，job_id: {job_id}")

# 2. 等待完成
status = wait_for_completion(job_id)
print(f"任务状态: {status['state']}")

# 3. 独立验收
passed, output = run_verification(workspace)
print(output)
if passed:
    print("✅ 验收通过")
else:
    print("❌ 验收失败，需要回修")
```

**v0.2 通过标准**：验收脚本能独立判断任务是否完成，不依赖 Claude Code 的输出。

---

### v0.3 — 加失败回修循环

**目标**：验收失败后自动将错误反馈给 Claude Code 重新修复，最多 3 轮。

```python
# orchestrate_v03.py - v0.3 编排脚本（带回修循环）
import subprocess
import json
import time

MAX_RETRIES = 3

def claude_code_execute(workspace, prompt):
    result = subprocess.run([
        "openclaw", "run",
        "--plugin", "claude-code",
        "--tool", "claude_code_start",
        "--input", json.dumps({
            "workspace": workspace,
            "prompt": prompt,
            "allowed_tools": ["read", "edit", "bash"]
        })
    ], capture_output=True, text=True)
    return json.loads(result.stdout).get("job_id")

def wait_for_completion(job_id, timeout=300):
    for _ in range(timeout // 5):
        result = subprocess.run([
            "openclaw", "run",
            "--plugin", "claude-code",
            "--tool", "claude_code_status",
            "--input", json.dumps({"job_id": job_id})
        ], capture_output=True, text=True)
        status = json.loads(result.stdout)
        if status.get("state") in ("completed", "failed"):
            return status
        time.sleep(5)
    return {"state": "timeout"}

def run_verification(project_dir):
    result = subprocess.run(
        ["./scripts/verify.sh", project_dir],
        capture_output=True, text=True
    )
    return result.returncode == 0, result.stdout + result.stderr

def execute_with_retry(workspace, task_description):
    """带回修循环的任务执行"""
    prompt = task_description

    for attempt in range(1, MAX_RETRIES + 1):
        print(f"\n--- 第 {attempt}/{MAX_RETRIES} 轮 ---")

        # 执行
        job_id = claude_code_execute(workspace, prompt)
        print(f"job_id: {job_id}")

        status = wait_for_completion(job_id)
        print(f"执行状态: {status['state']}")

        if status["state"] == "failed":
            print("⚠️ Claude Code 执行失败")
            prompt = f"上一轮执行失败。原始任务：{task_description}。请重试。"
            continue

        # 独立验收
        passed, output = run_verification(workspace)

        if passed:
            print(f"✅ 第 {attempt} 轮验收通过！")
            return True, attempt
        else:
            print(f"❌ 第 {attempt} 轮验收失败")
            # 将失败信息作为反馈，构造回修 prompt
            prompt = (
                f"上一轮修改未通过验收，以下是失败输出：\n"
                f"```\n{output}\n```\n"
                f"原始任务：{task_description}\n"
                f"请根据失败信息修复代码。"
            )

    print(f"⛔ {MAX_RETRIES} 轮回修均未通过，需要人工介入")
    return False, MAX_RETRIES

# ---- 主流程 ----
workspace = "~/.openclaw/workspaces/test-project"
success, attempts = execute_with_retry(
    workspace,
    "请实现 index.js 中的 uniqueArray 函数，使所有测试通过。运行 npm test 验证。"
)
```

**v0.3 通过标准**：失败后能自动回修，且 3 轮内能收敛到通过。

---

### v0.4 — 多任务队列 + 状态持久化

**目标**：支持多个任务排队执行，任务状态持久化到文件，重启后可恢复。

**任务队列文件 `tasks/queue.json`：**

```json
{
  "tasks": [
    {
      "id": "task-001",
      "name": "实现 uniqueArray",
      "description": "实现 index.js 中的 uniqueArray 函数，使所有测试通过",
      "workspace": "~/.openclaw/workspaces/test-project",
      "status": "pending",
      "attempts": 0,
      "max_retries": 3,
      "created_at": "2026-03-15T10:00:00Z",
      "completed_at": null,
      "error": null
    },
    {
      "id": "task-002",
      "name": "添加 flattenArray 函数",
      "description": "在 index.js 中添加 flattenArray 函数，将嵌套数组展平为一维数组，并添加测试",
      "workspace": "~/.openclaw/workspaces/test-project",
      "status": "pending",
      "attempts": 0,
      "max_retries": 3,
      "created_at": "2026-03-15T10:01:00Z",
      "completed_at": null,
      "error": null
    }
  ]
}
```

```python
# orchestrate_v04.py - v0.4 编排脚本（多任务队列 + 持久化）
import subprocess
import json
import time
from datetime import datetime, timezone

QUEUE_FILE = "tasks/queue.json"

def load_queue():
    with open(QUEUE_FILE, "r") as f:
        return json.load(f)

def save_queue(queue):
    with open(QUEUE_FILE, "w") as f:
        json.dump(queue, f, indent=2, ensure_ascii=False)

def claude_code_execute(workspace, prompt):
    result = subprocess.run([
        "openclaw", "run",
        "--plugin", "claude-code",
        "--tool", "claude_code_start",
        "--input", json.dumps({
            "workspace": workspace,
            "prompt": prompt,
            "allowed_tools": ["read", "edit", "bash"]
        })
    ], capture_output=True, text=True)
    return json.loads(result.stdout).get("job_id")

def wait_for_completion(job_id, timeout=300):
    for _ in range(timeout // 5):
        result = subprocess.run([
            "openclaw", "run",
            "--plugin", "claude-code",
            "--tool", "claude_code_status",
            "--input", json.dumps({"job_id": job_id})
        ], capture_output=True, text=True)
        status = json.loads(result.stdout)
        if status.get("state") in ("completed", "failed"):
            return status
        time.sleep(5)
    return {"state": "timeout"}

def run_verification(project_dir):
    result = subprocess.run(
        ["./scripts/verify.sh", project_dir],
        capture_output=True, text=True
    )
    return result.returncode == 0, result.stdout + result.stderr

def process_task(task):
    """处理单个任务，带回修循环"""
    prompt = task["description"]

    while task["attempts"] < task["max_retries"]:
        task["attempts"] += 1
        task["status"] = "running"
        save_queue(load_queue())  # 持久化状态

        print(f"\n[{task['id']}] 第 {task['attempts']}/{task['max_retries']} 轮")

        job_id = claude_code_execute(task["workspace"], prompt)
        status = wait_for_completion(job_id)

        if status["state"] == "failed":
            prompt = f"上一轮执行失败。原始任务：{task['description']}。请重试。"
            continue

        passed, output = run_verification(task["workspace"])

        if passed:
            task["status"] = "completed"
            task["completed_at"] = datetime.now(timezone.utc).isoformat()
            print(f"✅ [{task['id']}] 完成")
            return True
        else:
            prompt = (
                f"上一轮修改未通过验收：\n```\n{output}\n```\n"
                f"原始任务：{task['description']}\n请修复。"
            )

    task["status"] = "failed"
    task["error"] = "超过最大重试次数"
    print(f"⛔ [{task['id']}] 失败，需要人工介入")
    return False

# ---- 主流程：依次处理队列中的 pending 任务 ----
queue = load_queue()

for task in queue["tasks"]:
    if task["status"] == "pending":
        process_task(task)
        save_queue(queue)  # 每个任务完成后持久化

print("\n========== 队列处理完毕 ==========")
for task in queue["tasks"]:
    print(f"  [{task['id']}] {task['name']}: {task['status']}")
```

**v0.4 通过标准**：多个任务能排队执行，中途重启后从 queue.json 恢复继续。

---

### v0.5 — 通知 + 人工介入入口

**目标**：任务完成或失败时通知到 Slack/飞书，失败任务可等待人工审批后重试。

**Slack 通知脚本 `scripts/notify.sh`：**

```bash
#!/bin/bash
# notify.sh - 发送通知到 Slack
# 用法: ./scripts/notify.sh <channel> <message>
# 前提: 设置环境变量 SLACK_WEBHOOK_URL

CHANNEL="${1:-general}"
MESSAGE="$2"

if [ -z "$SLACK_WEBHOOK_URL" ]; then
  echo "⚠️ SLACK_WEBHOOK_URL 未设置，跳过通知"
  echo "消息内容: $MESSAGE"
  exit 0
fi

curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  -d "{\"channel\": \"#${CHANNEL}\", \"text\": \"${MESSAGE}\"}"
```

**飞书通知脚本 `scripts/notify_feishu.sh`：**

```bash
#!/bin/bash
# notify_feishu.sh - 发送通知到飞书
# 用法: ./scripts/notify_feishu.sh <message>
# 前提: 设置环境变量 FEISHU_WEBHOOK_URL

MESSAGE="$1"

if [ -z "$FEISHU_WEBHOOK_URL" ]; then
  echo "⚠️ FEISHU_WEBHOOK_URL 未设置，跳过通知"
  echo "消息内容: $MESSAGE"
  exit 0
fi

curl -s -X POST "$FEISHU_WEBHOOK_URL" \
  -H 'Content-Type: application/json' \
  -d "{\"msg_type\": \"text\", \"content\": {\"text\": \"${MESSAGE}\"}}"
```

**在编排脚本中集成通知和人工介入：**

```python
# 在 orchestrate_v04.py 基础上添加

import os

def notify(message, channel="openclaw-tasks"):
    """发送通知"""
    # Slack
    subprocess.run(["./scripts/notify.sh", channel, message])
    # 飞书
    subprocess.run(["./scripts/notify_feishu.sh", message])

def wait_for_human_approval(task_id):
    """等待人工审批（通过文件信号）"""
    approval_file = f"tasks/approvals/{task_id}.approved"
    print(f"⏳ 等待人工审批，创建 {approval_file} 文件以继续...")
    notify(f"任务 {task_id} 需要人工介入，审批后创建 {approval_file} 继续")

    while not os.path.exists(approval_file):
        time.sleep(10)

    os.remove(approval_file)
    print(f"✅ 收到人工审批，继续执行")

# 修改 process_task 的失败分支：
# if task["status"] == "failed":
#     notify(f"⛔ 任务 [{task['id']}] {task['name']} 失败，需要人工介入")
#     wait_for_human_approval(task["id"])
#     task["status"] = "pending"  # 重置状态，等待下一轮
#     task["attempts"] = 0
```

**v0.5 通过标准**：失败任务能发出通知，人工确认后可重新执行。

---

## 关键原则

- **执行层最优 ≠ 系统层最优** — 不要让 Claude Code 同时当 PM、QA、调度器
- **"写代码"和"判定完成"必须拆开** — 不让 coder 自己宣布"我修好了"
- **凭据不进 agent** — 借鉴 n8n 模式，Claude 凭据只通过容器挂载传递，编排层不接触
- **第一版别追求全自动** — 先跑通最小闭环，再逐步加自动化
- **完成条件必须机器可判断** — lint、typecheck、测试、review approval、最大重试次数

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
├── README.md                    # 本文件
├── scripts/
│   ├── verify.sh                # 独立验收脚本
│   ├── notify.sh                # Slack 通知
│   ├── notify_feishu.sh         # 飞书通知
│   └── security_check.sh       # 安全检查脚本（建议每日 cron 运行）
├── tasks/
│   ├── queue.json               # 任务队列（持久化）
│   └── approvals/               # 人工审批信号目录
├── orchestrate_v02.py           # v0.2 编排脚本
├── orchestrate_v03.py           # v0.3 编排脚本（带回修）
├── orchestrate_v04.py           # v0.4 编排脚本（多任务+持久化）
└── .openclaw/
    └── openclaw.json            # OpenClaw 插件配置
```

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
