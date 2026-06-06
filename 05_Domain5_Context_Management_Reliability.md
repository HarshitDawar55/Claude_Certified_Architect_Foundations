# Domain 5 — Context Management & Reliability
### Weight: 15% · Task statements 5.1 – 5.6

> **What this domain is about.** The context window is finite and lossy; multi-agent systems fail in partial ways; humans must be looped in correctly; and provenance must survive synthesis. Domain 5 is the "keep the system trustworthy as it scales in length and complexity" domain. It cuts across *every* scenario, but especially Scenario 1 (support), Scenario 3 (research), and Scenario 6 (extraction). Though it's the smallest weight (15%), its ideas reappear inside Domain 1 and Domain 4 questions.

**Mental model:** treat critical facts, errors, provenance, and confidence as **first-class data you deliberately preserve** — not as things you hope survive summarization. When in doubt, *extract structured facts into a persistent layer, propagate structured errors, and route uncertainty to humans.*

---

## Task Statement 5.1 — Manage conversation context to preserve critical information across long interactions

### The real-world problem
A billing-dispute conversation (Scenario 1) runs 40 turns. Early on the customer said "I was charged **$89.99 twice on May 30**, order **#12345**." Twenty turns later, after progressive summarization, the agent "remembers" only "customer has a billing concern" — the exact amount, date, and order number are gone, and it quotes the wrong refund. Meanwhile each `lookup_order` dumps 40+ fields into context though only 5 matter. The window is filling with noise and losing signal.

### What you must know
- **Progressive summarization risks:** condensing **numerical values, percentages, dates, and customer-stated expectations** into vague summaries loses exactly the facts you can't afford to lose.
- **The "lost in the middle" effect:** models reliably use info at the **beginning and end** of long inputs but may **omit findings from the middle.**
- **Tool results accumulate and consume tokens disproportionately to relevance** (40+ fields per order lookup when only 5 are relevant).
- **Pass complete conversation history in subsequent API requests** to maintain conversational coherence (the API is stateless; you resend the transcript).

### What you'd actually do (skills)
- **Extract transactional facts into a persistent "case facts" block** included in **every** prompt, *outside* the summarized history — amounts, dates, order numbers, statuses. The $89.99/May-30/#12345 facts live here and never get summarized away.
- **Extract and persist structured issue data** (order IDs, amounts, statuses) into a **separate context layer** for multi-issue sessions.
- **Trim verbose tool outputs to only relevant fields *before* they accumulate** (keep only the return-relevant fields from an order lookup).
- **Mitigate position effects:** put **key-findings summaries at the beginning** of aggregated inputs and organize details under **explicit section headers.**
- **Require subagents to include metadata** (dates, source locations, methodological context) in structured outputs to support accurate downstream synthesis.
- **Have upstream agents return structured data** (key facts, citations, relevance scores) **instead of verbose content/reasoning chains** when downstream agents have limited context budgets.

```text
# Persistent "case facts" block — included in EVERY prompt, never summarized
CASE FACTS (verbatim, do not summarize):
- Customer ID: 88231 (verified)
- Order: #12345
- Disputed charge: $89.99 charged twice on 2026-05-30
- Status: refund pending manager approval
--- (summarized conversation history follows below) ---
```

### Anti-patterns / distractors
- Summarizing away exact figures/dates and then quoting them wrong.
- Letting raw 40-field tool dumps pile up in context (trim first).
- Burying the most important finding in the middle of a long input.

### Exam signals & facts to memorize
- **Extract exact facts (amounts/dates/IDs) into a persistent "case facts" block, outside summaries.**
- **"Lost in the middle" → put key findings at the start/end; use section headers.**
- **Trim verbose tool outputs to relevant fields before they accumulate.**
- **Send full history each request; have upstream agents emit structured facts, not verbose prose.**

---

## Task Statement 5.2 — Design effective escalation and ambiguity resolution patterns

### The real-world problem
The support agent (Scenario 1) escalates *straightforward* cases (standard damage replacements with photo evidence) while *trying to handle* complex policy-exception cases itself — first-contact resolution sits at 55% vs the 80% target (Sample Question 3). It also picks one account when `get_customer` returns two matches, sometimes the wrong one. When should it escalate, when should it resolve, and how should it handle ambiguity?

### What you must know
- **Appropriate escalation triggers:** (1) the **customer explicitly requests a human**, (2) **policy exceptions/gaps** (not merely "complex" cases), and (3) **inability to make meaningful progress.**
- **Explicit demand vs offer to resolve:** escalate **immediately** when a customer explicitly demands a human; **offer to resolve** when the issue is straightforward and within capability.
- **Sentiment and self-reported confidence are unreliable proxies for case complexity.** (The agent is already over-confident on hard cases — so confidence scores don't help; sentiment doesn't correlate with complexity.)
- **Multiple customer matches require clarification** (ask for more identifiers), **not heuristic selection.**

### What you'd actually do (skills)
- **Add explicit escalation criteria with few-shot examples** to the system prompt demonstrating when to escalate vs resolve autonomously. (Sample Q3 answer **A** — fixes the unclear decision boundary at the root.)
- **Honor explicit requests for a human immediately**, without first attempting investigation.
- **Acknowledge frustration while offering resolution** when the issue is within capability; escalate **only if the customer reiterates** their preference.
- **Escalate when policy is ambiguous or silent** on the customer's specific request (e.g., competitor price-matching when policy only addresses own-site adjustments).
- **Ask for additional identifiers when tool results return multiple matches**, instead of guessing.

### Why Sample Q3 answer is A (and the rest are wrong)
- **A** — explicit escalation criteria + few-shot examples fix the unclear decision boundary. ✔
- **B** (self-reported confidence threshold) — LLM self-confidence is poorly calibrated; the agent is *already* wrongly confident on hard cases.
- **C** (train a separate classifier) — over-engineered; needs labeled data/ML infra before prompt optimization is even tried.
- **D** (sentiment-based escalation) — sentiment doesn't correlate with complexity; solves the wrong problem.

### Anti-patterns / distractors
- Escalating easy cases and attempting hard policy-exception cases (backwards calibration).
- Using **sentiment** or **self-reported confidence** as the escalation trigger.
- **Heuristically picking** one of multiple customer matches instead of asking for another identifier.
- Investigating first when the customer *explicitly* demanded a human.

### Exam signals & facts to memorize
- **Escalate on: explicit human request, policy gap/exception, no meaningful progress.**
- **Explicit demand → escalate now; straightforward → offer to resolve (escalate only if reiterated).**
- **Sentiment & self-confidence are NOT reliable complexity proxies.**
- **Multiple matches → ask for more identifiers, never guess.**

---

## Task Statement 5.3 — Implement error propagation strategies across multi-agent systems

### The real-world problem
In the research system (Scenario 3), the web-search subagent **times out** mid-task. How that failure reaches the coordinator determines whether the system recovers gracefully or collapses (Sample Question 8). Do you return a generic "search unavailable," silently return empty results, crash the whole workflow, or hand the coordinator something it can actually act on?

### What you must know
- **Structured error context enables intelligent coordinator recovery:** failure type, attempted query, **partial results**, and alternative approaches.
- **Access failures vs valid empty results:** a **timeout** (needs a retry decision) is *not* the same as a **successful search with zero matches.** Conflating them is a bug.
- **Generic error statuses hide valuable context** ("search unavailable" tells the coordinator nothing actionable).
- **Two anti-patterns:** **silently suppressing errors** (returning empty as success) and **terminating the whole workflow on a single failure.** Both are wrong.

### What you'd actually do (skills)
- **Return structured error context** including failure type, what was attempted, partial results, and potential alternatives — so the coordinator can decide (retry with a modified query, try another approach, or proceed with partial results). (Sample Q8 answer **A**.)
- **Distinguish access failures from valid empty results** in your error reporting.
- **Have subagents do local recovery for transient failures**, propagating only what they can't resolve — including **what was attempted and partial results.**
- **Structure synthesis output with coverage annotations** indicating which findings are well-supported vs which topic areas have **gaps due to unavailable sources.**

```json
// Structured error the coordinator can ACT on
{ "status": "error", "failureType": "timeout",
  "attemptedQuery": "AI in film production 2024",
  "partialResults": [ { "claim": "...", "source": "..." } ],
  "alternatives": ["retry with narrower date range", "use cached corpus"] }
```

### Why Sample Q8 answer is A (and the rest are wrong)
- **A** — structured error context (type, query, partial results, alternatives) → informed recovery. ✔
- **B** (retry with backoff, then generic "search unavailable") — the generic status hides context the coordinator needs.
- **C** (catch timeout, return empty marked successful) — suppresses the error; prevents recovery; risks incomplete output.
- **D** (propagate the exception to a top-level handler that kills the workflow) — terminates unnecessarily when recovery could succeed.

### Exam signals & facts to memorize
- **Propagate structured error context (type + attempted query + partial results + alternatives).**
- **Access failure ≠ empty result.**
- **Never: generic status, silent suppression (empty-as-success), or whole-workflow termination on one failure.**
- **Recover locally first; annotate synthesis with coverage gaps.**

---

## Task Statement 5.4 — Manage context effectively in large codebase exploration

### The real-world problem
The developer-productivity agent (Scenario 4) explores a huge legacy codebase over a long session. After a while it starts giving **inconsistent answers** and referencing "typical patterns" instead of the **specific classes it discovered earlier** — classic context degradation. And if it crashes mid-exploration, you don't want to restart from zero.

### What you must know
- **Context degradation in extended sessions:** the model starts giving inconsistent answers and referencing *generic* "typical patterns" rather than the **specific** classes/flows it found earlier.
- **Scratchpad files persist key findings across context boundaries.**
- **Subagent delegation isolates verbose exploration output** while the main agent coordinates high-level understanding.
- **Structured state persistence for crash recovery:** each agent **exports state to a known location**, and the coordinator **loads a manifest on resume.**

### What you'd actually do (skills)
- **Spawn subagents to investigate specific questions** ("find all test files," "trace refund-flow dependencies") while the main agent **preserves high-level coordination** (and a clean context).
- **Maintain scratchpad files recording key findings**, referencing them for later questions to **counteract context degradation.**
- **Summarize key findings from one phase before spawning subagents for the next**, injecting the summaries into the next phase's initial context.
- **Design crash recovery using structured agent state exports (manifests)** the coordinator loads on resume and injects into agent prompts.
- **Use `/compact`** to reduce context usage during extended exploration when context fills with verbose discovery output.

```text
# Scratchpad persisted across context boundaries
## findings/refund-flow.md
- RefundService.process() lives in src/billing/refund_service.py
- Calls PaymentGateway.refund() (src/payments/gateway.py:88)
- Idempotency key = order_id + "_refund"; duplicate guard at line 142
```

### Anti-patterns / distractors
- Pushing one giant session until it degrades, instead of scratchpads + subagent delegation + `/compact`.
- Letting verbose discovery output flood the main context (delegate it to a subagent / Explore — cf. 3.4).
- No state export → a crash means re-exploring everything.

### Exam signals & facts to memorize
- **Degradation tell: "typical patterns" instead of the specific classes found earlier.**
- **Scratchpad files** persist findings; **subagents** isolate verbose output; **`/compact`** reduces context use.
- **Crash recovery = structured state exports (manifests) the coordinator loads on resume.**

---

## Task Statement 5.5 — Design human review workflows and confidence calibration

### The real-world problem
Your extraction system (Scenario 6) reports **97% overall accuracy**, so leadership wants to drop human review. But you suspect the 97% hides poor performance on one document type (say, handwritten invoices) or one field (tax IDs). You have limited reviewer capacity. How do you decide what to automate and what to route to humans?

### What you must know
- **Aggregate accuracy can mask poor performance on specific document types or fields** (97% overall can hide 70% on one segment).
- **Stratified random sampling** measures error rates in **high-confidence** extractions and detects **novel error patterns.**
- **Field-level confidence scores, calibrated with labeled validation sets**, route review attention.
- **Validate accuracy by document type and field *before* automating** high-confidence extractions.

### What you'd actually do (skills)
- **Implement stratified random sampling of high-confidence extractions** for ongoing error-rate measurement and novel-pattern detection.
- **Analyze accuracy by document type and field** to verify consistent performance across all segments **before reducing human review.**
- **Have models output field-level confidence scores, then calibrate review thresholds** using labeled validation sets.
- **Route low-confidence or ambiguous/contradictory-source extractions to human review**, prioritizing limited reviewer capacity where it matters most.

### Anti-patterns / distractors
- Trusting a single **aggregate** accuracy number and dropping review (it can hide bad segments).
- Using **uncalibrated** model confidence to set thresholds.
- Reviewing randomly/uniformly instead of routing by **confidence + segment risk.**

### Exam signals & facts to memorize
- **Aggregate accuracy hides per-type/per-field weakness → segment before automating.**
- **Stratified sampling** finds error rates + novel patterns in high-confidence output.
- **Field-level confidence, calibrated on labeled data,** routes humans to the riskiest cases.

---

## Task Statement 5.6 — Preserve information provenance and handle uncertainty in multi-source synthesis

### The real-world problem
The research system (Scenario 3) synthesizes a report from many sources. During summarization, **source attribution gets lost** — the final report states facts with no citations. Two credible sources give **different statistics** for the same metric, and the synthesis just picks one. Two sources differ only because they were **collected in different years**, but the report frames them as contradictory. You need provenance and uncertainty to survive synthesis.

### What you must know
- **Source attribution is lost during summarization** when findings are compressed **without preserving claim→source mappings.**
- **Structured claim-source mappings** must be preserved and merged when combining findings.
- **Conflicting statistics from credible sources** should be **annotated with source attribution**, *not* arbitrarily resolved by picking one value.
- **Temporal data:** require **publication/collection dates** in structured outputs so temporal differences aren't misread as contradictions.

### What you'd actually do (skills)
- **Require subagents to output structured claim-source mappings** (source URLs, document names, relevant excerpts) that downstream agents **preserve through synthesis.**
- **Structure reports with explicit sections distinguishing well-established findings from contested ones**, preserving original source characterizations and methodological context.
- **Complete document analysis with conflicting values included and explicitly annotated**, letting the **coordinator decide** how to reconcile before passing to synthesis.
- **Require publication/data-collection dates** in structured outputs for correct temporal interpretation.
- **Render different content types appropriately** — financial data as **tables**, news as **prose**, technical findings as **structured lists** — rather than forcing everything into one uniform format.

```text
# Claim-source mapping that survives synthesis
{ "claim": "AI tools used by 38% of studios",
  "source": "https://example.org/report-2024", "excerpt": "...38% of studios...",
  "collected": "2024-09" }
# Conflict preserved, not resolved arbitrarily:
{ "metric": "adoption rate",
  "values": [ {"v":"38%","src":"A","collected":"2024-09"},
              {"v":"21%","src":"B","collected":"2022-03"} ],
  "note": "difference likely temporal (2024 vs 2022), not contradictory" }
```

### Anti-patterns / distractors
- Summarizing claims without their sources → an uncited report.
- **Arbitrarily picking one** of two conflicting credible statistics instead of annotating both.
- Treating **temporally-different** figures as contradictions because dates weren't carried.
- Flattening tables, prose, and lists into one uniform format.

### Exam signals & facts to memorize
- **Preserve claim→source mappings through every summarization/synthesis step.**
- **Conflicting credible sources → annotate both with attribution, don't pick one.**
- **Carry publication/collection dates** so temporal gaps aren't mistaken for contradictions.
- **Separate well-established from contested findings; render content types in their natural format.**

---

## Domain 5 — putting it together (decision flowchart)

```
Long conversation losing critical facts?
  → Extract exact facts (amounts/dates/IDs) into a persistent "case facts" block;
    trim verbose tool outputs; key findings at start/end (lost-in-the-middle). (5.1)

When to escalate / handle ambiguity?
  → Escalate on: explicit human request, policy gap, no progress.
  → Explicit demand → now; straightforward → offer to resolve.
  → NOT sentiment, NOT self-confidence; multiple matches → ask for identifiers. (5.2)

Subagent failed?
  → Propagate STRUCTURED error (type, attempted query, partial results, alternatives);
    distinguish access-failure vs empty result; recover locally first;
    never generic/silent/whole-workflow-kill. (5.3)

Long codebase exploration degrading?
  → Scratchpad files + subagent delegation + /compact;
    crash recovery via structured state manifests. (5.4)

Thinking about dropping human review?
  → Don't trust aggregate accuracy; segment by doc type & field;
    stratified sampling; calibrated field-level confidence routes humans. (5.5)

Synthesizing many sources?
  → Preserve claim->source mappings; annotate conflicts (don't pick one);
    carry dates (temporal != contradiction); render content types naturally. (5.6)
```

### The five Domain-5 truths most likely to earn you points
1. **Don't let summarization eat exact facts** — pull amounts/dates/IDs into a persistent layer; mind "lost in the middle"; trim verbose tool outputs.
2. **Escalate on explicit request / policy gap / no progress** — never on sentiment or self-reported confidence; ask for identifiers on multiple matches.
3. **Propagate structured errors** (type + attempted query + partial results + alternatives); access-failure ≠ empty result; never suppress or kill the whole workflow.
4. **Fight context degradation with scratchpads, subagent delegation, `/compact`, and state manifests** for crash recovery.
5. **Aggregate accuracy hides bad segments** — segment + stratified sample + calibrated confidence; and **preserve provenance + annotate conflicts + carry dates** through synthesis.
