# Domain 3 — Claude Code Configuration & Workflows
### Weight: 20% · Task statements 3.1 – 3.6

> **What this domain is about.** Claude Code is the coding agent; Domain 3 is how you *configure and drive it for a team*: where instructions live (CLAUDE.md hierarchy), how you package reusable workflows (slash commands and skills), how conventions load conditionally (path rules), when to plan vs just do it (plan mode vs direct execution), how to iterate toward correct code (refinement techniques), and how to run Claude Code headless inside CI/CD. Scenarios 2 (Code Generation) and 5 (CI) drive this domain.

**Mental model:** configuration *scope* is the recurring theme. Almost every Domain-3 question hinges on **shared-via-version-control (project)** vs **personal (user)** vs **conditional (path-scoped)**. Get the scope right and most questions answer themselves.

---

## Task Statement 3.1 — Configure CLAUDE.md files with appropriate hierarchy, scoping, and modular organization

### The real-world problem
A new teammate clones the repo but *isn't getting* the coding instructions everyone else seems to have — Claude behaves differently for them (Scenario 2). Separately, your root `CLAUDE.md` has grown into a 600-line monolith that's hard to maintain. Both are CLAUDE.md hierarchy/organization problems.

### What you must know (the hierarchy)
- **Three levels of CLAUDE.md:**
  - **User-level — `~/.claude/CLAUDE.md`** → applies only to *that user* on *their* machine. **Not shared with teammates via version control.**
  - **Project-level — `.claude/CLAUDE.md` or root `CLAUDE.md`** → committed to the repo, shared with everyone who clones/pulls.
  - **Directory-level — subdirectory `CLAUDE.md`** → applies when working within that directory's subtree.
- **The teammate bug:** if instructions live in **`~/.claude/CLAUDE.md`** (user-level), a new team member won't receive them — they're not in version control. The fix is to move them to **project-level** config.
- **`@import` syntax** references external files so CLAUDE.md stays **modular** — e.g., a package's CLAUDE.md imports just the standards files relevant to it.
- **`.claude/rules/` directory** organizes topic-specific rule files as an alternative to one giant CLAUDE.md (more in 3.3).

### What you'd actually do (skills)
- **Diagnose hierarchy issues:** "new teammate isn't getting instructions" → the instructions are at **user level**, not project level. Move them into the committed project config.
- **Use `@import` to selectively include relevant standards** in each package's CLAUDE.md based on what that package needs (the maintainer knows the domain).
- **Split a large CLAUDE.md** into focused files in **`.claude/rules/`** — e.g., `testing.md`, `api-conventions.md`, `deployment.md`.
- **Use the `/memory` command** to verify *which* memory files are loaded and diagnose inconsistent behavior across sessions.

```text
# Root CLAUDE.md kept modular via imports
@import .claude/rules/testing.md
@import .claude/rules/api-conventions.md
@import .claude/rules/deployment.md
```

### Anti-patterns / distractors
- Putting team-wide standards in `~/.claude/CLAUDE.md` → teammates never get them.
- Letting CLAUDE.md balloon into one unmaintainable file instead of using `@import` / `.claude/rules/`.
- Guessing why behavior is inconsistent instead of running **`/memory`** to see what's actually loaded.

### Exam signals & facts to memorize
- **User `~/.claude/CLAUDE.md` (private) → Project `.claude/CLAUDE.md` or root (shared) → Directory (subtree).**
- **"Teammate not getting instructions" = they're at user level; move to project level.**
- **`@import` = modular includes; `.claude/rules/` = topic files; `/memory` = inspect what's loaded.**

---

## Task Statement 3.2 — Create and configure custom slash commands and skills

### The real-world problem
You want a `/review` command that runs your team's code-review checklist, **available to every developer when they clone or pull the repo** (Sample Question 4). You also have a "codebase analysis" skill that dumps tons of verbose output into the conversation and ruins the main context. And you want a personal tweak to a shared skill without affecting teammates. These are slash-command and skill configuration problems.

### What you must know
- **Slash command scope:**
  - **Project — `.claude/commands/`** → shared via version control, available to everyone on clone/pull. (Sample Q4 answer = **A**.)
  - **User — `~/.claude/commands/`** → personal, not shared.
- **Skills live in `.claude/skills/`** with a **`SKILL.md`** file supporting **frontmatter**:
  - **`context: fork`** → run the skill in an **isolated sub-agent context**, so its output doesn't pollute the main conversation.
  - **`allowed-tools`** → restrict which tools the skill may use during execution.
  - **`argument-hint`** → prompt the developer for required parameters if they invoke without arguments.
- **Personal skill customization:** create a personal variant in **`~/.claude/skills/` with a *different name*** to avoid affecting teammates.
- **Skills vs CLAUDE.md:** **skills = on-demand** invocation for task-specific workflows; **CLAUDE.md = always-loaded** universal standards.

### What you'd actually do (skills)
- **Create project-scoped slash commands in `.claude/commands/`** for team-wide availability via version control (the `/review` command).
- **Use `context: fork`** to isolate skills that produce **verbose output** (codebase analysis) or **exploratory context** (brainstorming alternatives) from the main session.
- **Configure `allowed-tools`** in skill frontmatter to restrict tool access during execution (e.g., limit to file-write operations to prevent destructive actions).
- **Use `argument-hint`** to prompt for required parameters when the skill is invoked without arguments.
- **Choose skills vs CLAUDE.md** correctly: on-demand task workflow → **skill**; always-on universal standard → **CLAUDE.md**.

```yaml
# .claude/skills/codebase-analysis/SKILL.md (frontmatter)
---
name: codebase-analysis
context: fork              # isolate verbose output from the main conversation
allowed-tools: [Read, Grep, Glob]   # read-only; cannot write/delete
argument-hint: "<path or module to analyze>"
---
```

### Why Sample Q4 answer is A (and the rest are wrong)
- **B (`~/.claude/commands/`)** → personal, *not* shared via version control. Fails the "every developer on clone" requirement.
- **C (CLAUDE.md)** → holds project instructions/context, *not* command definitions.
- **D (`.claude/config.json` with a commands array)** → describes a mechanism that **doesn't exist** in Claude Code.

### Anti-patterns / distractors
- Putting a team command in `~/.claude/commands/` (personal) when it must be shared.
- Letting a verbose skill run in the main context instead of `context: fork`.
- Inventing config files/arrays that don't exist (the exam plants these).
- Overwriting a shared skill in place instead of making a differently-named personal variant.

### Exam signals & facts to memorize
- **`.claude/commands/` = shared/project; `~/.claude/commands/` = personal.**
- **Skill frontmatter: `context: fork` (isolation), `allowed-tools` (restriction), `argument-hint` (prompt for args).**
- **Personal skill variant → `~/.claude/skills/` with a different name.**
- **Skills = on-demand; CLAUDE.md = always-loaded.**

---

## Task Statement 3.3 — Apply path-specific rules for conditional convention loading

### The real-world problem
Your codebase has different conventions per area: React components use hooks, API handlers use async/await with specific error handling, DB models use a repository pattern. Crucially, **test files are spread throughout the codebase** next to the code they test (`Button.test.tsx` beside `Button.tsx`), and you want *all* tests to follow the same conventions regardless of location (Sample Question 6 — answer **A**). How do you make Claude apply the right conventions automatically based on the file being edited?

### What you must know
- **`.claude/rules/` files carry YAML frontmatter with a `paths:` field of glob patterns** for **conditional activation.**
- **Path-scoped rules load *only* when editing matching files** — reducing irrelevant context and token usage.
- **Glob-pattern rules beat directory-level CLAUDE.md** when conventions must span **multiple directories** (like test files scattered everywhere). A directory `CLAUDE.md` is *bound to its directory*; a glob like `**/*.test.tsx` follows the file *type* anywhere.

### What you'd actually do (skills)
- **Create `.claude/rules/` files with YAML `paths:` scoping**, e.g. `paths: ["terraform/**/*"]`, so the rule loads only when editing matching files.
- **Use glob patterns to apply conventions by file type regardless of directory** — `**/*.test.tsx` for all test files everywhere.
- **Choose path-specific rules over subdirectory CLAUDE.md** when conventions must apply to files **spread across the codebase.**

```yaml
# .claude/rules/testing.md
---
paths: ["**/*.test.tsx", "**/*.test.ts"]   # loads for ANY test file, any directory
---
# Testing conventions
- Use React Testing Library; no enzyme.
- One describe block per component; arrange-act-assert.
```

### Why Sample Q6 answer is A (and the rest are wrong)
- **A** — `.claude/rules/` with glob `**/*.test.tsx` applies automatically by path, anywhere. ✔
- **B** (consolidate in root CLAUDE.md under headers) — relies on Claude *inferring* which section applies; unreliable.
- **C** (skills per code type) — requires manual/optional invocation; contradicts "automatic."
- **D** (a CLAUDE.md per subdirectory) — can't handle files spread across many directories (CLAUDE.md is directory-bound).

### Exam signals & facts to memorize
- **`.claude/rules/` + YAML `paths:` globs = conditional, automatic, by-file-path loading.**
- **Globs follow file *type* across directories; directory CLAUDE.md is *location-bound*.**
- **Scattered-file conventions (tests, terraform) → path rules, not subdirectory CLAUDE.md.**
- Bonus: path-scoped rules **reduce token usage** by loading only when relevant.

---

## Task Statement 3.4 — Determine when to use plan mode vs direct execution

### The real-world problem
You're told to **restructure a monolith into microservices** — dozens of files, decisions about service boundaries and module dependencies (Sample Question 5). Elsewhere, you have a one-line date-validation bug with a clear stack trace. One of these screams "plan first," the other screams "just do it." Choosing wrong wastes time or causes costly rework.

### What you must know
- **Plan mode is for complex tasks:** large-scale changes, **multiple valid approaches**, **architectural decisions**, and **multi-file modifications.** It enables **safe codebase exploration and design *before* committing to changes**, preventing costly rework.
- **Direct execution is for simple, well-scoped changes** — e.g., adding a single validation check to one function.
- **The `Explore` subagent** isolates **verbose discovery output** and returns **summaries**, preserving the main conversation's context during multi-phase work.

### What you'd actually do (skills)
- **Choose plan mode for architectural tasks:** microservice restructuring, **library migrations affecting 45+ files**, choosing between integration approaches with different infrastructure requirements.
- **Choose direct execution for well-understood, clearly-scoped changes:** a single-file bug fix with a clear stack trace, adding a date-validation conditional.
- **Use the `Explore` subagent** for verbose discovery phases to **prevent context-window exhaustion** during multi-phase tasks (cf. Domain 5.4).
- **Combine both:** use plan mode to *investigate/design* (e.g., plan a library migration), then **direct execution to implement** the planned approach.

### Why Sample Q5 answer is A (and the rest are wrong)
- **A** — enter plan mode to explore, understand dependencies, and design before changing. ✔ (The complexity is *already stated* in the requirements.)
- **B** (start direct, let boundaries emerge) — risks costly rework when dependencies surface late.
- **C** (direct with comprehensive upfront instructions) — assumes you already know the right structure without exploring.
- **D** (start direct, switch to plan only if complexity appears) — ignores that the complexity is *already known* up front.

### Exam signals & facts to memorize
- **Plan mode → large-scale, multi-approach, architectural, multi-file.** It prevents costly rework.
- **Direct execution → small, clear-scope, single-file.**
- **`Explore` subagent → isolate verbose discovery, return summaries, protect context.**
- **Common combo: plan the design, then directly execute it.**

---

## Task Statement 3.5 — Apply iterative refinement techniques for progressive improvement

### The real-world problem
You describe a data transformation in prose and Claude keeps interpreting it inconsistently. A migration script keeps mishandling null values. You're implementing caching in an unfamiliar domain and aren't sure what you're forgetting. Each calls for a different *refinement* technique.

### What you must know (four techniques)
- **Concrete input/output examples** are the *most effective* way to communicate an expected transformation when prose is interpreted inconsistently.
- **Test-driven iteration:** write the test suite first, then iterate by **sharing test failures** to guide progressive improvement.
- **The interview pattern:** have Claude **ask you questions** to surface considerations you didn't anticipate *before* implementing (great for unfamiliar domains).
- **Single message vs sequential fixes:** provide **all issues in one message when they *interact***; fix **sequentially when they're *independent*.**

### What you'd actually do (skills)
- **Give 2–3 concrete input/output examples** to nail down transformation requirements when natural language produces inconsistent results.
- **Write test suites covering expected behavior, edge cases, and performance** *before* implementation, then iterate by sharing failures.
- **Use the interview pattern** to surface design considerations (cache invalidation strategy, failure modes) before implementing in an unfamiliar domain.
- **Provide specific test cases (input + expected output)** to fix edge-case handling (e.g., null values in migration scripts).
- **Batch interacting issues into one detailed message; iterate sequentially for independent issues.**

```text
# Communicating a transformation with examples (beats prose)
Input:  "12 oz"      -> Output: { "value": 12, "unit": "ounce" }
Input:  "1.5kg"      -> Output: { "value": 1.5, "unit": "kilogram" }
Input:  "a dozen"    -> Output: { "value": 12, "unit": "count" }   # informal case
```

### Anti-patterns / distractors
- Re-writing prose descriptions over and over instead of giving examples.
- Fixing interacting bugs one at a time (each fix breaks another) instead of one combined message.
- Implementing first in an unfamiliar domain instead of letting Claude interview you to surface unknowns.

### Exam signals & facts to memorize
- **Inconsistent transformation → concrete input/output examples.**
- **Quality bar → test-first, iterate on failures.**
- **Unfamiliar domain → interview pattern (Claude asks you questions first).**
- **Interacting issues → one message; independent issues → sequential.**

---

## Task Statement 3.6 — Integrate Claude Code into CI/CD pipelines

### The real-world problem
Your CI job runs `claude "Analyze this PR for security issues"` and **hangs forever** waiting for interactive input (Sample Question 10). You need machine-parseable output to post as inline PR comments. Re-running reviews after each commit spams duplicate comments. And the same session that *wrote* code is suspiciously bad at *reviewing* it. These are all CI integration problems (Scenario 5).

### What you must know (CLI flags + isolation)
- **`-p` (or `--print`)** runs Claude Code in **non-interactive mode** — processes the prompt, prints to stdout, exits without waiting for input. **This is the fix for the hanging CI job** (Sample Q10 answer **A**).
- **`--output-format json` with `--json-schema`** enforces **structured, machine-parseable output** in CI (for posting as inline PR comments).
- **CLAUDE.md provides project context to CI-invoked Claude Code** — testing standards, fixture conventions, review criteria.
- **Session context isolation:** the **same session that generated code is worse at reviewing its own changes** than an **independent review instance** (it retains its own reasoning context — cf. Domain 4.6).

### What you'd actually do (skills)
- **Run Claude Code in CI with `-p`** to prevent interactive hangs.
- **Use `--output-format json` + `--json-schema`** to produce machine-parseable findings posted as inline PR comments.
- **Include prior review findings in context when re-running after new commits**, instructing Claude to report **only new or still-unaddressed issues** — avoids duplicate comments.
- **Provide existing test files in context** so test generation **doesn't duplicate** scenarios already covered.
- **Document testing standards, valuable-test criteria, and available fixtures in CLAUDE.md** to improve test generation quality and reduce low-value output.

```bash
# Headless CI review with structured output
claude -p "Review the staged diff for security and correctness issues." \
  --output-format json \
  --json-schema ./review-schema.json
```

### Why Sample Q10 answer is A (and the rest are wrong)
- **A (`-p` flag)** — the documented non-interactive mode; processes, prints, exits. ✔
- **B (`CLAUDE_HEADLESS=true`)** — a **non-existent** environment variable.
- **C (`< /dev/null`)** — a Unix workaround that doesn't properly address Claude Code's syntax.
- **D (`--batch`)** — a **non-existent** flag (don't confuse with the *Message Batches API*, which is a different thing in Domain 4).

### Exam signals & facts to memorize
- **`-p` / `--print` = non-interactive CI mode** (the cure for hangs).
- **`--output-format json` + `--json-schema` = structured CI output for inline PR comments.**
- **CLAUDE.md carries CI context** (standards, fixtures, criteria).
- **Independent review instance > self-review** (the generator retains its reasoning).
- **Re-running reviews: feed prior findings, report only new/unaddressed.** Feed existing tests to avoid duplicates.

---

## Domain 3 — putting it together (decision flowchart)

```
Where do instructions/commands/skills live?
  → Shared with the team (clone/pull) → PROJECT scope:
       instructions: .claude/CLAUDE.md or root CLAUDE.md
       commands:     .claude/commands/
       skills:       .claude/skills/
  → Personal only → USER scope: ~/.claude/CLAUDE.md, ~/.claude/commands/, ~/.claude/skills/
  → Conditional by file type/path → .claude/rules/ with YAML paths: globs (3.3)

Big monolithic CLAUDE.md?
  → @import modular files / split into .claude/rules/. Use /memory to inspect. (3.1)

Skill output polluting the chat / needs guardrails / needs args?
  → context: fork, allowed-tools, argument-hint in SKILL.md frontmatter. (3.2)

Plan first or just do it?
  → Architectural / multi-approach / many files → PLAN mode (+ Explore subagent).
  → Small, clear, single-file → DIRECT execution.
  → Combine: plan the design, directly execute it. (3.4)

Code not coming out right?
  → Inconsistent transform → input/output examples; quality → test-first;
    unfamiliar domain → interview pattern; interacting bugs → one message. (3.5)

Running in CI?
  → -p (non-interactive) + --output-format json + --json-schema;
    CLAUDE.md for context; independent instance for review; feed prior findings/tests. (3.6)
```

### The five Domain-3 truths most likely to earn you points
1. **Scope is everything:** project (`.claude/...`, committed) is shared; user (`~/.claude/...`) is personal; "teammate didn't get it" = it's at user level.
2. **Skill frontmatter:** `context: fork` (isolate output), `allowed-tools` (restrict), `argument-hint` (prompt for args); skills are on-demand, CLAUDE.md is always-on.
3. **`.claude/rules/` + glob `paths:`** apply conventions by file type across directories — better than directory-bound CLAUDE.md for scattered files.
4. **Plan mode for architectural/multi-file/multi-approach work; direct execution for small clear changes;** `Explore` subagent protects context.
5. **CI: `-p` (non-interactive) + `--output-format json`/`--json-schema`;** independent review instance beats self-review; feed prior findings to avoid duplicate comments.
