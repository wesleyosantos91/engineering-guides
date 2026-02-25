# AWS Security — Compliance Checklist

Checklist prático de conformidade de segurança para workloads AWS categorizado por criticidade.

---

## Legenda

- **P0** — Crítico: deve ser implementado antes de ir para produção
- **P1** — Alto: implementar nas primeiras semanas de operação
- **P2** — Médio: implementar como melhoria contínua

---

## Account & Organization

| Prioridade | Item | Serviço AWS | Status |
|:----------:|------|-------------|:------:|
| P0 | MFA habilitado no root account (hardware token) | IAM | ☐ |
| P0 | Root account sem access keys | IAM | ☐ |
| P0 | AWS Organizations ativo com SCPs | Organizations | ☐ |
| P0 | Contas separadas por ambiente (dev/staging/prod) | Organizations | ☐ |
| P1 | IAM Identity Center (SSO) configurado | IAM Identity Center | ☐ |
| P1 | Conta de Log Archive centralizada | Organizations | ☐ |
| P1 | Conta de Security Tooling dedicada | Organizations | ☐ |
| P2 | Billing alerts configurados | Budgets | ☐ |
| P2 | Alternate contacts preenchidos | Account | ☐ |

---

## Identity & Access

| Prioridade | Item | Serviço AWS | Status |
|:----------:|------|-------------|:------:|
| P0 | IAM Roles para workloads (nunca access keys) | IAM | ☐ |
| P0 | Princípio do menor privilégio em todas as policies | IAM | ☐ |
| P0 | Sem políticas `*:*` (exceto em SCPs de deny) | IAM | ☐ |
| P0 | MFA para todos os usuários com acesso ao console | IAM | ☐ |
| P1 | Permission boundaries para roles criadas por automação | IAM | ☐ |
| P1 | IAM Access Analyzer habilitado | IAM | ☐ |
| P1 | Revisão trimestral de permissões IAM | IAM | ☐ |
| P1 | Credential Report gerado mensalmente | IAM | ☐ |
| P2 | Session tags para multi-tenancy | IAM/STS | ☐ |
| P2 | IAM policy conditions (IP, region, MFA) | IAM | ☐ |

---

## Data Protection

| Prioridade | Item | Serviço AWS | Status |
|:----------:|------|-------------|:------:|
| P0 | Criptografia em repouso em todos os datastores | KMS | ☐ |
| P0 | Criptografia em trânsito (TLS 1.2+) | ACM | ☐ |
| P0 | S3 public access block habilitado (account-level) | S3 | ☐ |
| P0 | Secrets no Secrets Manager (nunca em código) | Secrets Manager | ☐ |
| P1 | KMS CMK com rotação automática (90 dias) | KMS | ☐ |
| P1 | S3 bucket policies forçando SSL | S3 | ☐ |
| P1 | S3 versioning em buckets críticos | S3 | ☐ |
| P1 | RDS deletion protection habilitado | RDS | ☐ |
| P1 | Backup automatizado com retenção adequada | Backup | ☐ |
| P2 | Rotação automática de secrets | Secrets Manager | ☐ |
| P2 | Cross-region replication para DR | S3/RDS | ☐ |
| P2 | MFA Delete em buckets de compliance | S3 | ☐ |

---

## Network Security

| Prioridade | Item | Serviço AWS | Status |
|:----------:|------|-------------|:------:|
| P0 | Workloads em private subnets | VPC | ☐ |
| P0 | Security Groups com menor privilégio | VPC | ☐ |
| P0 | Databases sem acesso público | RDS/ElastiCache | ☐ |
| P1 | VPC Flow Logs habilitados | VPC | ☐ |
| P1 | VPC Endpoints para serviços AWS | VPC | ☐ |
| P1 | WAF no ALB/CloudFront/API Gateway | WAF | ☐ |
| P1 | NACLs como segunda camada de defesa | VPC | ☐ |
| P2 | Network Firewall para inspeção de tráfego | Network Firewall | ☐ |
| P2 | DNS Firewall habilitado | Route 53 | ☐ |
| P2 | AWS Shield Advanced (apps críticos) | Shield | ☐ |

---

## Detection & Monitoring

| Prioridade | Item | Serviço AWS | Status |
|:----------:|------|-------------|:------:|
| P0 | CloudTrail habilitado (multi-region) | CloudTrail | ☐ |
| P0 | GuardDuty habilitado em todas as contas | GuardDuty | ☐ |
| P1 | Security Hub habilitado com benchmarks | Security Hub | ☐ |
| P1 | AWS Config com rules de compliance | Config | ☐ |
| P1 | CloudWatch alarms para eventos de segurança | CloudWatch | ☐ |
| P1 | Alertas para mudanças IAM não autorizadas | EventBridge | ☐ |
| P1 | Log retention policy definida | CloudWatch | ☐ |
| P2 | CloudTrail Insights habilitado | CloudTrail | ☐ |
| P2 | Inspector para vulnerability scanning | Inspector | ☐ |
| P2 | Macie para descoberta de dados sensíveis | Macie | ☐ |

---

## Compute Security

| Prioridade | Item | Serviço AWS | Status |
|:----------:|------|-------------|:------:|
| P0 | Container images de base confiável (ECR) | ECR | ☐ |
| P0 | Scan de vulnerabilidades em imagens | ECR/Inspector | ☐ |
| P1 | ECS tasks com task role (não execution role) | ECS/IAM | ☐ |
| P1 | Read-only root filesystem em containers | ECS | ☐ |
| P1 | Non-root user em containers | ECS | ☐ |
| P1 | Lambda com menor privilégio | Lambda/IAM | ☐ |
| P2 | Runtime security (Falco/GuardDuty EKS) | EKS | ☐ |
| P2 | AMI hardening com CIS benchmarks | EC2 | ☐ |

---

## Application Security

| Prioridade | Item | Serviço AWS | Status |
|:----------:|------|-------------|:------:|
| P0 | Input validation em todas as APIs | Application | ☐ |
| P0 | CORS configurado corretamente | API GW/ALB | ☐ |
| P1 | Rate limiting habilitado | WAF/API GW | ☐ |
| P1 | OWASP Top 10 mitigado (WAF managed rules) | WAF | ☐ |
| P1 | Dependency scanning no CI/CD | CodeBuild | ☐ |
| P1 | SAST no pipeline | CodeBuild | ☐ |
| P2 | DAST em ambiente de staging | Inspector | ☐ |
| P2 | Penetration testing programado | - | ☐ |

---

## Incident Response

| Prioridade | Item | Serviço AWS | Status |
|:----------:|------|-------------|:------:|
| P1 | Runbooks de incident response documentados | SSM | ☐ |
| P1 | Alertas para findings de alta severidade | EventBridge/SNS | ☐ |
| P1 | Auto-remediação para cenários comuns | Lambda/SSM | ☐ |
| P1 | Plano de comunicação definido | - | ☐ |
| P2 | Game days de segurança trimestrais | - | ☐ |
| P2 | Forensics toolkit preparado | - | ☐ |
| P2 | Tabletop exercises semestrais | - | ☐ |

---

## Compliance Frameworks

### Mapeamento de Serviços por Framework

| Controle | CIS | SOC2 | PCI-DSS | Serviço AWS |
|----------|:---:|:----:|:-------:|-------------|
| Logging | 2.1 | CC7.2 | 10.1 | CloudTrail |
| Encryption at Rest | 2.8 | CC6.1 | 3.4 | KMS |
| Encryption in Transit | - | CC6.7 | 4.1 | ACM/TLS |
| Access Control | 1.x | CC6.1 | 7.1 | IAM |
| Network Segmentation | 4.x | CC6.6 | 1.3 | VPC/SG |
| Vulnerability Mgmt | - | CC7.1 | 6.1 | Inspector |
| Monitoring | 3.x | CC7.2 | 10.6 | CloudWatch/GuardDuty |
| Incident Response | - | CC7.3 | 12.10 | EventBridge/SSM |

---

## Referências

- [CIS AWS Foundations Benchmark v3.0](https://www.cisecurity.org/benchmark/amazon_web_services)
- [AWS Security Best Practices Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-security-best-practices/)
- [AWS Foundational Security Best Practices](https://docs.aws.amazon.com/securityhub/latest/userguide/fsbp-standard.html)
