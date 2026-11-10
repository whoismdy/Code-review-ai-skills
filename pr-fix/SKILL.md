---
name: pr-fix
description: Apply fixes for findings in an existing `pr-review` report (`report-<source>-<target>.md`). Verifies each finding against the current code, proposes one concrete fix at a time, and edits code only after the user explicitly approves that specific fix. Use this skill when the user asks to fix, address, apply, or resolve issues from a PR review or review report, such as "fix the review findings", "apply the Must fix items", "address the report for feature/x", "fix FIX-02 from the review", in any language. Do NOT use it for a plain review request; that is `pr-review`, which stays read-only.
allowed-tools: Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git rev-parse:*), Bash(git merge-base:*), Bash(git ls-files:*), Bash(git ls-tree:*), Bash(cat:*), Bash(head:*), Bash(tail:*), Bash(grep:*), Bash(ls:*), Bash(wc:*)
---

# PR Fix

Fix findings from a `pr-review` report, **one explicitly approved fix at a
time**. `pr-review` reports (read-only); this skill verifies, proposes, applies.

> ## ⛔ No edit without explicit approval of that exact fix (by ID).
>
> Not approval: the initial request ("fix the report", "fix all Must fix"), a
> `[Must fix]` label, a trivial-looking change, approval of another fix,
> silence, or any ambiguous reply. "Yes to all" → explain each fix still gets
> its own proposal and yes. When unsure, ask again.

Priorities: authorization → user's existing work → correctness → architecture
→ minimal scope → quality. **When in doubt, don't edit; ask.**

## Rules

- **Git read-only**: `status`, `diff`, `log`, `show`, `rev-parse`,
  `merge-base`, `ls-files`, `ls-tree`. Never anything that writes the working
  tree, index, or refs (`checkout`, `switch`, `reset`, `restore`, `clean`,
  `stash`, `add`, `commit`, `push`, `pull`, `fetch`, `merge`, `rebase`,
  `cherry-pick`, `tag`, `branch -d/-D/-m`…). Never stage or commit.
- Read-only shell commands run without asking. Change files only with `Edit`,
  or `Write` for a new file named in an approved proposal. No `sed -i`,
  codemods, formatters, or `--fix`.
- No tests, linters, builds, or installs. Validate by reading the diff. Never
  claim checks passed.
- Never modify any `report-*.md` or the `pr-review` skill.
- Reply in the user's language.

## Token economy

- Read the report once. Verify a finding only when its turn comes, not up front.
- Locate code with `Grep -n` first, then `Read` with `offset`/`limit` around
  the hit. Read related code (callers, callees, state/service layer) only as far as
  the fix needs. Never read whole large files or explore broadly.
- Use `git status --porcelain` and `git diff --stat` for overview. Run a full
  `git diff -- <file>` only for files a fix touches.
- After editing, check with `git diff -- <files>`, not by re-reading files.
- Keep messages short: no restating the report, no code dumps; a few
  before → after lines at most.

## Workflow

### 1. Find the report

Reports are `report-<source>-<target>.md` in the repo root. Branch names are
sanitized (non `[A-Za-z0-9-]` → `-`), which is lossy, so take the exact branch
names from the first line, `## PR Review: <source> → <target>`.

`Glob report-*.md`. Use the file the user named, or the one matching the named
branches, or the one whose header source is the current branch (read only the
first line of each, e.g. `head -1`). Exactly one match → use it and say so.
Otherwise → list candidates and ask. Never pick a report silently.

### 2. Check repository state

1. `git rev-parse --abbrev-ref HEAD`. If it isn't the report's source
   (or detached), say so and ask. Never switch branches.
2. `git status --porcelain`: every listed file is the user's work.
3. If the source branch has commits newer than the report
   (`git log -1 --format=%ci <source>` vs `ls -l <report>`), or the report has a
   `> Notes:` line, tell the user.
4. `git log --format=%s -15` once: note the commit style (scope or not, case,
   language) for the commit messages in 4g.

### 3. Parse and choose scope

Format: header; `**Summary:** <n> Must fix · <n> Should fix · <n> Minor`;
optional `> Notes:`; sections `### Readability`, `### Architecture & Data Flow`,
`### Performance`, `### Consistency` (empty: `No issues found.`); each finding
`- [Must fix|Should fix|Minor] <title>` + indented `Where:` / optional
`Code:` / `Problem:` / `Hint:`. `Code:` holds one or two exact source lines
in backticks (two are separated by ` ⏎ `). Older reports may not have it. `Where:` may be vague ("and N more", component or module names). "Some minor
items were omitted" → those aren't available; don't search for them.

Number findings `#1…` in report order, keeping priority, category, and title
verbatim. If the count doesn't match Summary or an item is malformed, ask.

Show a compact list (`#`, priority, category, title; one line each, Must fix
first) and ask which to work on. Choosing scope is **not** approval.

### 4. For each chosen finding, in priority order

**a. Verify.** The report is claims, not a spec. Check the working-tree code
against `docs/architecture/overview.md`: its rules (`R1`…) and conventions
apply, and its **Unconfirmed** items are hints only. If the file is missing,
follow the surrounding code and tell the user once that `review-setup` can
create it.
If the finding has `Code:`, locate it first with a fixed-string search
(`grep -nF -e '<line>' <Where files>`, with quotes in the line escaped for the
shell; fall back to the whole repo if `Where:` is vague):
- **Found**: that's the exact spot. Read around it with `offset`/`limit`.
- **Not found**: the code changed after the review. Look for what replaced
  it before deciding: "Already fixed" and "No longer applicable" are likely,
  but not certain.

`Code:` only locates the code. The verdict still comes from reading it.
Verdict: Valid · Valid, different fix · Already fixed · No
longer applicable · Incorrect · Unclear. Not Valid → one line with the reason,
move on (unless the user asks for a proposal). Never change code just to match
the Hint.

**b. Split into atomic fixes.** One logical, independently revertible change
per fix (`FIX-01`, `FIX-02`…). Split a finding that bundles independent
problems; never merge unrelated findings. A grouped finding that is truly one
change stays one fix, listing every file.

**c. Propose one fix, then end your turn.** Read everything it will touch
first so the proposal is complete. It is the contract: anything not listed is
out of scope.

```
**FIX-01 — [Must fix] <finding title>** (#3, Architecture & Data Flow)
- Location: `path` (`fn`), … (why multiple files must change together)
- Verified problem: …
- Proposed change: exact lines/functions; added/removed/moved; new files, deps
- Reason: …
- Impact: side effects, API/caller changes, overlap with uncommitted work
- Validation: what the diff check will confirm (no tests run)

May I apply FIX-01? (yes / no / later / change: …)
```

**d. Reply.** An explicit yes to this fix → apply. No → Rejected. Later →
Postponed, untouched. "Yes, but…" → revise, re-propose, ask again. Ambiguous →
ask again. One approval = one fix.

**e. Apply.**

1. `git diff -- <files>` = baseline (the user's changes). If they overlap the
   lines you'll edit, or the code changed since the proposal, stop and ask.
2. Make exactly the approved change: smallest correct edit, existing patterns,
   no extra refactoring, renaming, or reformatting, no unlisted deps, keep
   public APIs unless approved. Consider edge cases and every caller.
3. If more turns out to be needed (another file, caller, or helper), stop.
   Report what's changed so far and ask for that as a separate approval. Don't
   widen the fix or undo partial work on your own.

**f. Check.** `git diff -- <files>` vs baseline (read new files directly):
only listed files changed, only the approved change, user's changes intact.
If something unapproved slipped in, say so and offer to remove it (with
approval). Report in one or two lines: `FIX-01 applied`, files, what changed,
no tests run.

**g. Commit message.** Right after the check, give one commit message for this
fix in a code block. Text only: never stage or commit.

- [Conventional Commits](https://www.conventionalcommits.org):
  `<type>[(scope)]: <subject>`, matching the repo style from step 2.4.
- Type by what the change does, not by the finding's priority: `fix` (wrong
  behavior), `refactor` (same behavior, better structure), `perf`, `style`
  (formatting only, no meaning change), `chore`, `feat` (only if the approved
  fix adds a capability). Never `feat` for a pure cleanup.
- Subject: imperative, lowercase, no period, ≤ 72 chars, says what changed
  (not "fix review issue").
- Body (optional, only if the subject can't carry it): one or two lines on
  why. Skip `BREAKING CHANGE:` unless the approved fix changes a public API.
- If a listed file also holds the user's uncommitted changes, or an earlier
  fix touched the same file and isn't committed yet, say that committing this
  fix alone needs `git add -p`.

### 5. Final summary

Short table: FIX ID, `#`, priority, title, status (Applied / Rejected /
Postponed), commit subject (Applied only), plus findings with no proposal and
their verdict. List files this
skill changed, separately from files already modified before it started.
Nothing was staged or committed, and no tests were run. Don't edit the report;
re-run `pr-review` for an updated one.
