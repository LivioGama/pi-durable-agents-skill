---
name: durable-agents
description: Build durable, crash-surviving agents with Pi Durable (@earendil-works/pi-durable) and the Durable Agent Network (durable-pi repo). Use when creating long-running agents, agents that must survive restarts/crashes, multi-client or proactive agents, durable task workflows with checkpoints and human approvals, or when working in the durable-pi repo (dpi-backend, dpi-sdk, dpi-playground, dpi-tui, proactive-dpi-demo).
---

# Durable Agents with Pi Durable

Sources: [Pi Durable announcement](https://earendil.com/posts/pi-durable/), the Durable Pi talk ("Pi Durable PROACTIVE Agents"), and the `durable-pi` repo (`~/Documents/durable-pi`, docs in `ai_docs/pi/`, API contract in `apps/dpi-backend/API.md`).

Packages: `@earendil-works/pi-durable`, `@earendil-works/pi-ai`, `@earendil-works/chord` (context/cancellation). Repo pins `1.0.3`. Node ≥22.19 (repo uses 24). Whole pi-durable source ≈15k lines — an agent can read it.

## Mental model

- **Harness** = storage (memory | SQLite | JSONL) + machinery running many **conversations** in parallel. One process owns a storage; clients attach to that process.
- **Conversation** = transcript + its own agent config (model, thinking level, extensions/tools, instructions, cwd). The root conversation is created on first use and is the same after every restart.
- **Task** = every step the harness runs (model request, tool call, compaction, extension tasks). Each task **commits a checkpoint before it moves on** — that is what makes a run survive process death.
- **Extension** = named bundle of system-prompt `section`s, `tools`, `hooks`, `tasks`. Installed in a `registry`; conversations store only names, so code can change under a running agent (malleability).
- **Document** (`defineDoc`) = typed JSON application state stored next to the transcript and mutated in the same atomic commit — state never disagrees with the transcript.
- **Execution environment** = where tools run (`NodeExecutionEnv`; interface is small — implement for remote/sandboxed envs). `env({cwd})` is per conversation.

## Bootstrap pattern

```ts
import { BACKGROUND_CONTEXT } from "@earendil-works/chord/context";
import { createModels } from "@earendil-works/pi-ai/models";
import { openaiProvider } from "@earendil-works/pi-ai/providers/openai";
import { createRegistry, Harness } from "@earendil-works/pi-durable";
import { NodeExecutionEnv } from "@earendil-works/pi-durable/env/node";
import { openNodeSqliteStorage } from "@earendil-works/pi-durable/storage/sqlite/node";
import { CodingTools } from "@earendil-works/pi-durable/tools";

const context = BACKGROUND_CONTEXT; // every API call takes a Context for cancellation
const models = createModels();
models.setProvider(openaiProvider());
const registry = createRegistry();
registry.install(CodingTools); // read, write, edit, bash
const env = ({ cwd }: { cwd?: string }) => new NodeExecutionEnv({ cwd: cwd ?? process.cwd() });

const harness = await Harness.open(await openNodeSqliteStorage("./agent.sqlite"), { models, registry, env }, context);
const root = await harness.root(context, { agent: { model: { provider: "openai", modelId: "gpt-6.1-sol" }, cwd: "/work/repo" } });
```

Submit work with a `requestId` — it makes the submission exactly-once across client retries and crashes:

```ts
const job = { type: "input", content: "Fix the flaky login test", requestId: "job-42" } as const;
await root.submit(job, context);
// after a crash: reopen storage, harness.resume(), resubmit — same requestId returns the original
```

## The rules that make agents durable

1. **Checkpoint before progress.** Never do side effects "then record"; record intent first (tool calls store intent before running).
2. **Replay policy on every tool.** `replay: "safe"` for pure reads/idempotent ops — a crash rerun is fine. Omit `replay` for irreversible side effects (deploy, charge, page) — an interrupted call is reported to the model, never silently repeated.
3. **Idempotency keys on real side effects.** Pass a stable key (`payment-${task.id}`) to external APIs so a phase rerun doesn't double-charge.
4. **Ownership tree.** Tasks/conversations have owners (`{kind:"task",taskId}` or `{kind:"conversation"}` or `ownerless`). Abort propagates bottom-up; each task's `abort` cleans up its own effects (e.g., refund). `background: true` detaches a task from the conversation's current work — survives Esc, still abortable.
5. **Memos for hook decisions.** `api.memo("key", value)` — first write wins, survives restart; use for human approvals so a crash doesn't re-ask.
6. **Documents for app state** (`defineDoc`): `scope` (`conversation` | `task`), `history` (`rewindable` | `latest`), `fork` (`asOf` | `initial` | `current`), `initial()`. Mutate via `tx.doc(Doc, conversationId)` inside `api.commit`.
7. **Compaction is a task too.** Tuning via harness `settings.compaction` (`reserveTokens`, `backgroundTokens`); `root.compact("instructions")` manually; `reset(note)` starts a new context — nothing is deleted, so a `search_history` tool can still read everything.
8. **Subagents are conversations.** A tool creates an owned conversation, `configure`s a cheaper model/tool subset, `submit`s with a `requestId` derived from `api.taskId` (so a crash rerun finds the same subagent), waits, reads `AssistantEntry` for the answer. `api.details({conversationId})` links it for UIs.

## Recipes

- **Wait for a person**: a task phase commits `{status:"waiting", on:[approvalTask]}`; an approval tool/route resolves it. `hook(ToolTask,{beforeTool})` + `api.memo` gates dangerous tools (durable-pi's `jev-damage-control` extension judges each bash command and blocks destructive ones).
- **Fork**: `channel.fork(entryId, {ownership:{kind:"ownerless"}}, ctx)` — child sees parent history to that point without copying; both run concurrently.
- **Multi-client steering**: clients get a snapshot then deltas (`viewState`/`watch()` ops, or `watchEvents()` for coding-agent-style events). Any client can `submit` with `whenBusy: "steer" | "followUp" | "reject"`.
- **Self-handoff agent**: tool returns `control: { handoff: note }` + queues a `requestId:"handoff:<taskId>"` "Continue." submission.
- **Change code under a running agent**: `registry.install()` with the same extension name replaces it atomically; in-flight calls finish on old code.

## The Durable Agent Network (durable-pi repo)

`dpi-backend` wraps the harness in an authenticated HTTP+SSE API (`:4780`); clients are thin (web hub `:4781`, `dpi-tui`, curl). **The device owns the run; clients only watch and steer.**

Key API (`Authorization: Bearer <DPI_TOKEN>`, error shape `{error:{code,message}}`):

- `POST /v1/sessions` `{name,cwd,model:{provider,modelId},thinkingLevel?}` → a session = one harness + one agent DB
- `POST /v1/sessions/:s/agents/:a/messages` `{content, requestId, whenBusy?, waitMs?}` → send and optionally wait (≤300s) for the answer
- `POST .../agents/:a/fork` `{entryId}`; `PATCH .../agents/:a/config` (`model`, `readOnly`, `instructions`); `POST .../abort`; `POST .../compact`; `POST .../reset`
- Streams (SSE, snapshot→deltas, reconnect = fresh snapshot): `/v1/events` (device), `/v1/sessions/:s/events`, `.../agents/:a/events`
- `GET .../entries?scope=all|active`, `GET .../transcript?format=markdown`

`dpi-sdk` (`DpiClient`, `SessionHandle`, `AgentHandle`, `follow()`, `AgentModel`) is the typed contract — import by relative path, not published.

**Durable jobs** (`jobs.ts`): repeatable workflows as pi-durable tasks — named phases become checkpoint phases; a run record lives in a `demo.job-runs` document; approvals pause durably. Built-in: `workspace-snapshot`, `release-check` (gather→approve→publish), `change-watch`. Agent-facing tools: `list_durable_jobs`, `run_durable_job`, `check_durable_job`, `search_history`, `subagent`, `start_session`, `handoff`.

**Proactive pattern** (`proactive-dpi-demo`): watcher sends every event to a cheap judge (Jev, ~$0.00003/row); **code** counts threshold verdicts in a window and only then opens an incident + starts the agent; the agent investigates and fixes, then waits durably for human approval to deploy. Layering rule: code decides thresholds/side effects, model decides bounded judgments, agent does open-ended work.

Ops: `just net up|status|logs|restart|deploy <id>` per device in `dpi-network.yaml` (`local` | `ssh` | `exe-dev` | `url`); tokens live in `~/.dpi/backend.env`, registry in `~/.dpi/network.sqlite`.

## Live mutation and control (from the Durable Pi talk)

Everything about a running agent is changeable mid-run — the demo flips `readOnly` on and the agent's tool list shrinks between calls; toggling the `system-prompts` extension changes its prompt; enabling `jev-damage-control` makes Jev judge every bash command before it runs (it blocked `rm -rf` as irreversible, score 0.99). Forking a live agent creates an independent branch at any entry. Design for this: agents should tolerate tools/prompts changing between calls, and safety gates belong in hooks/extensions, not in prompts alone.

## Adoption posture (video's framing)

- Default to durable agents for **product work** (support bots, anything users reconnect to) — thin clients over one agent API (web, TUI, curl, mobile).
- **Proactive agents** are the target shape: the agent works before you show up, then waits durably for approval — "human near the loop" rather than fully out-of-loop.
- Cheap-model judge in front of the stream (Jev: pennies for thousands of rows) + code thresholds = the trigger; the durable agent is the responder.
- KPI: how many agents are doing useful work for you at any point in time.

## Pitfalls

- Always pass `requestId`/`Idempotency-Key` — never rely on "submit once" logic client-side.
- Don't mark mutating tools `replay:"safe"`.
- `whenBusy` defaults to `followUp` — use `"steer"` to interject mid-run, `"reject"` to fail fast (409 `busy`).
- Closing a stream or client never stops work; abort explicitly.
- SQLite storage keeps only the working set in memory; trust it — don't load whole transcripts.
