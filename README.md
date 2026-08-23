# Formula Insights Platform — Sprint 0

## Purpose

Formula Insights is a portfolio project designed to demonstrate senior Platform Engineering capability. It will collect public Formula 1 data, make a small API available, and run that workload using repeatable infrastructure, GitOps delivery, observability, security controls, and operational practices.

The first release is deliberately small. Its value is not a feature-rich sports app; it is the quality of the platform and the evidence behind its engineering decisions.

## Product statement

> Enable a fictional analytics team to access reliable historical Formula 1 results through a simple API, without requiring engineers to deploy or operate the service manually.

## Sprint 0 outcome

At the end of Sprint 0, the project has a clear scope, an agreed architecture direction, a documentation standard, and recorded decisions that future work must follow.

No production infrastructure or application code is created in this sprint.

## Release 1: in scope

- Import historical season, race, driver, constructor, and race-result data from Jolpica F1.
- Store normalized data in PostgreSQL.
- Expose an HTTP API for health, season races, driver standings, constructor standings, and race results.
- Run the ingestion process as a scheduled Kubernetes workload.
- Deliver the API to Kubernetes through GitLab CI and Argo CD.
- Provision AWS resources using Terraform.
- Operate the service with metrics, logs, dashboards, alerts, and three service-level objectives.

## Explicitly out of scope

- Football Insights and all other sports products.
- User accounts, authentication, payments, fantasy leagues, and a full frontend.
- Live timing and streaming data.
- Multi-region, multi-cloud, or an invented high-volume user scenario.
- A custom data warehouse, machine learning, or event streaming platform.

These are possible future extensions only after Release 1 is complete and documented.

## Planned API

```text
GET /health
GET /seasons/{year}/races
GET /seasons/{year}/standings/drivers
GET /seasons/{year}/standings/constructors
GET /seasons/{year}/races/{round}/results
```

## Architecture direction

```text
Jolpica F1 API
      |
      v
Kubernetes CronJob (ingestion)
      |
      v
PostgreSQL <--- Formula Insights API
                    |
                    v
             Kubernetes Service / Ingress

GitLab CI --> container registry --> GitOps repository --> Argo CD --> Kubernetes
                                         |
                                         v
                         Prometheus, Grafana, Loki, Alertmanager
```

The initial development environment may run locally with Kind or k3d. Cloud deployment will be designed for AWS, but cloud resources are only created after an explicit cost review in Sprint 1.

## SLOs for Release 1

| Objective | Target | Measurement |
| --- | --- | --- |
| API availability | 99.5% per 30 days | successful probe requests / all probe requests |
| API latency | 95% of read requests below 500 ms | API request-duration histogram |
| Data freshness | scheduled data import completes within 30 minutes | ingestion completion timestamp |

These are learning targets, not claims about a public production service.

## Definition of done for Sprint 0

- [x] Problem statement and Release 1 boundaries are documented.
- [x] Repository responsibilities are defined.
- [x] Documentation and ADR conventions are defined.
- [x] Architecture and data-source decisions have been recorded.
- [x] First backlog and acceptance criteria exist.
- [ ] Git repositories created.
- [ ] Initial issue board created.

## How to use this package

1. Read `BACKLOG.md` and create the listed work items in GitLab.
2. Create repositories using `REPOSITORIES.md`.
3. Copy the ADR templates and conventions into `platform-docs`.
4. Review every ADR before beginning implementation; change a decision by adding a superseding ADR rather than silently changing course.
5. Start Sprint 1 only when the remaining Sprint 0 checklist items are complete.

## Portfolio rule

Every implementation sprint must produce four kinds of evidence:

1. working code or infrastructure;
2. automated verification;
3. an architecture or operational decision record;
4. a short retrospective explaining a trade-off, result, or lesson.

That discipline is what turns the project into a credible Platform Engineering case study.
