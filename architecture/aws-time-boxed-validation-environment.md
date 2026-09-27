# AWS time-boxed validation environment

## Decision summary

Formula Insights will use AWS only for short, planned validation sessions while
the monthly cloud-cost ceiling remains USD 20. The normal development
environment stays local. This AWS design validates Terraform, AWS identity,
Kubernetes delivery, and teardown practice; it is not a persistent development
or production environment.

A continuously running EKS cluster is explicitly out of scope for this budget.
Amazon EKS control-plane pricing alone is USD 0.10 per cluster-hour, or roughly
USD 73 for a 730-hour month before nodes, storage, networking, or databases.
See the [AWS EKS pricing page](https://aws.amazon.com/eks/pricing/).

## Goals

- Prove that the platform can be created and destroyed reproducibly with
  Terraform.
- Run one short-lived Kubernetes deployment experiment in AWS.
- Capture operational evidence: a cost check, deployment verification, and
  teardown verification.
- Keep expected spend below the approved USD 20 monthly ceiling.
- Avoid long-lived credentials and manual console-created infrastructure.

## Non-goals

- A continuously available environment.
- Production availability, durability, backup, or disaster recovery.
- A public internet-facing API, load balancer, NAT gateway, RDS, or
  multi-region topology.
- A replacement for local Kind/k3d development.
- A commitment to EKS as the permanent production platform.

## Architecture

```text
Developer workstation / approved CI identity
                  |
                  v
          Terraform plan and apply
                  |
                  v
     One AWS region, one short-lived VPC
                  |
                  v
       Standard-support EKS control plane
                  |
                  v
   One on-demand managed worker node (min = max = 1)
                  |
        +---------+----------+
        |                    |
        v                    v
  Formula Insights API   Ephemeral PostgreSQL
        |
        v
  kubectl port-forward or temporary test access

Terraform destroy completes the experiment.
```

### AWS components

| Component | Design choice | Cost and security rationale |
| --- | --- | --- |
| Region | One configurable region; default to the closest supported region | Prevents accidental multi-region spend. |
| VPC | Two public subnets in separate availability zones | EKS-compatible demonstrator topology without NAT gateway cost. |
| EKS | Standard-support control plane, created only for a scheduled session | Preserves a realistic Kubernetes control plane while avoiding 24/7 cost. |
| Compute | One on-demand managed node; minimum, desired, and maximum count all set to 1 | No autoscaling surprises or idle node fleet. Instance family and architecture must be selected only after image compatibility is verified. |
| Database | PostgreSQL runs in-cluster with disposable data | Avoids RDS cost and deliberately does not claim durability. |
| API access | No public Ingress or load balancer; use port-forward or a short-lived controlled test path | Avoids load-balancer cost and unnecessary public exposure. |
| State | Terraform remote state and locking, designed separately in `platform-infrastructure` | State infrastructure must be tracked and costed before use. |

## Time-box and lifecycle

An experiment is an intentional, finite event—not an environment left running.

1. **Plan:** open or link the tracking issue and record the goal, expected
   duration, region, Terraform environment, and estimated cost.
2. **Preflight:** confirm AWS Budget alerts, required tags, no open cost alert,
   and a reviewed Terraform plan.
3. **Create:** apply Terraform and record the start time.
4. **Validate:** deploy the smallest workload, verify it, and collect the
   required evidence.
5. **Destroy:** run the reviewed Terraform destroy procedure in the same work
   session unless a written exception gives a specific expiry time.
6. **Verify:** confirm the cluster, nodes, volumes, public IPs, and other
   billable resources are gone; record the teardown time and estimated cost.

Every resource must include these tags where supported:

```text
project = formula-insights
environment = aws-lab
managed-by = terraform
owner = fabricio
expires-at = ISO-8601 timestamp
```

The `expires-at` tag is an audit and recovery signal. It is not an automatic
shutdown mechanism. The operator remains responsible for destruction.

## Cost-control rules

- The monthly ceiling is USD 20; it is a hard stop-and-review boundary.
- The budget owner receives alerts at USD 10, USD 16, and USD 20.
- A session must have a planned duration and an immediate destroy step.
- Do not start a new session after the 80% threshold without explicit approval.
- Do not rely on AWS Budgets as a real-time kill switch; billing data and
  notifications can lag. Session duration and verified teardown are the primary
  controls.
- Use the AWS Pricing Calculator before the first apply and whenever the
  topology changes.
- Do not introduce NAT gateways, Application Load Balancers, RDS, or persistent
  worker nodes under this budget without a new cost review.

The detailed response to thresholds and teardown procedure lives in
[`aws-cost-control.md`](../runbooks/aws-cost-control.md).

## Identity and access

- Terraform and future CI must use short-lived AWS credentials. GitHub Actions
  will use AWS OIDC when cloud automation is introduced; no long-lived AWS keys
  may be stored in a repository.
- Human access uses a least-privilege AWS identity and MFA.
- The EKS API endpoint is not exposed broadly. Administrative access is limited
  to the approved operator or automation identity for the session.
- Runtime secrets are not stored in Git. The specific secret-management
  mechanism remains a separate decision before production-like workloads.

## Required evidence for the first session

- Terraform plan and apply output, with secrets redacted.
- AWS Budget alert configuration and test-notification evidence.
- A running workload verification: API health response and importer completion.
- Resource inventory before destruction.
- Terraform destroy output and post-destroy verification.
- A short retrospective with the session duration, actual cost when available,
  and any resource that survived teardown.

## Exit criteria

This design is successful when one complete session has created, verified, and
destroyed the environment within the budget boundary, leaving documented
evidence. It does not authorize a persistent AWS environment. Any move to
always-on EKS, managed databases, ingress/load balancing, or production
availability requires a new cost estimate and explicit budget decision.
