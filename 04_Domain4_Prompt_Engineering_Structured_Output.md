# Domain 4 — Prompt Engineering & Structured Output
### Weight: 20% · Task statements 4.1 – 4.6

> **What this domain is about.** This is the "make the model's output precise, consistent, machine-parseable, and trustworthy" domain. It covers explicit criteria (to cut false positives), few-shot prompting (for consistency and generalization), `tool_use` + JSON schemas (for guaranteed structure), validation/retry loops (for extraction quality), the Message Batches API (for cost-efficient bulk work), and multi-instance/multi-pass review (for catching what a single pass misses). Scenarios 5 (CI review) and 6 (Structured Extraction) drive this domain.

**Mental model:** structure and precision come from **showing, schematizing, and validating**, not from vague exhortations. "Be conservative" and "only report high-confidence findings" *do not work*. Explicit criteria, few-shot examples, JSON-schema tool use, and validation loops *do*.

---

## Task Statement 4.1 — Design prompts with explicit criteria to improve precision and reduce false positives

### The real-world problem
Your CI code-review bot (Scenario 5) flags too many non-issues — minor style nits, local patterns that are actually fine — and developers start ignoring it. You tried adding "be conservative" and "only report high-confidence findings" to the prompt. It didn't help. Now developers don't trust *any* of its findings, even the accurate security ones. How do you restore precision and trust?

### What you must know
- **Explicit criteria beat vague instructions.** "Flag comments only when the claimed behavior contradicts the actual code behavior" is far better than "check that comments are accurate."
- **General hedges fail.** "Be conservative" / "only report high-confidence findings" do **not** improve precision the way **specific categorical criteria** do. (The model is already mis-calibrated about its own confidence.)
- **False positives poison trust transitively:** high false-positive categories **undermine confidence in the accurate categories** too. One noisy category can sink the whole tool's credibility.

### What you'd actually do (skills)
- **Write specific review criteria** defining which issues to **report** (bugs, security) versus **skip** (minor style, local patterns) — rather than relying on confidence-based filtering.
- **Temporarily disable high-false-positive categories** to restore developer trust *while* you improve the prompts for those categories. (Better to show fewer, trustworthy findings than many noisy ones.)
- **Define explicit severity criteria with concrete code examples** for each severity level, so classification is consistent.

```text
# Explicit criteria > vague instruction
BAD:  "Be conservative and only report high-confidence issues."
GOOD: "REPORT: (1) security vulns (injection, auth bypass), (2) bugs that change
        behavior (off-by-one, null deref). SKIP: naming/style, local conventions,
        formatting. SEVERITY: CRITICAL = exploitable security or data loss;
        MAJOR = incorrect results; MINOR = maintainability. Example CRITICAL: <code>."
```

### Anti-patterns / distractors
- "Be conservative" / "only high-confidence" → does not move precision; the model's self-confidence is unreliable.
- Leaving a noisy category on while you "tune the prompt" → keeps eroding trust; disable it temporarily instead.
- Confidence-threshold filtering instead of **categorical** criteria.

### Exam signals & facts to memorize
- **Specific categorical criteria > vague hedges.** ("Contradicts actual behavior," not "be accurate.")
- **One high-FP category undermines trust in all categories** → temporarily disable it, fix offline.
- **Severity needs concrete examples per level** for consistency.

---

## Task Statement 4.2 — Apply few-shot prompting to improve output consistency and quality

### The real-world problem
Detailed instructions alone still produce inconsistently-formatted review comments and inconsistent handling of ambiguous cases (a request that could go to two tools; a document with citations inline vs in a bibliography). You need the model to *generalize* good judgment to cases you didn't enumerate.

### What you must know
- **Few-shot examples are the *most effective* technique** for consistently formatted, actionable output when detailed instructions alone fall short.
- They **demonstrate ambiguous-case handling** (tool selection for ambiguous requests, branch-level coverage gaps).
- They let the model **generalize to novel patterns** rather than matching only the pre-specified cases.
- They **reduce hallucination in extraction** (informal measurements, varied document structures).

### What you'd actually do (skills)
- **Create 2–4 targeted few-shot examples for ambiguous scenarios** that **show the reasoning** for why one action was chosen over plausible alternatives (not just the answer — the *why*).
- **Include examples demonstrating the desired output format** (location, issue, severity, suggested fix) for consistency.
- **Provide examples distinguishing acceptable patterns from genuine issues** to reduce false positives *while enabling generalization* (complements 4.1).
- **Use examples for varied document structures** (inline citations vs bibliographies, methodology sections vs embedded details).
- **Add examples showing correct extraction from varied formats** to fix empty/null extraction of required fields.

```text
# Few-shot example that shows REASONING (ambiguous tool selection)
User: "What's going on with 12345?"
Reasoning: "12345" matches an order-number pattern, not a customer ID format,
so this is an ORDER query.
Action: call lookup_order(order_id="12345")   # not get_customer
```

### Anti-patterns / distractors
- Relying on ever-longer prose instructions when **2–4 examples** would fix consistency.
- Examples that show only the output but **not the reasoning** for ambiguous picks (less generalization).
- Enumerating every case instead of giving representative examples that generalize.

### Exam signals & facts to memorize
- **Few-shot = the most effective fix for consistency/format and ambiguous-case judgment.**
- **2–4 targeted examples; show the *reasoning*, not just the answer.**
- They **generalize** to novel patterns and **reduce extraction hallucination/empty fields.**

---

## Task Statement 4.3 — Enforce structured output using tool use and JSON schemas

### The real-world problem
Your extraction system (Scenario 6) must output clean JSON that downstream systems parse. Free-text JSON occasionally has syntax errors (trailing commas, unescaped quotes) and breaks the pipeline. Some documents don't contain a field you ask for, and the model *fabricates* a value to fill it. You need guaranteed-valid structure that doesn't invent data.

### What you must know
- **`tool_use` with JSON schemas is the *most reliable* approach for guaranteed schema-compliant output** — it **eliminates JSON syntax errors.**
- **`tool_choice` distinctions (memorize):**
  - **`"auto"`** — the model **may return text instead of** calling a tool.
  - **`"any"`** — the model **must call a tool** but can choose which.
  - **Forced** — `{"type": "tool", "name": "..."}` — the model must call a **specific** named tool.
- **Strict schemas eliminate *syntax* errors but NOT *semantic* errors.** Tool use guarantees valid JSON shape; it does **not** guarantee line items sum to the total, or that values landed in the right fields. (Semantic validation is Domain 4.4.)
- **Schema design:** required vs optional fields; **`enum` with an `"other"` + detail string** pattern for extensible categories.

### What you'd actually do (skills)
- **Define extraction tools with JSON schemas as input parameters**, then read the structured data out of the **`tool_use` response.**
- **Set `tool_choice: "any"`** to guarantee structured output when **multiple extraction schemas exist and the document type is unknown** (the model must pick *one* extraction tool, not return prose).
- **Force a specific tool** with `tool_choice: {"type": "tool", "name": "extract_metadata"}` to ensure a particular extraction **runs before enrichment** steps.
- **Make fields optional/nullable when the source may not contain them** — this *prevents the model from fabricating values* to satisfy required fields. (Directly fixes the hallucination problem above.)
- **Add `enum` values like `"unclear"`** for ambiguous cases and **`"other"` + a detail field** for extensible categories.
- **Include format-normalization rules in the prompt** alongside the strict schema to handle inconsistent source formatting.

```json
// Extraction tool schema — nullable fields prevent fabrication
{
  "name": "extract_invoice",
  "input_schema": {
    "type": "object",
    "properties": {
      "invoice_number": { "type": "string" },
      "tax_id":         { "type": ["string", "null"] },   // nullable: may be absent
      "category":       { "type": "string", "enum": ["goods","services","other"] },
      "category_detail":{ "type": ["string", "null"] }     // used when category="other"
    },
    "required": ["invoice_number", "category"]
  }
}
```

### Anti-patterns / distractors
- Asking for JSON in free text and parsing it → syntax errors; use `tool_use`.
- Marking everything `required` → the model fabricates values for missing data; make absent-able fields **nullable**.
- Believing strict schema = correct data → it only guarantees *shape*, not *semantics* (sums, field placement).

### Exam signals & facts to memorize
- **`tool_use` + JSON schema = guaranteed valid structure, zero syntax errors.**
- **`tool_choice`: `"auto"` (text OK) | `"any"` (must call a tool) | forced (specific tool).**
- **Schema kills *syntax* errors, not *semantic* ones.**
- **Nullable/optional fields prevent fabrication; `enum` + `"other"`/`"unclear"` for extensibility.**

---

## Task Statement 4.4 — Implement validation, retry, and feedback loops for extraction quality

### The real-world problem
Your extraction passes the JSON schema but the **line items don't sum to the stated total**, and sometimes a value lands in the wrong field. You add a retry — but for documents where the needed info simply **isn't in the source**, retries never help and just burn money. You need to know *when* a retry will work, and how to validate *meaning*, not just shape.

### What you must know
- **Retry-with-error-feedback:** on retry, **append the specific validation errors to the prompt** to steer the model toward correction.
- **The limit of retry:** retries are **ineffective when the required information is simply absent** from the source (as opposed to format/structural errors, which retries *can* fix).
- **Feedback-loop design:** track which code constructs trigger findings via a **`detected_pattern` field** to enable systematic analysis of dismissal patterns (which categories developers keep dismissing = false-positive sources).
- **Semantic vs syntax errors:** *semantic* errors (values don't sum, wrong field placement) are the ones you must validate yourself; *syntax* errors are already eliminated by `tool_use` (4.3).

### What you'd actually do (skills)
- **Implement follow-up requests that include the original document, the failed extraction, and the specific validation errors** for self-correction.
- **Identify when retries won't help** (info only exists in an external document you didn't provide) **versus when they will** (format mismatches, structural output errors). Don't retry the unwinnable cases.
- **Add `detected_pattern` fields** to structured findings so you can analyze false-positive patterns when developers dismiss findings.
- **Design self-correction validation flows:** extract **`calculated_total` alongside `stated_total`** to flag discrepancies; add a **`conflict_detected` boolean** for inconsistent source data.

```text
# Retry with error feedback (works for FORMAT/STRUCTURE errors)
Original document: <...>
Your previous extraction: { "line_items":[...], "total": 540 }
Validation error: line items sum to 520 but "total" = 540. Re-extract and reconcile.

# Do NOT retry this (information is ABSENT from source):
Validation error: "tax_id" missing.  -> If it's not in the document, return null, don't retry.
```

### Anti-patterns / distractors
- Retrying when the data is **absent from the source** → wasted calls; return null instead.
- Trusting schema validity as proof of correctness → add **semantic** checks (sum, field placement, conflicts).
- No `detected_pattern` tracking → you can't analyze *why* developers dismiss findings.

### Exam signals & facts to memorize
- **Retry works for format/structural errors; useless when info is absent from the source.**
- **On retry, include the document + failed extraction + specific errors.**
- **Validate semantics yourself:** `calculated_total` vs `stated_total`, `conflict_detected`.
- **`detected_pattern` field → analyze false-positive/dismissal patterns.**

---

## Task Statement 4.5 — Design efficient batch processing strategies

### The real-world problem
You run two workflows (Sample Question 11): (1) a **blocking pre-merge check** developers wait on, and (2) an **overnight technical-debt report.** Your manager wants to move *both* to the Message Batches API for the 50% cost savings. You also need to process 100 extraction documents and handle the ones that fail. When is batch right, and when is it a trap?

### What you must know (Message Batches API)
- **Message Batches API:** **~50% cost savings**, **up to a 24-hour processing window**, and **no guaranteed latency SLA.**
- **Appropriate for non-blocking, latency-tolerant workloads:** overnight reports, weekly audits, nightly test generation.
- **Inappropriate for blocking workflows:** pre-merge checks where a developer is waiting.
- **The batch API does NOT support multi-turn tool calling within a single request** — it cannot execute tools mid-request and feed results back. (So agentic loops can't run inside one batch request.)
- **`custom_id`** correlates each request with its response in the batch.

### What you'd actually do (skills)
- **Match the API to latency requirements:** **synchronous API for blocking pre-merge checks**, **batch API for overnight/weekly analysis.** (Sample Q11 answer **A**: batch the debt report, keep real-time for the pre-merge check.)
- **Calculate batch submission frequency from SLA constraints** — e.g., submit on **4-hour windows to guarantee a 30-hour SLA** given up-to-24-hour batch processing.
- **Handle failures by resubmitting only the failed documents** (identified by **`custom_id`**) with appropriate modifications (e.g., **chunking documents that exceeded context limits**).
- **Refine the prompt on a sample set before batch-processing large volumes** to maximize first-pass success and reduce costly resubmission cycles.

### Why Sample Q11 answer is A (and the rest are wrong)
- **A** — batch the overnight debt report (latency-tolerant), keep real-time for the blocking pre-merge check. ✔
- **B** (batch both with status polling) — "often faster" isn't acceptable for a blocking workflow with no SLA.
- **C** (keep real-time for both to avoid "ordering issues") — a misconception; **`custom_id`** correlates batch results.
- **D** (batch both with a timeout fallback to real-time) — needless complexity vs. simply matching each API to its use case.

### Anti-patterns / distractors
- Putting a **blocking** workflow on the batch API (no latency SLA) to save money.
- Trying to run a **multi-turn tool-calling agent loop inside a single batch request** (unsupported).
- Resubmitting the *entire* batch when only some documents failed (use `custom_id` to resubmit just those).

### Exam signals & facts to memorize
- **Message Batches API: ~50% cheaper, up to 24h, NO latency SLA, NO multi-turn tool calling, `custom_id` correlation.**
- **Batch = non-blocking/overnight; synchronous = blocking/pre-merge.**
- **Resubmit only failed `custom_id`s, with fixes (e.g., chunking oversized docs).**
- **Sample on a small set first** to maximize batch first-pass success.

---

## Task Statement 4.6 — Design multi-instance and multi-pass review architectures

### The real-world problem
Your CI reviewer (Scenario 5) misses subtle bugs in code it *just generated*, and on a 14-file PR it gives deep feedback on some files and shallow feedback on others — even contradicting itself (flagging a pattern in one file, approving identical code elsewhere). Telling it to "review carefully" or "think harder" doesn't fix it. The architecture is the problem.

### What you must know
- **Self-review limitation:** a model **retains its reasoning context from generation**, making it **less likely to question its own decisions** in the same session.
- **Independent review instances** (without the generator's prior reasoning context) are **more effective at catching subtle issues** than self-review instructions or extended thinking.
- **Multi-pass review:** split large reviews into **per-file local analysis passes + cross-file integration passes** to avoid **attention dilution and contradictory findings.**

### What you'd actually do (skills)
- **Use a second, independent Claude instance to review generated code** without the generator's reasoning context.
- **Split large multi-file reviews into focused per-file passes** (local issues) **plus a separate integration pass** (cross-file data flow). (Same fix as Domain 1.6 / Sample Q12 — answer **A**.)
- **Run verification passes where the model self-reports confidence alongside each finding** to enable calibrated review routing (cf. Domain 5.5).

### Anti-patterns / distractors
- Asking the generating session to review itself, or relying on "extended thinking," instead of an **independent instance.**
- Reviewing all 14 files in one pass → attention dilution and contradictions; split into per-file + integration passes.
- A **bigger context window** to "fit all files" — does **not** fix attention quality.
- Majority-vote across three full-PR runs — **suppresses** real bugs caught only intermittently.

### Exam signals & facts to memorize
- **Self-review is weak — use an independent instance** (no shared reasoning context).
- **Big reviews → per-file passes + cross-file integration pass** (cures attention dilution + contradictions).
- **Larger context window ≠ better attention; consensus voting hides intermittent bugs.**

---

## Domain 4 — putting it together (decision flowchart)

```
Too many false positives / inconsistent precision?
  → Explicit categorical criteria (report X, skip Y), not "be conservative";
    disable noisy categories temporarily; severity examples per level. (4.1)

Inconsistent format or ambiguous-case handling?
  → 2-4 few-shot examples that SHOW REASONING; generalize, don't enumerate. (4.2)

Need guaranteed machine-parseable output?
  → tool_use + JSON schema (kills syntax errors).
  → tool_choice: "any" (unknown doc type), forced (specific tool first).
  → nullable fields prevent fabrication; enum + "other"/"unclear" for edge cases. (4.3)

Output valid-shaped but semantically wrong / or fields missing?
  → Validate semantics (calculated vs stated total, conflict_detected).
  → Retry with document + failed extraction + errors — BUT not if info is absent. (4.4)

Processing lots of documents / cost pressure?
  → Batch API (50% off, <=24h, NO SLA, NO multi-turn tools, custom_id) for non-blocking;
    synchronous for blocking pre-merge; resubmit failed custom_ids only. (4.5)

Missing subtle bugs / inconsistent multi-file review?
  → Independent review instance (not self-review);
    per-file + cross-file passes; NOT a bigger context window. (4.6)
```

### The five Domain-4 truths most likely to earn you points
1. **Precision comes from explicit categorical criteria, not "be conservative"** — and one noisy category poisons trust in all of them.
2. **Few-shot examples (2–4, with reasoning) are the top fix for consistency and ambiguous-case judgment.**
3. **`tool_use` + JSON schema guarantees structure (syntax), not semantics;** nullable fields stop fabrication; `tool_choice` `"any"`/forced control *whether/which* tool runs.
4. **Retry only helps format/structural errors — never when the info is absent from the source;** validate semantics yourself.
5. **Batch API = 50% cheaper / ≤24h / no SLA / no multi-turn tools / `custom_id`** → non-blocking only; and **independent review instances + per-file passes** beat self-review and big context windows.
