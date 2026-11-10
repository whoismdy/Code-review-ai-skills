# Architecture overview

> Project context for code review, maintained with the `review-setup` skill
> and read by `pr-review` and `pr-fix`. Edit freely, but keep rules checkable
> against a diff and keep every rule backed by evidence or team agreement.

<!-- Contract: the review skills look up these headings by exact name, so keep
them even if empty ("None yet"): "Architecture rules", "Review priorities",
"Ignore in review", "Unconfirmed". All other sections are optional and can be
renamed. Plain Markdown only; link to files instead of copying lists. -->

## Stack and tooling

<!-- Language(s), framework(s), package manager, build tool, test runner,
lint/format tools. Monorepo: one line per package/app. -->

- Language / framework: …
- Package manager / build: …
- Tests: … (location: `…`), or "none"
- Lint / format: … (config: `…`)
- Commands: `…` (writes files: `…` — informational; reviews don't run them)

## Structure and layers

<!-- One row per top-level source folder that matters. "May depend on" lists
the layers it is allowed to import. -->

| Path | Layer | Responsibility | May depend on |
|---|---|---|---|
| `src/…` | UI | … | state, utils |
| `src/…` | Services / API | … | utils, config |

## Data flow

<!-- How data moves between layers. Name the source of truth for shared
data and where I/O / API calls are allowed. Include one traced example. -->

- Chain: UI → State → Service → Mapper → Data access → HTTP client (adjust to
  the real layers)
- UI may call only: …
- Source of truth: …
- I/O and API calls happen only in: …
- Example (`<feature>`): `path/entry` → `path/handler` → `path/service` → …

## Cross-cutting behavior

<!-- Infrastructure that acts implicitly: interceptors/middleware, global
error handling, auth/token handling, DI, i18n. Say what it does and what a
diff must not duplicate or bypass. One line each, with the file. -->

- …: … (`…`). Diffs must not: …

## Conventions

<!-- Each convention with one example path. Only patterns the code mostly
follows. Where variants coexist, state the preferred one with counts. -->

- Naming (files / symbols): … (e.g. `…`)
- Variant example: preferred for new code `*.utils.js` (N files); legacy
  `*.util.js` (M files)
- File organization: by feature / by type … (e.g. `…`)
- Module / component pattern: … (e.g. `…`)
- Shared state: … (e.g. `…`)
- Error handling: … (e.g. `…`)
- Tests: … (e.g. `…`)

## Architecture rules

<!-- Checkable against a diff. Stable IDs, never renumbered; append new ones.
Each ends with (evidence: path) or (confirmed). -->

### Must

- **R1** … (evidence: `…`)

- **R3** (new/changed code) … (confirmed)

### Must not

- **R2** … (confirmed)

## Known pitfalls

<!-- Project-specific anti-patterns and past incidents, each with what to
look for in a diff. -->

- …

## Review priorities

<!-- What reviewers should weigh most in this project, in order. -->

1. …

## Ignore in review

<!-- Tool-enforced rules, generated or vendored paths, accepted deviations. -->

- Formatting and lint rules enforced by `…`
- Generated / vendored: `…`

## Unconfirmed

<!-- Inferred from code but not confirmed by the team. Reviewers treat these
as hints, never as grounds for a Must fix. -->

- …

## Further docs

- …

<!-- review-setup: <commit sha> <YYYY-MM-DD> -->
