<div align="center">

# 🛡️ durable-agents

**The agent skill for building durable, crash-surviving agents with Pi Durable.**

[![Agent Skill](https://img.shields.io/badge/agent--skill-SKILL.md-blueviolet?style=for-the-badge)](SKILL.md)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-ready-DA7756?style=for-the-badge&logo=anthropic&logoColor=white)](#install)
[![Codex](https://img.shields.io/badge/Codex-ready-412991?style=for-the-badge&logo=openai&logoColor=white)](#install)
[![Cursor](https://img.shields.io/badge/Cursor-ready-000000?style=for-the-badge&logo=cursor&logoColor=white)](#install)
[![Gemini](https://img.shields.io/badge/Gemini-ready-1BA1E3?style=for-the-badge&logo=googlegemini&logoColor=white)](#install)
[![Devin](https://img.shields.io/badge/Devin-ready-0A0A0A?style=for-the-badge)](#install)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

*Teach any agent how to build agents that never stop.*

</div>

---

## What it does

Drop [`SKILL.md`](SKILL.md) into your agent's skills directory and it learns how to build **durable agents** on [Pi Durable](https://earendil.com/posts/pi-durable/) — agents that survive crashes, run on remote devices, serve many clients at once, and wait for your approval instead of waiting for work.

Covers the full stack:

| Layer | What the skill teaches |
|---|---|
| **Harness** | storage backends, conversations, checkpoints, `Harness.open` + `resume()` |
| **Tools** | replay policies (`replay: "safe"`), idempotency keys, side-effect safety |
| **Tasks** | phased checkpoints, ownership trees, `abort` cleanup, background work |
| **Extensions** | prompt sections, hooks (`api.memo` approvals), mid-run code swaps |
| **Network** | the `dpi-backend` HTTP+SSE API, `dpi-sdk` client, device ops (`just net`) |
| **Patterns** | proactive agents (judge → threshold → agent → approval), subagents, forking, handoff |

## Install

**One-click (agent-config deeplink):**

[![Install durable-agents skill](https://raw.githubusercontent.com/LivioGama/agent-config/main/assets/install-badge-small.jpg)](agent-config://https://raw.githubusercontent.com/LivioGama/pi-durable-agents-skill/main/SKILL.md)

*Requires the [agent-config](https://github.com/LivioGama/agent-config) deeplink handler.*

**Or copy the skill into whichever agent you use:**

```bash
# Claude Code
mkdir -p ~/.claude/skills/durable-agents && cp SKILL.md ~/.claude/skills/durable-agents/

# Codex / Cursor / Gemini / Devin — same shape
mkdir -p ~/.codex/skills/durable-agents   && cp SKILL.md ~/.codex/skills/durable-agents/
mkdir -p ~/.cursor/skills/durable-agents  && cp SKILL.md ~/.cursor/skills/durable-agents/
mkdir -p ~/.gemini/skills/durable-agents  && cp SKILL.md ~/.gemini/skills/durable-agents/
mkdir -p ~/.devin/skills/durable-agents   && cp SKILL.md ~/.devin/skills/durable-agents/
```

Then just ask your agent to "build a durable agent" — the skill activates on its own.

## Extra resource

[`video-transcript.txt`](video-transcript.txt) — the full transcript the skill was distilled from, kept for deep reference.

## Credits

- 📺 **[Pi Durable PROACTIVE Agents — Jev Triggers My ON-CALL Pi Agent](https://www.youtube.com/watch?v=DNDYYHsypZg)** — IndieDevDan's walkthrough that this skill distills.
- 📄 **[Pi Durable](https://earendil.com/posts/pi-durable/)** — the original announcement by [Earendil](https://earendil.com) (Mario & Armin), creators of the [Pi coding agent](https://github.com/earendil-works/pi).
- 🧰 Built against the [Durable Agent Network](https://github.com/earendil-works/pi) (`durable-pi`) reference repo: `dpi-backend`, `dpi-sdk`, `dpi-playground`, `dpi-tui`, `proactive-dpi-demo`.

## License

MIT — see [LICENSE](LICENSE).
