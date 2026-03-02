# Level 2 — Compute & Containers

> **Objetivo:** Containerizar as aplicações Go e Java do OrderFlow, publicar
> imagens no ECR, deploy no ECS Fargate com service discovery, auto-scaling
> e estratégias de deployment (rolling, blue/green).

---

## Objetivo de Aprendizado

- Criar Dockerfiles otimizados (multi-stage, non-root, health check)
- Publicar imagens no Amazon ECR via AWS CLI e SDK
- Criar ECS cluster, task definitions e services no Fargate
- Configurar service discovery com Cloud Map
- Implementar auto-scaling baseado em CPU, memória e métricas customizadas
- Entender estratégias de deploy: rolling update vs blue/green
- Monitorar containers via CloudWatch Container Insights

---

## Escopo Funcional

```
Level 2 — Compute Architecture do OrderFlow
══════════════════════════════════════════════════

                         Internet
                            │
                     ┌──────▼──────┐
                     │     ALB     │
                     │  (Level 1)  │
                     └──────┬──────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
   │  ECS Service │  │  ECS Service │  │  ECS Service │
   │  order-api   │  │  order-api   │  │  order-api   │
   │  (Go)        │  │  (Java)      │  │  (Go)        │
   │  Task 1      │  │  Task 2      │  │  Task 3      │
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                 │                 │
          └────────┬────────┘                 │
                   │                          │
            ┌──────▼──────┐                   │
            │  Cloud Map  │◄──────────────────┘
            │  (Service   │
            │  Discovery) │
            └──────┬──────┘
                   │
      ┌────────────┼────────────┐
      │            │            │
   ┌──▼──┐    ┌───▼──┐    ┌───▼──┐
   │ DDB │    │  S3  │    │ SQS  │
   └─────┘    └──────┘    └──────┘
          (Level 0)

  ┌──────────────────────────────────────────┐
  │              ECR Registry                │
  │  ┌─────────────┐  ┌─────────────┐       │
  │  │ orderflow/  │  │ orderflow/  │       │
  │  │ api-go:v1   │  │ api-java:v1 │       │
  │  └─────────────┘  └─────────────┘       │
  │                                          │
  │  Image scanning │ Lifecycle policies     │
  └──────────────────────────────────────────┘
```

---

## Tarefas

### Tarefa 2.1 — Dockerfile Otimizado (Go)

```dockerfile
# app-go/Dockerfile
# ──────────────────────────────────────────
# Stage 1: Build
# ──────────────────────────────────────────
FROM golang:1.24-alpine AS builder

RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /app

# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w -X main.version=${VERSION:-dev}" \
    -o /bin/orderflow-api ./cmd/api

# ──────────────────────────────────────────
# Stage 2: Runtime
# ──────────────────────────────────────────
FROM scratch

# Security: copy CA certs and timezone data
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Security: non-root user
COPY --from=builder /etc/passwd /etc/passwd
USER nobody

COPY --from=builder /bin/orderflow-api /orderflow-api

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD ["/orderflow-api", "healthcheck"]

ENTRYPOINT ["/orderflow-api"]
```

### Tarefa 2.2 — Dockerfile Otimizado (Java)

```dockerfile
# app-java/Dockerfile
# ──────────────────────────────────────────
# Stage 1: Build
# ──────────────────────────────────────────
FROM eclipse-temurin:25-jdk-alpine AS builder

WORKDIR /app

# Cache dependencies
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:resolve -B

# Build
COPY src ./src
RUN ./mvnw package -DskipTests -B \
    && java -Djarmode=layertools -jar target/*.jar extract --destination /extracted

# ──────────────────────────────────────────
# Stage 2: Runtime (layered)
# ──────────────────────────────────────────
FROM eclipse-temurin:25-jre-alpine

# Security hardening
RUN addgroup -S app && adduser -S app -G app

WORKDIR /app

# Spring Boot layered JAR (best cache)
COPY --from=builder /extracted/dependencies/ ./
COPY --from=builder /extracted/spring-boot-loader/ ./
COPY --from=builder /extracted/snapshot-dependencies/ ./
COPY --from=builder /extracted/application/ ./

USER app

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-XX:InitialRAMPercentage=50.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "org.springframework.boot.loader.launch.JarLauncher"]
```

### Tarefa 2.3 — ECR: Push de Imagens via CLI e SDK

```bash
#!/bin/bash
# scripts/push-ecr.sh — Build e push para ECR (LocalStack)

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
ACCOUNT="000000000000"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"
REGISTRY="$ACCOUNT.dkr.ecr.$REGION.localhost.localstack.cloud:4566"

echo "=== Creating ECR Repositories ==="
$AWS ecr create-repository --repository-name orderflow/api-go \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE || true

$AWS ecr create-repository --repository-name orderflow/api-java \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE || true

echo "=== Setting Lifecycle Policy ==="
LIFECYCLE_POLICY='{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Expire untagged after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}'

$AWS ecr put-lifecycle-policy \
  --repository-name orderflow/api-go \
  --lifecycle-policy-text "$LIFECYCLE_POLICY"

$AWS ecr put-lifecycle-policy \
  --repository-name orderflow/api-java \
  --lifecycle-policy-text "$LIFECYCLE_POLICY"

echo "=== Building Images ==="
VERSION=${1:-"v0.1.0"}

docker build -t orderflow/api-go:$VERSION ./app-go
docker build -t orderflow/api-java:$VERSION ./app-java

echo "=== Tagging & Pushing ==="
docker tag orderflow/api-go:$VERSION $REGISTRY/orderflow/api-go:$VERSION
docker tag orderflow/api-java:$VERSION $REGISTRY/orderflow/api-java:$VERSION

# Login to ECR
$AWS ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY

docker push $REGISTRY/orderflow/api-go:$VERSION
docker push $REGISTRY/orderflow/api-java:$VERSION

echo "=== Verifying ==="
$AWS ecr describe-images --repository-name orderflow/api-go
$AWS ecr describe-images --repository-name orderflow/api-java

echo "=== Push complete! Version: $VERSION ==="
```

```go
// internal/aws/ecr.go
package aws

import (
	"context"
	"fmt"
	"log/slog"

	awsv2 "github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/ecr"
)

type ECRClient struct {
	client *ecr.Client
}

func NewECRClient(cfg awsv2.Config, endpoint *string) *ECRClient {
	opts := []func(*ecr.Options){}
	if endpoint != nil {
		opts = append(opts, func(o *ecr.Options) {
			o.BaseEndpoint = endpoint
		})
	}
	return &ECRClient{client: ecr.NewFromConfig(cfg, opts...)}
}

type ImageInfo struct {
	RepositoryName string
	ImageTag       string
	ImageDigest    string
	SizeBytes      int64
	PushedAt       string
	ScanStatus     string
}

// ListImages lists images in an ECR repository.
func (c *ECRClient) ListImages(ctx context.Context, repoName string) ([]ImageInfo, error) {
	output, err := c.client.DescribeImages(ctx, &ecr.DescribeImagesInput{
		RepositoryName: awsv2.String(repoName),
	})
	if err != nil {
		return nil, fmt.Errorf("describe images: %w", err)
	}

	images := make([]ImageInfo, 0, len(output.ImageDetails))
	for _, img := range output.ImageDetails {
		tag := "untagged"
		if len(img.ImageTags) > 0 {
			tag = img.ImageTags[0]
		}

		scanStatus := "NOT_SCANNED"
		if img.ImageScanStatus != nil {
			scanStatus = string(img.ImageScanStatus.Status)
		}

		images = append(images, ImageInfo{
			RepositoryName: awsv2.ToString(img.RepositoryName),
			ImageTag:       tag,
			ImageDigest:    awsv2.ToString(img.ImageDigest),
			SizeBytes:      awsv2.ToInt64(img.ImageSizeInBytes),
			PushedAt:       img.ImagePushedAt.String(),
			ScanStatus:     scanStatus,
		})
	}

	slog.Info("ECR images listed", "repo", repoName, "count", len(images))
	return images, nil
}

// GetScanFindings returns vulnerability scan results for an image.
func (c *ECRClient) GetScanFindings(ctx context.Context, repoName, imageTag string) (map[string]int, error) {
	output, err := c.client.DescribeImageScanFindings(ctx, &ecr.DescribeImageScanFindingsInput{
		RepositoryName: awsv2.String(repoName),
		ImageId:        &ecr.types.ImageIdentifier{ImageTag: awsv2.String(imageTag)},
	})
	if err != nil {
		return nil, fmt.Errorf("describe scan findings: %w", err)
	}

	summary := make(map[string]int)
	for severity, count := range output.ImageScanFindings.FindingSeverityCounts {
		summary[severity] = int(count)
	}

	return summary, nil
}
```

### Tarefa 2.4 — ECS Cluster e Task Definition via CLI

```bash
#!/bin/bash
# scripts/create-ecs.sh — ECS Cluster + Task Definition + Service

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
ACCOUNT="000000000000"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"
PROJECT="orderflow"
ENV="dev"
VERSION=${1:-"v0.1.0"}

echo "=== Creating ECS Cluster ==="
$AWS ecs create-cluster \
  --cluster-name "${PROJECT}-${ENV}" \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    "capacityProvider=FARGATE,weight=1,base=1" \
    "capacityProvider=FARGATE_SPOT,weight=3"

echo "=== Registering Task Definition (Go) ==="
cat > /tmp/task-def-go.json << EOF
{
  "family": "${PROJECT}-api-go",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT}:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::${ACCOUNT}:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "api-go",
      "image": "${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/orderflow/api-go:${VERSION}",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "APP_ENV", "value": "${ENV}"},
        {"name": "AWS_REGION", "value": "${REGION}"},
        {"name": "AWS_ENDPOINT", "value": ""}
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:${REGION}:${ACCOUNT}:secret:${PROJECT}/db-password"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -qO- http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 10
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/${PROJECT}-api-go",
          "awslogs-region": "${REGION}",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
EOF

$AWS ecs register-task-definition --cli-input-json file:///tmp/task-def-go.json

echo "=== Registering Task Definition (Java) ==="
cat > /tmp/task-def-java.json << EOF
{
  "family": "${PROJECT}-api-java",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT}:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::${ACCOUNT}:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "api-java",
      "image": "${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/orderflow/api-java:${VERSION}",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "SPRING_PROFILES_ACTIVE", "value": "${ENV}"},
        {"name": "AWS_REGION", "value": "${REGION}"},
        {"name": "JAVA_OPTS", "value": "-XX:MaxRAMPercentage=75.0"}
      ],
      "secrets": [
        {
          "name": "SPRING_DATASOURCE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:${REGION}:${ACCOUNT}:secret:${PROJECT}/db-password"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -qO- http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/${PROJECT}-api-java",
          "awslogs-region": "${REGION}",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
EOF

$AWS ecs register-task-definition --cli-input-json file:///tmp/task-def-java.json

echo "=== Creating ECS Service (Go) ==="
PRIV_A=$(jq -r '.private_subnets[0]' scripts/vpc-outputs.json)
PRIV_B=$(jq -r '.private_subnets[1]' scripts/vpc-outputs.json)
APP_SG=$(jq -r '.security_groups.app' scripts/vpc-outputs.json)
TG_ARN=$(jq -r '.target_group_arn' scripts/alb-outputs.json)

$AWS ecs create-service \
  --cluster "${PROJECT}-${ENV}" \
  --service-name "${PROJECT}-api-go" \
  --task-definition "${PROJECT}-api-go" \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$PRIV_A,$PRIV_B],securityGroups=[$APP_SG],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=api-go,containerPort=8080" \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100" \
  --deployment-controller "type=ECS"

echo "=== ECS setup complete! ==="
```

### Tarefa 2.5 — ECS Service Management via SDK (Go)

```go
// internal/aws/ecs.go
package aws

import (
	"context"
	"fmt"
	"log/slog"

	awsv2 "github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/ecs"
	ecstypes "github.com/aws/aws-sdk-go-v2/service/ecs/types"
)

type ECSClient struct {
	client  *ecs.Client
	cluster string
}

func NewECSClient(cfg awsv2.Config, endpoint *string, cluster string) *ECSClient {
	opts := []func(*ecs.Options){}
	if endpoint != nil {
		opts = append(opts, func(o *ecs.Options) {
			o.BaseEndpoint = endpoint
		})
	}
	return &ECSClient{
		client:  ecs.NewFromConfig(cfg, opts...),
		cluster: cluster,
	}
}

// ServiceStatus represents the current state of an ECS service.
type ServiceStatus struct {
	Name           string
	Status         string
	DesiredCount   int32
	RunningCount   int32
	PendingCount   int32
	TaskDefinition string
	LaunchType     string
	Deployments    []DeploymentInfo
}

type DeploymentInfo struct {
	ID             string
	Status         string
	DesiredCount   int32
	RunningCount   int32
	TaskDefinition string
	RolloutState   string
	CreatedAt      string
}

// TaskInfo represents a running ECS task.
type TaskInfo struct {
	TaskArn        string
	TaskDefinition string
	LastStatus     string
	HealthStatus   string
	CPU            string
	Memory         string
	StartedAt      string
	StoppedReason  string
	ContainerIPs   []string
}

// DescribeService returns service status.
func (c *ECSClient) DescribeService(ctx context.Context, serviceName string) (*ServiceStatus, error) {
	output, err := c.client.DescribeServices(ctx, &ecs.DescribeServicesInput{
		Cluster:  awsv2.String(c.cluster),
		Services: []string{serviceName},
	})
	if err != nil {
		return nil, fmt.Errorf("describe service: %w", err)
	}

	if len(output.Services) == 0 {
		return nil, fmt.Errorf("service not found: %s", serviceName)
	}

	svc := output.Services[0]
	deployments := make([]DeploymentInfo, 0, len(svc.Deployments))
	for _, d := range svc.Deployments {
		deployments = append(deployments, DeploymentInfo{
			ID:             awsv2.ToString(d.Id),
			Status:         awsv2.ToString(d.Status),
			DesiredCount:   d.DesiredCount,
			RunningCount:   d.RunningCount,
			TaskDefinition: awsv2.ToString(d.TaskDefinition),
			RolloutState:   string(d.RolloutState),
			CreatedAt:      d.CreatedAt.String(),
		})
	}

	return &ServiceStatus{
		Name:           awsv2.ToString(svc.ServiceName),
		Status:         awsv2.ToString(svc.Status),
		DesiredCount:   svc.DesiredCount,
		RunningCount:   svc.RunningCount,
		PendingCount:   svc.PendingCount,
		TaskDefinition: awsv2.ToString(svc.TaskDefinition),
		LaunchType:     string(svc.LaunchType),
		Deployments:    deployments,
	}, nil
}

// ListTasks lists all tasks in a service.
func (c *ECSClient) ListTasks(ctx context.Context, serviceName string) ([]TaskInfo, error) {
	// List task ARNs
	listOutput, err := c.client.ListTasks(ctx, &ecs.ListTasksInput{
		Cluster:     awsv2.String(c.cluster),
		ServiceName: awsv2.String(serviceName),
	})
	if err != nil {
		return nil, fmt.Errorf("list tasks: %w", err)
	}

	if len(listOutput.TaskArns) == 0 {
		return nil, nil
	}

	// Describe tasks
	descOutput, err := c.client.DescribeTasks(ctx, &ecs.DescribeTasksInput{
		Cluster: awsv2.String(c.cluster),
		Tasks:   listOutput.TaskArns,
	})
	if err != nil {
		return nil, fmt.Errorf("describe tasks: %w", err)
	}

	tasks := make([]TaskInfo, 0, len(descOutput.Tasks))
	for _, t := range descOutput.Tasks {
		ips := make([]string, 0)
		for _, att := range t.Attachments {
			for _, detail := range att.Details {
				if awsv2.ToString(detail.Name) == "privateIPv4Address" {
					ips = append(ips, awsv2.ToString(detail.Value))
				}
			}
		}

		tasks = append(tasks, TaskInfo{
			TaskArn:        awsv2.ToString(t.TaskArn),
			TaskDefinition: awsv2.ToString(t.TaskDefinitionArn),
			LastStatus:     awsv2.ToString(t.LastStatus),
			HealthStatus:   string(t.HealthStatus),
			CPU:            awsv2.ToString(t.Cpu),
			Memory:         awsv2.ToString(t.Memory),
			StartedAt:      safeTimeString(t.StartedAt),
			StoppedReason:  awsv2.ToString(t.StoppedReason),
			ContainerIPs:   ips,
		})
	}

	return tasks, nil
}

// UpdateService updates the desired count or task definition.
func (c *ECSClient) UpdateService(ctx context.Context, serviceName string, desiredCount *int32, taskDef *string) error {
	input := &ecs.UpdateServiceInput{
		Cluster: awsv2.String(c.cluster),
		Service: awsv2.String(serviceName),
	}

	if desiredCount != nil {
		input.DesiredCount = desiredCount
	}
	if taskDef != nil {
		input.TaskDefinition = taskDef
	}

	_, err := c.client.UpdateService(ctx, input)
	if err != nil {
		return fmt.Errorf("update service: %w", err)
	}

	slog.Info("Service updated",
		"service", serviceName,
		"desired_count", desiredCount,
		"task_def", taskDef,
	)
	return nil
}

// WaitForStable waits until a service is stable (running = desired).
func (c *ECSClient) WaitForStable(ctx context.Context, serviceName string) error {
	waiter := ecs.NewServicesStableWaiter(c.client)
	err := waiter.Wait(ctx, &ecs.DescribeServicesInput{
		Cluster:  awsv2.String(c.cluster),
		Services: []string{serviceName},
	}, 5*60) // 5 minutes timeout
	if err != nil {
		return fmt.Errorf("wait for stable: %w", err)
	}

	slog.Info("Service is stable", "service", serviceName)
	return nil
}
```

### Tarefa 2.6 — ECS Management via SDK (Java)

```java
// infrastructure/aws/EcsService.java
package com.orderflow.infrastructure.aws;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.ecs.EcsClient;
import software.amazon.awssdk.services.ecs.model.*;

import java.util.List;

@Service
public class EcsService {

    private final EcsClient ecsClient;
    private final String cluster;

    public EcsService(
            EcsClient ecsClient,
            @Value("${orderflow.ecs.cluster}") String cluster) {
        this.ecsClient = ecsClient;
        this.cluster = cluster;
    }

    public record ServiceStatusDto(
            String name, String status,
            int desiredCount, int runningCount, int pendingCount,
            String taskDefinition, String launchType,
            List<DeploymentDto> deployments
    ) {}

    public record DeploymentDto(
            String id, String status,
            int desiredCount, int runningCount,
            String taskDefinition, String rolloutState
    ) {}

    public record TaskDto(
            String taskArn, String lastStatus, String healthStatus,
            String cpu, String memory, List<String> containerIps
    ) {}

    public ServiceStatusDto describeService(String serviceName) {
        var response = ecsClient.describeServices(DescribeServicesRequest.builder()
                .cluster(cluster)
                .services(serviceName)
                .build());

        if (response.services().isEmpty()) {
            throw new IllegalArgumentException("Service not found: " + serviceName);
        }

        var svc = response.services().getFirst();

        var deployments = svc.deployments().stream()
                .map(d -> new DeploymentDto(
                        d.id(), d.status(),
                        d.desiredCount(), d.runningCount(),
                        d.taskDefinition(), d.rolloutStateAsString()
                ))
                .toList();

        return new ServiceStatusDto(
                svc.serviceName(), svc.status(),
                svc.desiredCount(), svc.runningCount(), svc.pendingCount(),
                svc.taskDefinition(), svc.launchTypeAsString(),
                deployments
        );
    }

    public List<TaskDto> listTasks(String serviceName) {
        var taskArns = ecsClient.listTasks(ListTasksRequest.builder()
                        .cluster(cluster)
                        .serviceName(serviceName)
                        .build())
                .taskArns();

        if (taskArns.isEmpty()) return List.of();

        return ecsClient.describeTasks(DescribeTasksRequest.builder()
                        .cluster(cluster)
                        .tasks(taskArns)
                        .build())
                .tasks().stream()
                .map(t -> {
                    var ips = t.attachments().stream()
                            .flatMap(a -> a.details().stream())
                            .filter(d -> "privateIPv4Address".equals(d.name()))
                            .map(KeyValuePair::value)
                            .toList();
                    return new TaskDto(
                            t.taskArn(), t.lastStatus(),
                            t.healthStatusAsString(),
                            t.cpu(), t.memory(), ips
                    );
                })
                .toList();
    }

    public void updateDesiredCount(String serviceName, int desiredCount) {
        ecsClient.updateService(UpdateServiceRequest.builder()
                .cluster(cluster)
                .service(serviceName)
                .desiredCount(desiredCount)
                .build());
    }

    public void updateTaskDefinition(String serviceName, String taskDefinition) {
        ecsClient.updateService(UpdateServiceRequest.builder()
                .cluster(cluster)
                .service(serviceName)
                .taskDefinition(taskDefinition)
                .build());
    }
}
```

### Tarefa 2.7 — Auto-Scaling Configuration

```bash
#!/bin/bash
# scripts/configure-autoscaling.sh — ECS Service Auto Scaling

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"
PROJECT="orderflow"
ENV="dev"
CLUSTER="${PROJECT}-${ENV}"
SERVICE="${PROJECT}-api-go"

RESOURCE_ID="service/$CLUSTER/$SERVICE"

echo "=== Registering Scalable Target ==="
$AWS application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id "$RESOURCE_ID" \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

echo "=== CPU Target Tracking ==="
$AWS application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id "$RESOURCE_ID" \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name "${SERVICE}-cpu-scaling" \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'

echo "=== Memory Target Tracking ==="
$AWS application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id "$RESOURCE_ID" \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name "${SERVICE}-memory-scaling" \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 80.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageMemoryUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'

echo "=== ALB Request Count Scaling ==="
TG_ARN_SUFFIX=$(jq -r '.target_group_arn' scripts/alb-outputs.json | sed 's/.*targetgroup/targetgroup/')
ALB_ARN_SUFFIX=$(jq -r '.alb_arn' scripts/alb-outputs.json | sed 's/.*loadbalancer/loadbalancer/')

$AWS application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id "$RESOURCE_ID" \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name "${SERVICE}-request-scaling" \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration "{
    \"TargetValue\": 1000.0,
    \"PredefinedMetricSpecification\": {
      \"PredefinedMetricType\": \"ALBRequestCountPerTarget\",
      \"ResourceLabel\": \"${ALB_ARN_SUFFIX}/${TG_ARN_SUFFIX}\"
    },
    \"ScaleInCooldown\": 300,
    \"ScaleOutCooldown\": 60
  }"

echo "=== Auto-scaling configured! ==="
echo "Min: 2, Max: 10"
echo "Policies: CPU (70%), Memory (80%), Requests (1000/target)"
```

### Tarefa 2.8 — Deploy Endpoint (Go)

```go
// internal/handler/deploy.go
package handler

import (
	"net/http"

	"github.com/gin-gonic/gin"

	awsclient "orderflow/internal/aws"
)

type DeployHandler struct {
	ecs *awsclient.ECSClient
	ecr *awsclient.ECRClient
}

func NewDeployHandler(ecs *awsclient.ECSClient, ecr *awsclient.ECRClient) *DeployHandler {
	return &DeployHandler{ecs: ecs, ecr: ecr}
}

// GetServiceStatus returns current ECS service status.
// GET /api/v1/deploy/status/:service
func (h *DeployHandler) GetServiceStatus(c *gin.Context) {
	serviceName := c.Param("service")

	status, err := h.ecs.DescribeService(c.Request.Context(), serviceName)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	tasks, err := h.ecs.ListTasks(c.Request.Context(), serviceName)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"service": status,
		"tasks":   tasks,
	})
}

// GetImages returns available images from ECR.
// GET /api/v1/deploy/images/:repo
func (h *DeployHandler) GetImages(c *gin.Context) {
	repoName := c.Param("repo")

	images, err := h.ecr.ListImages(c.Request.Context(), repoName)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, gin.H{"images": images})
}

// ScaleService scales an ECS service.
// POST /api/v1/deploy/scale/:service
func (h *DeployHandler) ScaleService(c *gin.Context) {
	serviceName := c.Param("service")

	var req struct {
		DesiredCount int32 `json:"desired_count" binding:"required,min=0,max=20"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	if err := h.ecs.UpdateService(c.Request.Context(), serviceName, &req.DesiredCount, nil); err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"message":       "Service scaling initiated",
		"service":       serviceName,
		"desired_count": req.DesiredCount,
	})
}
```

---

## Container Best Practices

```
Dockerfile Checklist
══════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │  Multi-stage build                              ✅ MUST │
  │  Non-root user                                  ✅ MUST │
  │  HEALTHCHECK instruction                        ✅ MUST │
  │  Dependency caching (COPY go.mod first)          ✅ MUST │
  │  .dockerignore                                   ✅ MUST │
  │  Minimal base image (scratch, alpine, distroless)✅ MUST │
  │                                                          │
  │  Pinned base image tag                          ⚡ SHOULD│
  │  Security scan (ECR, Trivy)                     ⚡ SHOULD│
  │  OCI labels (version, commit)                   ⚡ SHOULD│
  │  Read-only root filesystem                      ⚡ SHOULD│
  └──────────────────────────────────────────────────────────┘

Fargate Resource Guidelines
══════════════════════════════════

  ┌────────────────┬───────────────┬────────────────────────┐
  │  Workload      │  CPU / Memory │  Rationale             │
  ├────────────────┼───────────────┼────────────────────────┤
  │  Go API        │  256 / 512    │  Low overhead, fast    │
  │  Java API      │  512 / 1024   │  JVM needs more RAM    │
  │  Worker (Go)   │  256 / 512    │  SQS consumer          │
  │  Worker (Java) │  512 / 1024   │  Heavier processing    │
  │  Load test     │  1024 / 2048  │  CPU-intensive         │
  └────────────────┴───────────────┴────────────────────────┘

JVM Container Flags
══════════════════════════════════

  -XX:+UseContainerSupport         → Detect cgroup limits
  -XX:MaxRAMPercentage=75.0        → Use 75% of container memory
  -XX:InitialRAMPercentage=50.0    → Start with 50%
  -XX:+UseG1GC                     → Good for containers
  -XX:+ExitOnOutOfMemoryError      → Fail fast, let ECS restart
```

---

## Anti-Patterns

| Anti-Pattern | Por que está errado | Como corrigir |
|---|---|---|
| Root user no container | Risco de privilege escalation | `USER nobody` ou `USER app` |
| `:latest` tag | Irreproducível, sem rollback | Tags imutáveis (`v1.2.3`, commit SHA) |
| Sem HEALTHCHECK | ECS não sabe se container está saudável | Definir no Dockerfile e task definition |
| Fat Docker image (>500MB) | Slow pull, mais superfície de ataque | Multi-stage, scratch/alpine |
| Secrets em ENV vars | Visíveis no task definition, logs | Usar Secrets Manager + `secrets` no ECS |
| Sem lifecycle policy no ECR | Imagens órfãs acumulam custo | Lifecycle policy com expiração |
| Single task (desired=1) | Sem redundância, downtime em deploy | Mínimo desired=2 em produção |
| Sem auto-scaling | Over/under provisioned | Target tracking (CPU, memory, requests) |

---

## Critérios de Aceite

### Dockerfiles
- [ ] Go: multi-stage com scratch, non-root, HEALTHCHECK, <20MB
- [ ] Java: multi-stage com layered JAR, non-root, HEALTHCHECK, <200MB
- [ ] `.dockerignore` configurado para ambos

### ECR
- [ ] Repositórios criados com scan-on-push e IMMUTABLE tags
- [ ] Lifecycle policy configurada (keep 10 tagged, expire untagged)
- [ ] Imagens publicadas com tag de versão semântica

### ECS
- [ ] Cluster criado com Fargate + Fargate Spot
- [ ] Task definitions para Go (256/512) e Java (512/1024)
- [ ] Services com desired=2, rolling update
- [ ] Health checks configurados no task definition
- [ ] Logs via awslogs para CloudWatch

### Auto-Scaling
- [ ] Scalable target registrado (min=2, max=10)
- [ ] CPU target tracking (70%)
- [ ] Memory target tracking (80%)

### SDK
- [ ] Go: ECS describe service, list tasks, update service
- [ ] Java: equivalente ao Go
- [ ] ECR: list images com scan status

---

## Definição de Pronto (DoD)

- [ ] Dockerfiles otimizados (Go + Java) com todas as best practices
- [ ] ECR setup com scan + lifecycle via script
- [ ] ECS cluster + task definitions + services via script
- [ ] Auto-scaling configurado via script
- [ ] SDK Go/Java com ECS management operations
- [ ] Deploy endpoint funcional
- [ ] Commit semântico: `feat(level-2): add ECS Fargate with ECR, auto-scaling and deploy management`

---

## Checklist

- [ ] Dockerfile Go: multi-stage, scratch, non-root
- [ ] Dockerfile Java: multi-stage, layered JAR, non-root
- [ ] ECR: repositórios com scan + lifecycle
- [ ] ECS: cluster + task def + service
- [ ] Auto-scaling: CPU + memory + request-based
- [ ] SDK Go: ECS describeService, listTasks, updateService
- [ ] SDK Java: equivalente
- [ ] ECR SDK: listImages, getScanFindings

---

## Exercícios Extras

1. **Blue/Green Deploy** — Configure ECS deployment controller como `CODE_DEPLOY`. Implemente um script que cria um segundo target group e troca o tráfego via CodeDeploy.
2. **Fargate Spot** — Configure o service para usar 75% Fargate Spot e 25% Fargate on-demand. Implemente graceful shutdown via SIGTERM handler no Go e Spring `@PreDestroy` no Java.
3. **Container Image Scanning** — Integre Trivy como step no build. Compare os findings do Trivy com os do ECR native scan. Bloqueie push se houver CVEs CRITICAL.
4. **ECS Exec** — Habilite ECS Exec para debug em containers running. Demonstre como acessar um container via `aws ecs execute-command`.
5. **Service Connect** — Configure ECS Service Connect como alternativa ao Cloud Map + ALB para comunicação service-to-service.
