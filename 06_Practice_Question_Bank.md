# CCA-F Practice Question Bank
### 45 new scenario-based questions + a recap of the 12 official samples

> **How to use this.** Each question mirrors the real exam: one realistic production scenario, four options, one best answer, three distractors that look right with incomplete knowledge. Answers are hidden in expandable blocks (`▶ Show answer`) so you can self-test. If your markdown viewer doesn't expand them, the answer key is also printed at the very bottom.
>
> **Drill method:** commit to an answer *before* expanding. If you get it wrong, re-read the cited task statement in the domain file. Getting the *reasoning* right matters more than the letter.

Distribution roughly tracks exam weighting: **D1 ×12, D2 ×8, D3 ×9, D4 ×9, D5 ×7.**

---

## Domain 1 — Agentic Architecture & Orchestration (Q1–Q12)

**Q1.** Your Agent SDK loop calls Claude, and the response comes back with `stop_reason: "tool_use"` and also contains a sentence of assistant text ("I'll look that up for you now."). Your code treats the presence of assistant text as "the agent is done" and ends the loop. Users complain the agent stops mid-task. What is the correct fix?

A) Keep ending on text, but add a regex to detect "I'll" phrases and continue only on those.
B) Drive the loop on `stop_reason`: continue while it is `"tool_use"`, terminate only on `"end_turn"`.
C) Set a maximum of 10 iterations and end the loop whenever that cap is hit or text appears.
D) Instruct the model in the system prompt to never emit text when it intends to call a tool.

<details><summary>▶ Show answer</summary>

**Correct: B.** The primary terminator must be `stop_reason`. `"tool_use"` means keep going; `"end_turn"` means stop. Assistant text can appear *alongside* a tool-use response, so text presence is not a completion signal (a named anti-pattern). **A** still relies on parsing natural language (anti-pattern). **C** uses an arbitrary iteration cap as the *primary* mechanism (anti-pattern; a cap is only a backstop). **D** tries to fix a control-flow bug with a probabilistic prompt instruction. *(Task 1.1)*
</details>

---

**Q2.** In your agentic loop, after Claude requests `lookup_order` and you execute it, the agent's next response ignores the order data and re-requests the same tool. What is the most likely implementation error?

A) You are not appending the tool result back into the conversation history before the next call to Claude.
B) You are using `stop_reason: "end_turn"` to continue the loop instead of `"tool_use"`.
C) The model needs a higher `max_tokens` to read the tool result.
D) You should switch from model-driven decisions to a fixed decision tree.

<details><summary>▶ Show answer</summary>

**Correct: A.** Tool results must be appended to the conversation context between iterations so the model can reason over them. If you don't feed the result back, the model never "sees" it and loops. **B** is backwards and would stop the loop, not repeat it. **C** is unrelated. **D** abandons the adaptive model-driven approach the exam favors. *(Task 1.1)*
</details>

---

**Q3.** A coordinator delegates to four subagents. The web-search subagent finds great sources, but the synthesis subagent writes a report that ignores them entirely. Logs show each subagent ran successfully. What is the most likely root cause?

A) The synthesis subagent's `tool_choice` is set to `"auto"` instead of `"any"`.
B) The coordinator never passed the web-search findings into the synthesis subagent's prompt; subagents don't inherit context automatically.
C) The synthesis subagent has too few tools.
D) The web-search subagent returned `isError: true`.

<details><summary>▶ Show answer</summary>

**Correct: B.** Subagents operate with isolated context and do **not** inherit the coordinator's history. The coordinator must place the search findings directly in the synthesis subagent's prompt. **A** affects whether a tool is called, not whether context was passed. **C** wouldn't cause it to ignore provided sources. **D** contradicts "each subagent ran successfully." *(Tasks 1.2, 1.3)*
</details>

---

**Q4.** You want your coordinator to spawn the document-analysis and web-search subagents to run *in parallel* to cut latency. What is the correct mechanism?

A) Emit two `Task` tool calls in a single coordinator response.
B) Call the first subagent, wait for `end_turn`, then call the second in the next turn.
C) Set `tool_choice: "any"` so both subagents fire automatically.
D) Increase `max_tokens` so both fit in one response.

<details><summary>▶ Show answer</summary>

**Correct: A.** Parallel subagents are spawned by emitting multiple `Task` tool calls in a *single* response. **B** serializes them (the opposite of parallel). **C** controls whether a tool is called, not parallelism. **D** is irrelevant to concurrency. (Also recall: the coordinator's `allowedTools` must include `"Task"`.) *(Task 1.3)*
</details>

---

**Q5.** A multi-agent research system on "the global semiconductor supply chain" returns a report that only discusses Taiwan. The coordinator's logs show it decomposed the topic into "TSMC capacity," "Taiwan export policy," and "Taiwan fab construction." What change best fixes coverage?

A) Give the synthesis agent gap-detection instructions.
B) Broaden the coordinator's decomposition to span all relevant regions/subtopics and add an iterative refinement loop that re-delegates for detected gaps.
C) Expand the web-search agent's queries with more keywords.
D) Loosen the document-analysis agent's relevance filter.

<details><summary>▶ Show answer</summary>

**Correct: B.** The logs show the *decomposition* was too narrow (Taiwan-only). The fix is broader decomposition plus an iterative refinement loop where the coordinator evaluates synthesis output for gaps and re-delegates. **A**, **C**, **D** blame downstream agents that executed their assigned scope correctly. *(Task 1.2)*
</details>

---

**Q6.** Your support agent must never call `process_refund` before `get_customer` has verified the customer's identity. A system-prompt instruction reduced violations but ~8% of refunds still skip verification, occasionally refunding the wrong account. What is the most effective change?

A) Add few-shot examples showing verification always happening first.
B) A programmatic prerequisite gate that blocks `process_refund` until `get_customer` returns a verified customer ID.
C) Strengthen the system prompt wording to say verification is "absolutely mandatory."
D) Lower the model temperature to make it follow instructions more reliably.

<details><summary>▶ Show answer</summary>

**Correct: B.** Identity-before-financial-operation is a deterministic requirement; only a programmatic gate/hook guarantees it. **A** and **C** are probabilistic (non-zero failure rate) and insufficient when errors are financial. **D** doesn't provide a guarantee. *(Tasks 1.4, 1.5)*
</details>

---

**Q7.** Three of your MCP tools return dates differently: Unix timestamps, ISO 8601 strings, and numeric status codes. The agent frequently misinterprets them. What is the cleanest deterministic fix?

A) Add a sentence to the system prompt explaining each format.
B) A `PostToolUse` hook that normalizes all tool results into one canonical date format before the model processes them.
C) A tool-call interception hook that blocks any tool returning a Unix timestamp.
D) Ask the model to convert formats itself at the start of every turn.

<details><summary>▶ Show answer</summary>

**Correct: B.** `PostToolUse` hooks intercept tool *results* for transformation/normalization before the model sees them — deterministic and reliable. **A** and **D** are probabilistic and waste reasoning. **C** misuses interception (which gates *outgoing calls*) and would break functionality. *(Task 1.5)*
</details>

---

**Q8.** Company policy: refunds over $500 must be handled by a human. You need a 100% guarantee. Which approach?

A) A tool-call interception hook that blocks `process_refund` when `amount > 500` and redirects to the human-escalation workflow.
B) A few-shot example showing the agent escalating a $600 refund.
C) A `PostToolUse` hook that flags refunds over $500 after they execute.
D) A system-prompt rule stating the agent must escalate refunds over $500.

<details><summary>▶ Show answer</summary>

**Correct: A.** Interception of the *outgoing* tool call blocks the action *before* it executes — a deterministic guarantee. **C** runs *after* the refund already happened (too late). **B** and **D** are probabilistic. *(Task 1.5)*
</details>

---

**Q9.** You must review a pull request that modifies 16 files. A single combined pass gives uneven depth and contradictory findings (a pattern flagged in one file, approved in another). What decomposition best fixes this?

A) Use a larger-context model so all 16 files fit at once.
B) Per-file local analysis passes plus a separate cross-file integration pass.
C) Require developers to submit no more than 4 files per PR.
D) Run the full-PR review three times and report only issues found in ≥2 runs.

<details><summary>▶ Show answer</summary>

**Correct: B.** This is attention dilution; the cure is decomposition into per-file passes (local issues) plus an integration pass (cross-file flow). **A** — bigger context doesn't fix attention quality. **C** shifts burden to developers. **D** suppresses real bugs caught only intermittently. *(Tasks 1.6, 4.6)*
</details>

---

**Q10.** You're told to "add comprehensive tests to a 300-file legacy service you've never seen." Which decomposition strategy fits?

A) A fixed prompt chain: write all unit tests, then all integration tests, then all e2e tests, in that order.
B) Dynamic/adaptive decomposition: first map structure, identify high-impact areas, then build a prioritized plan that adapts as dependencies are discovered.
C) Direct execution: start writing tests file-by-file from the top of the directory listing.
D) A single mega-prompt asking for tests for all 300 files at once.

<details><summary>▶ Show answer</summary>

**Correct: B.** Open-ended investigation calls for adaptive decomposition that generates subtasks from what's discovered. **A** is a fixed pipeline suited to *predictable* multi-aspect work, not open-ended exploration. **C** and **D** ignore the need to map structure and prioritize first. *(Task 1.6)*
</details>

---

**Q11.** Yesterday you ran a long Claude Code session named `payments-audit` mapping a complex flow. Today you've edited three files in that flow and want to continue the audit accurately. What's the best approach?

A) `--resume payments-audit`, and explicitly tell the agent which three files changed for targeted re-analysis.
B) `--resume payments-audit` and assume it will re-read everything automatically.
C) Start a brand-new session and re-explore the entire codebase from scratch.
D) Use `fork_session` to branch the audit into two parallel copies.

<details><summary>▶ Show answer</summary>

**Correct: A.** Resume the named session (context is mostly valid), but you must inform it which files changed so it re-analyzes those targeted areas instead of reasoning over stale analysis. **B** leaves it with stale state. **C** wastes the prior work. **D** is for exploring *divergent approaches*, not for handling a few edits. *(Task 1.7)*
</details>

---

**Q12.** From a shared codebase analysis, you want to compare two refactoring strategies without the explorations contaminating each other. What's the right tool?

A) `--resume` the same session twice.
B) `fork_session` to create independent branches from the shared analysis baseline.
C) Raise `max_tokens` and explore both in one linear session.
D) Run `/compact` between the two explorations.

<details><summary>▶ Show answer</summary>

**Correct: B.** `fork_session` creates independent branches from a shared baseline to explore divergent approaches in parallel. **A** would entangle the two in one history. **C** mixes them in a single context. **D** just compacts context; it doesn't branch. *(Task 1.7)*
</details>

---

## Domain 2 — Tool Design & MCP Integration (Q13–Q20)

**Q13.** Your agent has two tools: `search_kb` ("Searches the knowledge base") and `search_tickets` ("Searches tickets"). The agent keeps using the wrong one for ambiguous queries. What's the most effective *first* step?

A) Add 8 few-shot examples of correct routing to the system prompt.
B) Expand each tool's description with input formats, example queries, edge cases, and explicit "use this vs the other" boundaries.
C) Build a keyword router that pre-selects the tool before each turn.
D) Merge both into one `search_everything` tool.

<details><summary>▶ Show answer</summary>

**Correct: B.** Tool descriptions are the primary selection mechanism; thin descriptions are the root cause, and enriching them is the low-effort, high-leverage first step. **A** adds token overhead without fixing the root cause. **C** is over-engineered and bypasses the model's language understanding. **D** is a heavier architectural change than a "first step" warrants. *(Task 2.1)*
</details>

---

**Q14.** Two tools, `analyze_content` and `analyze_document`, have near-identical descriptions and are constantly confused. Besides editing descriptions, which structural change best removes the ambiguity?

A) Rename `analyze_content` to `extract_web_results` with a web-specific description so it no longer overlaps with `analyze_document`.
B) Give both tools to a single agent so it can pick freely.
C) Set `tool_choice: "auto"` on both.
D) Add a third tool, `analyze_data`, to cover the gap.

<details><summary>▶ Show answer</summary>

**Correct: A.** Renaming and rewriting to eliminate functional overlap is the recommended structural fix (you can also *split* a generic tool into purpose-specific ones). **B** worsens confusion. **C** doesn't address overlap. **D** adds more overlap. *(Task 2.1)*
</details>

---

**Q15.** An MCP tool `lookup_order` sometimes times out and sometimes is asked to act on a policy-violating request. Currently every failure returns `"Operation failed"`. The agent retries everything, including unretryable policy violations. What's the right design?

A) Return structured errors with `errorCategory` (transient/validation/permission), an `isRetryable` boolean, and a human-readable message.
B) Always return `isError: true` with no metadata so the agent stops on any failure.
C) Convert all failures to empty result sets so the agent moves on.
D) Retry every failure with exponential backoff up to 5 times.

<details><summary>▶ Show answer</summary>

**Correct: A.** Structured error metadata lets the agent choose recovery: retry transient errors, explain business errors. **B** loses the information needed to recover. **C** suppresses errors (empty-as-success anti-pattern). **D** wastes retries on non-retryable errors. *(Task 2.2)*
</details>

---

**Q16.** A search MCP tool runs successfully but finds zero matching records. How should it report this, and why does it matter?

A) Return `isError: true, errorCategory: "transient"` so the agent retries.
B) Return a successful response with an empty result set (`isError: false`), because a valid empty result is distinct from an access failure.
C) Return `"Operation failed"` so the coordinator terminates the workflow.
D) Return `isError: true` with no category.

<details><summary>▶ Show answer</summary>

**Correct: B.** A successful query with no matches is **not** an error; conflating it with an access failure (like a timeout) causes pointless retries or hides real failures. **A**, **C**, **D** misrepresent a valid empty result as a failure. *(Tasks 2.2, 5.3)*
</details>

---

**Q17.** Your synthesis subagent has been given all 18 platform tools "for flexibility," and it now frequently attempts web searches itself and mis-selects tools. Best fix?

A) Scope the synthesis agent to only its role's tools (~4–5), adding at most a limited cross-role tool for a specific high-frequency need.
B) Add few-shot examples of the synthesis agent not searching the web.
C) Keep all 18 tools but lower the temperature.
D) Force `tool_choice` to the synthesis tool on every call.

<details><summary>▶ Show answer</summary>

**Correct: A.** Too many tools degrades selection; scope each agent to its role (principle of least privilege), optionally adding a *scoped* cross-role tool for a common need. **B** patches symptoms. **C** doesn't reduce decision complexity. **D** would break multi-step synthesis that needs different tools across turns. *(Task 2.3)*
</details>

---

**Q18.** You must guarantee that `extract_metadata` is the *first* tool called before any enrichment tools run. Which configuration achieves this?

A) `tool_choice: "auto"`.
B) `tool_choice: "any"`.
C) Forced selection `tool_choice: {"type": "tool", "name": "extract_metadata"}`, then process enrichment in follow-up turns.
D) Remove all other tools permanently.

<details><summary>▶ Show answer</summary>

**Correct: C.** Forcing a specific named tool guarantees it runs first; subsequent steps proceed in follow-up turns. **A** lets the model return text or any tool. **B** guarantees *a* tool but not *which*. **D** breaks the enrichment that must follow. *(Tasks 2.3, 4.3)*
</details>

---

**Q19.** Your team needs a shared MCP server (an internal issue tracker) available to everyone who clones the repo, authenticating with a token that must not be committed. Where and how do you configure it?

A) In `~/.claude.json` with the token hard-coded.
B) In project-scoped `.mcp.json`, using environment-variable expansion (e.g., `${ISSUE_TOKEN}`) for the credential.
C) In the root `CLAUDE.md` as a documented command.
D) In `.claude/commands/` as a slash command.

<details><summary>▶ Show answer</summary>

**Correct: B.** Shared team servers go in project-scoped `.mcp.json` (committed), and env-var expansion keeps secrets out of version control. **A** is user-scoped (not shared) and leaks the secret. **C**/**D** are not where MCP servers are configured. *(Task 2.4)*
</details>

---

**Q20.** Your agent keeps using built-in `Grep` instead of your far more capable MCP search tool, which can query a semantic index. What's the most effective fix?

A) Disable the built-in `Grep` tool entirely.
B) Enhance the MCP tool's description to explain its capabilities and outputs in detail so the agent prefers it over the built-in.
C) Force `tool_choice` to the MCP tool on every turn.
D) Move the MCP server config from `.mcp.json` to `~/.claude.json`.

<details><summary>▶ Show answer</summary>

**Correct: B.** Adoption is a description problem: rich MCP tool descriptions prevent the agent from defaulting to built-ins like `Grep`. **A** is heavy-handed and removes a useful tool. **C** breaks other legitimate uses. **D** changes scope, not selection behavior. *(Task 2.4)*
</details>

---

## Domain 3 — Claude Code Configuration & Workflows (Q21–Q29)

**Q21.** A newly onboarded engineer reports that Claude Code ignores the team's coding standards that everyone else seems to benefit from. The standards currently live in `~/.claude/CLAUDE.md` on the tech lead's machine. What's the fix?

A) Move the standards into project-level config (`.claude/CLAUDE.md` or root `CLAUDE.md`) committed to the repo.
B) Have the new engineer copy the tech lead's `~/.claude/CLAUDE.md` manually.
C) Put the standards in `.claude/commands/`.
D) Add the standards to a `.claude/config.json` file.

<details><summary>▶ Show answer</summary>

**Correct: A.** User-level `~/.claude/CLAUDE.md` is private to that user and not shared via version control — that's why the new engineer doesn't get it. Project-level config is shared on clone/pull. **B** is a manual, non-scalable hack. **C** is for commands. **D** references a file that doesn't exist. *(Task 3.1)*
</details>

---

**Q22.** Your root `CLAUDE.md` has grown to 700 lines and is hard to maintain. Which approach keeps it modular and maintainable?

A) Delete most of it and rely on the model's defaults.
B) Use `@import` to reference external standards files and/or split topics into `.claude/rules/` files (e.g., `testing.md`, `deployment.md`).
C) Move everything into a single subdirectory `CLAUDE.md`.
D) Convert it into a slash command.

<details><summary>▶ Show answer</summary>

**Correct: B.** `@import` keeps CLAUDE.md modular, and `.claude/rules/` organizes topic-specific files as an alternative to a monolith. **A** loses the standards. **C** just relocates the monolith and limits it to one directory. **D** misuses commands. (Tip: use `/memory` to verify what's loaded.) *(Task 3.1)*
</details>

---

**Q23.** You want a `/deploy-check` slash command available to every developer automatically when they pull the repo. Where do you put it?

A) `~/.claude/commands/deploy-check.md`.
B) `.claude/commands/deploy-check.md` in the repo.
C) Root `CLAUDE.md` under a "Commands" header.
D) `.claude/config.json` with a `commands` array.

<details><summary>▶ Show answer</summary>

**Correct: B.** Project-scoped commands in `.claude/commands/` are version-controlled and shared on clone/pull. **A** is personal/not shared. **C** is for instructions, not command definitions. **D** is a non-existent mechanism. *(Task 3.2)*
</details>

---

**Q24.** You have a "codebase-analysis" skill that produces hundreds of lines of discovery output, which clutters the main conversation and exhausts context. It should also be prevented from writing or deleting files. Which frontmatter configuration fits?

A) `context: fork` plus `allowed-tools` limited to read-only tools (e.g., Read, Grep, Glob).
B) `context: main` plus `allowed-tools: [Write, Bash]`.
C) `argument-hint` only.
D) Put the skill in `~/.claude/skills/` so it doesn't affect the main context.

<details><summary>▶ Show answer</summary>

**Correct: A.** `context: fork` isolates verbose output in a sub-agent context; `allowed-tools` restricts the skill to read-only tools, preventing destructive actions. **B** keeps output in the main context and grants destructive tools. **C** only prompts for args. **D** changes scope (personal vs shared), not context isolation. *(Task 3.2)*
</details>

---

**Q25.** Test files live throughout the codebase (`Foo.test.tsx` next to `Foo.tsx`), and you want identical test conventions applied automatically wherever a test file is edited, regardless of directory. Best mechanism?

A) A `.claude/rules/` file with YAML frontmatter `paths: ["**/*.test.tsx"]`.
B) A subdirectory `CLAUDE.md` in each folder that contains tests.
C) A skill the developer invokes before editing tests.
D) A single "Testing" header in the root `CLAUDE.md`.

<details><summary>▶ Show answer</summary>

**Correct: A.** Glob-pattern path rules in `.claude/rules/` load automatically based on file path and follow the file *type* across directories. **B** is directory-bound and can't cover scattered files. **C** requires manual invocation (not automatic). **D** relies on the model inferring which section applies. *(Task 3.3)*
</details>

---

**Q26.** You must migrate a library across ~50 files, choosing between two integration approaches with different infrastructure implications. Plan mode or direct execution?

A) Direct execution, fixing issues as they appear.
B) Plan mode, to explore dependencies and design the approach before changing code.
C) Direct execution with exhaustive upfront instructions.
D) Start direct, switch to plan only if it gets complicated.

<details><summary>▶ Show answer</summary>

**Correct: B.** Plan mode is for large-scale, multi-file changes with multiple valid approaches and architectural decisions — exactly this. **A** risks costly rework. **C** presumes you already know the right design. **D** ignores that the complexity is already known. *(Task 3.4)*
</details>

---

**Q27.** During a multi-phase task, an exploration step generates huge volumes of discovery output that threatens to exhaust the main conversation's context. What's the recommended way to contain it?

A) Increase `max_tokens`.
B) Use the `Explore` subagent to isolate the verbose discovery and return a summary to the main conversation.
C) Disable tools during exploration.
D) Switch to direct execution.

<details><summary>▶ Show answer</summary>

**Correct: B.** The `Explore` subagent isolates verbose discovery output and returns summaries, preserving main-conversation context. **A** doesn't isolate output. **C** prevents exploration. **D** is unrelated to context isolation. *(Tasks 3.4, 5.4)*
</details>

---

**Q28.** Your CI job runs `claude "Review this PR"` and hangs forever waiting for interactive input. What's the correct fix, and what output configuration lets you post inline PR comments?

A) Set `CLAUDE_HEADLESS=true`; parse stdout text.
B) Add `-p` (non-interactive) and use `--output-format json` with `--json-schema` for machine-parseable findings.
C) Add `--batch` and poll for completion.
D) Redirect stdin from `/dev/null`.

<details><summary>▶ Show answer</summary>

**Correct: B.** `-p`/`--print` runs non-interactively (processes, prints, exits); `--output-format json` + `--json-schema` produce structured findings for inline PR comments. **A** (`CLAUDE_HEADLESS`) and **C** (`--batch`) are non-existent features. **D** is a Unix workaround that doesn't address the command syntax. *(Task 3.6)*
</details>

---

**Q29.** Your CI reviewer re-runs on every commit and keeps re-posting the *same* comments developers already addressed. What change reduces duplicate comments?

A) Include prior review findings in context and instruct Claude to report only new or still-unaddressed issues.
B) Switch the reviewer to a larger-context model.
C) Have the same session that generated the code review it.
D) Run the review three times and post the consensus.

<details><summary>▶ Show answer</summary>

**Correct: A.** Feeding prior findings into context and asking for only new/unaddressed issues prevents duplicate comments across re-runs. **B** doesn't address duplication. **C** weakens review (self-review retains generation bias). **D** suppresses intermittently-caught issues. *(Task 3.6)*
</details>

---

## Domain 4 — Prompt Engineering & Structured Output (Q30–Q38)

**Q30.** Your CI review bot flags many trivial style nits; developers now ignore even its accurate security findings. You already told it to "be conservative and only report high-confidence issues." What's the most effective change?

A) Add "be extra conservative" and raise an internal confidence threshold.
B) Replace vague hedges with explicit categorical criteria (report: bugs, security; skip: minor style, local patterns), and temporarily disable the noisy category while you improve it.
C) Ask the model to self-rate confidence 1–10 and suppress anything under 8.
D) Switch to a smaller model to reduce verbosity.

<details><summary>▶ Show answer</summary>

**Correct: B.** Explicit categorical criteria beat vague hedges; and since one high-false-positive category undermines trust in all categories, temporarily disabling it restores trust while you fix it. **A** and **C** rely on poorly-calibrated confidence and don't add specific criteria. **D** is irrelevant. *(Task 4.1)*
</details>

---

**Q31.** Detailed written instructions still produce inconsistently-formatted findings and inconsistent handling of ambiguous cases. What technique most reliably improves consistency and ambiguous-case judgment?

A) 2–4 targeted few-shot examples that show the reasoning for the chosen action and demonstrate the exact output format.
B) A longer, more detailed prose specification.
C) Lower temperature to 0.
D) A JSON schema with all fields required.

<details><summary>▶ Show answer</summary>

**Correct: A.** Few-shot examples are the most effective technique for consistent format and ambiguous-case handling, and showing the *reasoning* enables generalization to novel cases. **B** is the approach that already failed. **C** reduces randomness but not format/judgment design. **D** addresses structure, not format consistency or ambiguous reasoning. *(Task 4.2)*
</details>

---

**Q32.** Your extraction pipeline occasionally emits malformed JSON (trailing commas, unescaped quotes) that breaks downstream parsing. What's the most reliable way to guarantee schema-compliant output?

A) Post-process the text with a JSON repair library.
B) Use `tool_use` with a JSON schema and read the structured data from the tool-use response.
C) Add "output valid JSON only" to the prompt.
D) Lower `max_tokens` so the JSON is shorter.

<details><summary>▶ Show answer</summary>

**Correct: B.** `tool_use` with JSON schemas is the most reliable approach and eliminates JSON syntax errors. **A** is a fragile band-aid. **C** is probabilistic. **D** is irrelevant. (Remember: this kills *syntax* errors, not *semantic* ones.) *(Task 4.3)*
</details>

---

**Q33.** Documents you extract from sometimes lack a `tax_id`. With `tax_id` marked `required`, the model invents plausible-looking values. What schema change prevents fabrication?

A) Add few-shot examples of real tax IDs.
B) Make `tax_id` optional/nullable so the model can return null when the value is absent.
C) Force `tool_choice` to the extraction tool.
D) Add a regex pattern constraint to `tax_id`.

<details><summary>▶ Show answer</summary>

**Correct: B.** Making fields nullable/optional when the source may not contain them prevents the model from fabricating values to satisfy required fields. **A** could *encourage* pattern-matching fabrication. **C** controls *whether* the tool runs, not fabrication. **D** constrains format but still demands a (possibly invented) value. *(Task 4.3)*
</details>

---

**Q34.** Your extraction passes JSON schema validation, but line items frequently don't sum to the stated total. What's the right reliability design?

A) Trust the schema; if it validated, the data is correct.
B) Add semantic validation: extract `calculated_total` alongside `stated_total`, flag discrepancies, and retry with the document + failed extraction + the specific error.
C) Mark `total` as nullable.
D) Switch to free-text output and parse it.

<details><summary>▶ Show answer</summary>

**Correct: B.** Tool-use schemas eliminate *syntax* errors but not *semantic* ones (sums, field placement). Validate semantics yourself and retry with specific error feedback. **A** ignores semantic errors. **C** hides the problem. **D** reintroduces syntax errors. *(Tasks 4.3, 4.4)*
</details>

---

**Q35.** An extraction fails because a required figure exists only in a *separate* document you never provided to the model. You add an automatic retry loop. What happens?

A) The retry will succeed after a few attempts.
B) The retry will fail repeatedly and waste cost, because retries can't recover information that's absent from the provided source.
C) The retry will succeed only at temperature 0.
D) The retry will succeed if you force `tool_choice`.

<details><summary>▶ Show answer</summary>

**Correct: B.** Retries help with format/structural errors but are ineffective when the information is simply absent from the source. Recognize unwinnable retries and stop. **A**, **C**, **D** assume a retry can conjure missing data. *(Task 4.4)*
</details>

---

**Q36.** You run two workflows: a blocking pre-merge check developers wait on, and a weekly architecture audit. You want the 50% savings of the Message Batches API. How should you apply it?

A) Batch both; poll for completion.
B) Batch only the weekly audit (latency-tolerant); keep the synchronous API for the blocking pre-merge check.
C) Keep both synchronous to avoid batch ordering problems.
D) Batch both with a real-time fallback if batches run long.

<details><summary>▶ Show answer</summary>

**Correct: B.** The Batch API offers ~50% savings but up to 24-hour processing and no latency SLA — fine for the weekly audit, unsuitable for a blocking pre-merge check. **A** risks blocking on an SLA-less API. **C** wrongly assumes ordering issues (`custom_id` correlates results). **D** adds needless complexity. *(Task 4.5)*
</details>

---

**Q37.** You submit 100 documents via the Message Batches API; 6 fail because they exceeded context limits. What's the correct handling?

A) Resubmit the entire batch of 100.
B) Identify the 6 failures by `custom_id` and resubmit only those, chunking the oversized documents.
C) Abandon the batch and process all 100 synchronously.
D) Mark the 6 as empty results and continue.

<details><summary>▶ Show answer</summary>

**Correct: B.** Use `custom_id` to correlate and resubmit only the failed documents, applying fixes like chunking oversized inputs. **A** wastes cost. **C** discards batch savings. **D** silently drops data. *(Task 4.5)*
</details>

---

**Q38.** A single Claude session generated a module and is now asked to review its own output; it misses subtle bugs. What architecture catches more issues?

A) Tell the same session to "think harder" / use extended thinking.
B) Use a second, independent Claude instance without the generator's reasoning context to review the code.
C) Increase the context window.
D) Have the session review itself three times and vote.

<details><summary>▶ Show answer</summary>

**Correct: B.** A model retains reasoning context from generation and is less likely to question its own decisions; an independent instance (no shared context) catches more subtle issues. **A** and **D** keep the generator's bias. **C** doesn't fix self-review bias. *(Task 4.6)*
</details>

---

## Domain 5 — Context Management & Reliability (Q39–Q45)

**Q39.** In a 40-turn billing dispute, the agent loses the exact disputed amount and date after several summarization steps and later quotes a wrong refund figure. What's the most effective mitigation?

A) Summarize more aggressively to save tokens.
B) Extract transactional facts (amount, date, order number, status) into a persistent "case facts" block included verbatim in every prompt, outside the summarized history.
C) Lower the temperature.
D) Truncate the oldest turns and hope the figures were recent.

<details><summary>▶ Show answer</summary>

**Correct: B.** Progressive summarization risks condensing exact numbers/dates into vague summaries; pulling them into a persistent case-facts block keeps them intact. **A** worsens the loss. **C** is unrelated. **D** can drop the very facts you need. *(Task 5.1)*
</details>

---

**Q40.** You aggregate findings from many sources into one long input for synthesis. You worry important middle findings get ignored. What addresses the "lost in the middle" effect?

A) Place a key-findings summary at the beginning, organize details under explicit section headers, and keep critical items near the start/end.
B) Randomly shuffle the findings.
C) Put the least important findings first.
D) Remove all section headers to save tokens.

<details><summary>▶ Show answer</summary>

**Correct: A.** Models reliably use the beginning and end of long inputs; lead with key findings, use explicit section headers, and avoid burying critical items in the middle. **B**, **C**, **D** make position effects worse. *(Task 5.1)*
</details>

---

**Q41.** Your support agent escalates easy cases (standard replacements with photos) but tries to resolve hard policy-exception cases itself; first-contact resolution is well below target. What's the best fix?

A) Add explicit escalation criteria with few-shot examples showing when to escalate vs resolve autonomously.
B) Have the agent self-report confidence (1–10) and auto-escalate below a threshold.
C) Train a separate classifier on historical tickets.
D) Escalate whenever sentiment analysis detects frustration.

<details><summary>▶ Show answer</summary>

**Correct: A.** The root cause is unclear decision boundaries; explicit criteria plus few-shot examples fix it proportionately. **B** relies on poorly-calibrated self-confidence (the agent is already wrongly confident on hard cases). **C** is over-engineered before prompt optimization. **D** uses sentiment, which doesn't correlate with complexity. *(Task 5.2)*
</details>

---

**Q42.** `get_customer` returns two customers matching the provided name. What should the agent do?

A) Pick the most recently active account.
B) Ask the customer for an additional identifier (email, order number) to disambiguate, rather than guessing.
C) Pick the first result.
D) Escalate to a human immediately.

<details><summary>▶ Show answer</summary>

**Correct: B.** Multiple matches require clarification — request additional identifiers — not heuristic selection. **A** and **C** are heuristics that risk the wrong account. **D** over-escalates a resolvable ambiguity. *(Task 5.2)*
</details>

---

**Q43.** A web-search subagent times out. Which error-propagation design best enables the coordinator to recover intelligently?

A) Return structured error context: failure type, attempted query, any partial results, and possible alternatives.
B) Retry silently with backoff, then return a generic "search unavailable."
C) Return an empty result set marked as successful.
D) Throw to a top-level handler that terminates the whole research workflow.

<details><summary>▶ Show answer</summary>

**Correct: A.** Structured error context lets the coordinator retry with a modified query, try alternatives, or proceed with partial results. **B** hides context behind a generic status. **C** suppresses the error (empty-as-success). **D** kills the workflow when recovery was possible. *(Task 5.3)*
</details>

---

**Q44.** During a long codebase exploration, the agent starts giving inconsistent answers and refers to "typical patterns" instead of the specific classes it found earlier. Which combination best counteracts this?

A) Keep going in one session and raise `max_tokens`.
B) Maintain scratchpad files of key findings, delegate verbose investigations to subagents, and use `/compact`; design crash recovery via structured state manifests.
C) Restart from scratch every 20 minutes.
D) Disable tools to reduce output.

<details><summary>▶ Show answer</summary>

**Correct: B.** Scratchpad files persist findings across context boundaries, subagent delegation isolates verbose output, `/compact` reduces context usage, and state manifests enable crash recovery. **A** ignores degradation. **C** wastes work. **D** prevents exploration. *(Task 5.4)*
</details>

---

**Q45.** Your extraction system reports 97% aggregate accuracy, and leadership wants to drop human review. What should you do before automating?

A) Trust the 97% and automate everything.
B) Analyze accuracy by document type and field, use stratified random sampling of high-confidence extractions, and route low-confidence/ambiguous cases to humans using calibrated field-level confidence.
C) Route a fixed random 50% to humans regardless of confidence.
D) Drop review only for the longest documents.

<details><summary>▶ Show answer</summary>

**Correct: B.** Aggregate accuracy can mask poor performance on specific types/fields; segment the accuracy, use stratified sampling to catch novel error patterns, and route by calibrated field-level confidence. **A** trusts a number that can hide bad segments. **C** ignores confidence/segment risk. **D** uses an irrelevant proxy (length). *(Task 5.5)*
</details>

---

## Recap of the 12 official sample questions (from the exam guide)

You already have these in the guide; here is a fast-recall table so you can confirm you know each cold. Re-derive the reasoning, not just the letter.

| # | Scenario | Tests | Answer | One-line reason |
|---|---|---|---|---|
| 1 | Support | Enforcement vs prompt | **A** | Identity-before-refund needs a *programmatic prerequisite*, not a prompt/few-shot (probabilistic). |
| 2 | Support | Tool selection | **B** | Thin descriptions are the root cause; *expand descriptions* (inputs, examples, boundaries) first. |
| 3 | Support | Escalation calibration | **A** | Explicit escalation criteria + few-shot; self-confidence/sentiment/classifiers are wrong proxies. |
| 4 | Code Gen | Command scope | **A** | Shared command → `.claude/commands/` (version-controlled). |
| 5 | Code Gen | Plan vs direct | **A** | Monolith→microservices is architectural/multi-file → *plan mode* before changes. |
| 6 | Code Gen | Path rules | **A** | Scattered test files → `.claude/rules/` glob `**/*.test.tsx`, not directory CLAUDE.md. |
| 7 | Research | Decomposition root cause | **B** | Coordinator decomposed too narrowly (visual-arts only); subagents worked correctly. |
| 8 | Research | Error propagation | **A** | Return *structured error context* (type, query, partial results, alternatives). |
| 9 | Research | Tool distribution | **A** | Scoped `verify_fact` for the 85% simple case; route complex cases via coordinator. |
| 10 | CI | Non-interactive CLI | **A** | `-p`/`--print`; `CLAUDE_HEADLESS` and `--batch` don't exist. |
| 11 | CI | Batch vs sync | **A** | Batch the overnight report; keep real-time for the blocking pre-merge check. |
| 12 | CI | Multi-pass review | **A** | Per-file passes + cross-file integration pass cures attention dilution. |

---

## Answer key (for viewers without expandable blocks)

| Q | A | Q | A | Q | A | Q | A | Q | A |
|---|---|---|---|---|---|---|---|---|---|
| 1 | B | 10 | B | 19 | B | 28 | B | 37 | B |
| 2 | A | 11 | A | 20 | B | 29 | A | 38 | B |
| 3 | B | 12 | B | 21 | A | 30 | B | 39 | B |
| 4 | A | 13 | B | 22 | B | 31 | A | 40 | A |
| 5 | B | 14 | A | 23 | B | 32 | B | 41 | A |
| 6 | B | 15 | A | 24 | A | 33 | B | 42 | B |
| 7 | B | 16 | B | 25 | A | 34 | B | 43 | A |
| 8 | A | 17 | A | 26 | B | 35 | B | 44 | B |
| 9 | B | 18 | C | 27 | B | 36 | B | 45 | B |

**Scoring yourself:** the real exam passes at 720/1000 (~72%). On this 45-question bank, treat **33+/45 correct** as a comfortable pass signal, **29–32** as borderline (review weak domains), and **<29** as "more study needed." Weight your review toward Domain 1 (most questions) and any domain where you missed the *reasoning*, not just the letter.
