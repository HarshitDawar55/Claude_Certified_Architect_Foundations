# CCA-F Glossary & Flashcards
### Every key term and technology — flashcard format + alphabetical glossary

> **Part A (Flashcards):** active-recall cards grouped by topic. Cover the answer, recall it aloud, then expand `▶`. If your viewer doesn't expand `<details>`, the answer follows the prompt directly.
>
> **Part B (Glossary):** the same terms defined alphabetically for quick lookup.

---

# Part A — Flashcards

## A1. Claude Agent SDK

**Q: What are the two `stop_reason` values that drive an agentic loop, and what does each mean?**
<details><summary>▶</summary>`"tool_use"` = Claude wants to call a tool → **continue** the loop. `"end_turn"` = Claude is finished → **terminate** the loop. These are the *only* valid primary terminators.</details>

**Q: What happens to a tool's result after you execute it?**
<details><summary>▶</summary>You **append it to the conversation history** so the model can reason over the new information on the next iteration.</details>

**Q: Name three agentic-loop anti-patterns.**
<details><summary>▶</summary>(1) Parsing natural-language signals to decide termination; (2) using an arbitrary iteration cap as the *primary* stopping mechanism; (3) treating assistant text content as a completion indicator.</details>

**Q: What is the `Task` tool, and what must `allowedTools` contain to use it?**
<details><summary>▶</summary>The `Task` tool spawns subagents. The coordinator's **`allowedTools` must include `"Task"`** or it cannot delegate.</details>

**Q: How do you spawn subagents in parallel?**
<details><summary>▶</summary>Emit **multiple `Task` tool calls in a single coordinator response** (not one per turn).</details>

**Q: What is an `AgentDefinition`?**
<details><summary>▶</summary>The configuration for a subagent type: its **description**, **system prompt**, and **tool restrictions** (which tools that subagent may use).</details>

**Q: What does a `PostToolUse` hook do?**
<details><summary>▶</summary>Intercepts tool **results** before the model processes them — used to **normalize/transform** data (e.g., Unix timestamps, ISO 8601, status codes → one canonical format).</details>

**Q: What does a tool-call interception hook do?**
<details><summary>▶</summary>Intercepts **outgoing** tool calls before they execute — used to **enforce compliance** (e.g., block `process_refund` > $500 and redirect to human escalation).</details>

**Q: Hooks vs prompt instructions — what's the core difference?**
<details><summary>▶</summary>Hooks give **deterministic guarantees**; prompt instructions give only **probabilistic compliance** (non-zero failure rate). Use hooks when a rule must hold 100% of the time.</details>

---

## A2. Multi-agent orchestration

**Q: What is hub-and-spoke architecture?**
<details><summary>▶</summary>The coordinator (hub) manages **all** inter-subagent communication, error handling, and information routing; subagents (spokes) don't talk to each other directly. Gives observability and consistent error handling.</details>

**Q: Do subagents inherit the coordinator's conversation history?**
<details><summary>▶</summary>**No.** Subagents operate with **isolated context**. You must pass everything they need explicitly in their prompt.</details>

**Q: What are the coordinator's four jobs?**
<details><summary>▶</summary>Task **decomposition**, **delegation**, result **aggregation**, and **deciding which subagents** to invoke based on query complexity.</details>

**Q: A multi-agent report misses whole subtopics though every subagent succeeded. Root cause?**
<details><summary>▶</summary>**Overly narrow task decomposition by the coordinator.** Fix = broaden decomposition + add an **iterative refinement loop** that detects gaps and re-delegates.</details>

**Q: How should a coordinator prompt a subagent — procedures or goals?**
<details><summary>▶</summary>**Goals + quality criteria**, not step-by-step procedures, to preserve subagent adaptability.</details>

---

## A3. Enforcement, handoff, decomposition

**Q: A required tool order keeps being skipped ~10% of the time despite prompt instructions. Fix?**
<details><summary>▶</summary>A **programmatic prerequisite gate** that blocks the downstream tool until the prerequisite completes (e.g., block `process_refund` until `get_customer` returns a verified ID). Deterministic, not probabilistic.</details>

**Q: What must a human-escalation handoff package contain, and why?**
<details><summary>▶</summary>Customer ID, root-cause analysis, refund amount, recommended action — because the human agent has **no access to the conversation transcript**.</details>

**Q: Prompt chaining vs dynamic decomposition — when each?**
<details><summary>▶</summary>**Prompt chaining (fixed pipeline)** for predictable multi-aspect work; **dynamic/adaptive decomposition** for open-ended investigation where subtasks emerge from findings.</details>

**Q: How do you fix inconsistent, contradictory feedback on a large multi-file review?**
<details><summary>▶</summary>Split into **per-file local analysis passes + a separate cross-file integration pass** (cures attention dilution). A bigger context window does NOT fix it.</details>

---

## A4. Sessions

**Q: What does `--resume <session-name>` do?**
<details><summary>▶</summary>Continues a **specific named** prior conversation.</details>

**Q: What does `fork_session` do?**
<details><summary>▶</summary>Creates **independent branches from a shared analysis baseline** to explore divergent approaches in parallel without cross-contamination.</details>

**Q: Resume vs start-fresh — how to choose?**
<details><summary>▶</summary>**Resume** when prior context is mostly valid; **start fresh + inject a structured summary** when prior tool results are stale. After edits, tell the resumed session **which files changed**.</details>

---

## A5. Tool design & MCP

**Q: What is the primary mechanism LLMs use to select a tool?**
<details><summary>▶</summary>The **tool description.** Minimal descriptions → unreliable selection among similar tools. Include inputs, example queries, edge cases, and when-to-use-vs-not boundaries.</details>

**Q: Two near-identical tools cause misrouting. Two structural fixes?**
<details><summary>▶</summary>(1) **Rename + rewrite** to eliminate overlap (`analyze_content` → `extract_web_results`); (2) **split** a generic tool into purpose-specific ones (`analyze_document` → `extract_data_points` / `summarize_content` / `verify_claim_against_source`).</details>

**Q: What does the MCP `isError` flag do, and what metadata should accompany it?**
<details><summary>▶</summary>Signals a tool failure to the agent. Pair it with **`errorCategory`** (transient/validation/business/permission), **`isRetryable`** boolean, and a human-readable message.</details>

**Q: The four error categories?**
<details><summary>▶</summary>**Transient** (timeouts → retry), **validation** (bad input), **business** (policy violation → `retriable:false` + customer message), **permission** (not authorized).</details>

**Q: Is an empty search result an error?**
<details><summary>▶</summary>**No.** A successful query with zero matches is a **valid empty result** — distinct from an access failure (timeout). Don't conflate them.</details>

**Q: Roughly how many tools should an agent have, and why?**
<details><summary>▶</summary>About **4–5**, scoped to its role. Too many (e.g., 18) increases decision complexity and degrades selection reliability.</details>

**Q: The three `tool_choice` modes?**
<details><summary>▶</summary>**`"auto"`** (model may return text or call a tool), **`"any"`** (must call *some* tool), **forced** `{"type":"tool","name":"..."}` (must call a *specific* tool).</details>

**Q: Project vs user scope for MCP servers — which files?**
<details><summary>▶</summary>**`.mcp.json`** = project/shared (commit to repo). **`~/.claude.json`** = user/personal/experimental.</details>

**Q: How do you keep secrets out of `.mcp.json`?**
<details><summary>▶</summary>**Environment-variable expansion**, e.g. `"token": "${GITHUB_TOKEN}"`.</details>

**Q: What are MCP resources for?**
<details><summary>▶</summary>Exposing **content catalogs** (issue summaries, doc hierarchies, DB schemas) so the agent can see available data **without exploratory tool calls**.</details>

**Q: Community vs custom MCP servers?**
<details><summary>▶</summary>Prefer **existing community servers** for standard integrations (e.g., Jira); reserve **custom** servers for team-specific workflows.</details>

**Q: `Grep` vs `Glob`?**
<details><summary>▶</summary>**`Grep`** searches file **content** (patterns, function names, error strings). **`Glob`** matches file **paths** (e.g., `**/*.test.tsx`).</details>

**Q: When `Edit` can't find a unique anchor, what's the fallback?**
<details><summary>▶</summary>**`Read` the full file, then `Write` it back** with the change.</details>

---

## A6. Claude Code configuration

**Q: The CLAUDE.md hierarchy (3 levels)?**
<details><summary>▶</summary>**User** `~/.claude/CLAUDE.md` (private, not shared) → **Project** `.claude/CLAUDE.md` or root `CLAUDE.md` (shared via VCS) → **Directory** subdirectory `CLAUDE.md` (subtree).</details>

**Q: A new teammate isn't getting the team's instructions. Why, and fix?**
<details><summary>▶</summary>The instructions are at **user level** (`~/.claude/CLAUDE.md`), which isn't shared via version control. Move them to **project level**.</details>

**Q: What does `@import` do in CLAUDE.md?**
<details><summary>▶</summary>References external files to keep CLAUDE.md **modular** (e.g., import package-specific standards).</details>

**Q: What is `.claude/rules/` used for?**
<details><summary>▶</summary>Topic-specific rule files (an alternative to a monolithic CLAUDE.md), optionally with YAML **`paths:`** glob frontmatter for **conditional, path-scoped** loading.</details>

**Q: What does `/memory` do?**
<details><summary>▶</summary>Shows **which memory/config files are loaded**, to diagnose inconsistent behavior across sessions.</details>

**Q: Project vs user slash commands — which directories?**
<details><summary>▶</summary>**`.claude/commands/`** = shared/project (version-controlled). **`~/.claude/commands/`** = personal.</details>

**Q: Three skill `SKILL.md` frontmatter options and what each does?**
<details><summary>▶</summary>**`context: fork`** = run in an isolated sub-agent context (output doesn't pollute the main chat). **`allowed-tools`** = restrict which tools the skill may use. **`argument-hint`** = prompt the developer for required parameters.</details>

**Q: Skills vs CLAUDE.md — when each?**
<details><summary>▶</summary>**Skills** = on-demand invocation for task-specific workflows. **CLAUDE.md** = always-loaded universal standards.</details>

**Q: How do path-specific rules beat directory CLAUDE.md for scattered files?**
<details><summary>▶</summary>A **glob** like `**/*.test.tsx` follows the file **type** anywhere in the tree; a directory CLAUDE.md is **location-bound** and can't cover files spread across many directories.</details>

**Q: Plan mode vs direct execution — when each?**
<details><summary>▶</summary>**Plan mode** for large-scale, multi-approach, architectural, multi-file changes (prevents costly rework). **Direct execution** for small, well-scoped, single-file changes.</details>

**Q: What is the `Explore` subagent for?**
<details><summary>▶</summary>Isolating **verbose discovery output** and returning **summaries** to preserve main-conversation context during multi-phase tasks.</details>

**Q: Four iterative-refinement techniques?**
<details><summary>▶</summary>(1) Concrete **input/output examples**; (2) **test-driven iteration** (share failures); (3) the **interview pattern** (Claude asks you questions first); (4) **single message for interacting issues** vs sequential for independent ones.</details>

---

## A7. CI/CD & CLI

**Q: What flag runs Claude Code non-interactively in CI?**
<details><summary>▶</summary>**`-p`** (or **`--print`**) — processes the prompt, prints to stdout, exits. Cures CI hangs.</details>

**Q: How do you get structured, machine-parseable output in CI?**
<details><summary>▶</summary>**`--output-format json`** with **`--json-schema`** — for posting findings as inline PR comments.</details>

**Q: Two non-existent CI features the exam plants as distractors?**
<details><summary>▶</summary>**`CLAUDE_HEADLESS=true`** (env var) and **`--batch`** flag. Neither exists. (Don't confuse `--batch` with the Message Batches *API*.)</details>

**Q: Why use an independent instance to review generated code?**
<details><summary>▶</summary>The generating session **retains its reasoning context** and is less likely to question its own decisions; an independent instance catches more subtle issues than self-review or extended thinking.</details>

**Q: How do you avoid duplicate PR comments across re-runs?**
<details><summary>▶</summary>Include **prior review findings** in context and instruct Claude to report only **new or still-unaddressed** issues.</details>

---

## A8. Prompt engineering & structured output

**Q: Why doesn't "be conservative / only report high-confidence findings" improve precision?**
<details><summary>▶</summary>The model's self-confidence is poorly calibrated. Use **specific categorical criteria** (report bugs/security; skip minor style) instead.</details>

**Q: One noisy false-positive category — what's the effect and the fix?**
<details><summary>▶</summary>It **undermines trust in all the accurate categories** too. **Temporarily disable** it while you improve its prompt offline.</details>

**Q: Why are few-shot examples the top fix for consistency, and how many?**
<details><summary>▶</summary>They demonstrate format and ambiguous-case handling and let the model **generalize** to novel cases. Use **2–4** targeted examples that **show the reasoning**, not just the answer.</details>

**Q: Most reliable way to guarantee schema-compliant output?**
<details><summary>▶</summary>**`tool_use` with a JSON schema** — eliminates JSON **syntax** errors (but not semantic errors).</details>

**Q: Does `tool_use` + strict schema guarantee correct data?**
<details><summary>▶</summary>No — it guarantees valid **shape/syntax**, not **semantics** (line items may not sum; values may be in the wrong field). Validate semantics yourself.</details>

**Q: How do you stop the model fabricating values for missing fields?**
<details><summary>▶</summary>Make those fields **optional/nullable** so the model can return null instead of inventing data to satisfy a required field.</details>

**Q: Schema patterns for extensible/ambiguous categories?**
<details><summary>▶</summary>An **`enum`** with **`"other"` + a detail string**, and values like **`"unclear"`** for ambiguous cases.</details>

**Q: How does retry-with-error-feedback work, and when is retry useless?**
<details><summary>▶</summary>Append the **document + failed extraction + specific validation errors** to guide correction. Retry is **useless when the required info is absent** from the source (works only for format/structural errors).</details>

**Q: How do you self-validate semantic correctness in extraction?**
<details><summary>▶</summary>Extract **`calculated_total` alongside `stated_total`** to flag discrepancies; add a **`conflict_detected`** boolean for inconsistent source data.</details>

**Q: What is `detected_pattern` for?**
<details><summary>▶</summary>A field that tracks which code constructs trigger findings, enabling **analysis of false-positive/dismissal patterns**.</details>

---

## A9. Batch processing

**Q: Message Batches API — four defining facts?**
<details><summary>▶</summary>**~50% cost savings**, **up to 24-hour** processing window, **no guaranteed latency SLA**, and **no multi-turn tool calling** within a single request. Uses **`custom_id`** to correlate requests/responses.</details>

**Q: Batch vs synchronous — which workflows?**
<details><summary>▶</summary>**Batch** for non-blocking/latency-tolerant (overnight reports, weekly audits, nightly test gen). **Synchronous** for blocking workflows (pre-merge checks).</details>

**Q: 6 of 100 batch documents failed (too large). What do you do?**
<details><summary>▶</summary>Identify them by **`custom_id`** and **resubmit only those**, with fixes like **chunking** the oversized documents.</details>

---

## A10. Context management & reliability

**Q: What's the danger of progressive summarization?**
<details><summary>▶</summary>It condenses **exact numbers, percentages, dates, and customer-stated expectations** into vague summaries — losing facts you can't afford to lose.</details>

**Q: How do you preserve exact transactional facts in a long chat?**
<details><summary>▶</summary>Extract them into a persistent **"case facts" block** (amounts, dates, order numbers, statuses) included in **every** prompt, **outside** the summarized history.</details>

**Q: What is the "lost in the middle" effect, and how do you mitigate it?**
<details><summary>▶</summary>Models reliably use the **beginning and end** of long inputs but may omit middle content. Mitigate by placing **key findings at the start/end** and using **explicit section headers**.</details>

**Q: How do you stop verbose tool outputs from bloating context?**
<details><summary>▶</summary>**Trim** tool outputs to only the relevant fields **before** they accumulate (e.g., keep only return-relevant fields from a 40-field order lookup).</details>

**Q: The three legitimate escalation triggers?**
<details><summary>▶</summary>(1) Customer **explicitly requests a human**, (2) **policy exception/gap**, (3) **inability to make meaningful progress**.</details>

**Q: Why are sentiment and self-reported confidence bad escalation triggers?**
<details><summary>▶</summary>**Sentiment doesn't correlate with case complexity**, and **LLM self-confidence is poorly calibrated** (the agent is already wrongly confident on hard cases).</details>

**Q: `get_customer` returns multiple matches. What do you do?**
<details><summary>▶</summary>**Ask the customer for an additional identifier** to disambiguate — never select via heuristic.</details>

**Q: What should a subagent return to the coordinator when it fails?**
<details><summary>▶</summary>**Structured error context**: failure type, attempted query, partial results, and potential alternatives — so the coordinator can recover intelligently.</details>

**Q: Two error-propagation anti-patterns?**
<details><summary>▶</summary>(1) **Silently suppressing** errors (returning empty results as success); (2) **terminating the whole workflow** on a single failure. (Also: generic "search unavailable" status.)</details>

**Q: The tell-tale sign of context degradation in long exploration?**
<details><summary>▶</summary>The model gives inconsistent answers and references **"typical patterns"** instead of the **specific classes** it discovered earlier.</details>

**Q: Tools to fight context degradation?**
<details><summary>▶</summary>**Scratchpad files** (persist findings), **subagent delegation** (isolate verbose output), **`/compact`** (reduce context usage), and **structured state manifests** for crash recovery.</details>

**Q: Why is 97% aggregate accuracy risky for automating review?**
<details><summary>▶</summary>It can **mask poor performance on specific document types or fields**. Validate accuracy **by document type and field** before automating.</details>

**Q: What is stratified random sampling used for here?**
<details><summary>▶</summary>Measuring error rates in **high-confidence** extractions and **detecting novel error patterns**.</details>

**Q: How are field-level confidence scores made useful?**
<details><summary>▶</summary>**Calibrate** them using **labeled validation sets**, then route low-confidence/ambiguous extractions to human review.</details>

**Q: How do you preserve provenance through synthesis?**
<details><summary>▶</summary>Require subagents to output **claim→source mappings** (source URL, document name, excerpt, date) that downstream agents **preserve and merge**.</details>

**Q: Two credible sources report conflicting statistics. What do you do?**
<details><summary>▶</summary>**Annotate both values with source attribution** (and dates) rather than arbitrarily picking one; distinguish well-established from contested findings.</details>

**Q: Why require publication/collection dates in outputs?**
<details><summary>▶</summary>So that **temporal differences aren't misread as contradictions** (2024 data vs 2022 data isn't a conflict).</details>

---

# Part B — Alphabetical Glossary

**`--json-schema`** — Claude Code CLI flag that enforces a JSON schema on output; paired with `--output-format json` for structured CI findings.

**`--output-format json`** — Claude Code CLI flag producing machine-parseable JSON output (e.g., for posting inline PR comments).

**`--print` / `-p`** — Claude Code CLI flag for non-interactive (headless) mode: processes the prompt, prints to stdout, exits. The fix for CI jobs that hang on interactive input.

**`--resume <session-name>`** — Resumes a specific named prior session to continue an investigation across work sessions.

**`.claude/commands/`** — Project-scoped directory for custom slash commands, shared via version control (available to all on clone/pull). Contrast `~/.claude/commands/` (personal).

**`.claude/rules/`** — Directory of topic-specific rule files; with YAML `paths:` glob frontmatter, rules load conditionally only when editing matching files.

**`.claude/skills/`** — Project-scoped directory for skills; each skill has a `SKILL.md` with optional frontmatter (`context: fork`, `allowed-tools`, `argument-hint`).

**`.mcp.json`** — Project-scoped MCP server configuration, committed to the repo for shared team tooling; supports `${ENV_VAR}` expansion for secrets.

**`~/.claude.json`** — User-scoped MCP server configuration for personal/experimental servers (not shared via version control).

**`~/.claude/CLAUDE.md`** — User-level CLAUDE.md; applies only to that user and is **not** shared with teammates.

**AgentDefinition** — Configuration of a subagent type: description, system prompt, and tool restrictions.

**Agentic loop** — The repeating cycle: send request → inspect `stop_reason` → execute requested tools → append results → repeat, until `stop_reason == "end_turn"`.

**`allowed-tools` (skill frontmatter)** — Restricts which tools a skill may use during execution (e.g., read-only to prevent destructive actions).

**`allowedTools` (Agent SDK)** — The set of tools an agent may use; must include `"Task"` for a coordinator to spawn subagents.

**Attention dilution** — Degraded, uneven, or contradictory analysis when too much (e.g., a 14-file PR) is processed in a single pass. Cured by per-file + cross-file passes, not a bigger context window.

**`argument-hint` (skill frontmatter)** — Prompts the developer for required parameters when a skill is invoked without arguments.

**Batch (Message Batches API)** — Asynchronous bulk processing: ~50% cost savings, ≤24-hour window, no latency SLA, no multi-turn tool calling, `custom_id` correlation. For non-blocking workloads only.

**Business error** — A policy-violation failure (e.g., refund over limit); non-retryable; should return `retriable:false` plus a customer-friendly explanation.

**Calibration (confidence)** — Adjusting field-level confidence thresholds using labeled validation sets so confidence scores reliably route review attention.

**Case facts block** — A persistent block of exact transactional facts (amounts, dates, IDs, statuses) included in every prompt, outside summarized history, to survive progressive summarization.

**Claim-source mapping** — A structured pairing of each claim with its source (URL, document name, excerpt, date), preserved through synthesis to maintain provenance.

**CLAUDE.md** — Configuration/instruction file for Claude Code, organized in a hierarchy: user → project → directory.

**`/compact`** — Claude Code command that reduces context usage during extended sessions filled with verbose discovery output.

**Context degradation** — Loss of fidelity in long sessions; the model gives inconsistent answers and references generic "typical patterns" instead of specifics found earlier.

**`context: fork` (skill frontmatter)** — Runs a skill in an isolated sub-agent context so its (often verbose) output doesn't pollute the main conversation.

**Coordinator** — The hub agent that decomposes tasks, delegates to subagents, aggregates results, handles errors, and routes all inter-subagent communication.

**`custom_id`** — Identifier that correlates each Message Batches API request with its response; used to resubmit only failed items.

**Detected_pattern** — A field added to structured findings to track which constructs trigger them, enabling false-positive/dismissal analysis.

**Direct execution** — Making changes immediately without planning; appropriate for small, well-scoped, single-file changes.

**Edit** — Built-in tool for targeted file modification via a **unique** text anchor; falls back to Read + Write when the anchor isn't unique.

**`end_turn`** — `stop_reason` value meaning Claude has finished; terminates the agentic loop.

**Enum + "other"** — Schema pattern: an enumerated field plus an `"other"` value and a detail string, for extensible categories; `"unclear"` handles ambiguity.

**Environment-variable expansion** — Referencing secrets via `${VAR}` (e.g., `${GITHUB_TOKEN}`) in `.mcp.json` to avoid committing credentials.

**Escalation triggers** — Legitimate reasons to hand off to a human: explicit human request, policy gap/exception, or inability to make meaningful progress.

**Explore subagent** — A subagent that isolates verbose discovery output and returns summaries, preserving main-conversation context.

**Few-shot prompting** — Providing 2–4 targeted examples (ideally with reasoning) to improve format consistency, ambiguous-case handling, and generalization; reduces extraction hallucination.

**Field-level confidence** — Per-field confidence scores (calibrated on labeled data) used to route low-confidence extractions to human review.

**`fork_session`** — Creates independent branches from a shared analysis baseline to explore divergent approaches in parallel.

**Glob** — Built-in tool that matches file **paths** by pattern (e.g., `**/*.test.tsx`).

**Grep** — Built-in tool that searches file **content** for patterns (function names, error strings, imports).

**Hook** — Agent SDK mechanism to intercept the tool flow: `PostToolUse` (transform results) and tool-call interception (gate outgoing calls). Provides deterministic guarantees.

**Hub-and-spoke** — Architecture where the coordinator (hub) mediates all subagent (spoke) communication.

**Interview pattern** — Having Claude ask clarifying questions to surface considerations before implementing, useful in unfamiliar domains.

**`isError`** — MCP flag signaling a tool failure to the agent; should accompany structured metadata (category, retryable, message).

**Iterative refinement loop** — A coordinator loop that evaluates synthesis for gaps, re-delegates with targeted queries, and re-invokes synthesis until coverage is sufficient.

**Lost in the middle** — The tendency of models to reliably use the start and end of long inputs while omitting middle content.

**MCP (Model Context Protocol)** — Open protocol connecting Claude to backend systems via servers exposing tools and resources.

**MCP resource** — A way to expose content catalogs (issue summaries, doc hierarchies, schemas) so agents see available data without exploratory calls.

**`/memory`** — Claude Code command showing which memory/config files are loaded; used to diagnose inconsistent behavior.

**Model-driven decision-making** — Claude reasoning about which tool to call next based on context (vs hard-coded decision trees).

**Plan mode** — Claude Code mode for safe exploration and design before changes; for large-scale, architectural, multi-approach, multi-file work.

**`PostToolUse` hook** — Intercepts tool results before the model processes them, for normalization/transformation.

**Prerequisite gate** — A programmatic block preventing a downstream tool call until a prerequisite step completes (e.g., verify customer before refund).

**Progressive summarization** — Repeatedly condensing conversation history; risks losing exact figures, dates, and stated expectations.

**Prompt chaining** — Decomposing work into a fixed sequence of focused steps; suited to predictable, multi-aspect tasks.

**Pydantic** — Python validation library used for schema validation and semantic-error detection in validation-retry loops.

**Scratchpad file** — A file persisting key findings across context boundaries to counteract degradation in long sessions.

**Self-review limitation** — A model retains generation-time reasoning and is less likely to question its own output; independent instances review better.

**Session context isolation** — The principle that the session which generated code is less effective at reviewing it than an independent instance.

**Stratified random sampling** — Sampling high-confidence extractions across strata to measure error rates and detect novel patterns before automating review.

**Structured error context** — Failure type + attempted query + partial results + alternatives, enabling intelligent coordinator recovery.

**Subagent** — A specialized agent spawned by the coordinator via the `Task` tool; has isolated context and restricted tools.

**System prompt** — Instruction context for the model; keyword-sensitive wording can unintentionally bias tool selection.

**`Task` tool** — The mechanism for spawning subagents; `allowedTools` must include `"Task"`. Parallel subagents = multiple `Task` calls in one response.

**`tool_choice`** — API setting controlling tool calling: `"auto"` (text or tool), `"any"` (must call a tool), forced `{"type":"tool","name":"..."}` (specific tool).

**`tool_use`** — (1) `stop_reason` value meaning Claude wants to call a tool (continue the loop); (2) the API mechanism, with JSON schemas, for guaranteed structured output.

**Transient error** — A temporary failure (timeout, service unavailable); retryable.

**Validation-retry loop** — Re-prompting with the document, the failed extraction, and specific errors to guide correction; ineffective when info is absent from the source.
