# Claude Certified Architect – Foundations (CCA-F)
## Complete Preparation Material — Start Here

> **This is your one-stop study pack.** It covers every domain, every task statement, and every "Knowledge of" and "Skills in" bullet from the official exam guide, each explained through a real-world production scenario. It also includes a fresh practice question bank, per-domain cheat sheets, and a glossary/flashcard deck.

---

## 1. What this certification validates

The CCA-F certification validates that you can **make informed decisions about tradeoffs when implementing real-world solutions with Claude.** It is not a trivia test. It is a judgment test. Almost every question gives you a production situation that is already partly broken or partly designed, and asks: *what is the most effective change?*

The exam tests foundational knowledge across the four core technologies used to build production-grade applications with Claude:

| Technology | What it is | Where it shows up |
|---|---|---|
| **Claude Agent SDK** | The framework for building agentic applications (agentic loops, subagents, hooks, sessions) | Domains 1, 2, 5 |
| **Claude Code** | The CLI/agent for software development workflows (CLAUDE.md, skills, slash commands, plan mode) | Domains 2, 3, 5 |
| **Claude API** | The underlying message API (`tool_use`, `tool_choice`, `stop_reason`, Message Batches API) | Domains 1, 4 |
| **Model Context Protocol (MCP)** | The open standard for connecting Claude to backend systems (tools, resources, `isError`, `.mcp.json`) | Domains 2, 3 |

---

## 2. The ideal candidate (are you ready?)

The exam is written for a **solution architect** with roughly **6+ months of hands-on experience**. The guide expects you to have personally:

- Built agentic applications with the Agent SDK — multi-agent orchestration, subagent delegation, tool integration, and lifecycle hooks.
- Configured Claude Code for team workflows — CLAUDE.md files, Agent Skills, MCP server integrations, and plan mode.
- Designed MCP tool and resource interfaces for backend integration.
- Engineered prompts for reliable structured output — JSON schemas, few-shot examples, extraction patterns.
- Managed context windows across long documents, multi-turn conversations, and multi-agent handoffs.
- Integrated Claude into CI/CD pipelines — automated code review, test generation, PR feedback.
- Made escalation and reliability decisions — error handling, human-in-the-loop, self-evaluation.

If any of those bullets feels unfamiliar, the corresponding domain file in this pack is where you should spend the most time.

---

## 3. Exam mechanics (memorize these)

| Item | Detail |
|---|---|
| **Question format** | Multiple choice. **1 correct answer, 3 distractors.** Single best answer. |
| **Distractors** | Designed to look right to someone with *incomplete* knowledge or experience. They are usually real techniques applied to the wrong problem. |
| **Guessing** | Unanswered = incorrect. **There is no penalty for guessing — always answer every question.** |
| **Pass/fail** | Pass or fail designation only. |
| **Scoring** | Scaled score **100–1,000**. **Minimum passing score = 720.** |
| **Why scaled scoring** | It equates scores across multiple exam forms that may differ slightly in difficulty. |
| **Scenarios** | The exam draws **4 scenarios at random from a pool of 6**. Each scenario frames a set of questions. |
| **Guide version** | Version 0.1, last updated Feb 10 2025. |

**Practical takeaway:** Because the pass mark is 720/1000 and there is no guessing penalty, you must answer everything, and you cannot afford to be weak in the two biggest domains (Domain 1 at 27% and Domains 3+4 at 20% each). Together those three domains are **67% of the scored content.**

---

## 4. Domains and weightings

| # | Domain | Weight | Task statements | This pack's file |
|---|---|---|---|---|
| 1 | **Agentic Architecture & Orchestration** | **27%** | 1.1 – 1.7 (7) | `01_Domain1_...md` |
| 2 | **Tool Design & MCP Integration** | **18%** | 2.1 – 2.5 (5) | `02_Domain2_...md` |
| 3 | **Claude Code Configuration & Workflows** | **20%** | 3.1 – 3.6 (6) | `03_Domain3_...md` |
| 4 | **Prompt Engineering & Structured Output** | **20%** | 4.1 – 4.6 (6) | `04_Domain4_...md` |
| 5 | **Context Management & Reliability** | **15%** | 5.1 – 5.6 (6) | `05_Domain5_...md` |

Total: **30 task statements.** Domain 1 is the single largest and most heavily tested — prioritize it.

---

## 5. The 6 exam scenarios

Every scenario file in this pack maps task statements back to these. Learn them cold — recognizing *which* scenario a question belongs to tells you which mental model to apply.

**Scenario 1 — Customer Support Resolution Agent.** An Agent SDK agent handling high-ambiguity requests (returns, billing disputes, account issues) via custom MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`). Target: **80%+ first-contact resolution** while knowing when to escalate.
*Primary domains: 1, 2, 5.*

**Scenario 2 — Code Generation with Claude Code.** A team uses Claude Code for generation, refactoring, debugging, and documentation, integrated via custom slash commands and CLAUDE.md, deciding between plan mode and direct execution.
*Primary domains: 3, 5.*

**Scenario 3 — Multi-Agent Research System.** A coordinator delegates to specialized subagents (web search, document analysis, synthesis, report generation) to produce comprehensive, cited reports.
*Primary domains: 1, 2, 5.*

**Scenario 4 — Developer Productivity with Claude.** An Agent SDK agent helps engineers explore unfamiliar codebases, understand legacy systems, generate boilerplate, and automate tasks, using built-in tools (Read, Write, Bash, Grep, Glob) plus MCP servers.
*Primary domains: 2, 3, 1.*

**Scenario 5 — Claude Code for Continuous Integration.** Claude Code runs in a CI/CD pipeline doing automated code review, test generation, and PR feedback, with prompts designed for actionable feedback and minimal false positives.
*Primary domains: 3, 4.*

**Scenario 6 — Structured Data Extraction.** A system extracts information from unstructured documents, validates output against JSON schemas, maintains high accuracy, handles edge cases, and integrates with downstream systems.
*Primary domains: 4, 5.*

---

## 6. The single most important exam pattern

Almost every correct answer on this exam follows one meta-rule:

> **Match the mechanism to the guarantee you need, and choose the most proportionate fix.**

This shows up again and again:

- **Need a deterministic guarantee** (identity before refund, tool A before tool B)? → Use a **programmatic mechanism** (hook, prerequisite gate, forced `tool_choice`), *not* a prompt instruction. Prompts have a non-zero failure rate.
- **Need probabilistic improvement** (better tool selection, fewer false positives, consistent format)? → Use **prompt-level fixes first** (better tool descriptions, explicit criteria, few-shot examples) *before* building ML classifiers, routers, or infrastructure.
- **Need to diagnose a failure**? → Read the logs/evidence in the scenario. The root cause is usually stated directly (e.g., the coordinator's decomposition was too narrow). Don't blame downstream agents that are working correctly.
- **Distractors are usually over-engineered or mis-targeted**: a routing classifier, a fine-tuned model, a bigger context window, or sentiment analysis applied to a problem that a small prompt/architecture change solves.

If you remember nothing else: **deterministic problems get deterministic mechanisms; quality problems get the cheapest effective prompt/architecture fix; and you diagnose before you redesign.**

---

## 7. How to use this pack (recommended study order)

1. **Read this overview** until the mechanics and scenarios are second nature.
2. **Work through the 5 domain files in order** (`01`–`05`). Each task statement is taught through a scenario you must "solve," then summarized with the exact facts to memorize and the anti-patterns to avoid.
3. **Drill the practice question bank** (`06`). Do a domain's questions right after reading that domain. Re-read any explanation you got wrong.
4. **Compress with the cheat sheets** (`07`) — one page per domain for the final week.
5. **Self-test with the glossary/flashcards** (`08`) — cover the answer, recall it, flip.
6. **Take the official practice exam** (link provided separately by Anthropic) last, as a dress rehearsal.

### Suggested 2-week plan

| Day(s) | Focus |
|---|---|
| 1 | This overview + Domain 1 (read) |
| 2 | Domain 1 practice questions + re-read weak spots |
| 3 | Domain 2 (read + questions) |
| 4 | Domain 3 (read + questions) |
| 5 | Domain 4 (read + questions) |
| 6 | Domain 5 (read + questions) |
| 7 | Full practice question bank, mixed order |
| 8 | Cheat sheets + glossary/flashcards (active recall) |
| 9–10 | Hands-on: build a mini agent, configure Claude Code, build an extraction pipeline (see exercises below) |
| 11 | Re-drill every question you previously missed |
| 12 | Official practice exam |
| 13 | Review official practice exam misses against the relevant domain files |
| 14 | Light flashcard review + rest before the exam |

---

## 8. Hands-on exercises (do these — the exam rewards real experience)

The guide ships four exercises. Each reinforces multiple domains. Do all four if you have time; do **Exercise 1 and Exercise 4** at minimum, since Domain 1 is the heaviest.

**Exercise 1 — Build a multi-tool agent with escalation logic.** Define 3–4 MCP tools (include two deliberately similar ones), implement an agentic loop driven by `stop_reason`, add structured error responses (`errorCategory`, `isRetryable`, human-readable description), add a hook that blocks operations over a threshold and redirects to escalation, and test multi-concern messages. *Reinforces D1, D2, D5.*

**Exercise 2 — Configure Claude Code for a team.** Project-level CLAUDE.md, `.claude/rules/` with glob `paths:`, a `context: fork` skill with `allowed-tools`, an MCP server in `.mcp.json` with env-var expansion plus a personal one in `~/.claude.json`, and plan-mode-vs-direct-execution tests across tasks of varying complexity. *Reinforces D3, D2.*

**Exercise 3 — Build a structured-extraction pipeline.** Extraction tool with required/optional/nullable fields and an `enum` with `"other"` + detail, a validation-retry loop, few-shot examples for varied document formats, a 100-document batch via the Message Batches API with `custom_id` failure handling, and a confidence-based human-review routing strategy. *Reinforces D4, D5.*

**Exercise 4 — Design and debug a multi-agent research pipeline.** Coordinator with `allowedTools` including `Task`, explicit context passing into each subagent, parallel subagent execution (multiple `Task` calls in one response), structured subagent output (claim + evidence + source + date), simulated subagent timeout with structured error propagation, and conflicting-source handling with preserved attribution. *Reinforces D1, D2, D5.*

---

## 9. In-scope vs out-of-scope (don't waste time on the wrong things)

**In scope (study hard):** agentic loop implementation, multi-agent orchestration, subagent context management, tool interface design, MCP tool/resource design, MCP server configuration, error handling/propagation, escalation decision-making, CLAUDE.md configuration, custom commands/skills, plan vs direct execution, iterative refinement, structured output via `tool_use`, few-shot prompting, batch processing, context-window optimization, human-review workflows, information provenance.

**Out of scope (do NOT study — and be suspicious of answer options that go here):**
- Fine-tuning or training custom models; Constitutional AI / RLHF / safety training.
- API authentication, billing, account management, API key rotation, OAuth.
- Deep language/framework implementation beyond tool/schema config.
- Deploying/hosting MCP servers (infra, networking, containers); specific cloud configs (AWS/GCP/Azure).
- Claude's internal architecture, training process, model weights.
- Embedding models / vector DB implementation details.
- Computer use (browser/desktop automation); vision/image analysis.
- Streaming / server-sent events.
- Rate limits, quotas, pricing calculations; token-counting algorithms / tokenization specifics.
- Prompt caching implementation details (beyond knowing it exists).

> **Distractor tell:** if an answer option requires fine-tuning a model, building an ML classifier, standing up infrastructure, or doing sentiment analysis, it is *very often* the wrong answer — those are over-engineered or out-of-scope relative to the cheaper prompt/architecture fix the exam wants.

---

## 10. Files in this pack

```
CCA-F_Exam_Prep/
├── 00_START_HERE_Exam_Overview.md          ← you are here
├── 01_Domain1_Agentic_Architecture_Orchestration.md   (27%)
├── 02_Domain2_Tool_Design_MCP_Integration.md          (18%)
├── 03_Domain3_Claude_Code_Configuration_Workflows.md  (20%)
├── 04_Domain4_Prompt_Engineering_Structured_Output.md (20%)
├── 05_Domain5_Context_Management_Reliability.md        (15%)
├── 06_Practice_Question_Bank.md            ← 45 new MCQs + the 12 official samples explained
├── 07_Cheat_Sheets.md                      ← one page per domain
└── 08_Glossary_and_Flashcards.md           ← every key term, flashcard format
```

Good luck. Work the scenarios, not just the facts — that is exactly how the exam is written.
