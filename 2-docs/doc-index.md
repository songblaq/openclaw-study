# OpenClaw Documentation Index

*Generated: 2026-02-04*

Total English docs: ~260 files (excluding zh-CN translations)

## Category Overview

| Category | Files | Priority | Purpose |
|----------|-------|----------|---------|
| concepts/ | 28 | 🔴 HIGH | Core understanding |
| gateway/ | 27 | 🔴 HIGH | Gateway internals |
| channels/ | 23 | 🔴 HIGH | Channel integration |
| cli/ | 36 | 🟡 MED | CLI commands |
| tools/ | 19 | 🟡 MED | Agent tools |
| providers/ | 19 | 🟡 MED | AI providers |
| start/ | 10 | 🟡 MED | Getting started |
| install/ | 11 | 🟢 LOW | Installation |
| platforms/ | 30 | 🟢 LOW | Platform-specific |
| automation/ | 6 | 🔴 HIGH | Cron, heartbeat |
| nodes/ | 8 | 🟡 MED | Mobile nodes |
| plugins/ | 4 | 🟡 MED | Plugin system |
| reference/ | 18 | 🟢 LOW | Templates, misc |
| help/ | 3 | 🟢 LOW | FAQ, troubleshooting |
| web/ | 4 | 🟢 LOW | Web UI |
| Root level | 22 | 🟡 MED | Misc features |
| experiments/ | 6 | 🟢 LOW | Future plans |
| debug/ | 2 | 🟢 LOW | Debugging |

## Priority Order for Analysis

### Phase 2-1 Priority (AI Learning)

#### 1. Core Concepts (concepts/) - 28 files
Essential for understanding OpenClaw architecture:
- agent.md, agent-loop.md, agent-workspace.md
- session.md, sessions.md, session-pruning.md
- memory.md, compaction.md
- context.md, system-prompt.md
- channel-routing.md, messages.md
- models.md, model-providers.md, model-failover.md
- multi-agent.md, queue.md, retry.md
- architecture.md, streaming.md
- groups.md, group-messages.md
- presence.md, typing-indicators.md
- oauth.md, session-tool.md
- markdown-formatting.md, timezone.md, typebox.md, usage-tracking.md

#### 2. Gateway (gateway/) - 27 files
Core server functionality:
- index.md, configuration.md, configuration-examples.md
- protocol.md, bridge-protocol.md
- authentication.md, discovery.md, pairing.md
- sandboxing.md, sandbox-vs-tool-policy-vs-elevated.md
- heartbeat.md, health.md, doctor.md
- logging.md, troubleshooting.md
- remote.md, tailscale.md, multiple-gateways.md
- local-models.md, openai-http-api.md, openresponses-http-api.md
- background-process.md, gateway-lock.md, bonjour.md
- cli-backends.md, tools-invoke-http-api.md
- security/ (formal-verification.md, index.md)

#### 3. Channels (channels/) - 23 files
Messaging integration:
- index.md, troubleshooting.md, location.md
- discord.md, telegram.md, whatsapp.md, signal.md
- slack.md, msteams.md, googlechat.md
- imessage.md, bluebubbles.md
- matrix.md, mattermost.md, nextcloud-talk.md
- line.md, zalo.md, zalouser.md
- nostr.md, tlon.md, twitch.md, grammy.md

#### 4. Automation (automation/) - 6 files
Critical for autonomous operation:
- cron-jobs.md
- cron-vs-heartbeat.md
- poll.md
- webhook.md
- gmail-pubsub.md
- auth-monitoring.md

#### 5. Tools (tools/) - 19 files
Agent capabilities:
- index.md, exec.md, exec-approvals.md, elevated.md
- browser.md, browser-login.md, browser-linux-troubleshooting.md
- chrome-extension.md
- skills.md, skills-config.md, creating-skills.md, clawhub.md
- web.md, firecrawl.md, lobster.md
- subagents.md, agent-send.md
- thinking.md, slash-commands.md
- apply-patch.md, reactions.md, llm-task.md

### Phase 2-2 Priority (User Guide KO)

#### Essential for Luca:
1. start/getting-started.md → 시작하기
2. start/setup.md → 설정
3. gateway/configuration.md → 게이트웨이 설정
4. channels/discord.md → Discord 연동
5. channels/whatsapp.md → WhatsApp 연동
6. automation/cron-jobs.md → 크론잡
7. automation/cron-vs-heartbeat.md → 크론 vs 하트비트
8. tools/skills.md → 스킬 시스템
9. concepts/session.md → 세션 이해
10. concepts/memory.md → 메모리 시스템

## Document Structure

```
docs/
├── Root (22 files)
│   ├── bedrock.md, brave-search.md, broadcast-groups.md
│   ├── date-time.md, debugging.md, environment.md
│   ├── hooks.md, index.md, logging.md
│   ├── multi-agent-sandbox-tools.md, network.md
│   ├── perplexity.md, pi.md, pi-dev.md
│   ├── plugin.md, prose.md, scripts.md
│   ├── testing.md, token-use.md, tts.md, tui.md, vps.md
│
├── automation/ (6)
├── channels/ (23)
├── cli/ (36)
├── concepts/ (28)
├── debug/ (1)
├── diagnostics/ (1)
├── experiments/ (6)
├── gateway/ (27)
│   └── security/ (2)
├── help/ (3)
├── hooks/ (1)
├── install/ (11)
├── nodes/ (8)
├── platforms/ (30)
│   └── mac/ (18)
├── plugins/ (4)
├── providers/ (19)
├── refactor/ (5)
├── reference/ (18)
│   └── templates/ (12)
├── security/ (1)
├── start/ (10)
├── tools/ (19)
└── web/ (4)
```

## Next Steps

1. Read HIGH priority docs (concepts/, gateway/, channels/, automation/)
2. Extract key concepts for AI learning document
3. Create internals.md with deep mechanics
4. Build reference.md with all config options
5. Select essential docs for Korean translation
