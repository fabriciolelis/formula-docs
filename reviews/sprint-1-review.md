# Sprint 1 review

- Date: 2026-10-18
- Period: 2026-08-24 to 2026-10-18
- Status: complete

## Outcome

Sprint 1 delivered the local vertical slice defined in the Sprint 0 review:
PostgreSQL, an idempotent Jolpica import, a minimal read API, and automated
verification. It also added the quality gates and supply-chain controls the
API README requires, and recorded the AWS cost guardrails. No Kubernetes,
Argo CD, or AWS resources were created, as intended.

## Evidence reviewed

| Entry criterion | Evidence | Result |
| --- | --- | --- |
| PostgreSQL | `compose.yaml`; formula-insights-api#13 | Local Compose database; CI imports the 2024 season into PostgreSQL 16 end to end (24 races, 479 results). |
| Idempotent Jolpica import | formula-insights-api#2, #5 | Re-imports create no duplicates; results are paginated, totals are validated, and incomplete pages are rejected. |
| Minimal API | formula-insights-api#2, #15 | Five read endpoints; standings mirror the official tables, including sprint points (2024: Verstappen 437, McLaren 666). |
| Automated verification | formula-insights-api#7, #8, #9, #10, #13 | Every pull request runs tests (including PostgreSQL), ruff, the image build, a Trivy scan (HIGH and CRITICAL), and Semgrep (ERROR); branch protection requires them on `main`. |

## Additional work

- **Supply chain:** dependencies locked in `uv.lock`; the uv image and the
  Trivy action are pinned by digest and commit; eight unfixed Debian
  base-image CVEs are accepted in `.trivyignore` until 2027-01-15.
- **Cost control:** ADR-004 amended with a USD 20 monthly ceiling, an AWS
  cost-control runbook, and a time-boxed three-node K3s validation design
  (platform-docs#11). Resources are tagged `environment = dev` with
  `expires-at`.
- **Repository hygiene:** `formula-docs` renamed to `platform-docs`, English
  folder indexes, branch protection on all four repositories, and the
  Apache-2.0 license.

## Corrections made

- Standings summed Grand Prix points only, which was wrong for seasons with
  sprints. They now mirror Jolpica's official standings, including penalties
  and exclusions (formula-insights-api#4).
- The importer requested one oversized page of results and could import a
  season partially. It now paginates and validates totals
  (formula-insights-api#3, delivered in #5).
- The stacked pull request #14 was merged into its base branch instead of
  `main`; formula-insights-api#15 brought the work to `main`.

## Carried over

- platform-docs#8: ingestion policy and data SLOs.
- platform-docs#9: database migration strategy (the runtime decision is
  recorded in ADR-005).
- platform-docs#10: rollback runbook and ingestion signals.
- platform-gitops#2: GitOps bootstrap, secret management, and environments.
- Observability: Prometheus metrics, structured logs, and a readiness
  endpoint, which the API README promises but are not implemented.
- Non-blocking cleanups from the #14 review.

## Sprint 2 entry criteria (proposed)

Sprint 2 makes the API deployable: observability, a published container image
with immutable tags, the GitOps decisions in platform-gitops#2, and a local
cluster reconciled by Argo CD. The migration strategy (platform-docs#9) and the
rollback runbook (platform-docs#10) must exist before that first deployment.
AWS remains out of scope.

## Lessons

1. **Check derived data against a known reference.** Summed standings looked
   plausible; only a comparison with the official 2024 tables showed they were
   wrong. Prefer the authoritative upstream resource over recomputing it.
2. **Security gates need an exception policy from the start.** The first image
   scan failed on 44 findings that had no fix. An expiring allowlist kept the
   gate meaningful instead of weakening it.
3. **Stacked pull requests need their base branch retargeted.** GitHub only
   retargets automatically when the base branch is deleted on merge.
