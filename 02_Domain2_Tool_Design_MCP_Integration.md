# Domain 2 — Tool Design & MCP Integration
### Weight: 18% · Task statements 2.1 – 2.5

> **What this domain is about.** Tools are how Claude *acts on the world.* Domain 2 is about making those tools selectable (good descriptions), debuggable (structured errors), correctly scoped (the right tools for each agent), correctly wired in (MCP server configuration), and about knowing the built-in tools cold (Read/Write/Edit/Bash/Grep/Glob). Scenarios 1 (Customer Support), 3 (Research), and 4 (Developer Productivity) drive most questions.

**Mental model:** the model selects a tool almost entirely from its **description**. Treat every tool description as a tiny spec sheet the model reads to decide "is this the right tool for what the user wants?" Most Domain-2 problems are solved by improving descriptions, returning richer errors, or removing tools — not by adding routers or classifiers.

---

## Task Statement 2.1 — Design effective tool interfaces with clear descriptions and boundaries

### The real-world problem
In the support agent (Scenario 1), production logs show that when users ask about orders (*"check my order #12345"*), the agent frequently calls **`get_customer`** instead of **`lookup_order`**. Both tools have minimal one-line descriptions ("Retrieves customer information" / "Retrieves order details") and accept similar identifier formats. What's the most effective *first* step? (Sample Question 2 — answer **B**.)

### What you must know
- **Tool descriptions are the primary mechanism LLMs use for tool selection.** Minimal descriptions → unreliable selection among similar tools. This is the single most important fact in 2.1.
- A good description includes **input formats, example queries, edge cases, and boundary explanations** ("use this when…, not when…").
- **Ambiguous or overlapping descriptions cause misrouting.** Two tools like `analyze_content` and `analyze_document` with near-identical descriptions are a recipe for the model picking the wrong one.
- **System prompt wording affects tool selection.** Keyword-sensitive instructions in the system prompt can create *unintended* tool associations that override even well-written tool descriptions.

### What you'd actually do (skills)
- **Write descriptions that clearly differentiate** each tool's purpose, expected inputs, outputs, and *when to use it versus similar alternatives.* For the Scenario-1 bug, expanding `lookup_order`'s description to say "use this for any query containing an order number like #12345" fixes the misrouting at the root.
- **Rename tools and rewrite descriptions to eliminate overlap.** Example from the guide: rename `analyze_content` → **`extract_web_results`** with a web-specific description, so it no longer competes with `analyze_document`.
- **Split a generic tool into purpose-specific tools** with defined input/output contracts. Example: split a vague `analyze_document` into **`extract_data_points`**, **`summarize_content`**, and **`verify_claim_against_source`** — each unambiguous.
- **Review the system prompt for keyword-sensitive instructions** that might hijack selection (e.g., a system prompt that says "always analyze content carefully" can bias the model toward a tool literally named `analyze_content`).

### Why "improve the descriptions" beats the distractors (Sample Q2 logic)
- **Few-shot examples (distractor A):** add token overhead and don't fix the *root cause* (the descriptions are still inadequate). Good for ambiguity *after* descriptions are solid, not as the first step.
- **A keyword routing layer (distractor C):** over-engineered; it bypasses the LLM's natural-language understanding and is brittle.
- **Consolidating into one `lookup_entity` tool (distractor D):** a *valid architectural option*, but heavier than a "first step" warrants when the real problem is just thin descriptions.

> **Rule:** description quality is a *low-effort, high-leverage* fix. Reach for it first.

### Exam signals & facts to memorize
- **Descriptions = primary selection mechanism.** Thin descriptions = the usual root cause of misrouting.
- Good descriptions include **inputs, example queries, edge cases, boundaries (when-to-use-vs-not).**
- Fixes: **rewrite/rename to remove overlap**, or **split generic tools** into specific ones.
- **Check the system prompt** for keywords that distort selection.

---

## Task Statement 2.2 — Implement structured error responses for MCP tools

### The real-world problem
Your MCP tools sometimes fail. `lookup_order` times out (transient). `process_refund` is asked to refund $800 (policy violation). A search returns no matching orders (valid empty result). If every failure comes back as a generic `"Operation failed"`, the agent can't tell "retry this" from "explain to the customer this isn't allowed" from "that's just an empty result." You need errors that tell the agent *what kind* of failure happened.

### What you must know
- **The MCP `isError` flag** is the pattern for communicating a tool failure back to the agent.
- **Four error categories** you must distinguish:
  - **Transient** — timeouts, service unavailable. *Retryable.*
  - **Validation** — invalid input. *Usually not retryable without changing the input.*
  - **Business** — policy violations (refund too large). *Not retryable; explain to the user.*
  - **Permission** — not authorized. *Not retryable via retry.*
- **Uniform errors ("Operation failed") prevent appropriate recovery** — the agent can't decide what to do next.
- **Retryable vs non-retryable matters:** returning structured metadata prevents the agent from **wasting retries** on errors that will never succeed.

### What you'd actually do (skills)
- **Return structured error metadata:** `errorCategory` (transient / validation / permission), an `isRetryable` boolean, and a **human-readable description.**
- **For business-rule violations, include `retriable: false` plus a customer-friendly explanation** so the agent can communicate appropriately ("Refunds over $500 require a manager — I've escalated this for you").
- **Implement local recovery inside subagents for transient failures;** only propagate to the coordinator the errors that **can't be resolved locally** — and when you do, include **partial results and what was attempted.** (This connects to Domain 5.3.)
- **Distinguish access failures from valid empty results.** "Search timed out" (needs a retry decision) is *not* the same as "search succeeded, zero matches" (a legitimate answer). Conflating them causes pointless retries or hides real failures.

```json
// Transient (retry is worthwhile)
{ "isError": true, "errorCategory": "transient", "isRetryable": true,
  "message": "Order service timed out after 5s." }

// Business rule (do NOT retry; explain to the customer)
{ "isError": true, "errorCategory": "business", "isRetryable": false,
  "message": "Refund of $800 exceeds the $500 auto-approval limit; manager approval required." }

// Valid empty result (NOT an error)
{ "isError": false, "orders": [] }
```

### Anti-patterns / distractors
- Generic `"Operation failed"` for everything → the agent can't choose a recovery path.
- Marking a business violation as retryable → the agent burns retries on something that will never succeed.
- Returning an empty result as an *error* (or an error as a *success*) → wrong downstream behavior.

### Exam signals & facts to memorize
- **`isError` flag** signals failure; pair it with **`errorCategory` + `isRetryable` + human-readable message.**
- **Transient = retry; validation/business/permission = don't blind-retry.**
- **Business errors → `retriable: false` + customer-friendly text.**
- **Empty result ≠ error.** Access failure ≠ empty result.
- **Recover locally first; propagate only unresolved errors with partial results + what was attempted.**

---

## Task Statement 2.3 — Distribute tools appropriately across agents and configure tool choice

### The real-world problem
Two situations. (1) Your synthesis agent in the research system has been given access to *all 18 tools* in the platform, and it keeps trying to do web searches itself instead of synthesizing — and even when it should synthesize, it picks the wrong tool. (2) In the extraction pipeline you need to *guarantee* `extract_metadata` runs before any enrichment tool. Both are tool-distribution / tool-choice problems.

### What you must know
- **Too many tools degrades selection reliability.** Giving an agent **18 tools instead of 4–5** increases decision complexity and makes the model pick worse. *Fewer, well-scoped tools = more reliable selection.*
- **Agents misuse tools outside their specialization** (a *synthesis* agent attempting *web searches*). Keep tools aligned to role.
- **Scoped tool access:** give each agent only the tools its role needs, plus a *limited* set of cross-role tools for specific high-frequency needs.
- **`tool_choice` options:**
  - **`"auto"`** — the model may call a tool *or* return text.
  - **`"any"`** — the model **must** call *some* tool (no plain text), but chooses which.
  - **Forced** — `{"type": "tool", "name": "..."}` forces a **specific** named tool.

### What you'd actually do (skills)
- **Restrict each subagent's tool set to its role** to prevent cross-specialization misuse (synthesis agent gets synthesis tools, not the web-search suite).
- **Replace generic tools with constrained alternatives.** Example: replace `fetch_url` (fetches anything) with **`load_document`** (validates that the URL is a document) so the agent can't wander off.
- **Provide scoped cross-role tools for high-frequency needs.** This is exactly Sample Question 9: the synthesis agent does simple fact-checks 85% of the time, so give it a **scoped `verify_fact` tool** for those, while *complex* verifications still route through the coordinator to the web-search agent. (Answer **A** — least privilege: just enough for the common case, escalate the rest. *Not* C, which over-provisions the synthesis agent with all web tools.)
- **Use forced `tool_choice` to guarantee ordering:** force `extract_metadata` first with `{"type": "tool", "name": "extract_metadata"}`, then handle enrichment in follow-up turns.
- **Use `tool_choice: "any"` to guarantee the model calls *a* tool** rather than returning conversational text (useful when you must get structured output, see Domain 4.3).

### Anti-patterns / distractors
- Dumping all tools on every agent "for flexibility" → worse selection, cross-role misuse.
- Giving the synthesis agent the *entire* web-search toolset to avoid round-trips (Sample Q9 distractor C) → violates separation of concerns and over-provisions for a 15% case.
- Speculative caching of "everything it might need" (Sample Q9 distractor D) → can't reliably predict needs.

### Exam signals & facts to memorize
- **~4–5 tools per agent, scoped to role.** 18 tools is the cautionary number.
- **Least privilege + scoped cross-role tool for the common case;** route rare/complex cases through the coordinator.
- **`tool_choice`: `"auto"` (text or tool) | `"any"` (must call some tool) | forced `{"type":"tool","name":"..."}` (specific tool).**
- **Forced tool_choice runs a specific tool first; follow-up turns handle the rest.**

---

## Task Statement 2.4 — Integrate MCP servers into Claude Code and agent workflows

### The real-world problem
Your team (Scenario 4 / developer productivity) wants a shared MCP server for your internal issue tracker available to everyone who clones the repo, using a GitHub token *without committing the secret.* You personally also want to experiment with a half-finished MCP server that teammates shouldn't get. And you notice the agent keeps using built-in `Grep` instead of your far more capable MCP search tool. These are MCP configuration and adoption problems.

### What you must know (scoping)
- **MCP server scope:**
  - **Project-level — `.mcp.json`** (committed to the repo) → shared team tooling. Everyone who clones gets it.
  - **User-level — `~/.claude.json`** → personal / experimental servers, *not* shared via version control.
- **Environment-variable expansion in `.mcp.json`** (e.g., `${GITHUB_TOKEN}`) lets you reference credentials *without committing secrets.*
- **All configured MCP servers are discovered at connection time and available simultaneously** — tools from every connected server are in play at once.
- **MCP resources expose content catalogs** (issue summaries, documentation hierarchies, database schemas) to **reduce exploratory tool calls** — the agent can *see* what's available instead of probing for it.

### What you'd actually do (skills)
- **Configure shared servers in project-scoped `.mcp.json`** with env-var expansion for auth tokens (`"token": "${GITHUB_TOKEN}"`).
- **Configure personal/experimental servers in user-scoped `~/.claude.json`** so they don't affect teammates.
- **Enhance MCP tool descriptions** to explain capabilities and outputs in detail — otherwise the agent prefers built-in tools (like `Grep`) over your more capable MCP tool. (Adoption is a *description* problem again — cf. 2.1.)
- **Prefer existing community MCP servers for standard integrations** (e.g., Jira) over building your own; **reserve custom servers for team-specific workflows.**
- **Expose content catalogs as MCP resources** so agents have visibility into available data without exploratory calls.

```json
// .mcp.json (project scope, committed) — secret stays in the environment
{
  "mcpServers": {
    "issue-tracker": {
      "command": "npx",
      "args": ["-y", "@acme/issue-tracker-mcp"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

### Anti-patterns / distractors
- Hard-coding the token into `.mcp.json` and committing it → leaked secret. Use `${VAR}` expansion.
- Putting a shared team server in `~/.claude.json` → teammates don't get it (mirror of the Domain-3 hierarchy bug).
- Building a custom Jira MCP server when a solid community one exists → wasted effort.
- Leaving MCP tool descriptions thin → the agent ignores them in favor of built-ins.

### Exam signals & facts to memorize
- **`.mcp.json` = project/shared; `~/.claude.json` = user/personal.**
- **`${ENV_VAR}` expansion keeps secrets out of version control.**
- **All connected servers' tools are available simultaneously.**
- **MCP resources = content catalogs that cut exploratory tool calls.**
- **Community server for standard integrations; custom only for team-specific needs.**
- **Rich MCP tool descriptions** prevent the agent from defaulting to built-ins.

---

## Task Statement 2.5 — Select and apply built-in tools (Read, Write, Edit, Bash, Grep, Glob) effectively

### The real-world problem
In the developer-productivity agent (Scenario 4), you must explore an unfamiliar 200-file codebase: find every caller of `processRefund`, follow the imports, and then make a one-line change. If you `Read` all 200 files up front you blow the context window; if you reach for the wrong built-in you waste turns. Knowing exactly what each built-in is for is the skill.

### What you must know (the built-ins)
- **`Grep`** — **content search.** Search *inside* files for patterns: function names, error messages, import statements. ("Find all callers of `processRefund`.")
- **`Glob`** — **file-path pattern matching.** Find files by name/extension patterns. (`**/*.test.tsx` → all test files.)
- **`Read` / `Write`** — **full-file operations.** Read loads an entire file; Write replaces/creates a whole file.
- **`Edit`** — **targeted modification** using **unique text matching.** It finds a unique anchor string and replaces it.
- **Edit's failure mode + fallback:** when the anchor text is **not unique**, `Edit` fails. The reliable fallback is **`Read` the full file, then `Write` it back** with the change.
- (`Bash` runs shell commands; use it for things the file tools don't cover.)

### What you'd actually do (skills)
- **Use `Grep` to search code content** — find all callers of a function, locate an error message string.
- **Use `Glob` to find files by naming pattern** — e.g., `**/*.test.tsx`.
- **When `Edit` can't find a unique anchor, fall back to `Read` + `Write`** for a reliable modification.
- **Build understanding incrementally, not all at once:** start with **`Grep` to find entry points**, then **`Read` to follow imports and trace flows** — do *not* read every file up front (that wastes context and dilutes attention).
- **Trace function usage across wrapper modules** by first **identifying all exported names**, then **searching for each name** across the codebase (a wrapper may re-export under a different name).

### Decision table (memorize)
| You want to… | Use |
|---|---|
| Find *which files* match a name/extension pattern | **Glob** |
| Find *where in the code* a string/pattern appears | **Grep** |
| Load or completely replace a file | **Read / Write** |
| Make a small, surgical change via a unique anchor | **Edit** |
| Make a change but the anchor text isn't unique | **Read + Write** (Edit fallback) |
| Run a command / script | **Bash** |

### Anti-patterns / distractors
- `Read`-ing the whole codebase up front → context exhaustion + attention dilution. Explore incrementally with Grep→Read.
- Using `Glob` to search file *contents* (it matches *paths*), or `Grep` to enumerate files by name (it searches *content*).
- Retrying `Edit` repeatedly on non-unique text instead of switching to Read + Write.

### Exam signals & facts to memorize
- **Grep = content; Glob = file paths.** (Don't mix them up — this is a favorite distractor.)
- **Edit needs a *unique* anchor; non-unique → fall back to Read + Write.**
- **Explore incrementally: Grep for entry points → Read to follow flows.** Never read everything up front.

---

## Domain 2 — putting it together (decision flowchart)

```
Agent picks the wrong tool?
  → First fix the DESCRIPTIONS (inputs, examples, edge cases, when-to-use). (2.1)
  → Still overlapping? Rename/split into purpose-specific tools. (2.1)
  → Check the system prompt for biasing keywords. (2.1)

Tool failed?
  → Return isError + errorCategory + isRetryable + human message. (2.2)
  → Transient → retry; business → retriable:false + customer text.
  → Empty result is NOT an error. (2.2)

Agent misusing / over-using tools?
  → Scope each agent to ~4-5 role tools; replace generic tools with constrained ones;
    add a scoped cross-role tool for the high-frequency case only. (2.3)

Need to guarantee a tool is called / called first?
  → tool_choice "any" (must call something) or forced {"type":"tool","name":...}. (2.3)

Wiring in MCP?
  → Shared → .mcp.json (commit) with ${ENV} secrets; personal → ~/.claude.json. (2.4)
  → All servers available at once; expose resources as catalogs; rich descriptions
    so the agent doesn't default to built-ins. (2.4)

Exploring a codebase?
  → Grep (content) to find entry points → Read to follow imports;
    Glob (paths) for file patterns; Edit needs unique anchor else Read+Write. (2.5)
```

### The five Domain-2 truths most likely to earn you points
1. **Tool descriptions are the primary selection mechanism** — most misrouting is fixed by better descriptions, then by renaming/splitting tools.
2. **Structured errors enable recovery** — `isError` + `errorCategory` + `isRetryable`; never a generic "Operation failed"; empty ≠ error.
3. **Fewer, scoped tools select better** — ~4–5 per agent; least privilege; scoped cross-role tool for the common case only.
4. **`tool_choice`**: `"auto"` (maybe), `"any"` (must call some tool), forced (specific tool first).
5. **`.mcp.json` = shared/project, `~/.claude.json` = personal; `${ENV}` for secrets; Grep=content, Glob=paths, Edit needs a unique anchor.**
