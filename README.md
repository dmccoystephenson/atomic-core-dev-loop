# atomic-core-dev-loop

A Claude Code skill that runs an autonomous, iterative development loop for [atomic-core](https://github.com/dmccoystephenson/atomic-core) — a composable, plain-Three.js first-person dungeon-crawl library, forked from [philbgarner/atomic-core](https://github.com/philbgarner/atomic-core).

## What it does

Each loop cycle runs 10 phases in order:

| Phase | Name | Description |
|-------|------|-------------|
| 1 | Triage | Resolve and unshallow the checkout, pull `master`, adopt any open PR, check the fork against upstream, then work the **issue tracker** first and scan only to top it up |
| 2 | Work selection | Choose 0–3 candidates as a coherent PR (tiebreaker + early-cycle bias), or run Stage A (documentation sweep) or Stage B′ (example coverage); write a plan summary re-read at the start of Phase 3 |
| 3 | Implementation | Localization verification → edits that respect the per-file style and the strict compiler → `npm run typecheck` + `npm run build` + a browser pass over the relevant demo → staging hygiene (`dist/`, `src/libold`, `.claude/`) → scope ceiling (~400/~800 LOC) |
| 4 | PR | Push, open the PR **against the fork** (`--repo`/`--base`/`--head` are load-bearing here), then run a rubric-based self-review anchored on the local verification set — this repo runs no CI on pull requests |
| 5 | Wait for review | Poll for comments with `ScheduleWakeup` (270 s, 5 wakeups ≈ 22 min); skipped in batch and headless runs |
| 6 | Address comments | Fix or rebut each comment with evidence; external signals only — don't re-run the internal rubric |
| 7 | Doc accuracy check | Verify an eleven-row sources-of-truth table (`README.md`, `FEATURE_SOURCE.md`, generated `docs/`, `package.json`, `SET_OPTIONS_PLAN.md`, both example index pages, source-file headers) against this PR's implementation |
| 8 | Merge | Regression gate (the defect was *observed* reverted and restored, not argued) + do-not-auto-merge path check (`dist/`, `package.json`, `tsconfig.json`, release scripts, licenses, artwork, `src/server/`), squash-merge, then write a cycle summary |
| 9 | Self-audit | Run the structured rubric (Identity drift / Instruction clarity / Edge case coverage / Phase friction / Drift candidates / External-signal quality) and file labeled issues against **this** repo; never self-merge skill edits |
| 10 | Next cycle | Return to Phase 1 |

The Phase 4 self-review is rubric-based, externally anchored, and capped at one intrinsic-critique pass per PR — repeated intrinsic critique without an external signal is neutral-to-harmful.

## What makes this repo different

The skill is shaped around four facts about atomic-core that generic instructions get wrong:

- **The default branch is `master`, not `main`,** and the project repo is a **fork** — `gh pr create` would otherwise open the PR against someone else's project.
- **There is no test suite, no linter, and no CI on pull requests.** The only workflow fires on `v*` tags. The external anchor is local: `npm run typecheck`, `npm run build`, and a hand-walked example page. `gh pr checks` reporting nothing is normal, not a failure — and there is no CI to catch later what the local anchor could not.
- **`dist/` is committed and the examples load it.** A `src/lib` change is invisible in the browser until `npm run build`, and that build dirties tracked files that a feature PR must never stage — only `scripts/release.sh` commits `dist/`.
- **Issues are enabled on the fork** (since 2026-09-05), so triage starts from a real backlog and `Closes #N` resolves against this fork — but never against upstream, whose numbers belong to another repository.

## Usage

Invoke the skill from Claude Code:

```
/atomic-core-dev-loop
```

The skill is defined in [`atomic-core-dev-loop.md`](atomic-core-dev-loop.md). It resolves its working directory at runtime rather than assuming a fixed path — set `$ATOMIC_CORE_DIR` if your checkout lives somewhere the resolution order doesn't find, instead of editing the skill body.

## Files

```
atomic-core-dev-loop.md   — skill definition (phases, edge cases, config)
README.md                 — this file
LICENSE.md                — license text
```

## License

This project is licensed under the **MIT License**.

**License:** MIT © 2026 Daniel McCoy Stephenson  
See [LICENSE.md](./LICENSE.md) for the full text.
