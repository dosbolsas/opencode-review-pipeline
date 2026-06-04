# opencode-review-pipeline

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

> **Status:** complete and assembled, but **not yet fully verified end-to-end**. Run
> the dry-run checklist (§9) on a throwaway repo before relying on it — especially the
> build agent's force-push test, since OpenCode's permission matching has proven flaky.
> This is a working setup to adopt deliberately, not a guarantee.

## This is a template

These files are meant to be **copied into your own project's repo root** (or your
global `~/.config/opencode/`), not run as a standalone app. The agents reference
their prompt files with relative paths, so the `.md` files and `opencode.jsonc` must
live together.

## Quickstart

1. **Copy the files** (`opencode.jsonc` + the five `.md` prompts) into your project
   root, beside each other.
2. **Connect your providers.** This setup uses DeepSeek (direct) for plan/build and
   OpenCode Go for the review/check/code-review agents — so authenticate both (`/connect`).
   You can also consolidate everything onto one provider; see §6.
3. **Verify the model strings** in OpenCode's `/models` picker and fix any that don't
   resolve — a wrong `provider/model` prefix makes an agent silently fail to load.
4. **Run the dry-run (§9)** on a throwaway repo to confirm permissions, reasoning
   effort, and the build agent's safety guards actually hold on your version.
5. **Use it:** describe a feature to `plan` → `@reviewer` → Tab to `build` →
   `@drift-check` → `@code-review` → run it yourself → tell `build` "commit and push"
   to ship. (Full walkthrough in §7.)

## How to read the rest of this doc

Everything below is the *why* and the *detail* — the design rationale, the model and
temperature choices, the day-to-day workflow, the known failure modes, and the
verification checklist. Skim §2 for the core idea; read §5–6 before changing models
or temperatures; treat §9 as a required checklist, not an optional one.

---

## 1. Who this is for

The operator is assumed to be a **product-minded person, not a working engineer**.
You describe what you want in plain, UX-flavored language ("add a button that shows
the user's recent orders"), and the pipeline turns that into a technical plan,
builds it, checks the result, and (when you say so) commits it. You are not expected
to read code to use this — but you **are** the final reviewer of whether the result
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
| **Reviewer** | `reviewer` | Independently critiques the *plan* for subtle judgment flaws, before any code is written. | GLM-5.1 | **Different lab** from the builder → decorrelated blind spots. Strong reasoner for a pure-judgment task. |
| **Drift-Check** | `drift-check` | After the build, checks whether the implementation matches the plan — including anything built *in excess* of the spec. | Kimi K2.6 | **Different lab** from the builder → decorrelation at the second checkpoint. Strong at multi-step tool loops (status → diff → read → reconcile), which is exactly this job. |
| **Code-Review** | `code-review` | After the build and drift-check, examines the actual code for bugs, security issues, anti-patterns, and correctness. Does NOT check conformance. | Kimi K2.6 | Different lab from the builder; catches implementation quality problems drift-check explicitly ignores. Strong at code inspection.

**The single most important principle:** the Reviewer, Drift-Check, and Code-Review
are on *different model families* from the Builder **on purpose**. If every checker
were DeepSeek, they'd share DeepSeek's blind spots and wave through exactly the
mistakes that matter. Decorrelation is the point.

### What each checker can and cannot catch
- The **Reviewer** catches *errors of judgment* — a plan that would compile and pass
  its own tests but solves the wrong problem, rests on a hidden assumption, or fights
  the existing codebase. It runs **before** building, so a bad plan is caught before
  any tokens are spent implementing it.
- **Drift-Check** catches *conformance errors* — omissions (spec'd but missing),
  deviations (built differently), and **excess** (built but not sanctioned by the
  plan, or in its OUT OF SCOPE list). Catching excess depends on it seeing *new*
  files, not just the diff — which is why its prompt makes it run `git status` and
  read new files' contents, not just `git diff`. It does **not** judge whether the
  plan was good — that's the Reviewer's job.
- **Code-Review** catches *implementation quality problems* — bugs, security
  vulnerabilities, anti-patterns, and correctness issues that drift-check explicitly
  ignores (drift-check only checks conformance, not quality). It runs after
  drift-check, so it only inspects code that has already passed the conformance bar.
- **Tests + actually running the app** catch *mechanical breakage*. This is the
  truest reviewer and costs nothing — see §7.
- **You** catch "is this what I actually wanted." No model can do this for you; it's
  why the plan always surfaces a plain-English summary you can sanity-check, and why
  your judgment sits *between* code-review and the commit.

---

## 3. The files

| File | What it is |
|------|------------|
| `opencode.jsonc` | The config. Defines the five agents, their models, temperatures, prompts, and permissions. |
| `architect.md` | System prompt for the `plan` agent. |
| `build.md` | System prompt for the `build` agent. |
| `reviewer.md` | System prompt for the `reviewer` subagent. |
| `drift-check.md` | System prompt for the `drift-check` subagent. |
| `code-review.md` | System prompt for the `code-review` subagent. |
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
   @reviewer ........ (optional, for non-trivial tasks)
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
   @drift-check ..... reads PLAN.md + git diff + new files,
        │            returns MATCHES PLAN or DRIFT FOUND
        ▼
   @code-review ..... reads git diff + changed files,
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
architect read as live context (no lossy re-paste). `reviewer`, `drift-check`, and
`code-review` are **subagents** — you invoke them by typing `@reviewer` /
`@drift-check` / `@code-review` from whichever primary agent you're in (you do not
Tab into them). They run in isolated child sessions, which is *desirable* for the
checkers: a reviewer that inherited the architect's reasoning would just agree with
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
build → run it and look → tell build to commit and push.** The reviewer, drift-check,
and code-review are insurance you add when the stakes justify the extra passes (each
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
  (so the reviewer has complete context) but shows you only the plain-English sections
  in chat. The chat is a readable summary; the file is the source of truth.
- **The builder leaves changes uncommitted, and commits only when you say to ship.**
  The builder has access to `git commit`/`git push`, but its prompt instructs it to
  leave everything uncommitted by default and only stage/commit/push when you
  explicitly say "commit and push" or "ship it." The shipping gate is now a
  prompt-level behavioral rule rather than a separate permission-denied agent —
  accepted because fewer hand-offs on OpenCode's flaky multi-agent substrate outweigh
  the weaker guard, and your explicit invocation is the real gate regardless of which
  agent acts on it. Destructive operations (force-push, rebase, reset, etc.) remain
  permission-denied.
- **The checkers stay SEPARATE from each other and from the builder.** Tempting to
  merge drift-check and code-review into one "post-build check" step — don't.
  Drift-check's value is *independent* conformance verification; code-review's value is
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

### What's in PLAN.md (the actual product)

PLAN.md is the heart of the pipeline — everything else exists to produce and check
it. The architect writes a plain-English header (for you) plus a fenced
`<build_specification>` (for the build/drift-check agents). The full section set:

| Section | Purpose | Audience |
|---------|---------|----------|
| IN PLAIN ENGLISH | what the user will get, jargon-free | operator |
| THE BIG PICTURE | how it fits the system + the one tradeoff worth knowing | operator |
| ASSUMPTIONS I MADE | product choices decided for you, so you can correct them | operator |
| CONTEXT I VERIFIED | the real files/patterns/interfaces examined — evidence the plan is grounded, auditable by the reviewer | spec |
| FILES TO TOUCH | path · what changes there | spec |
| COMPONENTS | name · responsibility · key tech (+ verified external versions) | spec |
| SEQUENCING | ordered build steps, when order matters | spec |
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
| reviewer | GLM-5.1 | `opencode-go/` | **0.6** | Z.AI's own GLM-5.1 thinking-mode docs |
| drift-check | Kimi K2.6 | `opencode-go/` | 1.0 | Moonshot's card — K2.6 thinking mode wants 1.0 |
| code-review | Kimi K2.6 | `opencode-go/` | 1.0 | same |

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
  the reviewer "should be more deterministic because it's a critic." The reviewer runs
  at 0.6 because *GLM* wants 0.6, full stop. (Its discipline against inventing fake
  flaws comes from its prompt, not from a cold temperature.)
- **Kimi K2.6 → 1.0 (both drift-check and code-review).** Moonshot's model card recommends 1.0
  for thinking mode and notes K2.6 effectively wants its default sampling — overriding
  it downward fights the model. Both Kimi seats run at 1.0.

The pattern to internalize: **temperature follows the model, not the role.** Two
agents doing very different jobs (drift-check vs. code-review) share 1.0 because they share a
model; two "thinking-hard" agents (reviewer at 0.6, plan at 1.0) differ because they
are different models. If you ever swap a model, look up *that* model's recommended
thinking-mode temperature and use it — never carry over the number from the model it
replaced. There's no reasoning-effort knob exposed for the Kimi seats, only temperature.

**Provider note (mixed setup):** `plan`/`build` run on **direct DeepSeek**
(`deepseek/`), while `reviewer`/`drift-check`/`code-review` run on **OpenCode Go**
(`opencode-go/`) — a flat subscription with dollar-denominated usage limits,
US/EU/Singapore hosting, and a zero-retention policy (your code is not used for
training). DeepSeek V4 Pro is also on Go as `opencode-go/deepseek-v4-pro` — so you
*could* consolidate all five agents onto Go (one auth, one bill) if you'd rather not
run two providers. The high-volume DeepSeek seats are kept direct by choice.

**Endpoint gotcha:** on Go, the endpoint format varies by model (some use the
Anthropic-format `/v1/messages`, some the OpenAI-format `/chat/completions`), and
Anthropic-format endpoints have a history of scaling temperature. If a Kimi seat's
behavior looks off, confirm its effective temperature is really 1.0 (see §9).

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
4. *(Optional, for non-trivial work)* **Type `@reviewer`.** If it says REVISE, ask the
   architect to address the flaws; it will rewrite `PLAN.md`.
5. **Press Tab to switch to `build`.** Tell it to implement. It reads `PLAN.md` and
   `pipeline-memory.md` (for past lessons), builds exactly that, runs tests as it
   goes, and leaves the changes uncommitted.
6. *(Optional)* **Type `@drift-check`** to confirm the build matches the plan and that
   nothing was built in excess.
7. *(Optional)* **Type `@code-review`** to check the actual code for bugs, security
   issues, and anti-patterns. This catches quality problems drift-check won't see.
8. **Run the thing yourself.** Open the app, click the button, use the feature. This
   is the most important check and the one only you can do — a clean drift-check and
   code-review mean "the build followed the plan and the code looks sound," NOT "this
   is what you wanted."
9. **Tell the build agent "commit and push" when you're ready to ship.** It prints
   the files, commit message, and target branch, writes a memory entry to
   `pipeline-memory.md`, then commits and pushes to the current branch. It halts on
   secrets, detached HEAD, or anything genuinely anomalous.

---

## 8. Known caveats & where this can silently fail

This pipeline sits on top of OpenCode's multi-agent layer, which (as of ~1.15) has
several **version-dependent quirks**. They fail *silently*, which is the dangerous
kind.

- **Per-agent reasoning effort may not apply.** `reasoningEffort` is parsed but hasn't
  always been enforced per-agent, and the GUI reasoning selector can override config
  and "stick" across agent switches. If Plan and Build end up at the same effort, this
  is why. Fallback: set effort manually in the UI. The UI may also show "Default" even
  when a configured effort is active — don't trust that label.
- **Bash permission patterns are prefix-matched and proved UNRELIABLE in practice.**
  In a real run, a `git branch` command was denied even though `git branch*` was in
  the allow list. The matching is flaky. This matters most for the **build agent's
  force-push deny**: do not assume `git push --force*` actually blocks a force-push
  until you've tested it (§9). A force-push is the one git action that can lose work
  irreversibly, and it's guarded by the layer we've watched misbehave.
- **Permission `deny` rules have been ignored in some contexts** (notably via the SDK).
  Interactive desktop use should honor them, but test it.
- **Context inheritance for subagents is fuzzy.** This is why hand-offs go through
  `PLAN.md` on disk rather than relying on what a subagent "sees" from the parent.
  All three checker subagents (reviewer, drift-check, code-review) carry explicit
  instructions to trust disk state over any inherited context that conflicts with it.

**Bigger-picture caveat:** this is a sophisticated pipeline (five agents, two
providers, flaky reasoning-effort fields). For a small
codebase and product-driven work, there's a real question of whether it's *more
machinery than the job needs* — every agent is another hand-off on a flaky substrate
you can't easily debug. If a single strong coding agent reading the repo directly
ships your features faster and with fewer mystery failures, that's a legitimate and
possibly better choice. Measure it (§9, last step) rather than assuming more structure
is better.

---

## 9. Verifying it actually works (do this once, on a throwaway repo)

Test silent-failure points first, in order. Use a sacrificial git repo — and for
the build agent's push tests, a **throwaway GitHub remote**, not anything real.

**Stage 0 — Sandbox.** New folder, `git init`, a trivial starter project, the
six files at root, one initial commit (gives drift-check/code-review a clean `HEAD`).
Point it at a throwaway GitHub repo so push tests are safe.

**Stage 1 — Config loads, models resolve.** Restart OpenCode. Confirm: you land in
`plan`; all five agents appear; each is on the model you assigned (ask "which model
are you?" or check the indicator). A missing/fallback agent = wrong prefix. The
mixed-provider setup means check that BOTH the DeepSeek-direct and the Go agents load.

**Stage 2 — Permissions hold (the silent-danger tests).**
- Ask `plan` to edit a source file → must **refuse**.
- Ask `plan` to write `PLAN.md` → must **succeed**.
- Ask `plan` to run `git push` → must **refuse**.
- In `build`, ask it to implement a feature (shows git commit/push are allowed).
- **Critical:** ask the `build` agent to attempt `git push --force origin <branch>` AND
  `git push origin <branch> --force` → both must **refuse**. Given the proven
  flakiness of pattern-matching, do not skip this. If the trailing-flag form slips
  through, broaden the deny pattern before trusting the agent unattended.

**Stage 3 — Reasoning effort applies.** Compare a Plan call vs. a Build call in your
provider's token dashboard (Plan should show more thinking tokens). If identical, the
per-agent effort isn't applying — fall back to the manual UI toggle.

**Stage 4 — One real end-to-end pass** on a trivial feature ("add a /health endpoint
returning OK"):
- `plan` actually reads the repo (watch for tool calls) before planning, writes
  `PLAN.md`, shows the plain-English summary. Open `PLAN.md` and confirm it has the
  `<build_specification>` block with a real CONTEXT I VERIFIED line (not a thin/empty
  one). For a trivial feature it should *omit* the sections that don't apply
  (SEQUENCING, RISKS, etc.) rather than pad them — that's right-sizing working. Try a
  more involved task too and confirm SEQUENCING and HOW TO VERIFY then appear.
- `@reviewer` reads `PLAN.md` + code, returns SOUND/REVISE.
- Tab to `build`: confirm it follows the SEQUENCING order (if present) and runs the
  HOW TO VERIFY command(s), implements, and leaves changes **uncommitted** (verify
  with `git status`).
- `@drift-check`: confirm its git commands actually execute and it returns a verdict.
  Bonus: have the builder add a stray file the plan didn't mention, and confirm
  drift-check flags it under OUT-OF-SCOPE (tests the excess-detection path).
- `@code-review`: confirm it reads the diff and returns a code quality verdict.
  Bonus: have the builder introduce a minor bug (e.g. off-by-one), and confirm
  code-review flags it.
- Tell the `build` agent "commit and push": confirm it prints the file
  list/message/branch before acting, then commits and pushes correctly. Verify
  `PLAN.md` stays in the working directory (not staged, not committed, not deleted).
  Also verify: `pipeline-memory.md` was created (or appended to) with a properly
  formatted memory entry containing all five fields (Built, Key insight, Verified,
  Failures & lessons, Operator notes). Confirm `pipeline-memory.md` is NOT staged
  and NOT in the commit (`git show HEAD --stat` shows only source files).
- **Memory read-back:** start a second session and describe a trivial task to `plan`.
  Confirm its CONTEXT I VERIFIED line mentions `pipeline-memory.md` and the plan
  reflects knowledge from a previous build (e.g., references a past operator
  preference or avoids a known-fragile subsystem).
- **Memory fails-before-ship:** have the builder attempt a build where the memory
  write is deliberately broken (e.g., `pipeline-memory.md` is a directory). Confirm
  the builder stops without committing or pushing — the escalation halts before
  anything ships.
- **Memory size notification:** manually pad `pipeline-memory.md` to >300 lines,
  run a build, confirm the new entry contains the "consider trimming" note and no
  automatic condense or truncation occurred.

**Stage 5 — Kimi endpoint check.** Confirm the Kimi seats' effective temperature
isn't being scaled by the endpoint format, and that drift-check's verdict isn't
truncated.

**Final judgment.** Count the manual steps and time for that trivial task vs. just
asking one coding agent to do the same. That comparison — not any further config
tweak — tells you whether this pipeline earns its complexity for *your* work.

---

## 10. Quick reference

| Action | How |
|--------|-----|
| Start a task | Describe it to `plan` (default agent) |
| Plan an architecture | automatic — `plan` writes `PLAN.md` |
| Review the plan | type `@reviewer` |
| Build it | press **Tab** → `build`, tell it to implement |
| Check the build matches the plan | type `@drift-check` |
| Check the code for bugs | type `@code-review` |
| Confirm it's what you wanted | run/click/use it yourself |
| Commit & push | tell `build` "commit and push" (it shows what it does before acting) |
| Plan got flagged | tell `plan` to address the review; it rewrites `PLAN.md` |
| Builder got stuck | it honors the `BUILD ESCALATION` halt condition and reports back |

---

## 11. Status

As of this writing the pipeline is **v2 — code-review agent added, git agent
removed, PLAN.md persists, context inheritance hardened.** Not yet fully verified
end-to-end. Treat §9 as a required checklist, not an optional one, before trusting
this on real code — especially the force-push test on the build agent, since the
guard protecting your remote runs on the layer that has already misbehaved.
