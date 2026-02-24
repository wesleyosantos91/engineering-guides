# Shadow Deployment (Dark Launch / Traffic Mirroring)

> **Categoria:** Estratégias de Modernização e Deploy
> **Complementa:** Canary Release, Blue-Green Deploy, Strangler Fig
> **Keywords:** shadow deployment, dark launch, traffic mirroring, espelhamento de tráfego, teste em produção, validação silenciosa

---

## Problema

Mesmo com canary release, a nova versão atende **tráfego real** de **usuários reais**. Se houver um bug, esses usuários são afetados — mesmo que sejam apenas 5%.

Além disso:

- Dificuldade de saber se o **novo sistema produz os mesmos resultados** que o antigo
- **Performance** da nova versão sob carga real é desconhecida
- **Migração de legado** (Strangler Fig) sem certeza de que o novo é equivalente

---

## Solução

Copiar (mirror) o tráfego de produção para a nova versão **sem afetar os usuários**. O usuario recebe a resposta do sistema estável; a resposta da nova versão é **descartada** (ou comparada para validação).

```
                    ┌── v1 Stable ────────── Resposta ao Client ✅
Client ──▶ Proxy ──┤
                    └── v2 Shadow ────────── Resposta descartada 🗑️
                                              (ou comparada)
```

**O usuário nunca vê a resposta da versão shadow.** O shadow serve apenas para validação.

---

## Fluxo

```
1. Client envia request ao proxy/gateway

2. Proxy:
   a. Encaminha para v1-stable → retorna resposta ao client
   b. Copia o request para v2-shadow (async)

3. v2-shadow:
   a. Processa o request
   b. Resultado é logado/armazenado para análise
   c. Resultado NÃO é retornado ao client

4. Comparação (offline ou em tempo real):
   a. Compara v1-response vs. v2-response
   b. Registra divergências
   c. Reporta % de respostas iguais/diferentes
```

---

## Diagrama

```
                         ┌──────────────┐
                    ┌───▶│  v1 Stable   │───▶ Response ao Client
                    │    └──────────────┘
┌────────┐    ┌────┴────┐
│ Client │───▶│  Proxy  │
└────────┘    └────┬────┘
                    │    ┌──────────────┐
                    └───▶│  v2 Shadow   │───▶ Response descartada
                   async └──────────────┘         │
                                                  ▼
                                           ┌──────────────┐
                                           │  Comparador  │
                                           │  - diff      │
                                           │  - métricas  │
                                           │  - alertas   │
                                           └──────────────┘
```

---

## Modos de Operação

### 1. Traffic Mirroring (Mirror Puro)

O proxy **copia** o request e envia para o shadow. Geralmente feito de forma **assíncrona** — não impacta a latência do request principal.

```
Proxy:
  // Síncrono — cliente espera
  response = forward(request, target: v1Stable)
  
  // Assíncrono — fire and forget
  async(forward(request, target: v2Shadow))
  
  return response  // sempre do v1
```

### 2. Shadow com Comparação

Ambas as respostas são comparadas automaticamente:

```
Proxy:
  v1Response = forward(request, v1Stable)
  v2Response = forward(request, v2Shadow)
  
  if v1Response.body != v2Response.body:
      log.warn("DIVERGÊNCIA: request={}, v1={}, v2={}", 
               request.id, v1Response.body, v2Response.body)
      metrics.increment("shadow.divergence")
  else:
      metrics.increment("shadow.match")
  
  return v1Response  // sempre retorna v1
```

### 3. Shadow Read-Only

Shadow recebe **apenas reads** (GET). Writes (POST, PUT, DELETE) NÃO são espelhados para evitar efeitos colaterais:

```
Proxy:
  if request.method == "GET":
      mirrorToShadow(request)
  // POST, PUT, DELETE → NÃO mirror
```

---

## Cuidados com Side Effects

**O maior risco** do shadow deployment é **duplicar efeitos colaterais**:

| Operação | Risco se espelhada | Tratamento |
|---------|-------------------|-----------|
| **GET** (leitura) | Nenhum | ✅ Seguro para mirror |
| **POST** (criar pedido) | Cria pedido duplicado | ❌ NÃO mirror ou use banco shadow isolado |
| **PUT** (atualizar) | Modifica dados reais | ❌ NÃO mirror writes |
| **DELETE** | Deleta dados reais | ❌ NUNCA mirror |
| **Enviar email** | Email duplicado | ❌ Mock no shadow |
| **Cobrar pagamento** | Cobrança duplicada | ❌ Mock no shadow |

### Soluções para Side Effects

```
Opção 1: Shadow só para reads (GET)
  Mais seguro. Mirror apenas operações de leitura.

Opção 2: Shadow com banco isolado
  v2-shadow escreve em banco separado — sem afetar dados reais.

Opção 3: Shadow com mocks
  Dependências externas (email, pagamento) são mockadas no shadow.

Opção 4: Shadow com dry-run flag
  v2-shadow recebe header "X-Shadow: true"
  → Processa mas não efetua side effects
```

---

## Métricas de Validação

| Métrica | O que mede |
|---------|-----------|
| **Match rate** | % de respostas iguais entre v1 e v2 |
| **Divergence rate** | % de respostas diferentes |
| **Latency diff** | Diferença de latência entre v1 e v2 |
| **Error rate v2** | Erros no shadow que não ocorrem em v1 |
| **Throughput** | Shadow consegue processar o mesmo volume? |

```
Dashboard:
  Match rate:       98.5% ✅
  Divergence rate:  1.5%  (investigar)
  Latency v1 avg:  45ms
  Latency v2 avg:  52ms  (+15%)
  Error rate v2:   0.02% ✅
```

---

## Exemplo Conceitual (Pseudocódigo)

```
// Proxy com traffic mirroring
class ShadowProxy:
    v1Stable
    v2Shadow
    comparator
    
    function handle(request):
        // 1. Forward para stable (síncrono)
        v1Response = v1Stable.forward(request)
        
        // 2. Mirror para shadow (assíncrono)
        if shouldMirror(request):
            async(() -> {
                try:
                    v2Response = v2Shadow.forward(request)
                    comparator.compare(request, v1Response, v2Response)
                catch:
                    metrics.increment("shadow.error")
            })
        
        // 3. Retorna resposta do stable
        return v1Response
    
    function shouldMirror(request):
        // Apenas reads, ou % do tráfego
        return request.method == "GET" 
               OR (request.method == "POST" AND shadowWritesEnabled)

// Comparador
class ResponseComparator:
    function compare(request, v1Response, v2Response):
        if v1Response.statusCode != v2Response.statusCode:
            logDivergence("status", request, v1Response, v2Response)
            return
        
        if normalize(v1Response.body) != normalize(v2Response.body):
            logDivergence("body", request, v1Response, v2Response)
            return
        
        metrics.increment("shadow.match")
    
    function normalize(body):
        // Remove campos dinâmicos (timestamps, UUIDs) para comparação justa
        return removeFields(body, ["timestamp", "requestId", "traceId"])
```

---

## Shadow no Strangler Fig

Shadow deployment é a ferramenta ideal para validar a migração no **Strangler Fig**:

```
Fase 1: Shadow
  Legado atende → Shadow para novo sistema → Compara resultados
  Meta: 99%+ de match rate

Fase 2: Canary
  Novo sistema atende 5% do tráfego real (após validação via shadow)

Fase 3: Full switch
  Novo sistema atende 100%
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Mirror writes sem isolamento | Efeitos colaterais duplicados (emails, pagamentos) | Shadow só para reads OU banco isolado |
| Comparação sem normalização | Timestamps e IDs dinâmicos causam falsos positivos | Normalize antes de comparar |
| Shadow síncrono | Dobra a latência do request | Mirror assíncrono (fire and forget) |
| Ignorar shadow errors | Shadow falhando sem ninguém saber | Monitore error rate do shadow |
| Shadow sem recursos suficientes | Shadow não aguenta a carga | Dimensione shadow para o tráfego esperado |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Canary Release** | Shadow valida sem risco; Canary com risco controlado. Shadow → Canary → Full. |
| **Blue-Green** | Shadow pode validar o ambiente Green antes do switch |
| **Strangler Fig** | Shadow valida que o novo sistema é equivalente ao legado |
| **API Gateway** | Gateway implementa o mirroring de tráfego |
| **Feature Flags** | Flag controla se o shadow está ativado |

---

## Boas Práticas

1. **Apenas reads** (GET) em mirror por padrão — writes precisam de isolamento.
2. Mirror **assíncrono** — não impacte a latência do request principal.
3. **Normalize** respostas antes de comparar (remova timestamps, IDs dinâmicos).
4. Defina **meta de match rate** (ex: 99%+) antes de avançar para canary.
5. **Dimensione** o shadow para aguentar a carga de produção.
6. Use shadow como **primeira fase** antes do canary (shadow → canary → full).
7. **Monitore** error rate e latência do shadow separadamente.
8. Para writes, use **banco isolado** ou **dry-run mode** no shadow.

---

## Referências

- Istio — [Traffic Mirroring](https://istio.io/latest/docs/tasks/traffic-management/mirroring/)
- Microsoft — [Deployment Stamps Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp)
- Sam Newman — *Building Microservices* (O'Reilly) — Testing in Production
- Cindy Sridharan — [Testing in Production](https://medium.com/@copyconstruct/testing-in-production-the-safe-way-18ca102d0ef1)
