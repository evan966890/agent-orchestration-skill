---
name: agent-orchestration
description: Discord-based multi-agent orchestration for OpenClaw. Use when setting up agent collaboration, configuring inter-agent communication (sessions_spawn/sessions_send), building collaboration channels, debugging agent dispatch failures, or tuning the Star-Office-UI visualization monitor.
---

# Discord 多 Agent 编排技能

本技能覆盖 OpenClaw 多 Agent 系统的完整编排链路：权限配置 → 任务派发 → 频道通信 → 人格化消息 → 可视化监控。

## 架构总览

```
用户 Discord 消息
  ↓ (WebSocket)
OpenClaw Gateway (ws://127.0.0.1:18789)
  ↓ bindings 路由
Agent Session (e.g. chief-director)
  ↓ sessions_spawn (派发任务)
  ↓ exec team-msg.sh (频道 embed 消息)
Sub-Agent Session (e.g. content-editor)
  ↓ announce 回传
Parent Session → Discord 频道回复
```

**关键机制**:
- 所有 Agent 共用一个 Discord bot 身份（@macstudio）
- bot 自己的消息被 gateway 跳过，不会触发任何 agent
- Agent 间任务派发必须通过 `sessions_spawn` 工具，文字 @提及无效
- Agent 频道消息通过 `exec` + `team-msg.sh` 发送 Discord embed（带头像、彩色边框）
- Agent 直接回复（gateway 自动发的）应尽量为空或极简，避免与 embed 重复

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
    "visibility": "all"
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
  ]
}
```

### 陷阱: allowAgents 位置

```
❌ agents.defaults.subagents.allowAgents  → 报错 "Unknown config keys"
✅ agents.list[].subagents.allowAgents    → 必须写在每个 agent 的独立配置中
```

### 拓扑选择

| 拓扑 | 配置 | 适用场景 |
|------|------|---------|
| 星型 | 只给总监 `allowAgents: ["*"]` | 简单指令下发 |
| 网状 | 所有 agent 都配置 `allowAgents: ["*"]` | 协同工作流，推荐 |

## 二、团队频道消息系统（team-msg.sh）

### 为什么不用 openclaw message send

`openclaw message send` 发的是纯文本消息，没有头像、没有颜色、没有身份标识，可读性差。

### 解决方案：team-msg.sh — Discord Embed 消息

脚本位置：`~/.openclaw/bin/team-msg.sh`

**功能**：
- 每个 agent 有独立头像（DiceBear open-peeps 手绘风）、颜色边框、显示名
- 消息以 Discord embed 格式发送，可读性强
- 彩色圆形头像背景，兼容 dark mode

**用法**：
```bash
~/.openclaw/bin/team-msg.sh <agent-id> '消息内容'
```

**Agent ID → 显示名 / 颜色映射**：

| Agent ID | 显示名 | embed 颜色 | 头像背景色 |
|----------|--------|-----------|-----------|
| chief-director | 总监·策略 | #F4A460 (橙) | fed7aa |
| content-editor | 笔锋 · 内容编辑 | #22C55E (绿) | bbf7d0 |
| copywriter | 灵犀 · 文案策划 | #6366F1 (靛) | c7d2fe |
| visual-designer | 像素 · 视觉设计 | #E9C308 (黄) | fde68a |
| wechat-ops | 微澜 · 公众号运营 | #2EC4B6 (青) | a7f3d0 |
| community-ops | 织网 · 社群运营 | #9B59B6 (紫) | e9d5ff |
| data-analyst | 洞见 · 数据分析 | #3B82F6 (蓝) | bfdbfe |
| business-dev | 纽带 · 商务合作 | #EC4899 (粉) | fbcfe8 |
| video-director | 光影 · 视频编导 | #F97316 (橙) | fdba74 |
| video-editor | 蒙太奇 · 视频剪辑 | #607D8B (灰蓝) | cbd5e1 |
| short-video-ops | 流量 · 短视频运营 | #A020F0 (紫) | d8b4fe |

**头像系统**：
- API: `https://api.dicebear.com/9.x/open-peeps/png?size=128&radius=50&seed={seed}&backgroundColor={hex}`
- 每个 agent 有唯一 seed（如 `boss-nolan-v3`, `writer-aidan-v3`）
- `radius=50` 生成圆形头像
- `backgroundColor` 使用柔和色，确保 dark mode 可见
- 如果 Discord 缓存了旧头像，改 seed 后缀强制刷新（如 `-v3` → `-v4`）

**技术细节**：
- 使用 Discord Bot API 直接 POST embed 消息
- Bot Token: 从 team-msg.sh 脚本读取
- 需要 proxy `http://127.0.0.1:10808`
- `\n` 转换：shell 单引号中的 `\n` 是字面量，脚本内用 sed 转换后再 JSON 编码

### 文件上传：team-file.sh

脚本位置：`~/.openclaw/bin/team-file.sh`

**用途**：把文件作为 Discord 附件上传到频道（附带 embed 说明）
**用法**：`~/.openclaw/bin/team-file.sh <agent-id> <文件路径> '说明文字'`

Evan 在 Discord 上看不到本地文件路径。所有交付文件必须通过 team-file.sh 上传为 Discord 附件。

### 在 SOUL.md 中的使用方式

```markdown
## 📢 团队频道通信（必须遵守）

发消息：exec({ "command": "~/.openclaw/bin/team-msg.sh <agent-id> '你要说的话'" })
传文件：exec({ "command": "~/.openclaw/bin/team-file.sh <agent-id> /path/to/file.md '说明'" })
```

## 三、SOUL.md 编写规范（核心经验）

### 会议优先五步走派发流程（chief-director）

**核心规则：先开会讨论，至少2轮共识后才能动手。**

```
# 第一步：在频道宣布开会
exec({ "command": "~/.openclaw/bin/team-msg.sh chief-director '各位进会议室，Evan要三篇深度文...我的初步想法是...大家说说看法。'" })

# 第二步：spawn 全员讨论（第1轮——只讨论不动手）
sessions_spawn({ "agentId": "content-editor", "task": "选题讨论会（第1轮）：...请在频道发表意见。这轮只讨论，不要动笔。", "mode": "run" })
sessions_spawn({ "agentId": "copywriter", "task": "选题讨论会（第1轮）：...请在频道发表意见。这轮只讨论，不要动笔。", "mode": "run" })
sessions_spawn({ "agentId": "data-analyst", "task": "选题讨论会（第1轮）：...请在频道发表意见。这轮只讨论，不要动笔。", "mode": "run" })
# ... spawn 所有相关角色

# 第三步：收到全员意见后，总结 + 发起第2轮
exec({ "command": "~/.openclaw/bin/team-msg.sh chief-director '第一轮讨论收到了。笔锋说得有道理...灵犀的建议也好...综合调整为...大家觉得怎样？'" })
sessions_spawn({ ... 第2轮讨论 spawn，同样只讨论不动手 })

# 第四步：共识达成后，拍板 + 分配执行任务
exec({ "command": "~/.openclaw/bin/team-msg.sh chief-director '好，方向定了。笔锋写X，灵犀做Y，洞见出Z。各自开工。'" })
sessions_spawn({ "agentId": "content-editor", "task": "经过讨论共识：你的执行任务是...", "mode": "run" })

# 第五步：收到成果 → 审核 → team-file.sh 上传
# 直接回复只说：好的
```

### 直接回复规则（所有 agent 必须遵守）

Agent 有两个输出通道：
1. **直接文本回复** — gateway 自动发回 Discord，显示为 macstudio 的普通消息
2. **team-msg.sh embed** — 通过 exec 调用，显示为带头像的彩色 embed

**关键区分**：
- **顶级 agent（chief-director）**：直接回复只写 `NO_REPLY`。Gateway 识别该 token 后完全抑制消息发送。且**不能在工具调用旁附带任何文字**——每次文本输出都会被发到 Discord。
- **其他 10 个 agent**：直接回复只说 `已完成，详见频道。`。**绝对不能用 `NO_REPLY`**！

**为什么非总监 agent 不能用 NO_REPLY**：`NO_REPLY` 即 `SILENT_REPLY_TOKEN`（源码 `tokens-*.js` line 5）。`runSubagentAnnounceFlow()` 的 `isSilentReplyText()` 检查会匹配该 token — 如果子 agent 回复 NO_REPLY，整个 announce 被静默跳过，父 session（总监）永远收不到完成通知。

**为什么不用"好的"**：agent 的直接回复文本会作为 `findings` 出现在 announce 的 `triggerMessage` 中。总监收到 "Result: 好的" 两个字，会误以为 agent 在敷衍，发出"你这两个字是在敷衍我吗？"的回复。`已完成，详见频道。` 更有信息量。

**原因**：直接回复和 embed 内容重复，用户会看到同样的信息出现两次。

SOUL.md 写法：
```markdown
## ⚠️ 直接回复规则（极其重要）

你的直接文本回复应该尽量不输出。所有沟通通过 team-msg.sh embed 完成。
如果必须有文本输出，只说：`已完成，详见频道。`
绝对不要在直接回复里重复 embed 已经说过的内容。
```

### 格式禁令（所有 agent 必须遵守）

这是最容易被 LLM 违反的规则，必须写得极其明确。

```markdown
## ⛔ 格式禁令（违反等于任务失败，适用于你所有输出）

### 规则一：行首绝对禁止出现 emoji
不管什么 emoji，不管后面跟什么内容，行首不许有 emoji。
`📝 主题：` `📊 字数：` `✅ 案例：` `📋 计划：` `🔍 来源：` `📁 文件：` — 全部禁止。

### 规则二：禁止结构化汇报
不要像填表一样汇报。多行 emoji 列表、`字段名：值` 的罗列、`**加粗标签**：内容` 的格式、像 JIRA 一样的格式化报告 — 全部禁止。

### 规则三：称呼用名字，不用 agent ID
叫名字：笔锋、灵犀、洞见、像素、微澜、织网、纽带、光影、蒙太奇、流量、总监。
禁止：@content-editor、@copywriter、@data-analyst、@chief-director 等 agent ID。

### 对比示范
错误（像机器人）：
📝 主题：OpenClaw实战案例 / 📊 字数：2150字 / ✅ 案例：5个 / @data-analyst 帮忙核实

正确（像人说话）：
初稿写好了，2150字，5个案例都走完了。洞见帮忙核实下数据，灵犀看看标题传播力够不够。
```

**经验**：
- 仅列出几个常见 emoji 不够——LLM 会用其他 emoji 绕过（如 📝、🔍）
- 必须用"行首不许有 emoji"这种绝对规则
- 必须给正反对比示范，否则模型不知道"人话"长什么样
- 必须明确禁止 agent ID，并给出名字对照表

### 消息数量控制（chief-director 专用）

```markdown
### 消息数量控制
- 每次调度只发一条频道消息，把任务分工、截止时间、注意事项写在同一条里
- 不要先发一条"分工"、再发一条"分工总结" — 这是重复信息
- 后续有新进展（比如子 Agent 交稿）再发新消息
```

### 人格化性格系统

每个 agent 的 SOUL.md 包含 `## 性格与沟通` 章节，定义独特的叛逆性格：

| Agent | 核心性格 | 典型行为 |
|-------|---------|---------|
| 总监·策略 | 有主见但能被说服 | 会挑战团队方案，允许辩论后拍板 |
| 笔锋 | 内容质量洁癖 | 会说"这个选题太烂了"，跟灵犀互怼 |
| 灵犀 | 创意至上 | 挑战"安全"方案，觉得大多数内容太无聊 |
| 洞见 | 数据怼一切 | "数据不支持你这个判断"，用百分比说话 |
| 像素 | 审美固执 | 会直接说"这个丑"，拒绝低审美需求 |
| 微澜 | 数据务实 | "上次这种选题阅读量才800，你确定还要做？" |
| 织网 | 用户代言人 | "你们根本不懂用户"，带着社群真实反馈开会 |
| 纽带 | ROI 导向 | "做这个能带来什么商业价值？" |
| 光影 | 创作理念强 | 跟总监争创作方向，不喜欢被指挥怎么拍 |
| 蒙太奇 | 技术自信 | "这个效果做出来会很 low"，用时间码说话 |
| 流量 | 纯数据驱动 | "你这个封面点击率不会超过2%" |

**关键**：
- agent 之间有预设的互动关系（笔锋 vs 灵犀互怼、光影 vs 蒙太奇搭档但争节奏）
- 禁止"所有人都同意没人反驳"的和谐对话
- 禁止无脑说"收到"就开始干——先评估、先质疑、先提建议

### 会议讨论机制（关键）

任务下达后不能直接开工，必须先进"会议室"讨论。标准 3 轮讨论（亮牌→交锋→终极辩论）后才分工执行。

**流程**：
1. 总监宣布开会 → spawn 全员讨论（第1轮·亮牌，至少 5-6 人，只发意见不干活）
2. 总监等所有 announce 到齐 → 总结分歧+站队+挑衅 → spawn 第2轮（正面交锋）
3. 总监等所有 announce 到齐 → 判断碰撞是否充分 → 通常 spawn 第3轮（终极辩论）
4. 总监等所有 announce 到齐 → 拍板+立即 spawn 执行（同一次响应中完成，不能分两步）
5. 收到执行结果 → 审核点评 → 向 Evan 汇报

**讨论轮的 spawn task 写法**：
- 明确写"选题讨论会（第N轮）"
- 明确写"只讨论，不要动笔/不要执行"
- 要求用 team-msg.sh 在频道发表专业意见
- 第2轮要制造正面交锋："笔锋说X，灵犀说Y，总监倾向X——灵犀你怎么反驳？"

### ⛔ 计数规则（防止遗漏队友意见）

这是最容易出错的地方。总监收到大多数 announce 后会急着总结推进，导致最后一两个晚到几秒的 announce 被忽略。

**规则**：spawn 了几个 agent，就必须等几个 announce 回来。不是"大多数"，是"全部"。

- 每收到一个 announce，计数 +1
- 如果还没收到所有人的回复，文本回复只写 NO_REPLY（继续等下一个 announce）
- 只有当所有人都到齐了，才可以总结这一轮并推进下一轮
- 绝对不要说"XXX 可能因系统问题没送达"——announce 不是同时到的，后面的可能晚几秒

**真实错误**：在连续 3 轮测试中，总监每一轮都漏掉一个人——第1轮说"洞见可能因系统问题没送达"（其实洞见已发），第2、3轮连续说"灵犀可能没送达"（灵犀都发了）。根因是 SOUL.md 中有矛盾指令："收到大多数就推进" vs "尽量等所有人"，模型选了前者。

**修复**：清除所有 "收到大多数就推进" 的指令，统一为 "spawn 了 N 个就等 N 个"。在禁止做法中加入 "假设某人因系统问题没送达然后跳过他"。

### ⛔ 自动续流规则

总监的整个流程必须是一个连续动作，中间不停顿、不等 Evan 确认。

**致命错误**：拍板后说 "准备拍板派活，你如果对方向有异议现在说" — 这是在等 Evan 确认，流程会停死。

**正确做法**：拍板 = 立即 spawn 执行，在同一次响应中完成（exec team-msg.sh 发拍板消息 + sessions_spawn 派发任务 + NO_REPLY）。

**Discord 频道时间线效果**：
```
总监: 各位进会议室，这次要做三篇AI横评...
笔锋: 选题我有看法，纯横评太浅了，应该加实操对比...
灵犀: 标题要带争议性才有传播力，"AI工具横评"这种标题没人点...
洞见: 数据显示"怎么选AI工具"搜索量是"AI横评"的3倍...
总监: 第一轮讨论收到了。笔锋说得对，角度调一下...大家觉得新方向怎样？
笔锋: 新方向可以，但字数建议增加到3500...
灵犀: 同意调整，标题我有三个方案供选...
总监: 好，方向定了。笔锋写X，灵犀做Y。各自开工。
```

**关键**：讨论轮和执行轮是完全分开的 sessions_spawn 调用。第1-2轮只出意见，第3轮以后才出成果。

### 文件交付规则

Agent 完成任务后必须用 `team-file.sh` 上传文件到 Discord 频道。不能只给本地路径。

```bash
# 发消息
~/.openclaw/bin/team-msg.sh <agent-id> '消息'

# 上传文件（附带说明）
~/.openclaw/bin/team-file.sh <agent-id> /path/to/file.md '这是初稿，2800字，角度用了对比场景切入'
```

### 性格数据源

完整性格定义在 `/tmp/personality-update.json`，包含每个 agent 的：
- `personality_section`：完整的 `## 性格与沟通` markdown 内容
- 说话方式、互动关系、典型台词

### SOUL.md 调试实战经验（Round 3-9 总结）

这些经验来自连续 9 轮测试迭代，解决了 announce 泄露、流程停滞、重复上传、文字泄露等问题。

**1. 矛盾指令是最大的隐患**
SOUL.md 中如果有"尽量等所有人"和"收到大多数就推进"两条矛盾指令，模型会选择更容易的那条（推进）。务必检查所有指令的一致性。特别注意文档头部的"已知系统Bug"段落可能与后面的计数规则直接矛盾。

**2. 真实错误示范比抽象规则有效 10 倍**
"不要在直接回复中写详细内容"这种抽象规则，模型经常违反。改为列举 3-5 条"你在测试中犯过的真实错误"（附具体文字），效果立竿见影。

**3. GLM-5 会把代码块当文字输出**
SOUL.md 中的 ``` 代码块示例（如 `exec({...})`）会被 GLM-5 当成字面文字输出到 Discord。改用自然语言描述：「调用 exec 工具，command 参数填 `~/.openclaw/bin/team-msg.sh agent-id '消息'`」

**4. 清理旧会话必须覆盖所有 agent**
每个 agent 有独立的 sessions 目录：`~/.openclaw/agents/{agent-id}/sessions/`。之前连续 3 轮测试只清了 `agents/main/sessions/`，其他 10 个 agent 的旧会话一直残留——总监还记得"上一篇稿子已经完成了"就是因为旧 session 没清。workspace 中上次测试的残留文件（如 article-draft.md）也必须删除，否则总监会找到旧文件跳过整个讨论流程。

清理命令：
```bash
for agent_dir in ~/.openclaw/agents/*/sessions; do
  echo '{}' > "$agent_dir/sessions.json"
  find "$agent_dir" -maxdepth 1 -name "*.jsonl" -delete
done
```

**5. deliver: false 会破坏队列排水**
`shouldDeliverExternally = false` 看似安全（阻止外部投递），但实际破坏了 `maybeQueueSubagentAnnounce` 的队列排水机制，导致并发 announce 只有第一个能投递。

**6. 讨论轮数和发言长度建议写在 SOUL.md 里**
"讨论发言要简短有力。150-250字把观点和理由说清楚" — 没有这条约束，agent 会写 500+ 字的长篇大论。

**7. Opus 模型会在工具调用之间"想出声"**
Claude Opus 4.6 有强烈的"narrate actions"倾向——在工具调用之间输出 "I need to first search memory..."、"Now let me spawn agents..." 这样的计划性文字。这些文字全部被 Gateway 当作直接回复发到 Discord。Sonnet 较少出现此问题。

修复方法：在 SOUL.md 顶部明确列出这些泄露文字作为错误示范，并强调"工具调用之间不要写任何文字"。仅说"回复只写 NO_REPLY"不够——模型把 NO_REPLY 理解为"最终回复"，但中间步骤的旁白文字仍然会输出。

**8. 关键规则必须同时出现在 SOUL.md 开头和结尾**
LLM 对长文档的注意力分布是 U 型的——开头和结尾最强，中间最弱。如果 NO_REPLY 规则只出现在第 230 行（文档中部），模型在 340+ 行的长文档中很容易丢失它。解决方案：在文档第 3 行放一个加粗的最高优先级规则摘要，在文档最后放一个"最后提醒"段落，形成首尾夹击。

**9. 不同模型需要不同的规则措辞**
Sonnet 对"文本部分永远只写 NO_REPLY"的抽象规则遵守度较高。Opus 需要更具体的约束："工具调用之前/之间/之后都不要写文字"，配合真实错误示范。GLM-5 需要避免代码块被当文字输出。规则的具体程度要匹配目标模型的行为特点。

## 四、sessions_spawn 工具参数

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

**announce 机制**: 子 agent 完成后自动向父会话回传结果。如果父会话绑定了 Discord 频道，结果会自动发到该频道。

**子 agent 限制**: 默认无法使用 session 工具（sessions_spawn/send/list/history 被拒绝）。

## 四-B、Announce Flow 内部机制与泄露防护（深度解析）

### Announce 两条投递路径

源码位于 `dist/subagent-registry-CVXe4Cfs.js` 的 `runSubagentAnnounceFlow()` 函数。子 agent 完成后有两条路径将结果投递给父 agent：

| 路径 | 变量 | 行为 | 风险 |
|------|------|------|------|
| **Path A** (line ~78471) | `shouldSendCompletionDirectly` | 直接把 `completionMessage`（"✅ Subagent XXX finished\n\n{findings}"）通过 `callGateway({method:"send"})` 发到 Discord — **绕过父 agent** | 系统消息和 findings 原文泄露到 Discord |
| **Path B** (line ~78522) | `shouldDeliverExternally` | 把 `triggerMessage`（含 findings）通过 `callGateway({method:"agent", deliver: true/false})` 发给父 agent，父 agent 处理后回复 | 如果父 agent 的回复不是 NO_REPLY，回复文本会泄露到 Discord |

### `requesterIsSubagent` 陷阱（最关键的坑）

`requesterIsSubagent` 基于 session depth：
- depth > 0（子 agent 的子 agent）→ `requesterIsSubagent = true`
- **depth = 0（Discord-bound 顶级 session）→ `requesterIsSubagent = false`**

如果 chief-director 是从 Discord 频道触发的，它的 session depth = 0。即使它是其他 agent 的"父 agent"，在 announce flow 中 `requesterIsSubagent = false`。这意味着专门针对 `requesterIsSubagent = true` 的 patch 对 chief-director **完全不生效**。

### 推荐的源码 Patch（防止泄露）

```javascript
// 1. 禁用 Path A（防止 "✅ Subagent XXX finished" 泄露）
//    line ~78473
let shouldSendCompletionDirectly = false;  // 原值依赖条件计算

// 2. Path B 的 deliver 保持原值（不要改为 false！）
//    line ~78516 — 保持原始逻辑不动
//    ⚠️ 设 deliver: false 会破坏 announce 队列排水机制，只有 1/6 的 announce 能投递

// 3. 统一 buildAnnounceReplyInstruction（所有三个分支）
//    lines ~78649-78651
//    原值："Convert this completion into a concise internal orchestration update..."
//    新值（三个分支统一）：
`A subagent task has completed. Review the result and take any needed action using your tools (e.g. exec). Your text reply MUST be ONLY: ${SILENT_REPLY_TOKEN} — any descriptive text you output will leak to external channels. Communicate all updates through tool calls, not text output.`
```

**致命错误记录**：
- 把 `shouldDeliverExternally` 设为 `false` → 破坏 announce 队列（`maybeQueueSubagentAnnounce` 的排水依赖 `deliver: true`），6 个 announce 只有 1 个能投递到父 agent
- 只 patch `requesterIsSubagent = true` 的分支 → 对 Discord-bound 的 chief-director 无效（depth = 0 → false）

### `SILENT_REPLY_TOKEN = "NO_REPLY"` 的双面性

| 使用者 | 行为 | 安全性 |
|--------|------|--------|
| 顶级 agent（chief-director）| 回复 NO_REPLY → gateway regex 抑制发送 | ✅ 正确，防止泄露 |
| 子 agent（content-editor 等）| 回复 NO_REPLY → `isSilentReplyText()` 匹配 → announce 被静默跳过 | ❌ 危险！父 agent 永远收不到通知 |

**规则**：只有顶级 agent 使用 NO_REPLY，子 agent 使用 `已完成，详见频道。`

### 执行成果处理（防止重复上传）

当执行 agent（如笔锋）完成任务并通过 `team-file.sh` 上传了文件，总监收到 announce 时：
1. 不要再次上传同一个文件（agent 已上传）
2. 不要在回复中复述文章内容（会通过 Path B 的 deliver 泄露到 Discord）
3. 通过 `team-msg.sh` 发一条简短的 Evan 汇报
4. 文本回复只写 NO_REPLY

必须在 chief-director SOUL.md 中明确写出这个规则，并给出正反例。

## 五、Discord 协作频道配置

### 配置步骤

```json
// openclaw.json — 为每个 agent 添加频道绑定
"bindings": [
  // 已有的单 agent 频道绑定...

  // 协作频道: 所有 agent 都绑定同一个频道 (会议室)
  { "agentId": "chief-director", "match": { "channel": "discord", "peer": { "kind": "channel", "id": "1475813925772722228" } } },
  { "agentId": "content-editor", "match": { "channel": "discord", "peer": { "kind": "channel", "id": "1475813925772722228" } } },
  // ... 所有 11 个 agent
]

// 每个 agent 添加 groupChat 配置
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

### 关键限制

| 行为 | 是否触发 agent | 原因 |
|------|---------------|------|
| 用户在频道 @总监 | 触发 chief-director | 用户消息 + mentionPattern 匹配 |
| 用户在频道直接发消息 | 不触发 | 无 mentionPattern 匹配 |
| Bot embed 中写 @笔锋 | 不触发 | Gateway 跳过 bot 自己的消息 |
| Agent 调用 sessions_spawn | 触发目标 agent | 通过工具直接调度 |

## 六、会话管理与生效流程

### SOUL.md 修改后的生效流程（极其重要）

SOUL.md 的修改不会立即生效——旧会话中已经加载了旧版 SOUL.md 内容。必须：

```bash
# 1. 清理所有 agent 的活跃会话（包括 main！）
for agent_dir in ~/.openclaw/agents/*/sessions; do
  echo '{}' > "$agent_dir/sessions.json"
  find "$agent_dir" -maxdepth 1 -name "*.jsonl" -delete 2>/dev/null
done

# 2. 删除 workspace 中的残留文件（防止总监发现旧稿跳过讨论）
rm -f ~/.openclaw/workspace-chief-director/drafts/*.md
rm -f ~/.openclaw/workspace-chief-director/article-*.md

# 3. 重启 gateway
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist 2>/dev/null
sleep 1
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist

# 4. 验证
sleep 2 && pgrep -fl "openclaw-gateway"
```

**⛔ 致命陷阱：只清 agents/main/ 是不够的！**
每个 agent 都有独立的 sessions 目录（`~/.openclaw/agents/{agent-id}/sessions/`）。之前连续 3 轮测试只清了 `agents/main/sessions/`，其他 agent 的旧会话一直残留，导致总监还记得"上一篇稿子"、笔锋还有旧的对话上下文。必须用 `agents/*/sessions` 通配符清理所有 agent。

**其他陷阱**：
- gateway 重启时可能会立即重建某些 session（如正在等待 announce 的父会话），导致清理后文件立刻重新出现
- 解决：清理后立即覆写 `sessions.json` 为 `{}`，然后重启 gateway
- `launchctl kickstart -k` 比 `bootout + bootstrap` 更可靠（-k 表示 kill + restart）

## 七、Star-Office-UI 可视化监控

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
ACTIVE_THRESHOLD = 300   # 活跃判定阈值（秒）— thinking 模型单次 API 调用可达 2 分钟
```

### 活跃检测：双信号机制

单独用任何一种信号都不够可靠：

| 检测方式 | 优势 | 缺陷 |
|---------|------|------|
| `.jsonl` mtime | 实际会话活动的直接证据 | LLM API 调用期间（30-120s）不更新 |
| `sessions.json` 内容的 `updatedAt` | 精确记录每个 session 的更新时间 | 需要解析 JSON |

正确做法：双信号取 OR — 任一信号在 ACTIVE_THRESHOLD 内更新即判定活跃。

### app.py AUTO_IDLE_TTL 陷阱

`app.py` 的 `/status` 端点有 `_apply_auto_idle()` 函数，超过 `AUTO_IDLE_TTL` 秒后强制重置为 idle。

**必须设置 `AUTO_IDLE_TTL = 600`**（不能小于 ACTIVE_THRESHOLD）。

**monitor.py 必须每次 poll 都刷新活跃 agent 的 `updated_at`**，不仅仅在状态变化时。否则状态持续不变时 app.py 会误判为 idle。

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

### UI 状态映射

```javascript
// index.html 中必须包含所有 monitor 能输出的状态
STATE_COLORS.dispatching = '#E056A0';
ZONES.dispatching = { label: 'Meeting', rects: [[350, 60, 260, 140]] };
BUBBLES.dispatching = ['调度任务', '分配', '协调'];
```

## 八、代理架构（Proxy）

| 方案 | WebSocket | fetch/REST | 推荐 |
|------|-----------|-----------|------|
| `proxy-preload.cjs` | 不影响 | 生效 | 唯一正解 |
| `--use-env-proxy` | 断连 | 生效 | 绝对不用 |
| `all_proxy=socks5://` | 不兼容 | 不兼容 | 绝对不用 |

`proxy-preload.cjs` 只调用 `setGlobalDispatcher(new EnvHttpProxyAgent())`，仅影响 undici fetch，不碰 ws 库的 WebSocket。

## 九、team-supervisor 自动化守护

### 概述

`~/.openclaw/bin/team-supervisor.py` 是统一的 Python 守护进程，取代了 `discord-watchdog.sh` 和 `discussion-watchdog.py`，每 30 秒轮询 6 项检查，自动修复已知问题。

### Agent Session 催促机制（Check 4）

当 chief-director idle > 5 分钟且无活跃子 session 时，通过 `cron.add` 注入催促消息：

```python
# 催促消息注入（复用 cron.add 一次性任务模式）
params = {
    "name": "Supervisor",
    "agentId": agent_id,
    "schedule": {"kind": "at", "at": now_iso},
    "sessionTarget": "isolated",
    "wakeMode": "now",
    "payload": {"kind": "agentTurn", "message": "系统提醒：不要等 Evan 确认，自主决策继续执行。"},
    "delivery": {"mode": "announce", "channel": "discord", "to": "1475817567313723534"},
}
subprocess.run(["openclaw", "gateway", "call", "cron.add", "--json", "--params", json.dumps(params)], ...)
```

**冷却**: 同一 agent 10 分钟内只催一次。任何 agent idle > 20 分钟发 Discord 告警。

### 启动与调试

```bash
# 启动（tmux，幂等）
bash ~/.openclaw/bin/start-supervisor.sh

# 单次运行所有检查
python3 ~/.openclaw/bin/team-supervisor.py --once

# 只输出不执行
python3 ~/.openclaw/bin/team-supervisor.py --once --dry-run
```

### 与已有 watchdog 的关系

| 已有服务 | 被覆盖 | 说明 |
|---------|--------|------|
| `discord-watchdog.sh` (launchd 600s) | Check 2 | supervisor 30s 轮询更及时 |
| `discussion-watchdog.py` (launchd 60s) | Check 4 | supervisor 的 per-agent 去重更精准 |

supervisor 稳定运行 24h 后可禁用旧 watchdog。

## 九-B、微信公众号发布管线

### 完整管线流程

```
chief-director 产出文章 (.md)
  ↓
generate_image.sh (通义万象 qwen-image-max) → 封面图 (.png)
  ↓
wechat_publish.py → 上传图片 + 创建草稿
  ↓
pipeline.py → 发送预览 + 浏览器自动化确认 + 关闭 tab
```

### 图片生成（通义万象）

脚本：`~/.openclaw/bin/generate_image.sh`

```bash
# 用法
~/.openclaw/bin/generate_image.sh "AI technology futuristic office" /tmp/cover.png 16:9

# 支持比例: 16:9 | 4:3 | 1:1 | 3:4 | 9:16
```

- API: DashScope `qwen-image-max` 模型（同步调用）
- 不走代理（`--noproxy '*'`）—— 中国 API 直连
- JSON 构造用 Python（避免 prompt 中特殊字符破坏 JSON）
- **限频**: 连续调用需间隔 30 秒，否则 429
- **内容安全**: 敏感词触发 400，用中性英文 prompt 描述画面

### 浏览器 Tab 清理

`pipeline.py` 在微信公众号后台操作完后自动关闭 WeChat tab：

```python
def _browser_close_tab(cdp_port: str):
    """通过 CDP HTTP API 关闭 WeChat tab 防止 tab 堆积"""
    req = urllib.request.Request(f"http://localhost:{cdp_port}/json")
    with urllib.request.urlopen(req, timeout=5) as resp:
        targets = json.loads(resp.read())
    for t in targets:
        if t.get("type") == "page" and "mp.weixin.qq.com" in t.get("url", ""):
            urllib.request.urlopen(
                urllib.request.Request(f"http://localhost:{cdp_port}/json/close/{t['id']}"),
                timeout=5
            )
```

### 批量修复草稿封面

当图片生成失败导致草稿使用 placeholder 封面时（<10KB 灰色图），修复流程：

1. 获取所有草稿：`client.post("draft/batchget", data={"offset":0,"count":20})`
2. 下载 thumb_url 检查大小，<10KB 判定为 placeholder
3. 用通义万象重新生成封面（每张间隔 30 秒防限频）
4. 上传为永久素材：`client.material.add("image", f)`
5. 更新草稿（需传完整文章字段）：`client.post("draft/update", data={...})`

**陷阱**: `draft/update` 必须传 title/author/content/thumb_media_id 等所有必填字段，只传 thumb_media_id 会报 44004 "empty content"。

## 九-C、端到端编排检查清单

新建或修改多 Agent 编排系统时，逐项确认：

**权限与配置**
- [ ] 所有 agent 的 `subagents.allowAgents` 已配置（推荐 `["*"]`）
- [ ] `tools.agentToAgent.enabled: true` + `allow` 列表完整
- [ ] `tools.sessions.visibility: "all"`
- [ ] 每个 agent 配置了 `groupChat.mentionPatterns`
- [ ] 协作频道已创建，所有 agent 绑定了该频道

**SOUL.md**
- [ ] chief-director 使用会议优先五步走（开会 → 3轮讨论 → 拍板+执行 → 审核 → Evan汇报）
- [ ] chief-director 直接回复只用 `NO_REPLY`，且工具调用旁不附带任何文字
- [ ] 其他 10 个 agent 直接回复只用 `已完成，详见频道。`（不是"好的"，不是 NO_REPLY）
- [ ] 所有 agent 的 SOUL.md 包含 `## ⛔ 格式禁令`（行首无 emoji + 无结构化汇报 + 叫名字不叫 ID）
- [ ] 所有 agent 的 SOUL.md 包含 `## 性格与沟通`（叛逆性格 + 说话方式 + 150-250字发言限制）
- [ ] 所有 agent 的 SOUL.md 包含 `## 📢 团队频道通信`（使用 team-msg.sh）
- [ ] chief-director 有消息数量控制（每次调度只发一条 embed）
- [ ] chief-director 有计数规则（spawn N 个 agent 就等 N 个 announce，没到齐就 NO_REPLY）
- [ ] chief-director 有自动续流规则（拍板=立即spawn执行，不等 Evan 确认）
- [ ] chief-director 有执行成果处理规则（agent 已上传则不重复上传）
- [ ] chief-director SOUL.md 中无矛盾指令（"等所有人" vs "收到大多数就推进"）
- [ ] 所有 SOUL.md 含真实错误示范（比抽象规则有效 10 倍）

**源码 Patch（防止 announce 泄露）**
- [ ] `shouldSendCompletionDirectly = false`（line ~78473，禁用 Path A）
- [ ] `buildAnnounceReplyInstruction` 三分支统一为 NO_REPLY 指令（lines ~78649-78651）
- [ ] `shouldDeliverExternally` 保持原值不动（设为 false 会破坏队列排水）

**team-msg.sh**
- [ ] 脚本在 `~/.openclaw/bin/team-msg.sh` 且可执行
- [ ] 11 个 agent 全部有映射（name, color, avatar seed, backgroundColor）
- [ ] 头像使用 open-peeps 风格 + 彩色圆形背景（dark mode 兼容）
- [ ] proxy 配置正确（`--proxy http://127.0.0.1:10808`）

**监控**
- [ ] `proxy-preload.cjs` 已配置
- [ ] monitor.py 双信号检测活跃 + 每 poll 刷新 `updated_at`
- [ ] `ACTIVE_THRESHOLD >= 300` + `AUTO_IDLE_TTL >= 600`
- [ ] UI 的 STATE_COLORS/ZONES/BUBBLES 覆盖所有状态

**team-supervisor**
- [ ] `team-supervisor.py` 在 tmux 中运行（`start-supervisor.sh`）
- [ ] 6 项检查全部通过（`--once` 验证）
- [ ] Discord 连接检查用日志优先逻辑（避免 Health API 误报）
- [ ] Agent 催促冷却 10 分钟 / 告警去重 30 分钟
- [ ] 旧 watchdog 确认被 supervisor 覆盖后再禁用

**微信发布管线**
- [ ] `generate_image.sh` 使用通义万象 qwen-image-max（不走代理）
- [ ] `pipeline.py` 操作完关闭 WeChat tab（`_browser_close_tab`）
- [ ] 批量生成封面间隔 30 秒防限频
- [ ] `draft/update` 传完整文章字段（不只传 thumb_media_id）

**生效流程**
- [ ] SOUL.md 修改后清理所有 agent 会话（备份 .jsonl + 重置 sessions.json）
- [ ] 使用 `launchctl kickstart -k` 重启 gateway
- [ ] 验证新会话加载了最新 SOUL.md

## 十、诊断速查

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

### Embed 消息没发出去

```bash
# 手动测试 team-msg.sh
~/.openclaw/bin/team-msg.sh chief-director '测试消息，请忽略'

# 如果报 error，检查：
# 1. proxy 是否运行: curl --proxy http://127.0.0.1:10808 https://discord.com/api/v10/gateway
# 2. bot token 是否有效: 看返回的 HTTP status
# 3. channel ID 是否正确: 1475813925772722228
```

### Agent 直接回复太长/格式不对

1. 检查 SOUL.md 是否包含 `## ⚠️ 直接回复规则` 和 `## ⛔ 格式禁令`
2. 清理该 agent 的会话文件（旧会话可能加载了旧版 SOUL.md）
3. 重启 gateway
4. 如果仍不遵守，加强禁令措辞（"违反等于任务失败"比"不建议"有效得多）

### 头像在 dark mode 不可见

- 确认头像 URL 包含 `backgroundColor` 参数
- 确认不是 transparent 背景
- 如果 Discord 缓存了旧头像，修改 seed 值强制刷新（加后缀如 `-v4`）

### 可视化不显示活跃 Agent

```bash
# 检查 monitor 进程数（只应有一个）
ps aux | grep monitor.py | grep -v grep

# 检查 .jsonl 文件 mtime
for agent in chief-director content-editor copywriter; do
  latest=$(ls -t "$HOME/.openclaw/agents/$agent/sessions/"*.jsonl 2>/dev/null | head -1)
  if [ -n "$latest" ]; then
    mt=$(stat -f "%m" "$latest")
    age=$(($(date +%s) - mt))
    echo "$agent: age=${age}s (threshold=300s)"
  fi
done
```
