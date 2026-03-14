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

## 搭建指南

### 前置安装

```bash
# 1. 安装 OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw onboard --install-daemon

# 2. 安装 Claude Code（官方推荐方式，无需 Node.js）
curl -fsSL https://claude.ai/install.sh | bash
# 安装完成后登录（团队号用户选择团队账号即可，无需单独配 API key）
claude
```

### 集成方式

**方式 A：社区 Skill（推荐入门）**

使用 [openclaw-claude-code-skill](https://github.com/Enderfga/openclaw-claude-code-skill)，将 Claude Code 作为 OpenClaw 的 skill 注册，通过 `claude_code_start` 工具下发编码任务。

**方式 B：容器化插件（推荐生产）**

使用 [openclaw-plugin-claude-code](https://github.com/13rac1/openclaw-plugin-claude-code)，在 Docker/Podman 容器中运行 Claude Code，隔离更好，适合长期运行。

### 编排逻辑示例

```python
MAX_RETRIES = 3

for task in task_list:
    retries = 0
    while retries < MAX_RETRIES:
        # 调用 Claude Code 执行编码
        result = claude_code_execute(
            prompt=f"在 {repo_path} 中完成: {task.description}",
            allowed_tools=["read", "edit", "bash"]
        )

        # 独立验收（不让 coder 自己判断完成）
        verify_ok = run_verification(
            lint_check=True,
            type_check=True,
            test_suite=task.test_pattern
        )

        if verify_ok:
            notify(channel="slack", msg=f"Task {task.name} 完成")
            break
        else:
            retries += 1
            task.feedback = get_failure_details()

    if retries >= MAX_RETRIES:
        notify(channel="slack", msg=f"Task {task.name} 需要人工介入")
        wait_for_human_approval()
```

### 验收脚本

```bash
#!/bin/bash
# verify.sh - 独立于 Claude Code 的验收门
npm run lint && npm run typecheck && npm run test
```

---

## 渐进式落地路线

| 阶段 | 做什么 | 验证目标 |
|------|--------|----------|
| v0.1 | OpenClaw 手动下发单任务 → Claude Code 执行 → 手动检查 | 通路跑通 |
| v0.2 | 加自动验收脚本（lint + test） | 验收能否自动判断 |
| v0.3 | 加失败回修循环（≤3 轮） | 回修是否有效收敛 |
| v0.4 | 加多任务队列 + 状态持久化 | 长时间运行稳定性 |
| v0.5 | 加 Slack/Feishu 通知 + 人工介入入口 | 闭环完整性 |

---

## 关键原则

- **执行层最优 ≠ 系统层最优** — 不要让 Claude Code 同时当 PM、QA、调度器
- **"写代码"和"判定完成"必须拆开** — 不让 coder 自己宣布"我修好了"
- **第一版别追求全自动** — 先跑通最小闭环，再逐步加自动化
- **完成条件必须机器可判断** — lint、typecheck、测试、review approval、最大重试次数

---

## 参考资源

- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw 官网](https://openclaw.ai/)
- [OpenClaw Claude Code Skill 插件](https://github.com/Enderfga/openclaw-claude-code-skill)
- [OpenClaw Claude Code 容器化插件](https://github.com/13rac1/openclaw-plugin-claude-code)
- [OpenClaw 完整指南 - Milvus Blog](https://milvus.io/blog/openclaw-formerly-clawdbot-moltbot-explained-a-complete-guide-to-the-autonomous-ai-agent.md)
- [OpenClaw + Claude Code 集成指南](https://github.com/affaan-m/everything-claude-code/blob/main/the-openclaw-guide.md)
- [Setup OpenClaw with Claude & Gemini](https://vertu.com/ai-tools/the-ultimate-guide-setting-up-openclaw-with-claude-code-and-gemini-3-pro/)
