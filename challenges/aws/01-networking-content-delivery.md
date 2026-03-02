# Level 1 — Networking & Content Delivery

> **Objetivo:** Entender a arquitetura de rede AWS (VPC, subnets, security groups),
> configurar ALB para distribuição de tráfego, usar Route 53 para DNS e CloudFront
> para CDN, tudo via AWS CLI e SDK.

---

## Objetivo de Aprendizado

- Projetar uma VPC com subnets públicas, privadas e de dados (3-tier)
- Entender CIDR planning e alocação de IPs
- Configurar Security Groups e NACLs com princípio de menor privilégio
- Provisionar ALB com health checks e target groups
- Usar VPC Endpoints para tráfego privado a serviços AWS (S3, DynamoDB, ECR)
- Configurar Route 53 hosted zones e records via SDK
- Entender CloudFront distributions e cache behaviors

---

## Escopo Funcional

```
Level 1 — Arquitetura de Rede do OrderFlow
══════════════════════════════════════════════

                      ┌────────────┐
                      │  Route 53  │
                      │ DNS        │
                      └─────┬──────┘
                            │
                      ┌─────▼──────┐
      Internet ──────►│ CloudFront │
                      │ (CDN)      │
                      └─────┬──────┘
                            │
                 ┌──────────▼──────────┐
                 │   Internet Gateway   │
                 └──────────┬──────────┘
                            │
  ┌─────────────── VPC: 10.0.0.0/16 ──────────────────────────────┐
  │                         │                                       │
  │  ┌─ Public Subnets ────────────────────────────────────┐       │
  │  │  10.0.100.0/24 (AZ-a)  │  10.0.101.0/24 (AZ-b)     │       │
  │  │  ┌─────┐ ┌─────┐       │  ┌─────┐ ┌─────┐          │       │
  │  │  │ ALB │ │NAT  │       │  │ ALB │ │NAT  │          │       │
  │  │  │     │ │ GW  │       │  │     │ │ GW  │          │       │
  │  │  └──┬──┘ └──┬──┘       │  └──┬──┘ └──┬──┘          │       │
  │  └─────┼───────┼──────────┼─────┼───────┼─────────────┘       │
  │        │       │          │     │       │                       │
  │  ┌─ Private Subnets ─────┼─────────────┼──────────────┐       │
  │  │  10.0.0.0/24           │  10.0.1.0/24               │       │
  │  │  ┌──────────┐          │  ┌──────────┐              │       │
  │  │  │ ECS Tasks│          │  │ ECS Tasks│              │       │
  │  │  │ (Go/Java)│          │  │ (Go/Java)│              │       │
  │  │  └──────────┘          │  └──────────┘              │       │
  │  └────────────────────────┼────────────────────────────┘       │
  │                           │                                     │
  │  ┌─ Data Subnets ────────┼────────────────────────────┐       │
  │  │  10.0.200.0/24         │  10.0.201.0/24             │       │
  │  │  ┌──────┐ ┌───────┐   │  ┌──────┐ ┌───────┐       │       │
  │  │  │ RDS  │ │ Redis │   │  │ RDS  │ │ Redis │       │       │
  │  │  └──────┘ └───────┘   │  └──────┘ └───────┘       │       │
  │  └────────────────────────┼────────────────────────────┘       │
  │                           │                                     │
  │  VPC Endpoints: ─────────────────────────────────────          │
  │  │ S3 (Gateway) │ DynamoDB (Gateway) │ ECR (Interface) │      │
  │  │ Logs (Interface) │ STS (Interface) │ Secrets (Interface)│  │
  └────────────────────────────────────────────────────────────────┘
```

---

## AWS Services deste Level

| Serviço | Tipo | Propósito no OrderFlow |
|---|---|---|
| VPC | Networking | Rede isolada para toda a aplicação |
| Subnets | Networking | Segmentação em tiers (public/private/data) |
| Security Groups | Firewall | Controle de tráfego stateful (L4) |
| NACLs | Firewall | Controle de tráfego stateless (L3) |
| ALB | Load Balancer | Distribuição de tráfego HTTP/HTTPS |
| NAT Gateway | Networking | Acesso à internet para subnets privadas |
| VPC Endpoints | Networking | Acesso privado a serviços AWS |
| Route 53 | DNS | Resolução de nomes, health checks |
| CloudFront | CDN | Cache e distribuição global de conteúdo |

---

## Tarefas

### Tarefa 1.1 — Criar VPC 3-Tier via AWS CLI

```bash
#!/bin/bash
# scripts/create-vpc.sh — VPC 3-tier para OrderFlow no LocalStack

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"
PROJECT="orderflow"
ENV="dev"

echo "=== Creating VPC ==="
VPC_ID=$($AWS ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=${PROJECT}-${ENV}-vpc},{Key=Environment,Value=${ENV}}]" \
  --query 'Vpc.VpcId' --output text)

echo "VPC created: $VPC_ID"

# Enable DNS hostnames
$AWS ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames '{"Value": true}'
$AWS ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support '{"Value": true}'

echo "=== Creating Internet Gateway ==="
IGW_ID=$($AWS ec2 create-internet-gateway \
  --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=${PROJECT}-${ENV}-igw}]" \
  --query 'InternetGateway.InternetGatewayId' --output text)

$AWS ec2 attach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"
echo "IGW attached: $IGW_ID"

echo "=== Creating Subnets ==="
# Public subnets
PUB_A=$($AWS ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.100.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-${ENV}-public-a},{Key=Tier,Value=public}]" \
  --query 'Subnet.SubnetId' --output text)

PUB_B=$($AWS ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.101.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-${ENV}-public-b},{Key=Tier,Value=public}]" \
  --query 'Subnet.SubnetId' --output text)

# Private subnets (application)
PRIV_A=$($AWS ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.0.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-${ENV}-private-a},{Key=Tier,Value=private}]" \
  --query 'Subnet.SubnetId' --output text)

PRIV_B=$($AWS ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-${ENV}-private-b},{Key=Tier,Value=private}]" \
  --query 'Subnet.SubnetId' --output text)

# Data subnets (databases)
DATA_A=$($AWS ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.200.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-${ENV}-data-a},{Key=Tier,Value=data}]" \
  --query 'Subnet.SubnetId' --output text)

DATA_B=$($AWS ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.201.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${PROJECT}-${ENV}-data-b},{Key=Tier,Value=data}]" \
  --query 'Subnet.SubnetId' --output text)

echo "Subnets: public=[$PUB_A, $PUB_B] private=[$PRIV_A, $PRIV_B] data=[$DATA_A, $DATA_B]"

echo "=== Creating Route Tables ==="
# Public route table
PUB_RT=$($AWS ec2 create-route-table --vpc-id "$VPC_ID" \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${PROJECT}-${ENV}-public-rt}]" \
  --query 'RouteTable.RouteTableId' --output text)

$AWS ec2 create-route --route-table-id "$PUB_RT" --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"
$AWS ec2 associate-route-table --route-table-id "$PUB_RT" --subnet-id "$PUB_A"
$AWS ec2 associate-route-table --route-table-id "$PUB_RT" --subnet-id "$PUB_B"

echo "=== Creating Security Groups ==="
# ALB Security Group
ALB_SG=$($AWS ec2 create-security-group --vpc-id "$VPC_ID" \
  --group-name "${PROJECT}-${ENV}-alb-sg" \
  --description "ALB - HTTP/HTTPS from anywhere" \
  --query 'GroupId' --output text)

$AWS ec2 authorize-security-group-ingress --group-id "$ALB_SG" \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
$AWS ec2 authorize-security-group-ingress --group-id "$ALB_SG" \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# App Security Group
APP_SG=$($AWS ec2 create-security-group --vpc-id "$VPC_ID" \
  --group-name "${PROJECT}-${ENV}-app-sg" \
  --description "App - traffic from ALB only" \
  --query 'GroupId' --output text)

$AWS ec2 authorize-security-group-ingress --group-id "$APP_SG" \
  --protocol tcp --port 8080 --source-group "$ALB_SG"

# Data Security Group
DATA_SG=$($AWS ec2 create-security-group --vpc-id "$VPC_ID" \
  --group-name "${PROJECT}-${ENV}-data-sg" \
  --description "Data - traffic from App only" \
  --query 'GroupId' --output text)

$AWS ec2 authorize-security-group-ingress --group-id "$DATA_SG" \
  --protocol tcp --port 5432 --source-group "$APP_SG"
$AWS ec2 authorize-security-group-ingress --group-id "$DATA_SG" \
  --protocol tcp --port 6379 --source-group "$APP_SG"

echo "Security Groups: alb=$ALB_SG app=$APP_SG data=$DATA_SG"

echo "=== Saving outputs ==="
cat > scripts/vpc-outputs.json << EOF
{
  "vpc_id": "$VPC_ID",
  "igw_id": "$IGW_ID",
  "public_subnets": ["$PUB_A", "$PUB_B"],
  "private_subnets": ["$PRIV_A", "$PRIV_B"],
  "data_subnets": ["$DATA_A", "$DATA_B"],
  "security_groups": {
    "alb": "$ALB_SG",
    "app": "$APP_SG",
    "data": "$DATA_SG"
  }
}
EOF

echo "=== VPC setup complete! ==="
```

### Tarefa 1.2 — Consultar VPC e Subnets via SDK (Go)

```go
// internal/aws/networking.go
package aws

import (
	"context"
	"fmt"
	"log/slog"

	"github.com/aws/aws-sdk-go-v2/aws"
	awsconfig "github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/ec2"
	"github.com/aws/aws-sdk-go-v2/service/ec2/types"
)

type NetworkingClient struct {
	ec2 *ec2.Client
}

func NewNetworkingClient(cfg awsconfig.Config, endpoint *string) *NetworkingClient {
	opts := []func(*ec2.Options){}
	if endpoint != nil {
		opts = append(opts, func(o *ec2.Options) {
			o.BaseEndpoint = endpoint
		})
	}
	return &NetworkingClient{ec2: ec2.NewFromConfig(cfg, opts...)}
}

// VPCInfo contains VPC metadata for inspection.
type VPCInfo struct {
	VPCID     string
	CIDRBlock string
	State     string
	IsDefault bool
	Tags      map[string]string
}

// SubnetInfo contains subnet metadata.
type SubnetInfo struct {
	SubnetID         string
	VPCID            string
	CIDRBlock        string
	AZ               string
	AvailableIPs     int32
	MapPublicIP      bool
	Tags             map[string]string
}

// SecurityGroupRule represents an ingress/egress rule.
type SecurityGroupRule struct {
	Protocol    string
	FromPort    int32
	ToPort      int32
	Source      string // CIDR or SG ID
	Description string
}

// DescribeVPCs lists all VPCs in the account, with optional filters.
func (c *NetworkingClient) DescribeVPCs(ctx context.Context, filters map[string]string) ([]VPCInfo, error) {
	input := &ec2.DescribeVpcsInput{}

	if len(filters) > 0 {
		for k, v := range filters {
			input.Filters = append(input.Filters, types.Filter{
				Name:   aws.String(k),
				Values: []string{v},
			})
		}
	}

	output, err := c.ec2.DescribeVpcs(ctx, input)
	if err != nil {
		return nil, fmt.Errorf("describe VPCs: %w", err)
	}

	vpcs := make([]VPCInfo, 0, len(output.Vpcs))
	for _, vpc := range output.Vpcs {
		vpcs = append(vpcs, VPCInfo{
			VPCID:     aws.ToString(vpc.VpcId),
			CIDRBlock: aws.ToString(vpc.CidrBlock),
			State:     string(vpc.State),
			IsDefault: aws.ToBool(vpc.IsDefault),
			Tags:      extractTags(vpc.Tags),
		})
	}

	slog.Info("VPCs described", "count", len(vpcs))
	return vpcs, nil
}

// DescribeSubnets lists subnets by VPC, optionally filtered by tier tag.
func (c *NetworkingClient) DescribeSubnets(ctx context.Context, vpcID string, tier string) ([]SubnetInfo, error) {
	input := &ec2.DescribeSubnetsInput{
		Filters: []types.Filter{
			{Name: aws.String("vpc-id"), Values: []string{vpcID}},
		},
	}

	if tier != "" {
		input.Filters = append(input.Filters, types.Filter{
			Name:   aws.String("tag:Tier"),
			Values: []string{tier},
		})
	}

	output, err := c.ec2.DescribeSubnets(ctx, input)
	if err != nil {
		return nil, fmt.Errorf("describe subnets: %w", err)
	}

	subnets := make([]SubnetInfo, 0, len(output.Subnets))
	for _, s := range output.Subnets {
		subnets = append(subnets, SubnetInfo{
			SubnetID:     aws.ToString(s.SubnetId),
			VPCID:        aws.ToString(s.VpcId),
			CIDRBlock:    aws.ToString(s.CidrBlock),
			AZ:           aws.ToString(s.AvailabilityZone),
			AvailableIPs: aws.ToInt32(s.AvailableIpAddressCount),
			MapPublicIP:  aws.ToBool(s.MapPublicIpOnLaunch),
			Tags:         extractTags(s.Tags),
		})
	}

	return subnets, nil
}

// DescribeSecurityGroupRules returns rules for a given SG.
func (c *NetworkingClient) DescribeSecurityGroupRules(ctx context.Context, sgID string) ([]SecurityGroupRule, error) {
	output, err := c.ec2.DescribeSecurityGroups(ctx, &ec2.DescribeSecurityGroupsInput{
		GroupIds: []string{sgID},
	})
	if err != nil {
		return nil, fmt.Errorf("describe SG: %w", err)
	}

	if len(output.SecurityGroups) == 0 {
		return nil, fmt.Errorf("security group not found: %s", sgID)
	}

	sg := output.SecurityGroups[0]
	rules := make([]SecurityGroupRule, 0)

	for _, perm := range sg.IpPermissions {
		for _, cidr := range perm.IpRanges {
			rules = append(rules, SecurityGroupRule{
				Protocol:    aws.ToString(perm.IpProtocol),
				FromPort:    aws.ToInt32(perm.FromPort),
				ToPort:      aws.ToInt32(perm.ToPort),
				Source:      aws.ToString(cidr.CidrIp),
				Description: aws.ToString(cidr.Description),
			})
		}
		for _, sg := range perm.UserIdGroupPairs {
			rules = append(rules, SecurityGroupRule{
				Protocol:    aws.ToString(perm.IpProtocol),
				FromPort:    aws.ToInt32(perm.FromPort),
				ToPort:      aws.ToInt32(perm.ToPort),
				Source:      aws.ToString(sg.GroupId),
				Description: aws.ToString(sg.Description),
			})
		}
	}

	return rules, nil
}

func extractTags(tags []types.Tag) map[string]string {
	result := make(map[string]string, len(tags))
	for _, tag := range tags {
		result[aws.ToString(tag.Key)] = aws.ToString(tag.Value)
	}
	return result
}
```

### Tarefa 1.3 — VPC e Subnets via SDK (Java)

```java
// infrastructure/aws/NetworkingService.java
package com.orderflow.infrastructure.aws;

import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.ec2.Ec2Client;
import software.amazon.awssdk.services.ec2.model.*;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class NetworkingService {

    private final Ec2Client ec2Client;

    public NetworkingService(Ec2Client ec2Client) {
        this.ec2Client = ec2Client;
    }

    public record VpcInfo(
            String vpcId,
            String cidrBlock,
            String state,
            boolean isDefault,
            Map<String, String> tags
    ) {}

    public record SubnetInfo(
            String subnetId,
            String vpcId,
            String cidrBlock,
            String az,
            int availableIps,
            boolean mapPublicIp,
            Map<String, String> tags
    ) {}

    public record SecurityGroupRuleInfo(
            String protocol,
            int fromPort,
            int toPort,
            String source,
            String description
    ) {}

    public List<VpcInfo> describeVpcs(Map<String, String> filters) {
        var builder = DescribeVpcsRequest.builder();

        if (filters != null && !filters.isEmpty()) {
            builder.filters(filters.entrySet().stream()
                    .map(e -> Filter.builder()
                            .name(e.getKey())
                            .values(e.getValue())
                            .build())
                    .toList());
        }

        return ec2Client.describeVpcs(builder.build()).vpcs().stream()
                .map(vpc -> new VpcInfo(
                        vpc.vpcId(),
                        vpc.cidrBlock(),
                        vpc.stateAsString(),
                        vpc.isDefault(),
                        extractTags(vpc.tags())
                ))
                .toList();
    }

    public List<SubnetInfo> describeSubnets(String vpcId, String tier) {
        var filters = new java.util.ArrayList<Filter>();
        filters.add(Filter.builder().name("vpc-id").values(vpcId).build());

        if (tier != null && !tier.isEmpty()) {
            filters.add(Filter.builder().name("tag:Tier").values(tier).build());
        }

        return ec2Client.describeSubnets(DescribeSubnetsRequest.builder()
                        .filters(filters)
                        .build())
                .subnets().stream()
                .map(s -> new SubnetInfo(
                        s.subnetId(),
                        s.vpcId(),
                        s.cidrBlock(),
                        s.availabilityZone(),
                        s.availableIpAddressCount(),
                        s.mapPublicIpOnLaunch(),
                        extractTags(s.tags())
                ))
                .toList();
    }

    public List<SecurityGroupRuleInfo> describeSecurityGroupRules(String sgId) {
        var response = ec2Client.describeSecurityGroups(
                DescribeSecurityGroupsRequest.builder()
                        .groupIds(sgId)
                        .build());

        if (response.securityGroups().isEmpty()) {
            throw new IllegalArgumentException("SG not found: " + sgId);
        }

        var sg = response.securityGroups().getFirst();
        return sg.ipPermissions().stream()
                .flatMap(perm -> {
                    var fromCidr = perm.ipRanges().stream()
                            .map(cidr -> new SecurityGroupRuleInfo(
                                    perm.ipProtocol(),
                                    perm.fromPort(),
                                    perm.toPort(),
                                    cidr.cidrIp(),
                                    cidr.description()
                            ));
                    var fromSg = perm.userIdGroupPairs().stream()
                            .map(pair -> new SecurityGroupRuleInfo(
                                    perm.ipProtocol(),
                                    perm.fromPort(),
                                    perm.toPort(),
                                    pair.groupId(),
                                    pair.description()
                            ));
                    return java.util.stream.Stream.concat(fromCidr, fromSg);
                })
                .toList();
    }

    private Map<String, String> extractTags(List<Tag> tags) {
        if (tags == null) return Map.of();
        return tags.stream()
                .collect(Collectors.toMap(Tag::key, Tag::value));
    }
}
```

### Tarefa 1.4 — Network Health Check Endpoint (Go)

```go
// internal/handler/health.go
package handler

import (
	"context"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"

	awsclient "orderflow/internal/aws"
)

type HealthHandler struct {
	networking *awsclient.NetworkingClient
	vpcID      string
}

func NewHealthHandler(networking *awsclient.NetworkingClient, vpcID string) *HealthHandler {
	return &HealthHandler{networking: networking, vpcID: vpcID}
}

// NetworkHealth returns VPC status including subnets and SGs.
// GET /health/network
func (h *HealthHandler) NetworkHealth(c *gin.Context) {
	ctx, cancel := context.WithTimeout(c.Request.Context(), 10*time.Second)
	defer cancel()

	type subnetCheck struct {
		SubnetID     string `json:"subnet_id"`
		AZ           string `json:"az"`
		Tier         string `json:"tier"`
		AvailableIPs int32  `json:"available_ips"`
		Healthy      bool   `json:"healthy"`
	}

	type networkStatus struct {
		VPCID   string        `json:"vpc_id"`
		Status  string        `json:"status"`
		Subnets []subnetCheck `json:"subnets"`
	}

	// Get VPC info
	vpcs, err := h.networking.DescribeVPCs(ctx, map[string]string{
		"vpc-id": h.vpcID,
	})
	if err != nil || len(vpcs) == 0 {
		c.JSON(http.StatusServiceUnavailable, gin.H{
			"status": "unhealthy",
			"error":  "VPC not found or unreachable",
		})
		return
	}

	// Check subnets
	subnets, err := h.networking.DescribeSubnets(ctx, h.vpcID, "")
	if err != nil {
		c.JSON(http.StatusServiceUnavailable, gin.H{
			"status": "unhealthy",
			"error":  "Cannot describe subnets",
		})
		return
	}

	checks := make([]subnetCheck, 0, len(subnets))
	allHealthy := true
	for _, s := range subnets {
		healthy := s.AvailableIPs > 10 // At least 10 IPs available
		if !healthy {
			allHealthy = false
		}
		checks = append(checks, subnetCheck{
			SubnetID:     s.SubnetID,
			AZ:           s.AZ,
			Tier:         s.Tags["Tier"],
			AvailableIPs: s.AvailableIPs,
			Healthy:      healthy,
		})
	}

	status := "healthy"
	if !allHealthy {
		status = "degraded"
	}

	c.JSON(http.StatusOK, networkStatus{
		VPCID:   vpcs[0].VPCID,
		Status:  status,
		Subnets: checks,
	})
}
```

### Tarefa 1.5 — ALB Provisioning e Target Group via CLI

```bash
#!/bin/bash
# scripts/create-alb.sh — ALB para OrderFlow

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"
PROJECT="orderflow"

# Load VPC outputs
VPC_ID=$(jq -r '.vpc_id' scripts/vpc-outputs.json)
PUB_A=$(jq -r '.public_subnets[0]' scripts/vpc-outputs.json)
PUB_B=$(jq -r '.public_subnets[1]' scripts/vpc-outputs.json)
ALB_SG=$(jq -r '.security_groups.alb' scripts/vpc-outputs.json)

echo "=== Creating Target Group ==="
TG_ARN=$($AWS elbv2 create-target-group \
  --name "${PROJECT}-app-tg" \
  --protocol HTTP \
  --port 8080 \
  --vpc-id "$VPC_ID" \
  --target-type ip \
  --health-check-enabled \
  --health-check-path "/health" \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

echo "Target Group: $TG_ARN"

echo "=== Creating ALB ==="
ALB_ARN=$($AWS elbv2 create-load-balancer \
  --name "${PROJECT}-alb" \
  --subnets "$PUB_A" "$PUB_B" \
  --security-groups "$ALB_SG" \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

ALB_DNS=$($AWS elbv2 describe-load-balancers \
  --load-balancer-arns "$ALB_ARN" \
  --query 'LoadBalancers[0].DNSName' --output text)

echo "ALB: $ALB_ARN"
echo "DNS: $ALB_DNS"

echo "=== Creating Listener ==="
$AWS elbv2 create-listener \
  --load-balancer-arn "$ALB_ARN" \
  --protocol HTTP \
  --port 80 \
  --default-actions "Type=forward,TargetGroupArn=$TG_ARN"

echo "=== Saving ALB outputs ==="
cat > scripts/alb-outputs.json << EOF
{
  "alb_arn": "$ALB_ARN",
  "alb_dns": "$ALB_DNS",
  "target_group_arn": "$TG_ARN"
}
EOF

echo "=== ALB setup complete! ==="
echo "Endpoint: http://$ALB_DNS"
```

### Tarefa 1.6 — ALB Health e Target Status via SDK (Go)

```go
// internal/aws/loadbalancer.go
package aws

import (
	"context"
	"fmt"
	"log/slog"

	"github.com/aws/aws-sdk-go-v2/aws"
	awsconfig "github.com/aws/aws-sdk-go-v2/aws"
	elbv2 "github.com/aws/aws-sdk-go-v2/service/elasticloadbalancingv2"
	elbv2types "github.com/aws/aws-sdk-go-v2/service/elasticloadbalancingv2/types"
)

type LoadBalancerClient struct {
	client *elbv2.Client
}

func NewLoadBalancerClient(cfg awsconfig.Config, endpoint *string) *LoadBalancerClient {
	opts := []func(*elbv2.Options){}
	if endpoint != nil {
		opts = append(opts, func(o *elbv2.Options) {
			o.BaseEndpoint = endpoint
		})
	}
	return &LoadBalancerClient{client: elbv2.NewFromConfig(cfg, opts...)}
}

type ALBStatus struct {
	Name            string
	DNSName         string
	State           string
	Scheme          string
	AvailabilityZones []string
}

type TargetHealth struct {
	TargetID string
	Port     int32
	State    string
	Reason   string
}

// DescribeALB returns the ALB status.
func (c *LoadBalancerClient) DescribeALB(ctx context.Context, albArn string) (*ALBStatus, error) {
	output, err := c.client.DescribeLoadBalancers(ctx, &elbv2.DescribeLoadBalancersInput{
		LoadBalancerArns: []string{albArn},
	})
	if err != nil {
		return nil, fmt.Errorf("describe ALB: %w", err)
	}

	if len(output.LoadBalancers) == 0 {
		return nil, fmt.Errorf("ALB not found: %s", albArn)
	}

	alb := output.LoadBalancers[0]
	azs := make([]string, 0, len(alb.AvailabilityZones))
	for _, az := range alb.AvailabilityZones {
		azs = append(azs, aws.ToString(az.ZoneName))
	}

	return &ALBStatus{
		Name:              aws.ToString(alb.LoadBalancerName),
		DNSName:           aws.ToString(alb.DNSName),
		State:             string(alb.State.Code),
		Scheme:            string(alb.Scheme),
		AvailabilityZones: azs,
	}, nil
}

// DescribeTargetHealth returns health status of all targets.
func (c *LoadBalancerClient) DescribeTargetHealth(ctx context.Context, targetGroupArn string) ([]TargetHealth, error) {
	output, err := c.client.DescribeTargetHealth(ctx, &elbv2.DescribeTargetHealthInput{
		TargetGroupArn: aws.String(targetGroupArn),
	})
	if err != nil {
		return nil, fmt.Errorf("describe target health: %w", err)
	}

	targets := make([]TargetHealth, 0, len(output.TargetHealthDescriptions))
	for _, t := range output.TargetHealthDescriptions {
		reason := ""
		if t.TargetHealth.Reason != "" {
			reason = string(t.TargetHealth.Reason)
		}
		targets = append(targets, TargetHealth{
			TargetID: aws.ToString(t.Target.Id),
			Port:     aws.ToInt32(t.Target.Port),
			State:    string(t.TargetHealth.State),
			Reason:   reason,
		})
	}

	slog.Info("Target health described",
		"target_group", targetGroupArn,
		"healthy", countByState(targets, elbv2types.TargetHealthStateEnumHealthy),
		"unhealthy", countByState(targets, elbv2types.TargetHealthStateEnumUnhealthy),
	)

	return targets, nil
}

func countByState(targets []TargetHealth, state elbv2types.TargetHealthStateEnum) int {
	count := 0
	for _, t := range targets {
		if t.State == string(state) {
			count++
		}
	}
	return count
}
```

### Tarefa 1.7 — Route 53 DNS Management via SDK (Go)

```go
// internal/aws/dns.go
package aws

import (
	"context"
	"fmt"
	"log/slog"

	"github.com/aws/aws-sdk-go-v2/aws"
	awsconfig "github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/route53"
	r53types "github.com/aws/aws-sdk-go-v2/service/route53/types"
)

type DNSClient struct {
	client *route53.Client
}

func NewDNSClient(cfg awsconfig.Config, endpoint *string) *DNSClient {
	opts := []func(*route53.Options){}
	if endpoint != nil {
		opts = append(opts, func(o *route53.Options) {
			o.BaseEndpoint = endpoint
		})
	}
	return &DNSClient{client: route53.NewFromConfig(cfg, opts...)}
}

// CreateHostedZone creates a private or public hosted zone.
func (c *DNSClient) CreateHostedZone(ctx context.Context, domain string, vpcID string, region string) (string, error) {
	input := &route53.CreateHostedZoneInput{
		Name:            aws.String(domain),
		CallerReference: aws.String(fmt.Sprintf("%s-%d", domain, 1)),
	}

	// Private hosted zone (associated with VPC)
	if vpcID != "" {
		input.VPC = &r53types.VPC{
			VPCId:     aws.String(vpcID),
			VPCRegion: r53types.VPCRegion(region),
		}
		input.HostedZoneConfig = &r53types.HostedZoneConfig{
			PrivateZone: true,
		}
	}

	output, err := c.client.CreateHostedZone(ctx, input)
	if err != nil {
		return "", fmt.Errorf("create hosted zone: %w", err)
	}

	zoneID := aws.ToString(output.HostedZone.Id)
	slog.Info("Hosted zone created", "domain", domain, "zone_id", zoneID, "private", vpcID != "")
	return zoneID, nil
}

// UpsertRecord creates or updates a DNS record in a hosted zone.
func (c *DNSClient) UpsertRecord(ctx context.Context, zoneID, name, recordType, value string, ttl int64) error {
	_, err := c.client.ChangeResourceRecordSets(ctx, &route53.ChangeResourceRecordSetsInput{
		HostedZoneId: aws.String(zoneID),
		ChangeBatch: &r53types.ChangeBatch{
			Comment: aws.String(fmt.Sprintf("Upsert %s %s", recordType, name)),
			Changes: []r53types.Change{
				{
					Action: r53types.ChangeActionUpsert,
					ResourceRecordSet: &r53types.ResourceRecordSet{
						Name: aws.String(name),
						Type: r53types.RRType(recordType),
						TTL:  aws.Int64(ttl),
						ResourceRecords: []r53types.ResourceRecord{
							{Value: aws.String(value)},
						},
					},
				},
			},
		},
	})
	if err != nil {
		return fmt.Errorf("upsert record: %w", err)
	}

	slog.Info("DNS record upserted", "zone", zoneID, "name", name, "type", recordType, "value", value)
	return nil
}

// CreateAliasRecord creates an alias record pointing to an ALB.
func (c *DNSClient) CreateAliasRecord(ctx context.Context, zoneID, name, albDNS, albHostedZoneID string) error {
	_, err := c.client.ChangeResourceRecordSets(ctx, &route53.ChangeResourceRecordSetsInput{
		HostedZoneId: aws.String(zoneID),
		ChangeBatch: &r53types.ChangeBatch{
			Changes: []r53types.Change{
				{
					Action: r53types.ChangeActionUpsert,
					ResourceRecordSet: &r53types.ResourceRecordSet{
						Name: aws.String(name),
						Type: r53types.RRTypeA,
						AliasTarget: &r53types.AliasTarget{
							DNSName:              aws.String(albDNS),
							HostedZoneId:         aws.String(albHostedZoneID),
							EvaluateTargetHealth: true,
						},
					},
				},
			},
		},
	})
	if err != nil {
		return fmt.Errorf("create alias record: %w", err)
	}

	slog.Info("Alias record created", "name", name, "target", albDNS)
	return nil
}

// ListRecords returns all records in a hosted zone.
func (c *DNSClient) ListRecords(ctx context.Context, zoneID string) ([]r53types.ResourceRecordSet, error) {
	output, err := c.client.ListResourceRecordSets(ctx, &route53.ListResourceRecordSetsInput{
		HostedZoneId: aws.String(zoneID),
	})
	if err != nil {
		return nil, fmt.Errorf("list records: %w", err)
	}
	return output.ResourceRecordSets, nil
}
```

### Tarefa 1.8 — Route 53 DNS Management (Java)

```java
// infrastructure/aws/DnsService.java
package com.orderflow.infrastructure.aws;

import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.route53.Route53Client;
import software.amazon.awssdk.services.route53.model.*;

import java.util.List;

@Service
public class DnsService {

    private final Route53Client route53Client;

    public DnsService(Route53Client route53Client) {
        this.route53Client = route53Client;
    }

    public record DnsRecord(String name, String type, Long ttl, List<String> values) {}

    public String createHostedZone(String domain, String vpcId, String region) {
        var builder = CreateHostedZoneRequest.builder()
                .name(domain)
                .callerReference(domain + "-" + System.currentTimeMillis());

        if (vpcId != null && !vpcId.isEmpty()) {
            builder.vpc(VPC.builder()
                            .vpcId(vpcId)
                            .vpcRegion(VPCRegion.fromValue(region))
                            .build())
                    .hostedZoneConfig(HostedZoneConfig.builder()
                            .privateZone(true)
                            .build());
        }

        var response = route53Client.createHostedZone(builder.build());
        return response.hostedZone().id();
    }

    public void upsertRecord(String zoneId, String name, String type, String value, long ttl) {
        route53Client.changeResourceRecordSets(ChangeResourceRecordSetsRequest.builder()
                .hostedZoneId(zoneId)
                .changeBatch(ChangeBatch.builder()
                        .changes(Change.builder()
                                .action(ChangeAction.UPSERT)
                                .resourceRecordSet(ResourceRecordSet.builder()
                                        .name(name)
                                        .type(RRType.fromValue(type))
                                        .ttl(ttl)
                                        .resourceRecords(ResourceRecord.builder()
                                                .value(value)
                                                .build())
                                        .build())
                                .build())
                        .build())
                .build());
    }

    public void createAliasRecord(String zoneId, String name, String albDns, String albHostedZoneId) {
        route53Client.changeResourceRecordSets(ChangeResourceRecordSetsRequest.builder()
                .hostedZoneId(zoneId)
                .changeBatch(ChangeBatch.builder()
                        .changes(Change.builder()
                                .action(ChangeAction.UPSERT)
                                .resourceRecordSet(ResourceRecordSet.builder()
                                        .name(name)
                                        .type(RRType.A)
                                        .aliasTarget(AliasTarget.builder()
                                                .dnsName(albDns)
                                                .hostedZoneId(albHostedZoneId)
                                                .evaluateTargetHealth(true)
                                                .build())
                                        .build())
                                .build())
                        .build())
                .build());
    }

    public List<DnsRecord> listRecords(String zoneId) {
        var response = route53Client.listResourceRecordSets(
                ListResourceRecordSetsRequest.builder()
                        .hostedZoneId(zoneId)
                        .build());

        return response.resourceRecordSets().stream()
                .map(rrs -> new DnsRecord(
                        rrs.name(),
                        rrs.typeAsString(),
                        rrs.ttl(),
                        rrs.resourceRecords().stream()
                                .map(ResourceRecord::value)
                                .toList()
                ))
                .toList();
    }
}
```

### Tarefa 1.9 — VPC Endpoint Audit (Go)

```go
// internal/aws/vpc_endpoints.go
package aws

import (
	"context"
	"fmt"
	"log/slog"

	awsv2 "github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/ec2"
	"github.com/aws/aws-sdk-go-v2/service/ec2/types"
)

// VPCEndpointInfo describes a VPC endpoint.
type VPCEndpointInfo struct {
	EndpointID   string
	ServiceName  string
	Type         string // Gateway or Interface
	State        string
	RouteTableIDs []string
	SubnetIDs     []string
}

// Required VPC Endpoints for OrderFlow
var requiredEndpoints = []string{
	"com.amazonaws.us-east-1.s3",
	"com.amazonaws.us-east-1.dynamodb",
	"com.amazonaws.us-east-1.ecr.api",
	"com.amazonaws.us-east-1.ecr.dkr",
	"com.amazonaws.us-east-1.logs",
	"com.amazonaws.us-east-1.secretsmanager",
	"com.amazonaws.us-east-1.sts",
}

// DescribeVPCEndpoints lists all VPC endpoints.
func (c *NetworkingClient) DescribeVPCEndpoints(ctx context.Context, vpcID string) ([]VPCEndpointInfo, error) {
	output, err := c.ec2.DescribeVpcEndpoints(ctx, &ec2.DescribeVpcEndpointsInput{
		Filters: []types.Filter{
			{Name: awsv2.String("vpc-id"), Values: []string{vpcID}},
		},
	})
	if err != nil {
		return nil, fmt.Errorf("describe VPC endpoints: %w", err)
	}

	endpoints := make([]VPCEndpointInfo, 0, len(output.VpcEndpoints))
	for _, ep := range output.VpcEndpoints {
		endpoints = append(endpoints, VPCEndpointInfo{
			EndpointID:    awsv2.ToString(ep.VpcEndpointId),
			ServiceName:   awsv2.ToString(ep.ServiceName),
			Type:          string(ep.VpcEndpointType),
			State:         string(ep.State),
			RouteTableIDs: ep.RouteTableIds,
			SubnetIDs:     ep.SubnetIds,
		})
	}

	return endpoints, nil
}

// AuditVPCEndpoints checks if all required endpoints are configured.
func (c *NetworkingClient) AuditVPCEndpoints(ctx context.Context, vpcID string) (map[string]string, error) {
	endpoints, err := c.DescribeVPCEndpoints(ctx, vpcID)
	if err != nil {
		return nil, err
	}

	// Build a map of existing endpoints
	existing := make(map[string]string)
	for _, ep := range endpoints {
		existing[ep.ServiceName] = ep.State
	}

	// Check required endpoints
	audit := make(map[string]string)
	for _, required := range requiredEndpoints {
		if state, ok := existing[required]; ok {
			audit[required] = state
		} else {
			audit[required] = "MISSING"
		}
	}

	// Log results
	missing := 0
	for svc, state := range audit {
		if state == "MISSING" {
			missing++
			slog.Warn("VPC endpoint missing", "service", svc)
		}
	}

	if missing > 0 {
		slog.Warn("VPC endpoint audit incomplete", "missing", missing, "total", len(requiredEndpoints))
	} else {
		slog.Info("VPC endpoint audit passed", "total", len(requiredEndpoints))
	}

	return audit, nil
}
```

### Tarefa 1.10 — Teste de Integração: VPC (Go)

```go
// internal/aws/networking_test.go
package aws_test

import (
	"context"
	"testing"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/credentials"
	"github.com/aws/aws-sdk-go-v2/service/ec2"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"github.com/testcontainers/testcontainers-go/modules/localstack"

	awsclient "orderflow/internal/aws"
)

func setupEC2(t *testing.T) (*awsclient.NetworkingClient, *ec2.Client) {
	t.Helper()
	ctx := context.Background()

	container, err := localstack.Run(ctx, "localstack/localstack:latest")
	require.NoError(t, err)
	t.Cleanup(func() { container.Terminate(ctx) })

	endpoint, err := container.PortEndpoint(ctx, "4566/tcp", "http")
	require.NoError(t, err)

	cfg, err := config.LoadDefaultConfig(ctx,
		config.WithRegion("us-east-1"),
		config.WithCredentialsProvider(
			credentials.NewStaticCredentialsProvider("test", "test", ""),
		),
	)
	require.NoError(t, err)

	networking := awsclient.NewNetworkingClient(cfg, aws.String(endpoint))

	ec2Client := ec2.NewFromConfig(cfg, func(o *ec2.Options) {
		o.BaseEndpoint = aws.String(endpoint)
	})

	return networking, ec2Client
}

func TestDescribeVPCs(t *testing.T) {
	networking, ec2Client := setupEC2(t)
	ctx := context.Background()

	// Create a VPC
	createOutput, err := ec2Client.CreateVpc(ctx, &ec2.CreateVpcInput{
		CidrBlock: aws.String("10.0.0.0/16"),
	})
	require.NoError(t, err)

	vpcID := aws.ToString(createOutput.Vpc.VpcId)

	// Describe VPCs
	vpcs, err := networking.DescribeVPCs(ctx, map[string]string{
		"vpc-id": vpcID,
	})
	require.NoError(t, err)

	assert.Len(t, vpcs, 1)
	assert.Equal(t, vpcID, vpcs[0].VPCID)
	assert.Equal(t, "10.0.0.0/16", vpcs[0].CIDRBlock)
}

func TestDescribeSubnets(t *testing.T) {
	networking, ec2Client := setupEC2(t)
	ctx := context.Background()

	// Create VPC
	vpc, err := ec2Client.CreateVpc(ctx, &ec2.CreateVpcInput{
		CidrBlock: aws.String("10.0.0.0/16"),
	})
	require.NoError(t, err)

	vpcID := aws.ToString(vpc.Vpc.VpcId)

	// Create subnets
	_, err = ec2Client.CreateSubnet(ctx, &ec2.CreateSubnetInput{
		VpcId:     aws.String(vpcID),
		CidrBlock: aws.String("10.0.0.0/24"),
	})
	require.NoError(t, err)

	_, err = ec2Client.CreateSubnet(ctx, &ec2.CreateSubnetInput{
		VpcId:     aws.String(vpcID),
		CidrBlock: aws.String("10.0.1.0/24"),
	})
	require.NoError(t, err)

	// Describe all subnets in VPC
	subnets, err := networking.DescribeSubnets(ctx, vpcID, "")
	require.NoError(t, err)

	assert.GreaterOrEqual(t, len(subnets), 2)
}
```

---

## Security Groups — Regras de Referência

```
Regras de Security Group do OrderFlow
══════════════════════════════════════════

┌─────────────────────────────────────────────────────────┐
│ ALB Security Group (alb-sg)                             │
│                                                         │
│ Ingress:                                                │
│   TCP 80  ← 0.0.0.0/0       (HTTP redirect)            │
│   TCP 443 ← 0.0.0.0/0       (HTTPS)                    │
│                                                         │
│ Egress:                                                 │
│   TCP 8080 → app-sg          (para targets)             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ App Security Group (app-sg)                             │
│                                                         │
│ Ingress:                                                │
│   TCP 8080 ← alb-sg          (do ALB)                   │
│                                                         │
│ Egress:                                                 │
│   TCP 443  → 0.0.0.0/0       (AWS APIs via endpoints)   │
│   TCP 5432 → data-sg          (PostgreSQL)              │
│   TCP 6379 → data-sg          (Redis)                   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Data Security Group (data-sg)                           │
│                                                         │
│ Ingress:                                                │
│   TCP 5432 ← app-sg           (PostgreSQL)              │
│   TCP 6379 ← app-sg           (Redis)                   │
│                                                         │
│ Egress:                                                 │
│   None (deny all)                                       │
└─────────────────────────────────────────────────────────┘

Princípio: SG referencia outro SG (não CIDR)
→ Menor privilégio, desacoplado de IPs
```

---

## Anti-Patterns

| Anti-Pattern | Por que está errado | Como corrigir |
|---|---|---|
| SG com `0.0.0.0/0` na porta 8080 | App exposta diretamente à internet | Permitir apenas do ALB SG |
| Single AZ deployment | Single point of failure | Usar pelo menos 2 AZs |
| Sem VPC Endpoints | Tráfego AWS vai pela internet/NAT | Criar Gateway/Interface endpoints |
| NACL permissiva (allow all) | Sem segunda camada de defesa | Aplicar regras deny por padrão |
| Subnet /16 (toda a VPC como subnet) | Sem segmentação, blast radius máximo | Dividir em /24 por tier e AZ |
| Hardcoded IPs em SG rules | Quebra quando IPs mudam | Usar SG references entre security groups |
| Sem Flow Logs | Impossível auditar tráfego | Habilitar VPC Flow Logs para CloudWatch |

---

## Critérios de Aceite

### VPC
- [ ] VPC criada com CIDR /16 e DNS habilitado
- [ ] 6 subnets: 2 public, 2 private, 2 data (em 2 AZs)
- [ ] Internet Gateway attachado à VPC
- [ ] Route table pública com rota para IGW

### Security Groups
- [ ] ALB SG: ingress 80/443 de `0.0.0.0/0`
- [ ] App SG: ingress 8080 apenas do ALB SG
- [ ] Data SG: ingress 5432/6379 apenas do App SG
- [ ] Nenhum SG com regras excessivamente permissivas

### ALB
- [ ] ALB criado nas public subnets com internet-facing
- [ ] Target group com health check em `/health`
- [ ] Listener HTTP na porta 80 encaminhando para target group

### SDK (Go + Java)
- [ ] DescribeVPCs retorna informações corretas
- [ ] DescribeSubnets filtra por VPC e tier
- [ ] DescribeSecurityGroupRules lista regras do SG
- [ ] DNS: criação de hosted zone e records (A, CNAME, Alias)
- [ ] VPC Endpoint audit identifica endpoints faltantes

### Testes
- [ ] Teste de integração VPC com Testcontainers (Go ou Java)

---

## Definição de Pronto (DoD)

- [ ] VPC 3-tier funcional via script (`scripts/create-vpc.sh`)
- [ ] ALB provisionado via script (`scripts/create-alb.sh`)
- [ ] SDK Go: networking client com describe VPC/subnet/SG
- [ ] SDK Java: networking service equivalente
- [ ] Route 53: hosted zone + records via SDK
- [ ] VPC Endpoint audit implementado
- [ ] Pelo menos 1 teste de integração passando
- [ ] Commit semântico: `feat(level-1): add networking with VPC, ALB, Route 53 and VPC endpoints`

---

## Checklist

- [ ] VPC com DNS hostnames habilitado
- [ ] Subnets em pelo menos 2 AZs
- [ ] Security Groups seguem menor privilégio
- [ ] ALB com health check configurado
- [ ] Route 53 hosted zone + alias record para ALB
- [ ] VPC Endpoints para S3, DynamoDB, ECR, Logs
- [ ] SDK Go: DescribeVPCs, DescribeSubnets, DescribeSecurityGroupRules
- [ ] SDK Java: equivalente ao Go
- [ ] Teste de integração com LocalStack

---

## Exercícios Extras

1. **CloudFront Distribution** — Configure uma CloudFront distribution via CLI que aponta para o S3 bucket de receipts. Implemente presigned URLs para acesso temporário ao conteúdo.
2. **Network ACL Hardening** — Crie NACLs customizadas para cada tier, bloqueando tráfego entre tiers que não deveria existir (data ↛ public).
3. **Multi-AZ NAT Gateway** — Configure um NAT Gateway por AZ para alta disponibilidade. Compare custo vs. resiliência de NAT Gateway single vs. multi-AZ.
4. **VPC Flow Logs Analysis** — Habilite Flow Logs para CloudWatch. Escreva uma query CloudWatch Insights que identifica tráfego rejeitado por IP de origem.
5. **Service Discovery** — Configure AWS Cloud Map para service discovery interna. Registre serviços Go e Java e resolva via DNS privado.
