# Workflow Templates

Starter workflows for bufbuild repositories, available from the repository
**Actions → New workflow** page.

## dependabot-automerge

Enables GitHub native auto-merge on Dependabot PRs that match a confidence
policy, then lets GitHub merge once all required status checks pass. By
default it auto-merges PRs that fix a Dependabot security alert (when an
alert-lookup token is configured) and routine patch/minor version bumps.
Major updates always wait for a human.

This follows GitHub's documented approach for auto-approving and
auto-merging Dependabot PRs: [Automating Dependabot with GitHub
Actions](https://docs.github.com/en/code-security/tutorials/secure-your-dependencies/automate-dependabot-with-actions),
using the official
[dependabot/fetch-metadata](https://github.com/dependabot/fetch-metadata)
action.

### Merge strategies

The template supports two merge strategies. Pick one based on whether the
repository has required status checks. Neither ever holds a runner
waiting on CI.

**A. GitHub native auto-merge (default).** `gh pr merge --auto` hands the
PR to GitHub, which merges it once the default branch's required status
checks pass. Costs no CI minutes. It merges **immediately** if the default
branch has no required status checks, so use this only on repositories
that have them.

**B. Defer to the nightly sweep (`defer-to-sweep: true`).** The reactive
workflow only labels eligible PRs; the `dependabot-automerge-sweep`
workflow merges them on its next scheduled run, once CI has finished. Use
this on repositories with **no** required status checks (for example,
repos left intentionally unprotected), where native auto-merge would merge
immediately. No runner ever waits on CI: the reactive run exits in
seconds, and the sweep takes a single look at each PR — green checks merge
immediately, anything pending or failing is skipped and retried the next
night. PRs on a repository with no CI at all merge as-is: such a
repository already accepts every change unverified, and the label vetted
the update level. Trade-off: merges land at the sweep's cadence (next
morning), not same-hour. Requires the sweep template to also be
installed.

Neither strategy configures repository settings; a workflow template only
writes the workflow file. Complete the prerequisites below by hand.

**Repository with no CI at all:** both strategies still merge — such a
repository already accepts every change unverified, and letting
Dependabot PRs stack up unmergeable helps nobody. Strategy A merges
eligible PRs immediately; strategy B merges them on the nightly sweep,
announced with a notice. The eligibility policy (patch/minor only,
majors wait for a human) still applies either way.

#### Prerequisites for strategy A — order matters

Complete steps 1 and 2 **before** adding the workflow, or the first
Dependabot PR merges immediately with no gate.

1. Add a ruleset (or branch protection) on the default branch with
   **required status checks** naming the repository's CI jobs, for example
   `ci (go:latest)`, `conformance (go:latest)`, and so on.
2. Enable **Allow auto-merge** in the repository settings
   (Settings → General → Pull Requests).
3. Add the `dependabot-automerge` workflow from the template.

#### Prerequisites for strategy B

1. Add this workflow from the template and set `defer-to-sweep: true` in
   the `with:` block. No repository settings changes are required.
2. Also add the `dependabot-automerge-sweep` workflow from its template —
   without it, labeled PRs never merge. If the repository runs CI on pull
   requests, the sweep gates on it; if it runs none, eligible PRs merge
   as-is on the nightly pass.

### Options

The reusable workflow accepts:

| Input | Default | Meaning |
|-------|---------|---------|
| `merge-method` | `squash` | `merge`, `squash`, or `rebase`. |
| `target` | `minor` | Highest non-security semver update type to auto-merge (`patch` or `minor`). |
| `security-only` | `false` | Only auto-merge PRs that fix a security alert. Requires `alert-lookup-token`. |
| `defer-to-sweep` | `false` | Strategy B: label eligible PRs instead of merging, and let the nightly sweep merge them once CI has finished. Requires the sweep workflow. |
| `eligibility-label` | `automerge: eligible` | Label recorded on PRs that pass the merge policy (removed again if re-evaluation declines). The nightly sweep merges by this label. |

Secret `alert-lookup-token` (optional): a token that can read Dependabot
alerts (fine-grained PAT or GitHub App installation token). The default
`GITHUB_TOKEN` cannot read alerts, so without this secret the workflow
gates purely on semver level. Because Dependabot-triggered workflows read
from the **Dependabot** secret store, add it under
Settings → Secrets and variables → **Dependabot** (repo or org), not under
Actions.

### Caveats

- Merges armed with `GITHUB_TOKEN` do not fire `push` workflows on the
  default branch (GitHub suppresses runs for token-initiated events). PR
  checks still gate the merge, but repositories with automation on
  `push` to the default branch (releases, deploys, docs) should arm
  auto-merge with a GitHub App token instead.
- Repositories using a merge queue cannot arm auto-merge with
  `GITHUB_TOKEN`; they need a GitHub App or PAT with merge permission.
- "Require review from Code Owners" stalls the pipeline: the workflow's
  approval comes from `github-actions[bot]`, which cannot satisfy a code
  owner requirement. Some branch protection configurations also don't
  count bot approvals toward a generic "require approvals" count at all —
  verify the workflow's approval actually satisfies your ruleset before
  relying on it. Plain "dismiss stale approvals" is fine — the workflow
  re-approves on every Dependabot push.
- Authorization is layered. The reusable workflow gates on the PR's
  original author (`github.event.pull_request.user.login`), which blocks
  attacker-authored PRs that Dependabot merely synchronized (the
  "`@dependabot recreate` pwn request" spoof). The starter stub
  additionally gates on `github.actor`, so new runs skip when a human
  pushes commits to a Dependabot PR. Note that auto-merge armed by an
  earlier run **stays armed**, so human commits pushed to an armed PR
  still merge once checks pass — enable "dismiss stale approvals" in
  repos that require approvals if human pushes must force a fresh review,
  or disable auto-merge on the PR before pushing.
- Consider enabling Dependabot [grouped security
  updates](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/configure-security-updates#grouping-dependabot-security-updates-into-a-single-pull-request)
  at the org level so a multi-CVE disclosure produces one PR per manifest
  instead of many.

## dependabot-automerge-sweep

A **scheduled** counterpart to `dependabot-automerge`. The two split the
job: the reactive workflow classifies each Dependabot PR precisely at
`pull_request` time (when `dependabot/fetch-metadata` works — semver
level, grouped-update members, security alert state) and **labels**
eligible PRs with `automerge: eligible`. The sweep runs on a cron (and on
manual dispatch), finds open Dependabot PRs carrying that label, and
merges them. Policy lives in exactly one place; the sweep never
classifies anything.

**The sweep does nothing alone.** Without the reactive
`dependabot-automerge` workflow also installed, no PR is ever labeled and
the sweep finds nothing. Install the reactive template first. The sweep
is required for strategy B (`defer-to-sweep: true`) and optional for
strategy A, where it acts as a backstop that re-drives PRs whose per-PR
run never armed the merge.

**No runner time is spent waiting on CI.** The sweep takes a single look
at each labeled PR: checks all green → merge immediately; anything
pending or failing → skip, retried on the next scheduled run; no CI
checks at all → merge as-is, announced with a notice (a repository
without CI already accepts every change unverified). By sweep time CI
has normally been finished for hours, so merges are instantaneous.

### Why a label handoff

A scheduled run has no triggering PR, so `dependabot/fetch-metadata`
cannot run there — its classification only exists during the
`pull_request` event. Recording the verdict as a label carries that
precise classification over to schedule time, instead of re-deriving it
from PR titles (which fails on grouped updates). Two useful side effects:

- **Human veto:** remove the label from a PR and the sweep skips it. The
  reactive workflow re-evaluates on every Dependabot push, re-labeling or
  unlabeling as the policy verdict changes.
- **Visible state:** eligibility is inspectable on the PR itself rather
  than buried in workflow logs.

Two caveats:

- The reactive workflow skips runs when a human pushes to a Dependabot PR
  (actor gate), so a label applied earlier survives human commits. The
  sweep would still merge that PR once checks pass — the same caveat as
  GitHub native auto-merge staying armed. Remove the label (or close the
  PR) if human commits must force a fresh review.
- The label is dual-use. Anyone with triage access can also **add** it to
  a Dependabot PR the policy declined (say, a major bump) to deliberately
  arm it for the next sweep — useful, but be aware the sweep trusts the
  label without re-checking policy. It still only merges
  Dependabot-authored PRs whose checks are green (or that have no CI to
  check).

### Sweep options

The sweep accepts `merge-method`, `eligibility-label` (must match the
reactive workflow's), and `automerge-workflow` — the reactive workflow's
name, whose check runs are ignored so pending/failing decisions reflect
the repository's real CI rather than the labeler itself, and so merges
on CI-less repositories are announced as such. It needs no alert-lookup
token: security classification already happened in the reactive run.

The cron schedule lives in the starter template (`schedule:`), fires only
on the **default branch**, and is suppressed after 60 days of repository
inactivity. `workflow_dispatch` lets you run it by hand from the Actions
tab. Skipped PRs (checks pending or failing) are reported as notices and
retried next run; a `gh` error merging a PR is reported as a warning and
fails the run so it is visible.
