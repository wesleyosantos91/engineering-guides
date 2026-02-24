# Timeout (Time Limiter)

> **Categoria:** Resiliência
> **Complementa:** Circuit Breaker, Retry, Bulkhead
> **Keywords:** resiliência, timeout, time limiter, bloqueio, latência, thread pool, deadline propagation

---

## Problema

Uma chamada a um serviço externo (API, banco de dados, message broker) pode demorar **indefinidamente**. Sem um limite de tempo, a thread (ou conexão) fica bloqueada, recursos são consumidos e, eventualmente, todas as threads disponíveis são esgotadas — causando indisponibilidade total do serviço.

Uma única dependência lenta pode derrubar todo o sistema se não houver timeouts.

---

## Solução

Definir um **tempo máximo** para cada chamada. Se a operação não completar dentro do prazo, ela é **cancelada** e tratada como falha — liberando recursos imediatamente.

---

## Tipos de Timeout

| Tipo | O que controla | Exemplo |
|------|---------------|---------|
| **Connection Timeout** | Tempo máximo para **estabelecer** a conexão TCP | 3 segundos |
| **Read Timeout (Socket Timeout)** | Tempo máximo para **receber** dados após a conexão estar estabelecida | 5 segundos |
| **Request Timeout** | Tempo máximo para a **request inteira** (connect + send + receive) | 10 segundos |
| **Idle Timeout** | Tempo máximo que uma conexão pode ficar **ociosa** no pool | 30 segundos |
| **Application-Level Timeout** | Timeout definido no nível da aplicação (ex: Time Limiter) | 5 segundos |

---

## Camadas de Timeout

Timeouts devem ser aplicados em **múltiplas camadas**, cada uma com valores decrescentes (de fora para dentro):

```
┌─────────────────────────────────────────────────────┐
│ Load Balancer / API Gateway         timeout: 30s    │
│  ┌─────────────────────────────────────────────┐    │
│  │ Application-Level (Time Limiter)  timeout: 10s│   │
│  │  ┌──────────────────────────────────────┐    │   │
│  │  │ HTTP Client                timeout: 5s│   │   │
│  │  │  ┌───────────────────────────────┐   │   │   │
│  │  │  │ Connection Pool     timeout: 3s│  │   │   │
│  │  │  └───────────────────────────────┘   │   │   │
│  │  └──────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**Regra:** Cada camada interna deve ter timeout **menor** que a camada externa. Se o HTTP client tem timeout de 5s, o Time Limiter não precisa ser menor que 5s (perderia o sentido). O Time Limiter deve ser ligeiramente **maior** para capturar cenários onde o client não respeita o timeout.

---

## Valores Típicos por Contexto

| Tipo de Chamada | Connection Timeout | Read Timeout | Justificativa |
|-----------------|-------------------|-------------|---------------|
| Microsserviço interno (mesmo datacenter) | 1–2s | 3–5s | Latência deve ser baixa; se demorar, algo está errado |
| API externa (terceiros) | 3–5s | 10–15s | Rede pública é mais lenta e imprevisível |
| Banco de dados | 2–3s | 5–10s | Queries lentas devem ser otimizadas, não esperadas |
| Banco de dados (relatórios) | 2–3s | 30–60s | Queries analíticas são naturalmente mais lentas |
| Message broker (produção) | 3s | 5s | Publicar deve ser rápido |
| Serviço de pagamento | 2s | 3–5s | Crítico — falha rápida, fallback, retry |

---

## Diagrama de Decisão

```
Chamada ao serviço
       │
       ▼
  Inicia timer (timeout = X segundos)
       │
       ├───────────────────────────────┐
       │                               │
       ▼                               ▼
  Resposta chegou                Timer expirou
  antes do timeout?              (timeout!)
       │                               │
       │ SIM                           │
       │                               ▼
       ▼                    Cancela a operação
  Processa resposta          Libera recursos
  normalmente                       │
                                    ▼
                           Trata como falha
                           (TimeoutException)
                                    │
                                    ▼
                           Circuit Breaker
                           contabiliza falha
```

---

## Timeout e Cancelamento

Quando um timeout ocorre, **o que acontece com a operação em andamento?**

| Cenário | Comportamento | Risco |
|---------|--------------|-------|
| **Leitura (GET)** | Cancelar é seguro — sem side effects | Nenhum |
| **Escrita (POST/PUT)** | A operação pode já ter sido executada no servidor | Duplicação se houver retry |
| **Chamada a banco** | A query pode continuar executando no banco mesmo após timeout | Desperdício de recursos no banco |

**Importante:** Timeout no cliente **não cancela** a operação no servidor/banco. O servidor só sabe que o cliente desistiu quando tenta enviar a resposta e a conexão está fechada. A query no banco continua executando.

Para mitigar:
- Configure **statement timeout** no banco de dados
- Use **request cancellation** quando disponível
- Garanta **idempotência** para operações de escrita que podem sofrer timeout + retry

---

## Exemplo Conceitual (Pseudocódigo)

```
function callWithTimeout(operation, timeoutDuration):
    future = executeAsync(operation)
    
    try:
        result = future.get(timeout = timeoutDuration)
        return result
    catch TimeoutException:
        future.cancel()  // tenta cancelar
        log.warn("Timeout após {}ms chamando {}", timeoutDuration, operation.name)
        throw ServiceTimeoutException("Operação excedeu o tempo limite")
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Sem timeout | Chamada bloqueia indefinidamente | Defina timeout em **todas** as chamadas remotas |
| Timeout muito curto | Gera falsos positivos (operações válidas cortadas) | Analise o p99 da latência da dependência |
| Timeout muito longo | Não protege contra dependência lenta (120s!) | O timeout deve ser agressivo o suficiente para proteger |
| Timeout só no HTTP client | Não cobre camadas acima | Use timeout em múltiplas camadas |
| Timeout sem métrica | Não sabe quantos timeouts estão ocorrendo | Exponha métricas de timeout e alerte |
| Timeout sem retry | Um timeout transitório causa falha definitiva | Combine com Retry (para erros transitórios) |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Circuit Breaker** | Timeouts são contabilizados como falhas pelo Circuit Breaker. Muitos timeouts → circuito abre. |
| **Retry** | Após timeout, o Retry pode reexecutar a chamada com nova tentativa. |
| **Bulkhead** | Timeout libera recursos mais rápido, ajudando o Bulkhead a não esgotar seu pool. |
| **Rate Limiter** | Se a dependência retorna lentamente por sobrecarga, Rate Limiter ajuda a reduzir o volume. |
| **Backpressure** | Timeouts são um sinal de que o sistema downstream não consegue processar o volume. |

---

## Boas Práticas

1. Defina timeouts **em todas as camadas**: HTTP client, application-level, banco de dados.
2. Cada camada interna deve ter timeout **menor** que a externa.
3. Baseie os valores no **p99 da latência** em operação normal (não no p50).
4. Monitore **taxa de timeouts** por dependência — é um indicador de saúde.
5. Combine com Retry para falhas transitórias e Circuit Breaker para falhas persistentes.
6. Configure **statement timeout** no banco para evitar queries que rodam indefinidamente.
7. Timeouts em operações de escrita exigem **idempotência** — a operação pode ter sido executada no servidor.
8. Diferencie timeouts por tipo de operação: endpoints de consulta (curto) vs. relatórios (longo).

---

## Referências

- Microsoft — [Timeouts Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/)
- Sam Newman — *Building Microservices* (O'Reilly)
