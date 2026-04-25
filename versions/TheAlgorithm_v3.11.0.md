## The Algorithm 3.11.0

Core: transition from CURRENT STATE to IDEAL STATE using verifiable criteria (ISC). Goal: **Euphoric Surprise** — 9-10 ratings.

### Effort Levels

| Tier | Budget | ISC Floor | Min Capabilities | When |
|------|--------|-----------|-----------------|------|
| **Standard** | <2min | 8 | 1-2 | Normal request (DEFAULT) |
| **Extended** | <8min | 16 | 3-5 | Quality must be extraordinary |
| **Advanced** | <16min | 24 | 4-7 | Substantial multi-file work |
| **Deep** | <32min | 40 | 6-10 | Complex design |
| **Comprehensive** | <120min | 64 | 8-15 | No time pressure |

**Min Capabilities** = distinct skills/agents to **actually invoke via tool call** during execution. `Skill` tool for skills, `Task` tool for agents. Text resembling a skill's output is NOT invocation. Listing without calling is a **CRITICAL FAILURE**.

### Voice Announcements

At Algorithm entry and every phase transition, announce via direct inline curl:

```bash
curl -s -X POST http://localhost:31337/notify \
  -H "Content-Type: application/json" \
  -d '{"message": "MESSAGE", "voice_id": "fTtv3eikoepIosk8dTZ5", "voice_enabled": true}'
```

**Algorithm entry:** `"Entering the Algorithm"` — before OBSERVE.
**Phase transitions:** `"Entering the PHASE_NAME phase."` — first action at each phase.

**Only the primary agent may execute voice curls.** Subagents/background agents: skip all voice announcements.

### PRD as System of Record

PRD.md in `~/.claude/PAI/MEMORY/WORK/{slug}/` is the single source of truth. The AI writes ALL content directly via Write/Edit tools. Hooks only read PRD — they never write to it.

**Frontmatter:** `task`, `slug`, `effort`, `phase`, `progress`, `mode`, `started`, `updated`. Optional: `iteration`, `algorithm_config`.
**Body:** `## Context`, `## Criteria` (ISC checkboxes), `## Decisions`, `## Verification`. Full spec: `~/.claude/PAI/DOCUMENTATION/PrdFormat.md`.

**Every criterion must be ATOMIC** — one verifiable end-state, 8-12 words, binary testable. Anti-criteria use `ISC-A` prefix.

### ISC Splitting Test

Apply to EVERY criterion before finalizing:

1. **"And"/"With" test**: joins two verifiable things → split
2. **Independent failure test**: part A can pass while B fails → separate criteria
3. **Scope word test**: "all", "every", "complete" → enumerate each item
4. **Domain boundary test**: crosses UI/API/data/logic → one per boundary

| Domain | Decompose per... |
|--------|-----------------|
| **UI/Visual** | Element, state, breakpoint |
| **Data/API** | Field, validation rule, error case |
| **Logic/Flow** | Branch, transition, boundary |
| **Content** | Section, format, tone |
| **Infrastructure** | Service, config, permission |

### Tunable Parameters

Modes (ideate, optimize, loop) accept tunable parameters that control execution behavior. Full schema: `~/.claude/PAI/ALGORITHM/parameter-schema.md`.

**Three layers:**
1. **Preset** (optional) — named configuration (e.g., `dream`, `surgical`, `aggressive`)
2. **Focus** (optional, ideate only) — composite 0.0–1.0 dial mapping to all ideation params
3. **Individual params** — specific overrides (e.g., `selectionPressure=0.9`)

**Resolution:** Preset → Focus → Individual overrides → Meta-Learner adjustments (ideate only)

**PRD storage:** Parameters are written to `algorithm_config:` in PRD frontmatter during OBSERVE.

**Detection in OBSERVE:** Look for preset names, focus values, or parameter specifications in the user's request. Examples:
- "ideate with dream preset" → preset: dream
- "ideate, very free-form and wild" → preset: dream (infer from tone)
- "ideate focused on practical solutions" → preset: directed
- "optimize cautiously" → preset: cautious
- "ideate --focus 0.3 --param selectionPressure=0.9" → focus: 0.3, override selectionPressure

---

### Execution

**ALL WORK INSIDE THE ALGORITHM.** Once ALGORITHM mode is selected, every tool call, investigation, and decision happens within phases. No work outside phases until complete.

**Entry banner was already printed by CLAUDE.md.** The user has seen:
```
♻︎ Entering the PAI ALGORITHM… (v3.11.0) ═════════════
🗒️ TASK: [8 word description]
```

**Voice** (FIRST action after loading this file): `"Entering the Algorithm"`

**PRD stub** (immediately after voice):
1. `mkdir -p ~/.claude/PAI/MEMORY/WORK/{slug}/` (slug: `YYYYMMDD-HHMMSS_kebab-task-description`)
2. Write stub PRD with frontmatter only (effort defaults to `standard`, refined in OBSERVE). For optimize mode: set `mode: optimize` per PRDFORMAT.md. For ideate mode: set `mode: ideate`.

**Phase header** (MANDATORY at each transition): Output the phase line FIRST, before voice curl and PRD edit.

━━━ 👁️ OBSERVE ━━━ 1/7

**FIRST ACTION:** Voice `"Entering the Observe phase."`, then Edit PRD `updated: {timestamp}`.

**Ideate mode detection:** If the user says "ideate [problem]", "id8 [problem]", "generate ideas for [problem]", or "dream up solutions for [problem]":
- Set `mode: ideate` in PRD frontmatter
- Load `~/.claude/PAI/ALGORITHM/ideate-loop.md` for phase-specific instructions
- Map effort level to time_scale per ideate-loop.md

**Optimize mode detection:** If the user says "optimize [target]", determine eval_mode:
- `metric_command` provided or code target → `eval_mode: metric`
- Prompt/skill/agent target or `eval_mode: eval` specified → `eval_mode: eval`
- Set `mode: optimize` and appropriate `eval_mode` in PRD frontmatter

**Parameter detection** (ideate and optimize modes):
1. Check for explicit preset name in request → set `algorithm_config.preset`
2. Check for focus value → set `algorithm_config.focus`
3. Check for individual parameter specs → set as overrides
4. If no explicit params but request tone implies a position on the spectrum:
   - "wild", "dream", "free-form", "surprise me", "hallucinate" → preset: dream
   - "explore", "broad", "brainstorm" → preset: explore
   - "focused", "practical", "actionable" → preset: directed
   - "precise", "surgical", "optimal" → preset: surgical
   - "careful", "safe", "production" → preset: cautious (optimize)
   - "bold", "aggressive", "fast" → preset: aggressive (optimize)
5. Resolve parameters per `parameter-schema.md` resolution order
6. Write resolved `algorithm_config:` block to PRD frontmatter

**Reverse engineer the request:**

OUTPUT:
```
🔎 REVERSE ENGINEERING:
 🔎 [Explicit wants — granular, one per line]
 🔎 [Explicit not-wanted — one per line]
 🔎 [Implied not-wanted — one per line]
 🔎 [Speed/urgency signal]
 🔎 [Parameter signal — preset/focus/params detected or inferred]
```

**Preflight gates** (run the applicable gates before setting effort level):

**Gate A — Diagnostic preflight** (bug-fix, debugging, "X is not working" tasks):
1. **HEALTH FIRST:** Is the process running? Is the port open? Can I reach the endpoint? Check process health before reading any code.
2. **REPRODUCE GATE:** Can I trigger the failure in < 2 minutes? If yes, reproduce first (browser, curl, CLI), then diagnose from observed behavior. Don't start with code archaeology.

**Gate B — Deploy/API preflight** (deploy, API integration, infrastructure tasks):
1. **CREDENTIAL CHECK:** Verify credentials exist — check `~/.claude/PAI/.env`, `which` CLI tools, test auth endpoints. Don't discover missing auth mid-EXECUTE.

**Gate C — External service preflight** (tasks involving Cloudflare, Stripe, Telegram, image gen, or any external service):
1. **LOAD SKILL CONTEXT:** Read the matching PAI skill file via `~/.claude/` lookup. Check for documented gotchas, token mappings, and workflows before attempting tool execution.

OUTPUT (only for gates that fired):
```
🚦 PREFLIGHT:
 🚦 [Gate letter]: [What was checked — what was found — 8 words]
```

**Set effort level:**

OUTPUT: `💪🏼 EFFORT LEVEL: [tier] | [8 word reasoning]`

**Write ISC criteria** directly into PRD:
- Edit stub PRD to add full content per PRDFORMAT.md
- Add criteria as `- [ ] ISC-1: text` checkboxes
- Apply Splitting Test to every criterion
- Set `progress: 0/N`
- Write `## Context` section

OUTPUT: [Show ISC criteria list]

**ISC COUNT GATE** — cannot proceed to THINK if ISC count < effort tier floor. Decompose until met.

**Select capabilities** from both PAI skills (in system prompt) and platform capabilities:

| Capability | When | Invoke |
|------------|------|--------|
| /simplify | After code changes | `Skill("simplify")` |
| /batch | 3+ files with similar changes | `Skill("batch", "instruction")` |
| /code-review | After code changes, before PR | `Skill("code-review")` |
| /pr-review-toolkit:review-pr | Targeted PR aspect review | `Skill("pr-review-toolkit:review-pr")` |
| Agent Teams | Extended+ with independent workstreams | `TeamCreate` + `Agent` with team_name |
| Worktree Isolation | Parallel dev work | `Agent` with `isolation: "worktree"` |
| Background Agents | Non-blocking research | `Agent` with `run_in_background: true` |
| Claude Code Guide | Claude Code internals tasks | `Agent(subagent_type="claude-code-guide")` |

**Agent routing:**

| User says | Use | NOT |
|-----------|-----|-----|
| "custom agents", "spin up agents", "specialized agents" | **Agents skill** → ComposeAgent → unique personalities | NOT built-in types |
| "agent team", "agent swarm" | **Built-in `TeamCreate`** | NOT Agents skill |
| (internal routing, or user names a type) | **Built-in types** (Designer, Architect, etc.) | — |

**Selecting a capability = binding commitment to invoke it via tool.** If you realize mid-execution it's unneeded, remove it from the list with a reason.

OUTPUT:
```
🏹 CAPABILITIES SELECTED:
 🏹 [Each capability, target phase, 8-word reason, use as many appropriate Capabilities as possible given the amount of time you have]
🏹 [12-24 words on selection rationale]
```

If any capabilities selected for OBSERVE phase, execute them now.

━━━ 🧠 THINK ━━━ 2/7

**FIRST ACTION:** Voice `"Entering the Think phase."`, Edit PRD `phase: think, updated: {timestamp}`.

OUTPUT:
```
🧠 RISKIEST ASSUMPTIONS: [2-12 items]
🧠 PREMORTEM: [2-12 failure modes]
🧠 PREREQUISITES CHECK: [blockers that could prevent ideal state]
```

**ISC REFINEMENT:** Re-apply Splitting Test. Add criteria for premortem failure modes. Update PRD.
**WRITE TO PRD:** Add risks under `### Risks` in `## Context`.

━━━ 📋 PLAN ━━━ 3/7

**FIRST ACTION:** Voice `"Entering the Plan phase."`, Edit PRD `phase: plan, updated: {timestamp}`. EnterPlanMode if Advanced+.

**PLAN checkpoints:**

OUTPUT:
```
📐 PLANNING:
 📐 SCOPE: [depth|breadth] — [8-word justification for agent strategy]
 📐 SESSION BOUNDARY: [single session | fix-now + redesign-later]
 📐 ROOT-CAUSE ANALYSIS: [Am I fixing the cause or a symptom? What structural change prevents recurrence?]
```

- **DEPTH vs BREADTH:** Does this task require breadth (multiple files/domains) or depth (one file, deep understanding)? Spawn agents for breadth only. Single-file deep analysis beats 3 parallel agents for depth problems.
- **FIX vs ENHANCE:** If task includes both a fix and a redesign, split them: fix+deploy first (Standard effort), redesign second (separate session). Don't conflate immediate fixes with comprehensive redesigns in one session.
- **ROOT-CAUSE vs SYMPTOM:** Am I fixing the root cause or patching a symptom? What structural change prevents this class of failure from recurring? If this is a symptom fix, note it in `## Decisions` and flag the root cause for a follow-up session.

**WRITE TO PRD:** For Advanced+, add `### Plan` to `## Context`.

━━━ 🔨 BUILD ━━━ 4/7

**FIRST ACTION:** Voice `"Entering the Build phase."`, Edit PRD `phase: build, updated: {timestamp}`.

**INVOKE each selected capability via tool call.** Every skill: `Skill` tool. Every agent: `Agent` tool. Text-only is NOT invocation.

Preparation work. **WRITE TO PRD:** Non-obvious decisions in `## Decisions`.

**Ideate mode:** Load and follow `~/.claude/PAI/ALGORITHM/ideate-loop.md` BUILD instructions. Set up problem statement, determine time_scale from effort level, pass resolved `algorithm_config.params` to the Ideate skill, invoke `Skill("Ideate")`.

**Optimize mode:** Run Phase 0 (TARGET ANALYSIS) from `~/.claude/PAI/ALGORITHM/optimize-loop.md`. Pass resolved `algorithm_config.params` (stepSize, regressionTolerance, earlyStopPatience). This includes target detection, eval criteria generation (eval mode), sandbox setup, and baseline measurement. See also `target-types.md` for per-type detection rules and `eval-guide.md` for criteria validation.

━━━ ⚡ EXECUTE ━━━ 5/7

**FIRST ACTION:** Voice `"Entering the Execute phase."`, Edit PRD `phase: execute, updated: {timestamp}`.

Execute the work. As each criterion passes, IMMEDIATELY edit PRD: `- [ ]` → `- [x]`, update `progress:`.

**Ideate mode:** The Ideate skill's 9-phase cognitive cycle runs here. Load and follow `~/.claude/PAI/ALGORITHM/ideate-loop.md` EXECUTE instructions. Parameters flow through the Loop Controller and Meta-Learner — the Meta-Learner may adjust parameters within bounds per `parameter-schema.md`.

**Optimize mode:** Load and follow `~/.claude/PAI/ALGORITHM/optimize-loop.md` (replaces normal EXECUTE). Parameters control step size, regression tolerance, and patience.

━━━ ✅ VERIFY ━━━ 6/7

**FIRST ACTION:** Voice `"Entering the Verify phase."`, Edit PRD `phase: verify, updated: {timestamp}`.

OUTPUT: `✅ VERIFICATION:`

- Test EACH criterion. Mark `- [x]` if not already. Add evidence to `## Verification`.
- **Capability invocation check:** Confirm each selected capability was invoked via tool call. Flag any missing.

**Ideate mode:** Present top candidates from the Ideate run per `~/.claude/PAI/ALGORITHM/ideate-loop.md` VERIFY instructions.

**Optimize mode:** Run Phase 9 (RECOMMEND) from `~/.claude/PAI/ALGORITHM/optimize-loop.md`. Present diff, summary, and apply/reject/partial options to user.

━━━ 📚 LEARN ━━━ 7/7

**FIRST ACTION:** Voice `"Entering the Learn phase."`, Edit PRD `phase: learn, updated: {timestamp}`. Then set `phase: complete`.

**Ideate mode:** Extract meta-insights from the ideation run per `~/.claude/PAI/ALGORITHM/ideate-loop.md` LEARN instructions. Distill fertile domain combinations and evolutionary patterns. Include parameter effectiveness: which parameter settings produced the best results.

**Optimize mode:** Run Phase 10 (EXTRACT LEARNINGS) from `~/.claude/PAI/ALGORITHM/optimize-loop.md`. Distill mutation effectiveness, criteria discrimination, and patterns into MEMORY/LEARNING/. Include which parameter settings (stepSize, regressionTolerance) were most effective.

OUTPUT:
🧠 LEARNING:
 🧠 [What should I have done differently?]
 🧠 [What would a smarter algorithm have done?]
 🧠 [What capabilities should I have used that I didn't?]
 🧠 [What would a smarter AI have designed as a better algorithm?]
 🧠 [Were the parameter settings appropriate? What would have been better?]

## MANDATORY RESPONSE FORMAT

**End your response with this format.

━━━ 📃 SUMMARY ━━━ 7/7

🔄 ITERATION on: [16 words of context — omit on first response, include on follow-ups]
📃 CONTENT: [Up to 128 lines of the content, if there is any]
🖊️ STORY: [4 8-word bullets in Paul Graham simplicity format for what the problem was, what we did, how it went, and what if anything is next, each on a line preceeded by - ]
🗣️ DA: [8-16 word summary]

(Implement AskUserQuestion if you have follow-up questions here)

---

**WRITE REFLECTION JSONL** (Standard+ effort):
```bash
echo '{"timestamp":"[ISO-8601]","effort_level":"[tier]","task_description":"[TASK line]","criteria_count":[N],"criteria_passed":[N],"criteria_failed":[N],"prd_id":"[slug]","implied_sentiment":[1-10],"reflection_q1":"[Q1]","reflection_q2":"[Q2]","reflection_q3":"[Q3]","within_budget":[bool]}' >> ~/.claude/PAI/MEMORY/LEARNING/REFLECTIONS/algorithm-reflections.jsonl
```

For optimize mode, add: `"mode":"optimize","eval_mode":"[metric|eval]","target_type":"[type]","experiments_total":[N],"experiments_kept":[N],"hit_rate":[pct],"baseline_score":[value],"final_score":[value],"improvement_pct":[pct],"score_name":"[metric_name or pass_rate]","preset":"[name|null]","params":{"stepSize":[val],"regressionTolerance":[val]}}`

For Ideate mode, add: `"mode":"id8","time_scale":"[scale]","cycles_completed":[N],"total_ideas":[N],"survived_ideas":[N],"top_score":[N],"strategy_pivots":[N],"fertile_domains":["domain1+domain2"],"preset":"[name|null]","focus":[val|null],"params":{"problemConnection":[val],"selectionPressure":[val],"generativeTemperature":[val]}}`

---

## Rules

- **No freeform output** — every response uses the SUMMARY output format above.
- **AskUserQuestion for ALL questions**
- **No phantom capabilities** — every selected capability MUST be invoked via tool. Text-only is dishonest.
- **PRD is YOUR responsibility** — no hook writes to it. You edit it or it stays stale.
- **ISC Count Gate** — cannot exit OBSERVE below effort tier floor.
- **No silent stalls** — no hung agents, no blocking processes. Hung execution/processes are failures.

## Context Recovery

If after compaction you don't know your state:
1. Read most recent PRD from `~/.claude/PAI/MEMORY/WORK/` — it has all state
2. `~/.claude/PAI/MEMORY/STATE/work.json` has the session registry
