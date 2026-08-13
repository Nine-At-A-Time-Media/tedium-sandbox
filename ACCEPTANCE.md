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
| P1 | Repo exists with `tedium.toml` (`status = ["ci"]`, `timeout_sec = 3600`) | file at repo root |
| P2 | CI workflow job named exactly `ci`, triggering on push to `main`, `tedium/merge`, `tedium/try` + `pull_request` | `.github/workflows/ci.yml` |
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
| A2 | `tedium dryrun` | `tedium/try` branch created, `Try #<pr>:` commit, `ci` run green, then a `## try` success comment on the PR | branch + CI wiring, attemptor |
| A3 | `tedium land` | PR batched onto `tedium/merge`; when `ci` is green there, `main` fast-forwards to include it; PR closed as merged | full landing path |
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
   status, `main` does NOT move.

Further destructive cases (conflicts, bisection, cancel storms, priority,
squash double-bar): template-tools#386 matrix.

## Offboarding

Runbook `template-base/docs/ops/runbook.tedium-onboarding.md` section
"Offboarding": uninstall App from the repo, delete `tedium/*` branches,
remove `tedium.toml` + CI trigger lines.
