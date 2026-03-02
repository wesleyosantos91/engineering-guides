# Level 8 — High Availability & Disaster Recovery

> **Objetivo:** Projetar e implementar alta disponibilidade e recuperação de desastres
> para a plataforma CloudShop — PDBs, topology spread, multi-cluster e DR plans.

**Referência:** [High Availability Best Practices](../../.docs/k8s/08-high-availability.md)

---

## Objetivo de Aprendizado

- Configurar PodDisruptionBudgets para zero-downtime maintenance
- Implementar topology spread e anti-affinity para resiliência
- Projetar graceful shutdown com preStop hooks e terminationGracePeriodSeconds
- Planejar DR com tiers de RPO/RTO
- Implementar backup schedule por tier
- Avaliar estratégias multi-cluster

---

## Contexto do Domínio

A plataforma CloudShop é mission-critical. Ela precisa sobreviver a falhas de node, upgrades de cluster, e até falha de zona de disponibilidade. Os clients esperam 99.9% de disponibilidade, e o time precisa de um plano claro de DR para diferentes cenários.

---

## Desafios

### Desafio 8.1 — PodDisruptionBudgets

Configure PDBs para garantir disponibilidade durante manutenção de nodes.

**Requisitos:**
- PDB para cada microsserviço (minAvailable ou maxUnavailable)
- PDB para databases StatefulSet
- Testar com `kubectl drain`

```yaml
# PDB — microsserviços (maxUnavailable)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service
  namespace: cloudshop-prod
spec:
  maxUnavailable: 1   # pelo menos N-1 sempre disponíveis
  selector:
    matchLabels:
      app.kubernetes.io/name: order-service
---
# PDB — databases (minAvailable para quorum)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgresql
  namespace: cloudshop-data
spec:
  minAvailable: 2    # quorum: 2 de 3 sempre disponíveis
  selector:
    matchLabels:
      app: postgresql
```

```bash
# Verificar PDBs
kubectl get pdb -A

# Testar: drain node (respeita PDB)
kubectl drain node-worker-1 --ignore-daemonsets --delete-emptydir-data

# Verificar que drain é bloqueado se violar PDB
kubectl get events --field-selector reason=DisruptionBudget

# Uncordon após teste
kubectl uncordon node-worker-1
```

**Critérios de aceite:**
- [ ] PDB para cada microsserviço (maxUnavailable: 1)
- [ ] PDB para PostgreSQL (minAvailable: 2 de 3)
- [ ] PDB para Redis (minAvailable: 2 de 3)
- [ ] `kubectl drain` respeita PDB
- [ ] Drain é bloqueado quando viola budget

---

### Desafio 8.2 — Topology Spread e Anti-Affinity

Distribua Pods entre nodes e zonas para sobreviver a falhas.

**Requisitos:**
- topologySpreadConstraints por zona e node
- podAntiAffinity para evitar co-location de réplicas
- Preferência por spread em zonas (multi-AZ)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: cloudshop-prod
spec:
  replicas: 3
  template:
    spec:
      topologySpreadConstraints:
        # Spread entre zonas
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: order-service
        # Spread entre nodes
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: order-service
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - order-service
                topologyKey: kubernetes.io/hostname
```

```bash
# Verificar distribuição dos Pods
kubectl get pods -n cloudshop-prod -o wide \
  -l app.kubernetes.io/name=order-service

# Verificar labels dos nodes (zonas)
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone

# Simular falha de node
kubectl cordon node-worker-2
# Verificar que Pods são redistribuídos mantendo o spread
```

**Critérios de aceite:**
- [ ] topologySpreadConstraints por zona e por node
- [ ] Pods distribuídos em diferentes nodes
- [ ] Anti-affinity evita 2 réplicas no mesmo node
- [ ] Falha de 1 node não deixa serviço indisponível
- [ ] `whenUnsatisfiable: DoNotSchedule` para zonas

---

### Desafio 8.3 — Graceful Shutdown

Implemente shutdown gracioso para zero-downtime deploys.

**Requisitos:**
- preStop hook com sleep para dar tempo ao load balancer
- terminationGracePeriodSeconds adequado
- SIGTERM handling na aplicação
- Readiness probe desabilitada antes do shutdown

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: cloudshop-prod
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: order-service
          image: registry.cloudshop.io/order-service:v1.2.0
          ports:
            - containerPort: 8080
              name: http
          lifecycle:
            preStop:
              exec:
                command:
                  - sh
                  - -c
                  - |
                    # 1. Sinaliza para readiness probe que não está pronto
                    touch /tmp/shutdown
                    # 2. Espera LB remover o Pod do pool (propagação endpoints)
                    sleep 15
                    # 3. SIGTERM será enviado após preStop
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            periodSeconds: 5
            # App retorna 503 quando /tmp/shutdown existe
          livenessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 10
            initialDelaySeconds: 10
```

**Critérios de aceite:**
- [ ] preStop com sleep 15s (tempo de propagação endpoints)
- [ ] terminationGracePeriodSeconds > preStop + app drain
- [ ] Zero requests com erro 5xx durante rolling update
- [ ] Conexões em andamento finalizadas antes do shutdown
- [ ] Testado com carga contínua durante deploy

---

### Desafio 8.4 — Disaster Recovery Tiers

Defina tiers de DR com RPO/RTO para cada componente da CloudShop.

**Requisitos:**
- Classificar componentes por criticidade
- Definir RPO (Recovery Point Objective) e RTO (Recovery Time Objective)
- Configurar backups com frequência adequada por tier

| Tier | Componente | RPO | RTO | Backup |
|------|-----------|-----|-----|--------|
| 1 (Critical) | PostgreSQL (orders, payments) | 5min | 15min | Continuous WAL + Snapshot cada 5min |
| 2 (Important) | Redis (sessions, cache) | 1h | 30min | Snapshot cada hora |
| 3 (Standard)  | Kafka (eventos) | 4h | 2h | Snapshot diário |
| 4 (Low) | Logs, métricas | 24h | 4h | Backup diário |

```yaml
# Velero — Backup Tier 1 (cada 5 minutos)
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: tier1-critical
  namespace: velero
spec:
  schedule: "*/5 * * * *"
  template:
    includedNamespaces:
      - cloudshop-data
    labelSelector:
      matchLabels:
        backup-tier: "1"
    defaultVolumesToFsBackup: true
    ttl: 24h0m0s
---
# Velero — Backup Tier 2 (cada hora)
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: tier2-important
  namespace: velero
spec:
  schedule: "0 * * * *"
  template:
    includedNamespaces:
      - cloudshop-data
    labelSelector:
      matchLabels:
        backup-tier: "2"
    defaultVolumesToFsBackup: true
    ttl: 72h0m0s
---
# Velero — Backup Tier 3 (diário)
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: tier3-standard
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - cloudshop-data
      - cloudshop-prod
    labelSelector:
      matchLabels:
        backup-tier: "3"
    defaultVolumesToFsBackup: true
    ttl: 168h0m0s
```

**Critérios de aceite:**
- [ ] Tabela de tiers documentada (RPO/RTO por componente)
- [ ] Velero Schedules por tier configurados
- [ ] Tier 1: backup a cada 5min, TTL 24h
- [ ] Tier 2: backup a cada hora, TTL 72h
- [ ] Tier 3: backup diário, TTL 7 dias
- [ ] Labels `backup-tier` aplicados nos workloads

---

### Desafio 8.5 — DR Drill (Simulação de Desastre)

Execute um drill completo de DR: simule falha, execute restore, valide dados.

**Requisitos:**
- Simular perda de namespace `cloudshop-prod`
- Restore via Velero
- Validar integridade dos dados
- Medir tempo real de recovery (comparar com RTO)

```bash
# 1. Preparação — criar dados de teste
kubectl exec -it postgresql-0 -n cloudshop-data -- psql -U cloudshop -c \
  "INSERT INTO orders (id, status) VALUES ('dr-test-001', 'pending');"

# 2. Criar backup pre-drill
velero backup create pre-drill \
  --include-namespaces cloudshop-prod,cloudshop-data \
  --default-volumes-to-fs-backup \
  --wait

# 3. Simular desastre — deletar namespace
kubectl delete namespace cloudshop-prod
# ⚠️ APENAS em ambiente de teste!

# 4. Verificar que serviços estão down
kubectl get pods -n cloudshop-prod  # "No resources found"

# 5. Iniciar cronômetro
START_TIME=$(date +%s)

# 6. Restore
velero restore create dr-drill \
  --from-backup pre-drill \
  --include-namespaces cloudshop-prod \
  --wait

# 7. Verificar restore
kubectl get pods -n cloudshop-prod
kubectl wait --for=condition=ready pod -l app.kubernetes.io/part-of=cloudshop \
  -n cloudshop-prod --timeout=300s

# 8. Calcular RTO real
END_TIME=$(date +%s)
echo "Recovery Time: $(($END_TIME - $START_TIME)) seconds"

# 9. Validar dados
kubectl exec -it postgresql-0 -n cloudshop-data -- psql -U cloudshop -c \
  "SELECT * FROM orders WHERE id = 'dr-test-001';"
```

**Critérios de aceite:**
- [ ] Drill executado com sucesso
- [ ] Dados restaurados e verificados
- [ ] RTO real medido e documentado
- [ ] RTO real ≤ RTO target do tier
- [ ] Runbook de DR documentado
- [ ] Lições aprendidas registradas

---

### Desafio 8.6 — Multi-Cluster (Avançado)

Avalie e documente estratégias multi-cluster para HA/DR.

**Requisitos:**
- Comparar Active-Active, Active-Passive e Hub-Spoke
- Documentar prós/contras de cada estratégia
- Avaliar ferramentas: Submariner, Cilium Cluster Mesh, Liqo
- Propor arquitetura multi-cluster para CloudShop

```markdown
# Análise Multi-Cluster para CloudShop

## Active-Active (Multi-Region)
- Prós: Menor latência (proximidade), failover instantâneo
- Contras: Complexidade de dados (replicação cross-region), custo 2x
- Quando: RPO=0, RTO < 1min

## Active-Passive
- Prós: Mais simples, custo menor (standby pode ser menor)
- Contras: RTO maior (precisa promover standby), dados podem defasar
- Quando: RPO < 1h, RTO < 30min

## Hub-Spoke
- Prós: Gestão centralizada, policies consistentes
- Contras: Hub é SPOF para management plane
- Quando: Múltiplos clusters edge/regional

## Recomendação CloudShop
- **Fase 1**: Single cluster, multi-AZ (topology spread)
- **Fase 2**: Active-Passive com Velero cross-region
- **Fase 3**: Active-Active com database replication
```

**Critérios de aceite:**
- [ ] Comparação documentada (Active-Active vs Passive vs Hub-Spoke)
- [ ] Trade-offs de cada estratégia documentados
- [ ] Roadmap multi-cluster para CloudShop (Fase 1-3)
- [ ] Ferramentas avaliadas (pelo menos 2)
- [ ] Decisão justificada para fase atual

---

## Definição de Pronto (DoD)

- [ ] PDBs para todos os workloads
- [ ] Topology spread entre zones e nodes
- [ ] Graceful shutdown com preStop hooks
- [ ] DR tiers com RPO/RTO documentados
- [ ] DR drill executado com sucesso
- [ ] Estratégias multi-cluster avaliadas
- [ ] Commit: `feat(level-8): add high availability and dr`

---

## Checklist

- [ ] PDB para microsserviços (maxUnavailable: 1)
- [ ] PDB para databases (minAvailable: quorum)
- [ ] topologySpreadConstraints por zona
- [ ] podAntiAffinity entre réplicas
- [ ] preStop hook com sleep 15s
- [ ] terminationGracePeriodSeconds configurado
- [ ] DR tiers documentados (tabela RPO/RTO)
- [ ] Velero schedules por tier
- [ ] DR drill executado e medido
- [ ] Multi-cluster avaliado
