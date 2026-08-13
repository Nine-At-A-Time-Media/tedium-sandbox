# tedium-sandbox

Throwaway sandbox repo for end-to-end validation of the tedium merge queue
(Nine-At-A-Time-Media/template-tools `packages/naatm-tedium`), per
template-tools issues #386 (e2e matrix) and #387 (bootstrap).

The `tedium-nonprod` GitHub App is installed on this repo **only** --
containment per template-tools #394. Nothing here is real; destructive
cases (conflicts, red-bar bisection, cancel storms) run for real.

## How the bar works

CI passes iff `status.txt` contains exactly `green`. A PR that changes it
to anything else produces a red bar on demand -- that is the whole test
harness.

## Driving it

Comment on a PR:

- `tedium r+` (alias `tedium land`) -- queue for merge via `tedium/merge`
- `tedium try` (alias `tedium dryrun`) -- dry-run batch via `tedium/try`
- `tedium r-`, `tedium retry`, `tedium ping` -- the usual bors family
