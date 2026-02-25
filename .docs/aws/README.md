# AWS — Guia de Boas Práticas & Arquitetura

> **Objetivo deste guia:** Servir como referência completa de **boas práticas AWS** por domínio (segurança, compute, networking, storage, databases, serverless, observabilidade, CI/CD, custos, governança e resiliência), otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
>
> **Público-alvo:** Engenheiros, arquitetos de soluções e times DevOps projetando e operando workloads na AWS.

---

## Quick Reference — Domínios

| Domínio | Foco Principal | Serviços-chave |
|---------|---------------|----------------|
| **Well-Architected** | 6 pilares de excelência operacional | Framework transversal |
| **Security** | Defense in depth, zero trust, compliance | IAM, KMS, GuardDuty, Macie, Config |
| **Compute** | Containers e orquestração | ECS, EKS, Fargate, App Runner |
| **Networking** | Conectividade, segmentação, performance | VPC, ALB/NLB, CloudFront, Route 53, Transit Gateway |
| **Storage** | Object, block, file storage | S3, EBS, EFS, FSx |
| **Databases** | Bancos managed, modelagem | RDS, Aurora, DynamoDB, ElastiCache, Neptune |
| **Serverless** | Event-driven, patterns | Lambda, API Gateway, Step Functions, EventBridge |
| **Observability** | Monitoramento, tracing, logs | CloudWatch, X-Ray, OpenTelemetry |
| **CI/CD** | Pipelines, deploy strategies | CodePipeline, CodeBuild, CodeDeploy, CDK |
| **Cost Optimization** | FinOps, right-sizing | Cost Explorer, Savings Plans, Compute Optimizer |
| **Governance** | Multi-account, compliance | Organizations, Control Tower, SCPs |
| **Reliability** | DR, backup, resiliência | Multi-AZ, Cross-Region, Backup, Route 53 |

---

## Documentos

### [Well-Architected Framework](well-architected-framework/well-architected-framework.md)

Os **6 pilares** do AWS Well-Architected Framework — Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization e Sustainability — com boas práticas, anti-patterns e checklists.

### [Security — Boas Práticas](security/security-best-practices.md)

Segurança defense in depth na AWS — **IAM**, criptografia, gerenciamento de secrets, network security, detecção de ameaças, compliance e responsabilidade compartilhada.

### [Security — Compliance Checklist](security/compliance-checklist.md)

Checklist de compliance para workloads regulados — LGPD, GDPR, HIPAA, PCI-DSS, SOC 2 — com serviços AWS e controles necessários.

### [Compute — Containers](compute/containers-best-practices.md)

Boas práticas para orquestração de containers na AWS — **ECS vs EKS vs App Runner**, Fargate, networking, segurança, CI/CD e observabilidade de containers.

### [Networking — Boas Práticas](networking/networking-best-practices.md)

Arquitetura de rede na AWS — **VPC design**, subnets, security groups, NACLs, load balancers, CloudFront, Route 53, Transit Gateway e conectividade híbrida.

### [Storage — Boas Práticas](storage/storage-best-practices.md)

Serviços de armazenamento na AWS — **S3** (tiers, lifecycle, replication), **EBS**, **EFS**, **FSx** — quando usar cada um, otimizações e anti-patterns.

### [Databases — Boas Práticas](databases/databases-best-practices.md)

Bancos de dados managed na AWS — **RDS, Aurora, DynamoDB, ElastiCache, Neptune** — escolha do engine, modelagem, performance tuning e operações.

### [Serverless — Boas Práticas](serverless/serverless-best-practices.md)

Boas práticas de desenvolvimento serverless — **Lambda**, API Gateway, Step Functions, EventBridge — cold starts, limites, segurança e observabilidade.

### [Serverless — Patterns](serverless/serverless-patterns.md)

Patterns de arquitetura serverless — API backend, event processing, data pipeline, scheduled jobs, fan-out, saga e circuit breaker com serviços serverless.

### [Observability — Boas Práticas](observability/observability-best-practices.md)

Observabilidade na AWS — **CloudWatch** (métricas, logs, alarms), **X-Ray** (tracing distribuído), **OpenTelemetry**, dashboards e alerting.

### [CI/CD — Boas Práticas](cicd/cicd-best-practices.md)

Pipelines de CI/CD na AWS — **CodePipeline, CodeBuild, CodeDeploy**, CDK Pipelines, deploy strategies (blue/green, canary, rolling) e GitOps.

### [Cost Optimization](cost-optimization/cost-optimization-guide.md)

FinOps e otimização de custos na AWS — **Cost Explorer, Savings Plans, Reserved Instances, Spot, Compute Optimizer**, tagging strategy e budgets.

### [Governance — Multi-Account Strategy](governance/multi-account-strategy.md)

Estratégia multi-account na AWS — **Organizations, Control Tower, SCPs**, landing zone, OU structure e guardrails.

### [Reliability — Disaster Recovery](reliability/disaster-recovery.md)

Disaster recovery na AWS — estratégias **Backup & Restore, Pilot Light, Warm Standby, Active-Active**, RPO/RTO, multi-AZ, cross-region e runbooks.

---

## Mapa de Decisão — Por onde começar

```
Novo projeto na AWS
│
├─ Arquitetura geral → Well-Architected Framework
│
├─ Segurança
│   ├─ IAM, encryption, secrets → Security Best Practices
│   └─ LGPD/GDPR/HIPAA? → Compliance Checklist
│
├─ Compute
│   ├─ Containers? → Containers Best Practices
│   └─ Event-driven / sem servidor? → Serverless Best Practices
│
├─ Dados
│   ├─ Banco relacional? → Databases Best Practices
│   ├─ Armazenamento de objetos/arquivos? → Storage Best Practices
│   └─ Data platform completa? → Veja o guia DATA-ENGINEERING
│
├─ Rede → Networking Best Practices
├─ Deploy → CI/CD Best Practices
├─ Monitoramento → Observability Best Practices
├─ Custos → Cost Optimization Guide
├─ Multi-account → Governance / Multi-Account Strategy
└─ DR / Resiliência → Disaster Recovery
```

---

## Referências

- AWS Well-Architected Framework — [docs.aws.amazon.com/wellarchitected](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- AWS Architecture Center — [aws.amazon.com/architecture](https://aws.amazon.com/architecture/)
- AWS Whitepapers — [aws.amazon.com/whitepapers](https://aws.amazon.com/whitepapers/)
- AWS Best Practices — [docs.aws.amazon.com/general/latest/gr](https://docs.aws.amazon.com/general/latest/gr/aws-general.pdf)

