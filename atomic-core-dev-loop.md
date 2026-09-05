# atomic-core-dev-loop

<!-- template-version: fa50415 -->
<!-- generated-at: 2026-09-05T19:10:31Z -->
<!-- template-source: phase structure mirrored from ~/local-skills/forward-one-dev-loop (template-version fa50415); create-dev-loop.md itself was outside this session's allowed read paths, so the SHA above names the newest template revision actually observed, not one read directly -->

Autonomous iterative development loop for atomic-core.

**Identity:** the kind of skill that refuses to let a green `tsc` stand in for a rendered dungeon — every example in this repo loads the **committed** `dist/` bundle, so a `src/lib` change nobody rebuilt and walked around in a browser is unverified no matter how clean the compiler is. It rejects a new public symbol that never reaches `src/lib/index.ts`, a feature whose `FEATURE_SOURCE.md` row still names the file it used to live in, a hand-edited file under `docs/` (typedoc owns that tree), and a `dist/` rebuild smuggled into a feature PR — `scripts/release.sh` is the only thing that commits `dist/`.

**Working directory:** /home/userland/.cache/gardener/repos/dmccoystephenson__atomic-core (resolved at runtime — see Phase 1; do not assume this literal path exists)
**Project repo:** dmccoystephenson/atomic-core — a **fork** of `philbgarner/atomic-core`, default branch **`master`** (not `main`)
**Skill repo:** dmccoystephenson/atomic-core-dev-loop (issues for self-audit findings go here)
**Project guidance:** there is no `CLAUDE.md` and no `CONTRIBUTING.md` — `CLAUDE.md` is in `.gitignore`, deliberately. Read `README.md` and `FEATURE_SOURCE.md` at the start of each cycle if context is cold; they are the repo's own description of what it is and where each feature lives.

---

## Repo shape — read this before anything else

Everything below was established from the source, `package.json`, `tsconfig.json`, `.github/workflows/release.yml`, `scripts/release.sh` and the git history as of 2026-09-05. Re-check those sources if this section feels stale.

- **What it is:** a plain-Three.js, tile-based first-person dungeon-crawl library. TypeScript source under `src/lib/` (~17k LOC, 55 files), built by Vite into `dist/` as ES + UMD + IIFE bundles plus `.d.ts` declarations. `src/server/` holds a small authoritative WebSocket multiplayer server (`src/server/index.js`, plain JS) and its Node build entry.
- **No test suite. No linter. No formatter. No CI on pull requests.** The only workflow is `.github/workflows/release.yml`, and it triggers on `v*` **tags** only. `gh pr checks` will report nothing on a PR; that is the repo's normal state, not a fault to fix mid-cycle. The external anchor is local — see Phase 3.
- **The compiler is the only automated gate.** `tsconfig.json` runs `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, `isolatedModules`, with `emitDeclarationOnly` and `rootDir: src/lib`. `npm run typecheck` is `tsc --noEmit`.
- **`dist/` is committed** (119 tracked files). `.gitignore` has `dist` commented out with the note *"Turns out the github build pipeline does rely on this."* Every example page loads `dist/atomic-core.iife.js` from the working tree. `scripts/release.sh` is what rebuilds and commits it, at release time, in a `build: vX.Y.Z` commit. **A feature PR never commits `dist/`.**
- **`docs/` is generated** by `npm run docs` (typedoc + `typedoc-plugin-markdown`, 133 files). Never hand-edit a file under `docs/`; regenerate.
- **`src/libold` is a dangling gitlink.** `git ls-files -s src/libold` shows mode `160000` with no `.gitmodules` — an accidentally-committed nested repo that materializes as an empty directory. `tsconfig.json` excludes it. Leave it alone unless removing it is the whole point of a PR (it is a history-touching change; treat it as owner-gated).
- **Issues are enabled on the fork** (turned on 2026-09-05; `hasIssuesEnabled: true`). `gh issue list --repo dmccoystephenson/atomic-core --state open` returns a real backlog, seeded with `#2`–`#7` from a triage pass — three `bug`, two `enhancement`, one `documentation`. The tracker starts at `#2` because `#1` was the first PR, not an issue. Triage starts here; see Phase 1.
- **The fork is currently identical to upstream** (`git log origin/master..upstream/master` and its reverse are both empty as of 2026-09-05) and has no PRs in its history. Every convention below was learned from upstream's commits.
- **Two licenses, and they are not the same.** `LICENSE_code.txt` is the Unlicense (public domain). `LICENSE_art.txt` is a restrictive, non-transferable artwork license, last updated October 2025. The `.png` atlases under `examples/` are art, not code — never relicense, republish, or copy them into a new location without owner authorization.
- **React is vestigial.** `package.json`'s description says "for React Three Fiber", its `peerDependencies` require `react`, `react-dom`, `@react-three/fiber` and `@react-three/drei`, and `vite.config.ts` loads `@vitejs/plugin-react` — but there is not one `.tsx` file and nothing under `src/lib` imports React. `README.md` says the opposite in its first paragraph ("No React, no JSX, no build step required"). This is real, known drift, and it is a **published-package contract**: dropping the peer deps changes what installs for every consumer. Surface it; do not resolve it unilaterally (Phase 1, harness/owner-gated list).

---

## Full cycle

---

### Phase 1 — Triage

**Resolve the working tree before `cd`** — never assume a hardcoded absolute path exists on every machine/container (the first action of the cycle must not fail because the configured path is absent). Resolution order: explicit env var → configured path → a detected clean clone → fresh clone.

```bash
# Resolve REPO_ROOT: $ATOMIC_CORE_DIR if set & present > the configured path > a clean clone whose
# origin matches dmccoystephenson/atomic-core (prefer no uncommitted changes; skip /mnt/ user copies) > fresh clone.
if [ -n "$ATOMIC_CORE_DIR" ] && [ -d "$ATOMIC_CORE_DIR" ]; then
  REPO_ROOT="$ATOMIC_CORE_DIR"
elif [ -d "/home/userland/.cache/gardener/repos/dmccoystephenson__atomic-core" ]; then
  REPO_ROOT="/home/userland/.cache/gardener/repos/dmccoystephenson__atomic-core"
else
  REPO_ROOT=""
  for d in ~/local-skills/* ~/* ~/*/*; do
    [ -d "$d/.git" ] || continue
    case "$d" in /mnt/*) continue;; esac
    git -C "$d" remote get-url origin 2>/dev/null | grep -q "dmccoystephenson/atomic-core" || continue
    REPO_ROOT="$d"; [ -z "$(git -C "$d" status --porcelain)" ] && break
  done
  [ -z "$REPO_ROOT" ] && { REPO_ROOT="$HOME/atomic-core"; git clone https://github.com/dmccoystephenson/atomic-core.git "$REPO_ROOT"; }
fi
cd "$REPO_ROOT"
git rev-parse --is-shallow-repository   # gardener clones are shallow (depth 1)
git fetch --unshallow                   # only if the line above printed 'true'
git checkout master && git pull
gh pr list --repo dmccoystephenson/atomic-core --state open
git log --oneline -10
```

**The branch is `master`, not `main`.** Every `origin/main` in a generic instruction is `origin/master` here.

**Expect a restricted fetch refspec.** A gardener-managed clone is configured `+refs/heads/master:refs/remotes/origin/master` only, so `git fetch` never creates a remote-tracking ref for a feature branch. That breaks `gh pr create`'s head auto-detection and bare `--force-with-lease`; see the Edge cases entries for both.

**The project has an issue tracker, and it is the first place to look.** Issues were enabled on the fork on 2026-09-05, so this returns a backlog rather than erroring:

```bash
gh issue list --repo dmccoystephenson/atomic-core --state open
```

- **Prefer an open issue over a self-found candidate.** An issue is a work item someone already judged worth doing, and closing one is progress a human can see; a scan candidate is only this loop's own opinion. Scan to top up when the tracker is empty, or when every open issue is owner-gated or too large for one cycle.
- **Read the issue's own acceptance criteria before Phase 3.** The seeded issues each carry a `## Verifying a fix` section naming the concrete local anchor for that specific change — which example page to walk, or the fact that a docs-only change gives `tsc` nothing to say. That section is the acceptance criterion; do not substitute a generic anchor for it.
- **Check whether an issue is owner-gated before picking it.** Some carry decisions that are not this loop's to make — adding a test framework changes `package.json` and CI, and restoring a commented-out encoding may reopen a bug the author hit. Where an issue says the owner should decide, the cycle's job is to surface evidence, not to choose.
- **A deferred candidate now goes on the tracker.** File it as an issue rather than only listing it in a PR body, so it outlives the PR that mentions it. Keep the `## Deferred this cycle` section for candidates too small to be worth their own issue.
- `gh issue list --repo philbgarner/atomic-core --state open` remains **read-only context** — upstream's tracker is the closest thing to a roadmap, but those numbers belong to another repository. Never `Closes` one; see Phase 4.

**Check for open PRs from previous cycles first.** If any open PR exists, decide before doing anything else:
- If the PR is still valid (anchor green, no conflicts), **first confirm a Phase 4 self-review was actually posted** (a carried-over PR from a prior cycle may never have completed it). If none is recorded, perform the Phase 4 self-review now (the anchor must be green first) before jumping to Phase 5. Otherwise jump to Phase 5 to re-poll for review.
- If the PR is stale or conflicted, close it with a comment explaining why, then proceed with triage.
- **If the open PR was authored by a concurrent session/another author** (not this loop), do not misread "don't open a new PR" as "do nothing": adopt it — bring it current with `master`, re-run the full anchor, review, and merge if green (or close it with a reason).
- **If the PR's diff contains `dist/`**, that is the highest-priority finding on it: a feature PR that commits a rebuild collides with the next release commit and hides the real change. Drop `dist/` from the branch before anything else.

Do not open a new PR while one is already open against the same repo.

**Check whether the fork has fallen behind upstream.** The fork carries no independent history yet, so a divergence is worth knowing before writing code on top of it:
```bash
git fetch upstream
git log --oneline origin/master..upstream/master
```
Bringing the fork up to date with upstream is a legitimate cycle in its own right, but it is a fast-forward of `master` rather than a PR — report it and let the owner decide; do not force the fork's history sideways.

Scan for improvements not yet tracked:
- **A public symbol that never reaches `src/lib/index.ts`.** The IIFE bundle's `window.AtomicCore` is exactly what `index.ts` re-exports; an exported-but-unlisted function is invisible to every script-tag consumer. Compare `grep -rn '^export ' src/lib --include=*.ts` against `src/lib/index.ts`.
- **`FEATURE_SOURCE.md` drift** — its "Directory structure" block against the real `src/lib` tree, each feature's **Files:** list against files that still exist and still implement it, and its "Public API surface" section against `index.ts`.
- **`README.md` drift** — the Features list against the options the code actually accepts, the Quick Start snippet's option names against `createGame.ts` / `dungeonRenderer.ts`, the standalone-examples table against `ls examples/standalone`, the commands against `package.json` `scripts`.
- **`docs/` staler than the source** — public types or exports changed without `npm run docs`.
- **`SET_OPTIONS_PLAN.md` phases that have since shipped**, and its line-number citations (`createGame.ts` line 101, line 810) which drift on every edit above them.
- **`examples/localhost/` and `examples/standalone/` out of parity** — they are meant to be the same demos, differing only in how the atlas image is loaded (`<img>` vs. embedded Base64 via `utils/imageToBase64Js.sh`). A fix applied to one and not the other is a bug in the other.
- **An example listed in an index page that does not exist on disk**, or a directory with no index entry (`examples/localhost/index.html`, `examples/standalone/index.html`).
- **Stale project naming** — `src/server/index.js`'s header still calls the project `r3f-crawl-lib`, its pre-fork name.
- **A `noUncheckedIndexedAccess` / `exactOptionalPropertyTypes` escape** — a `!`, `as any`, or `@ts-expect-error` added to silence the compiler instead of narrowing.
- **A subsystem with no example that exercises it.** The examples are this repo's only executable verification path; a feature reachable by no page is a feature nothing can check.
- Unhelpful error messages, or a thrown string where the public API should return a typed result.

**Before filing/recording each candidate**, verify every claim against source:
- Symbol names/call sites — grep to confirm existence and behaviour
- "X doesn't exist" — read the file to confirm the absence
- Example output — trace through code to confirm it is realistic

**After writing a batch of candidates**, second-pass each one: the title accurately describes the body, and every claim still holds after re-reading the source.

**Classify harness-blocked and owner-gated operations up front.** Flag these at triage rather than discovering them mid-implementation:
- **Changing `package.json` `peerDependencies`, `exports`, `files`, `bin`, or `version`** — a published-package contract. The React-vestige cleanup above lives here. Surface it; do not do it unilaterally.
- **Running `scripts/release.sh`** (or any `npm version` / tag push). Releases are the owner's call, and the script pushes directly to `master` and creates a tag that fires the release workflow.
- **Committing a `dist/` rebuild.** Only the release script does that.
- **Removing the `src/libold` gitlink** — history-touching; owner-gated.
- **Adding a test runner, linter, or formatter, or a CI workflow.** Each is a deliberate infrastructure decision plus new dependencies, and would change what every future cycle's gate actually catches. It may well be the highest-value thing this repo could gain — propose it as its own PR with the owner's say-so, never as a side effect of another change.
- **Opening a PR against `philbgarner/atomic-core`.** Contributing upstream is a cross-repo, outward-facing act against someone else's project. This loop works on the fork; upstreaming needs explicit user authorization each time.
- **Anything under `examples/**/*.png` or `LICENSE_art.txt`** — restricted artwork license.

---

### Phase 2 — Work selection

Choose 0–3 candidates to implement as a coherent PR:

- **0** if all candidates are blocked or too large — note why and skip to Phase 9.
- **1–2** is the default. Prefer candidates that touch the same subsystem or naturally complement each other.
- **3** only when all three are small and clearly independent.

When candidates have a dependency relationship, implement the foundation first.

**Tiebreaker rule.** When choosing between candidates of comparable scope and no dependency relationship, bias toward (in descending preference): documentation fixes → build fixes → small refactors → bug fixes → performance work. Per RESEARCH.md §2, autonomous-agent PRs merge at substantially higher rates for the earlier categories. This is a soft tiebreaker, not a hard exclusion — a clearly-scoped bug fix is still better than no progress, and coherent-batch grouping still takes precedence when it applies.

**Early-cycle bias.** For the first 3–5 cycles after this skill is generated, weight the tiebreaker more strongly toward documentation and build fixes — the loop has minimal prior context for harder work, and this repo's verification story is weak enough that a renderer change is expensive to prove. Once the loop has shipped several cycles successfully, the bias relaxes. This is a preference, not a rule; an explicit user instruction overrides it.

Record the work item in the PR body under `## Work in scope`, and add `Closes #N` when the work resolves an open issue on this fork (see Phase 4 for which numbers may be closed).

**Alternative cycle work modes.** Implementing candidates is the default unit of work, but a cycle may instead be devoted to one of the stages below. Both are **first-class outcomes** (not filler) and produce a normal PR through Phases 4–8. Prefer them when the candidate list is thin, blocked, or all owner-gated. **For this repo, Stage A is the high-value default** — the documentation surface (`README.md`, `FEATURE_SOURCE.md`, `docs/`) is large, hand-maintained, and nothing mechanically checks it.

#### Stage A — Documentation accuracy review (sweep)

Sweep the documentation for drift against the *actual source*, independent of any recent change — the proactive, repo-wide complement to the PR-scoped check in Phase 7. Go through every documentation source of truth (the Phase 7 table) and verify each claim against the code, config, or command it documents. **Verify against source, never memory.**

- Fix drift **in the docs**. If the *code* is what's wrong (the docs describe the intended, correct behavior), do **not** silently change code under a docs cycle — record it as a candidate and leave it for an implementation cycle.
- `docs/` is regenerated, not edited: if the sweep finds `docs/` stale, the fix is `npm run docs` plus committing its output, nothing hand-written.
- Respect the Phase 3 scope ceiling. If drift is large, fix the highest-value subset this cycle and enumerate the rest in the PR body.
- If a complete sweep finds **no** drift, say so explicitly and fall back to another work mode — an empty docs PR is not an outcome.

#### Stage B — Unit-test expansion

**Does not apply as written.** There is no test runner, no test file, and no `test` script in `package.json`; adding one is owner-gated infrastructure work (Phase 1). Do not invent a suite as a side effect of another change.

The nearest equivalent, and a legitimate cycle in its own right, is **Stage B′ — example coverage**: add or extend a page under `examples/localhost/` (and its `examples/standalone/` twin) that exercises a subsystem nothing currently demonstrates, so that the subsystem becomes checkable by a human in a browser. Rules: match the structure of a neighbouring example (`index.html` + `<name>.js` + `styles.css` + atlas), register it in the relevant `index.html` index page, keep the two example sets in parity, and reuse an existing atlas rather than adding new artwork (restricted license).

```bash
git checkout -b feature/<short-description>
```
Use `fix/<short-description>` when the work is a correction rather than new capability. Upstream's merged branches follow `features/<name>` and `bugfix/<name>`; either shape reads correctly in this history — be consistent within a cycle.

**Plan summary (re-read at the start of Phase 3).** Before exiting Phase 2, write a compressed plan in this shape:
- **Work in scope:** the candidates, or `Stage A — documentation accuracy sweep`, or `Stage B′ — example coverage (<subsystem>)`.
- **Branch:** `feature/<name>` or `fix/<name>`
- **Files I expect to modify:** path, path, ...
- **Invariants to preserve:** `npm run typecheck` exits 0 and `npm run build` succeeds; no `dist/` file is staged; nothing under `docs/` is hand-edited; every new public symbol is re-exported from `src/lib/index.ts`; `FEATURE_SOURCE.md` still names the files that implement each feature it lists; `examples/localhost` and `examples/standalone` stay in parity.

Phase 3 begins by re-reading this summary. The point is to ground the implementation in a tight statement rather than the full accumulated triage transcript — per RESEARCH.md §4, context rot degrades performance even when the window isn't full.

---

### Phase 3 — Implementation

**Localization verification.** Before writing any code, list the files this PR intends to modify and verify each one:

1. **Confirm the file exists.** `test -f <path>` or `ls <path>`.
2. **Confirm the surface area is present.** For each file, grep for the symbol, heading, config key, or behavior named in the candidate. If the candidate says "`computeDoorProgress` clamps the wrong end", run `grep -n 'computeDoorProgress' <path>` and confirm the named entity is there. If it isn't, stop and re-triage — the localization is wrong and editing here would produce a misfire. This matters more than usual in `dungeonRenderer.ts` (3372 lines) and `createGame.ts` (2642 lines), where a plausible-looking symbol name often exists in three places.
3. **Confirm the behavioral claims, not just the symbol's existence.** For each behavior asserted ("X ignores the collider flag", "Y defaults to 0"), read the code path to the point where the behavior would actually be observable. If the description and the source disagree, **the source wins**: implement/document what the code does and record the discrepancy separately.

This catches the dominant agent failure mode on uncontaminated benchmarks: finding the right file to edit, not the patch itself (RESEARCH.md §3).

Follow project conventions:

- **The public surface is `src/lib/index.ts`.** A new exported function or type is not part of the API until it is re-exported there — the IIFE global `window.AtomicCore` is built from that file, and every example is a script-tag consumer. Add the value export and the `export type` line in the section its subsystem already occupies.
- **`import type` for every type import.** `verbatimModuleSyntax` and `isolatedModules` make the alternative a build error. The codebase separates them explicitly: `import { resolveEasing } from '../animations/easing'` next to `import type { EasingFn, EasingName } from '../animations/easing'`.
- **Fix strict errors by narrowing.** `strict`, `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` are all on, and indexed reads genuinely can be `undefined`. Never reach for `!`, `as any`, or `@ts-expect-error`.
- **Style is per-file, not repo-wide.** There is no formatter. `src/lib/api/createGame.ts` uses **tabs and double quotes**; `src/lib/dungeon/doors.ts` uses **two spaces and single quotes**; both use semicolons and JSDoc `/** ... */` on exported types. Match the file you are editing and do not reformat around your change — a reformat is a whole-file diff that buries the actual edit.
- **Each source file opens with a `// <path>` comment** and, for the larger modules, a short block explaining responsibilities. Keep that header accurate when a file's role changes.
- **Never hand-edit `dist/` or `docs/`.** `dist/` is release output; `docs/` is `npm run docs` output. If public types changed, run `npm run docs` and commit *its* result.
- **Do not commit `dist/`.** `npm run build` — which you will run as part of the anchor, and must run before any browser check — rewrites tracked files under `dist/`. That is expected locally and must never be staged. See the staging rule below.
- **Examples come in pairs.** A change to `examples/localhost/<demo>/` almost always needs the same change in `examples/standalone/<demo>/`. The only intended difference is atlas loading: `<img src="atlas.png">` on localhost, an embedded Base64 data URL on standalone (regenerate with `utils/imageToBase64Js.sh`, never by hand). Example JS is plain ES with double quotes and semicolons, and pulls the API off the `AtomicCore` global.
- **Server code is plain JavaScript.** `src/server/index.js` is not typechecked by `tsc` (the `include` is `src/lib/**` only); `src/server/dungeon-entry.ts` is, but it builds through `vite.config.server.ts` with `three` aliased to `src/server/three-shim.js` — a change there needs `npm run build:server` to be meaningful.
- **Version numbers belong to the release script.** Never edit `package.json`'s `version`, and never write a `build: vX.Y.Z` commit; both belong to `scripts/release.sh`.

Universal rules:
- **Match sibling structure.** Before creating a new file in a directory, read the structure of the existing files in the same directory and conform. `grep "^##" path/to/dir/*.md` for docs, or read 2–3 neighbouring source files for code.
- **Rename siblings together.** When renaming a heading or identifier that is part of a parallel pair or series (`setFloorSkirtTiles` / `setCeilSkirtTiles`, `setSkyPanelCount` / `setCeilingPanelCount`, `floorHeightOffset` / `ceilingHeightOffset`), scan for the siblings and rename them in the same commit — including their `index.ts` re-exports, their `FEATURE_SOURCE.md` mentions, and any example that calls them.
- **Scratch-file handling in a sandboxed harness.** When a step needs a scratch file (a PR body, a commit message, `git show` output for inspection — not a project source file), prefer the `Write` tool over `> file` shell redirection, and prefer `python3 -c "import os; os.remove(path)"` over `rm` to clean up. Some harness sandboxes statically block plain `>` redirection and `rm` outright, even for files the session just created inside the working directory, while `Write` and `os.remove` are not pattern-matched the same way. The same applies to scratch directory trees (`python3 -c "import shutil; shutil.rmtree(path)"`), and creating one at all may be blocked — keep scratch work to individual files written into the existing tree.
- **Name every scratch file uniquely per repo and per cycle.** Use `atomic-core-<branch>-pr-body.md`, never a generic `pr-body.md` — the harness scratchpad is shared between concurrently running dev loops, so a colliding name is overwritten silently, with no error to notice. Write the file immediately before the command that consumes it, and re-verify anything published from it (`gh pr view <number> --json body`) rather than trusting the file.
- **Avoid command substitution in Bash tool calls.** Some harnesses' command classifiers reject `$(...)` outright, so a prescribed `--body "$(cat <<'EOF' ... EOF)"` can fail before reaching the shell. Compose long bodies with the `Write` tool and pass them by file — `--body-file` (`gh pr create`, `gh pr comment`) or `-F` (`git commit`). Likewise prefer separate `grep` invocations over `\|` alternation, and prefer one command per Bash call: this harness rejects several chained operations in a single call.

**Verification for every change.** There is no test suite; the anchor is the compiler plus a build plus a browser:

```bash
npm ci                 # node_modules is absent in a fresh clone; npm install if no lockfile match
npm run typecheck      # tsc --noEmit — the only automated gate
npm run build          # tsc && vite build — proves the bundle still emits; rewrites dist/ (do not stage)
npm run build:server   # only when src/server/** or vite.config.server.ts changed
```

**Confirm the anchor actually ran.** A zero exit status is not evidence: `tsc --noEmit` on a tree whose `include` matches nothing also exits 0. Read the output — a real `npm run build` prints the emitted `dist/atomic-core.{js,umd.cjs,iife.js}` sizes and the `vite-plugin-dts` declaration pass. Where nothing was compiled, the green verifies nothing and must not be reported as "the build passes".

**Browser verification is not optional for renderer or gameplay changes.** `tsc` never draws a frame: no geometry rebuild, no shader, no raycast, no input handling, no turn resolution is exercised by it. After `npm run build` (which is what refreshes the `dist/` bundle the pages load), serve the examples and walk the one that covers the changed subsystem:

```bash
~/pocket-rig/bin/devsrv start atomic-core-examples --port 3000 -- npx serve .
# then open http://localhost:3000/examples/localhost/ and pick the demo:
#   rendering/lighting/AO → basic, ambient-occlusion, lighting
#   doors                 → doors            camera modes → camera-modes
#   billboards/sprites    → billboard-sprites  atlas/textures → texture-loader, textureAtlas
#   minimap → minimap     inventory → inventory     themes → themes
#   sky/ceiling panels    → sky, sky-panels    layers → layering
#   multiplayer           → npm run multiplayer, then two browser tabs
```
Register the server with `devsrv` even for a one-off check — port-based process lookup is permission-denied in this sandbox, and `devsrv` is the only thing that reliably stops it again (`devsrv stop atomic-core-examples`). Walk the demo: move (WASD), turn (Q/E), and exercise the specific thing that changed. Record in the PR body whether this pass was performed or was not possible.

Fix all failures before proceeding. Never skip a gate or bypass hooks.

**Git-staging hygiene — sharper here than usual.** Stage by name; **never** `git add -A` or `git add .`. Three things in this tree make a blanket add actively harmful: `dist/` is tracked and is rewritten by every build; `src/libold` is a dangling gitlink that a blanket add can re-stage; and the harness writes `.claude/` state into the tree (`.gitignore` covers `.claude` and `CLAUDE.md`, but do not rely on it). After staging, run `git status` and confirm no `dist/`, no `src/libold`, and no `.claude/` entries are staged:

```bash
git add <files>
git status --short
git diff --cached --name-only    # confirm this list is exactly the intended files
```
If a build left `dist/` dirty and you want a clean tree, `git checkout -- dist/` — but only after the browser pass, since reverting `dist/` reverts what the examples load.

**Scope ceiling.** Before pushing, check the cumulative net diff for this cycle:
```bash
git diff --stat origin/master
```
If changes exceed **~400 net LOC** or the PR modifies more than **~10 files**, stop and rescope: drop a batched candidate, or split into a follow-up PR. **Exception:** a regenerated `docs/` tree or a paired `examples/localhost` + `examples/standalone` change can blow past the file count for mechanical reasons — proceed but state the overage and the reason in the PR body and self-review. Hard stop unchanged at **~800 LOC** or **~20 files** (excluding generated `docs/`) — at that size, agent PRs fail to merge at substantially higher rates (RESEARCH.md §2).

**Implementation summary (re-read at the start of Phase 4).** Before pushing, write a compressed implementation summary:
- **Files actually modified:** path, path, ...
- **Commit summary:** one line per commit
- **Verification result:** PASS / FAIL for each of typecheck / build / browser pass — name the command and what it printed
- **Open carryovers:** anything in scope that wasn't done and why (becomes input for Phase 9)

---

### Phase 4 — PR

**`Closes #N` works now, but only for this fork's own numbers.** Issues are enabled on `dmccoystephenson/atomic-core`, so a bare `#N` in a PR body resolves against *this* tracker — write `Closes #N` when the PR genuinely resolves that issue, and let the merge close it. Two cautions: close an issue only when the whole issue is resolved (several seeded issues bundle related items, and a partial fix should say which part it covers rather than closing the lot); and **never `Closes` an upstream number** — reference upstream as a plain link (`context: philbgarner/atomic-core#12`), which makes clear it is not being closed and cannot be, since a fork PR's references resolve against the fork.

```bash
git push -u origin feature/<short-description>
# compose the body with the Write tool to a uniquely-named scratch file first (Phase 3 scratch-file rule)
gh pr create --repo dmccoystephenson/atomic-core --base master --head feature/<short-description> \
  --title "..." --body-file <scratch-file-path>
```

**`--repo` and `--base` are load-bearing on a fork.** Left to itself, `gh pr create` in a fork targets the **parent** repository — it would open a PR against `philbgarner/atomic-core`, someone else's project, which this loop is not authorized to do (Phase 1). Always pass `--repo dmccoystephenson/atomic-core --base master`, and pass `--head` explicitly because the restricted fetch refspec leaves no remote-tracking ref to auto-detect. If a PR is opened against upstream by mistake, close it immediately and say so in the report.

PR body must include:
- Summary bullet points (what changed and why)
- `## Work in scope` — the candidates this PR implements (Phase 2)
- `## Deferred this cycle` — every candidate found at triage and not picked, one line of reasoning each (Phase 1)
- Verification checklist: `npm run typecheck`, `npm run build`, and the browser pass (performed on which demo, or why it was not possible)

There is no configured reviewer, no `CODEOWNERS`, and no reviewer bot evidenced in the fork's history (it has no PRs). Proceed directly to the self-review.

Perform a self-review. This step is anchored on external signals (the verification set, the rubric below) rather than free-form judgment — empirical findings show LLM self-critique without an external signal is unreliable and can regress quality (see RESEARCH.md §1, §5).

1. **Confirm the anchor is green on the PR head.**

   ```bash
   npm run typecheck
   npm run build
   gh pr checks <number>   # expect "no checks reported" — this repo runs CI on tags only
   ```

   **"No checks reported" is this repo's normal state, not a failure and not a green.** `.github/workflows/release.yml` triggers on `v*` tags, so no workflow ever runs on a pull request. Never record `gh pr checks` as the anchor here; the anchor is the local verification set plus the browser pass.

   If the anchor fails, fix the underlying issue, push, and re-confirm. Do not start the rubric until it passes.

   **A green compiler does not cover what it cannot execute.** `tsc` renders nothing, and `src/server/index.js` is not typechecked at all. A change under `src/lib/rendering/`, `src/lib/dungeon/`, `src/lib/turn/`, `src/lib/ui/`, `src/server/`, or `examples/` therefore needs the Phase 3 browser (or multiplayer) pass on top of the green build. State in the PR body whether that pass was performed.

   **If the anchor cannot run** — `node_modules` absent with no network for `npm ci`, or a sandbox that restricts filesystem/tool access to the repo's own working directory (common in gardener-dispatched sessions), or no browser for the manual pass — do **not** claim it green and do **not** burn the cycle fighting the sandbox. Flag it **UNVERIFIED** and gate on scope: if the PR modifies files the anchor would have validated (anything under `src/`, `examples/`, or a build config), the anchor is required — mark UNVERIFIED, do not auto-merge, and hand to a human who can run it. If the PR touches none of those (docs-only), record UNVERIFIED-not-applicable and continue, stating it in the self-review and PR body. Note that there is no CI fallback here: unlike a repo with PR workflows, an unrunnable local anchor means *nothing* verified the change, so the hand-off is the whole remedy.

2. **Read the full diff:**
   ```bash
   gh pr diff <number>
   ```

3. **Run the self-review rubric.** Score each item PASS or FAIL with a one-line justification grounded in the diff or a command output — not in judgment. Frame this adversarially: assume FAIL unless you have direct evidence of PASS. Treat an all-PASS result as suspicious; a reviewer expects you to find at least one issue.

   Universal rubric:
   - **Scope:** every file modified is necessary for the work in scope (no unrelated formatting, renames, or comment churn).
   - **Tests-new:** N/A for this repo — there is no test suite (Phase 1). State N/A explicitly rather than silently skipping, and say what stands in its place (which demo was walked, or which typecheck now covers the new type).
   - **Tests-fix (empirical, not judged):** for each bug fix, prove the defect was real by observing it. With no test runner, the observation is the demo: revert the fix (`git checkout origin/master -- <src files>`, or the merge base where `master` has moved on), `npm run build`, reload the demo page and **see the wrong behavior**; then restore (`git checkout HEAD -- <src files>`), rebuild, and see it gone. **Confirm the revert actually changed the tree** (`git status --porcelain`) before trusting either half — a revert that changed nothing produces a "still correct" reading that means nothing. Do not score this from reasoning alone; where the browser pass is impossible, score it via the ladder below.
   - **Sibling structure:** every new file matches the conventions of its directory siblings (Phase 3 rule).
   - **Sibling renames:** every renamed identifier in a parallel pair/series has its siblings renamed in the same commit — including `index.ts` re-exports, `FEATURE_SOURCE.md`, and example call sites.
   - **Docs:** every row in the Phase 7 sources-of-truth table reflects the new behavior.
   - **Work resolution:** every item in `## Work in scope` is actually changed; nothing is partially done while claiming completion.
   - **Anchor:** `npm run typecheck` and `npm run build` are green on the PR head, and the browser pass is recorded as performed or explicitly not possible.

   **Tests-fix fallback ladder when the browser pass cannot run.** Enter this only when the environment genuinely cannot execute it (no browser, no network for `npm ci`), not when it is merely inconvenient. There is no CI to substitute — this repo runs no workflow on PRs — so the ladder is short:
   1. **Compiler-visible defect.** If the bug is one `tsc` can see (a type error the fix resolves, an option that never typechecked), revert-and-run `npm run typecheck` and quote the error it prints with the fix reverted, and its absence with the fix restored. That is a real FAIL→PASS.
   2. **A stale artifact the build itself demonstrates.** If the fix corrects generated or bundled output (a `docs/` regeneration, an export missing from the IIFE), diff the rebuilt artifact before and after and quote the differing lines.
   3. **Neither is available.** Score Tests-fix **FAIL**, do not auto-merge, and hand to a human with the exact demo page and steps that would show the defect. A bug-fix PR whose evidence rests only on reasoning does not clear the regression gate.

   Repo-specific rubric items:
   - **No `dist/` in the diff:** `git diff --name-only origin/master...HEAD | grep '^dist/'` returns nothing. `dist/` belongs to the release script.
   - **No hand-edited `docs/`:** any change under `docs/` is byte-for-byte what `npm run docs` produced, and the same PR contains the source change that caused it.
   - **Public surface complete:** every new exported symbol appears in `src/lib/index.ts`, on the correct side of the value/`export type` split.
   - **Type-only imports:** every type import uses `import type` — `verbatimModuleSyntax` makes the alternative a build error.
   - **No strictness escapes:** no `!`, `as any`, or `@ts-expect-error` added; strict and `noUncheckedIndexedAccess` errors were fixed by narrowing.
   - **Style matched to the file:** indentation and quote style follow the file being edited (tabs/double quotes in `createGame.ts`, spaces/single quotes elsewhere), with no incidental reformatting of untouched lines.
   - **Example parity:** any change to a demo under `examples/localhost/` has its `examples/standalone/` twin updated, and vice versa; index pages still list what exists.
   - **No new artwork:** no `.png` added, moved, or re-embedded — `LICENSE_art.txt` is restrictive.
   - **Package contract untouched:** no change to `package.json` `version`, `peerDependencies`, `exports`, `files`, or `bin` (owner-gated, Phase 1).
   - **Gitlink untouched:** `src/libold` is not staged and its mode is unchanged.
   - **`FEATURE_SOURCE.md` updated:** any file added, removed, or repurposed under `src/lib` is reflected in both the directory-structure block and the relevant feature's **Files:** list.

4. **For each FAIL item:**
   - If the fix is mechanical and small, fix it locally, commit, push, and re-score the failed item. Do not re-score items that previously passed.
   - If the item requires judgment (e.g. "is this scope creep?"), leave it as a comment for Phase 6.

5. **Post the self-review as a plain PR comment** — not a formal review API call. The auto-mode classifier blocks `POST /pulls/<n>/reviews` (it misrepresents independent review); a plain comment keeps the audit trail without implying reviewer independence. Compose the body with the `Write` tool and pass it by file — avoid command substitution:
   ```bash
   # write the body to a scratch file with the Write tool first, named uniquely per the
   # Phase 3 scratch-file rule, e.g. <repo-root>/.self-review-<branch>.md:
   #   Self-review rubric:
   #   - Scope: PASS — <justification>
   #   - Tests-new: N/A — no test suite in this repo; <what stood in>
   #   - Tests-fix: PASS — revert + rebuild showed the defect on <demo>, restore cleared it
   #   - ...
   #
   #   <one-line summary; fold any out-of-diff observations into this body>
   gh pr comment <number> --body-file <scratch-file-path>
   ```
   Remove the scratch file afterward per the scratch-file-handling rule. Reserve inline line-anchored comments for lines this PR actually adds or changes — an inline comment outside the diff hunk is rejected (`HTTP 422: Line could not be resolved`); fold those observations into the comment body instead. Do not retry a 422 with a different line number.

6. **Cap: one intrinsic-critique pass per PR.** After Phase 6 addresses the rubric's comments, do not re-run the rubric internally. Only an external signal — a reviewer comment or an anchor failure — reopens the iteration loop. Empirical findings (RESEARCH.md §5) show repeated intrinsic critique plateaus by iteration 2 and can regress.

7. Proceed to Phase 5 — address your own comments in Phase 6.

---

### Phase 5 — Wait for review

```bash
gh pr view <number> --repo dmccoystephenson/atomic-core --comments
gh api repos/dmccoystephenson/atomic-core/pulls/<number>/comments \
  --jq '.[] | "File: \(.path)\nLine: \(.line)\nBody: \(.body)\n"'
```

If no review yet, use `ScheduleWakeup` (delay 270 s) to check again.
After 5 wakeups (~22 min) with no review, proceed anyway.

**Autonomous multi-cycle batch mode** (`/atomic-core-dev-loop until you run out of work` / `do N cycles`) **and headless dispatch** (`gardener tend`): there is no human reviewer between back-to-back cycles, so the ~22 min poll is pure latency, and in a headless run there is no `ScheduleWakeup` tool at all. Treat the self-review rubric + green anchor as the merge gate and **skip (or cap at one short poll)** this phase — except for do-not-auto-merge and UNVERIFIED PRs, which still hand off for human approval. **Stop condition:** end the batch when the only remaining candidates are blocked, owner-gated, or too large for a polish-sized PR, or when a cycle yields no appropriately-scoped work.

---

### Phase 6 — Address comments

**External vs. internal signals.** Comments from a real reviewer and anchor failures are *external* — keep iterating until each is resolved. The self-review rubric posted in Phase 4 is *internal* — once its comments are addressed here, do not re-run it. Per RESEARCH.md §1 and §5, repeated intrinsic critique without an external signal is neutral-to-harmful.

For each comment:
1. Read the comment. Read the referenced source before changing anything.
2. If correct, fix the code.
3. If it conflicts with `README.md`, `FEATURE_SOURCE.md`, the stated work item, or a project config file, reply with the evidence and do not apply the change.

Compose the commit message with the `Write` tool to a scratch file so the trailer lands on its own line, then commit with `-F` — avoid command substitution. Stage by name (never `git add -A`). **Do not hardcode the co-author model name** — defer to the harness's standing git rule, which appends the *actual running model* (a hardcoded name misattributes the commit when a different model version is running):
```bash
git add <files>
# write the commit message (description + blank line + Co-Authored-By trailer) to a scratch file via the Write tool first
git commit -F <scratch-file-path>
git push
```
Remove the scratch file afterward. Commit subjects in this history are plain sentence-case prose ("Adding way to check if we've finished moving or not.") with occasional `fix:` prefixes; `build:` is reserved for release commits. Either shape is fine — never `build:`.

Re-run `npm run typecheck && npm run build` after every fix, and redo the browser pass if the fix touched rendering or gameplay. Do not push a broken build.

---

### Phase 7 — Documentation accuracy check

This phase is **PR-scoped**: it verifies the docs against *this PR's* implementation. The proactive, repo-wide version is Stage A in Phase 2.

**Read the implementation first**, then check each doc against it.

| Doc | What to verify |
|---|---|
| `README.md` — Features | Every bullet this PR could have invalidated: option names, defaults (`moveAnimMs` 130, `fallDamageHeight` 0), event names, and method names match the source. |
| `README.md` — Quick Start | The snippet still compiles conceptually against `createGame` / `createDungeonRenderer` / `attachKeybindings` option shapes, and the script paths it names (`/dist/atomic-core.iife.js`) still exist. |
| `README.md` — Examples tables | Every listed directory exists under `examples/standalone/`; the commands (`npm run examples`, `npm install`) match `package.json` `scripts`. |
| `FEATURE_SOURCE.md` — Directory structure | Matches the real `src/lib` tree after this PR. |
| `FEATURE_SOURCE.md` — feature entries | Each feature's **Files:** list names files that exist and still implement it, with the symbols it cites still present. |
| `FEATURE_SOURCE.md` — Public API surface | Matches `src/lib/index.ts`'s re-exports. |
| `docs/**` | Regenerated with `npm run docs` if any public export, type, or doc comment changed — never hand-edited. |
| `package.json` | `scripts` reflect any command this PR added or renamed; `description`, `exports`, `files`, `bin`, `peerDependencies` unchanged unless the owner authorized it. |
| `SET_OPTIONS_PLAN.md` | If this PR implemented one of its phases, the plan says so; its `createGame.ts` line-number citations still point at what they claim. |
| `examples/localhost/index.html`, `examples/standalone/index.html` | Every demo listed exists; every demo directory is listed; the two sets stay in parity. |
| Source file headers | The `// <path>` banner and responsibilities block at the top of each modified file still describe it (`src/server/index.js` still carries the pre-fork name `r3f-crawl-lib` — fix it if this PR touches that file). |

If any inaccuracy is found: fix it, commit, and restart this phase from the top. Only proceed when a complete pass finds nothing wrong.

---

### Phase 8 — Merge

**Regression gate.** For each bug fix in this PR, verify the evidence recorded in Phase 4's Tests-fix item is empirical — the defect was *observed* with the fix reverted (in the browser, in `tsc` output, or in a rebuilt artifact) and *observed gone* with it restored — not argued. If absent, do not merge: either produce the evidence or reclassify the change. Per RESEARCH.md §3, regression evidence is the only way to distinguish a real fix from a coincidental patch. This repo has no test suite to fall back on, which makes the gate stricter here, not looser.

**Do-not-auto-merge path check.** Before invoking `gh pr merge`, list the files this PR modifies and check them against the list below. If any modified path matches, do not merge automatically — leave the PR open and report to the user for manual review.

```bash
git diff --name-only origin/master...HEAD
```

Universal entries (any repo):
- `.github/workflows/*` — CI/release config; changes affect downstream automation
- Any path under a `security/` directory
- A single file with more than 50 lines deleted (check with `git diff --stat origin/master...HEAD`)

Repo-specific entries:
- `package.json` / `package-lock.json` — the published-package contract (`version`, `peerDependencies`, `exports`, `files`, `bin`) and the dependency set
- `tsconfig.json` — compiler-flag changes alter what every future cycle's only automated gate catches
- `dist/**` — release output; a `dist/` change outside a release commit is a mistake, not a merge candidate
- `scripts/release.sh`, `scripts/release.ps1` — edits here silently change every future release
- `src/libold` — dangling gitlink; touching it is history-touching work
- `LICENSE_code.txt`, `LICENSE_art.txt` — licensing terms
- `examples/**/*.png`, `examples/**/textureAtlas*.json`, `utils/imageToBase64Js.sh`, `utils/image.ToBase64Js.ps1` — restricted artwork and the tooling that embeds it
- `src/server/**` — the multiplayer server is not typechecked and not covered by any example that runs unattended; a change here needs a two-client manual check

If no modified path matches, proceed:

```bash
gh pr merge <number> --repo dmccoystephenson/atomic-core --squash --delete-branch
git checkout master && git pull
```

**Review-ready hand-off is a valid terminal state — not a failed cycle.** If `master` turns out to be branch-protected, or repository auto-merge is disabled, a clean, anchor-green PR's true terminal state is *open + self-review posted + awaiting human approval*. **Never use `--admin`** to bypass a deliberately-configured gate, and a later self-audit must not score "could not auto-merge" as a failure. A do-not-auto-merge path match blocks **autonomous** merge only: if the owner explicitly authorizes the merge after being shown which protected path matched, that authorization satisfies the hold — proceed (still run the Phase 7 docs sweep and the regression gate first), and state in the report which path was overridden and by whose authorization.

**Standing merge authorization does not satisfy the anchor gate.** A run-level pre-authorization granted before this PR existed (a headless dispatch launched with a blanket "may merge" flag) conveys *permission*, not *validation* — where Phase 4 marked the PR UNVERIFIED because the anchor could not run on anchor-relevant files, the hand-off stands regardless of merge authority. With no CI on pull requests, there is no second signal that arrives later; the human is the whole verification path.

**Never `git push` to `master` directly, and never create a tag.** Both are release actions belonging to `scripts/release.sh` and the owner.

**Proceed to Phase 9.** Do not stop here — the self-audit is mandatory whether or not anything was implemented.

**Cycle summary (re-read at the start of Phase 9).** After merging, write a compressed cycle summary:
- **Work completed:** the candidates implemented
- **PR(s) merged:** `#X`
- **Surprises:** anything that didn't go as expected (anchor for the self-audit's reflection prompts)
- **Carried forward:** the deferred candidate list, so the next cycle starts from it rather than rescanning cold

---

### Phase 9 — Self-audit

1. **Check for duplicates** before filing:
   ```bash
   gh issue list --repo dmccoystephenson/atomic-core-dev-loop --state open
   ```

2. **Run the self-audit rubric.** For each item, mark PASS / FAIL / "no signal this cycle" with a one-line justification grounded in the cycle's actual events:
   - **Identity drift:** did this cycle act in accordance with the identity stated at the top of this skill?
   - **Instruction clarity:** did any step require interpretation or produce a wrong first attempt?
   - **Edge case coverage:** did any failure mode arise that the Edge cases section does not cover?
   - **Phase friction:** did any phase require significantly more or fewer steps than expected?
   - **Drift candidates:** did any decision feel like it should be encoded in `create-dev-loop.md` rather than re-discovered each cycle?
   - **External-signal quality:** did the anchor (typecheck + build + browser pass) actually catch the kind of issue it exists to catch this cycle — and where it did not, was that recorded honestly?

   Per RESEARCH.md §1 and §5, structured rubrics outperform free-form prompts for LLM critique — the same logic that grounds the Phase 4 self-review applies to this retrospective.

3. **For each FAIL item not already tracked, file a labeled issue** so triage knows where the gap belongs:
   ```bash
   gh issue create --repo dmccoystephenson/atomic-core-dev-loop \
     --title "<short description of the gap>" \
     --body-file <scratch-file-path> \
     --label <one-of: template-rule | repo-specific | edge-case | research-gap | process>
   ```

   Label taxonomy:
   - `template-rule` — should be promoted into `create-dev-loop.md`
   - `repo-specific` — belongs only in this skill
   - `edge-case` — should be added to Edge cases
   - `research-gap` — new finding worth a RESEARCH.md entry
   - `process` — meta-issue about how the loop runs

4. **Do not implement or merge changes to the skill itself** — file issues only so a human reviews and approves skill edits.

5. If nothing notable is found, note that explicitly and proceed.

---

### Phase 10 — Next cycle

Return to Phase 1.

---

## Edge cases

**The anchor fails during implementation:** diagnose; never bypass it. There is no `--no-verify` equivalent to reach for and no suite to mark skipped — a failing `tsc` is the only automated warning this repo gives you.
**The anchor fails after addressing a comment:** same rule.
**`npm ci` fails or `node_modules` is absent with no network:** the anchor cannot run. Flag **UNVERIFIED** and gate on scope (Phase 4) — do not hand-verify TypeScript by reading it and call that green. There is no CI to catch it later.
**No browser is available for the manual pass:** say so plainly in the PR body. A green `tsc` on a renderer change is not evidence the dungeon draws; treat the PR as UNVERIFIED for anchor-relevant files and hand off.
**`gh pr checks` reports no checks:** expected — `.github/workflows/release.yml` runs on `v*` tags only, so no workflow ever runs on a PR. Do not read this as a pending or failing check, and do not add a CI workflow to "fix" it mid-cycle (owner-gated, Phase 1).
**`gh pr create` targets `philbgarner/atomic-core`:** this is a fork, and that is `gh`'s default base. Always pass `--repo dmccoystephenson/atomic-core --base master --head <branch>`. If a PR lands upstream by mistake, close it immediately with an explanation and reopen against the fork — an unsolicited PR on someone else's project is not this loop's call to make.
**`gh pr create` fails with "you must first push the current branch to a remote" despite a successful `git push -u`:** the clone's fetch refspec is restricted to `master` only (confirm with `git config --get-all remote.origin.fetch`), so no remote-tracking ref exists for the feature branch and `gh`'s auto-detection fails. Pass `--head <branch>` explicitly rather than retrying the push.
**The clone is shallow (`git rev-parse --is-shallow-repository` → `true`):** gardener clones this repo at depth 1. `git fetch --unshallow` before rebasing or reading history; a shallow tree makes `git rebase origin/master` and `git log` misleading, and hides the upstream conventions this skill was derived from.
**`git status` shows `dist/` modified after a build:** expected — `npm run build` rewrites tracked output. Never stage it. `git checkout -- dist/` restores the committed bundle, but only do that after the browser pass, since the examples load exactly those files.
**A change looks correct in the source but the demo behaves as before:** the page is loading the committed `dist/` bundle, not `src/`. Run `npm run build` and hard-reload. This is the single most common false negative in this repo.
**`src/libold` appears as an empty directory or shows up in a diff:** it is a gitlink (mode `160000`) with no `.gitmodules` — an accidentally-committed nested repo. Do not stage it, do not `git submodule add` it, do not delete it as cleanup; removing it is owner-gated history work.
**A doc claims React/R3F and the code has none:** known, deliberate-looking drift between `package.json` (description, `peerDependencies`, the Vite React plugin) and `README.md`'s "No React, no JSX". The peer-dependency set is a published contract — surface it to the owner; do not resolve it in passing.
**A candidate requires an owner-gated operation (package contract, release, `dist/` commit, new test/lint/CI infrastructure, upstream PR, artwork):** recognize it at triage (Phase 1) and surface it for explicit authorization rather than attempting it mid-cycle.
**A demo works on `examples/localhost/` but not `examples/standalone/`:** the standalone set embeds its atlas as Base64 to survive `file://`. Regenerate with `utils/imageToBase64Js.sh` rather than hand-editing the data URL, and check the fix landed in both twins.
**A `gh` command fails with a transient network/transport error (`http2: client conn could not be established`, `unexpected EOF`):** retry once or twice before treating it as blocked — do not let the first failure silently drop a self-review, commit, or comment.
**A Bash call is rejected for chaining or command substitution:** this harness's classifier rejects multi-operation commands and `$(...)`. Split into one command per call and pass long bodies by file (`--body-file`, `-F`) per the Phase 3 rules.
**Autonomous multi-cycle batch or headless dispatch:** skip/cap the Phase-5 wait (rubric + green anchor is the gate); still hand off do-not-auto-merge and UNVERIFIED PRs. Stop when only blocked, owner-gated, or too-large work remains, or a cycle yields no scoped work. In a headless run there is no `ScheduleWakeup` and no way to ask a question — end with the decision the owner needs to make rather than guessing at it.
**A concurrent session holds the tree or an open PR:** adopt its PR (bring current with `master`, re-run the anchor, review, merge if green) rather than doing nothing. In a headless dispatch the run owns a dedicated clone, so do not create a `git worktree`; if one is genuinely needed elsewhere, put it inside the checkout (`.worktrees/<branch>`, added to `.git/info/exclude`), never in `/tmp` — a path-restricted sandbox refuses every operation outside the checkout.
**The loaded skill may itself be stale:** this file is read from a local clone that nothing pulls. Before reporting the skill as out of date, `git -C ~/local-skills/atomic-core-dev-loop fetch -q origin` then `git -C ~/local-skills/atomic-core-dev-loop status -sb | head -1`. A checkout `behind` origin reads exactly like a stale skill from inside a session.
**A review comment is a false positive:** reply with evidence, do not apply the change.
**Branch is behind `master` or has a merge conflict at Phase 8:** rebase, re-run the **same verification set Phase 3 runs** (not just the typecheck), and force-push before retrying the merge:
```bash
git fetch origin
git rebase origin/master
npm run typecheck
npm run build
git ls-remote origin refs/heads/<branch>   # full 40-char SHA for the lease below
git push origin <branch>:<branch> --force-with-lease=<branch>:<full-40-char-sha>
```
**`(stale info)` has three unrelated causes; only one justifies backing off.** A bare `--force-with-lease` compares against a maintained remote-tracking ref, which this clone does not have (refspec covers `master` only), so the push is rejected as `(stale info)`. The explicit `--force-with-lease=<branch>:<sha>` form fixes that, but the expected value is compared literally and abbreviations are never expanded — a short SHA from `git log --oneline` is rejected with the *same* message. A restricted refspec, an abbreviated SHA, and a genuine concurrent push all read identically. Do not treat `(stale info)` as "another session pushed" until the expected SHA has been confirmed full 40-character and re-read from the remote at push time (`git ls-remote`). Never fall back to a plain `--force`, which clobbers unseen work in exactly the one case that warranted stopping.

If the rebase produces conflicts that cannot be resolved automatically, close the PR and delete the branch, then return to Phase 1:
```bash
gh pr close <number> --repo dmccoystephenson/atomic-core --comment "Closing: unresolvable merge conflict after rebase."
git checkout master
git push origin --delete feature/<name>
```
**Two candidates conflict mid-implementation:** finish the further-along one; record the other in the deferred list with what blocked it.
**Cycle exceeds abort budget:** if the cycle's tool calls exceed ~500 or accumulated context exceeds ~200k tokens without converging, abort rather than push through. The half-life model (RESEARCH.md §4) predicts persistence past a budget is strictly worse than restart with fresh context. Steps:
1. Mark the in-flight PR `Draft` (`gh pr ready --undo <number>`) or, if no PR is open, push the branch with a `WIP:` commit so work isn't lost.
2. File a gap issue on `dmccoystephenson/atomic-core-dev-loop` titled `Abort budget exceeded on <branch>` with: **Work in scope**, **Files modified so far**, **Where convergence stalled** (last phase reached and what blocked it), **Suggested next attempt**.
3. Exit. Do not return to Phase 1 in the same context — restart in a fresh session.
**No candidates and no improvements found:** report what was checked; end the loop.
