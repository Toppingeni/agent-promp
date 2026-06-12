# Agent Install — bootstrap Claude Code AI collaboration setup

> Paste this entire file into Claude Code in a fresh project. Claude will inspect the project, ask a few questions, then generate a `.claude/` configuration adapted to **this** project's tech stack, team, and conventions.
>
> The setup that follows is _stack-agnostic_ — it was distilled from a Node/React project but applies equally to Next.js, Vue, Django, Rails, Go, Flutter, Spring Boot, etc. Whenever you see `<placeholder>` or `# TODO`, the AI must replace it with something specific to this project.

---

## How the AI should run this

You (the AI) are about to install an opinionated AI-collaboration scaffold into this repository. Treat the steps below as a checklist. **Do not pause between every numbered step** — the steps are grouped into four batches, with explicit confirmation gates at the end of each batch:

- **Gate A — after Step 0**: confirm the pre-flight summary (stack detection + answers) before any file is written.
- **Gate B — after Step 7**: show the convention layer (rules + skills + commands + agents drafted) and confirm direction.
- **Gate C — after Step 11**: show the operational layer (settings + MCP + env + hooks + drift-guard) and confirm.
- **Gate D — after Step 15**: present what was created and ask whether to commit.

Within each batch, proceed without interruption. Be incremental — small commits, each scoped to one batch. Never push or open PRs unless the user asks.

If `.claude/` already exists in this project, **stop and ask** whether to merge non-destructively, replace, or abort. Default to merge.

---

## 0. Pre-flight — inspect the project

Before generating anything, gather context:

1. **Detect the stack.** Read whichever of these exist:
    - `package.json` (Node / web)
    - `pyproject.toml` / `requirements.txt` / `Pipfile` (Python)
    - `go.mod` (Go)
    - `Gemfile` (Ruby)
    - `Cargo.toml` (Rust)
    - `pom.xml` / `build.gradle` (Java/Kotlin)
    - `composer.json` (PHP)
    - `pubspec.yaml` (Flutter/Dart)
    - `*.csproj` / `*.sln` (.NET)
    - `mix.exs` (Elixir)
2. **Identify**: language(s), framework(s), database(s), API style (REST / GraphQL / RPC), test runner, linter, formatter, type-checker, package manager.
3. **Skim the tree** (top 2–3 levels) to understand the layout: monorepo? client/server split? layered architecture?
4. **Detect existing tooling**: pre-commit hooks (`.husky/`, `.pre-commit-config.yaml`, `lefthook.yml`), CI (`.github/workflows/`), editor configs, runtime pinning (`.nvmrc`, `.tool-versions`, `mise.toml`).
5. **Detect existing AI scaffold**: `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `.claude/`, `.agents/`, `.mcp.json`.
6. **Ask the user** — short, batched. Do not generate files until these are answered:
    - "What's the team's spoken/written language? (used for PR bodies, CHANGELOG, review summaries — code/commits stay English)"
    - "Is there a database whose schema must be verified before writing queries? If yes, where is it documented (catalog, migrations, MCP server, ORM schema)?"
    - "Are there business rules or domain concepts the AI must NEVER guess? (e.g. soft-delete convention, multi-tenancy, audit columns, status flow)"
    - "Any architecture rule you want enforced? (e.g. Controller→Service→Repository, Hexagonal, Feature-folder, MVVM)"
    - "What MCP servers should this project use? (Oracle / Postgres / data-catalog / context7 / serena / playwright / custom)"
    - "What external services / secrets does the app need? (DB credentials, API keys, OAuth, webhook secrets) — used to draft `.env.example`"
    - "Deploy target? (Vercel / self-hosted / Docker / k8s / serverless) — used to scaffold CI/CD if requested"
    - "Any existing AI-collaboration files (CLAUDE.md, AGENTS.md, .cursorrules) to import?"

Summarize back what you learned. **→ Gate A: confirm with the user before proceeding.**

---

## 1. Create folder structure

```
<repo-root>/
├── CLAUDE.md                    # root entrypoint for AI context (generated LAST in Step 14)
├── .claude/
│   ├── rules/                   # non-negotiable conventions, imported by CLAUDE.md
│   │   ├── working-principles.md
│   │   ├── backend.md           # only if project has a backend
│   │   ├── frontend.md          # only if project has a UI
│   │   ├── types.md             # only if shared types between layers
│   │   ├── testing.md           # test conventions (unit/integration/e2e)
│   │   └── git-workflow.md
│   ├── skills/                  # one folder per skill, each with SKILL.md (+ optional references/, templates/)
│   ├── commands/                # slash-commands (.md per command)
│   ├── agents/                  # subagents (.md per agent, with frontmatter)
│   ├── settings.json            # shared, checked-in: hooks + permissions
│   └── settings.local.json      # personal overrides, gitignored
├── .agents/                     # universal AI knowledge — read by ANY AI agent
│   ├── AGENTS.md                # universal rules — Claude, Cursor, Copilot, etc.
│   ├── active.md                # current-state tracker — what's in-flight, open questions
│   ├── topics/                  # business knowledge AI must NEVER guess
│   │   ├── domain.md
│   │   ├── data-schema.md
│   │   ├── auth-flow.md
│   │   └── deployment.md
│   ├── features/                # per-feature working notes (one folder per feature)
│   ├── sessions/                # per-session AI handoff notes (long-running tasks)
│   └── private/                 # gitignored — personal scratch, drafts
├── .mcp.json                    # per-project MCP servers (if any)
├── .env.example                 # documented secrets — checked in
├── .gitignore                   # must exclude: .env, .claude/settings.local.json, .agents/private/
├── docs/
│   ├── onboarding.md            # human-friendly walkthrough
│   ├── ai-references.md         # related plugins / reference repos
│   ├── ai-session-context.md    # cross-session continuity (mirror of memory)
│   └── codex-prompts.md         # vendor-neutral prompt recipes for non-Claude agents (optional, Step 6)
├── scripts/
│   └── check-template.mjs       # AI-scaffold drift guard (Node, cross-platform)
├── <db-schema-dir>/             # only for projects with a relational DB; e.g. server/schema/
└── (existing project files)
```

Create only the folders relevant to this stack — e.g. skip `frontend.md` for a pure CLI tool, skip `backend.md` for a static site, skip `<db-schema-dir>/` if the project has no relational DB.

---

## 2. Generate `.agents/` skeletons

`.agents/` is the **universal** AI knowledge folder — read by any AI agent (Claude, Cursor, Copilot, Aider). Anything here must work without Claude-specific syntax.

### Folder roles (one line each)

- `AGENTS.md` — universal rules every AI agent must follow on this repo (vendor-neutral).
- `active.md` — current-state tracker: what's in-flight, who's blocked, open questions.
- `topics/` — durable business knowledge AI must NEVER guess (domain, schema relations, auth, deployment).
- `features/` — per-feature working notes, one folder per feature; lives across multiple sessions.
- `sessions/` — per-session AI handoff notes for long-running tasks; trimmed when the task lands.
- `private/` — gitignored personal scratch / drafts. Not shared.

### `.agents/AGENTS.md` skeleton

```markdown
# AGENTS.md — universal rules for AI assistants

This file is read by **any** AI agent working on this repo (Claude Code, Cursor, Copilot, Aider, …). Vendor-specific tooling lives in `.claude/` (or `.cursor/`, etc.) — this file is the lowest common denominator.

## Read first

1. `CLAUDE.md` (or your tool's equivalent entrypoint).
2. `.claude/rules/working-principles.md` — think-before-coding, surgical changes, never-guess.
3. `.agents/topics/` — every business rule the AI must not invent.
4. `.agents/active.md` — what's currently in-flight.

## Hard rules

- Never invent business rules, schema columns, or relations. If unknown → ask, then record in `.agents/topics/`.
- Never commit secrets (`.env`, credentials, tokens). Reference them via env vars only.
- Never push, open PRs, or run destructive commands (`rm -rf`, `git reset --hard`, `git push --force`) unless explicitly asked.
- Match existing conventions before introducing new patterns.
- Surface uncertainty rather than guessing — a clarifying question is cheaper than a wrong scaffold.

## Plan before code

Non-trivial work (touches ≥ 2 files or ≥ 2 layers) must NOT start with code:

1. Read the relevant files.
2. Write the plan to `.agents/features/<feature>/spec.md` (requirements) + `ledger.md` (task tracking).
3. Get the user's confirmation on the plan files.
4. Implement following `ledger.md`, updating task status (`[x]`) as you go.

**Fast lane** — a change that meets ALL of: (a) ≤ 5 lines changed, (b) 1 file, (c) an existing test already covers the behavior → skip spec/ledger; the commit message + test output are the evidence. Exceeding any one condition = normal flow. **Never split large work into small pieces to dodge the process.**

## Pick the right entry point for the work type

| Work type                                       | Entry point    |
| ----------------------------------------------- | -------------- |
| New feature                                     | `/new-feature` |
| Something built is wrong/broken                 | `/fix`         |
| Extend/change an existing feature               | `/enhance`     |
| Restructure without behavior change             | `/refactor`    |
| Unknown root cause — analyze first (read-only)  | `/investigate` |
| End of session / before context fills           | `/checkpoint`  |

Non-Claude agents (Codex, Cursor agents, …) use the equivalent prompt recipes in `docs/codex-prompts.md`.

## TDD when possible

If the task is a bug fix or a testable new behavior: failing test first (red) → make it pass (green) → refactor.

**Testing guards (no reward-hacking):**

- Never add `.skip` / comment out tests without explicit human permission.
- Never write fake assertions (e.g. `expect(true).toBe(true)`).
- Bug fixes must reproduce the failure and show the log BEFORE fixing.

If the task isn't testable (UI polish, docs, config), state "skipping TDD because …" explicitly.

## Commit often, commit small

- 1 logical change = 1 commit (not 1 day = 1 commit).
- More than ~30 minutes of uncommitted work → stop and commit what's done.

## Review before declaring done

Before telling the user "done": self-review the diff against the project rules, fix every P0/P1 found, then declare complete.

### Review loop (tasks ≥ 5 steps)

1. **The parent prepares the diff for the reviewer** — run `git diff HEAD > /tmp/review.diff` and pass the PATH to the `code-reviewer` subagent. **Never let the reviewer collect the diff itself** (subagent-layer tool output gets truncated → the reviewer burns its budget re-reading files by hand and dies before a verdict).
2. `code-reviewer` (read-only) reviews + **actually runs the test suite and attaches the output** — returns **READY / NEEDS WORK**.
3. **Ownership of fix rounds**: NEEDS WORK → the parent sends the P0/P1 list back to the builder, then re-reviews **automatically without asking the user** — max 2 rounds (consistent with the 2-strike rule below); still NEEDS WORK after round 2 → escalate to the user with options.
4. Output **without** a Verdict line = a failed review → treat as **NEEDS WORK**. Never default to READY.
5. **Verify step (after READY)** — run `qa-tester` (read-only) against the acceptance criteria in the feature ledger: test suites always; e2e/browser smoke only when a dev server is already up. Anything it cannot run → mark SKIP with the reason, never silently pass.

## Reporting back / Review

- **Evidence-based Verification:** A passing type-check is not a passing test. "Looks correct" ≠ "I ran it and it succeeded".
- Do not claim a test passes or a bug is fixed unless you have attached the real terminal output log to the ledger or chat.
- If you can't verify something, say so explicitly.

## Debugging & Escalation (2-Strike Rule)

- If you attempt to fix an error and fail **2 consecutive times**, stop immediately.
- Do not blindly retry or rewrite the whole file.
- Summarize the situation and wait for human direction (PLAN / CORRECTION / STOP).

## Where to write things

- Long-lived domain knowledge → `.agents/topics/<slug>.md`
- Feature Specs & Ledgers → `.agents/features/<feature>/spec.md` (Requirements) and `ledger.md` (Task Tracking)
- Single-session scratchpad → `.agents/sessions/<date>-<slug>.md`
- Personal drafts → `.agents/private/` (gitignored)
```

### `.agents/active.md` skeleton

```markdown
# Active state

last_updated: <YYYY-MM-DD>

## In-flight

- <feature / branch / owner — one line each>

## Blocked / open questions

- <question — who can answer / when expected>

## Recently landed (last 2 weeks)

- <title — link to PR / commit>

> Trim items older than ~2 weeks. Long-lived facts move to `.agents/topics/`.
```

### `.agents/topics/<slug>.md` skeleton (one per topic)

Suggested starter set:

| File             | Contains                                                             |
| ---------------- | -------------------------------------------------------------------- |
| `domain.md`      | vocabulary, entities, statuses and their transitions                 |
| `data-schema.md` | entity relations, soft-delete convention, audit columns, ID strategy |
| `auth-flow.md`   | identity source, session/token lifecycle, multi-tenancy boundary     |
| `deployment.md`  | environments, secrets handling, release process, rollback            |

Skeleton for each:

```markdown
# <Topic Name>

last_updated: YYYY-MM-DD
filled_by: <name or "AI via /fill-topics">

## Summary

<one paragraph: what this topic covers and why the AI must not guess it>

## Key facts

- <fact 1>
- <fact 2>

## Open questions

- TODO: verify with <source> — <question>

## Sources

- <link / doc / Slack thread / person to ask>
```

Anything unknown stays as `TODO: verify with <source>` — that line itself becomes the AI's hint to ask before acting. Use the `/fill-topics <slug>` command (Step 6) to populate via interview rather than guessing.

---

## 3. Schema cache — data-layer source of truth

If the project has a relational DB (or any external data store with a schema), set up a **schema cache** so the AI never guesses column names, types, or relations.

### Folder

```
<schema-dir>/                # e.g. server/schema/
└── <TABLE_OR_COLLECTION>.md
```

### File format

```markdown
# <TABLE_NAME>

source: <where the schema came from — e.g. "oracle-mcp", "prisma migrate", "data-catalog">
last_refreshed: YYYY-MM-DD

## Columns

| Column | Type | Nullable | Note |
| ------ | ---- | -------- | ---- |

## Relationships

<for stores without FK constraints, record relations manually; mark unknowns as "TODO: verify">

## Notes

<any quirk — soft-delete column, computed, trigger-driven, vendor-specific>
```

### Workflow when writing data-layer code

1. **Check `<schema-dir>/<TABLE>.md`** — if it exists and `last_refreshed` is recent, use it.
2. **If missing or stale** — fetch from the project's source of truth in this order:
    1. Live MCP server (e.g. database-catalog MCP) for relations and cross-system context.
    2. ORM-generated types or Prisma schema, if available.
    3. Live DB query (`SELECT column_name, data_type FROM information_schema.columns WHERE table_name = '<...>'` or DB-specific equivalent).
3. **Save back** to `<schema-dir>/<TABLE>.md` with the new `last_refreshed`.
4. **Never proceed** to write SQL until the schema is verified.

For databases without FK constraints (Oracle 11g, MongoDB, BigQuery) — relationships **must be recorded manually**. Mark unknowns as `TODO: verify with <team>`; never invent a join.

---

## 4. Generate rule files (`.claude/rules/`)

### 4.1 `working-principles.md` — universal (always create)

> The principles below are distilled from Andrej Karpathy's collaboration skills — see [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) (also mirrored at `multica-ai/andrej-karpathy-skills`). When in doubt, consult the original repo for the full skill set; the items here are the minimum every project should adopt.

```markdown
# Working Principles

Cross-cutting rules for any layer. Read before any non-trivial change.
Distilled from Andrej Karpathy's collaboration skills (https://github.com/forrestchang/andrej-karpathy-skills).

## 1. Think Before Coding — surface assumptions first

- If the request is ambiguous, **stop and ask**. Do not guess-then-code.
- If multiple interpretations exist, **list the options** and ask — never pick silently.
- State assumptions explicitly: _"I will do X assuming Y — correct?"_
- Before writing data-layer code → verify schema. **Never guess column names, types, or relationships.**

## 2. Surgical Changes — touch only what's needed

- Every changed line must trace directly to the user's request.
- **No drive-by refactors.** Do not "improve" adjacent code, comments, or formatting.
- Do not rename vars, reorder imports, change quote style, or re-indent surrounding code.
- If your change orphans code, remove only what _your edit_ orphaned. Pre-existing dead code stays unless asked.
- Spot something worth fixing? Flag it separately ("noticed X looks like a bug — fix now or later?") — do not silent-fix.

## 3. Small, Reversible Steps

- Prefer a sequence of small commits over one large change.
- Verify each step (run tests / type-check / linter) before moving on.
- A failing build halts forward motion — fix root cause before continuing.

## 4. Never Invent Business Rules

- Domain knowledge (statuses, workflow rules, naming, soft-delete semantics, multi-tenant rules) lives in the team's head or `.agents/topics/*.md` — **ask, don't infer**.
- If the team confirms a rule that's missing from those docs, **propose adding it** there.

## 5. Verify Sources, Don't Trust Memory

- Library / API claims → check current docs (context7 MCP, official docs) or `grep` the codebase. Your training data may be outdated.
- File paths, function names, and flags get renamed/removed — verify they exist before recommending.
- "I think it works like this" is not evidence; reading the file is.

## 6. Honest Reporting — don't claim what you didn't verify

- A passing type-check is not a passing test.
- "Looks correct" ≠ "I ran it and it succeeded".
- If you can't test something (no UI access, missing credentials, slow build) — **say so explicitly** rather than asserting success.
- Surface failed tool calls; don't hide them with retries or fallbacks.

## 7. Match Scope to the Request

- Do not add features, refactor, or introduce abstractions beyond what was asked.
- A bug fix doesn't need surrounding cleanup. A one-shot operation doesn't need a helper function.
- Three similar lines is better than a premature abstraction.
- No half-finished implementations either — pick a complete, smaller scope over a sprawling, partial one.

## 8. Preserve Existing Conventions

- Match the surrounding code's style, naming, and patterns before introducing new ones.
- If the project has 5 examples of pattern X, use pattern X — don't invent pattern Y because it's "better".
- New patterns require a separate conversation with the user, not a silent introduction.

## 9. Fail Loudly — errors are signal

- Don't add error handling, fallbacks, or validation for scenarios that can't happen.
- Don't swallow errors with empty catch blocks or silent `try/except: pass`.
- Trust internal code and framework guarantees; only validate at system boundaries (user input, external APIs).
- A crash with a stack trace is more useful than a silent miscompile.

## 10. Comments Explain WHY, Not WHAT

- Default to writing no comments. Well-named identifiers already explain what the code does.
- Add a comment only when the WHY is non-obvious: a hidden constraint, a subtle invariant, a workaround for a specific bug, behavior that would surprise a reader.
- Don't reference the current task or PR ("added for issue #123") — that belongs in the commit message and rots as the codebase evolves.
```

### 4.2 `backend.md` — if the project has a server

```markdown
# Backend Rules

## Tech Stack

<language> · <framework> · <orm/query-builder> · <db> · <validation lib>

# TODO: fill from pre-flight.

## Architecture: <pattern>

State the chosen pattern explicitly. Common shapes — pick one or describe the project's own:

- **Layered**: Controller → Service → Repository
- **Hexagonal / Ports & Adapters**: domain at center, adapters at edges
- **Vertical slice / feature folder**: per-feature (handler + logic + data)
- **Clean architecture**: entities / use-cases / interface adapters / frameworks

For each layer, state:

- **What lives here** (one sentence)
- **What does NOT live here** (the line you do not cross)
- **Reference file** (a representative existing file in the repo)

## Data access strategy — when to inline vs externalize

| Query shape                                      | Use                                            | Why                    |
| ------------------------------------------------ | ---------------------------------------------- | ---------------------- |
| Simple, stable, single-source                    | inline                                         | closer to call site    |
| Reusable across services / non-devs need to edit | externalize (separate file / migration / view) | single source of truth |
| Dynamic shape                                    | inline (template-builder with safe binds)      | clearer at call site   |

# TODO: replace with the project's actual choice.

## Auth / identity

- How is the caller's identity established? (JWT middleware? session? mTLS?)
- **Read identity from the request context, never from request body.**
- Request DTOs/schemas in body must contain only client-mutable fields — never include `userId` / tenant id (leaks to API spec, implies clients can set identity).
- **Don't trust claim shapes.** Token claims can be missing, non-string, or whitespace-only — validate the type and `trim()` before using identity claims; reject empties at the controller boundary instead of passing them to the data layer.

## Schema verification (data layer)

Before writing any query, **verify the schema** via the schema cache (Step 3).

If the schema is unknown → fetch it, save/cache it, **never guess**.

## Hard rules

- Never edit auto-generated files. Common patterns to list explicitly:
    - OpenAPI / Swagger output (e.g. `<openapi-output>`)
    - ORM-generated client (Prisma `@prisma/client`, sqlx prepare cache)
    - Protobuf / gRPC stubs
    - Route trees (TanStack Router `routeTree.gen.ts`, file-based router caches)
    - GraphQL codegen output
- Always use parameterized queries / prepared statements — no string concatenation of user input.
- <db-specific gotchas — TODO: e.g. "Postgres only", "MySQL 5.7 has no JSON_TABLE", "Oracle 11g: no FETCH FIRST / OFFSET / JSON_TABLE">.
- Verify schema before writing SQL.

## Skills for details

`<stack>-development` · `<db>-connector` · `<api-framework>-layer-generator`

# TODO: fill after Step 5.
```

### 4.3 `frontend.md` — if the project has a UI

```markdown
# Frontend Rules

## Tech Stack

<framework> · <language> · <bundler> · <styling> · <ui kit> · <router> · <data fetching> · <forms>

# TODO: fill from pre-flight.

## Critical Rules (violation = reject)

> Details and full examples live in `.claude/skills/frontend-development/` and `.claude/skills/design-system/`. **Read both before writing UI.**

# TODO: replace each rule below with this project's equivalent. The _categories_ below are the universal pattern — the _specifics_ depend on stack.

1. **Forms** — use the project's standard form abstraction. List the canonical components and forbid raw alternatives.
2. **Inputs / native HTML** — use the design-system primitives, never raw `<input>` / `<select>` / `<textarea>` (or stack equivalent).
3. **Shared components** — list the existing reusable components (table below). Do not duplicate them.

    | Component         | Import   | Use for                 |
    | ----------------- | -------- | ----------------------- |
    | `<Form>`          | `<path>` | every form              |
    | `<DataTable>`     | `<path>` | server-paginated tables |
    | `<EmptyState>`    | `<path>` | no-data state           |
    | `<Loader>`        | `<path>` | loading state           |
    | `<ErrorBoundary>` | `<path>` | error boundary          |

    # TODO: fill from inspection of the project.

4. **Toast / notification** — single canonical import. Forbid alternatives.
5. **Styling** — choose ONE: utility-first / CSS modules / styled-components / tokens. No inline styles unless the chosen system explicitly allows them. Reference the design-system skill for tokens.
6. **Icons** — single canonical icon source (or split: feature icons vs. UI chrome icons).
7. **Data fetching** — single canonical client (e.g. TanStack Query / SWR / Apollo / RTK Query). No raw `useEffect + fetch`. Hooks live in a single conventional location.
8. **Generated / vendor components** — never modify. List paths.

## Component structure

    <client-or-src>/components/
    ├── ui/      # generated / primitives — NEVER modify
    ├── layout/  # app chrome
    ├── shared/  # reusable building blocks
    └── pages/   # page-specific

## Routes

# TODO: file-based router? config-based? Where do route definitions live?

## Self-check before commit

- [ ] No native input/form elements (per rule #2)
- [ ] All forms use the canonical form abstraction
- [ ] All API calls via the canonical data-fetching client
- [ ] Toast from the canonical source only
- [ ] No inline styles unless the chosen system permits
- [ ] Existing shared components reused, not duplicated
- [ ] Styling tokens come from design-system
- [ ] No edits to generated/vendor folders

## Required reading before UI work

- `.claude/skills/design-system/SKILL.md`
- `.claude/skills/frontend-development/`

## Docs to update when shared code changes

| Change                      | Update                                           |
| --------------------------- | ------------------------------------------------ |
| Add/remove shared component | this file (table) + `frontend-development` skill |
| Add/remove custom hook      | `frontend-development` skill                     |
| Change styling convention   | `design-system` skill                            |
```

### 4.4 `types.md` — if there are shared types between layers

```markdown
# Shared Types

Types shared between client and server (or between modules) — single source of truth.

    shared/types/
    ├── index.ts         # cross-cutting: API envelope, auth user, pagination, audit
    ├── api.generated.* # auto-generated from API spec — DO NOT edit by hand
    ├── db/              # storage-row types — naming matches storage (e.g. UPPER_SNAKE for SQL columns)
    └── dto/             # transport types — language idiom (e.g. camelCase for JS)

## Usage rules

| Layer                     | Imports from | Why                                   |
| ------------------------- | ------------ | ------------------------------------- |
| Repository / data access  | `db/`        | matches DB columns                    |
| Service / business logic  | `dto/`       | language-idiomatic, after a converter |
| Controller / API boundary | `dto/`       | forwarded to client                   |
| Frontend                  | `dto/`       | never imports `db/`                   |

**Data flow**: storage → `<Entity>Row` (db) → converter → `<Entity>` (dto) → consumer.

# TODO: replace `Row` / `Dto` naming with this project's convention if different.
```

### 4.5 `testing.md` — universal stub (always create if any test runner exists)

```markdown
# Testing Rules

## Test runner

<runner> — e.g. Vitest / Jest / Pytest / Go test / RSpec.

# TODO: fill from pre-flight. List the command(s) used.

## Test layers — pick the ones the project uses

- **Unit**: pure functions, isolated business logic. Fast (<1s each), no I/O.
- **Integration**: real DB / real HTTP layer / real filesystem. Slow but trustworthy.
- **End-to-end**: full app, browser/CLI driving real flows.

## Hard rules

- **Don't mock the database** for queries that depend on DB-specific behavior (constraints, triggers, dialect quirks). Mocked tests pass while prod migrations break.
- **Don't test implementation details.** Tests should describe observable behavior, not internal helper calls.
- **One assertion per concept.** A test that fails should tell you exactly what broke.
- **Test data is data, not code** — fixtures live in `<fixtures-dir>`, not as inline literals scattered across files.
- **Flaky tests are blocker bugs.** Don't retry-loop them; fix the root cause.
- **No Reward-Hacking:** Do not use `.skip` or comment out tests without explicit human permission. Do not write fake assertions (e.g., `expect(true).toBe(true)`).
- **Reproduce before fix:** For bug fixes, write a failing test or reproduce the error and show the log first before attempting to fix the code.

## Naming

- File: `<module>.test.<ext>` co-located with source, OR `__tests__/<module>.test.<ext>` — pick one and stick with it.
- Describe: state the behavior, not the function (`"rejects expired tokens"` not `"validateToken returns false"`).

# TODO: fill in conventions specific to this project's chosen stack.
```

### 4.6 `git-workflow.md`

> This file owns the **full** language policy table. Other files only summarize and link here.

```markdown
# Git Workflow

## Commit message — Conventional Commits

Format: `<type>(<scope>): <subject>` — subject in English, imperative mood, ≤72 chars, no trailing period.

| Type       | When                        |
| ---------- | --------------------------- |
| `feat`     | new user-visible capability |
| `fix`      | bug fix                     |
| `refactor` | behavior-preserving change  |
| `chore`    | infra/config/deps           |
| `docs`     | documentation only          |
| `test`     | tests only                  |
| `style`    | formatting/whitespace/lint  |

## Branch naming

| Pattern              | For                        |
| -------------------- | -------------------------- |
| `feat/<slug>`        | new feature                |
| `fix/<slug>`         | bug fix                    |
| `docs/<slug>`        | docs only                  |
| `chore/<slug>`       | infra/config               |
| `refactor/<slug>`    | behavior-preserving change |
| `claude/<slug>-<id>` | AI-generated branch        |

`<slug>` = kebab-case, ≤4 words.

# TODO: confirm the integration / release branch names with the user. Common shapes:

# - `<branch> → develop → main` (gitflow-lite)

# - `<branch> → main` (trunk-based)

## Pre-commit hooks

<hook-runner> runs <task-list>. See `.husky/pre-commit` (or `.pre-commit-config.yaml` / `lefthook.yml`).

- **Never skip with `--no-verify`** unless the user explicitly asks. Fix the root cause.
- Hook failure = no commit → don't `--amend` afterwards (would modify the _previous_ commit). Fix, re-stage, make a new commit.

## Language policy

| Artifact                      | Language        | Why                                                     |
| ----------------------------- | --------------- | ------------------------------------------------------- |
| Commit subject                | English         | Conventional Commits                                    |
| Commit body                   | mixed OK        | English for technical logs, <team-language> for context |
| PR title                      | English         | platform readability                                    |
| PR body                       | <team-language> | team reads it                                           |
| CHANGELOG                     | <team-language> | same                                                    |
| Review summary                | <team-language> | findings for the team                                   |
| Compiler / lint / test output | English         | raw tool output — never translate                       |

# TODO: replace `<team-language>` from pre-flight.

## Working with the AI

- Never commit secrets (`.env`, credentials) — warn the user if instructed to.
- Stage files by name, not `git add .` (avoid surprises).
- Create a new commit instead of `--amend` unless asked.
- Don't push or open PRs unless explicitly told.

## Co-author trailer for AI-assisted commits

    Co-Authored-By: Claude <noreply@anthropic.com>

> Teams that want model provenance can use a fuller form, e.g. `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`. Pick one and apply consistently.
```

---

## 5. Generate skill scaffolds (`.claude/skills/<skill>/`)

A **skill** is a focused playbook the AI loads when its trigger condition matches.

### Folder shape — progressive disclosure

```
.claude/skills/<skill-name>/
├── SKILL.md             # the playbook (always present)
├── references/          # deep-dive docs loaded on demand (optional)
│   └── <topic>.md
└── templates/           # code/config templates the skill stamps out (optional)
    └── <thing>-template.<ext>
```

**When to add subfolders**: if `SKILL.md` exceeds ~100 lines or you have reusable templates the skill stamps into the project, split:

- `references/` — long-form examples, edge-case tables, full code samples. The skill links into them on demand so the entry file stays scannable.
- `templates/` — actual files (or fragments) the skill copies into the project. Keep them in working condition — they're not pseudo-code.

For any `SKILL.md` longer than ~100 lines, **add a `## ⚠️ Critical Rules` section near the top** (the drift-guard checks for this header). It's where the must-not-violate items go, so a skimmer can't miss them.

### Universal `SKILL.md` template

```markdown
---
name: "<skill-name>"
description: "<one-paragraph: when to use this skill, what it produces. The AI matches against this string.>"
---

# <Skill Title>

<one-paragraph purpose>

## ⚠️ Critical Rules

- <non-negotiable 1>
- <non-negotiable 2>

## When to use

- <trigger 1>
- <trigger 2>

## Inputs

- <what the AI needs from the user before starting>

## Workflow

    Phase 1: <name>
       ↓
    Phase 2: <name>
       ↓
    Phase 3: <name>

## Phase 1 — <name>

<step-by-step instructions>

## Hard rules

- <project-specific don'ts>

## Templates

- [<thing>-template.md](templates/<thing>-template.md)

## References

- [<topic>.md](references/<topic>.md) — read on demand

## Related skills / agents

- `<other-skill>` — <when>
- `<agent-name>` — <when>
```

### Suggested skills (adapt to stack)

| Skill                             | Purpose                                                                    |
| --------------------------------- | -------------------------------------------------------------------------- |
| `<stack>-development`             | orchestrates the build flow for a backend feature (schema → impl → review) |
| `<stack>-frontend-development`    | orchestrates UI feature (design → build → review)                          |
| `design-system`                   | colors, spacing, typography, component patterns — read BEFORE writing UI   |
| `<db>-connector`                  | query patterns, transactions, error handling for the DB                    |
| `<db>-schema-cache`               | how to verify schema before writing queries (catalog → cache → live)       |
| `<api-framework>-layer-generator` | end-to-end recipe to add a new endpoint across layers                      |
| `create-table` / migrations       | conventions for new tables / collections / migrations                      |
| `logger-system`                   | logging conventions and context propagation                                |

> Generate **stubs** for skills the project clearly needs (based on pre-flight). Leave others for later. Each stub should be runnable as-is but explicitly mark `# TODO` where the user must fill in detail.

---

## 6. Generate command scaffolds (`.claude/commands/<command>.md`)

Slash commands are short prompts the user types as `/command [args]`.

### Universal command template

```markdown
---
description: <one-line: what this command does>
argument-hint: [optional-args]
---

# /<command> $ARGUMENTS

<short purpose>

## Step 1 — <name>

<instructions>

## Step 2 — <name>

<instructions>

## Output

<what to report back to the user>
```

### Suggested commands

| Command               | Purpose                                                              |
| --------------------- | -------------------------------------------------------------------- |
| `/new-feature <name>` | plan a NEW feature as Epic → Stories → Tasks; save to `.agents/features/<name>/spec.md` + `ledger.md` |
| `/fix <bug>`          | corrective work — reproduce-first, find root cause, minimal surgical fix; UI branch enforces shared-component reuse |
| `/enhance <feature>`  | evolutive work — extend an existing feature; append Story/Tasks to its existing ledger, not a new Epic |
| `/refactor <target>`  | perfective work — behavior-preserving; existing tests stay green, no API change |
| `/investigate <q>`    | read-only root-cause analysis before deciding to fix/enhance (no code changes) |
| `/review-pr [target]` | review pending diff against this project's rules                     |
| `/onboard`            | walk a new team member (or new AI session) through the repo          |
| `/checkpoint`         | write a structured session snapshot to `.agents/sessions/` + refresh `active.md` |
| `/check-template`     | run `scripts/check-template.mjs` and report drift in the AI scaffold |
| `/fill-topics <slug>` | Q&A interview to populate `.agents/topics/<slug>.md` (see Step 2)    |
| `/security-review`    | focused security pass on pending changes                             |

The first five cover the full work **lifecycle** — new (`/new-feature`), corrective (`/fix`), evolutive (`/enhance`), perfective (`/refactor`), understanding (`/investigate`). A single planning command isn't enough; real maintenance is mostly fix/enhance, which are different processes from greenfield. Route work-type → command in `AGENTS.md` (Step 2) so the AI picks the right one.

`/fill-topics` is the canonical way to put **business knowledge** into the project. It must:

1. Read the existing `.agents/topics/<slug>.md` (skeleton or partial fill).
2. Ask the user one question at a time, in their team language.
3. Write each answer back into the file with a `last_updated: YYYY-MM-DD` line.
4. Mark anything the user can't answer immediately as `TODO: verify with <source>` — never guess.

### Non-Claude agents — mirror the commands as prompt recipes

`.claude/commands/` is Claude-specific syntax. If the team also uses other coding agents (Codex, Cursor agents, …), mirror each lifecycle command as a short **vendor-neutral prompt recipe** in `docs/codex-prompts.md` — one heading per command, one paste-able prompt each (e.g. *"Fix \<bug\>. Reproduce it first, show the failing evidence, then make the smallest safe change."*). Route from `.agents/AGENTS.md`: Claude → `.claude/commands/`, others → `docs/codex-prompts.md`. Have the drift guard (Step 11) assert the cross-reference stays present so the second tool's path doesn't rot.

---

## 7. Generate agent scaffolds (`.claude/agents/<agent>.md`)

Subagents are specialized roles invoked by the main loop. They have isolated context and a restricted toolset.

### Universal agent template

```markdown
---
name: "<agent-name>"
description: "<when to invoke. Be explicit about workflow position — e.g. 'STEP 2 of feature flow, after design-spec'. The orchestrator matches against this.>"
tools: Read, Write, Edit, Bash, Grep, Glob
model: sonnet
color: <green|cyan|red|blue|yellow>
maxTurns: 25
---

You are a <Role> — <one-sentence identity>.

## Project conventions — read BEFORE acting

Read these source-of-truth files first; agent prompts can drift from current rules:

- `.claude/rules/<relevant>.md`
- `.claude/skills/<relevant>/SKILL.md`
- `.agents/topics/<relevant>.md`

## Core rules

<bulleted list of non-negotiables>

## Workflow

<step-by-step>

## Budget discipline

You have a hard turn limit (`maxTurns`). Reserve the last ~20–25% of turns for your final report — never end without one. Partial results with explicit gaps ("did not verify X/Y") beat dying silently mid-run.

## Checklist before completing

- [ ] <item>
```

### Suggested agents and orchestration

| Agent                        | Phase                | Tools (typical)                                                          |
| ---------------------------- | -------------------- | ------------------------------------------------------------------------ |
| `<stack>-design-spec`        | 1. design            | Read, Grep, Glob, Write                                                  |
| `<stack>-builder` (backend)  | 2a. implement server | Read, Write, Edit, Bash, Grep, Glob, + DB MCPs                           |
| `<stack>-builder` (frontend) | 2b. implement UI     | Read, Write, Edit, Bash, Grep, Glob                                      |
| `code-reviewer`              | 3. review            | Read, Grep, Glob, Bash (read-only — never modifies code)                 |
| `qa-tester`                  | 4. verify            | Read, Grep, Glob, Bash, + browser MCP (read-only — never modifies code) |

**Orchestration pattern**: `design-spec → builder(s) [parallel for full-stack] → code-reviewer → qa-tester`. The `code-reviewer` runs **proactively** after every builder; if it finds P0/P1 issues, fix and re-run — the orchestrator owns the fix rounds (re-review automatically, max 2 rounds, then escalate to the user).

**The `qa-tester` closes the loop** — review proves the code reads correctly; verify proves it *behaves* correctly. After `code-reviewer` returns READY, `qa-tester` executes the acceptance criteria from the feature ledger against the real system, cheapest first: unit/integration suites always → e2e only if a dev server is already reachable (it never starts servers) → browser-MCP smoke for UI criteria (console errors, screenshot evidence). Contract: every PASS cites raw output; anything unverifiable is **SKIP with a reason, never PASS**; FAIL is treated like NEEDS WORK (back to the builder, max 2 rounds). Like the reviewer, it must never end silently — a partial report with explicit SKIPs beats dying mid-run.

The `code-reviewer` should return a binary verdict gated on severity (any P0/P1 → `NEEDS WORK`, else `READY`), use the **same vocabulary as `/review-pr`** so the agent and the command agree, and carry an explicit **reward-hacking guard list** (added `.skip`, empty assertions like `expect(true).toBe(true)`, evidence-free "tests pass" claims, relaxed lint/type/hook config). Builders are told not to do these in `AGENTS.md`; the reviewer is the one that *catches* them — put the guard on both sides.

**Bound the reviewer's work to the diff, not the repo.** The biggest reliability failure for a review agent is dying mid-review and returning a partial result with no verdict. The cause is rarely the turn cap — it's that the agent's work scales with diff size (an instruction like "read each changed file completely" costs ~1 turn per file, so the run truncates once the file count is high enough; raising `maxTurns` just moves that cliff). Make the work bounded instead:

- **Hand the reviewer the diff as a pre-saved FILE** — the orchestrator (or `/review-pr`) collects it once (`git diff HEAD > /tmp/review.diff`) and passes the *path*; the reviewer `Read`s that file. Do NOT let the reviewer run `git diff` itself: tool output at the subagent layer gets truncated, so a self-collected diff comes back mangled and the agent falls back to re-reading every changed file by hand — that's what actually burns the budget. The agent analyzes; it does not re-discover context per file.
- **Exclude generated/vendored noise before handoff** — lockfiles (`*-lock.json`, `pnpm-lock.yaml`, `Cargo.lock`, `poetry.lock`…), generated API clients/types (OpenAPI / GraphQL codegen output), and router/build caches. These are tracked, often 5k–11k lines each, and have nothing to review.
- **Read diff-scoped** — open only the changed regions + immediate surrounding context (`Read` with offset), never whole large files top-to-bottom.
- **Size guard** — if the diff after exclusion still exceeds a threshold (~20 files / ~1500 lines), the reviewer refuses with `Verdict: NEEDS WORK — diff too large for single-pass review, split it` rather than attempt a doomed pass.
- **Budget discipline in the prompt** — tell the reviewer to reserve its last ~25% of turns for the report: the moment it senses it is running low, stop gathering and emit findings with whatever it has. A partial review ("4 issues confirmed; did not finish files X/Y") + `Verdict: NEEDS WORK` is far more useful than dying mid-read with nothing. `maxTurns` is a backstop, not a reliability mechanism.
- **A missing verdict is a FAILED review, never a pass** — the verdict sits at the end of the output, so a truncated run silently drops it; the orchestrator must treat "no verdict line" as `NEEDS WORK`. And don't make the reviewer re-read rule files it already receives via `CLAUDE.md` `@`-imports — pure wasted budget.

In each agent's `description`, explicitly state its workflow position so the orchestrator chains them correctly.

**→ Gate B: show the user the convention layer (Steps 1–7) and confirm direction before continuing.**

---

## 8. Configure `.claude/settings.json` and `.mcp.json`

### File split

- `.claude/settings.json` — **shared, checked-in**: hooks, common permissions, enabled MCP servers. Reviewed in PRs.
- `.claude/settings.local.json` — **personal, gitignored**: per-developer overrides (e.g. extra permissions while debugging). Add the file to `.gitignore`.
- `.mcp.json` — **per-project MCP servers**, checked in. Secrets via `${ENV_VAR}` references only.

### Shared settings template

```json
{
    "permissions": {
        "allow": [
            "Bash(<pkg-mgr> install:*)",
            "Bash(<pkg-mgr> run *)",
            "Bash(<test-runner> *)",
            "Bash(<linter> *)",
            "Bash(<formatter> *)",
            "Bash(<typechecker> *)",
            "Bash(git status)",
            "Bash(git diff:*)",
            "Bash(git log:*)"
        ],
        "deny": [
            "Bash(rm -rf:*)",
            "Bash(sudo:*)",
            "Read(./.env)",
            "Read(./.env.*)",
            "Read(**/*.pem)",
            "Read(**/*.key)",
            "Read(**/.ssh/**)"
        ],
        "ask": [
            "Bash(git push --force:*)",
            "Bash(git push --force-with-lease:*)",
            "Bash(git push origin <protected-branch>:*)",
            "Bash(git reset --hard:*)"
        ]
    },
    "enabledMcpjsonServers": ["<mcp-server-1>", "<mcp-server-2>"],
    "hooks": {
        "PreCompact": [
            {
                "matcher": "",
                "hooks": [
                    {
                        "type": "command",
                        "command": "cd \"$CLAUDE_PROJECT_DIR\" && mkdir -p .agents/sessions && F=.agents/sessions/_autosave-$(date +%Y%m%d-%H%M%S).md && { echo \"# pre-compact $(date -u +%FT%TZ)\"; echo \"branch: $(git branch --show-current 2>/dev/null)\"; echo; git status --short 2>/dev/null; echo; git log --oneline -5 2>/dev/null; } > \"$F\" || true"
                    }
                ]
            }
        ],
        "PreToolUse": [
            {
                "matcher": "Edit|Write",
                "hooks": [
                    {
                        "type": "command",
                        "command": "FILE=$(jq -r '.tool_input.file_path // empty'); case \"$FILE\" in *<repo-pattern-1>|*<repo-pattern-2>) echo '[schema-check] Editing data-layer file. Verify schema in <schema-cache-dir>/<TABLE>.md FIRST. If missing, fetch via data-catalog or DB MCP, then save. NEVER guess column names.' ;; esac"
                    }
                ]
            }
        ],
        "PostToolUse": [
            {
                "matcher": "Edit|Write",
                "hooks": [
                    {
                        "type": "command",
                        "command": "FILE=$(jq -r '.tool_input.file_path // empty'); cd \"$CLAUDE_PROJECT_DIR\" && [ -n \"$FILE\" ] && [ -f \"$FILE\" ] && <formatter-cmd> \"$FILE\" 2>&1 | tail -5 || true"
                    },
                    {
                        "type": "command",
                        "command": "cd \"$CLAUDE_PROJECT_DIR\" && <typechecker-cmd> 2>&1 | head -50"
                    },
                    {
                        "type": "command",
                        "command": "cd \"$CLAUDE_PROJECT_DIR\" && <linter-cmd> 2>&1 | tail -50"
                    }
                ]
            }
        ],
        "UserPromptSubmit": [
            {
                "hooks": [
                    {
                        "type": "command",
                        "command": "cd \"$CLAUDE_PROJECT_DIR\" && [ -f .agents/topics/domain.md ] && echo 'Reminder: domain rules live in .agents/topics/' || true"
                    }
                ]
            }
        ]
    }
}
```

The **PreToolUse schema-check** is the highest-leverage hook for projects with a relational DB and no FK constraints (Oracle 11g, MongoDB, BigQuery): it fires only when the AI is about to edit a repository or migration/SQLTab file (`*Repository.ts`, `*sqltabs/*.sql`, `*migrations/*.sql`, etc.) and reminds it to verify the schema cache before guessing column names. The reminder text appears in the AI's context before the edit runs — no blocking, just an audit trail.

The **`deny` / `ask` permission baseline** is config-only safety: `deny` blocks destructive commands (`rm -rf`, `sudo`) and reads of secrets (`.env`, `*.pem`, `*.key`, `.ssh`); `ask` gates irreversible git (force-push, push to a protected branch, `reset --hard`). It enforces the git-workflow rules with no runtime — scope it to the project's real protected branches, and do NOT add app-layer security (helmet/rate-limit) here.

The **PreCompact hook** writes a git-state breadcrumb (branch + `git status` + last commits) to `.agents/sessions/_autosave-*.md` before context is compacted, so a long session can recover "where was I" after auto-compaction. Keep it to git metadata (the conversation plan isn't reliably exposed to the hook) and gitignore `_autosave-*.md` (Step 9).

Hook payloads arrive on **stdin as JSON**. The `jq -r '.tool_input.file_path'` pattern reads the file path from that payload — keep the `jq` invocation receiving from stdin (don't redirect from a file).

Replace each `<...-cmd>` with the actual command for this stack:

| Stack             | Formatter               | Typechecker        | Linter              |
| ----------------- | ----------------------- | ------------------ | ------------------- |
| TypeScript / Node | `prettier --write`      | `tsc --noEmit`     | `eslint`            |
| Python            | `ruff format` / `black` | `mypy` / `pyright` | `ruff check`        |
| Go                | `gofmt -w`              | `go vet ./...`     | `golangci-lint run` |
| Rust              | `cargo fmt`             | `cargo check`      | `cargo clippy`      |
| Ruby              | `rubocop -a`            | `srb tc` (Sorbet)  | `rubocop`           |

Keep hooks _fast_. If full type-check is slow, scope it (incremental flag, only-changed-files), or move it to pre-commit instead of every edit.

For permissions, skim the user's recent transcripts and add commonly-used read-only commands.

### `.mcp.json` template

```json
{
    "mcpServers": {
        "<db>-catalog": {
            "command": "npx",
            "args": ["-y", "@<vendor>/<package>"],
            "env": {
                "DATABASE_URL": "${DATABASE_URL}"
            }
        },
        "context7": {
            "command": "npx",
            "args": ["-y", "@upstash/context7-mcp"]
        }
    }
}
```

For each MCP server, write **one paragraph in `CLAUDE.md` (or a rule file) describing its trigger** — when the AI should reach for it:

| MCP server     | Trigger                                                |
| -------------- | ------------------------------------------------------ |
| `<db>-catalog` | before writing any SQL — fetch column types, relations |
| `context7`     | before answering library/API questions — current docs  |
| `playwright`   | when verifying UI changes end-to-end                   |
| `serena`       | symbol-level code navigation in large codebases        |

Enable the servers you want active by default in `.claude/settings.json` → `enabledMcpjsonServers`. Servers not listed there must be enabled per session.

**Secrets**: never write API keys / credentials into `.mcp.json`. Reference them via `${VAR_NAME}` and document the variable in `.env.example` (Step 9).

---

## 9. Environment & secrets

### `.env.example`

Checked into the repo, no secret values. Lists every variable the app reads from the environment, with a one-line description and a placeholder.

```dotenv
# Database
DATABASE_URL=postgres://user:pass@host:5432/db

# Auth
JWT_SECRET=<generate-with-openssl-rand-hex-32>
SESSION_TIMEOUT_SECONDS=3600

# External services
SENDGRID_API_KEY=<from-sendgrid-dashboard>
STRIPE_WEBHOOK_SECRET=<from-stripe-cli>

# Feature flags
ENABLE_NEW_FLOW=false
```

### `.gitignore` — must include

```
# Secrets
.env
.env.local
.env.*.local

# Per-developer Claude Code config
.claude/settings.local.json

# Personal AI scratch
.agents/private/

# PreCompact autosave breadcrumbs (curated /checkpoint files stay tracked)
.agents/sessions/_autosave-*.md

# Build / cache
node_modules/
dist/
.cache/
```

(Adjust per stack — Python adds `__pycache__/`, `.venv/`; Go adds `vendor/` if used; etc.)

### Runtime version pinning

Pin the runtime so Claude Code's spawned shells (and CI) use the same version as the developer:

| Stack          | File                                                            |
| -------------- | --------------------------------------------------------------- |
| Node           | `.nvmrc` (e.g. `20.11.1`) or `package.json` `engines`           |
| Python         | `.python-version` (pyenv) or `pyproject.toml` `requires-python` |
| Multi-language | `.tool-versions` (asdf) or `mise.toml`                          |
| Go             | `go.mod` `go 1.22` line                                         |
| Ruby           | `.ruby-version`                                                 |

Document the runtime install in `docs/onboarding.md` (Step 12).

---

## 10. Set up the pre-commit hook

Use the project's existing tool if there is one (`husky`, `pre-commit`, `lefthook`, `overcommit`). If none, propose installing the most idiomatic one for the stack.

Example — Husky + lint-staged (Node ecosystem):

```bash
# .husky/pre-commit
<pkg-mgr> exec lint-staged
```

`package.json`:

```json
{
    "lint-staged": {
        "*.{ts,tsx,js,jsx}": ["eslint --fix", "prettier --write"],
        "*.{json,md,yml,yaml}": ["prettier --write"]
    }
}
```

Example — Python `pre-commit`:

```yaml
# .pre-commit-config.yaml
repos:
    - repo: https://github.com/astral-sh/ruff-pre-commit
      rev: v0.6.0
      hooks:
          - id: ruff
          - id: ruff-format
```

Run the install command (`<pkg-mgr> exec husky init`, `pre-commit install`, etc.) and verify by making a tiny test edit.

**Chain multiple hook steps with `&&`**, e.g.:

```bash
# .husky/pre-commit
<pkg-mgr> exec lint-staged && node scripts/check-template.mjs --fast
```

A hook script with steps on separate lines keeps going after a failure and exits with the *last* command's status — so a failing first step silently doesn't block the commit. `&&` makes the first failure abort the hook. (This bit a real install: lint-staged failed, the drift guard passed, the broken commit landed.)

---

## 11. Drift guard — `scripts/check-template.mjs`

A small script the team (and `/check-template`) runs to verify the AI scaffold is intact. Catches: deleted rule files, missing CLAUDE.md imports, broken `@` references, empty topic files, missing schema cache directory, long skills missing the `⚠️ Critical Rules` header, known-bad tokens that audits confirmed must never reappear.

Design it with a **`--fast` flag** from day one: structure + drift checks only, skipping the slow build-health section (typecheck/lint/tests). The fast mode is what makes it cheap enough to run on **every commit** (pre-commit hook); the full mode runs in CI. A gate that's too slow to run gets bypassed, then rots.

**Recommended: Node version (cross-platform — macOS, Linux, Windows).** Use this if the project has Node available (which is true for most web stacks even if the app itself is Python/Go/etc., because Claude Code itself is Node-based). The bash version is below as a fallback for projects where introducing Node is unwanted.

### `scripts/check-template.mjs`

Fill the two arrays (`requiredFiles`, `requiredDirs`) with paths specific to this project. The version below is the universal core — extend it with stack-specific build-health checks (e.g. `tsc --noEmit`, `pytest --collect-only`, `go vet`) once the basics work.

```javascript
#!/usr/bin/env node
// Template health check — cross-platform (macOS, Linux, Windows).
// Run from repo root:  node scripts/check-template.mjs
//
// Returns exit 0 if the template is healthy (warnings allowed),
// exit 1 if any required check fails.

import { existsSync, statSync, readFileSync, readdirSync } from "node:fs";
import { join } from "node:path";
import process from "node:process";

const isTTY = process.stdout.isTTY;
const C = {
    green: isTTY ? "\x1b[32m" : "",
    red: isTTY ? "\x1b[31m" : "",
    yellow: isTTY ? "\x1b[33m" : "",
    reset: isTTY ? "\x1b[0m" : "",
};

let failCount = 0;
let warnCount = 0;

const say = (msg) => console.log(msg);
const pass = (msg) => say(`  ${C.green}✓${C.reset} ${msg}`);
const warn = (msg) => {
    say(`  ${C.yellow}⚠${C.reset} ${msg}`);
    warnCount++;
};
const err = (msg) => {
    say(`  ${C.red}✗${C.reset} ${msg}`);
    failCount++;
};

function checkFile(path) {
    try {
        if (statSync(path).isFile()) {
            pass(path);
            return true;
        }
    } catch {}
    err(`${path} missing`);
    return false;
}
function checkDir(path) {
    try {
        if (statSync(path).isDirectory()) {
            pass(`${path}/`);
            return true;
        }
    } catch {}
    err(`${path}/ missing`);
    return false;
}

say("\n### Template structure — required files");
// TODO: replace with real paths from this project.
const requiredFiles = [
    "CLAUDE.md",
    ".gitignore",
    ".claude/rules/working-principles.md",
    ".claude/rules/git-workflow.md",
    ".agents/AGENTS.md",
    ".agents/active.md",
    "docs/onboarding.md",
    "docs/ai-references.md",
];
requiredFiles.forEach(checkFile);

say("\n### Template structure — required directories");
// TODO: replace with real paths from this project.
const requiredDirs = [
    ".claude/skills",
    ".claude/agents",
    ".claude/commands",
    ".agents/topics",
    ".agents/features",
    ".agents/sessions",
];
requiredDirs.forEach(checkDir);

say("\n### Content checks");

// CLAUDE.md @-imports must resolve
if (existsSync("CLAUDE.md")) {
    const claudeMd = readFileSync("CLAUDE.md", "utf8");
    const imports = claudeMd.match(/@\.(claude|agents)\/[A-Za-z0-9_./-]+/g) || [];
    for (const ref of imports) {
        const path = ref.slice(1); // strip leading @
        if (existsSync(path)) pass(`@${path} resolves`);
        else err(`@${path} broken (referenced in CLAUDE.md)`);
    }
}

// Topics: warn on missing last_updated line
if (existsSync(".agents/topics")) {
    for (const file of readdirSync(".agents/topics")) {
        if (!file.endsWith(".md")) continue;
        const path = join(".agents/topics", file);
        const content = readFileSync(path, "utf8");
        if (!/^last_updated:/m.test(content)) {
            warn(`${path} has no 'last_updated:' line — run /fill-topics`);
        }
        if (/TODO:/i.test(content)) {
            warn(`${path} still has TODO placeholders`);
        }
    }
}

// Long SKILL.md (>100 lines) must have ⚠️ Critical Rules header
if (existsSync(".claude/skills")) {
    const longSkillsWithoutCR = [];
    for (const entry of readdirSync(".claude/skills", { withFileTypes: true })) {
        if (!entry.isDirectory()) continue;
        const skillPath = join(".claude/skills", entry.name, "SKILL.md");
        if (!existsSync(skillPath)) continue;
        const content = readFileSync(skillPath, "utf8");
        const lines = content.split("\n").length;
        if (lines > 100 && !/##\s*⚠️\s*Critical Rules/i.test(content)) {
            longSkillsWithoutCR.push(`${skillPath} (${lines} lines)`);
        }
    }
    if (longSkillsWithoutCR.length === 0) {
        pass(`all long skills (>100 lines) have "⚠️ Critical Rules" header`);
    } else {
        warn(
            `skills >100 lines missing "⚠️ Critical Rules" header:\n      ${longSkillsWithoutCR.join("\n      ")}`,
        );
    }
}

// .gitignore must protect secrets
if (existsSync(".gitignore")) {
    const gi = readFileSync(".gitignore", "utf8");
    if (/^\.env$/m.test(gi)) pass(".gitignore lists '.env'");
    else err(".gitignore must list '.env'");

    if (/^\.claude\/settings\.local\.json$/m.test(gi))
        pass(".gitignore lists '.claude/settings.local.json'");
    else warn(".gitignore should list '.claude/settings.local.json'");
}

// .mcp.json validity (optional)
if (existsSync(".mcp.json")) {
    try {
        const parsed = JSON.parse(readFileSync(".mcp.json", "utf8"));
        const n = Object.keys(parsed.mcpServers || {}).length;
        pass(`.mcp.json is valid JSON (${n} MCP server${n === 1 ? "" : "s"})`);
    } catch (e) {
        err(`.mcp.json invalid JSON: ${e.message}`);
    }
}

say("\n### Summary");
if (failCount > 0) {
    console.log(`${C.red}✗${C.reset} ${failCount} check(s) failed, ${warnCount} warning(s)`);
    process.exit(1);
} else {
    console.log(`${C.green}✓${C.reset} template healthy — ${warnCount} warning(s)`);
    process.exit(0);
}
```

Wire it into three places: `/check-template` (Step 6), the pre-commit hook with `--fast` (Step 10), and CI in full mode as a required check (Step 13) — so drift fails the build instead of waiting for the next manual audit.

#### Recommended extension — known-bad-token guard (catches drift before the next audit)

Per Lesson #13, the highest-value addition is a scan of `.claude/**/*.md` for a **curated list of audit-confirmed mistakes**, so a recurring error fails the build instead of waiting for the next manual audit. The list is project-specific — seed it from your own audit findings. Pattern:

```javascript
// recursive .md finder (reuse for any content scan)
function findMarkdownFilesMatching(dir, pattern) {
    const out = [];
    if (!existsSync(dir)) return out;
    for (const e of readdirSync(dir, { withFileTypes: true })) {
        const p = join(dir, e.name);
        if (e.isDirectory()) out.push(...findMarkdownFilesMatching(p, pattern));
        else if (e.name.endsWith(".md") && pattern.test(readFileSync(p, "utf8"))) out.push(p);
    }
    return out;
}

// Each pattern must be SPECIFIC enough not to match the correct form.
// Seed from audit findings, e.g.: wrong MCP server prefix, removed helper imports,
// non-existent dirs, dead agent references, stale config filenames, frozen snapshots.
const knownBad = [
    { pattern: /mcp__<wrong-server>__/, label: "wrong MCP server prefix", level: "err" },
    { pattern: /<RemovedHelper>\(/, label: "removed helper — use the current API", level: "err" },
    { pattern: /currently empty/, label: "frozen memory snapshot — use a live pointer", level: "warn" },
];
// Gate dead-reference tokens on the file actually being absent, so they self-clear once created:
if (!existsSync(".claude/agents/<some-agent>.md"))
    knownBad.push({ pattern: /<some-agent>/, label: "dead agent reference", level: "err" });

let clean = true;
for (const t of knownBad) {
    const hits = findMarkdownFilesMatching(".claude", t.pattern);
    if (!hits.length) continue;
    clean = false;
    (t.level === "err" ? err : warn)(`${t.label}:\n` + hits.map((p) => "      " + p).join("\n"));
}
if (clean) pass("no known-bad tokens in .claude/");
```

Keep the patterns specific (`/mcp__oracle__/` must not match `mcp__oracledev__`; a capitalized `GetX` must not match the correct `getX`), and **document the guard token-free** (describing the tokens literally in a scanned `.md` would self-flag).

#### More extensions proven useful in practice

- **Scan `.agents/**/*.md` for known-bad tokens too** — but exclude `features/`, `sessions/`, and `private/`: those are historical work logs that legitimately *quote* bad tokens when documenting a fix. Only living guidance (`AGENTS.md`, `active.md`, `topics/`) is scanned.
- **Version sync** — in a workspace/monorepo where releases bump several `package.json` (or equivalent) files together, assert all versions match. Out-of-sync versions are the classic "someone bumped one file" drift.
- **Untracked test files** — warn on `git status` showing untracked `*.test.*` / `*.spec.*` files: TDD scratch tests must be tracked as regression tests or deleted before merge, not left floating.
- **Stale status claims** (full mode only) — status docs like `.agents/active.md` love to say "41 tests passing"; after actually running the suites, grep those docs for `N tests … passing` claims and warn when the number doesn't match the real run. Frozen snapshots that drift into lies are exactly what Lesson #13 is about.

### Bash fallback — `scripts/check-template.sh`

Use this only if the project doesn't have Node. It's intentionally minimal — fewer checks, but no external deps. Note the `done < <(...)` pattern (process substitution) — piping into `while` runs the body in a subshell so counter updates would be lost.

```bash
#!/usr/bin/env bash
# scripts/check-template.sh — minimal drift guard, bash-only.
set -euo pipefail
cd "$(git rev-parse --show-toplevel 2>/dev/null || pwd)"

fail=0; warn=0
log_fail() { echo "FAIL: $*" >&2; fail=$((fail + 1)); }
log_warn() { echo "WARN: $*" >&2; warn=$((warn + 1)); }
log_ok()   { echo "OK:   $*"; }

for f in CLAUDE.md .claude/rules/working-principles.md .claude/rules/git-workflow.md .agents/AGENTS.md; do
  [ -f "$f" ] && log_ok "$f present" || log_fail "$f missing"
done

# Resolve every @-import in CLAUDE.md (process substitution — NOT pipe-into-while)
if [ -f CLAUDE.md ]; then
  while read -r ref; do
    [ -z "$ref" ] && continue
    [ -f "$ref" ] && log_ok "@$ref resolves" || log_fail "@$ref broken (in CLAUDE.md)"
  done < <(grep -oE '@\.(claude|agents)/[A-Za-z0-9_./-]+' CLAUDE.md | sed 's/^@//')
fi

# Topics freshness
if [ -d .agents/topics ]; then
  for f in .agents/topics/*.md; do
    [ -f "$f" ] || continue
    grep -q "^last_updated:" "$f" 2>/dev/null || log_warn "$f missing 'last_updated:' line"
  done
fi

# .gitignore protects secrets
if [ -f .gitignore ]; then
  grep -q '^\.env$' .gitignore || log_fail ".gitignore must list '.env'"
  grep -q '^\.claude/settings\.local\.json$' .gitignore || log_warn ".gitignore should list '.claude/settings.local.json'"
fi

echo
echo "Summary: $fail failure(s), $warn warning(s)"
exit "$fail"
```

Make it executable: `chmod +x scripts/check-template.sh`.

**→ Gate C: show the user the operational layer (Steps 8–11) and confirm before continuing to docs + CLAUDE.md.**

---

## 12. Generate `docs/` skeletons

### `docs/onboarding.md` — for new humans (and new AI sessions without local context)

```markdown
# Onboarding

## Setup

1. Install runtime: <runtime + version> (see `.nvmrc` / `.tool-versions`).
2. Install dependencies: `<pkg-mgr> install`.
3. Copy `.env.example` to `.env` and fill secrets — ask `<who>` for shared values.
4. Run `<pkg-mgr> run dev` and verify <smoke-test URL or output>.

## Repository tour

- `<dir>/` — <one-line>
- ...

## Daily workflow

- Branch from `<integration-branch>`.
- Pre-commit hook runs <tasks> automatically.
- PR target: `<integration-branch>`. PR body in <team-language>.

## Where to ask

- Architecture / domain: `.agents/topics/`
- Conventions: `.claude/rules/`
- Skills (AI playbooks): `.claude/skills/`
- Cross-session AI context: `docs/ai-session-context.md`
```

### `docs/ai-references.md` — pointers AI agents may need

```markdown
# AI References

External repos / plugins / docs the AI should consult before reinventing patterns.

| Reference              | Purpose            | Link                                                   |
| ---------------------- | ------------------ | ------------------------------------------------------ |
| andrej-karpathy-skills | Working principles | https://github.com/forrestchang/andrej-karpathy-skills |
| <internal style guide> | <topic>            | <link>                                                 |
| <vendor SDK>           | <topic>            | <link>                                                 |

# TODO: fill from team knowledge.
```

### `docs/ai-session-context.md` — cross-session continuity

For AI sessions started without local memory (a fresh checkout, a different machine, a CI agent). Mirror the durable parts of your local memory here.

```markdown
# AI Session Context

> Snapshot of cross-session knowledge for AI agents that don't have local memory.
> Update this file when long-lived facts change. Ephemeral state (current task, in-flight branch) does NOT belong here — that's `.agents/active.md`.

## User / team preferences

- Team language: <language>
- Review style: <e.g. "Thai PR body, English commit subject">
- Strict no-bypass: <e.g. "never use --no-verify on git commits">

## Project status

- Current phase: <e.g. "v1.x stable", "migrating from X to Y">
- Active initiatives: <one-line each, with owner if known>

## Working framework

- <pointer to Working Principles>
- <pointer to any team-specific framework>

# TODO: fill from team knowledge. Keep under 200 lines so it fits in context.
```

---

## 13. CI/CD deploy workflow (optional)

Only scaffold this if the user said yes during pre-flight. The shape depends on deploy target.

### Generic GitHub Actions skeleton

Two-job shape: **`verify` runs lint+typecheck+tests, `deploy` runs only if verify passes**. Splitting the gate from the deploy guarantees that local pre-commit bypasses (`--no-verify`) can't ship broken code, and that re-running deploy without re-verifying is a deliberate choice.

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
    push:
        branches: [main]
    workflow_dispatch:

jobs:
    verify:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v4
            - uses: <setup-action>
              with:
                  <runtime>-version-file: <pinning-file>
            - run: <pkg-mgr> install --frozen-lockfile
            # Generated artifacts (OpenAPI routes, router trees, codegen output) are
            # usually gitignored — REGENERATE them before type-checking, or the
            # typecheck fails on a fresh checkout with unresolved imports even
            # though it passes on every dev machine.
            - run: <codegen-cmd> # e.g. OpenAPI route gen, GraphQL codegen, router tree build
            - run: node scripts/check-template.mjs # AI-scaffold drift guard gates the PR (Step 11)
            - run: <pkg-mgr> run lint
            - run: <typechecker-cmd> # e.g. tsc --noEmit, mypy, go vet
            - run: <pkg-mgr> test
            # Don't pass --silent to test runners in CI — failure output gets
            # suppressed and debugging from logs becomes painful.

    deploy:
        runs-on: <runner>
        needs: verify
        steps:
            - uses: actions/checkout@v4
            - uses: <setup-action>
              with:
                  <runtime>-version-file: <pinning-file>
            - run: <pkg-mgr> install --frozen-lockfile
            - run: <pkg-mgr> run build
            - name: Deploy
              env:
                  <SECRET_VAR>: ${{ secrets.<SECRET_VAR> }}
              run: <deploy-cmd>
```

### Common targets — quick pointers

| Target            | Approach                                                                             |
| ----------------- | ------------------------------------------------------------------------------------ |
| Vercel            | use `vercel` CLI or built-in Git integration; no workflow file needed                |
| Self-hosted (PM2) | runner = self-hosted; deploy step = `pm2 reload`                                     |
| IIS (Windows)     | runner = self-hosted Windows; deploy step = robocopy + `iisreset` / app pool recycle |
| Docker registry   | `docker buildx` + push + remote pull                                                 |
| Kubernetes        | `kubectl set image` or ArgoCD sync                                                   |
| Serverless        | `serverless deploy` / `sam deploy`                                                   |

List **every secret** the workflow uses in `.env.example` (Step 9) and in `docs/onboarding.md` so the team knows what to set in GitHub Actions secrets.

---

## 14. Generate `CLAUDE.md` at the repo root

Generated **last** so every `@`-import resolves to a real file by the time CLAUDE.md is written.

Short by design — long-form details live in `.claude/rules/*.md` (imported via `@` syntax).

```markdown
# CLAUDE.md (<project-name>)

> Root project guide — kept short; details delegated to `.claude/rules/*.md` and skills.

## At a glance

<one-paragraph stack summary — TODO: fill from pre-flight>
e.g. "<framework> + <language> + <db>. <key architectural choice>. Shared types in <path> if applicable."

## Commands

| Command           | What it does               |
| ----------------- | -------------------------- |
| `<run-dev>`       | start dev server           |
| `<run-test>`      | run tests                  |
| `<run-lint>`      | lint                       |
| `<run-build>`     | build                      |
| `<run-typecheck>` | type-check (if applicable) |

# TODO: fill in real commands from package.json / Makefile / etc.

## Rules — read before coding

@.claude/rules/working-principles.md
@.claude/rules/backend.md
@.claude/rules/frontend.md
@.claude/rules/types.md
@.claude/rules/testing.md
@.claude/rules/git-workflow.md
@.agents/AGENTS.md

# Remove the @-imports above for files you didn't create.
# KEEP @.agents/AGENTS.md — its rules (evidence-gate, 2-strike, never-guess, reward-hacking)
# only take effect when @-imported here. A plain markdown link is NOT auto-loaded into context.

## Skills (`.claude/skills/`)

| Skill                 | Use when                           |
| --------------------- | ---------------------------------- |
| `<stack>-development` | starting a new endpoint / feature  |
| `<db>-schema-cache`   | verifying DB schema before queries |
| `design-system`       | read BEFORE writing UI             |

# TODO: fill table from Step 5.

## Domain knowledge (AI must never guess)

Business-rule files live in `.agents/topics/`. Read these before answering anything domain-specific:

- `.agents/topics/domain.md` — vocabulary, statuses, workflow rules
- `.agents/topics/data-schema.md` — entity relations, soft-delete, audit columns
- `.agents/topics/auth-flow.md` — identity, session, multi-tenancy
- `.agents/topics/deployment.md` — environments, secrets, release process

Universal AI rules (Claude / Cursor / Copilot / …) live in `.agents/AGENTS.md` — `@`-imported above so they load for Claude Code; other tools read the file directly.
Current state / in-flight work: `.agents/active.md`.

## Onboarding

Long-form walkthrough: [`docs/onboarding.md`](docs/onboarding.md)
Cross-session context (for AI without local memory): [`docs/ai-session-context.md`](docs/ai-session-context.md)

## Language policy (summary)

- **Code, identifiers, commits, PR titles, tool output**: English.
- **PR body, CHANGELOG, review summaries to team**: <team-language — TODO from pre-flight>.
- Full table: see `.claude/rules/git-workflow.md`.
```

---

## 15. Verify

Final checks before declaring the install complete:

1. `ls -la .claude/ .agents/` — all expected folders exist.
2. Open `CLAUDE.md` — every `@.claude/rules/<file>.md` import resolves to a real file.
3. Open each `<file>.md` you generated — confirm there is **no leftover `<placeholder>`** that should have been filled.
4. Run `node scripts/check-template.mjs` (or `bash scripts/check-template.sh`) — should report 0 failures (warnings OK on a fresh install).
5. Trigger a hook — make a tiny edit to a file and watch the formatter/typechecker run.
6. Verify `.gitignore` excludes `.env`, `.claude/settings.local.json`, and `.agents/private/`.
7. Print a summary table to the user: what was created, what was skipped (and why), what `# TODO` markers remain.

**→ Gate D: present what was created and ask whether to commit.** Suggested first commit:

```
chore(ai): bootstrap Claude Code collaboration scaffold

Generated CLAUDE.md, .claude/{rules,skills,commands,agents}/, .agents/{AGENTS.md,active.md,topics,features,sessions}/,
schema cache, hooks, drift-guard, and pre-commit configuration adapted to <stack>.

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## Notes for the installing AI

- **Ask before destructive operations.** Never overwrite an existing `CLAUDE.md`, `.husky/`, `package.json` script, or `.pre-commit-config.yaml` without explicit confirmation.
- **Incremental commits.** One commit per batch (Gates A→D) is a reasonable default — easier to review and revert.
- **Don't push, don't open PRs** unless the user explicitly asks.
- **Don't invent business rules.** Anything domain-specific (statuses, audit columns, soft-delete convention, multi-tenancy) goes into `.agents/topics/*.md` with `TODO: verify` markers — to be filled later via `/fill-topics`.
- **Adapt, don't copy-paste.** The template above is a _shape_. Replace every `<placeholder>` and remove every section that doesn't apply to this project (e.g. drop `frontend.md` for a CLI; drop `types.md` if there's only one language; drop schema cache if there's no DB).
- **When uncertain, stop and ask.** A short clarifying question is cheaper than regenerating a wrong scaffold.
- **Keep `CLAUDE.md` short.** All depth lives in rule/skill files. The root file is just an index + a 3-line language summary that points to `git-workflow.md`.

---

## Lessons learned from real adoption — patterns that bite

These are real failures observed when the scaffold above was applied to a working Node + React + Oracle codebase. Worth checking for in any project that adopts this kit.

### 1. Singletons created at module-import time read the wrong config

A `Logger.getInstance()` evaluated at module top-level captures `config.LOG_LEVEL` / `config.WS_LOG_SERVER_URL` _before_ `initConfig()` populates them. Subsequent `setLogLevel()` calls work, but the defaults silently wrong out the rest of the lifetime. **Pattern**: make singletons read config via getters (lazy), not in the constructor. Same applies to any module-level service that depends on env-driven config.

### 2. Identity-injecting middleware is an anti-pattern with strict request schemas

A middleware that decodes JWT and writes `userId` / `orgId` into `req.body` looks ergonomic but breaks the moment the API framework rejects unknown body keys (TSOA `noImplicitAdditionalProperties`, FastAPI strict models, Spring `@JsonIgnoreProperties`). Worse, it usually relies on an unverified decode (no signature check). **Rule**: identity comes from the request context (`req.user` populated by the auth middleware after signature verification), never from body. Request DTOs should contain only client-mutable fields.

### 3. Auto-commit-off pools must commit on success too

Connection pools that disable auto-commit (Oracle 11g, certain Postgres setups) require _explicit_ commit on every successful DML. A success-path `return result;` without `await connection.commit()` looks fine, type-checks, passes basic tests — and silently rolls back every batch when the pool returns the connection. Add a smoke test that asserts `rowsAffected > 0` _and_ re-reads the row to confirm it persisted.

### 4. Aliased component exports cause docs/code drift

Exporting two names for the same component (`FormItem` and `FormFieldItem` rendering identical JSX) feels harmless. It isn't: docs pick one name, callers pick the other, and AI agents copy from whichever they read first. Pick a single canonical name in _code, docs, skill templates, and onboarding examples_ — and grep before merging to confirm there's only one.

### 5. Type tests catch what runtime tests can't

A `value as any` in a shared form/table primitive can mask a contract violation that nothing crashes on (e.g. `formState.errors[name]` always returning `undefined` because the context was the wrong shape). The TypeScript escape hatch hides the bug from both the compiler and test suite. **Rule**: treat `as any` in shared components as a P1 finding, not a style nit.

### 6. Dead deps with CVEs add up faster than you expect

A 12-month-old commented-out import (`// import { initSequelize }...`) leaves the dep in `package.json` and its CVEs in `npm audit`. Lint won't catch it. **Rule**: when commenting out an import, also remove the dep from `package.json`. Add a CI step that fails if `npm audit --audit-level=high` reports anything new.

### 7. "Fix the rule, not the code" is a real mode

A surprising number of findings end up being _the rule was wrong, the code was right_: `BaseRow` saying "every important table extends this" while real DB has 4+ different audit conventions, or `backend.md` telling AI to wrap with `asyncErrorWrapper` when TSOA already propagates errors. **Always verify both sides** before fixing. Sometimes the fix is rewording the rule and deleting the unused helper.

### 8. Schema verification has to be a _gate_, not a _guideline_

Telling the AI "check `<schema-dir>/<TABLE>.md` before writing SQL" in a rule file does not stop guessing. Adding the PreToolUse schema-check hook (Section 8) does — because it surfaces the reminder _at the moment_ the AI is about to edit a repository file. Rules that live in static markdown lose to context pressure; hooks fire at the relevant moment.

### 9. Docs drift compounds — three directions to grep before merging

Whenever a name, path, or script is renamed, drift can hide in:

- **Code** (the obvious one — covered by typecheck if the name is exported)
- **Skill files / templates** (`.claude/skills/<name>/templates/*.md` — AI copies these verbatim)
- **Onboarding docs** (the human walkthrough — copy/paste targets)

A reviewer pass that searches all three is mandatory. Two reviewer agents in parallel doing this catch about twice as much as one (verified empirically — the first agent missed 3 stale references that the second caught).

### 10. Rules in `.agents/AGENTS.md` don't load unless `@`-imported

`AGENTS.md` is the vendor-neutral rule file, but Claude Code only auto-loads what `CLAUDE.md` `@`-imports. Linking to it ("see `.agents/AGENTS.md`") leaves its rules — evidence-gate, 2-strike escalation, reward-hacking guards — **dormant**: the agent obeys them only if it happens to open the file. **Rule**: `@`-import `.agents/AGENTS.md` in CLAUDE.md (Step 14), and broaden the drift-guard's import check to validate `@.agents/...` too (Step 11). This is the same "gate not guideline" lesson as #8 — a rule only fires if it's actually in context.

### 11. One planning command isn't enough — cover the work lifecycle

`/new-feature` only fits greenfield work. Real maintenance is mostly *corrective* (a bug, a wrong date format, a duplicated component) and *evolutive* (add a field, change a calc) — different processes that a "plan a new feature" prompt handles badly. Ship a small lifecycle set: `/fix` (reproduce-first), `/enhance` (append to the existing feature ledger), `/refactor` (behavior-preserving), `/investigate` (read-only RCA), plus `/checkpoint` for session handoff. Route work-type → command in AGENTS.md so the AI picks the right entry point instead of forcing every task through feature-planning.

### 12. A review agent must bound its work to the diff, or it dies mid-review

The most common reliability failure for the `code-reviewer` subagent is dying part-way through and returning a partial review with **no verdict** — which the orchestrator then misreads as "nothing flagged → READY", shipping unreviewed code. The root cause is _not_ the turn budget: it's an instruction like *"read each changed file completely"*, which makes turn/context cost grow linearly with diff size. Raising `maxTurns` only moves the cliff — a bigger diff still hits it. Two compounding traps: tracked **lockfiles** (`*-lock.json`, `pnpm-lock.yaml` — often 5k–11k lines each) and **generated files** (OpenAPI / GraphQL output) the agent is told to "read completely" but that have nothing to review. **Fix**: (a) the orchestrator pre-collects the diff **into a file** (`git diff HEAD > /tmp/review.diff`) and hands the *path* over (analyze, don't discover) — a subagent running `git diff` itself gets the output truncated at the tool layer, and the mangled diff forces it back into per-file re-reads, which is the exact failure being prevented; (b) exclude lockfiles + generated files before handoff; (c) read diff-scoped, not whole-file; (d) add a size guard that refuses oversized diffs honestly; (e) treat a returned review with no `Verdict:` line as `NEEDS WORK`, never `READY`; (f) put **budget discipline in the prompt** — reserve the last ~25% of turns for the report, and prefer a partial review with an honest verdict over silent death. This is the same "gate not guideline" lesson as #8 — a fixed `maxTurns` is a backstop, not a reliability mechanism.

### 13. Agent/skill prompts drift from the codebase — audit them against the real repo periodically

Prompts are documentation that nothing compiles, so they rot silently. A single full audit of a working scaffold surfaced ~40 verified defects, almost all one root cause: **prompts hard-coding mutable conventions instead of pointing at the source-of-truth rule/skill files.** The high-severity classes to grep for specifically:

- **Tool / MCP names in `tools:` frontmatter that don't exist.** An agent listing `mcp__oracle__getOracleTableSchema` when the real server is `mcp__oracledev__*` (or wrong casing like `GetDataBySql` vs `getDataBySql`) does **not** error — the tool just silently never loads, so the agent quietly has no DB access. Verify every `mcp__*` token and `Agent(<name>)` against the servers/agents that actually exist.
- **Referenced agents/files/dirs that don't exist.** A `nextjs-builder` delegate or a `/server/sql/` path in a Vite/`src/sqltabs/` repo; example imports (`import { oracle }` when the export is `getOracle()`) that won't compile when copied.
- **Stale conventions the codebase has since changed.** ID strategy (sequences → app-generated), grants (a new read-only role), package manager (`npm` → `pnpm`), file types (`.js` → `.cjs`), a value flag (`Y` vs `F`/`T`) — every place a prompt restates a value is a future drift point.
- **Embedded snapshots that go stale**, e.g. a hard-coded "MEMORY.md is currently empty" line that becomes a lie once entries exist → replace with a live pointer, never a copy.

**Fixes**: (a) prefer pointers over restated values — "see `.claude/rules/backend.md`" beats duplicating the rule; (b) extend the drift-guard (Step 11) to scan `.claude/**/*.md` for known-bad path/tool tokens; (c) re-run a prompt-vs-codebase audit whenever the stack shifts (package manager, framework, schema convention). Same "gate not guideline" theme as #8/#12: a convention only stays correct if there's a single place that owns it.

### 14. A review loop without a verify step (and without an owner) isn't closed

Three gaps showed up only after the review pipeline had been running for a while:

- **Review ≠ verify.** `code-reviewer` proves the code *reads* correctly; nobody was proving it *behaves* correctly. A reviewed-READY change can still fail its own acceptance criteria. Fix: a fourth read-only agent (`qa-tester`) that runs AFTER READY and executes the ledger's acceptance criteria against the real system (suites → e2e if a server is up → browser smoke), reporting PASS/FAIL/SKIP per criterion with raw output. SKIP-with-reason is allowed; silent PASS is not.
- **Nobody owned the fix rounds.** After `NEEDS WORK`, the orchestrator would stop and ask the user what to do — turning an automatable loop into a human bottleneck. Fix: the orchestrator owns it — send P0/P1s back to the builder, re-review **automatically**, max 2 rounds, then escalate with a summary of options (mirrors the 2-strike rule).
- **Heavy process invites wholesale bypass.** If a 1-line fix requires spec.md + ledger.md, agents (and humans) start skipping the process entirely. Fix: an explicit **fast lane** with hard entry conditions (≤5 lines, 1 file, existing test coverage) where commit message + test output are the evidence — plus a rule that big work must not be sliced into fast-lane pieces to dodge the process. A documented escape valve beats undocumented evasion.

### 15. The gates themselves have failure modes — wire them so they actually gate

Three real ways a "passing" pipeline shipped a failure anyway:

- **Pre-commit steps on separate lines don't block.** A hook script exits with the *last* command's status, so `lint-staged` could fail and the commit still landed because the drift guard after it passed. Chain with `&&` (Step 10).
- **CI type-check fails (or silently diverges) on fresh checkouts** when generated artifacts (OpenAPI routes, router trees, codegen output) are gitignored — every dev machine has them, CI doesn't. Regenerate them in the verify job *before* type-checking (Step 13).
- **A slow gate is a skipped gate.** The full drift guard ran typecheck+lint+tests, too slow for every commit — so it wasn't run on every commit. Adding `--fast` (structure + drift only) made per-commit gating cheap, with the full mode reserved for CI (Step 11).

---

## Footer — provenance

This scaffold is distilled from a Node + React + Oracle template that used:

- Layered architecture (Controller → Service → Repository) on the server
- A canonical UI form/table/toast/data-fetching stack on the client
- A schema-cache pattern with an MCP catalog as source of truth
- A team that wrote PR bodies and CHANGELOGs in their local language but kept code/commits in English
- Slash commands for planning (`/new-feature`), review (`/review-pr`), onboarding (`/onboard`), domain-knowledge interview (`/fill-topics`), and drift detection (`/check-template`)
- Subagents simulating a 4-member team — design-spec, builders, a proactive code-reviewer, and a read-only qa-tester verify step (design → build → review → test)
- A 2-agent reviewer swarm (parallel cross-check) used after every implementation batch — catches stale references and drift that a single reviewer misses
- Working principles distilled from **andrej-karpathy-skills** ([forrestchang fork](https://github.com/forrestchang/andrej-karpathy-skills) / `multica-ai/andrej-karpathy-skills`) — think-before-coding, surgical changes, never-guess-business-rules, verify-sources, honest-reporting

Those project-specifics were intentionally **stripped** from the prompt above — what remains is the _pattern_. Replace the placeholders with whatever this project actually uses. The Karpathy reference is kept because the principles themselves are stack-agnostic and worth importing wholesale.

The "Lessons learned from real adoption" section above was added after a full review-and-fix cycle on that template (~37 P0+P1 fixes across 13 commits). Those failure modes are the ones most likely to bite the next adopter, so they got promoted from "things we noticed" to "things to actively guard against".
