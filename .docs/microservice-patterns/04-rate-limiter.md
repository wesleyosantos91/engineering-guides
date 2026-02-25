# Rate Limiter

> **Categoria:** Resiliência
> **Complementa:** Bulkhead, API Gateway, Circuit Breaker
> **Keywords:** resiliência, rate limiting, throttling, token bucket, sliding window, sobrecarga, proteção contra abuso

---

## Problema

Um consumidor (cliente legítimo, bug em outro serviço ou atacante) envia requisições em excesso, sobrecarregando o serviço. Sem limite, o serviço tenta atender todas as requisições, degrada e eventualmente fica indisponível — afetando **todos** os consumidores.

---

## Solução

Limitar o número de chamadas permitidas em uma **janela de tempo**. Quando o limite é atingido, as requisições excedentes são **rejeitadas imediatamente** (HTTP 429 — Too Many Requests) ou **enfileiradas** com timeout.

---

## Algoritmos de Rate Limiting

### 1. Fixed Window Counter

Divide o tempo em janelas fixas (ex: 1 segundo). Cada janela tem um contador. Quando atinge o limite, rejeita até a próxima janela.

```
Janela 1 (00:00-00:01): ████████░░  8/10 → OK
Janela 2 (00:01-00:02): ██████████  10/10 → LIMIT
Janela 3 (00:02-00:03): ███░░░░░░░  3/10 → OK
```

**Problema:** Burst na fronteira das janelas. Se 10 requests chegam no último 100ms da janela 1 e mais 10 no primeiro 100ms da janela 2, são 20 requests em 200ms.

### 2. Sliding Window Log

Mantém um log de timestamps de cada request. Conta quantas requests existem dentro da janela deslizante.

```
Janela deslizante de 1s (olhando para trás):
  t=0.0: [0.0] → 1 req → OK
  t=0.5: [0.0, 0.5] → 2 req → OK
  t=1.2: [0.5, 1.2] → 2 req → OK (0.0 saiu da janela)
```

**Vantagem:** Sem problema de burst na fronteira. **Desvantagem:** Mais memória (armazena timestamps).

### 3. Sliding Window Counter

Combinação do Fixed Window + Sliding Window. Usa contadores por janela mas aplica interpolação para criar efeito deslizante.

```
Janela anterior: 8 requests
Janela atual: 3 requests (30% decorrida)
Estimativa: 8 × 0.7 + 3 = 8.6 → dentro do limite de 10
```

**Melhor custo-benefício** para a maioria dos cenários.

### 4. Token Bucket

Um "balde" com N tokens. Cada request consome 1 token. Tokens são repostos a uma taxa fixa. Permite **bursts** controlados.

```
Bucket: capacidade=10, replenish=5/s

t=0:   tokens=10  │ 8 requests → tokens=2
t=1:   tokens=7   │ (2 + 5 repostos) → 3 requests → tokens=4
t=2:   tokens=9   │ (4 + 5 repostos, cap 10)
```

**Vantagem:** Permite bursts até a capacidade do bucket. **Ideal para:** APIs públicas.

### 5. Leaky Bucket

Requests entram no "balde". O balde "vaza" a uma taxa fixa. Se o balde está cheio, novas requests são descartadas. Garante **taxa de saída constante**.

```
Bucket: capacidade=10, leak=5/s

Burst de 15 requests:
  10 entram no bucket  │  5 rejeitadas (bucket cheio)
  Processamento: 5/s constantes
```

**Ideal para:** Suavizar tráfego irregular.

---

## Comparação de Algoritmos

| Algoritmo | Burst Handling | Memória | Precisão | Complexidade |
|-----------|---------------|---------|----------|-------------|
| Fixed Window | Ruim (burst na fronteira) | Baixa | Média | Simples |
| Sliding Window Log | Boa | Alta | Alta | Média |
| Sliding Window Counter | Boa | Média | Boa | Média |
| Token Bucket | Permite bursts controlados | Baixa | Boa | Média |
| Leaky Bucket | Suaviza para taxa constante | Baixa | Alta | Simples |

---

## Onde aplicar Rate Limiting

| Camada | O que protege | Exemplo |
|--------|-------------|---------|
| **API Gateway** | Toda a plataforma | 1000 req/min por API key |
| **Serviço (aplicação)** | Endpoints específicos | 50 req/s no endpoint de busca |
| **Banco de dados** | Queries | Máximo de conexões por pool |
| **Mensageria** | Consumers | Prefetch/poll rate limitado |

**Recomendação:** Aplique em **múltiplas camadas** — Gateway para proteção global, serviço para proteção granular.

---

## Rate Limiting por Dimensão

| Dimensão | Exemplo | Quando usar |
|----------|---------|------------|
| **Por IP** | 100 req/min por IP | APIs públicas sem autenticação |
| **Por usuário/API key** | 500 req/min por user | APIs autenticadas |
| **Por tenant** | 10.000 req/min por tenant | SaaS multi-tenant |
| **Por endpoint** | 50 req/s no POST /payments | Proteger operações pesadas |
| **Global** | 50.000 req/min total | Proteger capacidade do serviço |

---

## Resposta quando limitado

O serviço deve retornar **HTTP 429 (Too Many Requests)** com headers informativos:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 1
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1708700400

{
  "type": "https://api.example.com/errors/rate-limit",
  "title": "Limite de requisições excedido",
  "status": 429,
  "detail": "Limite de 100 requisições por segundo excedido. Tente novamente em 1 segundo.",
  "retryAfter": "1s"
}
```

| Header | Significado |
|--------|------------|
| `Retry-After` | Segundos para esperar antes de tentar novamente |
| `X-RateLimit-Limit` | Limite configurado para esta janela |
| `X-RateLimit-Remaining` | Requisições restantes na janela |
| `X-RateLimit-Reset` | Timestamp (epoch) de quando a janela reseta |

---

## Rate Limiting Distribuído

Em ambientes com múltiplas instâncias, o rate limiter precisa de **estado compartilhado** para que o limite seja global (não por instância):

```
Instância A ──┐                    ┌── Rate Limiter Store
Instância B ──┤── consulta/atualiza ──│ (Redis, Memcached, etc.)
Instância C ──┘                    └──
```

Sem estado compartilhado, cada instância tem seu próprio limite. Com 3 instâncias e limite de 100/s, o limite real é 300/s.

---

## Diagrama de Decisão

```
Request chega
     │
     ▼
Identifica a dimensão
(IP, user, tenant, endpoint)
     │
     ▼
Consulta contador no rate limiter
     │
     ├── Dentro do limite ──▶ Processa request
     │                        Incrementa contador
     │
     └── Limite excedido ──▶ Rejeita com 429
                              Retorna Retry-After
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Rate limit só por instância | Limite real = N × instâncias | Use store compartilhado (Redis) |
| Sem Retry-After header | Cliente não sabe quando retentar | Sempre retorne Retry-After |
| Limite igual para todos | API interna limitada como pública | Diferencie limites por tipo |
| Enfileirar em vez de rejeitar | Memória cresce sem controle | Prefira rejeição (timeout-duration: 0) |
| Rate limit sem métricas | Não sabe quantas requests são rejeitadas | Monitore taxa de rejeição |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Bulkhead** | Bulkhead limita concorrência simultânea; Rate Limiter limita requisições por tempo |
| **API Gateway** | Gateway é o local ideal para rate limiting global |
| **Circuit Breaker** | CB protege contra dependência; Rate Limiter protege o próprio serviço |
| **Retry** | Clients que recebem 429 devem respeitar Retry-After |
| **Backpressure** | Rate Limiter é uma forma de backpressure no nível HTTP |

---

## Boas Práticas

1. Rejeite imediatamente quando o limite for atingido — não enfileire (evita consumo de memória).
2. Diferencie limites entre APIs **públicas** e **internas**.
3. Retorne header **Retry-After** na resposta 429.
4. Use **estado compartilhado** (Redis) em ambientes com múltiplas instâncias.
5. Combine Rate Limiting no **Gateway** (proteção global) com Rate Limiting no **serviço** (proteção granular).
6. Monitore a **taxa de rejeição** — muitas rejeições podem indicar que o limite está baixo ou que há abuso.
7. Documente os limites na API — consumidores precisam saber os limites para dimensionar suas chamadas.

---

## Referências

- Stripe — [Rate Limiting](https://stripe.com/blog/rate-limiters)
- Google Cloud — [Rate Limiting Strategies](https://cloud.google.com/architecture/rate-limiting-strategies-techniques)
- Kong — [How to Design a Scalable Rate Limiting Algorithm](https://konghq.com/blog/engineering/how-to-design-a-scalable-rate-limiting-algorithm)
