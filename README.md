# wechat-agent-studio

12 个 AI Agent 通过 Discord 协作运营微信公众号 —— 从选题讨论、内容创作到封面生成、草稿发布的全自动化方案。

## What it does

Claude Code skill，提供多 Agent 公众号工作室的完整搭建和运维指南：

- **12 Agent 团队配置**: 总监、笔锋、灵犀、洞见、像素、微澜、织网、纽带、光影、蒙太奇、流量 — 各有独立性格和互动关系
- **会议讨论机制**: 3 轮讨论（亮牌→交锋→终极辩论）后拍板执行，禁止无脑"收到就干"
- **Discord 协作频道**: agent-channel 绑定、team-msg.sh embed 消息（带头像/颜色）、team-file.sh 文件上传
- **SOUL.md 编写规范**: 直接回复规则、格式禁令、计数规则、自动续流、Opus/Sonnet/GLM-5 模型差异应对
- **team-supervisor 守护进程**: 24/7 统一守护（6 项检查、30s 轮询、自动修复 + Discord 告警、agent 催促）
- **微信发布管线**: 通义万象图片生成、wechat_publish.py 草稿发布、pipeline.py 浏览器自动化预览、tab 自动清理
- **Star-Office-UI 可视化**: 像素风办公室监控 agent 实时状态
- **端到端检查清单**: 权限、SOUL.md、源码 patch、监控、supervisor、管线 — 逐项确认

## Key lessons (from 9 rounds of production testing)

| # | Lesson | Impact |
|---|--------|--------|
| 1 | Text @mentions don't trigger agents | Must use `sessions_spawn` tool |
| 2 | `allowAgents` must be per-agent config | `agents.defaults` rejects it |
| 3 | Contradictory SOUL.md instructions are fatal | Model picks the lazier option |
| 4 | Real error examples 10x more effective than abstract rules | List actual leaked text |
| 5 | Opus narrates between tool calls → text leaks to Discord | Need explicit "no text between tools" rule |
| 6 | Sub-agents must NOT use NO_REPLY | Breaks announce queue silently |
| 7 | Session cleanup must cover all 11 agent dirs | Only clearing `agents/main/` leaves stale context |
| 8 | Health API Discord status can be wrong | Use log-based detection instead |
| 9 | `draft/update` needs all article fields | Only sending `thumb_media_id` → 44004 error |
| 10 | Tongyi Wanxiang needs 30s delay between calls | 429 rate limit on batch generation |
| 11 | Browser tabs accumulate from pipeline runs | Close WeChat tabs via CDP after operations |
| 12 | Critical rules must appear at SOUL.md head AND tail | LLM attention is U-shaped on long docs |

## Install

```bash
# Option A: Clone
git clone https://github.com/evan966890/wechat-agent-studio.git
cp wechat-agent-studio/wechat-agent-studio.md ~/.claude/commands/

# Option B: Direct download
curl -o ~/.claude/commands/wechat-agent-studio.md \
  https://raw.githubusercontent.com/evan966890/wechat-agent-studio/main/wechat-agent-studio.md
```

## Usage

```
/wechat-agent-studio
```

Trigger examples:
- "帮我配置 12 个 agent 协作写公众号"
- "为什么总监 spawn 了 6 个 agent 但只收到 5 个 announce"
- "怎么让 agent 用通义万象生成封面图"
- "公众号发布管线报错了怎么排查"
- "team-supervisor 怎么催促卡住的 agent"

## Related

- [discord-ops-skill](https://github.com/evan966890/discord-ops-skill) — Discord 连接/代理/消息投递问题排查
- [chrome-crawl](https://github.com/evan966890/chrome-crawl) — 零检测网页爬取 + 浏览器 UI 自动化
- [wechat-autopublish](https://github.com/evan966890/wechat-autopublish) — 微信公众号自动发布管线

## Requirements

- OpenClaw gateway (v2026.2+)
- Discord bot with Message Content Intent
- HTTP proxy for Discord API access
- DashScope API key (通义万象图片生成)
- WeChat Official Account (公众号) with API credentials

## License

MIT
