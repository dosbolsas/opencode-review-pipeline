# openvibe

**A multi-agent [OpenCode](https://opencode.ai) setup that splits AI coding into
separate roles — plan, review, build, drift-check, code-review, ship — each on a
different model that checks the others' work.**

Most AI-coding setups use one model to plan, write, and check its own work, so it
grades its own homework and passes. This pipeline separates those roles across
*different model families* on purpose: the model that reviews the plan, the model
that checks the build, and the model that reviews the code are not the same lab as the
model that wrote the code, so mistakes have a real chance of being caught by a fresh
perspective.

It's designed for a **product-minded operator** who describes features in plain
language and isn't expected to read code — the architect owns all technical
decisions; you own product intent and the final "is this what I wanted" call.

## This is a template

These files are meant to be **copied into your own project's repo root** (or your
global `~/.config/opencode/`), not run as a standalone app. The agents reference
their prompt files with relative paths, so the `.md` files and `opencode.jsonc` must
live together.

## Quickstart

1. **Copy the files** (`opencode.jsonc` + the five `.md` prompts + `.gitignore`) into your project
   root, beside each other.
2. **Connect your providers.** This setup uses DeepSeek (direct) for plan/build and
   OpenCode Go for the review/check/code-review agents — so authenticate both (`/connect`).
   You can also consolidate everything onto one provider; see §6.
3. **Install Semgrep** for deterministic security scanning during code review:
   `brew install semgrep`. One-time; the 3-review-code agent uses it automatically.
   Ensure `SEMGREP_SEND_METRICS` is not set to `off` (the default `auto` config
   requires anonymous metrics; see §5). If you skip this step, 3-review-code falls back
   to LLM-only security analysis with a note.
4. **Verify the model strings** in OpenCode's `/models` picker and fix any that don't
   resolve — a wrong `provider/model` prefix means the agent won't load.
5. **Use it:** describe a feature to `plan` → `@1-review-plan` → Tab to `build` →
   `@2-check-drift` → `@3-review-code` → run it yourself → tell `build` "commit and push"
   to ship. (Full walkthrough in §7.)

## How to read the rest of this doc

Everything below is the *why* and the *detail* — the design rationale, the model and
temperature choices, and the day-to-day workflow. Skim §2 for the core idea; read §5–6 before changing models or temperatures.

---

## 1. Who this is for

The operator is assumed to be a **product-minded person, not a working engineer**.
You describe what you want in plain, UX-flavored language ("add a button that shows
the user's recent orders"), and the pipeline turns that into a technical plan,
builds it, checks the result, and (when you say so) commits it. You are not expected
to read code to use this — but you **are** the final 1-review-plan of whether the result
is what you actually wanted (see §7).

Everything below is tuned around that assumption. If you are an engineer feeding
precise specs, some choices (especially the "never ask the operator a technical
question" rule) would be tuned differently.

---

## 2. The core idea — and the one thing to understand

Most "AI coding agent" setups use one model to do everything: plan, write, and check
its own work. The problem is that a model checking its own work shares its own blind
spots — it grades its own homework and passes. This pipeline breaks the work into
**five roles with separation between them**, so that mistakes have a chance of being
caught by a different perspective:

| Role | Agent | Job | Model | Why this model |
|------|-------|-----|-------|----------------|
| **Architect** | `plan` | Reads the codebase, decides the architecture, writes the plan. Touches no source code. | DeepSeek V4 Pro (Think Max) | Strongest open reasoner; deep thinking is the whole value here, and verbosity doesn't matter for a few planning calls. |
| **Builder** | `build` | Implements the plan to the letter. Runs tests. Leaves changes uncommitted; commits only when told to ship. | DeepSeek V4 Pro (Think High) | Same model, lower reasoning effort — strong execution without the full Max token burn across a build loop. |
| **Plan Review** | `1-review-plan` | Step 1. Independently critiques the *plan* for subtle judgment flaws, before any code is written. | Kimi K2.6 | **Different lab** from the builder → decorrelated blind spots. Highest‑intelligence model on Go (AAII 53.9) — the hardest judgment task gets the strongest checker. |
| **Drift Check** | `2-check-drift` | Step 2. After the build, checks whether the implementation matches the plan — including anything built *in excess* of the spec. | GLM-5.1 | **Different lab** from the builder AND from both other checkers (Kimi K2.6 on plan‑review and code‑review). GLM‑5.1 (AAII 51.4) is a strong reasoner; at 0.6 temperature per Z.AI's docs, its output is disciplined and structured — exactly what drift‑check's rigid conformance‑report format demands. |
| **Code Review** | `3-review-code` | Step 3. After the build and drift-check, examines the actual code for bugs, security issues, anti-patterns, and correctness. Uses Semgrep for deterministic SAST behind its LLM judgment. Does NOT check conformance. | Kimi K2.6 | Different lab from the builder; catches implementation quality problems `2-check-drift` explicitly ignores. Semgrep adds a non-LLM guardrail — 5000+ rules, 35+ languages. |

**The single most important principle:** the Plan Review, Drift Check, and Code Review
are on *different model families* from the Builder **on purpose**. If every checker
were DeepSeek, they'd share DeepSeek's blind spots and wave through exactly the
mistakes that matter. Decorrelation is the point. Plan‑review and code‑review share Kimi K2.6, but they inspect different artifact types (PLAN.md vs. code diffs) under fundamentally different prompts — the shared model is acceptable because the two jobs exercise distinct reasoning domains. Drift‑check (GLM‑5.1) is the sole Z.AI seat, positioned between the two Kimi instances — three distinct model‑assignment patterns checking the same builder.

### What each checker can and cannot catch
- The **Plan Review** catches *errors of judgment* — a plan that would compile and pass
  its own tests but solves the wrong problem, rests on a hidden assumption, or fights
  the existing codebase. It runs **before** building, so a bad plan is caught before
  any tokens are spent implementing it.
- **Drift Check** catches *conformance errors* — omissions (spec'd but missing),
  deviations (built differently), and **excess** (built but not sanctioned by the
  plan, or in its OUT OF SCOPE list). Catching excess depends on it seeing *new*
  files, not just the diff — which is why its prompt makes it run `git status` and
  read new files' contents, not just `git diff`. It does **not** judge whether the
  plan was good — that's the plan review's job.
- **Code Review** catches *implementation quality problems* — bugs, security
  vulnerabilities, anti-patterns, and correctness issues that 2-check-drift explicitly
  ignores (drift-check only checks conformance, not quality). It runs after
  drift-check, so it only inspects code that has already passed the conformance bar.
  Security findings are backed by **Semgrep** — 5000+ deterministic rules across 35+
  languages — cited alongside the LLM's own judgment. Semgrep runs locally with no
  API key (just `brew install semgrep` once).
- **Tests + actually running the app** catch *mechanical breakage*. This is the
  truest 1-review-plan and costs nothing — see §7.
- **You** catch "is this what I actually wanted." No model can do this for you; it's
  why the plan always surfaces a plain-English summary you can sanity-check, and why
  your judgment sits *between* 3-review-code and the commit.

---

## 3. The files

| File | What it is |
|------|------------|
| `opencode.jsonc` | The config. Defines the five agents, their models, temperatures, prompts, and permissions. |
| `architect.md` | System prompt for the `plan` agent. |
| `build.md` | System prompt for the `build` agent. |
| `1-review-plan.md` | System prompt for the `1-review-plan` subagent. |
| `2-check-drift.md` | System prompt for the `2-check-drift` subagent. |
| `3-review-code.md` | System prompt for the `3-review-code` subagent. |
| `.gitignore` | Git ignore rules — keeps `PLAN.md` and `pipeline-memory.md` out of commits. |
| `PLAN.md` | **Generated at runtime** by the architect — the hand-off artifact. Not authored by you; persists in the working directory across tasks. |
| `pipeline-memory.md` | **Generated at runtime** by the builder — a chronological log of past builds, lessons, and operator preferences. Created on first use; never committed. Read by the architect before planning, read and written by the builder before shipping. |

All files live at the **repo root**, side by side. The config references the prompts
with `{file:./architect.md}` etc., resolved relative to `opencode.jsonc`'s location —
so they must stay together. Project config at the repo root overrides global/remote
config and is safe to commit to git, so the whole pipeline travels with the repo.

---

## 4. How the work flows

```
You describe a task (plain language)
        │
        ▼
 ┌──────────────┐   reads repo, reads pipeline-memory.md,
 │  plan (Max)  │   verifies API versions, writes PLAN.md,
 └──────────────┘   shows you a plain-English summary
        │
        ▼
   @1-review-plan ........ (optional, for non-trivial tasks)
        │            reads PLAN.md + the code itself,
        │            returns SOUND or REVISE
        │            └─ if REVISE: architect fixes PLAN.md, re-review
        ▼
   Tab to build ..... shared session: builder inherits the plan as live context
        │
        ▼
 ┌──────────────┐   reads PLAN.md, reads pipeline-memory.md,
 │ build (High) │   implements exactly that, runs tests,
 └──────────────┘   leaves changes UNCOMMITTED
        │
        ▼
   @2-check-drift ..... reads PLAN.md + git diff + new files,
        │            returns MATCHES PLAN or DRIFT FOUND
        ▼
   @3-review-code ..... reads git diff + changed files,
        │            returns PASS or ISSUES FOUND
        ▼
   You run it / click it / look at it  ← the real final check
        │
        ▼
   Tell build ......... you say "commit and push"; build shows what
   "commit and push"   it's committing → writes memory entry →
                        commits & pushes (no separate confirmation)
```

`plan` and `build` are **primary agents** — you switch between them with the **Tab**
key, and they share one session, so the builder sees the plan and everything the
architect read as live context (no lossy re-paste). `1-review-plan`, `2-check-drift`, and
`3-review-code` are **subagents** — you invoke them by typing `@1-review-plan` /
`@2-check-drift` / `@3-review-code` from whichever primary agent you're in (you do not
Tab into them). They run in isolated child sessions, which is *desirable* for the
checkers: a 1-review-plan that inherited the architect's reasoning would just agree with
it. The checkers read `PLAN.md` from disk instead, which is why the architect always
writes the full plan to that file.

### How pipeline memory works

`pipeline-memory.md` is the pipeline's persistent learning log — it accumulates
across sessions so the architect and builder don't start cold every time. It's
created automatically by the builder on first use and never committed.

- **Architect reads it before planning** — uses it as context to avoid repeating past
  mistakes and to honor operator preferences already expressed. (Treats it as
  context, not authority — the actual codebase on disk supersedes stale memory.)
- **Builder reads it before building** — checks for past failures on similar files
  and operator preferences.
- **Builder writes it before shipping** — after staging files but before committing,
  the builder appends a dated entry recording what was built, key insights, and
  lessons. (This ordering means if the memory write fails, the build escalation can
  halt before anything is pushed.)
- **No automatic cleanup** — if the file exceeds ~300 lines, the builder adds a note
  suggesting the operator trim old entries. The operator manages memory manually;
  automatic condense or rewrite is deliberately avoided (read-modify-write on a plain
  markdown file with no git history is fragile).
- **Local state only** — like PLAN.md's operational content, memory is never
  committed and doesn't travel across machines. Switching machines means starting
  fresh.

### When to use the full chain vs. a short version
The full chain is for **large or hard-to-reverse changes** where a wrong plan is
expensive. For a small change, the honest minimum that protects you is: **plan →
build → run it and look → tell build to commit and push.** The @1-review-plan, @2-check-drift,
and @3-review-code are insurance you add when the stakes justify the extra passes (each
is a real cost). Don't fire everything on a one-line tweak.

---

## 5. Key design decisions (the "why", so you don't undo them later)

- **The architect is read-only except for `PLAN.md`.** This is a load-bearing guard,
  not a limitation. It forces the architect to *hand off* rather than drift into
  implementation, and prevents the expensive deep-reasoning model from editing source
  in the seat meant only for thinking. Scoped so it can write exactly one file
  (`PLAN.md`) and run read-only git commands.
- **Same model, two reasoning efforts (Max for plan, High for build).** Deep thinking
  earns its cost on the few planning calls; the build loop runs many calls, so it
  uses a lower effort to avoid multiplying verbosity and latency.
- **Plans are handed off via a file on disk, not via conversation context.** OpenCode's
  multi-agent context-inheritance is fuzzy and version-dependent (see §8). Writing
  `PLAN.md` makes every hand-off deterministic and inspectable.
- **The "quarantine" output split.** The architect writes the *full* plan to `PLAN.md`
  (so the 1-review-plan has complete context) but shows you only the plain-English sections
  in chat. The chat is a readable summary; the file is the source of truth.
- **The builder leaves changes uncommitted, and commits only when you say to ship.**
  The builder has access to `git commit`/`git push`, but its prompt instructs it to
  leave everything uncommitted by default and only stage/commit/push when you
  explicitly say "commit and push" or "ship it." The shipping gate is now a
  prompt-level behavioral rule rather than a separate permission-denied agent —
  accepted because fewer hand-offs on OpenCode's multi-agent layer outweigh
  the weaker guard, and your explicit invocation is the real gate regardless of which
  agent acts on it. Destructive operations (force-push, rebase, reset, etc.) remain
  permission-denied.
- **The checkers stay SEPARATE from each other and from the builder.** Tempting to
  merge 2-check-drift and 3-review-code into one "post-build check" step — don't.
  Drift check's value is *independent* conformance verification; 3-review-code's value is
  *independent* quality inspection on top of that. An agent doing both would have
  conflated standards. They also serve different operator decision points: drift-found
  sends you back to plan; issues-found sends you back to build. Your judgment belongs
  *between* them.
- **Version verification.** The architect must never design around a third-party API
  from memory — it reads the pinned version in the dependency files, then web-searches
  that specific version's docs. Added after a real failure where a model confidently
  planned against an outdated API. Recording the verified version in the plan is what
  makes it actually do this.
- **Persistent pipeline memory.** The pipeline accumulates learnings in
  `pipeline-memory.md` — a chronological log of past builds, operator preferences,
  and failures, created automatically by the builder on first use and never committed.
  The architect reads it before planning; the builder reads it before building and
  writes to it before shipping (after staging, before committing — so a write failure
  halts before anything is pushed). No plugins, no structured sections, no automatic
  cleanup — just a markdown file with dated entries. The architect's discoveries reach
  memory through an optional MEMORY TO PERSIST line in PLAN.md; the builder transcribes
  it verbatim.
- **Deterministic SAST via Semgrep.** The 3-review-code agent runs each changed file
  through Semgrep's MCP server (`semgrep_scan`) — 5000+ rules across 35+ languages
  that fire the same way every time, regardless of the LLM's current state. This is
  the most heterogeneous checker in the pipeline: it's not an LLM at all. Findings are
  folded into the existing CODE REVIEW output format with `(Semgrep: <rule-id>)`
  citations. Gracefully degrades if Semgrep isn't installed — the agent notes the
  absence and proceeds with pure LLM analysis. One-time prerequisite:
  `brew install semgrep`. Ensure `SEMGREP_SEND_METRICS` is not set to `off` (the
  default `auto` config requires anonymous metrics — a Semgrep requirement, not a
  pipeline design choice).

### What's in PLAN.md (the actual product)

PLAN.md is the heart of the pipeline — everything else exists to produce and check
it. The architect writes a plain-English header (for you) plus a fenced
`<build_specification>` (for the build/drift-check agents). The full section set:

| Section | Purpose | Audience |
|---------|---------|----------|
| IN PLAIN ENGLISH | what the user will get, jargon-free | operator |
| THE BIG PICTURE | how it fits the system + the one tradeoff worth knowing | operator |
| ASSUMPTIONS I MADE | product choices decided for you, so you can correct them | operator |
| CONTEXT I VERIFIED | the real files/patterns/interfaces examined — evidence the plan is grounded, auditable by the 1-review-plan | spec |
| FILES TO TOUCH | path · what changes there | spec |
| COMPONENTS | name · responsibility · key tech (+ verified external versions) | spec |
| SEQUENCING | ordered build steps, when order matters | spec |
| ALTERNATIVES CONSIDERED | what else was evaluated and why it was rejected — makes decision-making auditable (optional) | spec |
| KEY DECISIONS | choice · one-line rationale | spec |
| INTERFACES / CONTRACTS | data shapes, APIs, boundaries the builder must honor | spec |
| HOW TO VERIFY | the concrete command(s) that prove it works | spec |
| ACCEPTANCE CRITERIA | what "done" looks like | spec |
| RISKS / WATCH-OUTS | the 1-2 places this change is most likely to go wrong | spec |
| BUILD ESCALATION | the task-specific halt condition so the builder can't loop forever | spec |
| OUT OF SCOPE | what was deliberately not built | spec |

**Right-sizing is the key principle.** This is *not* a rigid template the architect
fills out every time — it's a menu. The architect includes the sections that apply to
the task and omits the rest: a one-line CSS tweak skips SEQUENCING, INTERFACES, and
RISKS; a database migration leans on all of them. Only CONTEXT I VERIFIED, FILES TO
TOUCH, and ACCEPTANCE CRITERIA are effectively always present. No empty headers, no
"RISKS: n/a" padding. The plan's weight matches the change's weight. (A fixed template
would be simpler to parse but would either pad trivial changes or constrain complex
ones — right-sizing trades a little consistency for plans that fit the real task.)

---

## 6. Temperature, models & providers — and why each is what it is

**Rule: use each vendor's own recommended temperature for thinking mode. Do not pick
a number based on intuition about how "deterministic" a role should feel.** That
intuition ("checking should be cold, so lower the temperature") is wrong for
reasoning models — low temperature can degrade their reasoning trace.

| Agent | Model | Provider prefix | Temp | Why this number |
|-------|-------|-----------------|------|-----------------|
| plan | DeepSeek V4 Pro | `deepseek/` (direct) | 1.0 | DeepSeek's thinking-mode guidance |
| build | DeepSeek V4 Pro | `deepseek/` (direct) | 1.0 | same |
|1-review-plan| Kimi K2.6 | `opencode-go/` | 1.0 | Moonshot's model card — K2.6 thinking mode wants 1.0 |
|2-check-drift| GLM-5.1 | `opencode-go/` | **0.6** | Z.AI's own GLM-5.1 thinking-mode docs |
|3-review-code| Kimi K2.6 | `opencode-go/` | 1.0 | same as 1-review-plan — K2.6 thinking mode wants 1.0 |

**Why the temperatures differ — and why you should not "tidy" them to match.** These
are not arbitrary and not a style choice. Each is the value the *model's own makers*
recommend for that model in thinking mode, because temperature interacts with how each
model's reasoning trace was trained:

- **DeepSeek V4 Pro → 1.0.** DeepSeek's guidance is that lowering temperature in
  thinking mode collapses the chain-of-thought and *degrades* answer quality; you
  control output length with other levers, not by cooling the temperature. So both
  DeepSeek seats run hot at 1.0 even though one is "planning" and one is "building."
- **GLM-5.1 → 0.6.** This is the one that looks odd next to the others and the one
  people most want to "fix" to match. Don't. Z.AI's own GLM-5.1 docs specify 0.6 for
  thinking mode — it's simply a different vendor's recommended number, not a sign that
  the drift-check "should be more deterministic." The drift-check runs
  at 0.6 because *GLM* wants 0.6, full stop. (Its discipline against inventing fake
  drift comes from its prompt and structured output format, not from a cold temperature.)
- **Kimi K2.6 → 1.0 (both 1-review-plan and 3-review-code).** Moonshot's model card recommends 1.0
  for thinking mode and notes K2.6 effectively wants its default sampling — overriding
  it downward fights the model. Both Kimi seats run at 1.0.

The pattern to internalize: **temperature follows the model, not the role.** Two
agents doing very different jobs (1-review-plan vs. 3-review-code) share 1.0 because they share a
model (Kimi K2.6); the drift-check runs at 0.6 because it uses a different model (GLM-5.1).
If you ever swap a model, look up *that* model's recommended
thinking-mode temperature and use it — never carry over the number from the model it
replaced. There's no reasoning-effort knob exposed for the Kimi seats, only temperature.

**Provider note (mixed setup):** `plan`/`build` run on **direct DeepSeek**
(`deepseek/`), while `1-review-plan`/`2-check-drift`/`3-review-code` run on **OpenCode Go**
(`opencode-go/`) — a flat subscription with dollar-denominated usage limits,
US/EU/Singapore hosting, and a zero-retention policy (your code is not used for
training). DeepSeek V4 Pro is also on Go as `opencode-go/deepseek-v4-pro` — so you
*could* consolidate all five agents onto Go (one auth, one bill) if you'd rather not
run two providers. The high-volume DeepSeek seats are kept direct by choice.

**Endpoint gotcha:** on Go, the endpoint format varies by model (some use the
Anthropic-format `/v1/messages`, some the OpenAI-format `/chat/completions`), and
Anthropic-format endpoints have a history of scaling temperature. If a Kimi seat's
behavior looks off (now on 1-review-plan and 3-review-code), confirm its effective temperature is really 1.0.

> ⚠️ **Verify every model string** in the `/models` picker before trusting it
> (`deepseek/deepseek-v4-pro`, `opencode-go/glm-5.1`, `opencode-go/kimi-k2.6`). A wrong
> prefix means the agent silently won't load — and the mixed-provider setup is exactly
> where "looks right in the file, fails on load" hides. The mixed setup also means
> **two auths must both work**.

---

## 7. How to use it (day to day)

1. **Open your project in OpenCode.** You'll land in the `plan` agent by default.
2. **Describe your task in plain language.** Loose is fine — the architect fills in
   the technical detail and will only ask you a question if it's a genuine *product*
   fork (e.g. "should logged-out users see the feed or a sign-up screen?"), never a
   technical one.
3. **Read the plain-English summary** the architect shows you. Does the described
   outcome match what you wanted? It also writes the full plan to `PLAN.md`.
4. *(Optional, for non-trivial work)* **Type `@1-review-plan`.** If it says REVISE, ask the
   architect to address the flaws; it will rewrite `PLAN.md`.
5. **Press Tab to switch to `build`.** Tell it to implement. It reads `PLAN.md` and
   `pipeline-memory.md` (for past lessons), builds exactly that, runs tests as it
   goes, and leaves the changes uncommitted.
6. *(Optional)* **Type `@2-check-drift`** to confirm the build matches the plan and that
   nothing was built in excess.
7. *(Optional)* **Type `@3-review-code`** to check the actual code for bugs, security
   issues, and anti-patterns. This catches quality problems 2-check-drift won't see.
8. **Run the thing yourself.** Open the app, click the button, use the feature. This
   is the most important check and the one only you can do — a clean 2-check-drift and
  3-review-code mean "the build followed the plan and the code looks sound," NOT "this
   is what you wanted."
9. **Tell the build agent "commit and push" when you're ready to ship.** It prints
   the files, commit message, and target branch, writes a memory entry to
   `pipeline-memory.md`, then commits and pushes to the current branch. It halts on
   secrets, detached HEAD, or anything genuinely anomalous.

---

## 8. Platform notes

OpenCode's multi-agent layer (as of ~1.15) has a few behaviors worth knowing:

- **Per-agent reasoning effort may not always apply.** `reasoningEffort` can be overridden by the GUI reasoning selector, which can "stick" across agent switches. If Plan and Build end up at the same effort, set it manually in the UI.
- **Bash permissions are prefix-matched.** Keep deny patterns broad — especially `git push --force*` and `git push * --force`, which protect your remote from the one git action that can lose work irreversibly.
- **Permission `deny` rules behave differently via the SDK** than in interactive desktop use. Interactive use honors them as configured.
- **Context inheritance across subagents is version-dependent.** Hand-offs go through `PLAN.md` on disk rather than relying on inherited context. All three checker subagents (@1-review-plan, @2-check-drift, @3-review-code) are instructed to trust disk state over inherited context.

---
## 9. Quick reference

| Action | How |
|--------|-----|
| Start a task | Describe it to `plan` (default agent) |
| Plan an architecture | automatic — `plan` writes `PLAN.md` |
| Review the plan | type `@1-review-plan` |
| Build it | press **Tab** → `build`, tell it to implement |
| Check the build matches the plan | type `@2-check-drift` |
| Check the code for bugs | type `@3-review-code` |
| Confirm it's what you wanted | run/click/use it yourself |
| Commit & push | tell `build` "commit and push" (it shows what it does before acting) |
| Plan got flagged | tell `plan` to address the review; it rewrites `PLAN.md` |
| Builder got stuck | it honors the `BUILD ESCALATION` halt condition and reports back |


