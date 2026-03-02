# Level 9 — Service Coordination: Discovery, Health Checks & Leader Election

> **Objetivo:** Implementar Service Discovery (client-side e server-side), Health Checks
> com heartbeat e um mecanismo de Leader Election para coordenação distribuída.

**Referência:**
- [18-service-discovery.md](../../.docs/SYSTEM-DESIGN/18-service-discovery.md)
- [19-heartbeat-health-checks.md](../../.docs/SYSTEM-DESIGN/19-heartbeat-health-checks.md)
- [20-leader-election.md](../../.docs/SYSTEM-DESIGN/20-leader-election.md)

**Pré-requisito:** Level 8 completo.

---

## Parte 1 — ADR: Service Coordination Strategy

**Arquivo:** `docs/adrs/ADR-001-service-coordination.md`

**Decisões:**
1. Service Discovery: client-side vs server-side vs DNS-based
2. Health check model: push vs pull
3. Leader election algorithm

**Options — Service Discovery:**
1. **Client-side** — client consulta registry e faz balanceamento (ex: Eureka)
2. **Server-side** — load balancer consulta registry (ex: AWS ALB + Cloud Map)
3. **DNS-based** — SRV records atualizados automaticamente (ex: Consul DNS)
4. **Mesh-based** — sidecar proxy resolve (ex: Istio/Envoy)

**Options — Leader Election:**
1. **Bully Algorithm** — maior ID vence
2. **Ring Algorithm** — token ring
3. **Raft-based** — consensus para leader election
4. **ZooKeeper/etcd lock** — distributed lock

**Critérios de aceite:**
- [ ] Comparativo client-side vs server-side vs DNS-based discovery
- [ ] Push vs pull health check trade-offs
- [ ] Leader election algorithm justificado
- [ ] Failure scenarios documentados (split-brain, network partition)

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/09-service-coordination.drawio`

**View 1 — Service Discovery Flow:**
```
┌─────────┐  1. Register  ┌──────────┐
│Service A│───────────────▶│ Service  │
│ (inst1) │                │ Registry │
│ (inst2) │◀───────────────│(Consul/  │
│ (inst3) │  4. Health     │ etcd)    │
└─────────┘     Check      └────┬─────┘
                                │
┌─────────┐  2. Discover   ┌───▼─────┐
│Service B│───────────────▶│Registry │
│ (client)│◀───────────────│  API    │
└─────────┘  3. Endpoints  └─────────┘
```

**View 2 — Health Check & Heartbeat:** Push/Pull model com failure detection

**View 3 — Leader Election:** Raft-style election com term numbers

**Critérios de aceite:**
- [ ] Service registration e discovery flow
- [ ] Health check com failure detection timeline
- [ ] Leader election step-by-step

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── registry/main.go          ← Service Registry server
│   ├── service/main.go           ← Sample service (registers itself)
│   └── election/main.go          ← Leader election demo
├── internal/
│   ├── registry/
│   │   ├── registry.go           ← Service registry (in-memory)
│   │   ├── api.go                ← HTTP API (register, deregister, discover)
│   │   ├── health.go             ← Health checker (periodic)
│   │   └── registry_test.go
│   ├── discovery/
│   │   ├── client.go             ← Client-side discovery SDK
│   │   ├── dns.go                ← DNS-based discovery
│   │   └── loadbalancer.go       ← Client-side load balancing
│   ├── heartbeat/
│   │   ├── sender.go             ← Heartbeat sender (service side)
│   │   ├── detector.go           ← Failure detector (registry side)
│   │   ├── phi.go                ← Phi Accrual failure detector
│   │   └── heartbeat_test.go
│   ├── election/
│   │   ├── raft_election.go      ← Simplified Raft leader election
│   │   ├── bully.go              ← Bully algorithm
│   │   ├── node.go               ← Election participant
│   │   └── election_test.go
│   └── client/
│       └── sdk.go                ← Client SDK (register + heartbeat)
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Service Registry** com register, deregister, discover endpoints
2. **Client-side Discovery** SDK com local cache e refresh
3. **DNS-based Discovery** retornando SRV records
4. **Heartbeat Sender** com goroutine periódica
5. **Phi Accrual Failure Detector** (como Cassandra/Akka usam)
6. **Bully Algorithm** para leader election
7. **Simplified Raft Election** com term numbers e vote requests
8. **Split-brain protection** com fencing tokens
9. **Graceful deregistration** no shutdown

**Critérios de aceite Go:**
- [ ] Registry suporta register/deregister/discover
- [ ] Health check remove serviços unhealthy automaticamente
- [ ] Phi accrual detector adapta threshold baseado em latência
- [ ] Bully election: maior ID se torna leader
- [ ] Raft election: term numbers e majority votes
- [ ] Split-brain: fencing token previne dual-leader
- [ ] SDK: auto-register + heartbeat automático
- [ ] ≥ 20 testes

---

### 3.2 — Java

**Funcionalidades Java:**
1. **Service Registry** com Spring Boot
2. **Spring Cloud Netflix Eureka** integration (compare com custom)
3. **Scheduled health checks** com `@Scheduled` + Virtual Threads
4. **Leader election** com Spring Integration (JDBC lock)
5. **Records** para service instances
6. **Sealed interface** para election algorithms

**Critérios de aceite Java:**
- [ ] Custom registry + Eureka comparison
- [ ] Health checks com Virtual Threads
- [ ] Leader election funcional
- [ ] Testes com múltiplas instâncias (Testcontainers)
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] ADR com estratégia de coordenação
- [ ] DrawIO com 3 views
- [ ] Go e Java: registry + discovery + health + election + tests
- [ ] Demo com 3+ instâncias de serviço
- [ ] Commit: `feat(system-design-09): service discovery health checks leader election`
