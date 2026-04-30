# Capstone Project: End-to-End CI/CD on AWS

**Author:** Omao
**Repository:** https://github.com/omaomach/capstone-2
**Region:** `eu-west-1`

## Project Overview

A fully automated CI/CD pipeline on AWS that takes code from GitHub, builds and tests it, packages it into a Docker container, and deploys it to an Amazon ECS cluster behind a load balancer. The project integrates CodePipeline, CodeBuild, CodeDeploy, and ECR, along with monitoring and a manual approval stage.

**Live URL:** http://myapp-alb-913856284.eu-west-1.elb.amazonaws.com

## Architecture

- **Source:** GitHub repository containing a Node.js Express application
- **Build:** AWS CodeBuild compiles the app, runs unit tests, and builds a Docker image
- **Artifact Storage:** Docker image is pushed to Amazon ECR
- **Deploy:** AWS CodeDeploy deploys the new container to Amazon ECS (Fargate) using Blue/Green
- **Orchestration:** AWS CodePipeline ties the stages together
- **Monitoring:** CloudWatch alarm monitors ECS service health
- **Approval:** A manual approval stage runs before deployment

## Deliverables

- `src/app.js` — Node.js Express web app
- `tests/app.test.js` — Jest unit tests
- `Dockerfile` — Container definition
- `buildspec.yml` — CodeBuild instructions
- `appspec.yml` — CodeDeploy instructions
- `taskdef.json` — ECS Task Definition template
- `pipeline-diagram.png` — Final pipeline flow
- `docs/screenshots/` — Implementation evidence

The deployed system features:

- **AWS CodePipeline** with stages: Source (GitHub) → Build (CodeBuild) → Approval (manual) → Deploy (CodeDeploy → ECS)
- A **live ECS service** behind an Application Load Balancer
- Blue/Green deployments with auto-rollback on alarm
- CloudWatch + SNS email alerts on ECS service health

---

## Step-by-Step Guide

### Step 1: Prepare the Application Code

A simple Node.js Express app responds with `Hello from AWS CI/CD Capstone Project!` on port 3000. The app is containerized with a `Dockerfile` and includes Jest unit tests.

Verified locally: tests pass, Docker container runs, and the app responds at `localhost:3000`.

![Tests passing](docs/screenshots/01-tests-passing.png)
![Docker container running](docs/screenshots/02-docker-running.png)
![App response locally](docs/screenshots/03-app-response.png)

### Step 2: Set Up ECR

Created a private ECR repository named `myapp-repo` in `eu-west-1` and pushed the locally built image to validate the registry and capture the URI for downstream configuration.

![ECR repository created](docs/screenshots/04-ecr-repo-created.png)
![Image pushed to ECR](docs/screenshots/05-ecr-image-pushed.png)

### Step 3: Create ECS Cluster and Service

**IAM Role:** `ecsTaskExecutionRole` with `AmazonECSTaskExecutionRolePolicy` and an inline `AllowCreateLogGroup` policy to allow the task definition to auto-create its CloudWatch log group on first start.

![ECS Task Execution Role](docs/screenshots/06b-ecs-task-execution-role.png)

**Task Definition (`myapp-task`):** Registered with the `<IMAGE>` placeholder for CodeDeploy substitution and a `logConfiguration` block routing container logs to CloudWatch.

![ECS Task Execution Role](docs/screenshots/06a-ecs-task-def.png)
![Task definition registered](docs/screenshots/07-taskdef-registered.png)

**Security Groups:**

- `myapp-alb-sg` — allows ports 80 and 8080 from the internet
- `myapp-ecs-sg` — allows port 3000 only from `myapp-alb-sg`

![Security groups configured](docs/screenshots/08a-alb-security-group.png)
![Security groups configured](docs/screenshots/08b-ecs-security-group.png)

**Target Groups (Blue/Green):** `myapp-tg-blue` and `myapp-tg-green`, both target type `IP` (required for Fargate `awsvpc` networking) on HTTP/3000.

![Target groups](docs/screenshots/09-target-groups.png)

**Application Load Balancer (`myapp-alb`):** Internet-facing across all three AZs in eu-west-1, with two listeners required for Blue/Green:

- Port 80 (production) → `myapp-tg-blue`
- Port 8080 (test) → `myapp-tg-green`

![ALB created](docs/screenshots/10-alb-created.png)

**ECS Cluster (`myapp-cluster`):** Fargate-only, with Container Insights enabled.

![ECS cluster](docs/screenshots/11-ecs-cluster.png)

**ECS Service (`myapp-service`):** Created with `deploymentController.type: CODE_DEPLOY` to support the Blue/Green deployment type required by the lab. The service launched a Fargate task that registered into the blue target group, with the ALB serving traffic on port 80.

![ECS service running](docs/screenshots/12-blue-healkthy-target.png)
![App accessible via ALB](docs/screenshots/13-app-via-alb.png)

### Step 4: Configure CodeBuild

Created the build project `myapp-build` connected to the GitHub repository via AWS CodeStar Connections, with privileged mode enabled to allow Docker-in-Docker builds.

The `buildspec.yml` defines the build steps: log in to ECR, install dependencies, run unit tests, build the Docker image, push it to ECR (tagged with the commit hash and `:latest`), and generate `imageDetail.json` for CodeDeploy.

The auto-generated CodeBuild service role was augmented with an inline `AllowECRPush` policy granting ECR write permissions scoped to `myapp-repo`.

![CodeBuild project](docs/screenshots/14-codebuild-project.png)
![Successful build](docs/screenshots/15-codebuild-success.png)
![New image in ECR](docs/screenshots/16-ecr-new-image.png)

### Step 5: Configure CodeDeploy

**Service Role (`AWSCodeDeployRoleForECS`):** Created with the AWS-managed `AWSCodeDeployRoleForECS` policy.

![CodeDeploy IAM role](docs/screenshots/17-codedeploy-role.png)

**Application (`myapp-codedeploy`):** Compute platform ECS.

![CodeDeploy application](docs/screenshots/18-codedeploy-app.png)

**Deployment Group (`myapp-deployment-group`):** Configured with:

- Deployment type: Blue/Green with traffic control
- Deployment configuration: `CodeDeployDefault.ECSAllAtOnce`
- 5-minute bake time before terminating old tasks
- Auto-rollback on deployment failure or alarm
- Wired to `myapp-tg-blue` / `myapp-tg-green` and the production (80) and test (8080) listeners

The `appspec.yml` and `taskdef.json` files (deliverables in this repo) define the ECS deployment for CodeDeploy.

![Deployment group configured](docs/screenshots/19-codedeploy-deployment-group.png)

### Step 6: Create CodePipeline

Created `myapp-pipeline` chaining four stages:

| Stage                                   | Provider                                                                      |
| --------------------------------------- | ----------------------------------------------------------------------------- |
| Source (GitHub)                         | GitHub via GitHub App, push trigger on `main`                                 |
| Build (CodeBuild → build + Docker push) | AWS CodeBuild project `myapp-build`                                           |
| Approval (manual before production)     | Manual approval action                                                        |
| Deploy (CodeDeploy → ECS)               | Amazon ECS (Blue/Green) using `myapp-codedeploy` and `myapp-deployment-group` |

The pipeline triggers on every push to `main`. After Build completes, it pauses at the Approval stage for manual review, then proceeds to Deploy. The `<IMAGE>` placeholder in `taskdef.json` is substituted at deploy time using `imageDetail.json` from the build artifact.

![Pipeline pending approval](docs/screenshots/23-pipeline-pending-approval.png)
![Approval review modal](docs/screenshots/24-pipeline-approval-modal.png)
![Pipeline fully complete](docs/screenshots/25-pipeline-fully-complete.png)
![Pipeline succeeded](docs/screenshots/21-pipeline-success.png)

### Step 7: Add Monitoring

**SNS Topic (`myapp-alerts`):** Created with an email subscription to `omao@comraid.io`, confirmed via the AWS confirmation email.

![SNS Topic](docs/screenshots/sns-topic.png)
![SNS Subscription](docs/screenshots/sns-subscription.png)

**CloudWatch Alarm (`myapp-no-running-tasks`):** Monitors `RunningTaskCount` (from Container Insights) on the ECS service, firing when fewer than 1 task is running for 2 consecutive minutes. Both `ALARM` and `OK` transitions notify the SNS topic.

![CloudWatch ALARM email](docs/screenshots/cloud-watch-alarm.png)

**Auto-Rollback Wiring:** The alarm is associated with the CodeDeploy deployment group, so a failed deploy that leaves no running tasks triggers automatic rollback to the previous version.

The alarm was verified end-to-end by temporarily scaling the service to 0 tasks. The alarm transitioned `OK → ALARM`, an email notification was delivered, and the state returned to `OK` once the service was restored.

![CloudWatch ALARM email](docs/screenshots/26-cloudwatch-alarm-email.png)

---

## Pipeline Flow Diagram

![Pipeline diagram](docs/screenshots/pipeline-flow.png)
![Pipeline diagram](docs/screenshots/blue-green-switch.png)

---

## Repository Structure

```
capstone-2/
├── README.md
├── Dockerfile
├── package.json
├── package-lock.json
├── buildspec.yml
├── appspec.yml
├── taskdef.json
├── .dockerignore
├── .gitignore
├── src/
│   └── app.js
├── tests/
│   └── app.test.js
└── docs/
    └── screenshots/
```
