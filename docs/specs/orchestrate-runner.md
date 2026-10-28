---
spec_id: orchestrate-runner
triage: ready-for-agent
status: settled
date: 2026-10-15
author: Gab
target_project: any Ticket Runner target project
skill_destination: shared agent skills catalog (`.agents/skills/orchestrate-runner/`)
prerequisites:
  - provider resolved to `antigravity` (locked per T013 / AGENTS.md)
  - `test_cmd` resolved to YAML-integrity one-liner (validates `*/agents/openai.yaml` parses)
  - `.agent/` runtime directories present and writable in target project
  - `ticket-runner.yaml` overlay present and loadable
  - `doctor --local-only` passes before any `start` invocation
design_settled:
  - Round 1: role=notetaker+recommender; channels=all Runner emissions; artifact=per-ticket MD; timing=dedicated future session; debug=per-ticket MD + gotchas.md append + aggregate debug-report; reuse=Skill file
  - Round 2: prediction=two-layer (intent + predicted artifacts); citation=causal-chain; pass=full template with condensed Debug Notes; escalation=MD + Discord/TUI alert on critical + JSONL events; session=whole-queue with context-aware stop; continuity=gotchas.md + handoff; skill name=`orchestrate-runner`
  - Round 3: provider=`antigravity`, test_cmd=YAML one-liner, invocation=pre-flight + continuous + post-mortem, context-stop=per-ticket default with mid-queue handoff
related_invariants:
  - Worker↔Runner messages are durable JSON, never parsed from stdout (ADR 0004)
  - Token budget thresholds 120k/135k/150k
  - Only Gatekeeper commits; Worker never commits directly (ADR 0002)
  - `antigravity` is the locked worker provider (T013, AGENTS.md)
---

# orchestrate-runner

## Problem Statement

The user operates Ticket Runner — a Python autonomous ticket executor that reads a target project's codebase and implements tickets against it. Across multiple spec queues, the user has observed things "going in the wrong direction" anecdotally but lacks a structured way to:

- Capture what the Runner is *actually* doing versus what it is *supposed* to do.
- Diagnose *why* a run deviated at any phase (Worker, Gatekeeper, Evidence Card, commit).
- Build cumulative institutional knowledge across queues, so recurring failure modes don't get re-learned per queue.
- Cut latency on critical failures — otherwise the human notices too late (commit-time Evidence Cards arrive after the damage is already done).

Without an orchestration layer, every debugging session starts from zero. Recurring failure modes (provider mismatch, malformed signals, stuck suites, scope creep) get re-discovered per queue. Critical failures go unnoticed until Evidence Cards.

## Solution

`orchestrate-runner` is a Skill in the user's shared agent skills catalog that **orchestrates the experience of running Ticket Runner**. It is passive about Runner actions (never intervenes — Worker↔Runner messages are durable JSON, never parsed from stdout — ADR 0004) but active about coordinating the human's experience of the run.

The Orchestrator conducts three phases:

- **Pre-flight** — verifies provider on PATH, `test_cmd` exits 0 on a known-good fixture, `.agent/` writable, overlay loadable, queue present. Writes a brief of predicted behavior before runtime.
- **Continuous** — watches `.agent/` at each phase boundary (Worker ready signal, Gatekeeper verdict, Evidence Card, commit) and writes incremental per-ticket MD updates with two-layer predictions, observation capture, verdicts, and causal-chain recommendations.
- **Post-mortem** — aggregates per-ticket MDs into `observations/debug-report.md`. Appends recurring failure modes (2+ occurrences in a queue) to the project's `docs/tickets/gotchas.md` for forward inheritance by future Workers.

The Orchestrator is packaged as a Skill file at `.agents/skills/orchestrate-runner/SKILL.md`, written under writing-for-agents discipline (terse, leading-word-anchored, completion-criteria-driven, reference disclosed behind pointers where appropriate).

The single seam is filesystem-level: Orchestrator reads Runner emissions from the target project's `.agent/` directory (the canonical, allowlisted emission surface declared in `runner/domain/runtime_paths.py`). No Runner API, no ports, no CLI flag, no Runner source modifications.

## User Stories

1. As a Ticket Runner operator, I want a reusable orchestration Skill in my shared agent skills catalog, so that I can apply the same diagnostic discipline to any target project without rebuilding the workflow.
3. As an operator before a run, I want the Orchestrator to verify preconditions (provider on PATH, test_cmd exits 0 on a known-good fixture, `.agent/` writable, overlay loadable, queue present), so that I catch config drift before `start`.
4. As an operator before a run, I want a pre-flight brief stating predicted outcomes per phase (Worker intent + predicted artifacts), so that I have a baseline for comparison.
5. As an operator during a run, I want the Orchestrator to read all `.agent/` emission channels (signals, evidence, logs, state, git) at each phase, so that no failure context is lost.
6. As an operator when a phase completes, I want a two-layer observation: intent-level diff plus artifact-level diff, so that I can see both what was meant and what landed.
7. As an operator when a phase fails, I want a causal-chain recommendation (evidence → debugging principle → proposed fix) with file/line/signal-key references, so that I can decide whether to act without chasing citations.
8. As an operator when a critical failure occurs, I want a low-latency Discord/TUI alert (not just a buried MD), so that I notice without polling.
9. As an operator after a ticket completes, I want a per-ticket MD with fixed headings (Prediction, Observation, Verdict, Debug Notes, Recommendations), so that I can grep across tickets for patterns.
10. As an operator on PASS, I want the same MD shape with condensed Debug Notes ("no anomalies; observed matched predicted"), so that PASS is provably watched (not silently absent).
11. As an operator when context approaches limits, I want the Orchestrator to stop at a phase or ticket boundary, write a handoff MD, and let a fresh session resume, so that the run is never lost to context exhaustion.
12. As an operator at the end of a queue, I want an aggregate `observations/debug-report.md` summarizing pass/fail counts, recurring failure modes, and recommendations not yet applied, so that I see systemic vs one-off issues at a glance.
13. As an operator across spec queues, I want recurring failure patterns (2+ occurrences in a queue) appended to the project's gotchas log, so that future Workers inherit them via the Runner's spec-excerpt injection.
14. As an operator using Human-in-the-Loop mode, I want Orchestrator notes to complement — not replace — Evidence Cards, so that the human approval gate remains the authority on commits.
15. As an operator post-run, I want to read Orchestrator MDs without needing the Orchestrator's session context, so that observations are durable artifacts, not session-locked notes.
16. As an operator resuming after handoff, I want the Orchestrator to cite its own handoff MD as the resume point, so that a fresh session picks up exactly where the previous left off.
17. As an operator when the test suite hangs or cascades, I want the Orchestrator to flag the Stuck-Test-Suite pattern and recommend invocation of the diagnosing-bugs protocol, so that triage follows the established discipline.
18. As an operator when the circuit breaker trips, I want the Orchestrator to capture the three failed attempts with phase-level diagnostics and recommend ticket scope re-evaluation, so that I can decide whether the underlying issue is Worker, Gatekeeper, test_cmd, or ticket scope.
19. As an operator when the configured provider differs from PATH reality (e.g., overlay says `antigravity` but `agy` is missing), I want the Orchestrator to flag this in pre-flight before `start`, so that runtime never surprises me.
20. As an operator across many queues, I want the Orchestrator's MDs to be parseable as JSON-shaped data (sections are fixed headings), so that I can write small tools to aggregate over them later.

## Implementation Decisions

- **Single seam at the filesystem level.** Orchestrator reads Runner emissions from the target project's `.agent/` directory (the canonical, allowlisted emission surface in `runner/domain/runtime_paths.py`). No Runner API, no ports, no CLI flag.
- **Skill format under writing-for-agents discipline.** `orchestrate-runner/SKILL.md` in the shared agent skills catalog. Frontmatter uses a leading-word trigger ("orchestrate"); body is terse, with steps carrying explicit completion criteria and reference disclosed behind pointers where appropriate (e.g., runtime-paths allowlist lives in Runner source and is referenced, not inlined).
- **Three-phase invocation: pre-flight, continuous, post-mortem.**
  - **Pre-flight** runs before `doctor --local-only` (or alongside it) and verifies: provider on PATH, `test_cmd` exits 0 on a known-good fixture, `.agent/` writable, `ticket-runner.yaml` loads, queue present. Writes a brief of predicted behavior.
  - **Continuous** watches `.agent/` at each phase boundary (Worker emits ready, Gatekeeper verdict, Evidence Card, commit) and writes incremental MD updates.
  - **Post-mortem** writes `observations/debug-report.md` aggregating per-ticket MDs; appends recurring failures to gotchas.md.
- **Multi-channel capture.** Orchestrator reads: ready signals (`.agent/signals/{ticket}_ready.json`), question signals (`.agent/questions/{ticket}.json`), evidence artifacts (`.agent/evidence/{ticket}/*`), worker JSONL telemetry (`.agent/logs/{ticket}_session_{id}.jsonl`), stderr sidecars (`.agent/logs/{ticket}_session_{id}.stderr.log`), diagnostic MDs (`.agent/logs/{ticket}_diagnostic.md`), handoff MDs (`.agent/checkpoints/{ticket}/handoff.md`), state/status JSON (`.agent/state.json`, `.agent/status.json`), git log/reflog/status, and live provider output when accessible.
- **Per-ticket artifact structure.** Each observed ticket gets one MD at `<target_project>/observations/<ticket_id>.md` with fixed headings: **Prediction** (two-layer: intent + predicted artifacts), **Observation** (what actually happened), **Verdict** (PASS / FAIL / INCONCLUSIVE with reason), **Debug Notes** (FAIL: full causal-chain; PASS: condensed to "no anomalies"), **Recommendations** (only when Debug Notes is non-empty).
- **Two-layer prediction.** Top layer: phase intent (e.g., "Worker adds 8 `agents/openai.yaml` files matching schema X"). Bottom layer: predicted artifacts ("→ `.agent/signals/T101_ready.json` with `status: ready`", "→ Gatekeeper exit 0 on `test_cmd`").
- **Causal-chain citation.** Every recommendation traces: evidence (file path + line + signal key) → debugging principle → proposed fix. The Orchestrator never cites without reasoning.
- **PASS handling.** Full template (Prediction/Observation/Verdict) with condensed Debug Notes ("no anomalies; observed matched predicted"). Same documentation rigor as FAIL; PASS is provably watched.
- **Failure escalation.**
  - **Always**: write to per-ticket MD.
  - **On critical**: emit Discord/TUI alert via active presence channel (existing presence infrastructure: `runner/adapters/discord/`, `runner/adapters/ui/`).
  - **Always**: append JSONL line to `observations/<ticket_id>_events.jsonl` for downstream tooling.
- **3-value verdict convention.** **PASS** (observed matches predicted; phase advances cleanly), **FAIL** (observed diverges; phase does not advance), **INCONCLUSIVE** (insufficient evidence; flagged for human review).
- **gotchas.md accumulation rule.** A failure mode appearing 2+ times in a queue is appended to `<target_project>/docs/tickets/gotchas.md` with citations to both occurrences. Future Workers see these via the Runner's `--spec-excerpt` injection.
- **Context-aware stop.** Per-ticket stop is the default; a session may span a whole queue if context allows. Mid-queue context rise triggers a handoff MD at the next phase or ticket boundary. Standard budget applies: warn at 120k, handoff at 135k, hard stop at 150k (matches AGENTS.md token budget).
- **Handoff discipline.** When context crosses 135k mid-queue, Orchestrator writes `<target_project>/observations/handoff-<ticket_id>.md` summarizing state, predictions, and resume point. Fresh session resumes from this file.
- **Standalone debug report.** `observations/debug-report.md` aggregates: pass/fail counts, recurring failure modes (with frequency), recommendations not yet applied, links to per-ticket MDs.
- **Session boundary.** Orchestrator runs in a dedicated session. Mixing with planning, grilling, or spec-writing flows is discouraged.
- **No provider interception.** Orchestrator reads post-hoc from `.agent/logs/*.jsonl` rather than attaching to a live provider process. Preserves ADR 0004.
- **No Discord integration for raw output.** Discord is for *alerts* (critical failures), not for the full MD stream. MD output stays local.
- **Decoupled from Runner version.** Orchestrator reads the canonical `.agent/` paths declared by `runtime_paths.py` but does not import the Runner. When those paths change, Orchestrator updates its read paths; the change happens at the Runner's mutable boundary, not in the Skill.
- **Provider: `antigravity`.** Locked per T013 / AGENTS.md. Overlay at `<target_project>/ticket-runner.yaml`: `worker.provider: antigravity`. Invocation: `agy -p "<prompt>" --output-format stream-json [--conversation <uuid>]`. `agy.exe` confirmed at `C:\Users\Acer\AppData\Local\agy\bin\agy.exe`.
- **test_cmd: YAML-integrity one-liner.** `python -c "import glob,yaml; fs=glob.glob('*/agents/openai.yaml'); [yaml.safe_load(open(f,encoding='utf-8')) for f in fs]; print('validated %d openai.yaml files' % len(fs))"`. Empty `build_cmd`. Verified exiting 0 against the 37-file pre-T101 state; fails on malformed YAML.
- **Output destinations.**
  - Per-ticket MDs: `<target_project>/observations/<ticket_id>.md`
  - Standalone debug report: `<target_project>/observations/debug-report.md`
  - JSONL events: `<target_project>/observations/<ticket_id>_events.jsonl`
  - Handoff MDs: `<target_project>/observations/handoff-<ticket_id>.md`
  - All under `observations/`, gitignored (treated as runtime).

## Canonical First Run Sequence

The first end-to-end exercise of `orchestrate-runner` against `D:\Repos\agent-skills` (T101 → T102). Doubles as the canonical order-of-operations template for future target projects and spec queues — copy and adapt the ticket IDs and overlay specifics.

### Phase 0 — Pre-flight blockers (settled in this design session)
- Provider: `antigravity` (locked per T013 / AGENTS.md)
- test_cmd: YAML-integrity one-liner (validates `*/agents/openai.yaml` parses)
- build_cmd: empty string (no build step)
- Branch: `agent/ticket-runner`

### Phase 1 — Environment verification (operator-driven, non-destructive)
Before invoking the Skill:
1. Branch check: `git -C D:\Repos\agent-skills rev-parse --abbrev-ref HEAD` → expect `agent/ticket-runner`
2. Provider on PATH: `Get-Command agy` → expect `C:\Users\Acer\AppData\Local\agy\bin\agy.exe`
3. `.agent/` writable: `Test-Path D:\Repos\agent-skills\.agent -PathType Container`; confirm gitignored via `git -C D:\Repos\agent-skills check-ignore .agent`
4. Overlay loads: `python -c "import yaml; yaml.safe_load(open(r'D:\Repos\agent-skills\ticket-runner.yaml'))"`
5. Tickets present: `Get-ChildItem D:\Repos\agent-skills\docs\tickets\01-runner-live-test-bootstrap` → expect T101 + T102
6. Working tree expected untracked: `git -C D:\Repos\agent-skills status --short` → expect `?? docs/` + `?? ticket-runner.yaml`

### Phase 2 — Doctor (non-destructive)
- Command: `py -3 D:\Projects\ticket_runner.py --project-dir D:\Repos\agent-skills doctor --local-only`
- Expected: passes without errors
- On fail: STOP; do NOT proceed to start. Triage via `diagnosing-bugs` per the Stuck-Test-Suite Protocol if applicable.

### Phase 3 — Spec publishing (operator-driven, one-time)
- Move `orchestrate-runner.md` from `C:\Users\Acer\.opencode\plan\` to the canonical specs home (e.g., `D:\Repos\agent-runner-driver\docs\specs\orchestrate-runner.md` or `D:\Repos\agent-skills\docs\specs\orchestrate-runner.md`).
- Apply `ready-for-agent` triage label in the issue tracker (per to-spec convention).
- Record the move in `STATUS.md` under the next touchpoint.

### Phase 4 — Orchestrator pre-flight (Skill-driven)
The Skill is invoked. It then:
1. Verifies `agy.exe` on PATH matches overlay (precondition for antigravity provider).
2. Runs the configured `test_cmd` against the current `.agent/` (sanity check; expected exit 0 on the 37-file pre-T101 state).
3. Verifies `.agent/{signals,questions,checkpoints,evidence,logs}/` directories exist and are writable.
4. Verifies `ticket-runner.yaml` loads via the Runner's own `YamlConfigLoader` (catches drift between operator-driven Phase 1 and what the Runner actually sees).
5. Verifies queue file structure: T101 unblocked, T102 `Blocked by: T101`.
6. Writes `observations/T101_prediction.md` and `observations/T102_prediction.md` with two-layer predictions (intent + predicted artifacts).
7. If any check fails: STOP. Do NOT proceed to `start`. Write a Phase-4 FAIL MD explaining which precondition failed.

### Phase 5 — Ticket T101 run
1. Operator invokes: `py -3 D:\Projects\ticket_runner.py --project-dir D:\Repos\agent-skills start --local-only` (or via driver: `D:\Repos\agent-runner-driver\scripts\start.ps1 --local-only`).
2. Runner picks T101 (no blockers).
3. Worker (`antigravity`) implements T101.
4. Worker writes `.agent/signals/T101_ready.json`.
5. Gatekeeper runs `test_cmd` (YAML one-liner) — expected exit 0.
6. Evidence Card presented via active presence channel.
7. Human approves in HITL mode.
8. Gatekeeper authors one commit combining code, ticket relocation, and any gotchas logged in that ticket's session.
9. Orchestrator observes each step and writes incrementally:
   - Appends signal event to `observations/T101_events.jsonl`
   - Updates `observations/T101.md` (Observation, Verdict, Debug Notes, Recommendations as phases complete)
   - On phase divergence: causal-chain Debug Note + Recommendation block
   - On critical divergence: Discord/TUI alert via active presence channel

### Phase 6 — Ticket T102 run
Same protocol as Phase 5, with T102 scope (modifies `README.md`). T102 was `Blocked by: T101`; after T101 commit, T102 is unblocked and Runner picks it up automatically.

### Phase 7 — Orchestrator post-mortem (Skill-driven)
After T102 commit:
1. Skill writes `observations/debug-report.md`:
   - Pass/fail counts (expected: 2 PASS, 0 FAIL on a clean first run)
   - Recurring failure modes (expected: none on a clean first run)
   - Recommendations not yet applied (cumulative across both tickets)
   - Links to per-ticket MDs
2. If a failure mode appeared 2+ times in the queue: append a single consolidated entry to `docs/tickets/gotchas.md` with citations to both occurrences.
3. If context crossed 135k mid-run: handoff MD already exists at `observations/handoff-<ticket_id>.md`; post-mortem cites it as the resume point for future sessions.
4. Skill emits a final summary via the active presence channel: pass/fail counts, link to debug-report.md.

### Phase 8 — Human verification (separate from Orchestrator)
- Operator invokes `/live-qa` (per AGENTS.md HITL discipline).
- Verifies against T101/T102 acceptance criteria + smoke scenarios (per ticket frontmatter).
- Records verdict in `.agent/live-qa_log_01-runner-live-test-bootstrap.md`.
- The Orchestrator does NOT write the live-qa log; only the human does. The two artifacts are complementary, not redundant.

### Phase 9 — Cleanup and archival
- Tickets auto-relocate to `completed/` per Runner convention (driven by Gatekeeper on commit; per spec template's "Status: completed" rule).
- Future spec queues follow the same Phase 1–9 sequence, parameterized by ticket IDs, overlay, and queue path.

### Failure escalation during the sequence
| Trigger | Orchestrator action |
|---|---|
| Phase 1 environment check fails | Halt before Phase 2; recommend `diagnosing-bugs` if path/PATH issues |
| Phase 2 Doctor fails | Halt before Phase 4; recommend `diagnosing-bugs` for the specific failure |
| Phase 4 pre-flight fails | Halt before Phase 5; write Phase-4 FAIL MD citing the failed precondition |
| Worker fails to emit ready signal | Record FAIL + Recommendation in T-ticket MD; human decides retry |
| Gatekeeper fails test_cmd | Record FAIL + cite Gatekeeper-side evidence (`.agent/logs/{ticket}_diagnostic.md`) |
| Stuck test suite | Flag Stuck-Test-Suite pattern; recommend `diagnosing-bugs` Stuck-Test-Suite Protocol |
| Circuit breaker trips (3 fails) | Capture failure history with phase-level diagnostics; recommend ticket scope re-evaluation |
| Token context > 135k mid-queue | Write `observations/handoff-<ticket_id>.md`; fresh session resumes |
| Token context > 150k (hard ceiling) | Stop; write handoff; refuse to continue; human decides |

## Testing Decisions

The Orchestrator is a process discipline, not a software module. Tests are demonstrations: run the Orchestrator against a known-good run, verify MD shape is correct, verify the prediction was reasonable, verify recommendations trace to evidence with citations.

The list of demonstration scenarios below is guidance for the Orchestrator implementation. The implementation may add additional scenarios at its discretion — testing approach is AI-decided, not grill-decided — so long as each new scenario keeps the discipline (predictions match observed, citations trace to evidence, recommendations propose a fix the human can apply).

**Prior art in the skills catalog.** `live-qa`, `smoke-fail`, `diagnosing-bugs`, `handoff` are all diagnostic or recovery protocols. The Orchestrator follows the same conventions: durable MD artifacts, named cross-references, skill pointers instead of inlined reference.

**Demonstration scenarios:**

- **Happy path (PASS)**: Worker succeeds, Gatekeeper passes, commit lands — Orchestrator captures full template with condensed Debug Notes ("no anomalies; observed matched predicted").
- **Pre-flight catches config drift**: Overlay says `antigravity` but `agy` is missing on PATH — pre-flight flags before `start`. (Or: `test_cmd` exits 5 on a known-good fixture — pre-flight flags.)
- **Pre-flight brief written**: Before `start`, `observations/<ticket_id>.md` exists with Prediction section populated but Observation empty.
- **Worker self-reports false positive**: Worker emits ready signal but commit doesn't pass — Observation captures the divergence, Debug Notes flags Worker-side bug.
- **Gatekeeper false-pass**: Commit lands but evidence artifacts missing — Observation captures the Gatekeeper bug.
- **Critical alert**: Gatekeeper fails on the first phase — Discord/TUI alert fires immediately; events.jsonl records the alert.
- **Stuck suite**: Tests hang or cascade — Orchestrator captures timeout pattern, recommends `diagnosing-bugs` Stuck-Test-Suite Protocol.
- **Circuit breaker trip**: Three failed attempts — Orchestrator captures failure history with phase-level diagnostics, recommends ticket scope re-evaluation.
- **Gotcha accumulation**: Same failure mode twice in a queue — Orchestrator appends to gotchas.md with citations to both occurrences.
- **Handoff at 135k**: Mid-queue context rise — Orchestrator writes `handoff-<ticket_id>.md`; fresh session resumes and cites the handoff.
- **Hard stop at 150k**: Context ceiling reached — Orchestrator stops, writes handoff, refuses to continue; human decides.
- **Post-mortem aggregation**: Queue completes — `debug-report.md` shows pass/fail counts, recurring modes, recommendations not yet applied, links to per-ticket MDs.
- **PASS provability**: PASS MD contains all four sections, Debug Notes explicitly states "no anomalies" — distinguishable from a missing Orchestrator session.

## Out of Scope

- Modifying the Runner source (`runner/`). The Orchestrator is decoupled.
- Adding a Runner CLI flag such as `--orchestrate`. The Orchestrator is an external Skill.
- Live provider interception (attaching to a live provider stdout/stderr). The Orchestrator reads post-hoc from durable artifacts.
- Discord integration for raw MD output. Discord is alert-only.
- Spec-closing alignment audits. That is a separate ticket type, not part of this Orchestrator.
- **Provider choice reversal**: T013 locked `antigravity`; this spec does not reopen that decision.
- **Evidence Card replacement**: Orchestrator complements Evidence Cards; Evidence Cards remain the human approval gate on commits.
- **Automated remediation**: Orchestrator recommends; the human applies or rejects.

## Further Notes

- **First invocation target.** `D:\Repos\agent-skills` with T101/T102, in a dedicated session AFTER `doctor --local-only` passes. provider=`antigravity`, test_cmd=YAML one-liner.
- **Cluster-layout discipline.** Orchestrator reads Runner state from the target project; it does not touch the Runner source repo. The Skill file lives in the shared agent skills catalog, separate from both.
- **Token budget for Orchestrator sessions.** A small queue (2-3 tickets) fits comfortably under the warn/handoff/hard ceiling thresholds (120k/135k/150k). Larger queues may require per-ticket Orchestrator sessions or summarized cross-ticket reporting to keep the session inside budget.
- **Skill file structure (preview).** The Skill will follow writing-for-agents discipline. It will have:
  - Frontmatter description with a leading-word trigger that matches invocation phrases ("orchestrate the Runner", "conduct a Runner run", "watch this queue").
  - Steps with explicit completion criteria (e.g., "for each phase: predict two-layer, observe, verdict, debug if FAIL, recommend with causal chain").
  - Reference disclosed behind pointers. The runtime-paths allowlist lives in the Runner source; the Skill points to it rather than inlining it.
  - A single source of truth per concept. The MD template, the verdict convention, and the gotcha-append rule each appear once.
- **Companion specs that may follow.** (a) Runner instrumentation hooks — if `.agent/` paths prove insufficient for debugging, a follow-up spec could add structured Runner events without breaking the JSON-only invariant. (b) Discord observer channel — if alerts prove noisy in HITL mode, a follow-up could post summaries to a Discord thread. (c) Cross-queue prediction memory — if gotchas.md proves insufficient for institutional knowledge, a follow-up could add a dedicated predictions store. All deferred.
- **Triage label.** `ready-for-agent` — this spec is complete and ready to be turned into the Skill file by an agent in a future implementation session. The implementation session itself is out-of-scope for this spec; the spec captures *what* the Skill must do, not *how* to author it.
- **Order of operations after exiting Plan mode.** See **Canonical First Run Sequence** above. The high-level summary: (1) resolve blockers (already done), (2) Phase 1 environment verification, (3) Phase 2 Doctor, (4) Phase 3 spec publishing, (5) Phase 4 Orchestrator pre-flight, (6) Phase 5–6 ticket runs with continuous observation, (7) Phase 7 post-mortem + debug-report, (8) Phase 8 human `/live-qa`, (9) Phase 9 cleanup. Future spec queues follow the same sequence with new ticket IDs and overlay specifics.

## Next Session Prompt

Copy-paste this to start the execution session. It assumes the settled decisions in this spec are not re-litigated.

```
Resume the `orchestrate-runner` first-run sequence. Read
`C:\Users\Acer\.opencode\plan\orchestrate-runner.md` for the spec and
`D:\Projects\STATUS.md` for project state.

Settled decisions (do not re-grill): provider=`antigravity`,
`test_cmd`=YAML-integrity one-liner, branch=`agent/ticket-runner`,
target=`D:\Repos\agent-skills` (T101 → T102).

Execute the Canonical First Run Sequence (Phases 1–9 per the spec):
- Phase 1: environment verification (operator-driven, non-destructive)
- Phase 2: `py -3 D:\Projects\ticket_runner.py --project-dir D:\Repos\agent-skills doctor --local-only`
- Phase 3: move spec to canonical home
- Phase 4: invoke `orchestrate-runner` pre-flight
- Phase 5–6: `start --local-only` (T101 then T102, HITL approval at each Evidence Card)
- Phase 7: post-mortem + `observations/debug-report.md`
- Phase 8: `/live-qa` (human verification)
- Phase 9: cleanup

Pause for human approval at each Evidence Card. Stop and surface any
phase failure rather than retrying silently. Refresh `STATUS.md` at the
end.
```

A fuller handoff (with Phase 0 overlay edit, exact verification commands, and environment facts) lives at `C:\Users\Acer\AppData\Local\Temp\opencode\handoff-orchestrate-runner-first-run.md`.