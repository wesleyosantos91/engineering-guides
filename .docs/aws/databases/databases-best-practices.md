# AWS Databases — Boas Práticas

Guia de boas práticas para bancos de dados gerenciados na AWS: RDS, Aurora, DynamoDB, ElastiCache e DocumentDB.

---

## Decisão de Database

```
┌──────────────────────────────────────────────────────────────────┐
│              Árvore de Decisão: Database                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Dados relacionais com transações ACID?                         │
│  ├── SIM → Escala alta (>100K TPS) ou global?                  │
│  │    ├── SIM → Amazon Aurora (MySQL/PostgreSQL)                │
│  │    │         5x throughput MySQL, 3x PostgreSQL              │
│  │    └── NÃO → Amazon RDS                                     │
│  │              (PostgreSQL recomendado como padrão)            │
│  │                                                               │
│  └── NÃO → Dados key-value ou documents?                        │
│       ├── SIM → Latência < 10ms, escala ilimitada?              │
│       │    ├── SIM → DynamoDB                                   │
│       │    │         (Serverless, single-digit ms)              │
│       │    └── NÃO → Precisa de queries ricas tipo MongoDB?     │
│       │         ├── SIM → DocumentDB                            │
│       │         └── NÃO → DynamoDB                              │
│       │                                                          │
│       └── NÃO → Cache in-memory?                                │
│            ├── SIM → Estruturas avançadas (sorted sets, etc)?   │
│            │    ├── SIM → ElastiCache Redis                     │
│            │    └── NÃO → ElastiCache Memcached                 │
│            │              (mais simples, multi-thread)           │
│            │                                                     │
│            └── NÃO → Full-text search?                          │
│                 ├── SIM → OpenSearch Service                    │
│                 └── NÃO → Time-series → Amazon Timestream       │
│                           Graph → Amazon Neptune                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Amazon RDS / Aurora

### Configuração de Produção

```hcl
# RDS PostgreSQL — Configuração de produção
resource "aws_db_instance" "main" {
  identifier = "${var.project}-${var.environment}"

  # Engine
  engine         = "postgres"
  engine_version = "16.4"
  instance_class = "db.r6g.large"  # Graviton para melhor preço/performance

  # Storage
  allocated_storage     = 100
  max_allocated_storage = 500  # Auto-scaling de storage
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn

  # HA
  multi_az = true

  # Network
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
  publicly_accessible    = false

  # Credentials
  manage_master_user_password = true  # AWS gerencia a senha no Secrets Manager
  username                    = "dbadmin"

  # Backup
  backup_retention_period = 14
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"
  copy_tags_to_snapshot   = true
  
  # Protection
  deletion_protection       = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "${var.project}-${var.environment}-final"

  # Monitoring
  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  performance_insights_kms_key_id       = aws_kms_key.rds.arn
  monitoring_interval                   = 60
  monitoring_role_arn                   = aws_iam_role.rds_monitoring.arn
  enabled_cloudwatch_logs_exports       = ["postgresql", "upgrade"]

  # Parameters
  parameter_group_name = aws_db_parameter_group.main.name

  tags = var.tags
}

# Parameter Group otimizado
resource "aws_db_parameter_group" "main" {
  name_prefix = "${var.project}-pg16-"
  family      = "postgres16"

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements,auto_explain"
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"  # Log queries > 1s
  }

  parameter {
    name  = "log_connections"
    value = "1"
  }

  parameter {
    name  = "log_disconnections"
    value = "1"
  }

  parameter {
    name  = "log_lock_waits"
    value = "1"
  }

  parameter {
    name         = "rds.force_ssl"
    value        = "1"
    apply_method = "pending-reboot"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

### Aurora Serverless v2

```hcl
# Aurora Serverless v2 — escala automática
resource "aws_rds_cluster" "aurora" {
  cluster_identifier = "${var.project}-aurora"
  engine             = "aurora-postgresql"
  engine_mode        = "provisioned"
  engine_version     = "16.4"
  
  database_name   = var.db_name
  master_username = "dbadmin"
  manage_master_user_password = true

  # Serverless v2 scaling
  serverlessv2_scaling_configuration {
    min_capacity = 0.5   # Scale to near-zero
    max_capacity = 16.0  # Máximo ACUs
  }

  # Network & Security
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
  
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  # Backup
  backup_retention_period = 14
  preferred_backup_window = "03:00-04:00"
  
  deletion_protection      = true
  skip_final_snapshot      = false
  final_snapshot_identifier = "${var.project}-aurora-final"

  enabled_cloudwatch_logs_exports = ["postgresql"]
}

resource "aws_rds_cluster_instance" "main" {
  count = 2  # Writer + Reader

  identifier         = "${var.project}-aurora-${count.index}"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.serverless"  # Serverless v2
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version

  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn
}
```

### Read Replica Pattern

```hcl
# Read replica para offload de leitura
resource "aws_db_instance" "read_replica" {
  identifier          = "${var.project}-${var.environment}-replica"
  replicate_source_db = aws_db_instance.main.identifier

  instance_class = "db.r6g.large"
  
  # Replica pode estar em outra AZ
  multi_az = false

  # Performance
  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn

  tags = merge(var.tags, { Role = "read-replica" })
}
```

---

## Amazon DynamoDB

### Modelagem — Single Table Design

```
┌────────────────────────────────────────────────────────────────┐
│           DynamoDB — Single Table Design                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Princípios:                                                   │
│  • Desnormalizar dados (diferente de relacional)               │
│  • Pre-compute access patterns                                 │
│  • Uma tabela para múltiplas entidades                        │
│  • Partition Key + Sort Key = acesso eficiente                 │
│                                                                │
│  Exemplo: E-commerce (Orders + Products + Customers)           │
│                                                                │
│  PK              | SK                | Attributes              │
│  ─────────────── | ───────────────── | ─────────────           │
│  CUSTOMER#123    | PROFILE           | name, email, ...        │
│  CUSTOMER#123    | ORDER#2024-001    | total, status, ...      │
│  CUSTOMER#123    | ORDER#2024-002    | total, status, ...      │
│  ORDER#2024-001  | ITEM#prod-001     | qty, price, ...         │
│  ORDER#2024-001  | ITEM#prod-002     | qty, price, ...         │
│  PRODUCT#prod-001| INFO              | name, price, stock      │
│                                                                │
│  GSI1: (GSI1PK, GSI1SK) para access patterns invertidos       │
│  GSI1PK = ORDER#2024-001, GSI1SK = CUSTOMER#123               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Configuração DynamoDB

```hcl
# DynamoDB com PAY_PER_REQUEST (recomendado para início)
resource "aws_dynamodb_table" "main" {
  name         = "${var.project}-${var.table_name}"
  billing_mode = "PAY_PER_REQUEST"  # On-demand: pague por request
  hash_key     = "PK"
  range_key    = "SK"

  # Keys
  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  attribute {
    name = "GSI1PK"
    type = "S"
  }

  attribute {
    name = "GSI1SK"
    type = "S"
  }

  # Global Secondary Index
  global_secondary_index {
    name            = "GSI1"
    hash_key        = "GSI1PK"
    range_key       = "GSI1SK"
    projection_type = "ALL"
  }

  # Encryption
  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.dynamodb.arn
  }

  # Point-in-time recovery
  point_in_time_recovery {
    enabled = true
  }

  # TTL para expiração automática
  ttl {
    attribute_name = "expiresAt"
    enabled        = true
  }

  # Streams para CDC
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  # Table class — STANDARD_INFREQUENT_ACCESS para dados frios
  table_class = var.environment == "prod" ? "STANDARD" : "STANDARD"

  # Deletion protection
  deletion_protection_enabled = var.environment == "prod"

  tags = var.tags
}

# Backup automático
resource "aws_dynamodb_table_replica" "dr" {
  count = var.environment == "prod" ? 1 : 0
  
  global_table_arn = aws_dynamodb_table.main.arn
  region_name      = "us-east-1"  # DR region
}
```

### DynamoDB Boas Práticas

```java
// Exemplo de acesso otimizado com AWS SDK v2

@Repository
public class OrderRepository {
    
    private final DynamoDbEnhancedClient enhancedClient;
    private final DynamoDbTable<OrderEntity> table;

    // Query por partition key — mais eficiente
    public List<OrderEntity> findOrdersByCustomer(String customerId) {
        var key = Key.builder()
                .partitionValue("CUSTOMER#" + customerId)
                .sortValue(QueryConditional.sortBeginsWith(
                    Key.builder().sortValue("ORDER#").build()))
                .build();

        return table.query(QueryEnhancedRequest.builder()
                .queryConditional(QueryConditional.sortBeginsWith(key))
                .limit(50)  // Sempre paginar
                .build())
                .items()
                .stream()
                .toList();
    }

    // Batch write — até 25 items por batch
    public void batchSave(List<OrderEntity> orders) {
        var writeBatch = WriteBatch.builder(OrderEntity.class)
                .mappedTableResource(table)
                .addPutItem(builder -> orders.forEach(builder::addPutItem))
                .build();

        enhancedClient.batchWriteItem(BatchWriteItemEnhancedRequest.builder()
                .addWriteBatch(writeBatch)
                .build());
    }

    // Conditional write — idempotência
    public void createOrder(OrderEntity order) {
        try {
            table.putItem(PutItemEnhancedRequest.builder(OrderEntity.class)
                    .item(order)
                    .conditionExpression(Expression.builder()
                            .expression("attribute_not_exists(PK)")
                            .build())
                    .build());
        } catch (ConditionalCheckFailedException e) {
            throw new DuplicateOrderException(order.getOrderId());
        }
    }
}
```

### DynamoDB Anti-Patterns

| Anti-Pattern | Problema | Solução |
|-------------|----------|---------|
| Scan operations | Full table scan, lento e caro | Usar Query com PK/SK |
| Hot partition | 1 partition recebe todo tráfego | Adicionar sufixo aleatório ao PK |
| Large items | > 400KB por item | Armazenar blob no S3, referência no DDB |
| Relational modeling | Normalizar como SQL | Single Table Design |
| No pagination | Retornar milhares de items | Usar Limit + LastEvaluatedKey |
| Filter sem query | FilterExpression sem KeyCondition | Redesenhar partition key |

---

## Amazon ElastiCache

### Redis — Configuração de Produção

```hcl
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "${var.project}-redis"
  description          = "Redis for ${var.project}"
  
  # Engine
  engine               = "redis"
  engine_version       = "7.1"
  node_type            = "cache.r7g.large"  # Graviton
  num_cache_clusters   = 2  # 1 primary + 1 replica
  port                 = 6379
  parameter_group_name = aws_elasticache_parameter_group.redis.name

  # HA
  automatic_failover_enabled = true
  multi_az_enabled           = true

  # Security
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token  # Usar Secrets Manager
  
  # Network
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  # Maintenance
  snapshot_retention_limit = 7
  snapshot_window          = "03:00-05:00"
  maintenance_window       = "sun:05:00-sun:07:00"
  
  # Notifications
  notification_topic_arn = var.sns_alerts_arn

  # Auto minor version upgrade
  auto_minor_version_upgrade = true

  tags = var.tags
}

resource "aws_elasticache_parameter_group" "redis" {
  name   = "${var.project}-redis7"
  family = "redis7"

  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"  # LRU eviction para cache
  }

  parameter {
    name  = "notify-keyspace-events"
    value = "Ex"  # Notificar expiração de keys
  }
}
```

### Caching Patterns

```
┌──────────────────────────────────────────────────────────────┐
│                    Caching Patterns                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Cache-Aside (Lazy Loading)                                  │
│  ┌─────┐    miss    ┌─────┐    read    ┌──────┐            │
│  │ App │ ─────────→ │Cache│ ─────────→ │  DB  │            │
│  │     │ ←───────── │     │ ←───────── │      │            │
│  └─────┘   hit/set  └─────┘   result   └──────┘            │
│  • Prós: Dados em cache são sempre solicitados              │
│  • Contras: Cache miss = 3 trips, dados podem ficar stale  │
│                                                              │
│  Write-Through                                               │
│  ┌─────┐   write    ┌─────┐   write    ┌──────┐            │
│  │ App │ ─────────→ │Cache│ ─────────→ │  DB  │            │
│  └─────┘            └─────┘            └──────┘            │
│  • Prós: Cache sempre atualizado                            │
│  • Contras: Write latency maior, dados nunca lidos no cache │
│                                                              │
│  Write-Behind (Write-Back)                                   │
│  ┌─────┐   write    ┌─────┐   async    ┌──────┐            │
│  │ App │ ─────────→ │Cache│ ─ ─ ─ ─ → │  DB  │            │
│  └─────┘            └─────┘            └──────┘            │
│  • Prós: Write muito rápido                                 │
│  • Contras: Risco de perda de dados se cache falhar         │
│                                                              │
│  Recomendação:                                               │
│  • Cache-Aside para leitura (padrão)                        │
│  • TTL adequado (30s-5min para dados frequentes)            │
│  • Invalidação explícita em writes                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Database Monitoring

### Alarmes Essenciais

```hcl
# RDS CPU
resource "aws_cloudwatch_metric_alarm" "rds_cpu" {
  alarm_name          = "${var.project}-rds-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_actions       = [var.sns_alerts_arn]

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }
}

# RDS Free Storage
resource "aws_cloudwatch_metric_alarm" "rds_storage" {
  alarm_name          = "${var.project}-rds-storage"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FreeStorageSpace"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 10737418240  # 10 GB
  alarm_actions       = [var.sns_alerts_arn]

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }
}

# RDS Connections
resource "aws_cloudwatch_metric_alarm" "rds_connections" {
  alarm_name          = "${var.project}-rds-connections"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = var.max_connections * 0.8  # 80% do máximo
  alarm_actions       = [var.sns_alerts_arn]

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }
}

# DynamoDB Throttled Requests
resource "aws_cloudwatch_metric_alarm" "dynamodb_throttle" {
  alarm_name          = "${var.project}-dynamodb-throttle"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ThrottledRequests"
  namespace           = "AWS/DynamoDB"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_actions       = [var.sns_alerts_arn]

  dimensions = {
    TableName = aws_dynamodb_table.main.name
  }
}

# ElastiCache Memory
resource "aws_cloudwatch_metric_alarm" "redis_memory" {
  alarm_name          = "${var.project}-redis-memory"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseMemoryUsagePercentage"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_actions       = [var.sns_alerts_arn]

  dimensions = {
    CacheClusterId = aws_elasticache_replication_group.redis.id
  }
}
```

---

## Database Checklist

### Geral

- [ ] Multi-AZ habilitado para produção
- [ ] Backup automatizado com retenção adequada (≥ 7 dias)
- [ ] Criptografia em repouso (KMS CMK)
- [ ] Criptografia em trânsito (SSL/TLS)
- [ ] Sem acesso público
- [ ] Security Groups restritivos (apenas da aplicação)
- [ ] Parameter group customizado e otimizado
- [ ] Monitoring habilitado (Performance Insights, Enhanced Monitoring)
- [ ] Alarmes para CPU, Storage, Connections
- [ ] Deletion protection habilitado

### RDS/Aurora Específico

- [ ] Storage auto-scaling habilitado
- [ ] Logs exportados para CloudWatch
- [ ] Read replicas para offload de leitura (se necessário)
- [ ] Connection pooling (RDS Proxy ou application-level)
- [ ] Password gerenciada pelo Secrets Manager com rotação

### DynamoDB Específico

- [ ] Point-in-time recovery habilitado
- [ ] Single Table Design com access patterns definidos
- [ ] GSIs projetados para queries específicas
- [ ] TTL para dados temporários
- [ ] Streams habilitados para CDC (se necessário)
- [ ] Auto-scaling ou On-demand billing

---

## Referências

- [Amazon RDS Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_BestPractices.html)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [ElastiCache Best Practices](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/best-practices.html)
- [Aurora Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.BestPractices.html)
