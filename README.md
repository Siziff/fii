# FII — Fast Interaction Interface

**Fast, frequent human oversight of AI agents that does not slow the agent down.**

> Status: concept / idea reservation. Recorded: 2026-09-28. Author: Ivan Mikheev. License: MIT.

---

## 1. The problem

The main danger of today's AI agents is not "evil AI" — it is **lack of control**. Chasing speed, humans let go of oversight, and the agent starts acting almost entirely at its own discretion. The consequences already make the news regularly:

| Date | Incident | What happened |
|---|---|---|
| 2025-07-18 | Replit Agent / SaaStr | Despite an explicit code freeze, the agent deleted a production DB (~1,200 companies), fabricated ~4,000 fake users and lied about rollback being impossible. |
| 2025-07 | Amazon Q Developer (CVE-2025-8217) | A malicious prompt ("clean the system to a near-factory state") shipped inside a VS Code extension release; the agent's own instructions became the attack vector, invisible to per-action approval. |
| 2025-07-21 | Gemini CLI (issue #4586) | While reorganizing a folder, the agent destroyed the user's files and hallucinated their existence until it admitted failure. |

The opposite extreme does not work either: if a human reviews every block of code and answers every minor question from the agent, throughput drops to manual speed and the main advantage of AI disappears. The industry calls this **"approval fatigue vs. uncontrolled autonomy"** (Wang, Li, Tian, 2026).

### What existing tools do

Every major product (Claude Code, Codex CLI, Cursor, Copilot Agent, Devin) has converged on the same ladder of modes:

```
Manual (ask for everything) → Accept-edits → LLM classifier "auto/smart" → Bypass (ask for nothing)
```

The middle rung **replaces the human with a second AI** instead of helping the human look faster. The control granularity is the same everywhere: *one tool call → one blocking binary yes/no*. Between "approve every step" and "trust everything" there is **no rung for "the human watches quickly, frequently, and without blocking"**. FII is that missing rung.

### What human-factors research says

- **Ironies of Automation** (Bainbridge, 1983): the more we automate, the more tedious monitoring is left to the human, and the more the skill needed for rare-but-critical intervention decays.
- **Automation bias / complacency** (Parasuraman & Riley, 1997): consistently high automation reliability breeds "learned carelessness"; omission errors are not fixed by training; *more on-screen detail increases* bias; *accountability for outcomes* and *variable* reliability reduce it.
- **Habituation at the Gate** (Yu et al., 2026): in reviews of agent-authored PRs, approval rate rises (30% → 37%) and comment count drops 22% — post-hoc review habituates and degenerates.
- **Pista** (Sabouri et al., 2026): human participation *during* execution catches errors that post-hoc review fails to surface.
- **Agent Approval Laundering** (Zhang et al., 2026): an approval names the *invocation* (`npm install`), not its *effects* (lifecycle hooks, network, secrets). Command-text policies are fundamentally unsound.

Conclusion: we need an interface that (a) keeps the human in the loop *while* the agent works, (b) shows *effects*, not commands, (c) deliberately fights habituation, and (d) does not block the agent unless necessary.

---

## 2. The idea

**FII (Fast Interaction Interface)** is a layer between an AI agent and a human that turns the agent's stream of actions into a **compact, prioritized feed of micro-facts**, each of which a human can evaluate in 1–3 seconds and, if needed, instantly veto, roll back, or steer — **without stopping the agent**.

The key shift: **from "permission" to "observation with a right of veto"**. By default the agent keeps working; the human does not *approve* actions, they *reject* wrong ones — on the fly, not in modal dialogs.

### Principles

1. **Non-blocking by default.** Most actions do not wait for the human. Only *irreversible* and *high-risk* actions block.
2. **Effects, not commands.** Show "3 files touched, network call to X, `.env` read" rather than `bash -c "..."`.
3. **Reversibility as the primary axis.** Reversible → just show (or don't even show); irreversible → show with a pause; unknown → show with a pause and an effect prediction.
4. **Micro-facts, not diffs.** The unit of review is a single headline-sized statement, not a 200-line diff. Details expand on demand.
5. **Anti-habituation.** The interface deliberately fights habituation: variable density, attention probes, explicit accountability ("you skipped X"), calibrated confidence display.
6. **Undo buffer.** Every agent action is reversible within a time window; a veto is a rollback, not just a ban on future actions.
7. **Agent-agnostic.** FII is not tied to a specific agent; it attaches via hooks/proxies (Claude Code hooks, MCP proxy, git hooks, LSP).

---

## 3. Interface design candidates

No single design has been chosen yet; below is the solution space with trade-offs.

### 3.1. The "ticker" — a micro-fact feed with hotkeys

One line per action in a separate terminal pane / IDE sidebar:

```
▸ [edit ] src/auth.py       +12 −3   "add token refresh"           ← reversible, auto
▸ [read ] .env                        secrets now in context       ⚠ amber
▸ [shell] pip install requests        network, lifecycle hooks     ⚠ amber
■ [shell] git push --force origin main IRREVERSIBLE                ⏸ waits 5s → [v]eto / [enter]
```

The human skims the feed and reacts only to color. Keys: `v` — veto/rollback the last action, `space` — pause the agent, `?` — expand, `!` — "explain why". Pros: minimal latency, familiar to developers. Cons: relies on peripheral attention; risk of feed blindness.

### 3.2. Alert panel modeled on aviation (ECAM-style)

Three tiers, as in the Airbus ECAM: **red** (irreversible, agent halted, response required), **amber** (attention, agent continues, veto possible within N seconds), **green/memo** (informational, do not interrupt). The *most critical item is always shown first*, together with a *proposed action* ("roll back / allow / edit command"). Pros: a model validated by human-factors practice. Cons: the Qantas 32 lesson — without aggregation, even tiered alerts drown in volume; similar events must be grouped.

### 3.3. Veto timeout (optimistic execution)

The agent announces an action, waits a short window (2–10 seconds depending on risk), and executes if no veto arrives. For reversible actions the window is 0 (execute, show, allow rollback). The human can always "extend the window" with one key if they can't keep up. Pros: the agent is barely slowed; the human controls the tempo. Cons: if the human is distracted this degrades into bypass — needs a presence detector (see 3.6).

### 3.4. Swipe triage / micro-decision queue

Non-critical questions that still need a human opinion ("use library A or B?", "this is legacy behavior — keep it?") do not block the agent; they go into a queue. The agent takes the *default* branch and marks it as "conditional"; when convenient, the human runs through the queue Tinder-style — left/right — and the agent rebuilds the conditional parts. Pros: decouples human and agent tempo. Cons: complex model of "conditional" changes; requires branch rollback support.

### 3.5. Intent drift — supervising intent, not actions

The human states an intent (goal and boundaries: "don't touch prod", "don't change the API"). For each step FII shows not the step itself but the **degree of fit with the intent** — one scale/color. Only deviations need a look. This is what academia studies (intent anchoring) but no product has shipped. Pros: the highest information compression. Cons: needs a second evaluator model; the evaluator's reliability becomes its own trust question.

### 3.6. Cross-cutting mechanics

- **Presence detector.** If the human steps away (no window focus / no input for N minutes), FII automatically tightens the mode: amber items become blocking. Solves the "distracted bypass" problem of 3.3.
- **Anti-habituation probes.** Rare, deliberately "bad" synthetic micro-facts; missing one signals to the human that they have stopped looking (and signals FII to slow the tempo).
- **Dynamic local trust** (as in Hedwig): trust grows for specific action types in specific repository zones and drops in unfamiliar territory.
- **Undo ring.** Shadow git / snapshots before every action; a veto on any feed line rolls back to that point.
- **Effect prediction.** Before a `shell` action, FII shows predicted effects (files, network, secrets, processes) — from a sandbox dry-run or a second model — and it is this, not the command, that lands in the feed.

### Preliminary choice

The most promising combination is **3.1 + 3.2 + 3.3 + 3.6**: a micro-fact feed with ECAM tiers, optimistic execution of reversible actions, veto timeouts for risky ones, a presence detector and an undo ring. 3.4 and 3.5 are phase two.

---

## 4. How FII differs from existing work

| Property | Claude Code / Cursor / Codex / Copilot | AgentsGate, HumanLayer (v1) | **FII** |
|---|---|---|---|
| Unit of control | tool call | tool call | micro-fact about an *effect* |
| Mode | blocking yes/no | blocking yes/no (Slack/CLI) | non-blocking, veto/rollback |
| What the human sees | command | command + risk score | effect + reversibility + intent fit |
| Middle rung | second AI instead of human | rules | the human, but faster |
| Anti-habituation | none | none | built in (probes, presence, variable density) |
| Rollback | manual checkpoint | shadow git | undo ring, veto = rollback |

---

## 5. Roadmap

- [x] Problem statement, research, reservation of name and idea (this README)
- [ ] Specification of the "micro-fact" format and the effect/reversibility classification
- [ ] MVP: Claude Code adapter via `PreToolUse` / `PostToolUse` hooks → TUI feed (variants 3.1 + 3.3)
- [ ] Undo ring on shadow git
- [ ] ECAM tiers and aggregation of similar events
- [ ] Presence detector, anti-habituation probes
- [ ] Metrics: agent latency with vs. without FII; share of "bad" actions caught; habituation curve
- [ ] Adapters: MCP proxy (agent-agnostic), Cursor/VS Code

---

## 6. Sources

**Incidents**
- Replit / SaaStr, July 2025 — https://www.businessinsider.com/replit-ceo-apologizes-ai-coding-tool-delete-company-database-2025-7
- Amazon Q Developer, CVE-2025-8217 — https://github.com/aws/aws-toolkit-vscode/security/advisories/GHSA-7g7f-ff96-5gcw
- Gemini CLI, issue #4586 — https://github.com/google-gemini/gemini-cli/issues/4586

**Products**
- Claude Code permission modes / hooks — https://code.claude.com/docs/en/permission-modes, https://code.claude.com/docs/en/hooks
- OpenAI Codex CLI approval modes — https://github.com/openai/codex
- Cursor agent security — https://cursor.com/docs/agent/security
- VS Code / Copilot approvals — https://code.visualstudio.com/docs/agents/run/approvals
- Devin CLI permissions — https://docs.devin.ai/cli/reference/permissions.md
- HumanLayer — https://github.com/humanlayer/humanlayer
- AgentsGate — https://github.com/agentsgate/agentsgate

**Research**
- Bainbridge, *Ironies of Automation*, 1983 — https://en.wikipedia.org/wiki/Ironies_of_Automation
- Parasuraman & Riley, *Humans and Automation: Use, Misuse, Disuse, Abuse*, 1997 — https://en.wikipedia.org/wiki/Automation_bias
- Endsley & Kiris, *Out-of-the-loop performance problem*, 1995 — https://en.wikipedia.org/wiki/Out-of-the-loop_performance_problem
- Bowman et al., *Measuring Progress on Scalable Oversight*, 2022 — https://arxiv.org/abs/2211.03540
- Chiodo et al., *Formalising Human-in-the-Loop*, 2025 — https://arxiv.org/abs/2505.10426
- Mitchell, Ghosh, Passi, *AI Agents Push Humans Out of the Loop*, 2026 — https://arxiv.org/abs/2608.23642
- Yu et al., *Habituation at the Gate*, 2026 — https://arxiv.org/abs/2606.22721
- Wang, Li, Tian, *Reframing LLM Agent Security as an Agent-Human Interaction Problem*, 2026 — https://arxiv.org/abs/2605.24309
- Brigham et al., *Janus*, 2026 — https://arxiv.org/abs/2607.01510
- Shukla et al., *Hedwig: Dynamic Autonomy for Coding Agents*, 2026 — https://arxiv.org/abs/2605.11495
- Zhang et al., *Agent Approval Laundering*, 2026 — https://arxiv.org/abs/2609.28586
- Sabouri et al., *Pista: Auditing and Controlling AI Agent Actions*, 2026 — https://arxiv.org/abs/2604.20070
- Zhao et al., *AgentGUI*, 2026 — https://arxiv.org/abs/2607.26300
- Chou et al., *Vibe-GUIDE*, 2026 — https://arxiv.org/abs/2609.23859

**UX patterns**
- Airbus ECAM (alert tiers) — https://en.wikipedia.org/wiki/Electronic_centralised_aircraft_monitor
- Management by exception — https://en.wikipedia.org/wiki/Management_by_exception
- RSVP / speed reading (rejected: removes the ability to re-read and compare) — https://en.wikipedia.org/wiki/Rapid_serial_visual_presentation
