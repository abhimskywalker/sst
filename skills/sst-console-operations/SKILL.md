---
name: sst-console-operations
description: Operate and configure the SST Console for visibility, logs, issues, and deployments. Use when setting up Console access, connecting AWS accounts, or explaining permissions and workspace setup.
---
# SST Console operations

## Scope
- Set up the SST Console account and workspace.
- Connect AWS accounts with the CloudFormation stack.
- Explain Console capabilities and IAM permission options.

## Workflow
1. Create or confirm a Console account and workspace for the team.
2. Connect the AWS account by deploying the Console CloudFormation stack in us-east-1.
3. Validate that the Console shows apps, stages, resources, updates, logs, and issues.
4. Review IAM permissions for the Console role and adjust policies when least-privilege access is required.
5. Document any AWS quota changes needed (for example, Lambda concurrency limits in new accounts).

## Outputs
- Console setup checklist with workspace and AWS connection steps.
- IAM policy guidance or diffs for restricted access.
- Notes on Console capabilities used for monitoring and deploy workflows.
