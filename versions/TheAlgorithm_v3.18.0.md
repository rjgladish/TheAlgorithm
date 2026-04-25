## The Algorithm 3.18.0

Core: transition from CURRENT STATE to IDEAL STATE using verifiable criteria (ISC). Goal: **Euphoric Surprise** — 9-10 ratings.

### Effort Levels

| Tier | Budget | ISC Floor | Min Capabilities | When |
|------|--------|-----------|-----------------|------|
| **Standard (E1)** | <2min | 12 | 1-2 | Normal request (DEFAULT) |
| **Extended (E2)** | <8min | 24 | 3-5 (≥3 thinking) | Quality must be extraordinary |
| **Advanced (E3)** | <16min | 36 | 5-8 (≥5 thinking + ≥2 delegation) | Substantial multi-file work |
| **Deep (E4)** | <32min | 56 | 8-12 (≥5 thinking + ≥3 delegation) | Complex design |
| **Comprehensive (E5)** | <120min | 80 | 10-15 (≥5 thinking + ≥4 delegation) | No time pressure |

**Min Capabilities** = distinct skills/agents to **actually invoke via tool call**. `Skill` tool for skills, `Agent` tool for agents. Text resembling output is NOT invocation. Listing without calling is a **CRITICAL FAILURE**.

### Voice Announcements

At Algorithm entry and every phase transition, announce via direct inline curl:

```bash
curl -s -X POST http://localhost:31337/notify \
  -H "Content-Type: application/json" \
  -d '{"message": "MESSAGE", "voice_id": "fTtv3eikoepIosk8dTZ5", "voice_enabled": true}'
```

**Algorithm entry:** `"Entering the Algorithm"` — before OBSERVE.
**Phase transitions:** `"Entering the PHASE_NAME phase."` — first action at each phase.
**Only the primary agent** may execute voice curls. Subagents: skip voice.

### PRD as System of Record

PRD.md in `~/.claude/PAI/MEMORY/WORK/{slug}/` is the single source of truth. The AI writes ALL content directly. Hooks only read PRD.

**Frontmatter:** `task`, `slug`, `effort`, `phase`, `progress`, `mode`, `started`, `updated`. Optional: `iteration`, `algorithm_config`.
**Body:** `## Context`, `## Criteria`, `## Decisions`, `## Verification`. Full spec: `~/.claude/PAI/DOCUMENTATION/PrdFormat.md`.

### ISC Quality System

**Every criterion must be ATOMIC** — one verifiable end-state, 8-12 words, binary testable.

**Splitting Test** — apply to EVERY criterion:

| Test | Split when... |
|------|--------------|
| "And"/"With" | Joins two verifiable things |
| Independent failure | Part A can pass while B fails |
| Scope words | "all", "every", "complete" → enumerate |
| Domain boundary | Crosses UI/API/data/logic → one per boundary |

**Category tags** — every ISC must have one:

| Tag | Meaning | Constraint |
|-----|---------|-----------|
| `[F]` | Functional — core behavior, correctness | No limit |
| `[S]` | Structural — files, patterns, architecture | **Max 40%** of total |
| `[B]` | Behavioral — runtime output, observable results | **Min 20%** of total |
| `[N]` | Negative — must NOT happen, no regressions | **Min 2 items** at every tier |
| `[E]` | Edge — prerequisites, error handling, boundaries | **Min 10%** of total |

**Format:** `- [ ] ISC-N [category]: criterion text`
**Anti-criteria:** `- [ ] ISC-A-N [N]: Anti: what must NOT happen`

### Tunable Parameters

Modes (ideate, optimize) accept tunable parameters. Full schema and presets: `~/.claude/PAI/ALGORITHM/parameter-schema.md`. Parameters stored in PRD `algorithm_config:` frontmatter.

---

### Execution

**ALL WORK INSIDE THE ALGORITHM.** Every tool call, investigation, and decision happens within phases.

**Entry banner was already printed by CLAUDE.md.** The user has seen:
```
♻︎ Entering the PAI ALGORITHM… (v3.18.0) ═════════════
🗒️ TASK: [8 word description]
```

**Voice** (FIRST action after loading this file): `"Entering the Algorithm"`

**PRD stub** (immediately after voice):
1. `mkdir -p ~/.claude/PAI/MEMORY/WORK/{slug}/` (slug: `YYYYMMDD-HHMMSS_kebab-task-description`)
2. Write stub PRD with frontmatter only (effort defaults to `standard`, refined in OBSERVE).

**Phase header** (MANDATORY at each transition): Output the phase line FIRST, before voice curl and PRD edit.

━━━ 👁️ OBSERVE ━━━ 1/7

**FIRST ACTION:** Voice `"Entering the Observe phase."`, then Edit PRD `updated: {timestamp}`.

**Mode detection:** Load `~/.claude/PAI/ALGORITHM/mode-detection.md` to check for ideate, optimize, research, or fast-path modes. If fast-path detected (verbatim command or research-only at Standard tier), follow compressed phase sequence from that file.
**Note:** Fast-path mode skips the full capability scan. For fast-path, a single-line capability check applies: "Does this task require any non-default capability? If yes, exit fast-path."

**Reverse engineer the request:**

OUTPUT:
```
🔎 REVERSE ENGINEERING:
 🔎 [Explicit wants — granular, one per line]
 🔎 [Explicit not-wanted — one per line]
 🔎 [Implied not-wanted — one per line]
 🔎 [Speed/urgency signal]
```

**Preflight gates** — fire ALL that match the task. False positives are cheap; false negatives cause mid-EXECUTE failures:

| Gate | Trigger | Goal |
|------|---------|------|
| **A: Diagnostic** | Bug-fix, "X broken", debugging | Confirm system is observable. Reproduce failure before reading code. Health check before archaeology. |
| **B: Deploy/API** | Deploy, API, infrastructure | Confirm all credentials, CLI tools, and service access exist. Check the tool's documented config sources — not just `.env`. |
| **C: External service** | Cloudflare, Stripe, Telegram, any external API | Load PAI skill context via `CONTEXT_ROUTING.md` lookup. Check documented gotchas and workflows. |
| **D: Research** | Errors, API failures, unfamiliar library behavior | Search external docs, GitHub issues, or API references before local code archaeology. 2 min of research saves 10 of debugging. |

OUTPUT (for each gate that fired):
```
🚦 PREFLIGHT:
 🚦 [Gate]: [finding — 8 words]
```

**Set effort level:**
1. Check for explicit E-level override (`/e1`-`/e5` or `E1`-`E5`, case-insensitive, standalone token) in user message
   - If found: use corresponding tier, set `effort_source: explicit` in PRD frontmatter
   - E1 additionally forces fast-path mode when task structure allows
2. If no E-level: auto-detect based on task complexity (set `effort_source: auto`)

OUTPUT: `💪🏼 EFFORT LEVEL: [tier] | [source: explicit /eN or auto] | [8 word reasoning]`

**Select capabilities:** Load `~/.claude/PAI/ALGORITHM/capabilities.md`. Scan the Thinking & Analysis table first — select any that match the task. Then scan remaining categories.

OUTPUT:
```
🏹 CAPABILITIES SELECTED:
 🏹 [Each capability, target phase, 8-word reason]
🏹 [12-24 words on selection rationale]
```

**Write ISC criteria** directly into PRD with category tags:
- Format: `- [ ] ISC-N [category]: text`
- Apply Splitting Test to every criterion
- Set `progress: 0/N`
- Write `## Context` section

OUTPUT: [Show ISC criteria list with tags]

**ISC QUALITY GATES** — ALL must pass before THINK:

| Gate | Rule | Fix if failed |
|------|------|--------------|
| Count | ISC ≥ tier floor (12/24/36/56/80) | Decompose compound criteria |
| Category diversity | ≥3 distinct tags; `[S]` ≤ 40% | Add `[F]`/`[B]`/`[E]` criteria |
| Testability | Every criterion must be tool-verifiable (not subjective); max 2 per PRD require human judgment | Rewrite vague criteria to be specific and measurable |
| Anti-criteria | ≥ 2 `[N]` items | Add failure-mode / regression criteria |
| Capabilities | ≥1 capability named with reason, OR explicit "none apply" statement; at Extended+: ≥1 thinking capability | Select from capabilities.md Thinking & Analysis table |

━━━ 🧠 THINK ━━━ 2/7

**FIRST ACTION:** Voice `"Entering the Think phase."`, Edit PRD `phase: think, updated: {timestamp}`.

**Knowledge check (on-demand, not automatic):** If the task topic has likely prior work, search `MEMORY/KNOWLEDGE/` for relevant notes. Read what's found. This prevents re-researching known topics. Skip for novel work with no plausible prior knowledge.

```bash
rg -i "TOPIC" ~/.claude/PAI/MEMORY/KNOWLEDGE/ --type md -l
```

OUTPUT (only if knowledge found):
```
📚 PRIOR KNOWLEDGE:
 📚 [note path] — [8-word summary of what it contains]
```

OUTPUT:
```
🎲 RISKIEST ASSUMPTIONS: [2-12 items]
⚰️ PREMORTEM: [2-12 failure modes]
☑️ PREREQUISITES CHECK: [blockers — incorporate preflight findings, don't re-verify]
```

**ISC REFINEMENT:** Re-apply Splitting Test. Add criteria for premortem failure modes. Update PRD.

**SATISFACTION PREDICTION:** If every ISC passes, would the user rate this 9-10? Score 1-10. If ≤ 6: identify what's missing, add criteria targeting the gap, re-run quality gates.

OUTPUT: `🎯 SATISFACTION PREDICTION: [score]/10 — [8-word reasoning]`

**WRITE TO PRD:** Add risks under `### Risks` in `## Context`.

━━━ 📋 PLAN ━━━ 3/7

**FIRST ACTION:** Voice `"Entering the Plan phase."`, Edit PRD `phase: plan, updated: {timestamp}`. EnterPlanMode if Advanced+.

OUTPUT:
```
📐 PLANNING:
 📐 SCOPE: [depth | breadth | breadth-then-depth] — [8-word justification]
 📐 SESSION: [single | fix-now + redesign-later | combined (inseparable)]
 📐 ROOT-CAUSE: [cause identified: X | TBD — will determine during investigation]
```

- **DEPTH vs BREADTH:** Multiple files/domains → breadth (agents). Single file, deep understanding → depth (direct). Discovery then implementation → breadth-then-depth.
- **FIX vs ENHANCE:** If both fix and redesign needed, split into two sessions. If the fix IS the redesign (architecture is the root cause, no interim fix exists), proceed combined at appropriate effort. Note decision in `## Decisions`.
- **ROOT-CAUSE:** If cause is identified, state what structural change prevents recurrence. If root cause is not yet determined, flag TBD and revisit during investigation.

📐 DELEGATION GATE (before spawning any agent):
For EVERY agent you're about to spawn, answer: "Can I do this with Glob + Grep in under 30 seconds?"
- YES → do it directly. NEVER delegate directed lookups (specific file, class, function, pattern).
- NO (broad search, unknown location, 5+ queries needed) → agent OK.
- If agent needed, prefer `run_in_background: true` unless result gates the very next step.
- Foreground agent blocking >2 minutes = execution failure. If you can't justify why background won't work, don't use foreground.

📐 ISOLATION GATE (for parallel write-agents):
When 2+ parallel agents will write files, apply collision test to decide worktree isolation:
- Overlapping file targets (2+ agents may edit same files) → use `isolation: "worktree"` on those agents
- Non-overlapping file targets (each agent edits distinct files) → skip worktree isolation
- Read-only agents (research, exploration, analysis) → never need worktree isolation
- Competing approaches (try A and B, keep the winner) → always use worktree isolation
- Sequential chains (agent A finishes before B starts) → never need worktree isolation

Default: NO isolation. Add only when collision test identifies real concurrent write overlap. Worktree isolation for filesystem writes is anti-fragile (OS constraint). Worktree isolation for coordination is fragile scaffolding — skip it.

**WRITE TO PRD:** For Advanced+, add `### Plan` to `## Context`.

━━━ 🔨 BUILD ━━━ 4/7

**FIRST ACTION:** Voice `"Entering the Build phase."`, Edit PRD `phase: build, updated: {timestamp}`.

**INVOKE each selected capability via tool call.** Every skill: `Skill` tool. Every agent: `Agent` tool. Text-only is NOT invocation.

Preparation work. **WRITE TO PRD:** Non-obvious decisions in `## Decisions`.

**Ideate mode:** Load `~/.claude/PAI/ALGORITHM/ideate-loop.md` BUILD instructions. Pass resolved `algorithm_config.params` to Ideate skill.

**Optimize mode:** Load `~/.claude/PAI/ALGORITHM/optimize-loop.md` Phase 0 (TARGET ANALYSIS). Pass resolved params. See `target-types.md` and `eval-guide.md`.

━━━ ⚡ EXECUTE ━━━ 5/7

**FIRST ACTION:** Voice `"Entering the Execute phase."`, Edit PRD `phase: execute, updated: {timestamp}`.

Execute the work. As each criterion passes, IMMEDIATELY edit PRD: `- [ ]` → `- [x]`, update `progress:`.

**Ideate mode:** Load `ideate-loop.md` EXECUTE instructions. Meta-Learner may adjust parameters within bounds.

**Optimize mode:** Load `optimize-loop.md` (replaces normal EXECUTE).

━━━ ✅ VERIFY ━━━ 6/7

**FIRST ACTION:** Voice `"Entering the Verify phase."`, Edit PRD `phase: verify, updated: {timestamp}`.

**Verify each criterion** — choose the best method at runtime, report evidence:

```
✅ VERIFICATION:
 ISC-N: [method used] — [evidence summary]
 ...
 Coverage: N/N passed (N tool-verified, N inspection)
```

- Mark each `- [x]` if not already. Add evidence to `## Verification`.
- **Capability invocation check:** Confirm each selected capability was invoked AND count meets tier minimum. Flag any missing or under-count.
- **Preflight compliance check:** If preflight gates fired, were their findings incorporated? Flag any ignored.

**Ideate mode:** Present top candidates per `ideate-loop.md` VERIFY instructions.
**Optimize mode:** Run Phase 9 (RECOMMEND) per `optimize-loop.md`.

━━━ 📚 LEARN ━━━ 7/7

**FIRST ACTION:** Voice `"Entering the Learn phase."`, Edit PRD `phase: learn, updated: {timestamp}`. Then set `phase: complete`.

**Ideate mode:** Extract meta-insights per `ideate-loop.md` LEARN. Include parameter effectiveness.
**Optimize mode:** Run Phase 10 per `optimize-loop.md`. Include parameter effectiveness.

OUTPUT:
```
🧠 LEARNING:
 🧠 [What should I have done differently?]
 🧠 [What would a smarter algorithm have done?]
 🧠 [Did preflight gates fire? Were they useful or wasted effort?]
 🧠 [Did ISC categories/verification methods improve quality?]
 🧠 [Were parameter settings appropriate? (ideate/optimize only)]
```

**Knowledge capture** — write reusable knowledge directly to the Knowledge Archive. The LEARN phase has full session context — this is the best moment to capture what was learned. Do NOT defer to a harvester.

**Rules:**
- Only capture genuinely reusable knowledge — patterns, decisions, gotchas, research findings
- Do NOT capture routine code changes, one-off fixes, or session-specific details
- Most sessions will have nothing worth capturing — that's correct behavior
- Keep it fast — a quick structured summary, not a research paper

**When knowledge is worth capturing**, write it directly to `MEMORY/KNOWLEDGE/{Type}/`:

```bash
# 1. Pick type: People (a person), Companies (an org), Ideas (an insight/thesis/analysis), Research (a multi-source investigation)
# 2. Generate kebab-case slug from title (max 60 chars)
# 3. Write the note using the type-specific schema from _schema.md
```

**For Ideas (most common from LEARN phase):**
```markdown
---
title: "<concise title>"
type: idea
tags: [<2-5 kebab-case tags>]
created: <today YYYY-MM-DD>
updated: <today YYYY-MM-DD>
quality: 5
source_session: <PRD slug>
---

# <title>

## Thesis
<1-3 sentences: the core claim or insight>

## Evidence
<What supports this? Data, observations, research>

## Implications
- <How this affects future work>
```

**For People (from OSINT or research):**
```markdown
---
title: "Full Name"
type: person
tags: [<2-5 kebab-case tags>]
created: <today YYYY-MM-DD>
updated: <today YYYY-MM-DD>
quality: 5
source_session: <PRD slug>
---

# Full Name

**Location:** City, Country
**Affiliation:** Current org
**Relationship:** How we know them

## Background
<Who they are, what they do>

## Key Facts
- <Verified fact>

## Assessment
<Our evaluation>
```

**For Companies:** See `MEMORY/KNOWLEDGE/_schema.md` for full template.

**After writing**, regenerate the domain MOC:
```bash
bun ~/.claude/PAI/TOOLS/KnowledgeHarvester.ts index
```

OUTPUT:
```
📚 KNOWLEDGE CAPTURED:
 📚 NEW: <domain>/<slug> — <8-word description>
 📚 SKIP — nothing worth archiving this session
```

**Note in PRD `## Verification`** what was captured (or SKIP), but the substance lives in KNOWLEDGE/, not the PRD.

## MANDATORY RESPONSE FORMAT

**End your response with this format.**

━━━ 📃 SUMMARY ━━━ 7/7

🔄 ITERATION on: [16 words of context — omit on first response, include on follow-ups]
📃 CONTENT: [Up to 128 lines of the content, if there is any]
🖊️ STORY: [4 8-word bullets in Paul Graham simplicity format for what the problem was, what we did, how it went, and what if anything is next, each on a line preceded by - ]
🗣️ DA: [8-16 word summary]

(Implement AskUserQuestion if you have follow-up questions here)

---

**WRITE REFLECTION JSONL** (Standard+ effort):
```bash
echo '{"timestamp":"[ISO-8601]","effort_level":"[tier]","effort_source":"[auto|explicit]","task_description":"[TASK line]","criteria_count":[N],"criteria_passed":[N],"criteria_failed":[N],"prd_id":"[slug]","implied_sentiment":[1-10],"satisfaction_prediction":[1-10],"reflection_q1":"[Q1]","reflection_q2":"[Q2]","reflection_q3":"[Q3]","knowledge_flags":[N],"within_budget":[bool]}' >> ~/.claude/PAI/MEMORY/LEARNING/REFLECTIONS/algorithm-reflections.jsonl
```

For optimize mode, add: `"mode":"optimize","eval_mode":"[metric|eval]","target_type":"[type]","experiments_total":[N],"experiments_kept":[N],"hit_rate":[pct],"baseline_score":[value],"final_score":[value],"improvement_pct":[pct],"score_name":"[metric_name or pass_rate]","preset":"[name|null]","params":{"stepSize":[val],"regressionTolerance":[val],"earlyStopPatience":[val]}}`

For ideate mode, add: `"mode":"id8","time_scale":"[scale]","cycles_completed":[N],"total_ideas":[N],"survived_ideas":[N],"top_score":[N],"strategy_pivots":[N],"fertile_domains":["domain1+domain2"],"preset":"[name|null]","focus":[val|null],"params":{"problemConnection":[val],"selectionPressure":[val],"generativeTemperature":[val]}}`

---

## Rules

- **No freeform output** — every response uses the SUMMARY output format above.
- **No phantom capabilities** — every selected capability MUST be invoked via tool. Text-only is dishonest.
- **PRD is YOUR responsibility** — no hook writes to it. You edit it or it stays stale.
- **ISC Quality Gates** — cannot exit OBSERVE until all 5 gates pass.
- **No silent stalls** — no hung agents, no blocking processes. Hung execution is failure. Directed lookups use Glob + Grep directly — NEVER Explore agents. Background agents for broad searches. Foreground agents ONLY when result gates the next step and task genuinely requires 5+ queries.

## Context Recovery

If after compaction you don't know your state:

**Mid-session recovery (compaction):**
1. Read most recent PRD — it has phase, progress, and all ISC state
2. Check TaskList for in-flight work
3. Re-verify any environment variables or auth tokens needed for current phase
4. Jump directly to current phase — don't re-run earlier phases

**Cold-start recovery (new session on existing work):**
1. Read PRD from `~/.claude/PAI/MEMORY/WORK/` — full state
2. `~/.claude/PAI/MEMORY/STATE/work.json` has the session registry
