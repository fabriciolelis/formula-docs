# AWS time-boxed three-node K3s validation environment

## Decision summary

Formula Insights will use AWS only for short, planned validation sessions while
the monthly cloud-cost ceiling remains USD 20. The normal development
environment stays local. The AWS lab runs a self-managed, highly available
three-node K3s control plane on EC2. It validates Terraform, EC2 lifecycle,
Kubernetes operations, and teardown practice; it is not a persistent
development or production environment.

EKS is intentionally excluded from this lab. Its control-plane fee alone would
consume roughly USD 73 in a 730-hour month before nodes, storage, networking,
or databases. Three EC2 nodes also exceed the budget if left running, so every
session must be time-boxed and destroyed.

## Goals

- Prove that the platform can be created and destroyed reproducibly with
  Terraform.
- Operate a three-node Kubernetes control plane and observe basic node-failure
  and scheduling behaviour.
- Run one short-lived Formula Insights deployment experiment in AWS.
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
- A commitment to self-managed Kubernetes as the permanent production platform.

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
        +---------+---------+
        |         |         |
        v         v         v
  EC2 node 1  EC2 node 2  EC2 node 3
  K3s server  K3s server  K3s server
        \         |         /
         \--------+--------/
          Embedded etcd quorum
                  |
        +---------+----------+
        |                    |
        v                    v
  Formula Insights API   Ephemeral PostgreSQL
        |
        v
  SSM-assisted access / controlled port-forward

Terraform destroy completes the experiment.
```

## K3s topology

All three EC2 instances run K3s server nodes with embedded etcd. This provides
an odd-numbered etcd quorum: one server loss can be tolerated while two healthy
servers remain. Workloads may run on the server nodes for this lab; dedicated
worker nodes are intentionally excluded to control cost.

K3s is bootstrapped on the first node. It generates the cluster join token and
publishes it to AWS Systems Manager Parameter Store as a SecureString. The
second and third nodes retrieve the token through narrowly scoped instance-role
permissions and join the first node. The token must never be committed,
printed in CI logs, or stored in Terraform state.

## AWS components

| Component | Design choice | Cost and security rationale |
| --- | --- | --- |
| Region | One configurable region; default to the closest supported region | Prevents accidental multi-region spend. |
| VPC | Three public subnets across three availability zones and one internet gateway | Supports one node per availability zone without NAT gateway cost. |
| Compute | Three on-demand EC2 instances of one verified, low-cost instance type | Enables etcd quorum; instances are destroyed after each session. |
| Storage | One small encrypted root volume per node and K3s local-path storage | Data is disposable and removed with the nodes. |
| Kubernetes | K3s server on each node, embedded etcd, no dedicated workers | Demonstrates cluster operation without EKS control-plane fees or extra nodes. |
| Bootstrap secret | SSM Parameter Store SecureString, readable only by the K3s node role | Keeps the join token out of repositories and Terraform state. |
| Administrative access | AWS Systems Manager Session Manager; no inbound SSH | Avoids a permanent SSH exposure and supports auditable operator access. |
| Kubernetes API | Port 6443 restricted to the approved operator's current CIDR for the session | Prevents a broadly exposed control plane. |
| Database | PostgreSQL runs in-cluster with disposable data | Avoids RDS cost and deliberately does not claim durability. |
| API access | No public Ingress or load balancer; use controlled port-forward for validation | Avoids load-balancer cost and unnecessary public exposure. |
| State | Terraform remote state and locking, designed separately in `platform-infrastructure` | State infrastructure must be tracked and costed before use. |

## Time-box and lifecycle

An experiment is an intentional, finite event—not an environment left running.

1. **Plan:** open or link the tracking issue and record the goal, expected
   duration, region, Terraform environment, three-node instance type, and
   estimated cost.
2. **Preflight:** confirm AWS Budget alerts, required tags, no open cost alert,
   a reviewed Terraform plan, and the operator CIDR allowed to reach the
   Kubernetes API.
3. **Create:** apply Terraform and record the start time. Confirm that all
   three K3s servers are Ready and etcd has quorum.
4. **Validate:** deploy the smallest workload, verify API health and importer
   completion, and perform one controlled node-loss observation if it fits the
   session objective.
5. **Destroy:** run the reviewed Terraform destroy procedure in the same work
   session unless a written exception gives a specific expiry time.
6. **Verify:** confirm the instances, volumes, public IPs, SSM parameter, and
   other billable resources are gone; record the teardown time and estimated
   cost.

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
  instance type, storage, or topology changes.
- Do not introduce NAT gateways, Application Load Balancers, RDS, persistent
  nodes, or additional worker nodes under this budget without a new cost review.

The detailed response to thresholds and teardown procedure lives in
[`aws-cost-control.md`](../runbooks/aws-cost-control.md).

## Identity, network, and operational responsibility

- Terraform and future CI must use short-lived AWS credentials. GitHub Actions
  will use AWS OIDC when cloud automation is introduced; no long-lived AWS keys
  may be stored in a repository.
- Human access uses a least-privilege AWS identity and MFA.
- Security groups permit K3s and etcd traffic only between the three lab nodes.
  The Kubernetes API is limited to the session operator CIDR; SSH is disabled.
- Runtime secrets are not stored in Git. The specific workload
  secret-management mechanism remains a separate decision before
  production-like workloads.
- Unlike EKS, this design makes the team responsible for K3s upgrades,
  operating-system patching, certificate lifecycle, and etcd backup/recovery.
  Those responsibilities are deliberately part of the learning evidence, but
  are not claims of production readiness.

## Required evidence for the first session

- Terraform plan and apply output, with secrets redacted.
- AWS Budget alert configuration and test-notification evidence.
- All three K3s servers Ready and etcd-quorum verification.
- A running workload verification: API health response and importer completion.
- Resource inventory before destruction.
- Terraform destroy output and post-destroy verification.
- A short retrospective with the session duration, actual cost when available,
  node-failure observation, and any resource that survived teardown.

## Exit criteria

This design is successful when one complete session has created, verified, and
destroyed the three-node cluster within the budget boundary, leaving documented
evidence. It does not authorize a persistent AWS environment. Any move to
always-on nodes, managed databases, ingress/load balancing, or production
availability requires a new cost estimate and explicit budget decision.
