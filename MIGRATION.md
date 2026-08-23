# Migrating from v1 to v2

v2 renames the Action to **DSG Verified Execution Gate** and replaces the
readiness-probe contract with a proof-receipt contract backed by Cinema
`/verify/evaluate` and exact Z3 verification.

## v1 keeps working

v1 is published under immutable tags. Nothing in this release changes them:

- `v1.1.0`, `v1.0.2`, `v1.0.1`, `v1.0.0`

Pin to `@v1.1.0` (or `@v1`) to stay on the readiness-gate behaviour. Only
workflows that reference `@main` pick up v2 automatically.

## What changed

| | v1 — Secure Deploy Gate | v2 — Verified Execution Gate |
|---|---|---|
| Decision | `GO` / `NO-GO` | `ALLOW` / `REVIEW` / `BLOCK` |
| Source of truth | HTTP readiness probe run in the runner | Cinema `/verify/evaluate` with exact Z3 proof |
| Evidence file | `dsg-evidence.json` | `dsg-proof-receipt.json` |
| Failure mode | fails on `NO-GO` | fails on `BLOCK`, and on `REVIEW` when `fail_on_review: true` |
| Metering | none | optional `api_key` attributes proofs to your DSG account |

v1 decided the verdict inside the runner. v2 sends a bounded set of hashes and
booleans to Cinema, which returns a decision only when Z3 proves
`VERIFIED_GLOBAL_OPTIMUM`. An unverified proof is reported as `REVIEW`, never
as approval.

## Input mapping

There is no mechanical rewrite from v1 inputs to v2 inputs — the two Actions
answer different questions. v1 asked "is the deployed endpoint healthy". v2
asks "was this execution authorized, plan-aligned, constraint-satisfying,
replayable, and evidenced".

v2 requires the caller to state each result explicitly. There is no permissive
default for `authorized`, `plan_aligned`, `constraints_pass`,
`execution_succeeded`, `replay_match`, or `evidence_complete`.

```yaml
- uses: tdealer01-crypto/dsg-secure-deploy-gate-action@v2.0.0
  with:
    endpoint: https://dsg-cinema-production.nicetree-a005fe99.westus3.azurecontainerapps.io/verify/evaluate
    approved_plan_hash: ${{ steps.plan.outputs.sha256 }}
    proposed_action_hash: ${{ steps.action.outputs.sha256 }}
    authorized: 'true'
    plan_aligned: 'true'
    constraints_pass: ${{ steps.checks.outputs.passed }}
    execution_succeeded: ${{ steps.deploy.outcome == 'success' }}
    replay_match: 'true'
    evidence_complete: 'true'
```

## Running both

To keep the v1 readiness probe alongside v2, pin the v1 tag in a separate step:

```yaml
- uses: tdealer01-crypto/dsg-secure-deploy-gate-action@v1.1.0
  with:
    readiness_url: https://example.com/api/readiness
- uses: tdealer01-crypto/dsg-secure-deploy-gate-action@v2.0.0
  with:
    endpoint: https://dsg-cinema-production.nicetree-a005fe99.westus3.azurecontainerapps.io/verify/evaluate
    # ...
```

## Getting an API key

`api_key` is optional. Without it the Action still verifies, on the free
evaluation plan (25 proofs). To meter and attribute proofs to your account:

```bash
curl -X POST https://dsg-cinema-production.nicetree-a005fe99.westus3.azurecontainerapps.io/billing/activate \
  -H 'Content-Type: application/json' \
  -d '{"channel":"github_action","activation_id":"<your-org>"}'
```

Store the returned key as a repository secret and pass it as `api_key`.
