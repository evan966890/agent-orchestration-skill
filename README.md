# agent-orchestration

Claude Code skill for Discord-based multi-agent orchestration with OpenClaw.

## What it does

Provides battle-tested patterns and diagnostics for running 10+ AI agents as a collaborative team via Discord:

- **Inter-agent communication**: `sessions_spawn` / `sessions_send` permission setup, mesh vs star topology
- **SOUL.md authoring**: How to write agent instructions that actually dispatch tasks (not just text @mentions)
- **Collaboration channel**: Bind all agents to a shared Discord channel with `groupChat.mentionPatterns`
- **Visual monitoring**: Star-Office-UI `monitor.py` tuning — activity detection, threshold config, state accuracy
- **team-supervisor**: Unified 24/7 daemon replacing old watchdogs — 6 checks, auto-repair, Discord alerts, agent nudging via `cron.add`
- **WeChat publishing pipeline**: Tongyi Wanxiang image gen, browser tab cleanup, batch draft fix, `draft/update` field requirements
- **Proxy architecture**: `proxy-preload.cjs` vs `--use-env-proxy` — why only one of them works
- **End-to-end checklist**: Comprehensive verification for multi-agent setups (permissions, SOUL.md, patches, monitoring, supervisor, pipeline)

## Key lessons encoded

| # | Lesson | Impact |
|---|--------|--------|
| 1 | Text @mentions in bot replies don't trigger agents | Agents must use `sessions_spawn` tool |
| 2 | `allowAgents` must be per-agent, not in `agents.defaults` | Gateway rejects unknown config keys |
| 3 | `sessions.json` mtime is unreliable for activity detection | Gateway restart touches all files; use `.jsonl` mtime |
| 4 | `--use-env-proxy` breaks WebSocket connections | Use `proxy-preload.cjs` (undici-only patching) |
| 5 | Star topology creates a bottleneck at the director | Use mesh topology (`allowAgents: ["*"]` for all) |
| 6 | CLI-triggered sessions don't post results to Discord | Results go to the parent session's channel |
| 7 | `ACTIVE_THRESHOLD = 30` is too short for agent work | Use 120+ seconds |
| 8 | Multiple monitor processes overwrite each other | Kill stale processes before restarting |
| 9 | Health API Discord status false positive | Use log-based detection (team-supervisor pattern) |
| 10 | `draft/update` only sending thumb_media_id | WeChat API requires all article fields; 44004 error |
| 11 | Tongyi Wanxiang 429 rate limit on batch gen | Add 30s delay between requests |
| 12 | Browser tab accumulation from pipeline | Close WeChat tabs via CDP HTTP API after operations |

## Install

**Option A — Clone**:
```bash
git clone https://github.com/evan966890/agent-orchestration-skill.git
cp agent-orchestration-skill/agent-orchestration.md ~/.claude/commands/
```

**Option B — Direct download**:
```bash
curl -o ~/.claude/commands/agent-orchestration.md \
  https://raw.githubusercontent.com/evan966890/agent-orchestration-skill/main/agent-orchestration.md
```

## Usage

The skill activates automatically when you discuss multi-agent orchestration topics. You can also invoke it explicitly:

```
/agent-orchestration
```

Trigger examples:
- "帮我配置 agent 间的通信权限"
- "为什么 sessions_spawn 没有触发目标 agent"
- "可视化里 agent 不显示活跃状态"
- "怎么让所有 agent 在一个 Discord 频道里协作"

## Related

- [discord-ops-skill](https://github.com/evan966890/discord-ops-skill) — Discord connectivity troubleshooting (WebSocket, proxy, message delivery)

## Requirements

- OpenClaw gateway (v2026.2+)
- Discord bot with Message Content Intent
- HTTP proxy (for Discord API access from restricted networks)
- Star-Office-UI (optional, for pixel-art visualization)

## License

MIT
