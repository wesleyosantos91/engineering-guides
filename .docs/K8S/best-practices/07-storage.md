# Storage — Best Practices

## 1. Conceitos Fundamentais

### 1.1 Hierarquia de Storage
```
StorageClass        → Define o "tipo" de storage (SSD, HDD, NFS)
PersistentVolume    → Recurso de storage provisionado (estático ou dinâmico)
PersistentVolumeClaim → Requisição de storage por um Pod
Volume Mount        → Montagem do PVC no container
```

### 1.2 Access Modes
| Modo | Abreviação | Descrição |
|------|-----------|-----------|
| ReadWriteOnce | RWO | Leitura/escrita por um node |
| ReadOnlyMany | ROX | Leitura por múltiplos nodes |
| ReadWriteMany | RWX | Leitura/escrita por múltiplos nodes |
| ReadWriteOncePod | RWOP | Leitura/escrita por um único pod (K8s 1.27+) |

## 2. StorageClass

### 2.1 AWS EBS
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "5000"
  throughput: "250"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/xxx
reclaimPolicy: Retain  # ✅ Retain em produção
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # ✅ espera scheduling
```

### 2.2 GCP PD
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd  # ✅ HA com replicação regional
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### 2.3 Boas Práticas de StorageClass
- **`reclaimPolicy: Retain`** em produção (evita perda de dados).
- **`volumeBindingMode: WaitForFirstConsumer`** para topology-aware scheduling.
- **`allowVolumeExpansion: true`** para permitir resize.
- Habilite **encriptação** (encrypted: true, KMS).
- Crie StorageClasses específicas: `fast-ssd`, `standard-hdd`, `shared-nfs`.

## 3. PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
---
# Uso no Pod
spec:
  containers:
    - name: postgres
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata  # ✅ use subPath para evitar problemas com lost+found
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: postgres-data
```

## 4. Volume Types

### 4.1 EmptyDir
```yaml
# Temporário — morre com o Pod
volumes:
  - name: cache
    emptyDir:
      sizeLimit: 1Gi
  - name: fast-cache
    emptyDir:
      medium: Memory  # usa RAM (tmpfs)
      sizeLimit: 256Mi
```

### 4.2 ConfigMap e Secret como Volume
```yaml
volumes:
  - name: config
    configMap:
      name: app-config
      items:
        - key: config.yaml
          path: config.yaml
          mode: 0644
  - name: tls
    secret:
      secretName: app-tls
      defaultMode: 0400  # ✅ permissões restritivas
```

### 4.3 Projected Volume
```yaml
volumes:
  - name: combined
    projected:
      sources:
        - configMap:
            name: app-config
        - secret:
            name: app-secrets
        - serviceAccountToken:
            path: token
            expirationSeconds: 3600
            audience: api
```

## 5. CSI Drivers

### 5.1 CSI Drivers Comuns
| Driver | Provider | Descrição |
|--------|----------|-----------|
| **ebs.csi.aws.com** | AWS | EBS volumes |
| **efs.csi.aws.com** | AWS | EFS (NFS) |
| **pd.csi.storage.gke.io** | GCP | Persistent Disk |
| **disk.csi.azure.com** | Azure | Managed Disk |
| **file.csi.azure.com** | Azure | Azure Files (NFS/SMB) |
| **secrets-store.csi.k8s.io** | Multi | Secrets (Vault, KMS) |

### 5.2 Secrets Store CSI Driver
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/db/password"
        objectType: "secretsmanager"
---
spec:
  containers:
    - name: myapp
      volumeMounts:
        - name: secrets
          mountPath: /mnt/secrets
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: aws-secrets
```

## 6. Backup e Restore

### 6.1 Velero
```bash
# Instalar Velero
velero install \
  --provider aws \
  --bucket my-backup-bucket \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

# Backup
velero backup create prod-backup \
  --include-namespaces production \
  --include-resources deployments,services,pvc,pv,secrets,configmaps

# Schedule
velero schedule create daily-backup \
  --schedule "0 2 * * *" \
  --include-namespaces production \
  --ttl 720h  # 30 dias
```

### 6.2 VolumeSnapshot
```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Retain

---
# VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: postgres-data

---
# Restore de snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-restored
spec:
  dataSource:
    name: postgres-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
```

## 7. Boas Práticas Gerais

- **Nunca use `reclaimPolicy: Delete`** em produção para dados críticos.
- **Teste backups e restores** regularmente (disaster recovery drills).
- Use **VolumeSnapshots** para backups point-in-time de databases.
- **Monitore** uso de storage (alertas para >80% capacidade).
- Use **`subPath`** ao montar PVCs para evitar problemas com `lost+found`.
- **Encripte volumes** at rest (via StorageClass parameters).
- Implemente **storage quotas** por namespace.
- Use **ReadWriteOncePod (RWOP)** para garantir acesso exclusivo (K8s 1.27+).
- Use **`emptyDir` com sizeLimit** — evite consumir todo o disco do node.
- Para **shared storage**, avalie EFS (AWS), Filestore (GCP) ou Azure Files.
