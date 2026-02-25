# AWS Storage — Boas Práticas

Guia de serviços de storage na AWS: S3 avançado, EFS, FSx e estratégias de lifecycle.

---

## S3 — Padrões Avançados

### Bucket com Segurança Completa

```hcl
resource "aws_s3_bucket" "main" {
  bucket = "${var.project}-${var.environment}-data"

  tags = var.tags
}

# Versionamento — proteção contra deleção acidental
resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Criptografia — SSE-KMS com chave gerenciada
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true  # Reduz custo de KMS em ~99%
  }
}

# Block Public Access — SEMPRE habilitado
resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle Rules — otimização de custo
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  # Dados frequentemente acessados → Infrequent → Glacier
  rule {
    id     = "archive-old-data"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"  # Instant Retrieval (ms)
    }

    transition {
      days          = 180
      storage_class = "GLACIER_FLEXIBLE_RETRIEVAL"  # 1-5h
    }

    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"  # 12h, mais barato
    }
  }

  # Limpar versões antigas
  rule {
    id     = "cleanup-old-versions"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      noncurrent_days = 90
    }

    # Limpar incomplete multipart uploads
    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }

  # Expirar logs após 90 dias
  rule {
    id     = "expire-logs"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    expiration {
      days = 90
    }
  }
}
```

### S3 Storage Classes — Decisão

```
┌──────────────────────────────────────────────────────────────────┐
│            Qual Storage Class usar?                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Acesso frequente?                                              │
│  ├── SIM → S3 Standard                                          │
│  └── NÃO → Padrão de acesso imprevisível?                      │
│       ├── SIM → S3 Intelligent-Tiering                          │
│       └── NÃO → Acesso mensal ou menos?                        │
│            ├── SIM → Precisa de acesso imediato?                │
│            │    ├── SIM → S3 Standard-IA                        │
│            │    └── NÃO → Glacier Instant Retrieval             │
│            └── NÃO → Acesso anual ou menos?                    │
│                 ├── SIM → Glacier Flexible Retrieval            │
│                 └── NÃO → Glacier Deep Archive                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

| Storage Class | Latência | Min. Storage | Custo/GB (sa-east-1) | Use Case |
|--------------|----------|-------------|---------------------|----------|
| **Standard** | ms | — | $0.0264 | Dados ativos |
| **Intelligent-Tiering** | ms | — | $0.0264 + monitoring | Acesso imprevisível |
| **Standard-IA** | ms | 30 dias | $0.015 | Backups recentes |
| **One Zone-IA** | ms | 30 dias | $0.012 | Dados reproduzíveis |
| **Glacier IR** | ms | 90 dias | $0.005 | Arquivos médicos, compliance |
| **Glacier FR** | min-horas | 90 dias | $0.0045 | Logs antigos |
| **Deep Archive** | 12h | 180 dias | $0.002 | Compliance de longo prazo |

### S3 Access Points

```hcl
# Access Points — controle de acesso granular para múltiplos consumidores
resource "aws_s3_access_point" "analytics" {
  bucket = aws_s3_bucket.data_lake.id
  name   = "${var.project}-analytics-ap"

  vpc_configuration {
    vpc_id = var.analytics_vpc_id
  }

  public_access_block_configuration {
    block_public_acls       = true
    block_public_policy     = true
    ignore_public_acls      = true
    restrict_public_buckets = true
  }
}

resource "aws_s3_access_point_policy" "analytics" {
  access_point_arn = aws_s3_access_point.analytics.arn

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = var.analytics_role_arn
      }
      Action = ["s3:GetObject", "s3:ListBucket"]
      Resource = [
        "${aws_s3_access_point.analytics.arn}",
        "${aws_s3_access_point.analytics.arn}/object/processed/*"
      ]
    }]
  })
}
```

### S3 Event Notifications

```hcl
# Notificar Lambda quando arquivos são carregados
resource "aws_s3_bucket_notification" "main" {
  bucket = aws_s3_bucket.main.id

  # Processar uploads de imagem
  lambda_function {
    lambda_function_arn = aws_lambda_function.image_processor.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "uploads/images/"
    filter_suffix       = ".jpg"
  }

  # Indexar documentos
  lambda_function {
    lambda_function_arn = aws_lambda_function.document_indexer.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "uploads/documents/"
  }

  # Enviar para SQS (processamento batch)
  queue {
    queue_arn     = aws_sqs_queue.data_processing.arn
    events        = ["s3:ObjectCreated:*"]
    filter_prefix = "data/raw/"
  }
}

resource "aws_lambda_permission" "s3_invoke" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.image_processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.main.arn
}
```

### S3 Object Lock (WORM)

```hcl
# Object Lock — Write Once Read Many (compliance, regulatório)
resource "aws_s3_bucket" "compliance" {
  bucket              = "${var.project}-compliance-records"
  object_lock_enabled = true  # Deve ser habilitado na criação
}

resource "aws_s3_bucket_object_lock_configuration" "compliance" {
  bucket = aws_s3_bucket.compliance.id

  rule {
    default_retention {
      mode = "COMPLIANCE"  # Nem o root pode deletar
      days = 2555          # 7 anos
    }
  }
}
```

### Presigned URLs (Upload/Download Seguro)

```java
// Gerar presigned URL para upload direto do cliente
@Service
public class S3PresignedUrlService {

    private final S3Presigner presigner;
    private final String bucketName;

    public PresignedUrlResponse generateUploadUrl(String key, String contentType) {
        PutObjectRequest objectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key("uploads/" + key)
                .contentType(contentType)
                .serverSideEncryption(ServerSideEncryption.AWS_KMS)
                .build();

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(15))  // Expira em 15 min
                .putObjectRequest(objectRequest)
                .build();

        PresignedPutObjectRequest presigned = presigner.presignPutObject(presignRequest);

        return new PresignedUrlResponse(
            presigned.url().toString(),
            presigned.expiration()
        );
    }

    public PresignedUrlResponse generateDownloadUrl(String key) {
        GetObjectRequest getRequest = GetObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(60))
                .getObjectRequest(getRequest)
                .build();

        PresignedGetObjectRequest presigned = presigner.presignGetObject(presignRequest);

        return new PresignedUrlResponse(
            presigned.url().toString(),
            presigned.expiration()
        );
    }
}
```

### Multipart Upload (Arquivos Grandes)

```java
// Upload de arquivos grandes (> 100MB) com multipart
@Service
public class S3MultipartUploadService {

    private final S3Client s3Client;
    private static final long PART_SIZE = 50 * 1024 * 1024; // 50MB por parte

    public String uploadLargeFile(String bucket, String key, Path filePath) {
        // 1. Iniciar multipart upload
        CreateMultipartUploadResponse createResponse = s3Client.createMultipartUpload(
            CreateMultipartUploadRequest.builder()
                .bucket(bucket)
                .key(key)
                .serverSideEncryption(ServerSideEncryption.AWS_KMS)
                .build()
        );
        String uploadId = createResponse.uploadId();

        try {
            long fileSize = Files.size(filePath);
            int partCount = (int) Math.ceil((double) fileSize / PART_SIZE);
            List<CompletedPart> completedParts = new ArrayList<>();

            // 2. Upload de cada parte
            for (int i = 0; i < partCount; i++) {
                long start = i * PART_SIZE;
                long length = Math.min(PART_SIZE, fileSize - start);

                UploadPartResponse uploadResponse = s3Client.uploadPart(
                    UploadPartRequest.builder()
                        .bucket(bucket)
                        .key(key)
                        .uploadId(uploadId)
                        .partNumber(i + 1)
                        .contentLength(length)
                        .build(),
                    RequestBody.fromFile(filePath)
                );

                completedParts.add(CompletedPart.builder()
                    .partNumber(i + 1)
                    .eTag(uploadResponse.eTag())
                    .build());
            }

            // 3. Completar upload
            s3Client.completeMultipartUpload(
                CompleteMultipartUploadRequest.builder()
                    .bucket(bucket)
                    .key(key)
                    .uploadId(uploadId)
                    .multipartUpload(CompletedMultipartUpload.builder()
                        .parts(completedParts)
                        .build())
                    .build()
            );

            return key;

        } catch (Exception e) {
            // Abort em caso de erro (limpar partes órfãs)
            s3Client.abortMultipartUpload(
                AbortMultipartUploadRequest.builder()
                    .bucket(bucket)
                    .key(key)
                    .uploadId(uploadId)
                    .build()
            );
            throw new RuntimeException("Multipart upload failed", e);
        }
    }
}
```

---

## EFS — Elastic File System

```hcl
# EFS — file system compartilhado (NFS) para ECS/EKS/Lambda
resource "aws_efs_file_system" "main" {
  creation_token = "${var.project}-efs"
  encrypted      = true
  kms_key_id     = aws_kms_key.efs.arn

  performance_mode = "generalPurpose"  # ou maxIO para alta concorrência
  throughput_mode  = "elastic"          # Escala automática (recomendado)

  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"  # Mover para IA após 30 dias
  }

  lifecycle_policy {
    transition_to_archive = "AFTER_90_DAYS"  # Mover para Archive após 90 dias
  }

  lifecycle_policy {
    transition_to_primary_storage_class = "AFTER_1_ACCESS"  # Voltar para Standard ao acessar
  }

  tags = merge(var.tags, { Name = "${var.project}-efs" })
}

# Mount targets (1 por AZ)
resource "aws_efs_mount_target" "main" {
  count           = length(var.private_subnet_ids)
  file_system_id  = aws_efs_file_system.main.id
  subnet_id       = var.private_subnet_ids[count.index]
  security_groups = [aws_security_group.efs.id]
}

# Access Point — isolar acesso por aplicação
resource "aws_efs_access_point" "app" {
  file_system_id = aws_efs_file_system.main.id

  posix_user {
    uid = 1000
    gid = 1000
  }

  root_directory {
    path = "/app-data"
    creation_info {
      owner_uid   = 1000
      owner_gid   = 1000
      permissions = "755"
    }
  }

  tags = { Name = "${var.project}-app-access-point" }
}

# Security Group — NFS
resource "aws_security_group" "efs" {
  name_prefix = "${var.project}-efs-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 2049
    to_port         = 2049
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
    description     = "NFS from application"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Backup automático
resource "aws_efs_backup_policy" "main" {
  file_system_id = aws_efs_file_system.main.id

  backup_policy {
    status = "ENABLED"
  }
}
```

### EFS no ECS (Task Definition)

```hcl
resource "aws_ecs_task_definition" "with_efs" {
  family                   = "${var.project}-app"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 1024
  memory                   = 2048

  volume {
    name = "shared-data"

    efs_volume_configuration {
      file_system_id     = aws_efs_file_system.main.id
      transit_encryption = "ENABLED"
      authorization_configuration {
        access_point_id = aws_efs_access_point.app.id
        iam             = "ENABLED"
      }
    }
  }

  container_definitions = jsonencode([{
    name  = "app"
    image = "${var.ecr_repo}:latest"
    mountPoints = [{
      sourceVolume  = "shared-data"
      containerPath = "/app/shared"
      readOnly      = false
    }]
  }])
}
```

---

## FSx — File Systems Gerenciados

### Quando usar cada serviço

| Serviço | Protocolo | Use Case |
|---------|-----------|----------|
| **EFS** | NFS 4.1 | Linux workloads, containers, serverless |
| **FSx for Lustre** | Lustre | HPC, ML training, processamento de vídeo |
| **FSx for NetApp ONTAP** | NFS, SMB, iSCSI | Migração de NAS on-premises, multi-protocol |
| **FSx for Windows** | SMB | Windows workloads, .NET, SQL Server |
| **FSx for OpenZFS** | NFS | Linux workloads com snapshots rápidos |

### FSx for Lustre (HPC/ML)

```hcl
# FSx Lustre — integrado com S3 para ML training
resource "aws_fsx_lustre_file_system" "ml" {
  storage_capacity            = 1200  # GiB (mínimo)
  subnet_ids                  = [var.private_subnet_ids[0]]
  security_group_ids          = [aws_security_group.fsx.id]
  deployment_type             = "PERSISTENT_2"
  per_unit_storage_throughput = 250  # MB/s por TiB

  # Integração com S3 (import/export automático)
  import_path      = "s3://${aws_s3_bucket.ml_data.id}"
  export_path      = "s3://${aws_s3_bucket.ml_data.id}/output"
  auto_import_policy = "NEW_CHANGED_DELETED"

  log_configuration {
    level       = "WARN_ERROR"
    destination = "${aws_cloudwatch_log_group.fsx.arn}:*"
  }

  tags = merge(var.tags, { Name = "${var.project}-lustre" })
}
```

---

## Storage Checklist

### S3

- [ ] Block Public Access habilitado em todas as contas (SCP)
- [ ] Criptografia SSE-KMS com bucket key habilitado
- [ ] Versionamento habilitado para dados críticos
- [ ] Lifecycle rules configuradas (transition + expiration)
- [ ] Abort incomplete multipart uploads (7 dias)
- [ ] Access logging habilitado
- [ ] Object Lock para dados regulatórios (WORM)
- [ ] S3 Access Points para consumidores distintos
- [ ] Event notifications para processamento automatizado
- [ ] Cross-Region Replication para DR

### EFS

- [ ] Criptografia at-rest e in-transit
- [ ] Access Points para isolamento por aplicação
- [ ] Lifecycle policies (IA + Archive)
- [ ] Throughput mode elastic (recomendado)
- [ ] Backup automático habilitado
- [ ] Security Group restritivo (porta 2049 apenas)

---

## Referências

- [S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/)
- [S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
- [EFS User Guide](https://docs.aws.amazon.com/efs/latest/ug/)
- [FSx Documentation](https://docs.aws.amazon.com/fsx/)
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
