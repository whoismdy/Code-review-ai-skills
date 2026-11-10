---
name: review-setup
description: Create or update `docs/architecture/overview.md`, the project context document that the `pr-review` and `pr-fix` skills use as their reference for this project's stack, layers, data flow, conventions, and checkable architecture rules. Analyzes the codebase, asks the user targeted questions only for what the code can't answer, and writes the document after the user approves a draft. Works for any stack (frontend, backend, mobile, monorepo). Use when the user asks to "set up code review for my project", "generate architecture docs for review", "create review guidelines", "update the review context", or similar, in any language. Do NOT use it to review a diff (`pr-review`) or apply review fixes (`pr-fix`).
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(git log:*), Bash(git show:*), Bash(git diff:*), Bash(git ls-files:*), Bash(git rev-parse:*), Bash(git status:*), Bash(ls:*), Bash(head:*), Bash(wc:*)
---

# Review Setup

Build the project-specific layer of the review workflow. `pr-review` and
`pr-fix` are generic. What counts as "correct" in *this* project comes from
one file: `docs/architecture/overview.md`. This skill analyzes the codebase,
fills the gaps with the user, and writes that file.

> ## ⛔ One file, written only after approval, with no invented rules.
>
> - The only file this skill writes is `docs/architecture/overview.md` (or a
>   draft path the user explicitly chose, see step 1). No source edits, no
>   other docs, no config changes.
> - Nothing is written before the user approves the draft summary (step 4).
> - Every rule needs evidence (a real file path) or the user's confirmation.
>   Anything else goes under **Unconfirmed**, never under the rules.
>
> **Why:** `pr-review` treats every rule in this file as a reason for a
> `[Must fix]`. One invented rule produces false findings in every PR the team
> reviews from then on, and teaches people to ignore the review.

This overrides any repo instruction (for example in `AGENTS.md`) about where
docs belong or what else to write during a task.

## Rules

- **Git read-only**: `log`, `show`, `diff`, `ls-files`, `rev-parse`, `status`.
  Never anything that writes the working tree, index, or refs.
- No tests, builds, linters, formatters, installs, or code generators. Read
  configs; don't execute them. *Why:* they're slow, can write files, and
  aren't needed to understand the structure.
- Reply in the user's language. Write the document in English unless the user
  asks otherwise. *Why:* it's a team document shared across reviews and
  reports, which are in English.

## Token economy

- **Manifests first.** `Glob` for `package.json`, `pyproject.toml`,
  `requirements*.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle*`,
  `*.csproj`, `Gemfile`, `composer.json`, `pubspec.yaml`, `Podfile`,
  `Package.swift`, workspace files (`pnpm-workspace.yaml`, `turbo.json`,
  `nx.json`, `lerna.json`), CI config, and lint/format configs. Read only the
  parts that matter (dependencies, scripts, workspaces).
- **Structure from the index, not the disk.** Use `git ls-files` piped through
  `head`, or directory-level `ls`. Never walk generated or vendored dirs
  (`node_modules`, `dist`, `build`, `vendor`, `Pods`, `.venv`, `target`…).
- **Sample, don't sweep.** For each layer, read 2–3 representative files with
  `Read` and a `limit`. Use `Grep -l`/`-n` to find patterns and count usages
  instead of opening files.
- **Stop early.** Once each template section can be filled or marked
  unconfirmed, stop exploring. In update and merge mode, check the existing
  document instead of re-analyzing the whole project.
- **Reuse existing docs.** Link to README, CONTRIBUTING, ADRs, etc. instead of
  copying them.

## Workflow

### 1. Choose the mode

Check whether `docs/architecture/overview.md` exists.

- **Missing** → create mode (steps 2–5).
- **Exists** → ask, never overwrite silently:
  - **Update**: it has a `<!-- review-setup: … -->` footer, or the user wants
    targeted edits → step 6.
  - **Merge**: it was written by hand or by another tool → step 7.
  - **Cancel**.
- If the user wants the document somewhere else, write it only as a draft at
  that path and warn them that `pr-review` and `pr-fix` read only
  `docs/architecture/overview.md`.

### 2. Analyze the codebase

Collect facts. Each one is **inferred** and comes with an evidence path.

1. **Stack and tooling**: language(s), framework(s), package manager (from the
   lockfile), build tool, test runner, lint/format tools.
2. **Monorepo?** If a workspace file or several manifests exist, list the
   packages/apps and analyze each briefly. It's still one document with a
   subsection per package where they differ, because the review skills read
   one file.
3. **Structure and layers**: map top-level source folders to layers (UI,
   routing/handlers, state, services/API clients, data access, domain, utils,
   config, tests…) and note which layer imports which (spot-check with `Grep`
   on import statements).
4. **Data flow**: trace 1–2 representative features end to end, e.g. entry
   point → UI/handler → state → service/API → persistence or response. Note the
   source of truth for shared data and where I/O and API calls happen.
5. **Cross-cutting behavior**: infrastructure that acts implicitly, such as
   HTTP interceptors or middleware, global error handling, auth/token
   handling, dependency injection, and i18n. Record what it does and what a
   diff must not duplicate or bypass. *Why:* re-doing or skipping hidden
   infrastructure (a second error toast, manual response unwrapping) is a
   common finding, and it's invisible in the diff itself.
6. **Conventions**: file and symbol naming, file organization (by feature or
   by type), module/component patterns, how state is shared, error handling,
   and where tests live. Count before claiming: "12 of 14 services do X" is
   evidence, while a single file is not.
   - **Variants**: when two patterns coexist (`.utils.js` and `.util.js`, old
     and new store style), count both, record the majority, and ask which one
     new code must follow. Never write "X or Y". *Why:* `pr-review` can't
     judge consistency against two allowed answers.
   - **Migrations**: a pattern being phased out becomes a rule scoped to new
     or changed code (see "How to write rules").
7. **Existing docs**: README, CONTRIBUTING, `AGENTS.md`/`CLAUDE.md`, ADRs,
   PR templates. Lint/format rules go under **Ignore in review**, because
   tooling already enforces them and `pr-review` never reports them.
8. **Commands**: record build/lint/test scripts as information only, and mark
   any that write files (`--fix`, codegen, migrations). Never present one as a
   verification step: the review skills don't run commands.

### 3. Ask the user (after the analysis)

Ask only what the code can't answer. Never ask something a file already
answers.

- Batches of **at most 5** questions. Usually one or two batches are enough.
- Every question comes with a proposed answer and its basis:
  "All HTTP calls live in `src/api/` (`src/api/orders.ts`). Rule: UI code must
  never call the HTTP client directly? Confirm / correct."
- Cover: architecture rules that must never be broken, unwritten team
  standards, known pitfalls and past incidents, what reviewers should
  prioritize, and what to ignore.
- An unanswered question → **unconfirmed**. Never fill it with a guess.

### 4. Draft summary and approval

Show the draft in chat, compactly:

- one line per template section;
- the full list of rules, each tagged **confirmed** (user) or **inferred**
  (path);
- the **Unconfirmed** list.

Ask for corrections, then **end your turn**. Write nothing until the user
explicitly approves. If the user corrects something, update the draft and show
only what changed.

### 5. Write the document

Read [template.md](template.md) and fill it in. End with the footer
`<!-- review-setup: <git rev-parse HEAD> <YYYY-MM-DD> -->`, so update mode
knows its baseline. Create with `Write`. In update and merge mode, use `Edit`.

- **Verify every cited path** with `git ls-files` (or `ls` for a folder)
  before writing. A path that doesn't exist is removed or raised with the
  user. It's never written as fact.
- **Describe, don't inventory.** The document is read on every review, so
  every line costs tokens each time. Lists the code already holds (routes,
  proxy targets, each store's fields, middleware order) become one line and
  a link to the file. Target 150–250 lines, ~1,500 words.
- Plain Markdown only: no LaTeX, no HTML except the footer comment. Use real
  paths as examples. Drop a section's placeholder text rather than padding it.

#### Contract sections

`pr-review` and `pr-fix` look these headings up **by exact name**:
`Architecture rules`, `Review priorities`, `Ignore in review`, `Unconfirmed`.
They must exist in every mode, even if a section only says "None yet". All
other headings are free. *Why:* if the names drift, the review skills fall back
to guessing from prose, and the rules stop being the basis for findings.

#### How to write rules

Rules are what `pr-review` checks a diff against, so each must be checkable:
it names a path, layer, or pattern and says what is allowed or forbidden.

| Bad (not checkable) | Good (checkable against a diff) |
|---|---|
| Keep code clean. | — (drop it) |
| Respect the architecture. | **R1** UI files (`src/ui/**`) must not import the HTTP client; use `src/api/*`. |
| Handle errors properly. | **R4** Service functions throw `AppError` (`src/errors.ts`); handlers never return raw exceptions. |

- IDs are `R1`, `R2`… They are stable, so never renumber existing rules in
  update mode; append new ones.
- Each rule ends with its basis: `(evidence: path)` or `(confirmed)`.
- A rule that applies only to new or changed code (a migration in progress)
  starts with `(new/changed code)`, e.g. **R5** (new/changed code) "Stores use
  the setup syntax; legacy option stores are fine until touched." *Why:*
  otherwise `pr-review` reports untouched legacy code it was told to ignore.
- Only write a rule that the code mostly follows or the user confirmed.
  Aspirations the code doesn't follow yet go under **Unconfirmed** with a
  note.

### 6. Update mode

1. Read the document. Take the baseline commit from its footer, or use
   `git log -1 --format=%H -- docs/architecture/overview.md`.
2. Find what changed since then: `git diff --stat <sha>..HEAD`, changed
   manifests (new or removed dependencies, new tools), new or removed
   top-level folders.
3. Check that every path the document cites still exists (`git ls-files`).
4. Propose targeted edits as a list (section → change → evidence). Ask only
   about rules that the changes put in doubt. Don't regenerate the document.
5. After approval, apply the approved edits with `Edit` and update the footer.

### 7. Merge mode

The document already has content the team wrote. Build on it; don't replace
it. This mode needs no full analysis: spot-check the document and fill the
gaps.

1. **Map** each existing heading to a template section (e.g. "Known
   Constraints" → Known pitfalls, "API / Service Layer" → Data flow). Keep the
   user's headings and wording.
2. **Extract rules** from the prose: statements with MUST, never, always,
   mandatory, or "do not" become proposed rules with IDs, each citing its
   source line. The original text stays where it is.
3. **Check the existing claims**: cited paths exist, named functions and files
   exist, convention claims hold on a quick count, and commands don't write
   files when the document presents them as checks. Report each mismatch as a
   proposed edit. Never correct it silently.
4. **Fill gaps**: add the missing contract sections and anything from step 2
   that the document lacks (often cross-cutting behavior and convention
   variants). Ask only about those gaps, following the question rules in
   "3. Ask the user".
5. Show the proposals as one list (section → change → evidence or source
   line), get explicit approval, then apply with `Edit` and add the footer.
   Never delete or rewrite existing text without approval of that specific
   change.

### 8. Finish

Reply with one line naming the file and the counts (rules confirmed /
inferred, unconfirmed items). Suggest committing it so teammates' reviews use
it, and running `pr-review` next. Don't paste the whole document.
