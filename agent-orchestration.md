---
name: agent-orchestration
description: Discord-based multi-agent orchestration for OpenClaw. Use when setting up agent collaboration, configuring inter-agent communication (sessions_spawn/sessions_send), building collaboration channels, debugging agent dispatch failures, or tuning the Star-Office-UI visualization monitor.
---

# Discord 多 Agent 编排技能

本技能覆盖 OpenClaw 多 Agent 系统的完整编排链路：权限配置 → 任务派发 → 协作频道 → 可视化监控。

## 架构总览

```
用户 Discord 消息
  ↓ (WebSocket)
OpenClaw Gateway (ws://127.0.0.1:18789)
  ↓ bindings 路由
Agent Session (e.g. chief-director)
  ↓ sessions_spawn
Sub-Agent Session (e.g. content-editor)
  ↓ announce 回传
Parent Session → Discord 频道回复
```

**关键机制**:
- 所有 Agent 共用一个 Discord bot 身份（@macstudio）
- bot 自己的消息被 gateway 跳过，不会触发任何 agent
- Agent 间任务派发必须通过 `sessions_spawn` 工具，文字 @提及无效

## 一、Agent 间通信权限

### 两套独立权限体系

| 权限 | 配置位置 | 控制的工具 | 作用 |
|------|----------|-----------|------|
| agentToAgent | `tools.agentToAgent.enabled` + `allow` | `sessions_send` | 向已有 session 发消息 |
| subagents | `agents.list[].subagents.allowAgents` | `sessions_spawn` | 创建子 agent session |

### 配置示例

```json
// openclaw.json

// 1) 全局: 允许 agent 间发消息
"tools": {
  "agentToAgent": {
    "enabled": true,
    "allow": ["chief-director", "content-editor", "copywriter", ...]
  },
  "sessions": {
    "visibility": "all"  // agent 可看到所有 session
  }
}

// 2) 每个 agent: 允许 spawn 子 agent
"agents": {
  "list": [
    {
      "id": "chief-director",
      "subagents": { "allowAgents": ["*"] }
    },
    {
      "id": "content-editor",
      "subagents": { "allowAgents": ["*"] }
    }
    // ... 所有 agent 都需要配置
  ]
}
```

### ⚠️ 陷阱: allowAgents 位置

```
❌ agents.defaults.subagents.allowAgents  → 报错 "Unknown config keys"
✅ agents.list[].subagents.allowAgents    → 必须写在每个 agent 的独立配置中
```

### 拓扑选择

| 拓扑 | 配置 | 适用场景 |
|------|------|---------|
| 星型 | 只给总监 `allowAgents: ["*"]` | 简单指令下发，总监是唯一调度者 |
| 网状 | 所有 agent 都配置 `allowAgents: ["*"]` | 协同工作流，agent 间可互相协作 |

**推荐网状拓扑** — 避免总监成为瓶颈，每个角色可以直接找协作方。

## 二、SOUL.md 编写规范（核心教训）

### 问题场景

SOUL.md 写了"收到图文任务 → 路由给 content-editor"，但 agent 不知道"路由"的技术手段，
只是在回复文字中写 `@笔锋 请完成xxx` → **无任何效果**（bot 自己的消息被 gateway 跳过）。

### 解决方案: 在 SOUL.md 中明确 sessions_spawn 用法

在 SOUL.md 的调度规则后面，必须添加**派发方法**章节：

```markdown
## ⚠️ 派发方法（必须遵守）

**分配任务时，你必须使用 `sessions_spawn` 工具来实际派发任务。**
仅仅在消息文本中写 @某人 是无效的——对方不会收到任何通知，任务不会被执行。

### 正确做法
1. 先在频道回复中说明你的调度计划（让用户看到）
2. 然后对每个任务，调用 `sessions_spawn` 工具：
   - `agentId`: 目标 Agent 的 ID
   - `task`: 完整的任务描述
   - `mode`: 使用 `run`

### 示例
sessions_spawn({
  "agentId": "content-editor",
  "task": "完成本周内容排期主表。要求：...",
  "mode": "run"
})

### 禁止做法
- ❌ 仅在回复文本中写 @Agent名称
- ❌ 把多个任务合并到一条消息里用@提及
- ✅ 每个任务单独调用一次 sessions_spawn
```

### AGENTS.md 配套修改

任务分配模板改为两步流程：

```markdown
## 任务分配流程

### 第一步：在频道中发布调度计划（让用户可见）
📋 任务分配
- 负责人: [Agent名称]
- 任务: [具体描述]
- 交付物: [预期产出]
- 截止时间: [日期时间]

### 第二步：用 sessions_spawn 实际派发
sessions_spawn({
  "agentId": "content-editor",
  "task": "完整任务描述...",
  "mode": "run"
})

⚠️ 仅在消息文本中写 @Agent名称 不会触发任何Agent执行。
```

## 三、sessions_spawn 工具参数

```json
{
  "task": "string (必填) — 任务描述",
  "agentId": "string (可选) — 目标 agent ID，默认为请求者自身",
  "mode": "string — 'run'(执行完汇报) | 'session'(持久会话)",
  "label": "string (可选) — 任务标签",
  "model": "string (可选) — 覆盖默认模型",
  "thinking": "string (可选) — 'low'|'medium'|'high'",
  "runTimeoutSeconds": "number (可选) — 超时秒数，0=无超时",
  "thread": "boolean — 是否绑定 Discord thread",
  "cleanup": "string — 'delete'|'keep'"
}
```

**返回值**: `{ status: "accepted", runId, childSessionKey }`

**announce 机制**: 子 agent 完成后自动向父会话回传结果（状态、产出、耗时、token 用量）。
如果父会话绑定了 Discord 频道，结果会自动发到该频道。

**子 agent 限制**: 默认无法使用 session 工具（sessions_spawn/send/list/history 被拒绝）。

## 四、Discord 协作频道配置

### 目标
让所有 agent 在一个 Discord 频道中协作，用户可以看到讨论过程。

### 配置步骤

```bash
# 1. 获取频道 ID（从 Discord URL 或右键复制 ID）
CHANNEL_ID="1475813925772722228"
```

```json
// 2. openclaw.json — 为每个 agent 添加频道绑定
"bindings": [
  // ... 已有的单 agent 频道绑定 ...

  // 协作频道: 所有 agent 都绑定同一个频道
  { "agentId": "chief-director", "match": { "channel": "discord", "peer": { "kind": "channel", "id": "CHANNEL_ID" } } },
  { "agentId": "content-editor", "match": { "channel": "discord", "peer": { "kind": "channel", "id": "CHANNEL_ID" } } },
  { "agentId": "copywriter",     "match": { "channel": "discord", "peer": { "kind": "channel", "id": "CHANNEL_ID" } } }
  // ... 所有 agent
]

// 3. 每个 agent 添加 groupChat 配置
"agents": {
  "list": [
    {
      "id": "chief-director",
      "groupChat": {
        "mentionPatterns": ["@总监", "@chief"],
        "historyLimit": 10
      }
    }
  ]
}
```

### ⚠️ 关键限制

| 行为 | 是否触发 agent | 原因 |
|------|---------------|------|
| 用户在频道 @总监 | ✅ 触发 chief-director | 用户消息 + mentionPattern 匹配 |
| 用户在频道直接发消息 | ❌ 不触发 | 无 mentionPattern 匹配 |
| Bot 回复中写 @笔锋 | ❌ 不触发 | Gateway 跳过 bot 自己的消息 |
| Agent 调用 sessions_spawn | ✅ 触发目标 agent | 通过工具直接调度 |

**结论**: 协作频道中，agent 间的任务传递必须通过 `sessions_spawn`，不能靠文字 @提及。

## 五、Star-Office-UI 可视化监控

### 架构

```
monitor.py (LaunchAgent, 每5秒扫描)
  ↓ 扫描 ~/.openclaw/agents/<id>/sessions/*.jsonl
  ↓ 检测状态: writing/researching/executing/syncing/error/idle
  ↓ 写入 state.json
Star-Office-UI (port 18800, 读取 state.json)
  → 像素风格办公室，agent 角色在对应工位活动
```

### monitor.py 关键配置

```python
POLL_INTERVAL = 5        # 扫描间隔（秒）
ACTIVE_THRESHOLD = 300   # 活跃判定阈值（秒）— 5分钟，因为 thinking 模型单次 API 调用可达 2 分钟
```

### ⚠️ 活跃检测：双信号机制

单独用任何一种信号都不够可靠：

| 检测方式 | 优势 | 缺陷 |
|---------|------|------|
| `.jsonl` mtime | 实际会话活动的直接证据 | LLM API 调用期间（30-120s）不更新 |
| `sessions.json` mtime | 更新频率高 | gateway 重启会刷新所有文件 |
| `sessions.json` 内容的 `updatedAt` | 精确记录每个 session 的更新时间 | 需要解析 JSON |

**正确的检测逻辑 — 双信号取 OR**:

```python
def _is_agent_active(sessions_dir, now):
    # Signal 1: .jsonl file mtime
    _, jsonl_mtime = _find_latest_session_file(sessions_dir)
    if jsonl_mtime > 0 and now - jsonl_mtime < ACTIVE_THRESHOLD:
        return True

    # Signal 2: sessions.json content — updatedAt 字段
    # 捕获 LLM 调用期间 jsonl 未更新但 session 已注册的情况
    session_ts = _get_latest_session_updated_at(sessions_dir)
    if session_ts > 0 and now - session_ts < ACTIVE_THRESHOLD:
        return True

    return False

def _get_latest_session_updated_at(sessions_dir):
    """Parse sessions.json to find most recent updatedAt (ms → seconds)."""
    try:
        with open(sessions_dir / "sessions.json") as f:
            data = json.load(f)
        max_ts = 0
        for key, val in data.items():
            if isinstance(val, dict):
                ts = val.get("updatedAt", 0)
                if isinstance(ts, (int, float)) and ts > max_ts:
                    max_ts = ts
        return max_ts / 1000.0 if max_ts > 0 else 0
    except (FileNotFoundError, json.JSONDecodeError, OSError):
        return 0
```

### 状态检测规则（从 jsonl 末尾 8KB 反向扫描）

| JSONL 内容关键词 | 映射状态 |
|-----------------|---------|
| `"exec"`, `"bash"`, `"shell"` | executing |
| `"sessions_spawn"`, `"sessions_send"`, `"subagent"` | dispatching |
| `"web_search"`, `"web_fetch"`, `"memory_search"` | researching |
| `"agentmessage"`, `"sendmessage"`, `"message"` | syncing |
| `"role":"assistant"` | writing |
| `"error"` + `"role":"system"` | error |
| 无匹配 / 文件不存在 | idle |

### ⚠️ app.py AUTO_IDLE_TTL 陷阱（关键 Bug）

`app.py` 的 `/status` 端点有一个 `_apply_auto_idle()` 函数，会在 `updated_at` 超过 `AUTO_IDLE_TTL` 秒后强制将 agent 状态改回 idle。

**问题链**:
1. monitor.py 检测到 agent 活跃，写入 `state = "writing"`, `updated_at = now`
2. monitor.py 只在状态**变化**时更新 `updated_at`
3. agent 持续 writing，状态不变，`updated_at` 不刷新
4. `AUTO_IDLE_TTL` 秒后，`_apply_auto_idle` 在 `/status` 返回前强制重置为 idle
5. UI 轮询 `/status` 看到 idle，agent 回到休息区

**修复（两处都必须改）**:

```python
# app.py — 增大 TTL，让 monitor.py 负责真正的 idle 检测
AUTO_IDLE_TTL = 600  # 原来是 25，远小于 ACTIVE_THRESHOLD

# monitor.py — 每次 poll 都刷新活跃 agent 的 updated_at，不仅仅在状态变化时
if active:
    # ... detect state ...
    if agent.get("state") != new_state:
        agent["state"] = new_state
        agent["updated_at"] = str(now)
        changed = True
    else:
        # 即使状态没变，也刷新时间戳，防止 app.py 的 safety-net 误判
        agent["updated_at"] = str(now)
        changed = True
```

### UI 状态映射

UI 需要为所有 monitor.py 能输出的状态定义 STATE_COLORS、ZONES 和 BUBBLES，否则未映射状态（如 `dispatching`）会 fallback 到 idle 区域。

```javascript
// 必须在 index.html 中添加:
STATE_COLORS.dispatching = '#E056A0';
ZONES.dispatching = { label: 'Meeting', rects: [[350, 60, 260, 140]] };
BUBBLES.dispatching = ['📋 派发任务', '🚀 分配', '📤 调度'];
```

### 常见问题

| 问题 | 原因 | 修复 |
|------|------|------|
| Agent 在工作但 25 秒后回到休息区 | `app.py` 的 `AUTO_IDLE_TTL=25` 太短 + monitor 不刷新时间戳 | AUTO_IDLE_TTL=600 + monitor 每 poll 刷新 updated_at |
| Agent 在工作但可视化显示 idle | ACTIVE_THRESHOLD 太短（LLM 调用期间 jsonl 不更新） | 改为 300 秒 + 启用 sessions.json updatedAt 双信号 |
| 所有 agent 都显示非 idle | sessions.json mtime 被 gateway 重启刷新 | 不用 sessions.json mtime，用其 JSON 内容的 updatedAt |
| 可视化不更新 | 多个 monitor 进程冲突 | `pkill -f monitor.py` 杀掉所有旧进程再重启 |
| state.json 解析失败 | 文件损坏（多余的 `}`） | 手动修复 JSON |
| dispatching 状态 agent 在休息区 | UI 缺少 dispatching 的颜色/区域/气泡映射 | 添加到 STATE_COLORS、ZONES、BUBBLES |
| 部分 agent 检测到、部分没检测到 | 只用单信号、阈值不够 | 双信号 OR + 300s 阈值 |

## 六、代理架构（Proxy）

### 方案对比

| 方案 | WebSocket | fetch/REST | memorySearch | 推荐 |
|------|-----------|-----------|-------------|------|
| `--use-env-proxy` | ❌ 断连 | ✅ | ✅ | ❌ 绝对不用 |
| `proxy-preload.cjs` | ✅ 不影响 | ✅ | ✅ | ✅ 唯一正解 |
| `all_proxy=socks5://` | ❌ 不兼容 | ❌ | ❌ | ❌ 绝对不用 |

### proxy-preload.cjs 原理

```
NODE_OPTIONS=--require=/path/to/proxy-preload.cjs

proxy-preload.cjs:
  → undici.setGlobalDispatcher(new EnvHttpProxyAgent())
  → 只影响 undici 的 fetch（Discord REST API, OpenAI embedding, CDN）
  → 不影响 ws 库的 WebSocket（ws 用原生 TCP/TLS）
  → 额外 patch: 移除 Discord 域名的 SSRF guard pinned dispatcher
```

## 七、OAuth Token 管理

### openai-codex Token 同步

Codex CLI 和 OpenClaw 的 OAuth token 没有自动同步机制。

```bash
# 1. 检查 token 状态
openclaw models auth status

# 2. 如果过期，终端重新登录
codex  # 自动跳转浏览器登录

# 3. 同步 token
# 源: ~/.codex/auth.json
# 目标: ~/.openclaw/agents/main/agent/auth-profiles.json
```

**字段映射**:

| Codex CLI (auth.json) | OpenClaw (auth-profiles.json) | 转换 |
|----------------------|------------------------------|------|
| `access_token` | `access` | 直接复制 |
| `refresh_token` | `refresh` | 直接复制 |
| JWT `exp` claim | `expires` | exp × 1000（秒→毫秒） |

```bash
# 4. 重启 gateway 生效
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist

# 5. 验证
openclaw models auth status
```

## 八、端到端编排检查清单

新建多 Agent 编排系统时，逐项确认：

```
□ 所有 agent 的 subagents.allowAgents 已配置（推荐 ["*"]）
□ tools.agentToAgent.enabled: true + allow 列表完整
□ tools.sessions.visibility: "all"
□ 总监/调度者的 SOUL.md 包含 sessions_spawn 使用说明和示例
□ AGENTS.md 的任务分配模板包含"两步流程"
□ 协作频道已创建，所有 agent 绑定了该频道
□ 每个 agent 配置了 groupChat.mentionPatterns
□ proxy-preload.cjs 已配置（plist 中的 NODE_OPTIONS）
□ monitor.py 使用双信号检测活跃（.jsonl mtime + sessions.json updatedAt 内容）
□ monitor.py 每次 poll 都刷新活跃 agent 的 updated_at（不仅仅在状态变化时）
□ ACTIVE_THRESHOLD >= 300 秒（thinking 模型单次调用可达 2 分钟）
□ app.py 的 AUTO_IDLE_TTL >= 600 秒（不能小于 ACTIVE_THRESHOLD，否则 /status 会提前重置）
□ index.html 的 STATE_COLORS / ZONES / BUBBLES 包含所有 monitor 输出的状态（含 dispatching）
□ 只有一个 monitor 进程在运行（`pkill -f monitor.py` 后再启动）
□ OAuth token 已同步且未过期
```

## 九、诊断速查

### Agent 没有响应 sessions_spawn

```bash
# 1. 检查 allowAgents 配置
python3 -c "
import json
with open('$HOME/.openclaw/openclaw.json') as f:
    cfg = json.load(f)
for a in cfg['agents']['list']:
    sa = a.get('subagents',{}).get('allowAgents','NOT SET')
    print(f'{a[\"id\"]:20s} allowAgents={sa}')
"

# 2. 检查子会话是否创建
openclaw sessions --all-agents --active 5

# 3. 检查日志
tail -50 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep -E "spawn|subagent|lane"
```

### Agent 结果没有回到 Discord 频道

1. 确认触发来源：从 Discord 频道触发 → 结果回频道 ✅ / 从 CLI 触发 → 结果回 CLI ❌
2. `announce` 机制将结果发回**父会话的频道**，不是子 agent 自己的频道
3. 如果 CLI 触发但需要结果发到 Discord: `openclaw agent --deliver --reply-channel discord --reply-to CHANNEL_ID`

### 可视化不显示活跃 Agent

```bash
# 检查 monitor 进程数
ps aux | grep monitor.py | grep -v grep

# 检查 .jsonl 文件 mtime
for agent in chief-director content-editor copywriter; do
  latest=$(ls -t "$HOME/.openclaw/agents/$agent/sessions/"*.jsonl 2>/dev/null | head -1)
  if [ -n "$latest" ]; then
    mt=$(stat -f "%m" "$latest")
    age=$(($(date +%s) - mt))
    echo "$agent: age=${age}s (threshold=120s)"
  fi
done

# 手动测试 monitor
cd ~/.openclaw/workspace/star-office-ui
python3 -c "import monitor; monitor.scan_agents()"
cat state.json | python3 -m json.tool
```
