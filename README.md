# openvibe

**A multi-agent [OpenCode](https://opencode.ai) setup that splits AI coding into separate roles — plan, review, build, drift-check, code-review, ship — each on a different model that checks the others' work.**

The problem with letting one model plan, write, and check its own code is that it grades its own homework. And it passes. Every time. openvibe breaks the work across *different model families* so that mistakes have a real shot at getting caught by something that doesn't share the author's blind spots.

Designed for a **product-minded operator**: you describe features in plain language, the pipeline handles every technical decision, and you make the final "is this what I wanted" call. You are not expected to read code to use this. You are expected to actually run the thing and click the button before shipping — no model can do that for you.

---

## The five roles

| Role | Agent | Job | Model | Why |
|------|-------|-----|-------|-----|
| **Architect** | `plan` | Reads the codebase, decides the architecture, writes `PLAN.md`. Cannot touch source code. | DeepSeek V4 Pro (Think Max) | Best open reasoner; verbosity doesn't matter for a few planning calls. |
| **Builder** | `build` | Implements the plan to the letter, runs tests, leaves changes uncommitted until you say ship. | DeepSeek V4 Pro (Think High) | Same model, lower reasoning effort — strong execution without burning Max tokens across a build loop. |
| **Plan Reviewer** | `@1-review-plan` | Independently critiques the *plan* for judgment flaws **before any code is written**. Reads the real codebase itself — does not trust the architect's description of it. | Kimi K2.6 | Different lab → decorrelated blind spots. Highest-intelligence model on Go. |
| **Drift Checker** | `@2-check-drift` | After the build: did the implementation match the plan, no more and no less? | GLM-5.1 | Third lab. Disciplined, structured output — exactly what a conformance report needs. |
| **Code Reviewer** | `@3-review-code` | After drift-check: bugs, security issues, anti-patterns in the actual code. Backed by Semgrep (5000+ deterministic rules). Does **not** re-litigate architecture. | Kimi K2.6 | Different lab from the builder; the shared model with `@1-review-plan` is acceptable because these two agents examine entirely different artifacts — a plan vs. a code diff — under fundamentally different prompts. |

**The single most important thing:** the three checkers are on different model families from the builder on purpose. A checker that shared the builder's training distribution would share its blind spots. Decorrelation is the whole game.

### What each stage actually catches

- **Plan Review** — judgment errors: a plan that would compile, pass its own tests, and solve the subtly wrong problem. Catches this *before* any build tokens are spent.
- **Drift Check** — conformance errors: omissions, deviations, and things built *in excess* of the spec. It's looking at new files, not just diffs — "excess" matters.
- **Code Review** — implementation quality: bugs, security holes, anti-patterns. Everything drift-check deliberately ignores.
- **Running it yourself** — whether it's actually what you wanted. This is the realest test and it's on you.

---

## The files

| File | What it is |
|------|------------|
| `opencode.jsonc` | Config. Five agents, their models, temperatures, permissions. |
| `architect.md` | System prompt for `plan`. |
| `build.md` | System prompt for `build`. |
| `1-review-plan.md` | System prompt for `@1-review-plan`. |
| `2-check-drift.md` | System prompt for `@2-check-drift`. |
| `3-review-code.md` | System prompt for `@3-review-code`. |
| `.gitignore` | Keeps `PLAN.md` and `pipeline-memory.md` out of commits. |
| `PLAN.md` | **Generated at runtime** by the architect. The handoff artifact. Not authored by you. |
| `pipeline-memory.md` | **Generated at runtime** by the builder — a running log of past builds, lessons, and operator preferences. Created on first use; never committed. |

All files live at the **repo root**, side by side. The config references prompts as `{file:./architect.md}` etc., resolved relative to `opencode.jsonc` — they must stay together. Project config at the repo root overrides global/remote config and is safe to commit, so the pipeline travels with the repo.

---

## How the work flows

```
You describe a task (plain language)
        │
        ▼
 ┌─────────────┐   reads repo + pipeline-memory.md,
 │  plan (Max) │   verifies API versions, writes PLAN.md,
 └─────────────┘   shows you a plain-English summary
        │
        ▼
   @1-review-plan .... (optional, for non-trivial tasks)
        │           reads PLAN.md + the actual code,
        │           returns SOUND or REVISE
        │           └─ if REVISE: architect fixes PLAN.md, re-review
        ▼
   Tab to build .... shared session: builder has the plan as live context
        │
        ▼
 ┌──────────────┐   reads PLAN.md + pipeline-memory.md,
 │ build (High) │   implements exactly that, runs tests,
 └──────────────┘   leaves changes UNCOMMITTED
        │
        ▼
   @2-check-drift ... reads PLAN.md + git diff + new files,
        │           returns MATCHES PLAN or DRIFT FOUND
        ▼
   @3-review-code ... reads git diff + changed files + Semgrep,
        │           returns PASS or ISSUES FOUND
        ▼
   You run it / click it / use it  ← the only check that counts
        │
        ▼
   Tell build "commit and push" → it shows you exactly what it's
        committing → writes a memory entry → commits → pushes
```

`plan` and `build` are **primary agents** — switch with Tab, shared session, no lossy re-paste. The three checkers are **subagents** — invoke with `@1-review-plan` / `@2-check-drift` / `@3-review-code` from whichever primary agent you're in. They run in isolated child sessions, which is *by design*: a reviewer that inherited the architect's reasoning would just agree with it. The checkers read `PLAN.md` from disk instead.

### Pipeline memory

`pipeline-memory.md` is the pipeline's persistent learning log. It accumulates across sessions so the architect and builder don't start cold.

- **Architect reads it before planning** — avoids repeating past mistakes, honors preferences already expressed. Disk state supersedes stale memory.
- **Builder reads it before building** — checks for past failures on similar files.
- **Builder writes it before shipping** — appends a dated entry after staging, before committing. If the write fails, escalation halts before anything is pushed.
- **No automatic cleanup.** At ~300 lines the builder suggests trimming. You manage it manually — read-modify-write on a plain markdown file with no git history is fragile enough without a model trying to summarize it.
- **Local only.** Never committed; doesn't travel across machines.

### When to use the full chain vs. not

Full chain for large or hard-to-reverse changes. For a one-line tweak, the honest minimum is: **plan → build → run it → commit**. The three checkers are insurance you add when the stakes justify extra passes. Don't summon a five-model review pipeline to change a button color.

---

## Key design decisions (don't undo these)

**The architect is read-only on source files.** A load-bearing constraint, not a limitation. Forces handoff rather than drift into implementation; keeps the expensive deep-reasoning seat from editing code it was only meant to think about. It can write exactly one file (`PLAN.md`) and run read-only git commands.

**Same model, two reasoning efforts.** Max for planning (a few calls, depth matters), High for building (a loop of many calls, Max would multiply latency and verbosity without commensurate gain).

**Plans hand off via a file on disk, not via conversation context.** OpenCode's multi-agent context inheritance is fuzzy and version-dependent. `PLAN.md` makes every handoff deterministic and inspectable. All three checkers are explicitly instructed to trust disk state over inherited context.

**The builder leaves changes uncommitted until you say ship.** It has access to `git commit`/`git push` but waits for your explicit "commit and push" or "ship it." Before acting, it prints the files, the commit message, and the target branch. It halts on secrets, detached HEAD, or anything genuinely anomalous. Destructive operations (force-push, rebase, reset) are permission-denied at the config level.

**Drift check and code review stay separate.** Tempting to merge them into one post-build pass — don't. Drift-found sends you back to the architect. Issues-found sends you back to the builder. They're different decision points. Merging them would conflate standards and muddy the signal.

**Version verification is mandatory.** The architect must never design against a third-party API from memory. It reads the pinned version from dependency files, then checks that version's docs via context7. This rule was added after a real failure where a model confidently planned against a deprecated API. Documenting the verified version in the plan is what actually enforces the check.

**Semgrep is the non-LLM guardrail.** `@3-review-code` runs every changed file through Semgrep's MCP — 5000+ deterministic rules, 35+ languages, fires the same way regardless of the LLM's current mood. Findings are cited as `(Semgrep: <rule-id>)`. Degrades gracefully if Semgrep isn't installed. One-time setup: `brew install semgrep`. Note: `SEMGREP_SEND_METRICS=off` breaks the default `auto` config — leave it unset.

---

## What's in PLAN.md

PLAN.md is the actual product of the pipeline — everything else exists to produce and check it. The architect writes a plain-English header (for you) plus a fenced `<build_specification>` (for the agents).

| Section | Purpose |
|---------|---------|
| IN PLAIN ENGLISH | What you'll get, jargon-free |
| THE BIG PICTURE | How it fits the system + the one tradeoff worth knowing |
| ASSUMPTIONS I MADE | Product choices the architect made for you — correct any that are wrong |
| CONTEXT I VERIFIED | The real files and patterns examined — evidence the plan isn't hallucinated |
| FILES TO TOUCH | path · what changes there |
| COMPONENTS | name · responsibility · verified external versions |
| SEQUENCING | Ordered build steps, when order matters |
| ALTERNATIVES CONSIDERED | What else was evaluated and why it lost (optional) |
| KEY DECISIONS | choice · one-line rationale |
| INTERFACES / CONTRACTS | Data shapes and boundaries the builder must honor |
| HOW TO VERIFY | The exact command(s) that prove it works |
| ACCEPTANCE CRITERIA | What "done" looks like |
| RISKS / WATCH-OUTS | The 1–2 places this is most likely to go wrong |
| BUILD ESCALATION | The task-specific halt condition so the builder can't loop forever |
| OUT OF SCOPE | What was deliberately not built |

**Right-sizing matters.** This is a menu, not a mandatory template. A one-line CSS tweak skips SEQUENCING, INTERFACES, and RISKS. A database migration leans on all of them. CONTEXT I VERIFIED, FILES TO TOUCH, and ACCEPTANCE CRITERIA are always present; the rest earn their place. No "RISKS: n/a" padding.

---

## Temperature and models

**Rule: use each vendor's own recommended temperature for thinking mode.** The intuition that "checking should be colder, so lower the temperature" is wrong for reasoning models — it can collapse the chain-of-thought and actively degrade output. Temperature follows the model, not the role.

| Agent | Model | Temp | Why |
|-------|-------|------|-----|
| plan | DeepSeek V4 Pro | 1.0 | DeepSeek's thinking-mode guidance |
| build | DeepSeek V4 Pro | 1.0 | same |
| @1-review-plan | Kimi K2.6 | 1.0 | Moonshot's recommendation for K2.6 thinking mode |
| @2-check-drift | GLM-5.1 | **0.6** | Z.AI's own GLM-5.1 docs — different vendor, different number |
| @3-review-code | Kimi K2.6 | 1.0 | same as @1-review-plan |

GLM-5.1 at 0.6 looks out of place next to the others. Don't "fix" it to match. It's simply what Z.AI recommends for that model in thinking mode — the drift-check's discipline comes from its prompt and output format, not from a cooler temperature. If you ever swap a model, look up *that* model's recommended thinking-mode temperature and use it; never carry the number over from the model it replaced.

**Provider setup:** `plan` and `build` run on **direct DeepSeek** (`deepseek/`). The three checkers run on **OpenCode Go** (`opencode-go/`) — flat subscription, dollar-denominated limits, zero-retention policy. DeepSeek V4 Pro is also available on Go as `opencode-go/deepseek-v4-pro`, so you could consolidate all five onto Go if you'd rather manage one auth and one bill.

> ⚠️ **Verify every model string** in the `/models` picker before trusting it. A wrong `provider/model` prefix means the agent silently won't load — and the mixed-provider setup is exactly where "looks right in the file, fails on load" hides. Two providers means two auths must both work.

---

## Platform notes (OpenCode ~1.15)

- **Per-agent reasoning effort may not always apply.** The GUI reasoning selector can "stick" across agent switches. If plan and build end up at the same effort, set it manually in the UI.
- **Bash permissions are prefix-matched.** Keep deny patterns broad — especially `git push --force*` and `git push * --force`.
- **Permission `deny` rules behave differently via the SDK** than in interactive desktop use. Interactive use honors them as configured.
- **Context inheritance across subagents is version-dependent.** Handoffs go through `PLAN.md` on disk rather than relying on inherited context — all three checker prompts are written around this.

---

## Quickstart

1. **Copy the files** (`opencode.jsonc` + the five `.md` prompts + `.gitignore`) into your project root, beside each other.
2. **Connect your providers.** Authenticate DeepSeek (direct) and OpenCode Go (`/connect`). Or consolidate everything onto Go — see the temperature section.
3. **Install Semgrep** once: `brew install semgrep`. Leave `SEMGREP_SEND_METRICS` unset. Skip this and `@3-review-code` falls back to LLM-only analysis with a note.
4. **Verify model strings** in `/models`. Fix any that don't resolve.
5. **Use it:**

| Action | How |
|--------|-----|
| Start a task | Describe it to `plan` (default agent) |
| Review the plan | `@1-review-plan` |
| Build it | Tab → `build`, tell it to implement |
| Check the build matches the plan | `@2-check-drift` |
| Check the code for bugs | `@3-review-code` |
| Confirm it's what you wanted | Run/click/use it yourself |
| Commit & push | Tell `build` "commit and push" |
| Plan got flagged | Tell `plan` to address the review; it rewrites `PLAN.md` |
| Builder got stuck | It hits the `BUILD ESCALATION` condition and reports back |