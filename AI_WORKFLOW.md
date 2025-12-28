# AI_WORKFLOW (nmoe)

This repo is optimized for **parallel AI-agent development** with a **merge-captain** workflow and **maintainer-controlled B200 integration gates**.

If you read only one thing: **Issues are contracts** and **PRs are attestations**.

## Non-negotiables (public repo)

- **No internal infra** in GitHub Issues/PRs or tracked docs:
  - no hostnames, namespaces, pod names, PVCs, internal URLs, or runbooks
  - no credentials, tokens, kubeconfigs
- **Container-first**: supported execution path is via `docker/` + TOML configs.
- **B200-only** (`sm_100a`): fail fast off-target; no silent downshift/fallbacks.
- **No NCCL all-to-all on the MoE path** (RDEP only).

## Roles

- **Coordinator / Merge captain** (human or AI): creates issues, assigns work, ensures gates and sequencing, merges to `master`.
- **Agent implementer** (human or AI): works in an isolated worktree, stays within scope, makes gates pass, opens PR.
- **Maintainer**: approves and runs Tier‑B (B200) gates (via GitHub Environment protection), and approves merges.

## Canonical artifacts

- **Issue contract**: a structured YAML block in the GitHub issue body (template: `work_item.yml`).
- **PR attestation**: a structured YAML block at the top of the PR description (template: `pull_request_template.md`).
- **Gate registry**: `configs/gates/gates.toml` (gate IDs + bundles + required env/files).
- **Tier‑B runner workflow**: `.github/workflows/tier-b.yml` (maintainer-controlled).

## Gate model

### Tier A (CPU/container) — agent-run

Gate IDs (stable):
- `cpu:import`
- `cpu:compileall`

Tier A should be run by the agent before opening a PR (or at least before asking for review).

### Tier B (B200) — maintainer-run, integration-correct

Tier B is triggered manually and runs against `refs/pull/<PR>/merge` (the merge result), not the PR head.

Gate bundles (stable):
- `hot_path_min`
- `blockscaled_full`
- `zero2_full`
- `frozen_full`
- `full`
- `single` (debug: one gate id)

Tier B posts a **sanitized** results table as a PR comment and also writes a job summary.

## Creating a work item (one issue per PR)

Create an issue using the **Work item (agent contract)** template:
- `.github/ISSUE_TEMPLATE/work_item.yml`

### What a good contract includes

- `scope.allow` / `scope.deny` (strict; no drive-bys)
- `contract.invariants` (B200-only, no A2A, TOML-only, determinism when applicable)
- `gates.required` (gate IDs only; do not paste cluster commands)
- `gates.perf` (if hot path; budget is explicit)
- `depends_on` + `merge_hotspots` (helps parallel planning)

The contract should be complete enough that an AI agent can execute without additional context.

## Starting work (branch + worktree)

Create an isolated worktree per issue. Naming convention:
- Worktree: `../nmoe-worktrees/issue-<NN>-<slug>`
- Branch: `issue/<NN>-<slug>`

Example:
```bash
git fetch origin
ISSUE=03a
SLUG=attention
git worktree add ../nmoe-worktrees/issue-${ISSUE}-${SLUG} -b issue/${ISSUE}-${SLUG} origin/master
cd ../nmoe-worktrees/issue-${ISSUE}-${SLUG}
```

Parallelism: run many issues in parallel via separate worktrees; coordinate via `depends_on` and shared-file hotspots.

## Implementing (agent loop)

The agent implementer:
1) Reads the issue contract.
2) Stays inside `scope.allow` (and avoids `scope.deny`).
3) Makes Tier‑A gates pass.
4) Commits clean, issue-scoped changes.

Tier‑A commands:
```bash
python -c "import nmoe"          # cpu:import
python -m compileall -q nmoe/    # cpu:compileall
```

## Opening a PR (attestation)

PRs must include the **PR Attestation YAML** at the top:
- `.github/pull_request_template.md`

The agent fills:
- `issue`, `risk`
- Tier‑A gates run
- (if perf is claimed / hot path) `baseline_sha`

Tier‑B is filled after the maintainer runs the Tier‑B workflow.

Create PR (recommended: GitHub CLI):
```bash
git push origin HEAD
gh pr create --base master
```

## Running Tier‑B (maintainer-controlled)

Tier‑B is triggered manually (GitHub Actions → “Tier B gates (B200, maintainer-controlled)”).

Inputs:
- `pr_number`: PR number
- `bundle`: one of the bundle names (or `single`)
- `single_gate`: gate id when `bundle=single`
- `baseline_ref` + `perf_budget_pct`: optional perf parameters

After approval, ARC scales the B200 runner up, executes the gates, comments results on the PR, and scales back to zero.

## Merge-down (to master)

We merge **one PR per issue** after:
1) Review (scope + invariants + public safety)
2) Tier‑A attestation present
3) Tier‑B comment is green for the required bundle

### Sequencing

Respect `depends_on` from contracts. When in doubt, merge shared-hotspot PRs sequentially (e.g., changes touching `nmoe/train.py`, `nmoe/config.py`, `nmoe/moe.py`, `nmoe/blockscaled/grouped.py`).

### Conflict policy

- Mechanical conflict resolution first (no refactors while resolving conflicts).
- If scope needs to expand, update the issue contract first.

## Runner image policy (important)

The **runner image** is rebuilt automatically on `master` **only when the Docker environment changes**:
- `docker/Dockerfile.runner`
- `docker/Dockerfile.train`
- `docker/Dockerfile.base`

It intentionally does **not** rebuild on every code change. Tier‑B always:
1) checks out `refs/pull/<PR>/merge`
2) builds the CUDA extension from that checkout (`make -C nmoe/csrc`)
3) runs the selected gates

This keeps “runner image == environment” while “PR checkout == code under test”.

## Troubleshooting (public-safe)

- Tier‑B says “missing required env vars”: the runner pod is missing a required env listed in `configs/gates/gates.toml` (`requires`). Fix runner config; do not paste cluster details into issues.
- Tier‑B says “missing required files”: gate references a config file that doesn’t exist on the merge ref.
- Perf gate fails: confirm you ran the intended compare gate (typically `b200:moonlight_8x20`) and that the perf budget in the issue contract matches expectations.

