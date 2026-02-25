# AWS CI/CD — Boas Práticas

Guia de pipelines de CI/CD na AWS: CodePipeline, CodeBuild, CodeDeploy, GitHub Actions e estratégias de deploy.

---

## Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    CI/CD Pipeline                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SOURCE        BUILD           TEST            DEPLOY            │
│  ┌──────┐     ┌──────────┐   ┌──────────┐    ┌──────────┐      │
│  │GitHub│ ──→ │CodeBuild │──→│CodeBuild │──→ │CodeDeploy│      │
│  │      │     │(build +  │   │(unit +   │    │(ECS/     │      │
│  │      │     │ docker)  │   │ integr.) │    │ Lambda)  │      │
│  └──────┘     └──────────┘   └──────────┘    └──────────┘      │
│       │              │              │               │            │
│       │              ▼              ▼               ▼            │
│       │         ECR Push      Test Report     Blue/Green        │
│       │                                       Canary            │
│       │                                       Rolling           │
│       │                                                         │
│  ALTERNATIVA: GitHub Actions                                    │
│  ┌──────┐     ┌──────────┐   ┌──────────┐    ┌──────────┐      │
│  │GitHub│ ──→ │GH Action │──→│GH Action │──→ │GH Action │      │
│  │      │     │(build)   │   │(test)    │    │(deploy   │      │
│  │      │     │          │   │          │    │ via AWS) │      │
│  └──────┘     └──────────┘   └──────────┘    └──────────┘      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## CodePipeline + CodeBuild

### Pipeline Terraform

```hcl
# CodePipeline — orquestra o pipeline completo
resource "aws_codepipeline" "main" {
  name     = "${var.project}-pipeline"
  role_arn = aws_iam_role.codepipeline.arn

  artifact_store {
    location = aws_s3_bucket.pipeline_artifacts.bucket
    type     = "S3"

    encryption_key {
      id   = aws_kms_key.pipeline.arn
      type = "KMS"
    }
  }

  # Stage 1: Source (GitHub via CodeStar Connection)
  stage {
    name = "Source"

    action {
      name             = "Source"
      category         = "Source"
      owner            = "AWS"
      provider         = "CodeStarSourceConnection"
      version          = "1"
      output_artifacts = ["source_output"]

      configuration = {
        ConnectionArn    = aws_codestarconnections_connection.github.arn
        FullRepositoryId = var.github_repo
        BranchName       = var.branch
        DetectChanges    = true
      }
    }
  }

  # Stage 2: Build + Test
  stage {
    name = "Build"

    action {
      name             = "BuildAndTest"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      version          = "1"
      input_artifacts  = ["source_output"]
      output_artifacts = ["build_output"]

      configuration = {
        ProjectName = aws_codebuild_project.build.name
      }
    }
  }

  # Stage 3: Deploy to Staging
  stage {
    name = "DeployStaging"

    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "ECS"
      version         = "1"
      input_artifacts = ["build_output"]

      configuration = {
        ClusterName = var.staging_cluster_name
        ServiceName = var.staging_service_name
        FileName    = "imagedefinitions.json"
      }
    }
  }

  # Stage 4: Manual Approval
  stage {
    name = "Approval"

    action {
      name     = "ManualApproval"
      category = "Approval"
      owner    = "AWS"
      provider = "Manual"
      version  = "1"

      configuration = {
        NotificationArn = aws_sns_topic.pipeline_approval.arn
        CustomData      = "Approve deployment to production?"
      }
    }
  }

  # Stage 5: Deploy to Production (Blue/Green)
  stage {
    name = "DeployProd"

    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "CodeDeployToECS"
      version         = "1"
      input_artifacts = ["build_output"]

      configuration = {
        ApplicationName                = aws_codedeploy_app.main.name
        DeploymentGroupName            = aws_codedeploy_deployment_group.prod.deployment_group_name
        TaskDefinitionTemplateArtifact = "build_output"
        TaskDefinitionTemplatePath     = "taskdef.json"
        AppSpecTemplateArtifact        = "build_output"
        AppSpecTemplatePath            = "appspec.yaml"
      }
    }
  }
}
```

### CodeBuild Project

```hcl
resource "aws_codebuild_project" "build" {
  name          = "${var.project}-build"
  build_timeout = 15  # minutos
  service_role  = aws_iam_role.codebuild.arn

  source {
    type      = "CODEPIPELINE"
    buildspec = "buildspec.yml"
  }

  artifacts {
    type = "CODEPIPELINE"
  }

  environment {
    compute_type                = "BUILD_GENERAL1_MEDIUM"
    image                       = "aws/codebuild/amazonlinux2-aarch64-standard:3.0"  # ARM64 (mais barato)
    type                        = "ARM_CONTAINER"
    image_pull_credentials_type = "CODEBUILD"
    privileged_mode             = true  # Necessário para docker build

    environment_variable {
      name  = "AWS_ACCOUNT_ID"
      value = data.aws_caller_identity.current.account_id
    }

    environment_variable {
      name  = "ECR_REPO"
      value = aws_ecr_repository.main.repository_url
    }

    environment_variable {
      name  = "ENVIRONMENT"
      value = var.environment
    }
  }

  cache {
    type  = "S3"
    location = "${aws_s3_bucket.pipeline_artifacts.bucket}/cache"
  }

  logs_config {
    cloudwatch_logs {
      group_name  = "/aws/codebuild/${var.project}"
      stream_name = "build"
    }
  }

  vpc_config {
    vpc_id             = var.vpc_id
    subnets            = var.private_subnet_ids
    security_group_ids = [aws_security_group.codebuild.id]
  }

  tags = var.tags
}
```

### Buildspec (Java + Docker)

```yaml
# buildspec.yml
version: 0.2

env:
  variables:
    JAVA_VERSION: "21"
    DOCKER_BUILDKIT: "1"

phases:
  install:
    runtime-versions:
      java: corretto21

  pre_build:
    commands:
      - echo "Logging into ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPO
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:-latest}

  build:
    commands:
      # Build da aplicação
      - echo "Building application..."
      - ./gradlew build -x test --no-daemon
      
      # Testes
      - echo "Running tests..."
      - ./gradlew test --no-daemon
      
      # Security scan (SAST)
      - echo "Running security scan..."
      - ./gradlew dependencyCheckAnalyze --no-daemon || true
      
      # Docker build (multi-arch)
      - echo "Building Docker image..."
      - docker build -t $ECR_REPO:$IMAGE_TAG -t $ECR_REPO:latest .
      
      # Scan de vulnerabilidades na imagem
      - echo "Scanning image..."
      - aws ecr start-image-scan --repository-name $ECR_REPO_NAME --image-id imageTag=$IMAGE_TAG || true

  post_build:
    commands:
      - echo "Pushing to ECR..."
      - docker push $ECR_REPO:$IMAGE_TAG
      - docker push $ECR_REPO:latest
      
      # Gerar artefatos para deploy
      - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $ECR_REPO:$IMAGE_TAG > imagedefinitions.json

reports:
  junit-reports:
    files:
      - "build/test-results/test/*.xml"
    file-format: JUNITXML

  jacoco-reports:
    files:
      - "build/reports/jacoco/test/jacocoTestReport.xml"
    file-format: JACOCOXML

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yaml
    - taskdef.json

cache:
  paths:
    - "/root/.gradle/caches/**/*"
    - "/root/.gradle/wrapper/**/*"
```

---

## GitHub Actions (Alternativa)

### Workflow Completo

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write   # OIDC para assumir IAM Role
  contents: read

env:
  AWS_REGION: sa-east-1
  ECR_REPOSITORY: my-app
  ECS_CLUSTER: my-cluster
  ECS_SERVICE: my-service

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: corretto
          java-version: '21'
          cache: gradle

      - name: Build and Test
        run: ./gradlew build test

      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: build/reports/tests/

  security-scan:
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          scan-ref: .
          severity: CRITICAL,HIGH

  deploy-staging:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    needs: [build-and-test, security-scan]
    environment: staging
    steps:
      - uses: actions/checkout@v4

      # OIDC — sem access keys! Assume IAM Role
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_STAGING }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

      - name: Deploy to ECS Staging
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: task-definition.json
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true

  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production  # Requer approval manual no GitHub
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_PROD }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Deploy to ECS Production (Blue/Green)
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: task-definition.json
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          codedeploy-appspec: appspec.yaml
          codedeploy-application: my-app
          codedeploy-deployment-group: my-app-prod
```

### OIDC Provider para GitHub Actions (sem Access Keys)

```hcl
# GitHub OIDC Provider — NUNCA usar access keys em CI/CD
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  thumbprint_list = ["ffffffffffffffffffffffffffffffffffffffff"]
}

# IAM Role para GitHub Actions
resource "aws_iam_role" "github_actions" {
  name = "${var.project}-github-actions"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRoleWithWebIdentity"
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:${var.github_org}/${var.github_repo}:ref:refs/heads/main"
        }
      }
    }]
  })
}

# Policy: apenas o necessário para deploy
resource "aws_iam_role_policy" "github_actions" {
  role = aws_iam_role.github_actions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ecs:UpdateService",
          "ecs:DescribeServices",
          "ecs:DescribeTaskDefinition",
          "ecs:RegisterTaskDefinition",
          "ecs:DescribeTasks"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = ["iam:PassRole"]
        Resource = [
          aws_iam_role.ecs_task_execution.arn,
          aws_iam_role.ecs_task.arn
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "codedeploy:CreateDeployment",
          "codedeploy:GetDeployment",
          "codedeploy:GetDeploymentConfig",
          "codedeploy:RegisterApplicationRevision"
        ]
        Resource = "*"
      }
    ]
  })
}
```

---

## Deployment Strategies

### Comparação

```
┌───────────────────────────────────────────────────────────────┐
│                Deployment Strategies                           │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Rolling Update (ECS Default)                                 │
│  ├── Substitui tasks gradualmente                            │
│  ├── Rollback: novo deploy com versão anterior               │
│  ├── Downtime: zero (com min_healthy_percent)                │
│  └── Risco: versões mistas durante deploy                    │
│                                                               │
│  Blue/Green (CodeDeploy)                                      │
│  ├── Cria novo target group (green)                          │
│  ├── Testa green com test listener                           │
│  ├── Shift de tráfego: all-at-once ou linear                 │
│  ├── Rollback: instantâneo (volta para blue)                 │
│  └── Risco: custo dobrado durante deploy                     │
│                                                               │
│  Canary                                                       │
│  ├── Envia % pequeno de tráfego para nova versão             │
│  ├── Monitora métricas (error rate, latency)                 │
│  ├── Se OK: shift gradual (10% → 50% → 100%)                │
│  ├── Se NOK: rollback automático                             │
│  └── Risco: menor risco de todos                             │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### CodeDeploy — Blue/Green para ECS

```hcl
resource "aws_codedeploy_app" "main" {
  compute_platform = "ECS"
  name             = var.project
}

resource "aws_codedeploy_deployment_group" "prod" {
  app_name               = aws_codedeploy_app.main.name
  deployment_group_name  = "${var.project}-prod"
  deployment_config_name = "CodeDeployDefault.ECSCanary10Percent5Minutes"
  service_role_arn       = aws_iam_role.codedeploy.arn

  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE"]
  }

  blue_green_deployment_config {
    deployment_ready_option {
      action_on_timeout = "CONTINUE_DEPLOYMENT"
    }

    terminate_blue_instances_on_deployment_success {
      action                           = "TERMINATE"
      termination_wait_time_in_minutes = 60  # Manter blue por 60min para rollback
    }
  }

  deployment_style {
    deployment_option = "WITH_TRAFFIC_CONTROL"
    deployment_type   = "BLUE_GREEN"
  }

  ecs_service {
    cluster_name = var.cluster_name
    service_name = var.service_name
  }

  load_balancer_info {
    target_group_pair_info {
      prod_traffic_route {
        listener_arns = [var.https_listener_arn]
      }

      test_traffic_route {
        listener_arns = [var.test_listener_arn]
      }

      target_group {
        name = var.blue_target_group_name
      }

      target_group {
        name = var.green_target_group_name
      }
    }
  }

  alarm_configuration {
    alarms  = [var.error_rate_alarm_name, var.latency_alarm_name]
    enabled = true
  }
}
```

### AppSpec para ECS Blue/Green

```yaml
# appspec.yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "app"
          ContainerPort: 8080

Hooks:
  - BeforeInstall: "LambdaFunctionToValidateBeforeInstall"
  - AfterInstall: "LambdaFunctionToValidateAfterInstall"
  - AfterAllowTestTraffic: "LambdaFunctionToRunIntegrationTests"
  - BeforeAllowTraffic: "LambdaFunctionToRunSmokeTests"
  - AfterAllowTraffic: "LambdaFunctionToValidateHealthCheck"
```

---

## Pipeline Security

### Princípios

| Princípio | Implementação |
|-----------|---------------|
| No long-lived credentials | OIDC (GitHub) ou IAM Roles (CodeBuild) |
| Least privilege | IAM Role por stage, escopo mínimo |
| Secrets in Secrets Manager | Nunca em variáveis de ambiente no pipeline |
| Signed artifacts | Sigstore/Cosign para imagens Docker |
| SBOM | Gerar Software Bill of Materials em cada build |
| Branch protection | Require PR reviews, status checks |
| Audit trail | CloudTrail para todas as ações de deploy |

### Pipeline Notifications

```hcl
resource "aws_codestarnotifications_notification_rule" "pipeline" {
  name        = "${var.project}-pipeline-notifications"
  resource    = aws_codepipeline.main.arn
  detail_type = "FULL"

  event_type_ids = [
    "codepipeline-pipeline-pipeline-execution-failed",
    "codepipeline-pipeline-pipeline-execution-succeeded",
    "codepipeline-pipeline-manual-approval-needed",
  ]

  target {
    address = aws_sns_topic.pipeline_notifications.arn
    type    = "SNS"
  }
}
```

---

## CI/CD Checklist

### Pipeline

- [ ] Source: webhook trigger (não polling)
- [ ] Build: reproduzível (cache de dependências, versions fixas)
- [ ] Tests: unit + integration + security scan
- [ ] Deploy staging automático (merge em main)
- [ ] Deploy prod com approval manual ou canary automático
- [ ] Rollback automático por métricas (error rate, latency)
- [ ] Notifications (Slack, email) para falhas e approvals

### Security

- [ ] OIDC para GitHub Actions (sem access keys)
- [ ] IAM Roles com menor privilégio por stage
- [ ] Secrets no Secrets Manager (nunca hardcoded)
- [ ] Container image scanning no build
- [ ] SAST/DAST integrado no pipeline
- [ ] Branch protection (reviews, status checks)
- [ ] Signed container images

### Observability

- [ ] Build metrics (duration, success rate)
- [ ] Deploy frequency tracking (DORA metrics)
- [ ] Mean Time to Recovery (MTTR)
- [ ] Change Failure Rate
- [ ] Pipeline execution history (CloudTrail)

---

## Referências

- [AWS CodePipeline User Guide](https://docs.aws.amazon.com/codepipeline/latest/userguide/)
- [GitHub Actions — AWS Deploy](https://github.com/aws-actions)
- [ECS Blue/Green Deployment](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html)
- [DORA Metrics](https://dora.dev/guides/dora-metrics-four-keys/)
