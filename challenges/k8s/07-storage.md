# Level 7 — Storage

> **Objetivo:** Dominar armazenamento no Kubernetes — StorageClasses, PVs, PVCs, CSI Drivers,
> Volume Snapshots e backup com Velero — para os workloads stateful da CloudShop.

**Referência:** [Storage Best Practices](../../.docs/k8s/07-storage.md)

---

## Objetivo de Aprendizado

- Configurar StorageClasses para diferentes tiers
- Criar PVCs com modos de acesso adequados
- Usar volumes efêmeros (emptyDir, projected)
- Implementar Volume Snapshots para backups point-in-time
- Configurar CSI Drivers para cloud providers
- Implementar backup/restore com Velero

---

## Contexto do Domínio

A plataforma CloudShop tem workloads stateful (PostgreSQL, Redis, Kafka) que precisam de storage confiável, com backups automáticos e capacidade de restore. Diferentes tiers de storage (SSD vs HDD) são usados para balancear performance e custo.

---

## Desafios

### Desafio 7.1 — StorageClasses

Configure StorageClasses para diferentes necessidades de storage.

**Requisitos:**
- StorageClass `fast` (SSD) para databases
- StorageClass `standard` para logs e dados menos críticos
- Definir `reclaimPolicy`, `volumeBindingMode` e `allowVolumeExpansion`

```yaml
# StorageClass — SSD (para databases)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: rancher.io/local-path  # ou ebs.csi.aws.com em AWS
parameters:
  type: gp3             # AWS
  iopsPerGB: "3000"
reclaimPolicy: Retain    # NÃO apagar dados ao deletar PVC
volumeBindingMode: WaitForFirstConsumer  # binding quando Pod schedule
allowVolumeExpansion: true
---
# StorageClass — Standard (para logs)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# Definir default
kubectl patch storageclass standard \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**Critérios de aceite:**
- [ ] StorageClass `fast-ssd` com reclaimPolicy Retain
- [ ] StorageClass `standard` com reclaimPolicy Delete
- [ ] `volumeBindingMode: WaitForFirstConsumer` em ambas
- [ ] `allowVolumeExpansion: true` em ambas
- [ ] Uma StorageClass marcada como default

---

### Desafio 7.2 — PVCs para StatefulSets

Configure PVCs para os databases da CloudShop via VolumeClaimTemplates.

**Requisitos:**
- PostgreSQL: 10Gi SSD, ReadWriteOnce
- Redis: 5Gi SSD, ReadWriteOnce
- Kafka: 20Gi Standard, ReadWriteOnce

```yaml
# PostgreSQL StatefulSet com VolumeClaimTemplate
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: cloudshop-data
spec:
  serviceName: postgresql
  replicas: 3
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: postgres:16
          ports:
            - containerPort: 5432
              name: postgres
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
              subPath: pgdata    # evitar conflito com lost+found
            - name: config
              mountPath: /etc/postgresql/conf.d
          env:
            - name: POSTGRES_DB
              value: cloudshop
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 500m
              memory: 1Gi
      volumes:
        - name: config
          configMap:
            name: postgresql-config
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 10Gi
```

```bash
# Verificar PVCs criados
kubectl get pvc -n cloudshop-data
# data-postgresql-0   Bound   pvc-xxx   10Gi   RWO   fast-ssd
# data-postgresql-1   Bound   pvc-yyy   10Gi   RWO   fast-ssd
# data-postgresql-2   Bound   pvc-zzz   10Gi   RWO   fast-ssd

# Verificar PVs
kubectl get pv

# Expandir PVC (se necessário)
kubectl patch pvc data-postgresql-0 -n cloudshop-data \
  -p '{"spec": {"resources": {"requests": {"storage": "20Gi"}}}}'
```

**Critérios de aceite:**
- [ ] PostgreSQL: 3 PVCs de 10Gi em fast-ssd
- [ ] Redis: 3 PVCs de 5Gi em fast-ssd
- [ ] Kafka: 3 PVCs de 20Gi em standard
- [ ] Todos os PVCs em status Bound
- [ ] `subPath` usado para evitar conflito com lost+found
- [ ] Volume expansion testada

---

### Desafio 7.3 — Volumes Efêmeros

Use emptyDir e projected volumes para dados temporários e configuração.

**Requisitos:**
- emptyDir para cache e dados temporários
- emptyDir com `medium: Memory` para tmpfs
- Projected volume combinando ConfigMap + Secret + downward API

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: cloudshop-prod
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: registry.cloudshop.io/order-service:v1.2.0
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /app/cache
            - name: pod-info
              mountPath: /etc/pod-info
              readOnly: true
      volumes:
        # tmpfs — dados em memória (rápido, limitado)
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: 64Mi
        # Cache efêmero em disco
        - name: cache
          emptyDir:
            sizeLimit: 256Mi
        # Projected — combina múltiplas fontes
        - name: pod-info
          projected:
            sources:
              - configMap:
                  name: order-service-config
              - secret:
                  name: order-service-secret
                  items:
                    - key: db-password
                      path: db-password
              - downwardAPI:
                  items:
                    - path: namespace
                      fieldRef:
                        fieldPath: metadata.namespace
                    - path: pod-name
                      fieldRef:
                        fieldPath: metadata.name
                    - path: cpu-request
                      resourceFieldRef:
                        containerName: order-service
                        resource: requests.cpu
```

**Critérios de aceite:**
- [ ] emptyDir `Memory` para /tmp (tmpfs)
- [ ] emptyDir com sizeLimit para cache
- [ ] Projected volume com ConfigMap + Secret + downward API
- [ ] Pod-info acessível: `cat /etc/pod-info/namespace`
- [ ] sizeLimit evita consumo excessivo

---

### Desafio 7.4 — Volume Snapshots

Implemente snapshots point-in-time dos PVs para backup rápido.

**Requisitos:**
- VolumeSnapshotClass configurada
- VolumeSnapshot manual para PostgreSQL
- CronJob para snapshots periódicos
- Restore a partir de snapshot

```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: fast-ssd-snapshot
driver: rancher.io/local-path   # ou ebs.csi.aws.com
deletionPolicy: Retain
---
# VolumeSnapshot — backup do PostgreSQL
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgresql-snapshot-20240101
  namespace: cloudshop-data
spec:
  volumeSnapshotClassName: fast-ssd-snapshot
  source:
    persistentVolumeClaimName: data-postgresql-0
---
# Restore — criar PVC a partir do snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-postgresql-restored
  namespace: cloudshop-data
spec:
  storageClassName: fast-ssd
  dataSource:
    name: postgresql-snapshot-20240101
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**Critérios de aceite:**
- [ ] VolumeSnapshotClass criada
- [ ] VolumeSnapshot do PostgreSQL criado com sucesso
- [ ] Snapshot status: readyToUse = true
- [ ] PVC restaurado a partir do snapshot
- [ ] Dados verificados no PVC restaurado

---

### Desafio 7.5 — Backup com Velero

Configure Velero para backup/restore completo dos workloads e dados.

**Requisitos:**
- Instalar Velero com provider adequado
- Backup schedule para namespaces CloudShop
- Incluir PVs no backup (restic/kopia)
- Testar restore completo

```bash
# Instalar Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket cloudshop-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-node-agent \
  --default-volumes-to-fs-backup

# Backup manual
velero backup create cloudshop-full \
  --include-namespaces cloudshop-prod,cloudshop-data \
  --default-volumes-to-fs-backup \
  --wait

# Schedule (diário)
velero schedule create cloudshop-daily \
  --schedule="0 2 * * *" \
  --include-namespaces cloudshop-prod,cloudshop-data \
  --default-volumes-to-fs-backup \
  --ttl 168h    # reter 7 dias

# Verify
velero backup get
velero backup describe cloudshop-full

# Restore
velero restore create --from-backup cloudshop-full \
  --include-namespaces cloudshop-prod \
  --wait
```

```yaml
# Velero Schedule como manifest
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: cloudshop-daily
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - cloudshop-prod
      - cloudshop-data
    defaultVolumesToFsBackup: true
    ttl: 168h0m0s
    storageLocation: default
    snapshotVolumes: true
```

**Critérios de aceite:**
- [ ] Velero instalado e funcional
- [ ] Backup manual inclui PVs (volumes)
- [ ] Schedule diário criado
- [ ] Backup status: Completed
- [ ] Restore testado com sucesso
- [ ] Dados verificados após restore

---

### Desafio 7.6 — Storage Monitoring

Monitore uso de storage e configure alertas para capacidade.

**Requisitos:**
- Métricas de PVC usage via kubelet
- Alertas para PVC >80% utilização
- Dashboard de storage no Grafana

```yaml
# PrometheusRule — alertas de storage
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: storage-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: storage
      rules:
        - alert: PVCNearFull
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.8
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} está >80% cheio"
            description: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} usando {{ $value | humanizePercentage }}"

        - alert: PVCAlmostFull
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.95
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} está >95% cheio — AÇÃO URGENTE"
```

```bash
# Verificar uso de volumes
kubectl exec -it postgresql-0 -n cloudshop-data -- df -h /var/lib/postgresql/data

# Métricas kubelet
kubectl get --raw /api/v1/nodes/<node>/proxy/stats/summary | jq '.pods[].volume[]'
```

**Critérios de aceite:**
- [ ] Métricas de PVC expostas no Prometheus
- [ ] Alerta warning para >80% utilização
- [ ] Alerta critical para >95% utilização
- [ ] Dashboard Grafana com uso de storage
- [ ] `df -h` verificado nos StatefulSets

---

## Definição de Pronto (DoD)

- [ ] StorageClasses configuradas (fast-ssd, standard)
- [ ] PVCs para todos os StatefulSets (PostgreSQL, Redis, Kafka)
- [ ] Volumes efêmeros (emptyDir, projected) configurados
- [ ] Volume Snapshots implementados
- [ ] Velero backup/restore testado
- [ ] Alertas de storage configurados
- [ ] Commit: `feat(level-7): add storage configuration`

---

## Checklist

- [ ] 2+ StorageClasses com políticas diferentes
- [ ] VolumeClaimTemplates nos StatefulSets
- [ ] emptyDir com sizeLimit
- [ ] Projected volumes (ConfigMap + Secret + downward API)
- [ ] VolumeSnapshot para PostgreSQL
- [ ] Velero instalado com schedule diário
- [ ] Restore testado e dados verificados
- [ ] Alertas PVC >80% e >95%
- [ ] Volume expansion testada
