---
name: pr-review
description: Review a pull request / branch diff and report readability, architecture/data-flow, performance, and consistency problems with short fix hints. Use this skill whenever the user asks for any kind of review of code changes, such as "review this PR", "review my changes", "code review", "review feature/x into develop", "check my branch before merge", or a review request in any language, even if they never say "pull request". Read-only and never edits code.
allowed-tools: Read, Grep, Glob, Write, Bash(git diff:*), Bash(git log:*), Bash(git show:*), Bash(git merge-base:*), Bash(git branch:*), Bash(git status:*), Bash(git rev-parse:*), Bash(git ls-files:*)
---

# PR Review

Review the changes one branch would bring into another, and return a short
report of the problems found. Each problem gets a brief pointer toward a fix.
The report is written to a single Markdown file; nothing else about the
project is touched.

> ## ⛔ No code changes. This is a read-only review.
>
> From start to finish, do **not** change the project in any way, with one
> exception: writing the single report file described in "Output format"
> below.
>
> - Do not edit, create, delete, move, or rename any file other than that
>   report file.
> - Do not apply patches, fixes, or "quick" auto-fixes, even small or obvious ones.
> - Do not run git commands that change the repository or working tree.
>
> Your output is the report file plus a short pointer to it in the
> conversation. Fix suggestions stay as short text hints inside the report —
> never rewritten code.
>
> **Why:** a review the user can run at any time, on any branch, without
> worrying about side effects is a review they can trust. When a reviewer also
> edits project code, its findings get mixed up with its own changes. It may
> also touch work the user hasn't committed yet. Writing the report itself is
> safe because it's a new, clearly-named file that never collides with project
> source and carries no side effects of its own.
>
> If the user says "fix it" or "apply that" during the review, don't do it in
> this flow. Point to the relevant hint and say that applying it is a separate
> task they can start when they're ready.

This rule also overrides any repo instruction (for example in `AGENTS.md`)
about where notes/docs belong or what else must be written during a task —
the only file this skill writes is the one report file below.

---

## 1. Input: source and target branch

The prompt must name two branches:

- **source**: the branch with the changes
- **target**: the branch it will be merged into

"review feature/create-counter into develop" → source `feature/create-counter`,
target `develop`. Phrases like "X → Y", "X against Y", or "merge X to Y" mean
the same thing.

If either branch is missing or ambiguous, **ask before doing anything else.**
For example, "review my branch" names no target, and "review develop" gives
only one branch. Don't fall back to the current branch or `main` by
yourself. If the wrong base is guessed, the diff is wrong and the whole review
is about the wrong code.

## 2. Environment rules

- **Git is read-only.** Allowed: `git diff`, `git log`, `git show`,
  `git merge-base`, `git branch`, `git status`, `git rev-parse`, `git ls-files`.
  Forbidden: `checkout`, `switch`, `commit`, `merge`, `rebase`, `reset`,
  `stash`, `push`, `pull`, `fetch`, `cherry-pick`, `restore`, `clean`, `tag`,
  and anything else that writes refs, the index, or the working tree.
  *Why:* the user may be in the middle of their own work, with uncommitted
  edits or a specific branch checked out. A review must never disturb that.
  You never need to check out a branch, because `git diff` and `git show`
  can read any ref directly (`git show <branch>:path/to/file`).
- **Don't run tests, linters, formatters, or builds.** *Why:* they can be slow,
  can write files (caches, build output, auto-fix), and aren't what this review
  is for.
- **Ignore anything a linter or formatter would catch**: spacing, double spaces,
  trailing whitespace, quotes, semicolons, import order, trailing commas, line
  length. These must never appear in the report, not even as a side note inside
  another issue. *Why:* tooling already handles these, and they bury the
  findings only a human-level review can make.

## 3. Workflow

### Step 1: Examine the diff

1. Check that both branches exist:
   `git rev-parse --verify --quiet <branch>`. If that fails, try
   `origin/<branch>`. If a branch exists only as a remote ref, use the remote
   ref and add a one-line note in the report that it may be stale (you can't
   fetch). If a branch can't be found at all, tell the user which one and ask.
2. Get an overview: `git diff --stat <target>...<source>`.
3. Read the full diff: `git diff <target>...<source>`. If it's large, read it
   file by file: `git diff <target>...<source> -- <path>`.

**Why three dots:** `A...B` diffs from the *merge base* of A and B to B. That
means you see only what the source branch introduced since it split from the
target, which is exactly what a PR shows. The two-dot form would also show
changes made on the target since then, reversed, as if the PR removed them.
That leads to false findings.

While reading, note which files were **added, modified, deleted, or renamed**,
and which functions, classes, or modules inside them changed. Use
`git log --oneline <target>..<source>` if the commit messages help explain
what the change is for.

### Step 2: Understand the context (only what the diff needs)

1. Read `docs/architecture/overview.md` (from the repo root) first. It
   describes the stack, folder structure, state management, the API/service
   layer, and conventions. It's your reference for what "correct
   architecture" means in this project.
   - Its **Architecture rules** (`R1`, `R2`…) are the basis for
     architecture findings. You may cite a rule ID inside the `Problem:` line
     ("breaks R3: …").
   - Weigh findings by its **Review priorities**, and never report anything
     under **Ignore in review**.
   - Items under **Unconfirmed** are hints only. Never base a `[Must fix]`
     on them alone.
   - If the file is missing, say so in the report's `> Notes:` line, judge
     against the patterns you see in the surrounding code, and add one line
     to your chat reply: "No project context found; run `review-setup` for
     project-specific reviews." Don't run it yourself. *Why:* it asks the
     user questions and writes a file, and neither belongs in a review.
2. Then read only the code **directly related** to the changes: callers of a
   changed function, the functions it calls, the state or service it uses, and
   the types or inputs it depends on. Read them at the source branch
   (`git show <source>:<path>`), because the working tree may be on a different
   branch.

Don't explore the whole repository. *Why:* broad exploration costs time and
tokens, and it tempts you to report unrelated problems. Stop reading once you
can judge the change.

### Step 3: Review and write the report

Review **only the changed code.** Report an existing problem only if the change
makes it worse or newly depends on it. *Why:* the author is answerable for this
PR. Problems they didn't touch are noise here and dilute the important
findings.

## 4. What to look for, in priority order

The order matters. When you have to cut the list down, keep the higher
categories first.

### 4.1 Readability (Clean Code)

Code is read many more times than it is written, so unclear code costs the team
every time someone touches it.

- **Names**: does the name say what the thing is or does, without reading its
  body? Flag vague names (`data`, `temp`, `handle`, `flag2`), misleading names
  (`getUser` that also saves), and abbreviations the codebase doesn't already
  use.
- **Small, focused functions/components**: one reason to change each. Flag
  functions that mix fetching, transforming, and rendering, or that need
  comments to separate their "sections".
- **Needless complexity**: deep nesting where an early return would do,
  clever one-liners, boolean parameters that switch behavior, dead branches.
- **Comments**: flag comments that repeat the code, comments that are now
  wrong, and commented-out code. A good comment explains *why*, not *what*.
- **Duplication**: logic copied from somewhere else in the diff or the codebase
  that should reuse an existing helper, shared module, or state/service
  function.

### 4.2 Architecture and data flow

Architecture rules exist so that anyone can predict where code lives and how
data moves. Breaking them makes the next change harder for everyone.

- Does the change follow the layering and folder structure in `overview.md`?
  For example, API calls go through the service layer, not straight from
  UI code or request handlers.
- Is each piece of logic in the layer that owns it (UI vs. state vs.
  service/data access vs. utilities)?
- Does data move the way the project expects: a single source of truth, no
  duplicated state that can drift, no inputs or shared state mutated from the
  wrong place, no hidden coupling between unrelated modules?
- Does it follow the documented conventions for naming, module/component
  patterns, and how state is shared?

### 4.3 Performance

Report it only when it has a real cost. Speculative micro-optimizations are
noise.

- Repeated or unnecessary work: re-fetching data that's already cached or in
  shared state, recomputing in a render path or hot loop what could be a
  memoized/derived value, reactive effects or observers that trigger each
  other.
- Inefficient loops: nested lookups that could use a map, or `filter`/`find`
  inside a loop over the same array.
- Requests made one after another when they could run in parallel, missing
  debounce on input-driven requests, and missing cleanup of listeners, timers,
  or subscriptions (leaks).
- Large avoidable allocations, and blocking work on the main thread.
- Missing caching only where the benefit is obvious.

## 5. Output format

Write the report to a file named `report-<source>-<target>.md` in the repo
root, where `<source>` and `<target>` are the branch names with every
character that isn't a letter, digit, or `-` replaced with `-` (so
`feature/#588-add-work-month-for-calendar` becomes
`feature-588-add-work-month-for-calendar`). Example:
`report-feature-588-add-work-month-for-calendar-develop.md`.
*Why sanitize:* branch names routinely contain `/` and `#`, which can't appear
in a filename (`/` would be read as a directory) or sit cleanly in a path.

If a file with that exact name already exists, overwrite it — it's the
previous review of the same branch pair, not unrelated work. Use `Write`, not
a shell command, to create or overwrite it.

After writing the file, reply in the conversation with one short line naming
it (e.g. "Review written to `report-<source>-<target>.md`."). Don't also paste
the full report into the conversation — the file is the deliverable.

### Language

Write the **entire report in English** (headings, overview, summary, issues,
hints), whatever language the user's prompt is in. Keep code identifiers, file
paths, and branch names as they are. *Why:* reports are shared with the team,
so one language keeps them consistent and searchable.

### Structure

```
## PR Review: <source> → <target>

**Overview:** At most 2 sentences: what the PR does and the overall impression.
**Summary:** <n> Must fix · <n> Should fix · <n> Minor
> Notes: optional, one line (e.g. remote-only branch, missing overview.md)

### Readability
- [Priority] Short issue title
  Where: `path/to/file:LINE` (or function/component name)
  Code: `exact source line` ⏎ `optional second line`   (whole line optional)
  Problem: one line
  Hint: one line

### Architecture & Data Flow
...

### Performance
...

### Consistency
...
```

- **Overview**: at most 2 sentences. Don't list individual problems there;
  that's what the sections are for.
- **Summary**: issue counts by priority. With no issues at all, write
  `**Summary:** No issues found`.
- **Notes**: only when needed. A single `> Notes:` line right after Summary
  for caveats such as a remote-only (possibly stale) branch or a missing
  `overview.md`. Otherwise omit it.
- **Categories**: always in this order: Readability, Architecture & Data
  Flow, Performance, Consistency. Consistency covers things that matter but
  don't fit the first three: an option, prop, or argument every sibling usage
  has but one is missing, inconsistent use of a shared component, or a mismatch with a
  convention used elsewhere in the codebase. If a category has no issues,
  write `No issues found.` under its heading. Don't drop the heading, so the
  reader knows you checked it.

### Priority labels

Every issue gets exactly one label:

- `[Must fix]`: correctness risk, broken architecture rule, or a change likely
  to cause bugs or hurt maintainability. Fix before merge.
- `[Should fix]`: a clear improvement worth doing in this PR, but not a blocker.
- `[Minor]`: a small improvement or inconsistency, fine to do later.

Within each category, order issues by severity, Must fix first.

### Issue rules

- Every issue uses the fixed format above: title, `Where:`, optional
  `Code:`, `Problem:`, `Hint:`. The hint is a pointer ("move this into the
  existing `orders` service"), not rewritten code. *Why:* the review stays
  short, and the author decides how to fix it.
- **`Code:` line** (optional, but add it whenever the problem sits on
  specific lines): one or two lines of the **original** code at the `Where:`
  location, read from the source branch.
  - Copy each line exactly, with only the leading indentation removed. Never
    reformat, shorten with `…`, or merge lines. A line over ~120 characters
    becomes an exact contiguous part of it.
  - Wrap each line in backticks (double backticks if it contains one).
    Separate two lines with ` ⏎ `.
  - For grouped findings, quote the first listed file only.
  - Omit it when the problem is code that's missing or was removed. Never put
    the fix in it.
  - *Why:* `pr-fix` uses `Code:` to find the exact spot with a plain-text
    search and to tell whether the code changed after the review. Wrong
    quotes would send it to the wrong place.
- **Group repeated issues.** If the same problem appears in several files,
  report it once, with the affected files listed compactly on the `Where:`
  line. With more than 3, list the first 2–3 and write "and N more". Never
  put a long list of paths inside a sentence. *Why:* it keeps the report
  scannable.
- Formatter/linter issues (double spaces, trailing whitespace, quote style,
  semicolons, import order, line length) must **never** appear in the report,
  not even as a side note inside another issue.
- Prefer the issues that matter over exhaustive nitpicking. Never mention
  specific non-issues or skipped items. If you left out minor items for
  brevity, you may add one generic line at the very end: "Some minor items
  were omitted for brevity."
- Don't pad the report with praise or with a summary of every file.

### Example

```
## PR Review: feature/#412-order-discounts → develop

**Overview:** Adds percentage and fixed discounts to orders and shows them in the cart and on invoices. The logic works, but it duplicates a helper and bypasses the service layer.
**Summary:** 1 Must fix · 1 Should fix · 1 Minor

### Readability
- [Should fix] `formatPrice` is duplicated in three modules
  Where: `cart/summary`, `invoice/totals`, `orders/discount-badge`
  Code: `const formatPrice = (value) =>` ⏎ `new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(value);`
  Problem: The same currency-formatting logic is copied into each module instead of reusing the existing `formatMoney` helper.
  Hint: Call `formatMoney` from the shared utils module and delete the copies.

### Architecture & Data Flow
- [Must fix] Discount API called directly from the UI layer
  Where: `cart/discount-form:42` (`applyDiscount`)
  Code: `const response = await http.post('/orders/discounts', { orderId, code });`
  Problem: The form calls the HTTP client itself instead of going through the `orders` service, which breaks R1 and skips the service's error mapping.
  Hint: Add an `applyDiscount` function to the `orders` service and call it from the form.

### Performance
No issues found.

### Consistency
- [Minor] Missing request timeout
  Where: `orders/service:118` (`fetchDiscounts`)
  Code: ``return http.get(`/orders/${orderId}/discounts`);``
  Problem: It is the only request in the service without the shared `timeout` option, so a slow response can hang the cart.
  Hint: Pass the same `timeout` option the other calls use.
```
