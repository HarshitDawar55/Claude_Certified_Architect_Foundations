# Domain 1 — Agentic Architecture & Orchestration
### Weight: 27% (the single largest domain) · Task statements 1.1 – 1.7

> **Why this domain dominates.** More than a quarter of the exam lives here. Domain 1 is the "how do you build and wire up agents" domain: the loop that drives a single agent, the coordinator-subagent patterns that drive many agents, how context flows between them, how you *guarantee* ordering, how hooks intercept the flow, how you decompose work, and how you persist and branch sessions. Scenarios 1 (Customer Support), 3 (Multi-Agent Research), and 4 (Developer Productivity) are the main hunting grounds.

**The mental model to carry through every task statement below:** Claude-driven agents decide *what to do next* by reasoning over the conversation history. Your job as the architect is to (a) keep that loop running correctly, (b) give the model the right context and tools at each step, and (c) insert **deterministic** controls (hooks, prerequisite gates, forced tool choice) wherever a probabilistic model decision is not good enough.

---

## Task Statement 1.1 — Design and implement agentic loops for autonomous task execution

### The real-world problem
You are building the **Customer Support Resolution Agent** (Scenario 1). A customer writes: *"I want to return order #12345 and get a refund."* Your agent must call `get_customer`, then `lookup_order`, decide whether the return is allowed, possibly call `process_refund`, and finally reply. None of that is a single API call — it is a *loop* of "ask Claude → Claude requests a tool → you run the tool → you feed the result back → ask Claude again" until Claude has everything it needs and produces a final answer.

If you get the loop's control flow wrong, the agent either stops too early (replying before it has finished the refund) or never stops (spinning forever).

### What you must know (the agentic loop lifecycle)
The loop has four repeating steps:

1. **Send the request** to Claude (conversation history + available tools).
2. **Inspect `stop_reason`** on the response.
   - `stop_reason == "tool_use"` → Claude wants to call one or more tools. The work is **not** done.
   - `stop_reason == "end_turn"` → Claude is finished and has produced its final answer. **Terminate the loop.**
3. **Execute the requested tool(s)** that Claude asked for.
4. **Return the tool results** by appending them to the conversation history, then go back to step 1.

The reason step 4 matters: **tool results are appended to conversation history so the model can reason about its next action.** Each iteration, Claude sees the full accumulated transcript — the original request, every tool call it made, and every result you returned — and uses that to decide the next move.

The deepest conceptual point in 1.1 is the distinction between two philosophies of control:

- **Model-driven decision-making** (correct for agents): Claude reasons about *which tool to call next* based on the current context. The path is not hard-coded; it adapts to what the tools returned. If `lookup_order` reveals the order is already refunded, Claude can change course.
- **Pre-configured decision trees / fixed tool sequences** (the thing you are usually *not* supposed to do): a rigid `if order then refund` script. This is brittle and defeats the purpose of an agent — it cannot adapt to the messy reality of returns, billing disputes, and account issues.

### What you'd actually do (skills)
- **Implement control flow driven purely by `stop_reason`:** `while stop_reason == "tool_use": run tools; append results; call Claude again`. Exit the loop the moment `stop_reason == "end_turn"`.
- **Append every tool result to context between iterations** so the model incorporates new information into its reasoning (the order status, the customer ID, the refund confirmation).

```python
# Canonical agentic loop (pseudocode)
messages = [{"role": "user", "content": user_request}]
while True:
    response = claude.create(messages=messages, tools=TOOLS)
    if response.stop_reason == "end_turn":
        return response            # <-- Claude is done. Terminate.
    if response.stop_reason == "tool_use":
        messages.append(response)                  # record Claude's tool request
        tool_results = run_tools(response.tool_calls)
        messages.append({"role": "user",
                         "content": tool_results}) # feed results back as context
        continue                                   # loop again
```

### Anti-patterns the exam will try to bait you with
These are explicitly called out as **anti-patterns** — recognize them as *wrong answers*:

- **Parsing natural-language signals to decide when to stop** (e.g., looking for the word "done" or "I have completed" in Claude's text). Unreliable; `stop_reason` is the real signal.
- **Using an arbitrary iteration cap as the *primary* stopping mechanism** (e.g., "stop after 10 loops"). A safety cap is fine as a backstop, but the *primary* terminator must be `stop_reason == "end_turn"`.
- **Checking for assistant text content as a completion indicator** (e.g., "if the message has text, we're done"). Claude can emit text *and* still request tools in the same turn; text presence does not mean completion.

### Exam signals & facts to memorize
- The two `stop_reason` values you care about: **`"tool_use"` (keep going)** and **`"end_turn"` (stop)**.
- **Continue on `tool_use`, terminate on `end_turn`.** Nothing else is the primary terminator.
- Tool results go back into the conversation as new context every iteration.
- Model-driven > hard-coded decision trees for adaptive tasks.

---

## Task Statement 1.2 — Orchestrate multi-agent systems with coordinator-subagent patterns

### The real-world problem
You are building the **Multi-Agent Research System** (Scenario 3). A **coordinator** agent must research a topic and delegate to specialized **subagents**: one searches the web, one analyzes documents, one synthesizes findings, one generates the report. You run it on *"impact of AI on creative industries"* and the final report covers only visual arts — it completely misses music, writing, and film. Every subagent succeeded at its task. So why is the output incomplete? (This is literally Sample Question 7 — answer below.)

### What you must know (hub-and-spoke architecture)
- **Hub-and-spoke:** the coordinator is the hub. It manages **all** inter-subagent communication, error handling, and information routing. Subagents do not talk to each other directly — everything flows through the coordinator. This gives you **observability, consistent error handling, and controlled information flow**.
- **Subagents operate with isolated context.** A subagent does **not** automatically inherit the coordinator's conversation history. If the synthesis agent needs the web-search results, the coordinator must *hand them over explicitly* (see 1.3).
- **The coordinator's four jobs:** task **decomposition**, **delegation**, **result aggregation**, and **deciding which subagents to invoke** based on query complexity.
- **The classic failure mode: overly narrow task decomposition.** If the coordinator breaks "creative industries" into only "digital art," "graphic design," and "photography," the subagents will faithfully research exactly those — and the broad topic is under-covered. *The bug is in the decomposition, not the subagents.* This is the root cause in Sample Question 7 (answer: **B**, the coordinator's decomposition is too narrow).

### What you'd actually do (skills)
- **Design coordinators that analyze the query and dynamically select subagents** rather than always routing through the full pipeline. A simple factual lookup should not necessarily spin up the document-analysis and synthesis agents.
- **Partition research scope across subagents to minimize duplication** — assign each agent a *distinct* subtopic or source type (e.g., agent A handles academic papers, agent B handles news; or agent A handles "music," agent B handles "film"). This directly prevents the Scenario-3 coverage gap.
- **Implement iterative refinement loops:** after synthesis, the coordinator **evaluates the output for gaps**, re-delegates to search/analysis subagents with **targeted queries** for the missing areas, and re-invokes synthesis until coverage is sufficient. (This is the structural fix that would have caught the missing "music/writing/film" coverage.)
- **Route all subagent communication through the coordinator** for observability, consistent error handling, and controlled information flow.

### Anti-patterns / distractors
- Blaming a downstream agent (synthesis, web search, document analysis) when the **logs show** the coordinator decomposed the topic wrongly. Working agents executing a bad plan ≠ broken agents.
- Letting subagents communicate peer-to-peer (you lose observability and centralized error handling).
- Always running the entire pipeline regardless of query complexity (wasteful and can dilute focus).

### Exam signals & facts to memorize
- **Coordinator = hub; subagents = spokes; all communication flows through the hub.**
- **Subagents start with isolated context — no automatic inheritance.**
- **Narrow decomposition is the #1 multi-agent coverage bug — diagnose it from the coordinator's logs.**
- The fix for coverage gaps is **better/broader decomposition + an iterative refinement loop**, not patching the downstream agents.

---

## Task Statement 1.3 — Configure subagent invocation, context passing, and spawning

### The real-world problem
Back in the research system: your coordinator needs to actually *spawn* the synthesis subagent and *give it* the web-search results and document analyses. You discover the synthesis agent is producing reports that ignore the sources the search agent found. Why? Because you assumed the subagent could "see" the coordinator's history. It can't.

### What you must know
- **The `Task` tool is the mechanism for spawning subagents.** For a coordinator to invoke subagents, its **`allowedTools` must include `"Task"`.** If `Task` isn't in `allowedTools`, the coordinator literally cannot delegate.
- **Subagent context must be provided explicitly in the prompt.** Subagents do **not** inherit parent context or share memory across invocations. Whatever the subagent needs to know must be written into *its* prompt.
- **`AgentDefinition`** configures each subagent type: its **description**, **system prompt**, and **tool restrictions** (which tools that subagent type may use).
- **Fork-based session management** lets you explore divergent approaches from a shared analysis baseline (covered more in 1.7).

### What you'd actually do (skills)
- **Pass complete findings from prior agents directly into the subagent's prompt.** E.g., put the web-search results *and* the document-analysis outputs into the synthesis subagent's prompt — don't assume it can retrieve them.
- **Use structured data formats to separate content from metadata** when passing context, so attribution survives. Keep the claim/text separate from its source URL, document name, and page number. (This is what later lets the synthesis agent cite correctly — see Domain 5.6.)
- **Spawn parallel subagents by emitting multiple `Task` tool calls in a single coordinator response** — not across separate turns. Multiple `Task` calls in one assistant turn run in parallel; issuing them one per turn serializes them and adds latency. (Exercise 4 has you measure exactly this latency improvement.)
- **Write coordinator prompts that specify goals and quality criteria, not step-by-step procedures.** Tell the subagent *what good looks like* (research goal, depth, citation requirements) and let it adapt, rather than scripting every step — that preserves subagent adaptability.

```text
# Coordinator → synthesis subagent prompt (explicit context passing)
GOAL: Produce a cited synthesis on "AI in music production."
QUALITY CRITERIA: Every claim must cite a source; flag any conflicting figures.

WEB SEARCH FINDINGS (provided — you cannot retrieve these yourself):
- claim: "...", source_url: "...", date: "2024-11"
- ...

DOCUMENT ANALYSIS FINDINGS (provided):
- claim: "...", document: "Smith2023.pdf", page: 12
- ...
```

### Anti-patterns / distractors
- Expecting a subagent to "remember" earlier work or read the coordinator's transcript automatically.
- Forgetting to add `"Task"` to `allowedTools` and wondering why delegation fails.
- Emitting `Task` calls one per turn when you intended them to run in parallel.
- Over-specifying procedural steps to a subagent, which kills its ability to adapt to what it finds.

### Exam signals & facts to memorize
- **`Task` tool spawns subagents; coordinator needs `"Task"` in `allowedTools`.**
- **No automatic context inheritance — pass everything in the prompt.**
- **Parallel = multiple `Task` calls in ONE response.**
- **Goals + quality criteria > procedural scripts** for subagent prompts.
- Keep **content separate from metadata** (source/date/page) to preserve attribution.

---

## Task Statement 1.4 — Implement multi-step workflows with enforcement and handoff patterns

### The real-world problem
Production data on the support agent shows that in **12% of cases the agent skips `get_customer`** and calls `lookup_order` using only the customer's stated name — sometimes refunding the wrong account (Sample Question 1). You've already written "you must verify the customer first" into the system prompt. It still happens 12% of the time. How do you make verification *actually* mandatory?

### What you must know (enforcement vs guidance)
- **Programmatic enforcement** (hooks, **prerequisite gates**) vs **prompt-based guidance.** A prompt instruction is a *request*; the model complies most of the time but not always. **Prompt instructions alone have a non-zero failure rate.**
- **When deterministic compliance is required** — e.g., **identity verification before any financial operation** — prompt instructions are insufficient. You need a programmatic block. (This is why Sample Question 1's answer is **A**, a programmatic prerequisite, not **B**/system prompt or **C**/few-shot, which are both probabilistic.)
- **Structured handoff protocols** for mid-process escalation must carry everything the human needs: **customer details, root-cause analysis, and recommended actions** — because the human agent does **not** have access to the conversation transcript.

### What you'd actually do (skills)
- **Implement programmatic prerequisites that block downstream tool calls until prerequisites complete.** Concretely: **block `process_refund` (and `lookup_order`) until `get_customer` has returned a verified customer ID.** The gate is code, not a sentence in the prompt.
- **Decompose multi-concern requests into distinct items, investigate each in parallel using shared context, then synthesize a unified resolution.** If a customer writes "my order is late AND I was double-charged AND I want to update my address," split into three items, work them (in parallel where independent), and reply once with all three resolved.
- **Compile structured handoff summaries** when escalating to humans: customer ID, root cause, refund amount, recommended action — a self-contained package, since the human can't see the chat.

```text
# Structured escalation handoff (the human has NO transcript)
Customer ID: 88231 (verified)
Issue: Double charge on order #12345 ($89.99 x2 on 2026-05-30)
Root cause: Duplicate capture due to retry on gateway timeout
Refund amount: $89.99
Recommended action: Refund one charge; waive restocking fee; send apology credit
```

### Anti-patterns / distractors
- "Strengthen the system prompt to say verification is mandatory" — still probabilistic; **wrong** when the cost of failure is financial.
- "Add few-shot examples of always verifying first" — improves the odds but is still non-deterministic; **wrong** as the *guaranteed* fix.
- "Add a routing classifier that enables only certain tools per request" — that controls tool *availability*, not the *ordering* prerequisite that the problem actually requires.
- Escalating to a human with just "customer is upset" and no structured package — the human starts from zero.

### Exam signals & facts to memorize
- **Deterministic requirement → programmatic gate/hook. Always.** (Identity-before-refund is the canonical example.)
- **Prompt/few-shot = probabilistic = insufficient when errors have financial/safety consequences.**
- **Handoff packages must be self-contained** (the human lacks the transcript).
- Multi-concern requests → **decompose → parallel investigate → synthesize one resolution.**

---

## Task Statement 1.5 — Apply Agent SDK hooks for tool call interception and data normalization

### The real-world problem
Two issues in the support agent: (1) your MCP tools return dates in *three different formats* — `lookup_order` returns Unix timestamps, the billing tool returns ISO 8601, the account tool returns numeric status codes — and Claude keeps misreading them. (2) Company policy says **no agent may auto-process a refund over $500** — those must go to a human. You can't trust a prompt to enforce the $500 rule every single time. Hooks solve both.

### What you must know (hook patterns)
- **`PostToolUse` hooks** intercept tool **results** *before the model processes them*, so you can transform/normalize the data the model sees.
- **Tool-call interception hooks** intercept **outgoing** tool calls *before they execute*, so you can enforce compliance rules (block, modify, or redirect).
- **The core distinction (this is the heart of 1.5):** hooks give **deterministic guarantees**; prompt instructions give only **probabilistic compliance.** When a business rule must hold 100% of the time, use a hook.

### What you'd actually do (skills)
- **`PostToolUse` normalization:** write a hook that converts heterogeneous formats (Unix timestamps, ISO 8601, numeric status codes) into one canonical format *before* the agent reasons about them. The model then sees clean, uniform data.
- **Tool-call interception for policy:** write a hook that inspects an outgoing `process_refund` call, and **if the amount exceeds $500, block it and redirect to the human-escalation workflow.** The model never gets to "decide" to break the rule.
- **Choose hooks over prompt-based enforcement whenever the rule requires guaranteed compliance.**

```python
# PostToolUse: normalize dates before the model sees them
def post_tool_use(tool_name, result):
    if "timestamp" in result:
        result["date"] = to_iso8601(result["timestamp"])  # Unix → ISO 8601
    return result

# Tool-call interception: hard-block refunds over $500
def pre_tool_use(tool_name, tool_input):
    if tool_name == "process_refund" and tool_input["amount"] > 500:
        return redirect_to_human_escalation(tool_input)   # deterministic block
    return allow(tool_input)
```

### Anti-patterns / distractors
- Relying on "the prompt says don't refund over $500" — non-zero failure rate; **wrong** for a hard policy.
- Asking the model to normalize date formats itself on every turn — unreliable and wastes reasoning; normalize deterministically in a hook.

### Exam signals & facts to memorize
- **`PostToolUse` = transform incoming tool *results* (normalization).**
- **Tool-call interception = gate outgoing tool *calls* (compliance/policy).**
- **Hooks = deterministic guarantee; prompts = probabilistic.** Pick hooks for hard rules.
- The canonical examples: **normalize timestamps/ISO-8601/status codes** (PostToolUse) and **block refunds over a threshold → redirect to human** (interception).

---

## Task Statement 1.6 — Design task decomposition strategies for complex workflows

### The real-world problem
Two different tasks land on your desk. (1) Review a pull request that touches 14 files — you need consistent, thorough feedback (Scenario 5). (2) "Add comprehensive tests to this 200-file legacy codebase" — you have no idea what's there yet (Scenario 4). These need *opposite* decomposition strategies, and choosing wrong wastes huge effort.

### What you must know (fixed vs adaptive decomposition)
- **Fixed sequential pipelines (prompt chaining):** break the work into a *predetermined* sequence of focused steps. Best when the structure is **predictable**. Example: a multi-aspect code review = "analyze each file individually, then run a cross-file integration pass."
- **Dynamic / adaptive decomposition:** generate subtasks **based on what you discover at each step.** Best for **open-ended** investigation where you can't plan the steps up front.
- The value of adaptive plans is that they **adapt as dependencies are discovered** — you don't have to know the whole map before you start.

### What you'd actually do (skills)
- **Pick the pattern to match the workflow:**
  - **Prompt chaining** → predictable, multi-aspect reviews (the 14-file PR).
  - **Dynamic decomposition** → open-ended investigation (adding tests to an unknown legacy codebase).
- **Split large code reviews** into **per-file local analysis passes** plus a **separate cross-file integration pass**, to avoid *attention dilution* (this is exactly the fix in Sample Question 12 — answer **A**).
- **Decompose open-ended tasks** like "add comprehensive tests to a legacy codebase" by **first mapping structure**, then **identifying high-impact areas**, then **creating a prioritized plan that adapts as dependencies are discovered.** You don't write all the tests in one shot; you build a living plan.

### Anti-patterns / distractors
- Analyzing all 14 files together in a single pass → **attention dilution**: detailed feedback on some files, superficial on others, contradictory findings (flagging a pattern in one file, approving identical code elsewhere). The fix is decomposition, **not** a bigger context window (a larger window does not fix attention quality), **not** forcing developers to split PRs, and **not** majority-vote across three runs (which suppresses real bugs caught intermittently).
- Trying to plan an open-ended legacy-test task fully up front before exploring — you can't; use adaptive decomposition.

### Exam signals & facts to memorize
- **Predictable multi-aspect work → prompt chaining (fixed pipeline).**
- **Open-ended investigation → dynamic/adaptive decomposition.**
- **Big multi-file review → per-file passes + a cross-file integration pass** (cures attention dilution).
- A **larger context window does NOT fix attention dilution** — decomposition does.

---

## Task Statement 1.7 — Manage session state, resumption, and forking

### The real-world problem
You spent an hour with Claude Code mapping a complex refund flow across a codebase (Scenario 4). The next day you want to (a) continue that exact investigation, (b) compare two different refactoring strategies from the same starting analysis, and (c) you've since edited three files and don't want stale analysis. Each of these is a different session-management tool.

### What you must know
- **Named session resumption — `--resume <session-name>`** continues a *specific* prior conversation by name.
- **`fork_session`** creates an **independent branch from a shared analysis baseline** so you can explore *divergent* approaches without them interfering.
- **When resuming after code changes, you must inform the agent which files changed** — otherwise it reasons over stale analysis.
- **Sometimes a fresh session with a structured summary beats resuming.** If the prior tool results are **stale**, starting new and *injecting a clean summary* is more reliable than resuming with outdated context.

### What you'd actually do (skills)
- **Use `--resume <session-name>`** to continue a named investigation across work sessions (pick up yesterday's refund-flow analysis exactly where you left off).
- **Use `fork_session`** to create **parallel exploration branches** from a shared baseline — e.g., compare *two testing strategies* or *two refactoring approaches* that both start from the same codebase analysis, without cross-contamination.
- **Choose between resuming and starting fresh:** resume when prior context is **mostly valid**; start fresh with an injected summary when prior tool results are **stale**.
- **When you do resume after edits, tell the agent exactly which files changed** so it does *targeted* re-analysis instead of re-exploring everything.

```bash
# Continue a specific named investigation
claude --resume refund-flow-investigation

# (conceptually) fork from a shared baseline to compare approaches
# branch A: explore "extract a RefundService"; branch B: explore "inline + guard clauses"
```

### Anti-patterns / distractors
- Resuming a session after modifying files **without telling the agent what changed** → it reasons over stale state and gives inconsistent answers.
- Resuming when the prior tool results are stale, instead of starting fresh with a clean summary.
- Using one linear session to explore two competing approaches (they pollute each other) instead of forking.

### Exam signals & facts to memorize
- **`--resume <session-name>` = continue a specific named conversation.**
- **`fork_session` = branch from a shared baseline to explore divergent approaches in parallel.**
- **Resume when context is mostly valid; start fresh + inject summary when results are stale.**
- **After edits, explicitly inform the resumed session which files changed** for targeted re-analysis.

---

## Domain 1 — putting it together (decision flowchart)

```
Single agent doing a task?
  → Run the agentic loop: continue on stop_reason="tool_use", stop on "end_turn",
    append tool results each iteration. (1.1)

Many agents?
  → Hub-and-spoke: coordinator routes everything; subagents have isolated context. (1.2)
  → Spawn with the Task tool (allowedTools must include "Task");
    pass ALL needed context explicitly in each subagent's prompt;
    parallel = multiple Task calls in ONE response. (1.3)
  → Coverage gap? Fix the coordinator's decomposition + add an iterative
    refinement loop — don't blame working subagents. (1.2)

A step MUST happen before another (identity before refund)?
  → Programmatic prerequisite gate / hook — never a prompt. (1.4, 1.5)

Tool outputs messy or a hard policy threshold?
  → PostToolUse hook to normalize; interception hook to block/redirect. (1.5)

How to break up the work?
  → Predictable multi-aspect → prompt chaining; open-ended → adaptive decomposition;
    big review → per-file + cross-file passes. (1.6)

Continue / branch / refresh a session?
  → --resume <name> to continue; fork_session to branch; fresh+summary if stale;
    tell the agent which files changed. (1.7)
```

### The five Domain-1 truths most likely to earn you points
1. **`stop_reason` drives the loop** — `tool_use` continues, `end_turn` stops; never parse natural language to decide.
2. **Subagents have isolated context** — pass everything explicitly; `Task` must be in `allowedTools`; parallel = many `Task` calls in one turn.
3. **Deterministic requirements need deterministic mechanisms** — prerequisite gates and hooks, not prompts, when money/identity/policy is on the line.
4. **Coverage bugs in multi-agent systems usually trace to narrow coordinator decomposition** — diagnose from the logs; fix decomposition + add refinement loops.
5. **Match decomposition to predictability** — prompt chaining for predictable work, adaptive decomposition for open-ended work, per-file + integration passes for big reviews.
