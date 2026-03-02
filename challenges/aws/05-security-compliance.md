# Level 5 — Security & Compliance

> **Objetivo:** Implementar camadas de segurança na aplicação OrderFlow — IAM least-privilege,
> KMS envelope encryption, Secrets Manager com rotação, CloudTrail audit, GuardDuty detection,
> e AWS Config compliance rules. Tudo via SDK (Go + Java).

---

## Objetivo de Aprendizado

- Criar IAM policies least-privilege e entender a hierarquia (SCP → boundary → policy)
- Implementar KMS envelope encryption para dados sensíveis
- Gerenciar secrets com Secrets Manager (rotação automática, cross-account)
- Configurar CloudTrail para audit trail completo
- Habilitar GuardDuty para threat detection
- Implementar security audit via SDK para validar compliance
- Compreender IAM Roles for ECS Tasks vs Execution Role

---

## Escopo Funcional

```
Level 5 — Security Architecture
════════════════════════════════

  ┌──────────────────────────────────────────────────────────────┐
  │  AWS Organizations                                           │
  │  ┌───────────────────────────────────────────────────────┐   │
  │  │  Service Control Policies (SCPs)                      │   │
  │  │  • Deny non-approved regions                          │   │
  │  │  • Deny disable security services                     │   │
  │  └───────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │  Application (ECS / Lambda)                                  │
  │                                                              │
  │  ┌──────────┐   ┌───────────┐   ┌──────────────────┐       │
  │  │ IAM Role │   │    KMS    │   │ Secrets Manager  │       │
  │  │(task role)│──▶│  Encrypt  │   │  (credentials)   │       │
  │  └──────────┘   └───────────┘   └──────────────────┘       │
  │       │                               │                     │
  │       │ least privilege               │ rotation            │
  │       ▼                               ▼                     │
  │  ┌──────────┐   ┌───────────┐   ┌──────────────────┐       │
  │  │ DynamoDB │   │    S3     │   │    RDS/Aurora    │       │
  │  │(SSE-KMS) │   │ (SSE-KMS) │   │ (master secret)  │       │
  │  └──────────┘   └───────────┘   └──────────────────┘       │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │  Detection & Monitoring                                      │
  │                                                              │
  │  CloudTrail ──▶ S3 (logs) ──▶ CloudWatch (metric filters)  │
  │       │                              │                       │
  │       ▼                              ▼                       │
  │  GuardDuty ──▶ EventBridge ──▶ SNS (alerts)                │
  │       │                                                      │
  │  AWS Config ──▶ Rules ──▶ EventBridge ──▶ Lambda (remediate)│
  └──────────────────────────────────────────────────────────────┘
```

---

## Tarefas

### Tarefa 5.1 — IAM Policies Least-Privilege (CLI)

```bash
#!/bin/bash
# scripts/create-iam-roles.sh — Create least-privilege IAM roles

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"
ACCOUNT="000000000000"

echo "=== Creating ECS Task Role ==="

# Trust policy: ECS tasks can assume this role
$AWS iam create-role \
  --role-name orderflow-task-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {"aws:SourceAccount": "'$ACCOUNT'"}
      }
    }]
  }'

# Inline policy: least privilege for OrderFlow
$AWS iam put-role-policy \
  --role-name orderflow-task-role \
  --policy-name orderflow-service \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DynamoDBAccess",
        "Effect": "Allow",
        "Action": [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:Query",
          "dynamodb:BatchWriteItem"
        ],
        "Resource": [
          "arn:aws:dynamodb:'$REGION':'$ACCOUNT':table/orderflow-*",
          "arn:aws:dynamodb:'$REGION':'$ACCOUNT':table/orderflow-*/index/*"
        ]
      },
      {
        "Sid": "S3ReceiptsAccess",
        "Effect": "Allow",
        "Action": [
          "s3:PutObject",
          "s3:GetObject",
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::orderflow-receipts-*",
          "arn:aws:s3:::orderflow-receipts-*/*"
        ]
      },
      {
        "Sid": "SQSAccess",
        "Effect": "Allow",
        "Action": [
          "sqs:SendMessage",
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes"
        ],
        "Resource": "arn:aws:sqs:'$REGION':'$ACCOUNT':orderflow-*"
      },
      {
        "Sid": "KMSDecrypt",
        "Effect": "Allow",
        "Action": [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ],
        "Resource": "arn:aws:kms:'$REGION':'$ACCOUNT':key/*",
        "Condition": {
          "StringEquals": {
            "kms:ViaService": [
              "dynamodb.'$REGION'.amazonaws.com",
              "s3.'$REGION'.amazonaws.com"
            ]
          }
        }
      },
      {
        "Sid": "SecretsAccess",
        "Effect": "Allow",
        "Action": [
          "secretsmanager:GetSecretValue"
        ],
        "Resource": "arn:aws:secretsmanager:'$REGION':'$ACCOUNT':secret:orderflow/*"
      }
    ]
  }'

echo "=== Creating ECS Execution Role ==="
# Execution role: pulling images and writing logs (not app logic)

$AWS iam create-role \
  --role-name orderflow-execution-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

$AWS iam put-role-policy \
  --role-name orderflow-execution-role \
  --policy-name ecr-logs \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage"
        ],
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ],
        "Resource": "arn:aws:logs:'$REGION':'$ACCOUNT':log-group:/ecs/orderflow-*"
      }
    ]
  }'

echo "=== IAM Roles created ==="
echo "Task Role:      orderflow-task-role (app permissions)"
echo "Execution Role: orderflow-execution-role (infra permissions)"
```

### Tarefa 5.2 — KMS Envelope Encryption (Go)

```go
// internal/security/kms_client.go
package security

import (
	"context"
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"fmt"
	"io"
	"log/slog"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/kms"
	kmstypes "github.com/aws/aws-sdk-go-v2/service/kms/types"
)

type KMSClient struct {
	client *kms.Client
	keyID  string
}

func NewKMSClient(client *kms.Client, keyID string) *KMSClient {
	return &KMSClient{client: client, keyID: keyID}
}

// EncryptionResult holds the encrypted data and the encrypted data key.
type EncryptionResult struct {
	CiphertextBlob   []byte `json:"ciphertext_blob"`    // Encrypted data
	EncryptedDataKey []byte `json:"encrypted_data_key"`  // Encrypted AES key (store alongside data)
}

// EnvelopeEncrypt implements envelope encryption:
// 1. Request a data key from KMS (plaintext + encrypted)
// 2. Encrypt data locally with the plaintext key (AES-GCM)
// 3. Discard plaintext key, store encrypted data + encrypted key
func (k *KMSClient) EnvelopeEncrypt(ctx context.Context, plaintext []byte, encContext map[string]string) (*EncryptionResult, error) {
	// 1. Generate data key
	keyOutput, err := k.client.GenerateDataKey(ctx, &kms.GenerateDataKeyInput{
		KeyId:             aws.String(k.keyID),
		KeySpec:           kmstypes.DataKeySpecAes256,
		EncryptionContext: encContext,
	})
	if err != nil {
		return nil, fmt.Errorf("generate data key: %w", err)
	}

	// 2. Encrypt data locally with AES-GCM
	block, err := aes.NewCipher(keyOutput.Plaintext)
	if err != nil {
		return nil, fmt.Errorf("create cipher: %w", err)
	}

	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, fmt.Errorf("create GCM: %w", err)
	}

	nonce := make([]byte, gcm.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return nil, fmt.Errorf("generate nonce: %w", err)
	}

	ciphertext := gcm.Seal(nonce, nonce, plaintext, nil)

	// 3. Clear plaintext key from memory
	for i := range keyOutput.Plaintext {
		keyOutput.Plaintext[i] = 0
	}

	slog.Info("Envelope encrypted", "key_id", k.keyID, "data_size", len(plaintext))

	return &EncryptionResult{
		CiphertextBlob:   ciphertext,
		EncryptedDataKey: keyOutput.CiphertextBlob,
	}, nil
}

// EnvelopeDecrypt reverses envelope encryption:
// 1. Decrypt the data key using KMS
// 2. Decrypt the data locally with the plaintext key
func (k *KMSClient) EnvelopeDecrypt(ctx context.Context, encrypted *EncryptionResult, encContext map[string]string) ([]byte, error) {
	// 1. Decrypt data key via KMS
	keyOutput, err := k.client.Decrypt(ctx, &kms.DecryptInput{
		CiphertextBlob:    encrypted.EncryptedDataKey,
		KeyId:             aws.String(k.keyID),
		EncryptionContext: encContext,
	})
	if err != nil {
		return nil, fmt.Errorf("decrypt data key: %w", err)
	}

	// 2. Decrypt data locally
	block, err := aes.NewCipher(keyOutput.Plaintext)
	if err != nil {
		return nil, fmt.Errorf("create cipher: %w", err)
	}

	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, fmt.Errorf("create GCM: %w", err)
	}

	nonceSize := gcm.NonceSize()
	if len(encrypted.CiphertextBlob) < nonceSize {
		return nil, fmt.Errorf("ciphertext too short")
	}

	nonce, ciphertext := encrypted.CiphertextBlob[:nonceSize], encrypted.CiphertextBlob[nonceSize:]
	plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
	if err != nil {
		return nil, fmt.Errorf("decrypt data: %w", err)
	}

	// Clear plaintext key
	for i := range keyOutput.Plaintext {
		keyOutput.Plaintext[i] = 0
	}

	return plaintext, nil
}
```

### Tarefa 5.3 — KMS Envelope Encryption (Java)

```java
// infrastructure/security/KmsEncryptionService.java
package com.orderflow.infrastructure.security;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.SdkBytes;
import software.amazon.awssdk.services.kms.KmsClient;
import software.amazon.awssdk.services.kms.model.*;

import javax.crypto.Cipher;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.nio.ByteBuffer;
import java.security.SecureRandom;
import java.util.Arrays;
import java.util.Map;

@Service
public class KmsEncryptionService {

    private static final int GCM_NONCE_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 128;

    private final KmsClient kmsClient;
    private final String keyId;

    public KmsEncryptionService(
            KmsClient kmsClient,
            @Value("${orderflow.kms.key-id}") String keyId) {
        this.kmsClient = kmsClient;
        this.keyId = keyId;
    }

    /**
     * Envelope encrypt data using KMS + AES-256-GCM.
     */
    public EncryptionResult envelopeEncrypt(byte[] plaintext, Map<String, String> encryptionContext) {
        try {
            // 1. Generate data key from KMS
            var keyResponse = kmsClient.generateDataKey(GenerateDataKeyRequest.builder()
                    .keyId(keyId)
                    .keySpec(DataKeySpec.AES_256)
                    .encryptionContext(encryptionContext)
                    .build());

            byte[] plaintextKey = keyResponse.plaintext().asByteArray();

            // 2. Encrypt data locally with AES-GCM
            byte[] nonce = new byte[GCM_NONCE_LENGTH];
            new SecureRandom().nextBytes(nonce);

            var cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.ENCRYPT_MODE,
                    new SecretKeySpec(plaintextKey, "AES"),
                    new GCMParameterSpec(GCM_TAG_LENGTH, nonce));

            byte[] encrypted = cipher.doFinal(plaintext);

            // Prepend nonce to ciphertext
            var combined = ByteBuffer.allocate(nonce.length + encrypted.length);
            combined.put(nonce);
            combined.put(encrypted);

            // 3. Clear plaintext key
            Arrays.fill(plaintextKey, (byte) 0);

            return new EncryptionResult(
                    combined.array(),
                    keyResponse.ciphertextBlob().asByteArray()
            );
        } catch (Exception e) {
            throw new RuntimeException("Envelope encryption failed: " + e.getMessage(), e);
        }
    }

    /**
     * Envelope decrypt using KMS + AES-256-GCM.
     */
    public byte[] envelopeDecrypt(EncryptionResult encrypted, Map<String, String> encryptionContext) {
        try {
            // 1. Decrypt data key via KMS
            var decryptResponse = kmsClient.decrypt(DecryptRequest.builder()
                    .ciphertextBlob(SdkBytes.fromByteArray(encrypted.encryptedDataKey()))
                    .keyId(keyId)
                    .encryptionContext(encryptionContext)
                    .build());

            byte[] plaintextKey = decryptResponse.plaintext().asByteArray();

            // 2. Extract nonce and decrypt locally
            var buffer = ByteBuffer.wrap(encrypted.ciphertextBlob());
            byte[] nonce = new byte[GCM_NONCE_LENGTH];
            buffer.get(nonce);
            byte[] ciphertext = new byte[buffer.remaining()];
            buffer.get(ciphertext);

            var cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.DECRYPT_MODE,
                    new SecretKeySpec(plaintextKey, "AES"),
                    new GCMParameterSpec(GCM_TAG_LENGTH, nonce));

            byte[] decrypted = cipher.doFinal(ciphertext);

            // Clear key
            Arrays.fill(plaintextKey, (byte) 0);

            return decrypted;
        } catch (Exception e) {
            throw new RuntimeException("Envelope decryption failed: " + e.getMessage(), e);
        }
    }

    public record EncryptionResult(byte[] ciphertextBlob, byte[] encryptedDataKey) {}
}
```

### Tarefa 5.4 — Secrets Manager (Go)

```go
// internal/security/secrets_client.go
package security

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"sync"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/secretsmanager"
)

type SecretsClient struct {
	client *secretsmanager.Client
	cache  sync.Map // In-memory cache with TTL
}

func NewSecretsClient(client *secretsmanager.Client) *SecretsClient {
	return &SecretsClient{client: client}
}

type cachedSecret struct {
	value     string
	expiresAt time.Time
}

// GetSecret retrieves a secret with local caching (5-minute TTL).
func (s *SecretsClient) GetSecret(ctx context.Context, secretName string) (string, error) {
	// Check cache
	if cached, ok := s.cache.Load(secretName); ok {
		cs := cached.(*cachedSecret)
		if time.Now().Before(cs.expiresAt) {
			slog.Debug("Secret cache hit", "name", secretName)
			return cs.value, nil
		}
		s.cache.Delete(secretName)
	}

	output, err := s.client.GetSecretValue(ctx, &secretsmanager.GetSecretValueInput{
		SecretId: aws.String(secretName),
	})
	if err != nil {
		return "", fmt.Errorf("get secret %s: %w", secretName, err)
	}

	value := aws.ToString(output.SecretString)

	// Cache for 5 minutes
	s.cache.Store(secretName, &cachedSecret{
		value:     value,
		expiresAt: time.Now().Add(5 * time.Minute),
	})

	slog.Info("Secret retrieved and cached", "name", secretName, "version", aws.ToString(output.VersionId))
	return value, nil
}

// GetSecretJSON retrieves a secret and unmarshals it into a struct.
func (s *SecretsClient) GetSecretJSON(ctx context.Context, secretName string, target any) error {
	value, err := s.GetSecret(ctx, secretName)
	if err != nil {
		return err
	}

	if err := json.Unmarshal([]byte(value), target); err != nil {
		return fmt.Errorf("unmarshal secret %s: %w", secretName, err)
	}

	return nil
}

// CreateSecret creates a new secret with JSON value.
func (s *SecretsClient) CreateSecret(ctx context.Context, name string, value any) (string, error) {
	data, err := json.Marshal(value)
	if err != nil {
		return "", fmt.Errorf("marshal secret: %w", err)
	}

	output, err := s.client.CreateSecret(ctx, &secretsmanager.CreateSecretInput{
		Name:         aws.String(name),
		SecretString: aws.String(string(data)),
		Description:  aws.String("OrderFlow service secret"),
		Tags: []secretsmanager.Tag{
			{Key: aws.String("project"), Value: aws.String("orderflow")},
			{Key: aws.String("managed-by"), Value: aws.String("sdk")},
		},
	})
	if err != nil {
		return "", fmt.Errorf("create secret %s: %w", name, err)
	}

	return aws.ToString(output.ARN), nil
}

// RotateSecret triggers a rotation using a Lambda function.
func (s *SecretsClient) RotateSecret(ctx context.Context, secretName, rotationLambdaArn string) error {
	_, err := s.client.RotateSecret(ctx, &secretsmanager.RotateSecretInput{
		SecretId:           aws.String(secretName),
		RotationLambdaARN:  aws.String(rotationLambdaArn),
		RotationRules: &secretsmanager.RotationRulesType{
			AutomaticallyAfterDays: aws.Int64(30),
		},
	})
	if err != nil {
		return fmt.Errorf("rotate secret %s: %w", secretName, err)
	}

	slog.Info("Secret rotation configured", "name", secretName, "interval_days", 30)
	return nil
}
```

### Tarefa 5.5 — Secrets Manager (Java)

```java
// infrastructure/security/SecretsService.java
package com.orderflow.infrastructure.security;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.*;

import java.util.Map;

@Service
public class SecretsService {

    private final SecretsManagerClient secretsClient;
    private final ObjectMapper mapper;

    public SecretsService(SecretsManagerClient secretsClient, ObjectMapper mapper) {
        this.secretsClient = secretsClient;
        this.mapper = mapper;
    }

    /**
     * Retrieve a secret with Spring Cache (5-minute TTL configured externally).
     */
    @Cacheable(value = "secrets", key = "#secretName")
    public String getSecret(String secretName) {
        var response = secretsClient.getSecretValue(GetSecretValueRequest.builder()
                .secretId(secretName)
                .build());

        return response.secretString();
    }

    /**
     * Retrieve and deserialize JSON secret into a typed object.
     */
    public <T> T getSecretAs(String secretName, Class<T> clazz) {
        try {
            var json = getSecret(secretName);
            return mapper.readValue(json, clazz);
        } catch (Exception e) {
            throw new RuntimeException("Failed to get secret: " + e.getMessage(), e);
        }
    }

    /**
     * Create a new secret.
     */
    public String createSecret(String name, Object value) {
        try {
            var response = secretsClient.createSecret(CreateSecretRequest.builder()
                    .name(name)
                    .secretString(mapper.writeValueAsString(value))
                    .description("OrderFlow service secret")
                    .tags(
                            Tag.builder().key("project").value("orderflow").build(),
                            Tag.builder().key("managed-by").value("sdk").build()
                    )
                    .build());

            return response.arn();
        } catch (Exception e) {
            throw new RuntimeException("Failed to create secret: " + e.getMessage(), e);
        }
    }

    /**
     * Configure automatic rotation via Lambda.
     */
    public void configureRotation(String secretName, String rotationLambdaArn) {
        secretsClient.rotateSecret(RotateSecretRequest.builder()
                .secretId(secretName)
                .rotationLambdaARN(rotationLambdaArn)
                .rotationRules(RotationRulesType.builder()
                        .automaticallyAfterDays(30L)
                        .build())
                .build());
    }

    /**
     * Database credentials DTO.
     */
    public record DatabaseCredentials(String username, String password, String host, int port, String dbname) {}
}
```

### Tarefa 5.6 — CloudTrail Audit (Go)

```go
// internal/security/audit_client.go
package security

import (
	"context"
	"fmt"
	"log/slog"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/cloudtrail"
)

type AuditClient struct {
	client *cloudtrail.Client
}

func NewAuditClient(client *cloudtrail.Client) *AuditClient {
	return &AuditClient{client: client}
}

type AuditEvent struct {
	EventTime  string `json:"event_time"`
	EventName  string `json:"event_name"`
	UserName   string `json:"user_name"`
	SourceIP   string `json:"source_ip"`
	ResourceID string `json:"resource_id,omitempty"`
	IsError    bool   `json:"is_error"`
	ErrorCode  string `json:"error_code,omitempty"`
}

// LookupSecurityEvents queries CloudTrail for security-relevant events.
func (a *AuditClient) LookupSecurityEvents(ctx context.Context, since time.Time) ([]AuditEvent, error) {
	output, err := a.client.LookupEvents(ctx, &cloudtrail.LookupEventsInput{
		StartTime:  aws.Time(since),
		EndTime:    aws.Time(time.Now()),
		MaxResults: aws.Int32(50),
	})
	if err != nil {
		return nil, fmt.Errorf("lookup events: %w", err)
	}

	events := make([]AuditEvent, 0, len(output.Events))
	for _, e := range output.Events {
		event := AuditEvent{
			EventTime: aws.ToTime(e.EventTime).Format(time.RFC3339),
			EventName: aws.ToString(e.EventName),
			UserName:  aws.ToString(e.Username),
		}

		// Check for error events
		if e.CloudTrailEvent != nil {
			event.IsError = false // Parse from CloudTrailEvent JSON for error codes
		}

		events = append(events, event)
	}

	slog.Info("Audit events retrieved", "count", len(events), "since", since.Format(time.RFC3339))
	return events, nil
}

// LookupEventsByUser finds events for a specific user (investigation).
func (a *AuditClient) LookupEventsByUser(ctx context.Context, username string, since time.Time) ([]AuditEvent, error) {
	output, err := a.client.LookupEvents(ctx, &cloudtrail.LookupEventsInput{
		LookupAttributes: []cloudtrail.LookupAttribute{
			{
				AttributeKey:   "Username",
				AttributeValue: aws.String(username),
			},
		},
		StartTime:  aws.Time(since),
		MaxResults: aws.Int32(50),
	})
	if err != nil {
		return nil, fmt.Errorf("lookup by user: %w", err)
	}

	events := make([]AuditEvent, 0, len(output.Events))
	for _, e := range output.Events {
		events = append(events, AuditEvent{
			EventTime: aws.ToTime(e.EventTime).Format(time.RFC3339),
			EventName: aws.ToString(e.EventName),
			UserName:  aws.ToString(e.Username),
		})
	}

	return events, nil
}
```

### Tarefa 5.7 — Security Audit Endpoint (Go)

```go
// internal/handler/security_handler.go
package handler

import (
	"net/http"
	"time"

	"github.com/gin-gonic/gin"

	"orderflow/internal/security"
)

type SecurityHandler struct {
	audit   *security.AuditClient
	secrets *security.SecretsClient
	kms     *security.KMSClient
}

func NewSecurityHandler(audit *security.AuditClient, secrets *security.SecretsClient, kms *security.KMSClient) *SecurityHandler {
	return &SecurityHandler{audit: audit, secrets: secrets, kms: kms}
}

// SecurityAudit performs a comprehensive security check.
// GET /api/v1/security/audit
func (h *SecurityHandler) SecurityAudit(c *gin.Context) {
	ctx := c.Request.Context()

	type AuditResult struct {
		Check   string `json:"check"`
		Status  string `json:"status"` // PASS, FAIL, WARN
		Detail  string `json:"detail,omitempty"`
	}

	results := make([]AuditResult, 0)

	// 1. Check KMS key is accessible
	testData := []byte("security-audit-test")
	encCtx := map[string]string{"purpose": "audit"}
	encrypted, err := h.kms.EnvelopeEncrypt(ctx, testData, encCtx)
	if err != nil {
		results = append(results, AuditResult{
			Check:  "KMS Encryption",
			Status: "FAIL",
			Detail: err.Error(),
		})
	} else {
		_, err = h.kms.EnvelopeDecrypt(ctx, encrypted, encCtx)
		if err != nil {
			results = append(results, AuditResult{
				Check:  "KMS Decryption",
				Status: "FAIL",
				Detail: err.Error(),
			})
		} else {
			results = append(results, AuditResult{
				Check:  "KMS Envelope Encryption",
				Status: "PASS",
			})
		}
	}

	// 2. Check Secrets Manager access
	_, err = h.secrets.GetSecret(ctx, "orderflow/database")
	if err != nil {
		results = append(results, AuditResult{
			Check:  "Secrets Manager Access",
			Status: "WARN",
			Detail: "Cannot access database secret: " + err.Error(),
		})
	} else {
		results = append(results, AuditResult{
			Check:  "Secrets Manager Access",
			Status: "PASS",
		})
	}

	// 3. Check CloudTrail for unauthorized events (last 1 hour)
	events, err := h.audit.LookupSecurityEvents(ctx, time.Now().Add(-1*time.Hour))
	if err != nil {
		results = append(results, AuditResult{
			Check:  "CloudTrail Audit",
			Status: "WARN",
			Detail: err.Error(),
		})
	} else {
		errorCount := 0
		for _, e := range events {
			if e.IsError {
				errorCount++
			}
		}

		status := "PASS"
		if errorCount > 5 {
			status = "WARN"
		}
		results = append(results, AuditResult{
			Check:  "CloudTrail Audit (1h)",
			Status: status,
			Detail: fmt.Sprintf("%d events, %d errors", len(events), errorCount),
		})
	}

	// Overall status
	overallStatus := "PASS"
	for _, r := range results {
		if r.Status == "FAIL" {
			overallStatus = "FAIL"
			break
		}
		if r.Status == "WARN" {
			overallStatus = "WARN"
		}
	}

	c.JSON(http.StatusOK, gin.H{
		"status":    overallStatus,
		"timestamp": time.Now().UTC().Format(time.RFC3339),
		"checks":    results,
	})
}
```

### Tarefa 5.8 — Setup Security Stack (CLI)

```bash
#!/bin/bash
# scripts/setup-security.sh — Create KMS keys, Secrets, CloudTrail in LocalStack

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"

echo "=== Creating KMS Key ==="
KMS_KEY_ID=$($AWS kms create-key \
  --description "OrderFlow encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --tags '[{"TagKey":"project","TagValue":"orderflow"},{"TagKey":"purpose","TagValue":"data-encryption"}]' \
  --query 'KeyMetadata.KeyId' --output text)

$AWS kms create-alias \
  --alias-name alias/orderflow-key \
  --target-key-id "$KMS_KEY_ID"

# Enable automatic key rotation (every 90 days)
$AWS kms enable-key-rotation --key-id "$KMS_KEY_ID"

echo "KMS Key: $KMS_KEY_ID"

echo "=== Creating Secrets ==="
$AWS secretsmanager create-secret \
  --name orderflow/database \
  --secret-string '{"username":"orderflow","password":"local-dev-password","host":"localhost","port":5432,"dbname":"orderflow"}' \
  --description "OrderFlow database credentials"

$AWS secretsmanager create-secret \
  --name orderflow/api-keys \
  --secret-string '{"stripe_key":"sk_test_fake","sendgrid_key":"SG.fake"}' \
  --description "OrderFlow external API keys"

echo "=== Creating CloudTrail ==="
# Create S3 bucket for CloudTrail logs
$AWS s3 mb s3://orderflow-cloudtrail-logs

# Bucket policy for CloudTrail
$AWS s3api put-bucket-policy --bucket orderflow-cloudtrail-logs \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "CloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::orderflow-cloudtrail-logs/AWSLogs/*"
    },{
      "Sid": "CloudTrailCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::orderflow-cloudtrail-logs"
    }]
  }'

$AWS cloudtrail create-trail \
  --name orderflow-audit \
  --s3-bucket-name orderflow-cloudtrail-logs \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --include-global-service-events

$AWS cloudtrail start-logging --name orderflow-audit

echo "=== Enabling GuardDuty ==="
DETECTOR_ID=$($AWS guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --query 'DetectorId' --output text)

echo "GuardDuty Detector: $DETECTOR_ID"

echo "=== Security stack created ==="
echo "KMS Key ID:      $KMS_KEY_ID"
echo "Secrets:         orderflow/database, orderflow/api-keys"
echo "CloudTrail:      orderflow-audit"
echo "GuardDuty:       $DETECTOR_ID"
```

---

## IAM Hierarchy Reference

```
Hierarquia de Controles IAM
════════════════════════════

  Permissão efetiva = interseção de TODOS os níveis

  ┌─────────────────────────────────────────────────────┐
  │  SCP (Organization level)                           │
  │  "Não pode usar regiões fora de sa-east-1/us-east-1"│
  │                                                     │
  │  ┌─────────────────────────────────────────────┐   │
  │  │  Permission Boundary (account/role level)    │   │
  │  │  "Máximo: DynamoDB + S3 + SQS + KMS"        │   │
  │  │                                              │   │
  │  │  ┌──────────────────────────────────────┐   │   │
  │  │  │  Identity Policy (role level)        │   │   │
  │  │  │  "Pode: DynamoDB Query/Put + S3 Get" │   │   │
  │  │  │                                      │   │   │
  │  │  │  ┌──────────────────────────────┐   │   │   │
  │  │  │  │  Resource Policy             │   │   │   │
  │  │  │  │  (on the resource itself)    │   │   │   │
  │  │  │  │  "Allow from this role"      │   │   │   │
  │  │  │  └──────────────────────────────┘   │   │   │
  │  │  └──────────────────────────────────────┘   │   │
  │  └─────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────┘

  Task Role vs Execution Role (ECS):
  ┌───────────────────────────────────────────────────────┐
  │  Execution Role                                       │
  │  • Pull images from ECR                               │
  │  • Write logs to CloudWatch                           │
  │  • Retrieve secrets (env vars at start)               │
  │  → Usado pelo ECS agent, NÃO pela aplicação          │
  ├───────────────────────────────────────────────────────┤
  │  Task Role                                            │
  │  • DynamoDB read/write                                │
  │  • S3 upload/download                                 │
  │  • SQS send/receive                                   │
  │  • KMS encrypt/decrypt                                │
  │  → Usado pela aplicação em runtime                   │
  └───────────────────────────────────────────────────────┘
```

---

## Anti-Patterns

| Anti-Pattern | Por que está errado | Como corrigir |
|---|---|---|
| IAM policy com `Action: *` | Permissão total, blast radius máximo | Listar ações específicas |
| IAM policy com `Resource: *` | Acessa qualquer recurso do tipo | ARN específico com wildcards mínimos |
| Access keys de longa duração | Podem vazar, sem expiração | IAM Roles (assume role via STS) |
| Secrets em environment variables | Visíveis em logs/config dumps | Secrets Manager (retrieve at runtime) |
| Secrets em código-fonte | Vazam com git push | `.gitignore` + Secrets Manager |
| KMS key sem rotation | Compliance risk, key compromise | Enable auto-rotation (90 days) |
| CloudTrail desabilitado | Sem audit trail, blind spot | Multi-region, log file validation |
| Mesmo role para execução e task | Excessive permissions | Execution role (infra) vs Task role (app) |

---

## Critérios de Aceite

### IAM
- [ ] Task role com least-privilege: DynamoDB, S3, SQS, KMS, Secrets Manager
- [ ] Execution role separado: ECR pull, CloudWatch logs
- [ ] Conditions em policies (Region, ViaService, SourceAccount)
- [ ] Sem `Action: *` ou `Resource: *` em nenhuma policy

### KMS
- [ ] Envelope encryption implementado (Go + Java)
- [ ] AES-256-GCM com nonce único
- [ ] Plaintext key limpo da memória após uso
- [ ] Encryption context para auditoria
- [ ] Key rotation habilitada (90 dias)

### Secrets Manager
- [ ] Database credentials armazenadas no Secrets Manager
- [ ] Secrets retrieved via SDK (Go + Java) com cache local
- [ ] Rotação configurada (via Lambda)

### Audit
- [ ] CloudTrail habilitado (multi-region, log file validation)
- [ ] GuardDuty detector ativo
- [ ] Security audit endpoint funcional (`GET /api/v1/security/audit`)

---

## Definição de Pronto (DoD)

- [ ] IAM roles: task role + execution role com least-privilege
- [ ] KMS: envelope encryption (Go + Java) com AES-GCM
- [ ] Secrets Manager: create, retrieve, rotate (Go + Java)
- [ ] CloudTrail: audit event lookup via SDK
- [ ] Security audit endpoint funcional
- [ ] Setup script para LocalStack: KMS, Secrets, CloudTrail, GuardDuty
- [ ] Commit semântico: `feat(level-5): add IAM, KMS, Secrets Manager, CloudTrail, GuardDuty`

---

## Checklist

- [ ] IAM: task role + execution role, least-privilege, conditions
- [ ] KMS: envelope encryption, AES-256-GCM, key rotation
- [ ] Secrets Manager: CRUD, cache, rotation config
- [ ] CloudTrail: event lookup, multi-region
- [ ] GuardDuty: detector habilitado
- [ ] Security audit endpoint: KMS + Secrets + CloudTrail checks
- [ ] Scripts: `create-iam-roles.sh`, `setup-security.sh`

---

## Exercícios Extras

1. **IAM Access Analyzer** — Use o SDK para criar um Analyzer e listar findings (resources acessíveis externamente).
2. **AWS Config Rules** — Crie Config rules via SDK para validar: S3 encryption enabled, DynamoDB encrypted, Security Groups sem 0.0.0.0/0 ingress.
3. **GuardDuty Findings** — Gere sample findings via SDK (`CreateSampleFindings`) e implemente um consumer que processa findings via EventBridge.
4. **Permission Boundary** — Crie um permission boundary que limita roles criadas via automação a apenas DynamoDB + S3 + SQS.
5. **KMS Key Policy** — Crie um key policy que permite encrypt pelo service e decrypt apenas pelo task role, com conditions por VPC endpoint.
