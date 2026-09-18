# tedium acceptance tests (nonprod)

Re-runnable smoke/acceptance sequence for a tedium instance, executed
against this sandbox. First run: 2026-08-13 (log in template-tools#386).
Worker-side smoke tests (health, webhook auth, dashboard Access) also
live in `template-tools/packages/naatm-tedium/smoketest/`.

Substitute `$HOST` = `tedium.nonprod.api9.com` (nonprod) or
`tedium.api9.com` (production), and `$REPO` = this repo's
`owner/name`.

## S. Worker smoke (no repo needed)

| # | Command | Expect |
|---|---------|--------|
| S1 | `curl -s -o /dev/null -w '%{http_code}' https://$HOST/health` | `200` |
| S2 | `curl -s -o /dev/null -w '%{http_code}' -X POST https://$HOST/webhook` | `401` (unsigned) |
| S3 | `curl -s -o /dev/null -w '%{http_code}' https://$HOST/` | `302` to Cloudflare Access login |

## P. Prerequisites (once per repo)

| # | Step | Verify |
|---|------|--------|
| P1 | Repo exists with `tedium.toml` in the go-live shape: `status = ["gate"]`, `pr_status = ["review-settled"]`, `block_labels = ["hold"]`, `max_batch_size = 3`, `use_codeowners = true`, `timeout_sec = 3600` | file at repo root |
| P2 | CI workflow with a `ci` job and an always-present `gate` job needing it, triggering on push to `main`, `tedium/merge`, `tedium/try` + `pull_request`; a `review-settled` workflow calling the tds-utils reusable workflow; `CODEOWNERS` naming `tedium.toml`, `CODEOWNERS`, `.github/workflows/` | `.github/workflows/ci.yml`, `.github/workflows/review-settled.yml`, `CODEOWNERS` |
| P3 | Green/red toggle: CI passes iff `status.txt` contains exactly `green` | `grep -qx green status.txt` |
| P4 | GitHub App (`tedium-<env>`) installed, **Only select repositories** -> this repo only | `gh api orgs/<org>/installations` shows `repository_selection: selected` |
| P5 | App events include at least `issue_comment`, `check_suite`, `pull_request` | same API call, `events` array |

Note (first run): the App is org-private, so the sandbox must live in
the owning org -- GitHub refuses to install it on personal repos.

## A. Acceptance sequence (throwaway PR)

Open a PR with a trivial green change:

    git checkout -b test/kick-the-tires-NNN
    printf '\nTire-kick PR marker: NNN\n' >> README.md
    git commit -am "Tire-kick NNN" && git push -u origin HEAD
    gh pr create --title "Tire-kick NNN" --body "tedium acceptance"

Then, each step is a PR comment; watch with
`gh api repos/$REPO/issues/<pr>/comments --jq '.[] | .user.login + ": " + .body'`.

| # | Comment | Expect | Proves |
|---|---------|--------|--------|
| A1 | `tedium ping` | `pong` from `tedium-<env>[bot]` within seconds | webhook delivery + enrollment |
| A2 | `tedium dryrun` | `tedium/try` branch created, `Try #<pr>:` commit, `gate` run green, then a `## try` success comment on the PR | branch + CI wiring, attemptor |
| A3 | request a Copilot review, resolve its threads, then `tedium land` | `review-settled` green on the head first (red before the review, red while a thread is open); PR batched onto `tedium/merge`; when `gate` is green there, `main` fast-forwards to include it; PR closed as merged | full landing path incl. the review gate |
| A4 | Dashboard at `https://$HOST/` lists the repo (behind Access login) | dashboard + syncer |

### Timing caveat found on first run (A2/A3)

The App subscribes `check_suite` but NOT `check_run`/`status`. The
check_suite-completed webhook triggers an attemptor/batcher poll, but the
attemptor's poll is gated on `last_polled + batch_poll_period_sec`
(default 1800s) and the webhook path does not short-circuit that gate
(the batcher path does). Net effect: dryrun results can take up to ~30
minutes to report even though CI finished in seconds. Tracked in
template-tools (attemptor poll-gate issue). Do not declare A2 failed
before ~35 min have passed.

## R. Red-bar case (destructive matrix entry)

1. PR that sets `status.txt` to `red` -> `tedium land`.
2. Expect: batch fails on `tedium/merge`, bot reports the failed `ci`
   status, `main` does NOT move, PR stays open.

Verified 2026-08-14 (sandbox PR 3): failed in ~40s, main unmoved.

## M. Destructive matrix (each step re-runnable)

### M1. Clean-state cancel (also template-tools#425 criterion a)

1. On a green PR: `tedium land`, then `tedium cancel` within ~30s
   (pre-build window).
2. Expect: bot replies `Canceled.`; NO batch build results from the
   canceled land; a fresh `tedium land` is accepted (NOT "Already
   running a review") and merges normally.

Verified 2026-08-14 (sandbox PR 2): fresh land accepted 21s after
cancel, merged ~60s later. Caveat: cancel strictly mid-CI (~7s window
here) not yet exercised; ghost-merge after cancel was only ever seen on
the #421 crash-corrupted state.

### M2. Red-bar bisection

1. Have one green PR and one red PR (status.txt red) open.
2. Comment `tedium land` on the green and `tedium retry` (or `land`) on
   the red within ~10s (batch_delay window) so they batch together.
3. Expect, in order:
   - combined `Merge #A #B` build on `tedium/merge` goes red;
   - both PRs get `Build failed (retrying...)` -- the split;
   - the red PR's solo `Merge #B` fails -> `Build failed` names the
     culprit, PR stays open;
   - the green PR's solo `Merge #A` passes -> merged, main advances.

Verified 2026-08-14 (sandbox PRs 3+4): full sequence in ~3 min
(02:46:52 pair red -> 02:47:31 culprit isolated -> 02:49:35 green
landed).

### M3-M5. Remaining (unrun)

- Conflict: two PRs touching the same line; land both; expect the
  conflicted one reported and retried solo.
- Priority: `tedium land p=10` jumps the queue.
- Squash double-bar: `use_squash_merge = true` + a PR changing
  `.github/workflows/` -- see template-tools#385/#386.

## T. Go-live cases (added 2026-09-18; template-tools#386 go-live note)

These exercise what the go-live set adds. Run with the go-live config in
P1/P2 and the nonprod worker deployed with template-tools#594, #595 and
#597's consumers.

### T1. A batch that edits tedium.toml is judged by main's rules (#595)

1. Open a PR whose only change is `tedium.toml` with `status = []` and
   `block_labels = ["x"]` (a config that validates and, if the PR's copy
   were in force, would make the batch green instantly with no check).
2. Get it to `review-settled` green, then `tedium land`.
3. Expect: tedium seeds a `gate` status row and the batch waits for
   `gate` on `tedium/merge` (`Build succeeded: gate` once CI reports);
   it does NOT complete before the check runs. Because CODEOWNERS names
   `tedium.toml`, expect the land to be refused first with the
   code-owner-approval preflight message until an owner approves --
   that refusal is itself part of the case.
4. Reverse the change (or close the PR) so main keeps the go-live shape.

### T2. A paused project refuses land (#594)

1. `cloudflared access curl https://$HOST/api/admin/pause -X POST -H
   'content-type: application/json' -d '{"project":"$REPO","reason":"T2"}'`
   (or the dashboard's Pause button). `GET /api/admin/pauses` lists it.
2. On a green, review-settled PR: `tedium land`.
3. Expect: a bot comment naming the pause and the reason `T2`; no batch;
   `tedium dryrun` on the same PR still runs (dryrun is not paused).
4. `POST /api/admin/resume` with the same project; expect the queue to
   move at once: `tedium retry` (or `land`) on the PR lands it without
   waiting for the 30-minute sweep.
5. Global variant: pause with no `project`; the same PR is refused; the
   dashboard shows the banner on every repo; resume with no `project`.

### T3. A `hold` label keeps a PR out of the queue

1. Label a green PR `hold`; `tedium land`.
2. Expect the block-label preflight refusal (`:-1: Rejected by label`;
   tedium's reason tag is `blocked_labels`, bors parity, while the
   `tedium.toml` key is `block_labels`); remove the label, `tedium retry`,
   expect it to land.

### T4. A stale review is refused (review-settled freshness)

1. On a PR with `review-settled` green, push one more trivial commit.
2. `tedium land` before Copilot re-reviews: expect the `pr_status`
   preflight refusal (`review-settled` is red on the new head: "newest
   review ... is on <old sha>").
3. Re-request Copilot, resolve, comment; expect green, then land.

### Rollout order once T1-T4 pass here

tds-utils -> Skills -> template-tools: on each, one `tedium dryrun` and
one trivial `tedium land` (a README marker, as A1-A3 above) before any
real PR is landed by an agent.

## Offboarding

Runbook `template-base/docs/ops/runbook.tedium-onboarding.md` section
"Offboarding": uninstall App from the repo, delete `tedium/*` branches,
remove `tedium.toml` + CI trigger lines.
