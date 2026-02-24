# AWS Networking — Boas Práticas

Guia de boas práticas para design de rede na AWS: VPC, subnets, conectividade, DNS e CDN.

---

## VPC Design

### Arquitetura de Referência

```
┌─────────────────────────────── Region: sa-east-1 ──────────────────────────────┐
│                                                                                 │
│  ┌──────────────────────── VPC: 10.0.0.0/16 ────────────────────────┐          │
│  │                                                                    │          │
│  │  AZ-a (sa-east-1a)        AZ-b (sa-east-1b)        AZ-c          │          │
│  │  ┌──────────────┐        ┌──────────────┐        ┌──────────┐    │          │
│  │  │ Public        │        │ Public        │        │ Public   │    │          │
│  │  │ 10.0.100.0/24│        │ 10.0.101.0/24│        │10.0.102  │    │          │
│  │  │ NAT GW, ALB  │        │ NAT GW, ALB  │        │ /24      │    │          │
│  │  └──────────────┘        └──────────────┘        └──────────┘    │          │
│  │  ┌──────────────┐        ┌──────────────┐        ┌──────────┐    │          │
│  │  │ Private       │        │ Private       │        │ Private  │    │          │
│  │  │ 10.0.0.0/24  │        │ 10.0.1.0/24  │        │10.0.2    │    │          │
│  │  │ ECS, Lambda  │        │ ECS, Lambda  │        │ /24      │    │          │
│  │  └──────────────┘        └──────────────┘        └──────────┘    │          │
│  │  ┌──────────────┐        ┌──────────────┐        ┌──────────┐    │          │
│  │  │ Data          │        │ Data          │        │ Data     │    │          │
│  │  │ 10.0.200.0/24│        │ 10.0.201.0/24│        │10.0.202  │    │          │
│  │  │ RDS, Redis   │        │ RDS, Redis   │        │ /24      │    │          │
│  │  └──────────────┘        └──────────────┘        └──────────┘    │          │
│  │                                                                    │          │
│  │  VPC Endpoints: S3, DynamoDB, ECR, Logs, Secrets Manager, STS    │          │
│  └────────────────────────────────────────────────────────────────────┘          │
│                                                                                 │
│  Internet Gateway ←→ Public Subnets                                            │
│  NAT Gateways (1 per AZ) ←→ Private Subnets                                  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### CIDR Planning

```
Regras para planejamento de CIDR:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. VPC CIDR: /16 (65.536 IPs) — espaço suficiente para crescer
2. Subnet CIDR: /24 (251 IPs usáveis) para a maioria dos casos
3. NÃO sobrepor CIDRs entre VPCs (importante para peering/TGW)
4. Reservar espaço para expansão futura
5. Considerar RFC 1918: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16

Exemplo de alocação por ambiente:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Production: 10.0.0.0/16
  Staging:    10.1.0.0/16
  Development:10.2.0.0/16
  Shared:     10.10.0.0/16

Alocação de subnets dentro da VPC:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Public:  10.0.100.0/24, 10.0.101.0/24, 10.0.102.0/24
  Private: 10.0.0.0/24,   10.0.1.0/24,   10.0.2.0/24
  Data:    10.0.200.0/24, 10.0.201.0/24, 10.0.202.0/24
```

### VPC Terraform Module

```hcl
# VPC completa com 3 tiers de subnets
module "vpc" {
  source = "./modules/vpc"

  project     = var.project
  environment = var.environment
  
  vpc_cidr             = "10.0.0.0/16"
  availability_zones   = ["sa-east-1a", "sa-east-1b", "sa-east-1c"]
  
  public_subnet_cidrs  = ["10.0.100.0/24", "10.0.101.0/24", "10.0.102.0/24"]
  private_subnet_cidrs = ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
  data_subnet_cidrs    = ["10.0.200.0/24", "10.0.201.0/24", "10.0.202.0/24"]
  
  enable_nat_gateway   = true
  single_nat_gateway   = var.environment != "prod"  # 1 NAT para dev, 1 por AZ para prod
  
  enable_vpn_gateway   = false
  enable_flow_logs     = true
  
  flow_log_retention_days = var.environment == "prod" ? 90 : 14
  
  vpc_endpoints = {
    s3              = { type = "Gateway" }
    dynamodb        = { type = "Gateway" }
    ecr_api         = { type = "Interface", service = "ecr.api" }
    ecr_dkr         = { type = "Interface", service = "ecr.dkr" }
    ecs             = { type = "Interface", service = "ecs" }
    logs            = { type = "Interface", service = "logs" }
    secretsmanager  = { type = "Interface", service = "secretsmanager" }
    ssm             = { type = "Interface", service = "ssm" }
    sts             = { type = "Interface", service = "sts" }
  }
  
  tags = var.tags
}
```

### Flow Logs

```hcl
resource "aws_flow_log" "vpc" {
  vpc_id                   = aws_vpc.main.id
  traffic_type             = "ALL"
  log_destination_type     = "cloud-watch-logs"
  log_destination          = aws_cloudwatch_log_group.flow_logs.arn
  iam_role_arn             = aws_iam_role.flow_logs.arn
  max_aggregation_interval = 60

  tags = { Name = "${var.project}-vpc-flow-logs" }
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/flow-logs/${var.project}"
  retention_in_days = var.flow_log_retention_days
  kms_key_id        = var.kms_key_arn
}

# Metric filter para tráfego rejeitado
resource "aws_cloudwatch_log_metric_filter" "rejected_traffic" {
  name           = "${var.project}-rejected-traffic"
  log_group_name = aws_cloudwatch_log_group.flow_logs.name
  pattern        = "[version, account_id, interface_id, srcaddr, dstaddr, srcport, dstport, protocol, packets, bytes, start, end, action=\"REJECT\", log_status]"

  metric_transformation {
    name          = "RejectedTrafficCount"
    namespace     = "${var.project}/VPC"
    value         = "1"
    default_value = "0"
  }
}

resource "aws_cloudwatch_metric_alarm" "high_rejected_traffic" {
  alarm_name          = "${var.project}-high-rejected-traffic"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 5
  metric_name         = "RejectedTrafficCount"
  namespace           = "${var.project}/VPC"
  period              = 60
  statistic           = "Sum"
  threshold           = 1000
  alarm_description   = "High volume of rejected VPC traffic"
  alarm_actions       = [var.sns_alerts_arn]
}
```

---

## Load Balancing

### ALB (Application Load Balancer)

```hcl
resource "aws_lb" "main" {
  name               = "${var.project}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = var.environment == "prod"
  enable_http2               = true
  drop_invalid_header_fields = true  # Segurança: rejeita headers inválidos
  
  idle_timeout = 60

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = var.project
    enabled = true
  }

  tags = var.tags
}

# Redirect HTTP → HTTPS
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# HTTPS Listener com routing rules
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "application/json"
      message_body = "{\"error\":\"Not Found\"}"
      status_code  = "404"
    }
  }
}

# Path-based routing
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }

  condition {
    http_header {
      http_header_name = "X-Custom-Header"
      values           = [var.custom_header_value]
    }
  }
}
```

### NLB (Network Load Balancer) — Quando Usar

```
Usar ALB quando:
  • HTTP/HTTPS routing necessário
  • Path-based ou host-based routing
  • WebSocket support
  • WAF integration necessária

Usar NLB quando:
  • Ultra-low latency necessária (< 1ms)
  • TCP/UDP/TLS protocol level
  • Static IP necessário
  • Milhões de requests por segundo
  • Preserve source IP sem X-Forwarded-For
  • gRPC load balancing
```

---

## DNS — Route 53

### Configuração Base

```hcl
# Hosted Zone
resource "aws_route53_zone" "main" {
  name = var.domain_name
}

# A Record com Alias para ALB
resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

# Health Check
resource "aws_route53_health_check" "api" {
  fqdn              = "api.${var.domain_name}"
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 30
  
  regions = ["sa-east-1", "us-east-1", "eu-west-1"]

  tags = {
    Name = "${var.project}-api-health-check"
  }
}
```

### Routing Policies

```
┌─────────────────────────────────────────────────────────────────┐
│              Route 53 Routing Policies                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Simple              → 1 record, 1 value                       │
│  Weighted            → Distribuir tráfego (A/B, canary)        │
│  Latency-based       → Menor latência para o usuário          │
│  Geolocation         → Baseado na localização do usuário      │
│  Geoproximity        → Com bias para atrair/repelir tráfego   │
│  Failover            → Active-passive DR                        │
│  Multivalue Answer   → Até 8 healthy records                  │
│  IP-based            → Roteamento por CIDR do client          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```hcl
# Failover routing para DR
resource "aws_route53_record" "primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  set_identifier = "primary"
  
  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.primary.id
}

resource "aws_route53_record" "secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  set_identifier = "secondary"
  
  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

---

## CDN — CloudFront

### Distribuição Completa

```hcl
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "${var.project} - ${var.environment}"
  default_root_object = "index.html"
  price_class         = "PriceClass_200"  # Inclui América do Sul
  web_acl_id          = aws_wafv2_web_acl.cloudfront.arn
  
  aliases = ["app.${var.domain_name}"]

  # Origin: S3 (static assets)
  origin {
    domain_name              = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id                = "s3-static"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  # Origin: ALB (API)
  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "alb-api"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }

    custom_header {
      name  = "X-Custom-Header"
      value = var.cloudfront_custom_header  # Validar no ALB
    }
  }

  # Behavior: API (no cache, proxy to ALB)
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "alb-api"

    cache_policy_id          = aws_cloudfront_cache_policy.api_no_cache.id
    origin_request_policy_id = aws_cloudfront_origin_request_policy.api.id

    viewer_protocol_policy = "redirect-to-https"
    compress               = true
  }

  # Behavior default: Static assets (cached)
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-static"

    cache_policy_id          = aws_cloudfront_cache_policy.static.id
    origin_request_policy_id = aws_cloudfront_origin_request_policy.cors.id
    
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security.id

    viewer_protocol_policy = "redirect-to-https"
    compress               = true
  }

  # SPA: redirecionar 404 para index.html
  custom_error_response {
    error_code         = 404
    response_code      = 200
    response_page_path = "/index.html"
  }

  custom_error_response {
    error_code         = 403
    response_code      = 200
    response_page_path = "/index.html"
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cloudfront.arn  # us-east-1
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  logging_config {
    include_cookies = false
    bucket          = aws_s3_bucket.cf_logs.bucket_domain_name
    prefix          = "cloudfront/"
  }
}

# Security Headers
resource "aws_cloudfront_response_headers_policy" "security" {
  name = "${var.project}-security-headers"

  security_headers_config {
    content_type_options {
      override = true
    }
    frame_options {
      frame_option = "DENY"
      override     = true
    }
    strict_transport_security {
      access_control_max_age_sec = 31536000
      include_subdomains         = true
      preload                    = true
      override                   = true
    }
    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }
    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }
    content_security_policy {
      content_security_policy = "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"
      override                = true
    }
  }
}
```

---

## Conectividade Multi-Account

### Transit Gateway

```hcl
# Transit Gateway para conectar múltiplas VPCs
resource "aws_ec2_transit_gateway" "main" {
  description                     = "${var.project} Transit Gateway"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  dns_support                     = "enable"
  vpn_ecmp_support                = "enable"

  tags = { Name = "${var.project}-tgw" }
}

# Attachment para cada VPC
resource "aws_ec2_transit_gateway_vpc_attachment" "production" {
  subnet_ids         = module.vpc_prod.private_subnet_ids
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = module.vpc_prod.vpc_id

  transit_gateway_default_route_table_association = false
  transit_gateway_default_route_table_propagation = false

  tags = { Name = "${var.project}-prod-attachment" }
}

# Route tables para segmentação
resource "aws_ec2_transit_gateway_route_table" "production" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "${var.project}-prod-rt" }
}

resource "aws_ec2_transit_gateway_route_table" "shared" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "${var.project}-shared-rt" }
}

# Prod pode alcançar Shared, mas não Dev
resource "aws_ec2_transit_gateway_route_table_association" "prod" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.production.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.production.id
}

resource "aws_ec2_transit_gateway_route_table_propagation" "prod_to_shared" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.shared.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.production.id
}
```

---

## Networking Decision Guide

| Cenário | Solução |
|---------|---------|
| Conectar 2 VPCs | VPC Peering |
| Conectar 3+ VPCs | Transit Gateway |
| Acessar serviço AWS sem internet | VPC Endpoint (Interface/Gateway) |
| Expor serviço interno para outra conta | PrivateLink |
| CDN para conteúdo estático | CloudFront + S3 |
| CDN para API | CloudFront + ALB como origin |
| DNS público | Route 53 Public Hosted Zone |
| DNS interno (VPC) | Route 53 Private Hosted Zone |
| Load balancing HTTP/S | ALB |
| Load balancing TCP/UDP, ultra-low latency | NLB |
| Proteção DDoS básica | AWS Shield Standard (grátis) |
| Proteção DDoS avançada | AWS Shield Advanced |
| Firewall de aplicação | WAF |
| Filtragem de tráfego de rede | Network Firewall |
| VPN site-to-site | AWS Site-to-Site VPN |
| Conexão dedicada | AWS Direct Connect |

---

## Referências

- [AWS VPC Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-best-practices.html)
- [AWS Networking & Content Delivery](https://aws.amazon.com/products/networking/)
- [CloudFront Best Practices](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/best-practices.html)
