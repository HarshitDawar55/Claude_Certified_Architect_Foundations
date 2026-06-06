# CCA-F Quick-Reference Cheat Sheets
### One condensed page per domain — for final-week review

> Print these. Each sheet is the *minimum* you must recall instantly: key facts, the exact flags/syntax, the decision rules, and the anti-patterns the exam disguises as answers.

---

## 0. The universal exam heuristics (read first)

**Pick the answer that matches the guarantee to the mechanism, at the most proportionate effort.**

- **Hard guarantee needed** (order, identity, policy threshold) → **programmatic** mechanism: hook, prerequisite gate, forced `tool_choice`. *Never a prompt.*
- **Quality/consistency improvement** → cheapest prompt/architecture fix **first**: better tool descriptions → explicit criteria → few-shot examples → structural change. *Not* a classifier, fine-tune, router, or bigger model.
- **Diagnose before redesign** → the scenario's logs/evidence usually name the root cause. Don't blame components that the evidence shows are working.
- **Distractor smells:** sentiment analysis, self-reported confidence, train a classifier, fine-tune, bigger context window, majority-vote, "strengthen the prompt" for a deterministic need, or anything in the out-of-scope list.

**Numbers to memorize:** pass = **720/1000**; **4 of 6** scenarios appear; Domain weights **27 / 18 / 20 / 20 / 15**; Batch API = **50% off, ≤24h, no SLA**; tools per agent ≈ **4–5** (18 = too many); refund-policy example threshold = **$500**.

---

## 1. Domain 1 — Agentic Architecture & Orchestration (27%)

**Agentic loop (1.1)**
- Continue while `stop_reason == "tool_use"`; stop on `"end_turn"`.
- Append tool results to history every iteration (the model reasons over them).
- Model-driven decisions > hard-coded decision trees.
- ❌ Anti-patterns: parse natural language to stop; iteration cap as *primary* terminator; treat assistant text as "done."

**Coordinator–subagent (1.2)**
- Hub-and-spoke: all communication routes through the coordinator (observability, error handling).
- Subagents have **isolated context** (no auto-inheritance).
- Coordinator does: decompose, delegate, aggregate, choose which subagents.
- #1 bug: **narrow decomposition** → coverage gaps. Fix = broaden decomposition + **iterative refinement loop**.

**Spawning & context passing (1.3)**
- Spawn via the **`Task`** tool; coordinator `allowedTools` must include **`"Task"`**.
- Pass **all** needed context explicitly in the subagent prompt.
- Parallel = **multiple `Task` calls in ONE response**.
- Separate content from metadata (source/date/page) to preserve attribution.
- Prompt subagents with **goals + quality criteria**, not procedural steps.

**Enforcement & handoff (1.4)**
- Deterministic need (identity before refund) → **programmatic prerequisite gate** (block `process_refund` until `get_customer` verified). Prompts have non-zero failure rate.
- Multi-concern request → decompose → parallel-investigate → synthesize one resolution.
- Human handoff package = customer ID + root cause + amount + recommended action (human has **no transcript**).

**Hooks (1.5)**
- **`PostToolUse`** → transform/normalize tool **results** (Unix/ISO-8601/status codes → canonical).
- **Tool-call interception** → gate **outgoing** calls (block refund > $500 → redirect to human).
- Hooks = deterministic; prompts = probabilistic.

**Task decomposition (1.6)**
- Predictable multi-aspect → **prompt chaining** (fixed pipeline).
- Open-ended investigation → **adaptive/dynamic** decomposition.
- Big review → **per-file passes + cross-file integration pass** (cures attention dilution; bigger context does NOT).

**Sessions (1.7)**
- **`--resume <session-name>`** = continue a named session.
- **`fork_session`** = branch from a shared baseline to explore divergent approaches.
- Resume when context mostly valid; start fresh + inject summary when results are stale.
- After edits, tell the resumed session **which files changed**.

---

## 2. Domain 2 — Tool Design & MCP Integration (18%)

**Tool descriptions (2.1)**
- Descriptions are the **primary** tool-selection mechanism. Thin descriptions = misrouting.
- Include inputs, example queries, edge cases, "use this vs that" boundaries.
- Fixes: rewrite/rename to remove overlap (`analyze_content` → `extract_web_results`); split generic tools (`analyze_document` → `extract_data_points` / `summarize_content` / `verify_claim_against_source`).
- Check the **system prompt** for biasing keywords.

**Structured errors (2.2)**
- Use the **`isError`** flag + `errorCategory` (**transient / validation / business / permission**) + **`isRetryable`** + human-readable message.
- Transient = retry; business = `retriable: false` + customer-friendly text.
- **Empty result ≠ error.** Access failure ≠ empty result.
- Recover locally first; propagate only unresolved errors with partial results + what was attempted.

**Tool distribution & choice (2.3)**
- ~**4–5 tools per agent**, scoped to role (18 = too many → degraded selection).
- Replace generic tools with constrained ones (`fetch_url` → `load_document`).
- Scoped cross-role tool for the high-frequency case (`verify_fact`); route complex via coordinator.
- **`tool_choice`**: `"auto"` (text or tool) · `"any"` (must call *some* tool) · forced `{"type":"tool","name":"..."}` (specific tool first).

**MCP integration (2.4)**
- **`.mcp.json`** = project/shared (commit it); **`~/.claude.json`** = user/personal.
- **`${ENV_VAR}`** expansion keeps secrets out of version control.
- All connected servers' tools available **simultaneously**.
- **MCP resources** = content catalogs (issue summaries, schemas) → fewer exploratory calls.
- Community server for standard integrations (Jira); custom only for team-specific.
- Rich MCP tool descriptions so the agent doesn't default to built-ins (`Grep`).

**Built-in tools (2.5)**
- **`Grep`** = search file **content**; **`Glob`** = match file **paths** (`**/*.test.tsx`).
- **`Read`/`Write`** = full file; **`Edit`** = targeted via **unique** anchor.
- Edit fails on non-unique text → fallback **Read + Write**.
- Explore incrementally: Grep entry points → Read to follow imports. Never read everything up front.

---

## 3. Domain 3 — Claude Code Configuration & Workflows (20%)

**CLAUDE.md hierarchy (3.1)**
- **User** `~/.claude/CLAUDE.md` (private) → **Project** `.claude/CLAUDE.md` or root (shared) → **Directory** subdir CLAUDE.md.
- "Teammate not getting instructions" = they're at **user** level → move to **project**.
- **`@import`** = modular includes; **`.claude/rules/`** = topic files; **`/memory`** = inspect loaded files.

**Commands & skills (3.2)**
- Commands: **`.claude/commands/`** (shared) vs **`~/.claude/commands/`** (personal).
- Skill `SKILL.md` frontmatter: **`context: fork`** (isolate output), **`allowed-tools`** (restrict), **`argument-hint`** (prompt for args).
- Personal skill variant → `~/.claude/skills/` with a **different name**.
- Skills = on-demand; CLAUDE.md = always-loaded.
- ❌ `.claude/config.json` with a commands array = **does not exist**.

**Path rules (3.3)**
- **`.claude/rules/`** + YAML **`paths:`** globs → conditional, automatic, by-file-path loading; reduces tokens.
- Globs follow file **type** across directories; directory CLAUDE.md is location-bound.
- Scattered files (tests, terraform) → **path rules**, not subdir CLAUDE.md.

**Plan vs direct (3.4)**
- **Plan mode** → large-scale, multiple approaches, architectural, multi-file (e.g., 45+ files); prevents costly rework.
- **Direct** → small, clear-scope, single-file.
- **`Explore` subagent** → isolate verbose discovery, return summaries, protect context.
- Combine: plan the design → directly execute it.

**Iterative refinement (3.5)**
- Inconsistent transform → **concrete input/output examples** (2–3).
- Quality bar → **test-first**, iterate on failures.
- Unfamiliar domain → **interview pattern** (Claude asks you questions first).
- **Interacting** issues → one message; **independent** issues → sequential.

**CI/CD (3.6)**
- **`-p`/`--print`** = non-interactive (cures CI hangs).
- **`--output-format json`** + **`--json-schema`** = structured output for inline PR comments.
- CLAUDE.md carries CI context (standards, fixtures, criteria).
- Independent review instance > self-review; feed prior findings → report only new/unaddressed; feed existing tests → avoid duplicates.
- ❌ `CLAUDE_HEADLESS`, `--batch` = don't exist.

---

## 4. Domain 4 — Prompt Engineering & Structured Output (20%)

**Explicit criteria (4.1)**
- Specific **categorical** criteria > vague hedges ("contradicts actual behavior," not "be accurate"/"be conservative").
- One high-false-positive category undermines trust in all → **temporarily disable** it, fix offline.
- Severity needs concrete examples per level.

**Few-shot (4.2)**
- The **most effective** fix for format consistency + ambiguous-case judgment.
- **2–4** targeted examples; **show the reasoning**, not just the answer → generalizes to novel cases.
- Reduces extraction hallucination / empty required fields.

**Structured output (4.3)**
- **`tool_use` + JSON schema** = guaranteed structure, **zero syntax errors** (but NOT semantic errors).
- **`tool_choice`**: `"auto"` (may return text) · `"any"` (must call a tool, unknown doc type) · forced (specific tool first).
- **Nullable/optional** fields prevent fabrication; `enum` + `"other"`/`"unclear"` for extensibility.
- Add format-normalization rules in the prompt.

**Validation & retry (4.4)**
- Retry **with error feedback** (document + failed extraction + specific errors).
- Retry helps **format/structural** errors; **useless when info is absent** from source.
- Validate semantics yourself: `calculated_total` vs `stated_total`; `conflict_detected` boolean.
- `detected_pattern` field → analyze false-positive/dismissal patterns.

**Batch processing (4.5)**
- **Message Batches API: ~50% cheaper, ≤24h window, NO latency SLA, NO multi-turn tool calling, `custom_id` correlation.**
- Batch = non-blocking/overnight; **synchronous = blocking/pre-merge.**
- Resubmit only failed `custom_id`s, with fixes (chunk oversized docs).
- Sample-tune the prompt before large batches.

**Multi-pass review (4.6)**
- **Independent** review instance > self-review (generator retains reasoning bias).
- Big reviews → per-file + cross-file integration passes.
- ❌ Bigger context window ≠ better attention; consensus voting hides intermittent bugs.

---

## 5. Domain 5 — Context Management & Reliability (15%)

**Conversation context (5.1)**
- Summarization eats exact figures → extract amounts/dates/IDs into a persistent **"case facts"** block, outside summaries.
- **Lost in the middle**: key findings at start/end; explicit section headers.
- **Trim** verbose tool outputs to relevant fields *before* they accumulate.
- Send full history each request; upstream agents emit **structured facts**, not verbose prose.

**Escalation & ambiguity (5.2)**
- Escalate on: **explicit human request, policy gap/exception, no meaningful progress.**
- Explicit demand → escalate now; straightforward → offer to resolve (escalate only if reiterated).
- ❌ Sentiment and self-reported confidence = unreliable complexity proxies.
- Multiple matches → **ask for more identifiers**, never guess.

**Error propagation (5.3)**
- Propagate **structured error context**: failure type + attempted query + partial results + alternatives.
- Access failure ≠ empty result.
- ❌ Generic status; silent suppression (empty-as-success); whole-workflow termination on one failure.
- Recover locally first; annotate synthesis with **coverage gaps**.

**Large codebase context (5.4)**
- Degradation tell: "typical patterns" instead of specific classes found earlier.
- **Scratchpad files** persist findings; **subagent delegation** isolates verbose output; **`/compact`** reduces usage.
- Crash recovery = structured **state manifests** the coordinator loads on resume.

**Human review & confidence (5.5)**
- Aggregate accuracy (97%) can hide bad **document types/fields** → segment before automating.
- **Stratified sampling** of high-confidence output finds error rates + novel patterns.
- **Field-level confidence**, calibrated on labeled sets, routes humans to riskiest cases.

**Provenance & uncertainty (5.6)**
- Preserve **claim→source mappings** through every summarization/synthesis step.
- Conflicting credible sources → **annotate both** with attribution; don't pick one.
- Carry **publication/collection dates** (temporal ≠ contradiction).
- Separate well-established vs contested; render content types naturally (financial=tables, news=prose, technical=lists).

---

## 6. "If you see X, answer Y" rapid-fire table

| If the scenario says… | The answer involves… |
|---|---|
| "must happen before," "mandatory," "verify identity first" | Programmatic prerequisite **gate / hook** (not a prompt) |
| "block refunds over $X," "must hold 100%" | **Tool-call interception hook** |
| Tools return mismatched date/number formats | **`PostToolUse`** normalization hook |
| Agent picks the wrong tool | **Improve tool descriptions** first |
| Two near-identical tools | **Rename / split** to remove overlap |
| Tool failed; agent retries everything | **Structured errors** (`errorCategory`, `isRetryable`) |
| Search found nothing | **Valid empty result**, not an error |
| Agent has 18 tools, misusing them | **Scope to ~4–5** role tools |
| Guarantee a specific tool runs first | Forced **`tool_choice {"type":"tool","name":...}`** |
| Must call *some* tool, unknown doc type | **`tool_choice: "any"`** |
| Shared team MCP server, secret token | **`.mcp.json`** + **`${ENV}`** expansion |
| Personal/experimental MCP server | **`~/.claude.json`** |
| Find callers of a function | **`Grep`** (content) |
| Find files by name pattern | **`Glob`** (paths) |
| `Edit` can't find unique text | **Read + Write** fallback |
| New teammate missing instructions | Move config from **user → project** level |
| Command for everyone on clone | **`.claude/commands/`** |
| Skill output floods the chat | **`context: fork`** |
| Conventions for files spread everywhere | **`.claude/rules/`** glob `paths:` |
| Monolith → microservices / 45-file migration | **Plan mode** |
| Single-file bug with clear stack trace | **Direct execution** |
| CI job hangs on input | **`-p` / `--print`** |
| CI needs machine-parseable findings | **`--output-format json` + `--json-schema`** |
| Reviewer re-posts addressed comments | Feed **prior findings**, report only new |
| Too many trivial false positives | **Explicit categorical criteria**; disable noisy category |
| Inconsistent format / ambiguous cases | **2–4 few-shot examples (with reasoning)** |
| Malformed JSON output | **`tool_use` + JSON schema** |
| Model fabricates missing fields | **Nullable / optional** fields |
| Line items don't sum | **Semantic validation** (`calculated` vs `stated`) |
| Retry when info is absent from source | **Won't work** — don't retry |
| Cheap overnight bulk processing | **Message Batches API** (non-blocking only) |
| Blocking pre-merge check | **Synchronous API** |
| Some batch docs failed | Resubmit failed **`custom_id`s** (chunk oversized) |
| Model misses bugs in its own code | **Independent review instance** |
| Long chat loses exact $ / dates | Persistent **"case facts"** block |
| Important middle findings ignored | Key findings **at start/end** + headers |
| Customer demands a human | **Escalate immediately** |
| Two matching customers | **Ask for another identifier** |
| Subagent timed out | **Structured error context** to coordinator |
| Long exploration degrades | **Scratchpad + subagent + `/compact` + manifest`** |
| 97% accuracy, drop human review? | **Segment by type/field + stratified sampling** |
| Two credible sources conflict | **Annotate both** with attribution + dates |
