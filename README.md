# AWS CI/CD Capstone Project

This repository contains a sample Node.js app and AWS configuration files for a full CI/CD pipeline using CodePipeline, CodeBuild, CodeDeploy, ECR, and ECS.

## Steps

1. Push code to GitHub.
2. CodePipeline fetches source → triggers CodeBuild.
3. CodeBuild runs tests → builds Docker image → pushes to ECR.
4. CodeDeploy updates ECS service with new image.
5. ECS service serves app behind Application Load Balancer.

