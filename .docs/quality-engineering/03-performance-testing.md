# Performance Testing — Guia Completo

> **Versão:** 1.0
> **Última atualização:** 2026-02-24
> **Público-alvo:** Times de engenharia (back-end, plataforma, SRE, QA)
> **Documentos relacionados:** [Testing Strategy](testing-strategy.md) · [Observability & Quality](observability-quality.md)

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Tipos de Teste de Performance](#2-tipos-de-teste-de-performance)
3. [Processo e Metodologia](#3-processo-e-metodologia)
4. [Métricas e SLOs](#4-métricas-e-slos)
5. [Ferramentas](#5-ferramentas)
6. [Padrões e Receitas](#6-padrões-e-receitas)
7. [Ambientes e Infraestrutura](#7-ambientes-e-infraestrutura)
8. [Análise de Resultados](#8-análise-de-resultados)
9. [Anti-Patterns](#9-anti-patterns)
10. [Glossário](#10-glossário)

---

## 1. Visão Geral

### 1.1 Por que Testar Performance

| Razão                          | Impacto                                                                     |
|--------------------------------|-----------------------------------------------------------------------------|
| **Prevenir degradação**        | Detectar regressões de performance antes que cheguem a produção             |
| **Validar SLOs**               | Garantir que os Service Level Objectives são atingíveis sob carga real      |
| **Dimensionar infraestrutura** | Saber quantas instâncias/pods são necessários para a carga esperada         |
| **Encontrar gargalos**         | Identificar componentes que limitam throughput ou aumentam latência          |
| **Planejar capacidade**        | Projetar crescimento e antecipar necessidades de scaling                    |

### 1.2 Princípios

| #  | Princípio                       | Descrição                                                                                |
|----|---------------------------------|------------------------------------------------------------------------------------------|
| P1 | **Testar cedo e continuamente** | Performance não é fase final — integrar ao CI/CD                                         |
| P2 | **Ambiente representativo**     | Resultados só são confiáveis em ambiente que simula produção (dados, infra, config)       |
| P3 | **Baseline antes de otimizar**  | Sempre medir o estado atual antes de fazer qualquer otimização                            |
| P4 | **Um gargalo por vez**          | Resolver um gargalo por iteração, re-medir e seguir                                      |
| P5 | **Dados realistas**             | Carga e dados devem refletir padrões reais de produção                                   |
| P6 | **Automação**                   | Testes de performance devem ser reproduzíveis e automatizados                             |
| P7 | **Correlação com negócio**      | Métricas técnicas devem ser relacionadas a impacto no negócio (conversão, receita)        |

---

## 2. Tipos de Teste de Performance

### 2.1 Matriz de Tipos

| Tipo                | Objetivo                                                        | Duração típica    | Quando rodar                |
|---------------------|-----------------------------------------------------------------|-------------------|-----------------------------|
| **Load Test**       | Validar comportamento sob carga **esperada** (normal)           | 10-60 min         | Pré-release, semanal        |
| **Stress Test**     | Encontrar o ponto de quebra sob carga **acima do esperado**     | 30-120 min        | Pré-release (major changes) |
| **Spike Test**      | Validar reação a picos **súbitos** de carga                     | 5-15 min          | Quando há risco de viral/evento |
| **Soak/Endurance**  | Detectar memory leaks, degradação sob carga **constante prolongada** | 2-24 horas   | Mensal ou nightly           |
| **Scalability**     | Verificar se o sistema escala **linearmente** com mais recursos | 30-60 min         | Revisão de arquitetura      |
| **Baseline**        | Estabelecer referência de performance do estado atual           | 10-30 min         | Antes de qualquer mudança   |
| **Benchmark**       | Comparar performance entre versões, configs ou tecnologias      | 10-30 min         | Antes de decisões técnicas  |

### 2.2 Perfis de Carga

```
Load Test:        ──────────────────────────────
                  Carga constante = N esperado

Stress Test:      ──────────╱──────────╲────────
                  Rampa até exceder capacidade

Spike Test:       ────╱╲─────────╱╲─────────────
                  Picos súbitos e rápidos

Soak Test:        ──────────────────────────────
                  Carga moderada por longo período

Scalability:      ─╱──╱──╱──╱──╱──╱─────────────
                  Incrementos graduais de carga
```

---

## 3. Processo e Metodologia

### 3.1 Fluxo de Trabalho

```
┌─────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  1. Definir  │ → │ 2. Preparar  │ → │ 3. Executar  │ → │ 4. Analisar  │ → │ 5. Otimizar  │
│  Objetivos   │   │  Ambiente    │   │  Testes      │   │  Resultados  │   │  e Re-testar │
└─────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

### 3.2 Passo a Passo

| Passo | Atividade                                | Artefato                                        |
|-------|------------------------------------------|-------------------------------------------------|
| 1     | Definir SLOs e cenários críticos         | Documento de SLOs e cenários prioritários        |
| 2     | Preparar ambiente + dados realistas      | Scripts de provisionamento e seed de dados       |
| 3     | Criar scripts de carga                   | Arquivos k6/Gatling/JMeter versionados           |
| 4     | Executar baseline                        | Relatório baseline (antes de mudanças)           |
| 5     | Executar testes por tipo                 | Relatórios por tipo de teste                     |
| 6     | Analisar resultados vs. SLOs            | Dashboard comparativo                            |
| 7     | Identificar gargalos (profiling)         | Flame graphs, query plans, traces                |
| 8     | Aplicar otimização                       | Código/config modificado                         |
| 9     | Re-testar e comparar com baseline        | Relatório comparativo pré/pós otimização         |
| 10    | Automatizar no pipeline                  | Job de performance no CI/CD                      |

### 3.3 Cenários Prioritários

Priorizar cenários por **impacto** × **frequência**:

| Prioridade | Cenários                                                                   |
|------------|----------------------------------------------------------------------------|
| **P0**     | Fluxos de receita (checkout, pagamento, assinatura)                        |
| **P1**     | Fluxos de alta frequência (busca, listagem, login)                         |
| **P2**     | Fluxos de processamento batch/assíncrono (importação, relatórios)          |
| **P3**     | Fluxos de administração e backoffice (baixa frequência, menor impacto)     |

---

## 4. Métricas e SLOs

### 4.1 Métricas Essenciais

| Métrica                 | Definição                                                        | Meta típica          |
|-------------------------|------------------------------------------------------------------|----------------------|
| **Latência P50**        | Mediana do tempo de resposta                                     | ≤ 200ms              |
| **Latência P95**        | 95% dos requests são atendidos abaixo deste tempo                | ≤ 500ms              |
| **Latência P99**        | 99% dos requests são atendidos abaixo deste tempo                | ≤ 1s                 |
| **Throughput (RPS)**    | Requests por segundo sustentados sem degradação                  | Depende do cenário   |
| **Error Rate**          | % de respostas com erro (5xx, timeout) sob carga                 | ≤ 0.1%               |
| **Apdex**               | Application Performance Index (satisfação do usuário)            | ≥ 0.9                |
| **TTFB**                | Time to First Byte — tempo até o primeiro byte da resposta       | ≤ 100ms              |
| **CPU utilization**     | Uso de CPU sob carga máxima esperada                             | ≤ 70%                |
| **Memory utilization**  | Uso de memória sob carga (detectar leaks)                        | Estável, ≤ 80%       |
| **Connection pool**     | Uso de conexões de banco/cache sob carga                         | ≤ 80% do max pool    |
| **GC pause time**       | Tempo de pausa do Garbage Collector (JVM, .NET, Go)              | ≤ 50ms P99           |

### 4.2 Template de SLO

```yaml
slos:
  - name: "API de Pedidos — Criar Pedido"
    endpoint: "POST /api/v1/orders"
    metrics:
      latency_p95: "≤ 300ms"
      latency_p99: "≤ 800ms"
      error_rate: "≤ 0.1%"
      throughput: "≥ 500 RPS"
    conditions:
      concurrent_users: 1000
      data_volume: "100K pedidos no banco"
      environment: "staging (2 pods, 2 CPU, 4GB RAM cada)"
```

---

## 5. Ferramentas

### 5.1 Comparativo

| Ferramenta   | Linguagem do script | Protocolo                      | Distribuído | Ideal para                       |
|--------------|---------------------|--------------------------------|-------------|----------------------------------|
| **k6**       | JavaScript          | HTTP, gRPC, WebSocket          | ✅ (k6 Cloud) | Desenvolvedores, CI/CD         |
| **Gatling**  | Scala/Java          | HTTP, WebSocket, JMS           | ✅           | JVM teams, cenários complexos   |
| **JMeter**   | XML/GUI             | HTTP, JDBC, JMS, LDAP, FTP     | ✅           | QA teams, protocolos diversos   |
| **Locust**   | Python              | HTTP (extensível)              | ✅           | Python teams, prototipagem      |
| **Vegeta**   | Go/CLI              | HTTP                            | ❌           | Testes rápidos de endpoint      |
| **wrk/wrk2** | Lua/CLI             | HTTP                            | ❌           | Micro-benchmarks HTTP           |
| **Artillery**| YAML/JS             | HTTP, WebSocket, Socket.io     | ✅           | Serverless, Node.js teams       |

### 5.2 Exemplo — k6

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp up
    { duration: '5m', target: 100 },   // carga sustentada
    { duration: '2m', target: 200 },   // stress
    { duration: '5m', target: 200 },   // carga sustentada alta
    { duration: '2m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<300', 'p(99)<800'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const payload = JSON.stringify({
    customerId: `customer-${__VU}`,
    items: [{ sku: 'PROD-001', quantity: 1 }],
  });

  const params = {
    headers: { 'Content-Type': 'application/json' },
  };

  const res = http.post('https://api.staging.example.com/v1/orders', payload, params);

  check(res, {
    'status is 201': (r) => r.status === 201,
    'response time < 300ms': (r) => r.timings.duration < 300,
    'has orderId': (r) => JSON.parse(r.body).orderId !== undefined,
  });

  sleep(1); // think time entre requests
}
```

### 5.3 Exemplo — Gatling (Java DSL)

```java
public class CreateOrderSimulation extends Simulation {

    HttpProtocolBuilder httpProtocol = http
        .baseUrl("https://api.staging.example.com")
        .acceptHeader("application/json");

    ScenarioBuilder createOrder = scenario("Criar Pedido")
        .exec(
            http("POST /v1/orders")
                .post("/v1/orders")
                .body(StringBody("""
                    {"customerId":"cust-#{userId}","items":[{"sku":"PROD-001","quantity":1}]}
                """))
                .header("Content-Type", "application/json")
                .check(status().is(201))
                .check(jsonPath("$.orderId").exists())
        );

    {
        setUp(
            createOrder.injectOpen(
                rampUsers(100).during(Duration.ofMinutes(2)),
                constantUsersPerSec(100).during(Duration.ofMinutes(5)),
                rampUsers(200).during(Duration.ofMinutes(2))
            )
        ).protocols(httpProtocol)
         .assertions(
            global().responseTime().percentile3().lt(300),
            global().successfulRequests().percent().gt(99.0)
         );
    }
}
```

---

## 6. Padrões e Receitas

### 6.1 Think Time

> Simule o tempo real que um usuário leva entre ações. Sem think time, o teste gera carga irrealista.

```javascript
// k6 — think time variável (mais realista)
import { sleep } from 'k6';
import { randomIntBetween } from 'https://jslib.k6.io/k6-utils/1.2.0/index.js';

export default function () {
  // ... ações do cenário
  sleep(randomIntBetween(1, 5)); // 1-5 segundos entre ações
}
```

### 6.2 Dados Parametrizados

```javascript
// k6 — CSV com dados variados
import papaparse from 'https://jslib.k6.io/papaparse/5.1.1/index.js';
import { SharedArray } from 'k6/data';

const customers = new SharedArray('customers', function () {
  return papaparse.parse(open('./data/customers.csv'), { header: true }).data;
});

export default function () {
  const customer = customers[Math.floor(Math.random() * customers.length)];
  // usar customer.id, customer.name, etc.
}
```

### 6.3 Correlação e Extração de Dados

```javascript
// k6 — extrair token de login e usar em requests subsequentes
export default function () {
  const loginRes = http.post('/auth/login', JSON.stringify({
    username: 'test-user',
    password: 'test-pass',
  }));

  const token = JSON.parse(loginRes.body).accessToken;

  const ordersRes = http.get('/v1/orders', {
    headers: { Authorization: `Bearer ${token}` },
  });
}
```

### 6.4 Warmup

Sempre incluir uma fase de warmup antes de medir:

```javascript
export const options = {
  stages: [
    { duration: '1m', target: 10 },   // warmup (JIT, connection pools, caches)
    { duration: '1m', target: 50 },   // ramp up
    { duration: '5m', target: 50 },   // steady state — medir aqui
    { duration: '1m', target: 0 },    // ramp down
  ],
};
```

---

## 7. Ambientes e Infraestrutura

### 7.1 Requisitos do Ambiente de Performance

| Requisito                      | Descrição                                                                  |
|-------------------------------|----------------------------------------------------------------------------|
| **Paridade com produção**      | Mesma arquitetura, mesmas versões, mesma topologia de rede                 |
| **Isolamento**                 | Sem outros testes ou cargas concorrentes durante o teste                   |
| **Dados representativos**      | Volume e distribuição de dados similar à produção                          |
| **Monitoramento completo**     | Métricas de infra (CPU, mem, disco, rede) + APM + logs                    |
| **Reprodutibilidade**          | Ambiente provisionado via IaC, configuração versionada                    |

### 7.2 Dimensionamento

| Produção                    | Ambiente de Performance                                   |
|-----------------------------|-----------------------------------------------------------|
| 4 pods, 4 CPU, 8GB cada    | 2-4 pods, mesma spec (escalar proporcionalmente)          |
| RDS r6g.2xlarge             | RDS r6g.xlarge (ou mesma classe, menor tamanho)           |
| ElastiCache r6g.large       | ElastiCache r6g.large (manter mesma config de cache)      |

> **Regra:** Se o ambiente de performance tem metade dos recursos de produção, a carga alvo deve ser metade da carga esperada em produção. Extrapolar linearmente só funciona se o sistema escala linearmente.

### 7.3 Performance no CI/CD

```yaml
# Exemplo: GitHub Actions com k6
performance-test:
  runs-on: ubuntu-latest
  needs: [deploy-staging]
  steps:
    - uses: actions/checkout@v4
    - uses: grafana/k6-action@v0.3
      with:
        filename: tests/performance/load-test.js
        flags: --out json=results.json
    - name: Check thresholds
      run: |
        # k6 retorna exit code != 0 se thresholds falharem
        echo "Performance test completed"
    - uses: actions/upload-artifact@v4
      with:
        name: k6-results
        path: results.json
```

---

## 8. Análise de Resultados

### 8.1 Checklist de Análise

| Item                                       | O que verificar                                                     |
|--------------------------------------------|---------------------------------------------------------------------|
| **Latência por percentil**                 | P50, P95, P99 estão dentro dos SLOs?                               |
| **Throughput estável**                     | RPS se mantém constante durante steady state?                       |
| **Error rate sob carga**                   | Erros aparecem? Em qual ponto de carga?                             |
| **Recursos de infra**                      | CPU/mem/disco saturam antes do throughput alvo?                     |
| **Connection pools**                       | Conexões de banco/cache atingem o limite?                           |
| **GC pauses**                              | GC pauses aumentam sob carga? Memory leaks no soak test?            |
| **Saturação de threads**                   | Thread pools atingem max? Requests ficam na fila?                   |
| **Latência de dependências**               | Qual dependência (DB, cache, API ext) contribui mais para latência? |
| **Comparação com baseline**               | Houve regressão significativa (> 10%) vs. baseline?                 |

### 8.2 Flame Graph e Profiling

| Ferramenta            | Linguagem       | O que encontra                                    |
|-----------------------|-----------------|---------------------------------------------------|
| **async-profiler**    | Java/Kotlin     | CPU hotspots, alocação de memória, lock contention |
| **pprof**             | Go              | CPU profile, heap, goroutine, block profile        |
| **py-spy**            | Python          | CPU sampling sem overhead                          |
| **0x / clinic.js**    | Node.js         | Flame graphs, doctor, bubbleprof                   |
| **dotnet-trace**      | C#/.NET         | CPU, GC, exceptions, contention                    |

### 8.3 Template de Relatório

```markdown
## Relatório de Performance — [Nome do Teste]

**Data:** YYYY-MM-DD
**Versão:** vX.Y.Z
**Ambiente:** staging (N pods, spec)

### Resumo
- **Resultado:** ✅ PASS / ❌ FAIL
- **Throughput máximo sustentável:** X RPS
- **Ponto de quebra:** Y concurrent users

### Métricas vs. SLOs
| Métrica       | SLO       | Resultado  | Status |
|---------------|-----------|------------|--------|
| Latência P95  | ≤ 300ms   | 245ms      | ✅     |
| Latência P99  | ≤ 800ms   | 720ms      | ✅     |
| Error Rate    | ≤ 0.1%    | 0.05%      | ✅     |
| Throughput    | ≥ 500 RPS | 620 RPS    | ✅     |

### Gargalos Identificados
1. [Descrição do gargalo]
2. [Ação recomendada]

### Comparação com Baseline
| Métrica      | Baseline   | Atual      | Delta  |
|--------------|------------|------------|--------|
| P95          | 280ms      | 245ms      | -12.5% |
```

---

## 9. Anti-Patterns

| Anti-pattern                                 | Problema                                           | Solução                                              |
|----------------------------------------------|----------------------------------------------------|------------------------------------------------------|
| Testar em ambiente diferente de produção     | Resultados não representam realidade                | Manter paridade de ambiente (IaC)                    |
| Sem think time                               | Carga irrealista, resultados inflados              | Adicionar think time variável entre ações            |
| Medir apenas média de latência               | Média esconde outliers (P99 pode ser 10x a média) | Medir P50, P95, P99 e max                            |
| Testes manuais e não reproduzíveis           | Impossível comparar resultados entre execuções     | Scripts versionados, ambiente automatizado            |
| Otimizar sem baseline                        | Não sabe se a otimização melhorou ou piorou         | Sempre medir baseline antes de qualquer mudança      |
| Ignorar warmup                               | JIT, caches frios distorcem métricas iniciais      | Fase de warmup antes de medir                         |
| Testar apenas happy path                     | Não descobre performance sob erro/retry            | Incluir cenários de erro e degradação graceful        |
| Volume de dados de teste irrealista          | Performance boa com 100 registros, ruim com 10M    | Usar volume de dados proporcional à produção         |
| Não correlacionar com métricas de infra      | Sabe que está lento mas não sabe por quê           | APM + métricas de infra durante o teste               |

---

## 10. Glossário

| Termo                  | Definição                                                                                          |
|------------------------|----------------------------------------------------------------------------------------------------|
| **Apdex**              | Application Performance Index — score de 0 a 1 que mede satisfação do usuário com performance.    |
| **Baseline**           | Medição de referência da performance atual, usada para comparar após mudanças.                     |
| **Flame graph**        | Visualização de profiling que mostra onde o tempo de CPU é gasto de forma hierárquica.             |
| **Latência**           | Tempo decorrido entre o envio de uma requisição e o recebimento da resposta.                       |
| **P50/P95/P99**        | Percentis — ex.: P95 = 95% dos requests são atendidos abaixo deste tempo.                         |
| **RPS**                | Requests Per Second — throughput do sistema.                                                        |
| **Saturação**          | Ponto onde adicionar mais carga não aumenta throughput, apenas latência e erros.                    |
| **SLO**                | Service Level Objective — meta de qualidade de serviço definida internamente.                      |
| **Soak test**          | Teste de longa duração para detectar memory leaks e degradação gradual.                             |
| **Spike test**         | Teste que simula aumento súbito e intenso de carga por curto período.                               |
| **Stress test**        | Teste que excede a carga máxima esperada para encontrar ponto de quebra.                            |
| **Think time**         | Pausa entre ações do usuário virtual, simulando comportamento real.                                 |
| **Throughput**         | Volume de trabalho processado por unidade de tempo (ex.: requests/segundo).                         |
| **TTFB**               | Time to First Byte — tempo do request até o primeiro byte da resposta ser recebido.                |
| **Warmup**             | Período inicial de execução para estabilizar JIT, caches e connection pools antes de medir.         |
