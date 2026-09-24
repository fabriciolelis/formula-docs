# AWS cost-control runbook

## Purpose

This runbook protects the Formula Insights project from unplanned AWS spend. It
implements the guardrails required by ADR-004 before any AWS environment is
provisioned.

## Approved cost boundary

| Control | Decision |
| --- | --- |
| Monthly ceiling | USD 20 per calendar month |
| Budget owner | Fabricio Lelis |
| Billing currency | USD |
| Scope | All Formula Insights AWS resources in the selected AWS account |
| Provisioning method | Terraform only |
| Exception | Further spend requires explicit approval recorded in the relevant pull request or issue |

The ceiling is a stop-and-review control, not a target to spend.

## Required controls before provisioning

1. Create an AWS Budget for USD 20 on actual monthly cost.
2. Configure notifications to the budget owner at 50%, 80%, and 100%.
3. Verify that each notification destination receives a test notification.
4. Ensure Terraform applies the required project, environment, managed-by, and
   owner tags to every supported resource.
5. Record the AWS account, region, budget identifier, and notification
   destinations in the implementation pull request. Do not record secrets.
6. Keep the teardown commands and resource inventory with the relevant Terraform
   environment documentation.

No Terraform apply may create billable AWS resources until this checklist is
complete and reviewed.

## Threshold response

### 50% — USD 10

- Review the AWS Cost Explorer breakdown and resource inventory.
- Confirm that the spend corresponds to an approved experiment.
- Check whether idle resources can be stopped or destroyed.
- Add the finding to the active issue or pull request.

### 80% — USD 16

- Stop starting new billable experiments.
- Identify the resources responsible for the increase.
- Schedule teardown of nonessential resources.
- Decide whether the work can continue locally for the rest of the month.
- Escalate to the budget owner before any cost-increasing change.

### 100% — USD 20

- Stop nonessential workloads immediately.
- Run the documented Terraform destroy procedure for the affected environment,
  after confirming its exact target and state.
- Preserve only resources explicitly approved as necessary for investigation.
- Record the cause, estimated final monthly cost, and teardown result.
- Resume billable work only after the budget owner gives explicit approval.

## Emergency teardown procedure

1. Confirm the AWS account, region, Terraform workspace, and environment root.
2. Run a read-only Terraform plan to identify the exact resources targeted for
   deletion.
3. Stop or disable workload sources first: scheduled jobs, deployments, test
   runners, and any externally triggered automation.
4. Run the environment's reviewed Terraform destroy command.
5. Verify in AWS Cost Explorer and the provider console that billable resources
   are gone or intentionally retained.
6. Update the resource inventory and record the result in the incident or
   tracking issue.
7. If the teardown did not complete, keep the workload disabled and open a
   follow-up issue with the remaining resources and owner.

## Monthly review

Before beginning a new month of AWS work, review the prior month's actual cost,
remaining resources, and teardown evidence. Adjust the ceiling only through a
reviewed documentation change and explicit approval.
