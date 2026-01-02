# AWS CodeDeploy

AWS CodeDeploy automates application deployments to **EC2 instances**, **on-premises servers**, **AWS Lambda**, **Amazon ECS**, and **Amazon EKS** without managing Elastic Beanstalk. It serves as a managed alternative to open-source tools like Ansible, Chef, Puppet, or Terraform.[web:30][web:35]

## How It Works

1. **CodeDeploy Agent** (EC2/On-Prem only): Installed on target instances, polls CodeDeploy service for jobs. Not required for Lambda/ECS/EKS.  
2. **AppSpec.yml**: Defines deployment instructions, sent with the revision.  
3. **Source**: Pulls application bundle from S3, GitHub, Bitbucket, or CodeCommit.  
4. **Execution**: Runs hooks/scripts on instances; agent reports success/failure back to CodeDeploy.  
5. **Grouping**: Targets **deployment groups** (tagged instances, ASGs) for staged rollouts.[web:26][web:30]

**Integrates** with CodePipeline for CI/CD pipelines, supports Auto Scaling, and reuses existing tools. **Note**: Blue/Green works with EC2 + ALB (not on-premises); no resource provisioning.

## Primary Components

- **Application**: Logical container (unique name).  
- **Compute Platform**: EC2/On-Prem, Serverless (Lambda), ECS, EKS.  
- **Deployment Configuration**: Rules for rollout/success (e.g., min healthy hosts).  
- **Deployment Group**: Logical set of instances/ASGs (dev/prod tags).  
- **Deployment Type**: In-place or Blue/Green.  
- **IAM Roles**: Instance profile (S3/GitHub access); Service role (CodeDeploy permissions).  
- **Revision**: Code bundle + AppSpec.yml.  
- **Target Revision**: Specific version to deploy.[web:35]

## AppSpec File Structure

Defines files to copy and **hooks** (scripts with timeouts/runAs). Fixed order:

| Hook | Purpose |
|------|---------|
| **ApplicationStop** | Gracefully stop old app. |
| **DownloadBundle** | Fetch revision (auto). |
| **BeforeInstall** | Prep environment/prereqs. |
| **AfterInstall** | Post-copy setup (permissions/migrations). |
| **ApplicationStart** | Start new app. |
| **ValidateService** | Health checks (critical for success).[web:26][web:30] |

## Deployment Configurations (In-Place)

Control batching/success criteria for EC2/On-Prem:

| Config | Rollout | Failure Behavior | Use Case |
|--------|---------|------------------|----------|
| **OneAtATime** | Sequential (1 instance). | Stops/rolls back on first failure. | High availability prod. |
| **HalfAtATime** | 50% → 50%. | Succeeds if >50% healthy. | Balanced prod. |
| **AllAtOnce** | 100% simultaneous. | Full downtime risk. | Dev/test. |
| **Custom** | e.g., MinHealthyHosts=75%. | Threshold-based. | Tailored ASGs.[web:35][web:45] |

**Failures**: Instances marked failed; new deploys retry them first. Rollback via redeploy old revision or auto-rollbacks.

**Targets**: Tags, ASG, or mix; use `DEPLOYMENT_GROUP_NAME` env var in scripts.

## Deployment Types

### In-Place
Updates existing instances:
- AllAtOnce, HalfAtATime, OneAtATime, Linear (custom batches every N mins).[web:35]

### Blue/Green (Zero-Downtime)
Deploys to new "green" env, swaps traffic:
- **ASG Replacement**: Updates same ASG (terminate old → green instances).  
- **New ASG**: Creates green ASG → validate → swap ALB target group → delete blue.  
Supports Canary (10%→100%) or Linear traffic shifting. EC2/ALB, ECS, Lambda.[web:36][web:42]
