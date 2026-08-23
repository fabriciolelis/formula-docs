# Formula Insights Platform Documentation

This repository contains the durable engineering evidence for Formula Insights: architecture, decisions, operating expectations, and lessons learned.

## Purpose

Formula Insights is a cloud-native platform project that imports historical Formula 1 data and exposes it through a small API. The project demonstrates how a Platform Engineer designs reusable delivery, infrastructure, observability, security, and reliability capabilities around a workload.

The product is intentionally limited to a Formula 1 workload in its first release. A football workload may be added later to validate that the platform patterns are genuinely reusable.

## Documentation principles

- Prefer short documents that record a clear decision or operational fact.
- Include trade-offs; no architecture has only benefits.
- Update documentation in the same merge request as the technical change.
- Do not publish credentials, internal endpoints, personal data, or sensitive cost details.

## Repository structure

```text
adr/                   # Architecture Decision Records
architecture/          # system context and technical design
diagrams/              # source-controlled diagrams
runbooks/              # operational and recovery procedures
slos/                  # SLI, SLO, and alert definitions
postmortems/           # blameless incident reviews
reviews/               # sprint and architecture review notes
```

## Release 1 scope

- Scheduled import of historical Formula 1 data.
- PostgreSQL persistence and a small read API.
- Terraform-managed AWS foundations.
- Kubernetes deployment through GitLab CI, GitOps, and Argo CD.
- Metrics, logs, dashboards, alerts, and service-level objectives.

## Initial SLOs

| Objective | Target |
| --- | --- |
| API availability | 99.5% per 30 days |
| API latency | 95% of read requests below 500 ms |
| Data freshness | scheduled import completes within 30 minutes |

## Key decisions

- ADR-001: start with one Formula 1 workload and a reusable platform boundary.
- ADR-002: use Jolpica for historical Formula 1 data.
- ADR-003: use GitOps for Kubernetes delivery.
- ADR-004: develop locally first and gate AWS provisioning on cost controls.

## Connected repositories

| Repository | Role |
| --- | --- |
| `formula-insights-api` | API and ingestion application |
| `platform-infrastructure` | Terraform-managed cloud foundations |
| `platform-gitops` | Kubernetes desired state and Argo CD configuration |
| `platform-docs` | architecture and operational evidence |

## Definition of a portfolio-quality release

A release is complete only when it includes working implementation, automated validation, updated architecture/operational documentation, and a short retrospective that explains what was learned or improved.
