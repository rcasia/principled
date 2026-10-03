# ADR-0046: Scope push mutation runs to changed files, verify the full score weekly

**Status**: Accepted
**Date**: 2026-10-03

## Context

Push mutation runs take ~1.5 hours: 7264 mutants × the full `bun test`
suite (~2.2 s) across 4 runners. ADR-0038 already decoupled mutation from
delivery (advisory, standalone workflow), but every main push still occupies
a runner for 1.5 h, and each new push queues behind the last
(`cancel-in-progress: false`). The incremental cache meant to avoid this
(ADR-0021) crashes at this suite size (`RangeError: Invalid string length`
in `writeIncrementalReport`), so dependency/test/config changes force a full
run without `--incremental` — meaning a Dependabot-only push with zero
source changes still costs 1.5 h. Verified locally: a single-file subset
(`stryker run --mutate packages/web/src/shared/version.ts`) finishes in
~5 s at 100%.

## Decision Drivers

- **No push action runs for hours.** The push signal must arrive in
  minutes, so the workflow is actually watched instead of ignored.
- **Agent-sized diffs stay sound.** The typical push touches a few
  colocated source/test files; the score for those files must mean what a
  full run would have said about them.
- **No silent scope creep.** Whatever push runs skip must be re-verified on
  a schedule, explicitly — not eroded by flags.
- **Keep the 95 semantics.** `thresholds.break: 95` still fails the run
  when the tested subset drops below it.

## Considered Options

### Option 1: Shard the full run across a job matrix (one job per package)

- **Pros**: Keeps full-run semantics every push; wall time falls by roughly
  the shard count.
- **Cons**: Still tens of minutes (the largest package dominates), doubles
  runner consumption, and needs report-merging plus per-shard cache keys to
  stay correct. Complexity for a signal that is advisory.

### Option 2: Diff-scoped push runs plus a weekly full run (chosen)

- **Pros**: Push cost scales with the diff: a few files finish in
  seconds–minutes; pushes with no source/test changes skip in seconds. No
  incremental file is ever written in CI, so the `RangeError` class is
  gone, not worked around. The weekly full run keeps the global score
  honest (dependency drift, helper changes, config edits).
- **Cons**: A push subset passing 95% proves the global score only when the
  previous global was green and uncovered inputs (deps, shared helpers,
  config) did not move killing power — the weekly full is what catches
  that, up to a week later.

### Option 3: Scheduled full runs only, nothing on push

- **Pros**: Zero push cost; simplest workflow.
- **Cons**: Loses the per-push signal entirely — the author of a surviving
  mutant hears about it days later, disconnected from the diff. Rejected:
  the push signal is the point of running it per push.

## Decision

- **Push: diff-scoped, no incremental.** Diff the push base
  (`github.event.before`, falling back to `HEAD~1`) against `HEAD` for
  `packages/*/src/**/*.ts(x)`. Changed test files map to their source
  (`foo.test.ts` → `foo.ts`, which the repo's colocated layout makes
  exact); barrels/entries Stryker never mutates (`index.ts`, `bin.ts`,
  `lambda-entry.ts`) stay excluded so the CLI flag cannot override the
  config exclusions. The deduplicated set is passed as Stryker `--mutate`.
  No `--incremental` is passed in CI, so no incremental report is read or
  written and the serialization crash cannot trigger.
- **Skip fast when there is nothing to mutate.** Pushes touching no source
  or test files (docs, Dependabot-only, release commits) exit green in
  seconds with an explicit log line instead of a vacuous dry-run.
- **Weekly full run.** Mondays 02:00 UTC (`bun run test:mutation`, no
  incremental), so dependency drift and anything the scoping misses is
  re-verified globally within a week. (Audit runs Mondays 06:00; no
  conflict.)
- **Manual dispatch runs full.** An explicit `workflow_dispatch` is a rare,
  opt-in request for the global signal, so it takes the full run.
- **Concurrency unchanged.** Queue (`cancel-in-progress: false`): fast runs
  drain in seconds–minutes, so every main commit still gets its diff score
  without overlap.
- **Thresholds unchanged.** Subset runs still break below 95.
- The `reports/stryker-incremental.json` cache steps leave the workflow.
  `stryker.config.json` keeps `incrementalFile` and the
  `test:mutation:incremental` script keeps working as the local opt-in;
  CI simply never passes `--incremental`.

## Consequences

### Positive

- Push mutation feedback drops from ~1.5 h to seconds–minutes and scales
  with the diff; the queue stops piling up hour-long runs.
- The `RangeError` crash class disappears from CI (nothing writes the
  incremental report there anymore) instead of being routed around.
- The workflow halves in steps (no restore/snapshot/compare/save dance),
  so there is no cache machinery to keep in sync.

### Negative

- Push runs no longer refresh any shared incremental state — there is none
  to refresh. Local `--incremental` runs reuse only local state.
- A subset passing 95% is not a fresh global 95%: global drift between
  weeklies (e.g., a `happy-dom` bump changing test runtime behavior) waits
  for Monday. Accepted deliberately: the signal is advisory (ADR-0038).

### Risks and mitigations

- *Risk*: A test-only change weakens a test whose source is not its
  colocated file, so the mapped subset misses it. *Mitigation*: helpers
  live in `src` and are treated as source — changing one mutates the
  helper itself, which surfaces immediately; the weekly full catches the
  rest.
- *Risk*: `package.json`/`bun.lock` bumps change killing power while push
  runs skip or scope narrowly. *Mitigation*: weekly full re-verifies;
  devDeps that affect `bun test` runtime are the exception, not the rule
  (Bun itself is workflow-pinned, and `bun test` does no typechecking, so
  the TypeScript pin has no killing effect).
- *Risk*: `stryker.config.json` or workflow edits change score semantics
  while their own push skips. *Mitigation*: such edits are rare and
  reviewed as config; the weekly full validates the new semantics within
  days.
- *Risk*: Previous global red + subset green reads green. *Mitigation*:
  the weekly full is the backstop; the run log states its scope explicitly
  ("testing mutants in: …") so a scoped green is never mistaken for a
  fresh global 95.
