# API Composition

> **Categoria:** Persistência e Processamento de Dados
> **Complementa:** CQRS, API Gateway/BFF, Circuit Breaker
> **Keywords:** API Composition, agregação de dados, consulta distribuída, join entre serviços, orquestração de queries

---

## Problema

O cliente precisa de dados que estão **espalhados em múltiplos microsserviços**. Cada serviço é dono do seu domínio e do seu banco de dados. Fazer N chamadas diretamente do frontend:

- **Aumenta latência** — N roundtrips sequenciais
- **Acopla o frontend** — precisa conhecer cada serviço
- **Complexifica o frontend** — lógica de agregação, tratamento de erros parciais
- **Problemas de rede** — mais chamadas = mais pontos de falha

---

## Solução

Um serviço **compositor** (ou o BFF) agrega dados de múltiplos serviços em uma **única resposta**. O cliente faz uma única chamada; o compositor consulta os serviços necessários, combina os dados e retorna.

---

## Fluxo

```
                    ┌── UserService ──────── (dados do usuário)
                    │
Client ──▶ Compositor ──── OrderService ──── (últimos pedidos)
                    │
                    └── WalletService ───── (saldo)
                    
                    ◀── Resposta agregada (uma única chamada)
```

**Sem compositor:**
```
Client → GET /users/123      → UserService      (roundtrip 1)
Client → GET /orders?user=123 → OrderService     (roundtrip 2)  
Client → GET /wallet/123     → WalletService     (roundtrip 3)
= 3 chamadas, 3 roundtrips, lógica de agregação no client
```

**Com compositor:**
```
Client → GET /dashboard/123  → Compositor
                                 ├→ UserService      ┐
                                 ├→ OrderService      ├ paralelo
                                 └→ WalletService     ┘
                              ← Resposta unificada
= 1 chamada do client
```

---

## Chamadas Paralelas vs. Sequenciais

### Paralelas (recomendado quando possível)

Quando as chamadas aos serviços são **independentes** (uma não depende do resultado da outra):

```
       ┌── UserService ────────── 120ms ──┐
       │                                   │
Start ─┼── OrderService ──── 200ms ───────┼── Agrega ── Responde
       │                                   │   (210ms total)
       └── WalletService ──── 80ms ───────┘
```

**Latência total = latência do serviço mais lento (200ms) + overhead**

### Sequenciais (quando há dependência)

Quando o resultado de uma chamada é necessário para fazer a próxima:

```
Start ── UserService (120ms) ── usa userId ── OrderService (200ms) ── Responde
                                                                      (320ms total)
```

**Latência total = soma das latências**

### Misto

```
Start ─── UserService (120ms) ──┐
                                 ├── usa dados ── EnrichmentService (100ms) ── Responde
       ─── OrderService (200ms) ┘
```

---

## Tratamento de Falhas Parciais

**Cenário:** O UserService e OrderService responderam, mas o WalletService está indisponível.

| Estratégia | Comportamento | Quando usar |
|-----------|--------------|-------------|
| **Falha total** | Retorna erro 503 se qualquer serviço falhar | Todos os dados são essenciais (ex: checkout) |
| **Degradação graceful** | Retorna dados disponíveis + campo nulo/default para o serviço falho | Dados parciais são úteis (ex: dashboard) |
| **Cache fallback** | Retorna último valor cacheado do serviço falho | Dados mudam pouco (ex: perfil do usuário) |

### Exemplo de Degradação Graceful

```
// WalletService falhou → retorna null para wallet
{
  "user": { "id": "123", "name": "João" },
  "lastOrders": [ ... ],
  "wallet": null,          // ← serviço indisponível
  "warnings": [
    "Saldo indisponível temporariamente"
  ]
}
```

---

## Onde colocar o Compositor?

| Opção | Quando usar |
|-------|-------------|
| **Serviço dedicado (Compositor)** | Quando a agregação é complexa ou reutilizada por múltiplos clientes |
| **BFF (Backend for Frontend)** | Quando cada tipo de frontend (mobile, web) precisa de dados diferentes |
| **API Gateway** | Quando a agregação é simples (apenas juntar payloads) |
| **GraphQL** | Quando o cliente quer controlar quais campos retornar |

---

## Exemplo Conceitual (Pseudocódigo)

```
class UserDashboardCompositor:
    userClient
    orderClient
    walletClient
    
    function getDashboard(userId):
        // Chamadas paralelas
        userFuture = async(() -> userClient.findById(userId))
        ordersFuture = async(() -> orderClient.findByUserId(userId))
        walletFuture = async(() -> walletClient.findByUserId(userId))
        
        // Aguarda todas (com timeout individual por client)
        user = userFuture.get()       // obrigatório
        orders = tryGet(ordersFuture) // opcional — fallback para lista vazia
        wallet = tryGet(walletFuture) // opcional — fallback para null
        
        return DashboardResponse(user, orders, wallet)
    
    function tryGet(future):
        try:
            return future.get(timeout = 3s)
        catch TimeoutException, ServiceException:
            log.warn("Serviço indisponível, usando fallback")
            return null  // degradação graceful
```

---

## Performance: Caching

Dados que mudam pouco podem ser **cacheados** pelo compositor:

| Dado | Frequência de Mudança | Cache TTL |
|------|----------------------|-----------|
| Perfil do usuário | Raramente | 5–15 min |
| Últimos pedidos | A cada novo pedido | 30s–1min |
| Saldo da carteira | A cada transação | 10–30s |
| Catálogo de produtos | Poucas vezes por dia | 5–30 min |

**Regra:** Cache TTL curto para dados transacionais, longo para dados estáveis.

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Compositor com lógica de negócio | Viola single responsibility; acoplamento | Compositor só agrega e transforma — sem regras |
| Chamadas sequenciais (sem necessidade) | Latência = soma de todas as chamadas | Use chamadas paralelas quando não há dependência |
| Sem timeout por client | Um serviço lento bloqueia toda a composição | Defina timeout individual por dependência |
| Sem fallback parcial | Um serviço indisponível derruba todo o endpoint | Implemente degradação graceful |
| N+1 problem | Para cada item da lista, faz chamada a outro serviço | Crie endpoint batch no serviço dependente |
| Sem Circuit Breaker | Serviço indisponível consome recursos indefinidamente | Aplique CB em cada client |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **CQRS** | CQRS pré-materializa dados agregados; API Composition consulta em tempo real. CQRS é melhor para alto volume. |
| **BFF** | BFF é um tipo de compositor adaptado para um frontend específico |
| **API Gateway** | Gateway pode fazer composição simples; compositor é melhor para agregação complexa |
| **Circuit Breaker** | Cada client do compositor deve ter seu próprio CB |
| **Timeout** | Cada client deve ter timeout independente |
| **Cache** | Cache no compositor reduz carga nos serviços downstream |

---

## Boas Práticas

1. Use **chamadas paralelas** para reduzir latência (concurrent requests, virtual threads, async).
2. Aplique **Circuit Breaker** em cada client individualmente.
3. Defina **fallbacks parciais** — se um serviço falhar, retorne os dados dos outros.
4. O compositor **não deve ter lógica de negócio** — só agrega e transforma.
5. Considere **caching** para dados que mudam pouco.
6. Defina **timeout individual** por dependência.
7. Evite **N+1 problem** — use endpoints batch quando precisar de dados para listas.
8. Monitore **latência do compositor** — ela revela o serviço mais lento da cadeia.

---

## Referências

- Chris Richardson — *Microservices Patterns* (Manning) — API Composition Pattern
- Microsoft — [Gateway Aggregation Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/gateway-aggregation)
- Sam Newman — *Building Microservices* (O'Reilly) — Capítulo: Splitting the Monolith
