# CyberChef Fork — CI/CD Rebuild Roadmap

## Context
This is a personal fork of CyberChef (GCHQ), forked to learn DevOps hands-on and to have a concrete, demonstrable project for a resume/portfolio. Rather than keeping upstream's GitHub Actions setup, the plan is to delete it and rebuild CI/CD from scratch, one stage at a time, each stage its own commit/PR on `feat/ci-cd-learning` — so the git history itself tells a story of progressive capability (lint → test → build → security → deploy → release) rather than landing as one large, unreviewable dump.

Repo facts that shape the plan:
- `package.json` scripts: `lint` (`npx grunt lint`), `lint:grammar` (`cspell ./src`), `test` (unit + node tests), `testnodeconsumer`, `testui` (Puppeteer/Nightwatch, needs Chrome + xvfb), `build`/`node` (`npx grunt prod`/`node`).
- `engines.node`: `>=24 <25` — all workflows must pin Node 24.
- `.github/dependabot.yml` already exists and tracks `npm`, `github-actions`, and `docker` ecosystems weekly, grouped by patch/minor — this can be leaned on directly in later stages instead of building dependency-update automation from scratch.
- A `Dockerfile` already exists at the repo root — Stage 6 builds on it rather than writing one.
- Upstream's original workflows (`master.yml`, `pull_requests.yml`, `releases.yml`, plus two CLA-bot workflows) were deleted; a copy is kept in the session scratchpad for reference only, not for copying wholesale.

## Overall stage roadmap

| Stage | Name | Goal | Status |
|---|---|---|---|
| 1 | Baseline CI | Lint + unit tests on every PR | **Done** (`ci.yml`, commit `8624f512`) |
| 2 | Build verification | Prove `grunt prod` succeeds; upload artifact | **In progress** (`build` job added; `if: always()` bug still outstanding) |
| 3 | Matrix + caching | Node version matrix, dependency caching | Not started |
| 4 | Security scanning | CodeQL + `npm audit` as required checks | Not started |
| 5 | Browser/UI tests | Reintroduce Puppeteer/Chrome UI tests | Not started |
| 6 | Docker build | Build image from existing `Dockerfile`, push to GHCR (your namespace) | Not started |
| 7 | Deploy | Auto-deploy `master` to your own GitHub Pages | Not started |
| 8 | Release automation | Tag-triggered release: build, notes, attach zip to GitHub Release | Not started |

Each later stage is additive to `ci.yml` (new jobs) or a new workflow file triggered on `push` to `master` / on tag, kept separate from the PR-gating `ci.yml` so PR feedback stays fast.

## Stage 1 — Baseline CI (detail)

**Status: implemented and committed** (`.github/workflows/ci.yml`, commit `8624f512` on `feat/ci-cd-learning`, pushed to origin).

**Trigger:** `pull_request` (`opened`, `synchronize`, `reopened`) + `workflow_dispatch` for manual runs.

**Permissions:** `contents: read` only — no write scopes needed since this stage only reads code and reports status.

**Job `lint-and-test`** (`ubuntu-latest`):
1. `actions/checkout@v5`
2. `actions/setup-node@v5` with `node-version: 24`
3. `npm ci` — clean, lockfile-exact install (not `npm install`, so CI matches what a fresh clone would get)
4. `npm run lint` — runs `npx grunt lint` (ESLint over `src/`)
5. `npm test` — runs the Grunt config-test step + the two Node-based test suites (`tests/node/index.mjs`, `tests/operations/index.mjs`)

**Known simplification vs. upstream, intentionally deferred:** actions are pinned to floating major tags (`@v5`) rather than upstream's pinned commit SHAs. Since `.github/dependabot.yml` already has a `github-actions` ecosystem entry, SHA-pinning can be introduced later (Stage 4 or as a standalone hardening pass) and Dependabot will keep the pins current automatically — no need to hand-roll that now.

**Verification:**
- Open a PR (or push to an existing PR branch) on the fork and confirm the `CI / lint-and-test` check appears and passes in the Actions tab.
- Locally (if Node 24 is available): `nvm install 24 && npm ci && npm run lint && npm test` should exit 0 before pushing, to avoid burning CI minutes on failures catchable locally.
- Deliberately break a lint rule or a test locally, push, and confirm the check goes red — proves the gate actually blocks bad PRs, not just that it runs.

## Stage 2 — Build verification (in progress)

**Goal:** Catch build-only failures (e.g. bundler errors) that unit tests don't cover, and produce a downloadable artifact for manual inspection.

**Changes to `ci.yml`:** a `build` job was added (`needs: lint-and-test`) with checkout, Node 24 setup, `npm ci`, `npm run build`, and an `actions/upload-artifact` step for `build/prod/*.zip` (5-day retention).

**Outstanding issue:** the `build` job currently has `if: always()`, which makes it run even if `lint-and-test` fails — the opposite of the intended gating. Should be changed to `if: success()` (or the `if:` removed entirely, since that's the default behavior for a job with `needs:`).

**Verification:** open a PR, confirm the build job only runs after lint/test pass (once the `if: always()` bug is fixed), and download the artifact from the workflow run summary to confirm it's a valid CyberChef build.

## Verification for this plan overall
This is an infrastructure/process plan, not a single code change — "testing" it means executing each stage in turn on `feat/ci-cd-learning`, confirming the corresponding GitHub Actions check appears correctly in the Actions tab and in the PR checks UI, then committing that stage before moving to the next.
