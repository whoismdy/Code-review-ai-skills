# PR Review Skills

Three Claude Code skills that review a branch, fix the findings you approve,
and learn your project's architecture once so every review is project-aware.
They work with any stack: frontend, backend, mobile, or monorepo.

| Skill          | What it does                                                                                                                   | Writes                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------- |
| `review-setup` | Analyzes the codebase, asks you a few targeted questions, and creates or updates the project context document.                 | `docs/architecture/overview.md` only, after you approve a draft |
| `pr-review`    | Reviews what one branch would bring into another and reports problems with short fix hints.                                    | One report file: `report-<source>-<target>.md`                  |
| `pr-fix`       | Checks each finding in a report against the current code and applies fixes **one at a time, only after you approve each one**. | Only the code changes you approved                              |

None of them commits, stages, switches branches, or runs tests, builds,
linters, or installs.

## How they fit together

```
review-setup ──► docs/architecture/overview.md   (once per project, then occasional updates)
                        │
                        ▼ read as the project's rules
pr-review ─────► report-<source>-<target>.md    (per branch)
                        │
                        ▼ parsed finding by finding
pr-fix ────────► approved code changes          (per finding, with your yes)
```

- The skills themselves are generic. Everything project-specific lives in
  `docs/architecture/overview.md`.
- `pr-fix` depends on the exact report format `pr-review` produces. Change
  one, check the other.

## Installation

Copy the three folders into a skills directory:

- **One project**: `<repo>/.claude/skills/`. Commit them so the whole team
  gets them.
- **All your projects**: `~/.claude/skills/`.

```
.claude/skills/
├── README.md
├── pr-review/SKILL.md
├── pr-fix/SKILL.md
└── review-setup/
    ├── SKILL.md
    └── template.md
```

Requirements: Claude Code and a git repository. The source and target
branches must exist locally or as `origin/` refs. The skills never fetch.

## Quick start

1. **Set up the project context (once):**

   > set up code review for my project

   `review-setup` reads manifests, folder structure, and a few representative
   files. It then asks up to 5 questions at a time, each with a proposed
   answer, and shows a draft. It writes `docs/architecture/overview.md` only
   after you approve. Commit that file so teammates' reviews use it too.

2. **Review a branch:**

   > review feature/order-discounts into develop

   Both branches are required. If one is missing, the skill asks instead of
   guessing. The result is `report-feature-order-discounts-develop.md` in the
   repo root.

3. **Fix findings:**

   > fix the issues in report-feature-order-discounts-develop.md

   `pr-fix` lists the findings, asks which ones to work on, then for each one:
   verifies it → proposes `FIX-01` → **waits for your yes** → applies → checks
   the diff → suggests a Conventional Commits message.

You can run `pr-review` without step 1. It falls back to the patterns in the
surrounding code and suggests running `review-setup`.

## The context document

`docs/architecture/overview.md` is the source of truth for what "correct"
means in your project. Its sections are listed in
[review-setup/template.md](review-setup/template.md).

The review skills look up these headings **by exact name**. Keep them, even if
one only says "None yet":

| Heading              | How the skills use it                                                                               |
| -------------------- | --------------------------------------------------------------------------------------------------- |
| `Architecture rules` | Rules `R1`, `R2`… are the basis for architecture findings, and reports may cite them ("breaks R3"). |
| `Review priorities`  | Decides which findings matter most.                                                                 |
| `Ignore in review`   | Never reported.                                                                                     |
| `Unconfirmed`        | Hints only, never grounds for a `[Must fix]`.                                                       |

Good rules are checkable against a diff:

- ✗ "Respect the architecture."
- ✓ "**R1** UI files under `src/ui/**` must not import the HTTP client; use `src/api/*`."

Rules can be scoped with `(new/changed code)` for migrations in progress, so
untouched legacy code isn't flagged.

**Keeping it current:** run "update the review context". The document ends
with a `<!-- review-setup: <sha> <date> -->` marker. Update mode diffs from
that commit and proposes targeted edits instead of regenerating the document.
A hand-written document is handled in merge mode: your text is kept, rules
are extracted from it, and its claims are checked against the code.

## Report format

```
## PR Review: <source> → <target>

**Overview:** At most 2 sentences.
**Summary:** <n> Must fix · <n> Should fix · <n> Minor
> Notes: optional one-line caveat

### Readability
- [Must fix|Should fix|Minor] Short issue title
  Where: `path/to/file:LINE`
  Code: `exact original line` ⏎ `optional second line`
  Problem: one line
  Hint: one line

### Architecture & Data Flow
### Performance
### Consistency
```

- All four category headings always appear. An empty one says `No issues found.`
- `Code:` is optional. It quotes the original source exactly, so `pr-fix` can
  find the spot with a plain-text search and tell whether the code has changed
  since the review.
- Reports are always in English. Branch names are sanitized in the filename
  (`feature/#588-x` → `feature-588-x`). The header keeps the exact names.
- Formatting and lint issues are never reported, since tooling handles them.

## Safety guarantees

|                                  | `review-setup`                 | `pr-review` | `pr-fix`                           |
| -------------------------------- | ------------------------------ | ----------- | ---------------------------------- |
| Edits project source             | never                          | never       | only approved fixes                |
| Files it writes                  | `overview.md` (after approval) | the report  | approved files only                |
| Git                              | read-only                      | read-only   | read-only, never stages or commits |
| Tests, builds, linters, installs | never                          | never       | never                              |

`pr-fix` approval rules:

- One explicit "yes" covers **one** fix ID.
- None of these count as approval: "fix everything", a `[Must fix]` label, a
  trivial-looking change, or silence.
- If a fix turns out to need more than it proposed, it stops and asks again.
- It never touches lines that hold your uncommitted changes without asking.

Each skill's `allowed-tools` pre-approves reading tools and read-only git
commands, plus write tools only where the skill's single output needs them:

- `Write` in `pr-review`, for the report.
- `Write` and `Edit` in `review-setup`, for the context document.

`pr-fix` deliberately pre-approves neither, so Claude Code's own permission
prompt is a second safeguard for every code edit.

## Customizing

- **Project rules, priorities, and what to ignore**: edit
  `docs/architecture/overview.md`, not the skills.
- **Report format**: change `pr-review` and `pr-fix` together.
  `pr-fix` §3 describes what it parses.
- **Language**: skills reply in your language. Reports and the context
  document are written in English for team consistency.

## Troubleshooting

| Symptom                                | Cause / fix                                                                                                       |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Review ignores your rules              | Check that `docs/architecture/overview.md` exists at the repo root and the contract headings are spelled exactly. |
| "Branch not found"                     | The skills can't fetch. Run `git fetch` yourself, then retry.                                                     |
| `pr-fix` asks which report to use      | Several reports match, or none matches the current branch. Name the file.                                         |
| `pr-fix` says the branch doesn't match | Fixes go on the report's source branch. Switch yourself; the skill never switches.                                |
| Finding marked "Already fixed"         | The quoted `Code:` no longer exists. Re-run `pr-review` for a fresh report.                                       |
