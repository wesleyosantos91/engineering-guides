# Estratégia de Testes e Qualidade — [NOME DO SISTEMA]

> **Versão:** 1.0
> **Última atualização:** 2026-02-23
> **Público-alvo:** Times de engenharia (back-end, front-end, plataforma, QA)
> **Classificação:** Documento normativo — decisões aqui são obrigatórias para novos componentes e fortemente recomendadas para legado.

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Pirâmide de Testes (versão moderna)](#2-pirâmide-de-testes-versão-moderna)
3. [Padrão de Escrita: Cenário → Ação → Resultado](#3-padrão-de-escrita-cenário--ação--resultado)
4. [Estratégia por Camada/Componente](#4-estratégia-por-camadacomponente)
5. [Estratégia para Assíncrono e Resiliência](#5-estratégia-para-assíncrono-e-resiliência)
6. [Dados de Teste e Ambientes](#6-dados-de-teste-e-ambientes)
7. [CI/CD e Gates de Qualidade](#7-cicd-e-gates-de-qualidade)
8. [Métricas e Observabilidade de Testes](#8-métricas-e-observabilidade-de-testes)
9. [Definition of Done (DoD) e Checklists](#9-definition-of-done-dod-e-checklists)
10. [Glossário](#10-glossário)

---

## 1. Visão Geral

### 1.1 Escopo (o que este documento cobre)

- Estratégia de testes **por nível** (pirâmide) e **por componente** (camadas da arquitetura).
- Padrões de escrita, nomeação e organização de testes.
- Critérios de promoção entre níveis de teste.
- Estratégia específica para fluxos assíncronos, resiliência e consistência eventual.
- Gates de qualidade no CI/CD.
- Métricas e observabilidade da suíte de testes.

### 1.2 Não-escopo (o que este documento **não** cobre)

- Testes de performance/carga (documento dedicado: `[LINK_DOC_PERFORMANCE]`).
- Testes de segurança/penetração (documento dedicado: `[LINK_DOC_SECURITY]`).
- Detalhes de configuração de ferramentas específicas (cada time define suas ferramentas).
- Testes exploratórios manuais (cobertos no processo de QA do time).

### 1.3 Princípios Fundamentais

| #  | Princípio                    | Descrição                                                                                                         |
|----|------------------------------|--------------------------------------------------------------------------------------------------------------------|
| P1 | **Feedback rápido**          | A maioria dos testes deve rodar em **segundos**, não minutos. Quanto mais rápido o feedback, mais cedo se corrige. |
| P2 | **Determinismo**             | O mesmo teste, com os mesmos inputs, **sempre** produz o mesmo resultado. Zero tolerância para flakiness crônico.  |
| P3 | **Isolamento**               | Cada teste deve ser independente. Nenhum teste pode depender do estado deixado por outro.                          |
| P4 | **Observabilidade no teste** | Quando um teste falha, a mensagem de erro e os logs devem ser suficientes para diagnosticar **sem** depuração.     |
| P5 | **Custo/Benefício**          | Escrever testes onde o risco é maior. Não buscar 100% de cobertura — buscar **100% de confiança** nas partes críticas. |
| P6 | **Teste como documentação**  | O nome e o corpo do teste devem permitir que qualquer engenheiro entenda o comportamento esperado do sistema.      |
| P7 | **Propriedade coletiva**     | Testes são código de produção. Merecem review, refatoração e manutenção.                                          |

---

## 2. Pirâmide de Testes (versão moderna)

```
          ╱  ╲
         ╱ E2E╲              ← Poucos, lentos, alto custo
        ╱ Smoke ╲
       ╱─────────╲
      ╱ Contrato   ╲         ← Moderados, validam compatibilidade
     ╱───────────────╲
    ╱   Integração     ╲     ← Moderados, validam colaboração
   ╱─────────────────────╲
  ╱      Unitários         ╲  ← Muitos, rápidos, barato
 ╱───────────────────────────╲
```

> **Regra de ouro:** Se um bug pode ser encontrado num nível mais baixo da pirâmide, ele **deve** ser testado nesse nível. Subir de nível somente quando o nível inferior é insuficiente.

### 2.1 Testes Unitários

| Aspecto                      | Detalhe                                                                                            |
|------------------------------|----------------------------------------------------------------------------------------------------|
| **Definição**                | Testa uma unidade de lógica (função, método, classe) em completo isolamento, sem I/O real.         |
| **Que tipo de bug pega**     | Erros de lógica de negócio, cálculos incorretos, validações faltantes, edge cases, regras condicionais. |
| **Custo/Tempo**              | ⬇ Muito baixo. Execução em milissegundos. Sem dependências externas.                              |
| **Critérios de entrada**     | Toda regra de negócio, validação, cálculo, transformação de dados e lógica condicional.            |
| **Critérios de saída**       | Remover quando a regra de negócio subjacente for removida ou o teste se tornar redundante.          |

**Anti-patterns e como evitar:**

| Anti-pattern                                | Problema                                          | Solução                                                  |
|---------------------------------------------|---------------------------------------------------|----------------------------------------------------------|
| Testar getters/setters triviais             | Zero valor, custo de manutenção                   | Testar apenas lógica com comportamento real               |
| Mock de tudo (over-mocking)                 | Testa a configuração dos mocks, não o comportamento | Usar mocks apenas para dependências de I/O               |
| Teste acoplado à implementação              | Quebra ao refatorar sem mudar comportamento        | Testar via interface pública, não detalhes internos       |
| Múltiplos cenários em um único teste        | Difícil diagnosticar qual cenário falhou           | Um cenário por teste; usar testes parametrizados          |
| Asserts genéricos (`assertNotNull`)         | Não valida o comportamento real                    | Asserts específicos sobre o valor/estado esperado         |

### 2.2 Testes de Integração

| Aspecto                      | Detalhe                                                                                              |
|------------------------------|------------------------------------------------------------------------------------------------------|
| **Definição**                | Testa a colaboração entre 2+ componentes reais (ex.: serviço + banco, serviço + fila).               |
| **Que tipo de bug pega**     | Queries incorretas, serialização/deserialização, configuração de conexão, transações, mapeamento ORM. |
| **Custo/Tempo**              | ⬆ Moderado. Requer infraestrutura (containers, bancos in-memory). Segundos a poucos minutos.        |
| **Critérios de entrada**     | Qualquer interação com recurso externo (banco, fila, cache, filesystem).                             |
| **Critérios de saída**       | Quando a integração é removida ou substituída por outra testada separadamente.                        |

**Anti-patterns e como evitar:**

| Anti-pattern                                  | Problema                                       | Solução                                                   |
|-----------------------------------------------|-------------------------------------------------|-----------------------------------------------------------|
| Testar lógica de negócio via integração       | Lento e frágil para validar regras simples      | Mover lógica para unitário, testar apenas a integração    |
| Dados compartilhados entre testes             | Testes acoplados, ordem importa                 | Cada teste cria e limpa seus próprios dados                |
| Banco persistente entre execuções             | Estado residual causa falsos positivos/negativos | Usar containers efêmeros ou truncar antes de cada suíte   |
| Testar toda a stack via integração            | Testes lentos, difíceis de manter               | Usar integração apenas para o "ponto de contato" com I/O  |

### 2.3 Testes de Contrato

| Aspecto                      | Detalhe                                                                                                 |
|------------------------------|---------------------------------------------------------------------------------------------------------|
| **Definição**                | Valida que a interface entre produtor e consumidor (API, evento, mensagem) **não quebra compatibilidade**. |
| **Que tipo de bug pega**     | Campos removidos/renomeados, tipos alterados, mudanças de schema, breaking changes em APIs e eventos.   |
| **Custo/Tempo**              | ⬆ Moderado. Não requer infraestrutura pesada, mas exige orquestração entre repositórios/times.         |
| **Critérios de entrada**     | Toda API pública (REST, gRPC, GraphQL) e todo evento/mensagem publicado para consumidores externos.     |
| **Critérios de saída**       | Quando o contrato (endpoint/evento) é deprecado e todos os consumidores migraram.                       |

**Anti-patterns e como evitar:**

| Anti-pattern                                | Problema                                         | Solução                                                   |
|---------------------------------------------|--------------------------------------------------|-----------------------------------------------------------|
| Contratos definidos apenas pelo produtor    | Consumidor descobre breaking change em produção   | Consumer-driven contracts — consumidor define expectativas |
| Validar apenas status code (APIs)           | Payload pode estar completamente errado           | Validar schema completo (campos, tipos, obrigatoriedade)  |
| Ignorar versionamento de eventos            | Consumidores antigos quebram silenciosamente       | Versionamento explícito no schema do evento               |

### 2.4 Testes E2E / Smoke

| Aspecto                      | Detalhe                                                                                               |
|------------------------------|-------------------------------------------------------------------------------------------------------|
| **Definição**                | Testa um fluxo completo de ponta a ponta, incluindo todas as integrações reais.                       |
| **Que tipo de bug pega**     | Problemas de configuração, infraestrutura, rede, orquestração entre serviços, falhas de deploy.       |
| **Custo/Tempo**              | ⬆⬆ Alto. Lento, frágil, depende de ambiente completo. Minutos a dezenas de minutos.                 |
| **Critérios de entrada**     | Fluxos críticos de negócio (happy path), verificação pós-deploy.                                      |
| **Critérios de saída**       | Quando o fluxo de negócio é removido ou coberto por testes de contrato + integração suficientes.      |

**Anti-patterns e como evitar:**

| Anti-pattern                                 | Problema                                        | Solução                                                  |
|----------------------------------------------|-------------------------------------------------|----------------------------------------------------------|
| Muitos testes E2E                            | Suíte lenta, frágil, custo de manutenção alto   | Máximo [5-10] cenários E2E por serviço                   |
| E2E como substituto de unitário/integração   | Feedback lento, difícil diagnosticar falhas      | E2E apenas para "fumaça" — happy path crítico            |
| Dados hardcoded no ambiente                  | Quebra quando ambiente muda                      | Setup/teardown dinâmico ou dados sintéticos por execução |

### 2.5 Tabela Resumo Comparativa

| Dimensão                | Unitário          | Integração              | Contrato              | E2E / Smoke            |
|-------------------------|-------------------|--------------------------|------------------------|------------------------|
| **Escopo**              | 1 unidade         | 2+ componentes reais     | Interface entre partes | Fluxo ponta a ponta    |
| **Velocidade**          | ms                | s – poucos min           | s                      | min – dezenas de min   |
| **Infraestrutura**      | Nenhuma           | Containers / in-memory   | Broker de contratos    | Ambiente completo      |
| **Isolamento**          | Total             | Parcial                  | Parcial                | Nenhum                 |
| **Custo manutenção**    | Baixo             | Médio                    | Médio                  | Alto                   |
| **Confiabilidade**      | Altíssima         | Alta                     | Alta                   | Média (flaky)          |
| **Proporção ideal**     | ~70%              | ~20%                     | ~5-8%                  | ~2-5%                  |
| **Roda no PR**          | ✅ Sempre          | ✅ Subconjunto rápido     | ✅ Se alterou contrato  | ❌ Apenas pós-deploy    |
| **Encontra bug de…**    | Lógica/Cálculo    | I/O / Mapeamento / Query | Compatibilidade        | Config / Infra / Rede  |

---

## 3. Padrão de Escrita: Cenário → Ação → Resultado

### 3.1 O Padrão AAA / Given-When-Then

Os dois padrões são **equivalentes** e devem ser usados de forma intercambiável conforme a preferência do time:

| Fase      | AAA (técnico) | GWT (comportamental)  | Responsabilidade                                         |
|-----------|---------------|-----------------------|-----------------------------------------------------------|
| **Setup** | Arrange       | Given                 | Preparar dados, stubs, estado inicial — o **mínimo** necessário |
| **Ação**  | Act           | When                  | Executar **uma única** operação / comportamento           |
| **Verificação** | Assert  | Then                  | Validar **um aspecto** do resultado                       |

### 3.2 Regras de Ouro

| Regra | Descrição                                                                                         |
|-------|---------------------------------------------------------------------------------------------------|
| R1    | **Um "Ato" por teste.** Se há 2 ações, são 2 testes.                                             |
| R2    | **Asserts focados.** Validar um comportamento por assert. Múltiplos asserts somente se validam o mesmo aspecto. |
| R3    | **Dados mínimos.** Criar apenas os dados estritamente necessários para o cenário. Usar builders/factories. |
| R4    | **Sem lógica no teste.** Proibido `if`, `for`, `try/catch` dentro de testes. Teste deve ser linear. |
| R5    | **Nome descritivo.** O nome do teste deve descrever o cenário, a ação e o resultado esperado.     |
| R6    | **Setup explícito.** Preferir setup visível no próprio teste, não escondido em fixtures globais.  |

### 3.3 Convenção de Nomeação

```
[unidadeSobTeste]_[cenário]_[resultadoEsperado]
```

**Exemplos:**

```
calcularDesconto_clienteVIP_aplicaDesconto20Porcento
validarEmail_formatoInvalido_lancaErroDeValidacao
salvarPedido_produtosValidos_persisteComStatusPendente
publicarEvento_topicIndisponivel_enfileiraParaRetry
```

> **Alternativa aceita:** `deve_[resultado]_quando_[cenário]` — escolha **um** padrão por repositório e mantenha consistência.

### 3.4 Exemplos em Pseudocódigo

#### Exemplo 1 — Teste unitário de regra de negócio (sucesso)

```
TESTE: calcularDesconto_clienteVIP_aplicaDesconto20Porcento

  // Arrange
  cliente = criarCliente(tipo: VIP, ativo: verdadeiro)
  pedido  = criarPedido(valorBruto: 100.00)
  calculadora = new CalculadoraDesconto()

  // Act
  resultado = calculadora.calcular(cliente, pedido)

  // Assert
  VERIFICAR resultado.valorDesconto == 20.00
  VERIFICAR resultado.valorFinal   == 80.00
```

**Por que é unitário:** Não há I/O. A `CalculadoraDesconto` opera apenas sobre os objetos recebidos. Roda em milissegundos.

#### Exemplo 2 — Teste unitário validando erro/exceção (falha esperada)

```
TESTE: validarTransferencia_saldoInsuficiente_lancaErroNegocio

  // Arrange
  contaOrigem   = criarConta(saldo: 50.00)
  valorTransfer  = 100.00
  validador      = new ValidadorTransferencia()

  // Act + Assert
  VERIFICAR_QUE_LANÇA ErroSaldoInsuficiente QUANDO:
    validador.validar(contaOrigem, valorTransfer)

  // Assert adicional (opcional — validar detalhes do erro)
  VERIFICAR erro.codigo    == "SALDO_INSUFICIENTE"
  VERIFICAR erro.mensagem CONTÉM "50.00"
```

**Por que é unitário:** Valida uma regra de negócio pura (saldo insuficiente). O `ValidadorTransferencia` não depende de banco ou serviço externo.

#### Exemplo 3 — Teste de integração de persistência

```
TESTE: salvarPedido_dadosValidos_persisteNoBancoCorretamente

  // Arrange (Setup: banco efêmero já provisionado pelo framework de teste)
  repositorio = obterRepositorioDePedidos()  // conectado ao banco real/container
  pedido = criarPedido(
    cliente:  "cliente-123",
    produtos: [{ sku: "ABC", qtd: 2, preco: 25.00 }],
    status:   PENDENTE
  )

  // Act
  idPersistido = repositorio.salvar(pedido)

  // Assert
  pedidoRecuperado = repositorio.buscarPorId(idPersistido)
  VERIFICAR pedidoRecuperado != NULO
  VERIFICAR pedidoRecuperado.cliente  == "cliente-123"
  VERIFICAR pedidoRecuperado.status   == PENDENTE
  VERIFICAR pedidoRecuperado.produtos.tamanho == 1
  VERIFICAR pedidoRecuperado.produtos[0].sku  == "ABC"

  // Teardown (implícito: container descartado ao fim da suíte, OU transação revertida)
```

**Por que é integração:** Há I/O real com o banco de dados. Valida que o mapeamento ORM, os tipos de colunas e as queries funcionam corretamente.

#### Exemplo 4 — Teste de contrato para payload (compatibilidade para trás)

```
TESTE: contratoPedidoCriado_v1_mantemCamposObrigatorios

  // Arrange
  schemaEsperado = carregarSchema("pedido-criado-v1.schema")
  // O schema define: { pedidoId: string, clienteId: string, valorTotal: numero, criadoEm: timestamp }

  eventoAtual = gerarEventoPedidoCriado(
    pedidoId:   "ped-001",
    clienteId:  "cli-123",
    valorTotal: 150.00,
    criadoEm:   "2026-01-15T10:30:00Z"
  )

  // Act
  resultadoValidacao = validarContraSchema(eventoAtual, schemaEsperado)

  // Assert
  VERIFICAR resultadoValidacao.valido == verdadeiro
  VERIFICAR resultadoValidacao.erros  ESTÁ VAZIO

  // Verificação adicional: campos obrigatórios presentes
  VERIFICAR eventoAtual CONTÉM CAMPO "pedidoId"
  VERIFICAR eventoAtual CONTÉM CAMPO "clienteId"
  VERIFICAR eventoAtual CONTÉM CAMPO "valorTotal"
  VERIFICAR eventoAtual CONTÉM CAMPO "criadoEm"
```

**Por que é contrato:** Não testa lógica nem I/O. Valida que o **formato** do evento produzido é compatível com o schema que os consumidores esperavam. Se alguém remover um campo obrigatório, este teste falha antes de chegar em produção.

---

## 4. Estratégia por Camada/Componente

### 4.1 Modelo de Camadas (genérico)

```
┌──────────────────────────────────────────────────────────┐
│                  Bordas Externas                         │
│  (APIs públicas, UI, Consumers de fila, Schedulers)      │
├──────────────────────────────────────────────────────────┤
│              Adaptadores / Integrações                   │
│  (Repositórios, Clients HTTP, Producers de fila, Cache)  │
├──────────────────────────────────────────────────────────┤
│            Aplicação / Orquestração                      │
│  (Casos de uso, coordenação entre serviços e domínio)    │
├──────────────────────────────────────────────────────────┤
│              Domínio / Regra de Negócio                  │
│  (Entidades, Value Objects, Validações, Cálculos)        │
└──────────────────────────────────────────────────────────┘
```

### 4.2 Domínio / Regra de Negócio

| Aspecto               | Detalhe                                                                           |
|------------------------|-----------------------------------------------------------------------------------|
| **O que testar**       | Cálculos, validações, regras condicionais, state machines, transformações de dados |
| **O que NÃO testar aqui** | Persistência, serialização, chamadas HTTP, configuração de infra                |
| **Tipo de teste**      | ✅ **Unitário** (100% dos cenários)                                                |
| **Exemplos**           | Cálculo de desconto, validação de CPF/CNPJ, regras de transição de status         |

**Diretriz:** Esta camada deve ter a **maior densidade** de testes unitários. Toda ramificação lógica (`if`, `switch`, `match`) deve ter pelo menos um teste para cada caminho.

### 4.3 Aplicação / Orquestração

| Aspecto               | Detalhe                                                                           |
|------------------------|-----------------------------------------------------------------------------------|
| **O que testar**       | Fluxo de orquestração, ordem de chamadas, tratamento de erros, transações         |
| **O que NÃO testar aqui** | Lógica de negócio pura (pertence ao domínio), detalhes de I/O                  |
| **Tipo de teste**      | ✅ **Unitário** (com mocks de dependências) + ✅ **Integração** (fluxo completo)   |
| **Exemplos**           | "Ao criar pedido, salva no banco, publica evento e retorna resposta"              |

**Diretriz:** Testar unitariamente a **orquestração** (ex.: "se o save falha, o evento NÃO é publicado"). Testar via integração quando precisar garantir que a transação realmente funciona.

### 4.4 Adaptadores / Integrações

| Aspecto               | Detalhe                                                                            |
|------------------------|------------------------------------------------------------------------------------|
| **O que testar**       | Queries/comandos de banco, serialização/deserialização, mapeamento, retries, timeouts |
| **O que NÃO testar aqui** | Regras de negócio, lógica de orquestração                                       |
| **Tipo de teste**      | ✅ **Integração** (com infra real/container) + ✅ **Contrato** (para APIs/eventos)  |
| **Exemplos**           | Query complexa retorna dados corretos, client HTTP lida com 404/500               |

**Diretriz:** Usar containers efêmeros para banco/cache/fila. Para clients HTTP de terceiros, usar **stubs** que simulam comportamento real (incluindo erros).

### 4.5 Bordas Externas

| Aspecto               | Detalhe                                                                           |
|------------------------|-----------------------------------------------------------------------------------|
| **O que testar**       | Deserialização de request, validações de entrada, status codes, headers, paginação |
| **O que NÃO testar aqui** | Lógica de negócio, queries de banco                                             |
| **Tipo de teste**      | ✅ **Integração** (HTTP) + ✅ **Contrato** (schema) + ✅ **E2E/Smoke** (fluxo crítico) |
| **Exemplos**           | POST com body inválido retorna 400, header de autenticação ausente retorna 401    |

**Diretriz:** Testar a **borda** (parsing, validação, serialização), não a lógica por trás. A lógica já foi testada nas camadas inferiores.

### 4.6 Matriz Componente × Tipo de Teste

| Componente / Camada               | Unit | Integration | Contract | E2E/Smoke |
|-------------------------------------|:----:|:-----------:|:--------:|:---------:|
| Regras de negócio / Cálculos        | ✅✅  |             |          |           |
| Validações de domínio               | ✅✅  |             |          |           |
| State machines / Transições         | ✅✅  |             |          |           |
| Orquestração (caso de uso)          | ✅   | ✅           |          |           |
| Transações                          |      | ✅✅         |          |           |
| Queries de banco                    |      | ✅✅         |          |           |
| Mapeamento ORM                      |      | ✅✅         |          |           |
| Serialização/Deserialização         | ✅   | ✅           | ✅        |           |
| Client HTTP (API externa)           |      | ✅ (stub)    | ✅        |           |
| Producer de evento                  |      | ✅           | ✅✅      |           |
| Consumer de evento                  |      | ✅✅         | ✅        |           |
| Validação de entrada (API)          | ✅   | ✅           |          |           |
| Autenticação / Autorização          |      | ✅           |          | ✅        |
| Fluxo crítico ponta a ponta        |      |             |          | ✅✅      |
| Health check / Readiness            |      |             |          | ✅        |

> **Legenda:** ✅ = recomendado · ✅✅ = obrigatório / alta prioridade · (vazio) = não aplicável ou baixo valor

---

## 5. Estratégia para Assíncrono e Resiliência

### 5.1 O que Testar em Fluxos Assíncronos

| Cenário                         | O que validar                                                | Tipo de teste       |
|---------------------------------|--------------------------------------------------------------|---------------------|
| **Idempotência**                | Processar a mesma mensagem 2+ vezes produz o mesmo resultado | Integração          |
| **Duplicidade**                 | Mensagem duplicada não gera efeito colateral duplicado       | Integração          |
| **Ordenação**                   | Sistema se comporta corretamente com mensagens fora de ordem | Unitário + Integração |
| **Retries**                     | Após falha transitória, retry processa com sucesso           | Integração          |
| **DLQ (Dead Letter Queue)**     | Após N retries falhando, mensagem vai para DLQ               | Integração          |
| **Backoff exponencial**         | Intervalo entre retries cresce conforme configuração         | Unitário (lógica)   |
| **Timeouts**                    | Operação que excede timeout é interrompida e tratada         | Integração          |
| **Consistência eventual**       | Após processamento assíncrono, estado final é consistente    | Integração          |

### 5.2 Padrões para Testar Idempotência

```
TESTE: processarPagamento_mensagemDuplicada_naoCobraDuasVezes

  // Arrange
  mensagem = criarMensagemPagamento(pedidoId: "ped-001", valor: 100.00)
  processador = obterProcessadorPagamento()

  // Act — processar a mesma mensagem 2 vezes
  processador.processar(mensagem)
  processador.processar(mensagem)  // duplicada

  // Assert
  pagamentos = buscarPagamentosPorPedido("ped-001")
  VERIFICAR pagamentos.tamanho == 1         // apenas 1 pagamento criado
  VERIFICAR pagamentos[0].valor == 100.00   // valor correto
```

### 5.3 Simulação de Falhas sem Flakiness

| Estratégia                           | Como funciona                                                                  | Quando usar                         |
|--------------------------------------|--------------------------------------------------------------------------------|--------------------------------------|
| **Stub de dependência**              | Substituir o client real por um que retorna erro controlado                     | Timeout, 500, 503, connection refused |
| **Proxy de falha programável**       | Interceptar chamadas de rede e injetar latência/erro                           | Testar circuit breaker, retries      |
| **Clock determinístico**             | Substituir relógio do sistema por um controlável no teste                      | Testar TTL, expiração, backoff       |
| **Fila in-memory**                   | Usar fila embarcada com controle de quando entregar mensagens                  | Testar ordenação, duplicidade        |
| **Trigger de falha por configuração**| Flag que força o componente a falhar em cenário específico                     | Testar fallback, DLQ                 |

**Regra:** Nunca usar `sleep()` / `wait(tempo_fixo)` para aguardar processamento assíncrono. Usar **polling com timeout** ou **mecanismo de callback/notificação**.

```
// ❌ ERRADO — flaky, lento
processar(mensagem)
esperar(3000)  // sleep 3 segundos
verificar(resultado)

// ✅ CORRETO — determinístico
processar(mensagem)
aguardarAté(
  condição: () => buscarResultado(id) != NULO,
  timeoutMaximo: 10_segundos,
  intervaloPolling: 100_milissegundos
)
verificar(resultado)
```

### 5.4 Validação de Efeitos Colaterais e Consistência Eventual

| Padrão                                  | Descrição                                                                         |
|-----------------------------------------|-----------------------------------------------------------------------------------|
| **Assert com polling**                  | Verificar resultado com retry até convergir ou timeout                             |
| **Verificar todos os efeitos**          | Se a operação gera evento + persiste + notifica, verificar **todos** os efeitos   |
| **Verificar ausência de efeito**        | Em caso de erro/duplicidade, verificar que o efeito colateral **não** ocorreu     |
| **Snapshot antes/depois**               | Capturar estado do sistema antes e depois da operação e comparar o delta          |

---

## 6. Dados de Teste e Ambientes

### 6.1 Estratégia de Test Data

| Estratégia               | Quando usar                                     | Cuidados                                              |
|--------------------------|-------------------------------------------------|-------------------------------------------------------|
| **Builders / Factories** | Unitários e integração — criar objetos de domínio| Defaults sensatos; sobrescrever apenas o relevante     |
| **Fixtures (JSON/YAML)** | Dados complexos, snapshots, contratos            | Versionar junto ao código; não compartilhar entre suítes |
| **Seeds (scripts DDL/DML)** | Popular banco para testes de integração       | Executar no setup da suíte; limpar no teardown         |
| **Dados sintéticos**     | Testes E2E, staging, dados sensíveis             | Gerar via algoritmo determinístico; nunca usar dados reais de produção |
| **Copy de produção (anonimizado)** | Testes de carga, validação de migração  | Obrigatório anonimização; considerar volume             |

### 6.2 Regras de Dados

| Regra  | Descrição                                                                                              |
|--------|--------------------------------------------------------------------------------------------------------|
| D1     | **Nunca** usar dados de produção reais em ambientes de teste (LGPD/GDPR/compliance).                  |
| D2     | Cada teste deve criar **seus próprios dados**. Proibido depender de dados pré-existentes no banco.     |
| D3     | Usar **identificadores únicos** por teste (UUID, sufixo timestamp) para evitar colisão em paralelismo. |
| D4     | Dados de teste são **código** — versionados, revisados e mantidos.                                     |
| D5     | Preferir **builders com defaults** sobre fixtures estáticas — builders são mais flexíveis e legíveis.  |

### 6.3 Isolamento e Paralelismo

| Mecanismo                           | Descrição                                                               |
|--------------------------------------|-------------------------------------------------------------------------|
| **Container efêmero por suíte**      | Cada suíte de integração sobe seu próprio container de banco/cache      |
| **Schema/namespace isolado**         | Cada execução paralela usa um schema/namespace diferente                |
| **Transação revertida**              | Cada teste roda dentro de uma transação que faz rollback ao final       |
| **Nomes com sufixo único**           | Filas, tópicos e tabelas temporárias usam sufixo único por execução     |
| **Clock determinístico**             | Relógio injetável permite testar lógica temporal sem depender de `now()` |

### 6.4 Ambientes

| Ambiente     | Propósito                                           | Tipos de teste que rodam                     |
|--------------|------------------------------------------------------|----------------------------------------------|
| **Local**    | Desenvolvimento; feedback imediato                   | Unitários + integração com containers locais |
| **Pipeline** | CI/CD; validação automatizada                        | Unitários + Integração + Contrato            |
| **Staging**  | Réplica de produção; validação pré-release           | Smoke + E2E + Testes de contrato cruzados    |
| **Produção** | Verificação pós-deploy                               | Smoke (health checks, canary)                |

**Diferença entre Smoke e E2E:**

| Aspecto            | Smoke                                      | E2E                                            |
|--------------------|---------------------------------------------|------------------------------------------------|
| **Escopo**         | Verificação mínima: "o sistema está no ar?" | Fluxo completo de negócio                      |
| **Profundidade**   | Health checks, ping nos endpoints críticos   | Criação, leitura, processamento, notificação   |
| **Quando rodar**   | Pós-deploy (sempre)                         | Staging (pré-release) ou nightly               |
| **Duração**        | Segundos                                    | Minutos                                        |
| **Tolerância a falha** | Zero — se smoke falha, rollback imediato | Média — pode haver falha de ambiente           |

---

## 7. CI/CD e Gates de Qualidade

### 7.1 Pipeline Recomendado

```
┌─────────────────────────────────────────────────────────────────────┐
│                           PIPELINE CI/CD                            │
├─────────────┬───────────────────────────────────────────────────────┤
│             │                                                       │
│   PR        │  ┌──────────┐  ┌──────────────┐  ┌───────────────┐   │
│  (a cada    │  │ Lint +   │→ │ Unitários    │→ │ Integração    │   │
│   push)     │  │ Compile  │  │ (100%)       │  │ (subset fast) │   │
│             │  └──────────┘  └──────────────┘  └───────────────┘   │
│             │       ↓ GATE: compile ok + unit pass + coverage ≥ X  │
├─────────────┼───────────────────────────────────────────────────────┤
│             │                                                       │
│   Merge     │  ┌──────────┐  ┌──────────────┐  ┌───────────────┐   │
│  (main)     │  │ Unitários│→ │ Integração   │→ │ Contratos     │   │
│             │  │ (100%)   │  │ (completa)   │  │               │   │
│             │  └──────────┘  └──────────────┘  └───────────────┘   │
│             │       ↓ GATE: all pass + no breaking contracts       │
├─────────────┼───────────────────────────────────────────────────────┤
│             │                                                       │
│   Nightly   │  ┌──────────┐  ┌──────────────┐  ┌───────────────┐   │
│  (schedule) │  │ Suíte    │→ │ Mutation     │→ │ E2E           │   │
│             │  │ completa │  │ testing      │  │ (staging)     │   │
│             │  └──────────┘  └──────────────┘  └───────────────┘   │
│             │       ↓ GATE: mutation score ≥ Y + e2e pass          │
├─────────────┼───────────────────────────────────────────────────────┤
│             │                                                       │
│  Pós-deploy │  ┌──────────┐  ┌──────────────┐                      │
│  (produção) │  │ Smoke    │→ │ Health       │                      │
│             │  │          │  │ checks       │                      │
│             │  └──────────┘  └──────────────┘                      │
│             │       ↓ GATE: smoke pass → promote; fail → rollback  │
└─────────────┴───────────────────────────────────────────────────────┘
```

### 7.2 Gates de Qualidade

| Gate                          | Valor mínimo                     | Onde aplica               | Ação se falhar              |
|-------------------------------|----------------------------------|----------------------------|-----------------------------|
| **Cobertura de linhas**       | ≥ [80]% por módulo               | PR + Merge                | Bloqueia merge              |
| **Cobertura de branches**     | ≥ [70]% por módulo               | PR + Merge                | Bloqueia merge              |
| **Mutation score**            | ≥ [60]% em código de domínio     | Nightly                   | Notificação + ticket        |
| **Tempo máximo do pipeline**  | ≤ [10] min (PR) / ≤ [30] min (merge) | Todos                | Investigar e otimizar       |
| **Taxa de flake**             | ≤ [2]% da suíte                  | Semanal (métrica)          | Quarentena do teste         |
| **Testes E2E/Smoke**          | 100% pass                        | Pós-deploy                | Rollback automático         |
| **Contratos**                 | 0 breaking changes não versionadas| Merge                    | Bloqueia merge              |

> **Nota:** Os valores entre colchetes `[X]` são sugestões. Cada time deve calibrar com base na maturidade e criticidade do serviço.

### 7.3 Política de Quarentena para Testes Flakey

| Etapa | Ação                                                                                      |
|-------|-------------------------------------------------------------------------------------------|
| 1     | Teste identificado como flaky (falha intermitente ≥ 2 vezes em 7 dias sem mudança de código) |
| 2     | Teste é movido para suíte de **quarentena** (roda separado, não bloqueia pipeline)         |
| 3     | Um **owner** é atribuído e um **ticket** é criado com prazo máximo de **[5] dias úteis**   |
| 4     | Se não corrigido no prazo: **escalar** para tech lead e considerar remoção                 |
| 5     | Teste corrigido: mover de volta para suíte principal e monitorar por 3 dias                |

**Regras:**

- Máximo de **[10]** testes em quarentena simultânea por serviço.
- Se o limite é atingido, **nenhuma feature nova** entra até resolver pelo menos metade.
- Dashboard de quarentena visível para todo o time.

---

## 8. Métricas e Observabilidade de Testes

### 8.1 Métricas Obrigatórias

| Métrica                          | Fórmula / Definição                                                    | Meta               | Frequência   |
|----------------------------------|-------------------------------------------------------------------------|---------------------|--------------|
| **Tempo da suíte (PR)**         | Tempo total desde o trigger até resultado final                         | ≤ [10] min          | Todo PR      |
| **Tempo da suíte (merge)**      | Tempo total da suíte completa na branch principal                       | ≤ [30] min          | Todo merge   |
| **Taxa de flake**               | (Nº testes flakey / Nº total de testes) × 100                          | ≤ [2]%              | Semanal      |
| **Cobertura por risco**         | % de cobertura nas classes/módulos classificados como críticos          | ≥ [90]%             | Todo merge   |
| **Defeitos escapados**          | Nº de bugs encontrados em produção que deveriam ter sido pegos em teste | Tendência decrescente | Mensal      |
| **Mutation score (domínio)**    | % de mutantes mortos / total de mutantes gerados                        | ≥ [60]%             | Nightly      |
| **Testes em quarentena**        | Nº de testes atualmente na suíte de quarentena                          | ≤ [10]              | Contínuo     |
| **Tempo médio de correção de flake** | Média de dias entre identificação e correção de teste flaky       | ≤ [5] dias úteis    | Mensal       |

### 8.2 Observabilidade nos Testes

| Prática                               | Descrição                                                                         |
|----------------------------------------|-----------------------------------------------------------------------------------|
| **Logs estruturados no teste**         | Cada teste deve gerar logs com `testName`, `testId` e contexto suficiente          |
| **Trace ID no teste de integração**    | Propagar trace ID nos testes de integração para rastrear chamadas entre componentes |
| **Screenshot/Snapshot em falha**       | Testes E2E devem capturar estado visual ou dump do sistema ao falhar               |
| **Diff de estado**                     | Em falhas de integração, logar o estado esperado vs. obtido em formato diff        |
| **Métricas de tempo por teste**        | Identificar testes lentos (> [2]s para unitário, > [30]s para integração)          |
| **Rerun automático com log enriquecido** | Na primeira falha, re-executar com nível de log aumentado para capturar mais contexto |

### 8.3 Dashboard Recomendado

```
┌──────────────────────────────────────────────────────────────┐
│               DASHBOARD DE QUALIDADE DE TESTES               │
├──────────────┬──────────────┬──────────────┬─────────────────┤
│  Pipeline    │  Cobertura   │  Flakiness   │  Escapados      │
│  Duration    │  (por risco) │  Rate        │  (último mês)   │
│  ■■■■□ 8min  │  ■■■■■ 92%   │  ■□□□□ 1.2%  │  ■□□□□ 2 bugs   │
├──────────────┴──────────────┴──────────────┴─────────────────┤
│  Testes em Quarentena: 3/10          Mutation Score: 68%     │
│  Suíte mais lenta: [MÓDULO_X] (4.2min)                      │
│  Último E2E: ✅ PASS (staging, 12min atrás)                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 9. Definition of Done (DoD) e Checklists

### 9.1 Definition of Done — Features

Uma feature é considerada **done** quando **todos** os itens abaixo são atendidos:

| #   | Critério                                                                                   |
|-----|--------------------------------------------------------------------------------------------|
| D1  | Código implementado, revisado (code review) e aprovado.                                    |
| D2  | Testes **unitários** cobrem toda nova lógica de negócio e ramificações.                    |
| D3  | Testes de **integração** cobrem toda nova interação com banco, fila, cache ou API externa.  |
| D4  | Testes de **contrato** atualizados se houve alteração em API pública ou schema de evento.   |
| D5  | Cobertura de linhas ≥ [80]% e de branches ≥ [70]% no módulo afetado.                      |
| D6  | Todos os testes passam na pipeline (zero falhas).                                           |
| D7  | Nenhum alerta novo de segurança estática (SAST) ou dependência vulnerável.                  |
| D8  | Documentação atualizada (API docs, README, ADR se decisão significativa).                   |
| D9  | Métricas/alertas de observabilidade configurados para o novo fluxo.                         |
| D10 | Smoke test pós-deploy atualizado se novo fluxo crítico.                                     |

### 9.2 Definition of Done — Bugs

| #   | Critério                                                                    |
|-----|-----------------------------------------------------------------------------|
| B1  | Bug reproduzido com teste que falha **antes** do fix.                       |
| B2  | Fix implementado e teste agora passa.                                       |
| B3  | Teste adicionado permanentemente à suíte (não removido após fix).           |
| B4  | Root cause documentado no ticket.                                           |
| B5  | Verificado se o mesmo padrão de bug existe em outros módulos (busca ampla). |

### 9.3 Checklist de Pull Request

```
## Checklist de PR — Qualidade

### Testes
- [ ] Testes unitários adicionados/atualizados para nova lógica
- [ ] Testes de integração adicionados/atualizados para novas interações com I/O
- [ ] Testes de contrato atualizados (se API pública ou evento alterado)
- [ ] Cenários de erro/exceção testados (inputs inválidos, timeout, indisponibilidade)
- [ ] Casos de borda testados (lista vazia, null, valor limite, overflow)
- [ ] Nomes dos testes seguem convenção: [unidade]_[cenário]_[resultado]

### Qualidade
- [ ] Cobertura ≥ [80]% linhas, ≥ [70]% branches no módulo
- [ ] Sem lógica no teste (sem if/for/try-catch)
- [ ] Cada teste tem um único "Act"
- [ ] Dados de teste isolados (sem compartilhamento entre testes)
- [ ] Sem sleep/wait fixo em testes assíncronos

### Segurança e Observabilidade
- [ ] Sem credenciais/secrets no código de teste
- [ ] Logs/métricas adicionados para novos fluxos
- [ ] Alertas configurados para novos pontos de falha
```

### 9.4 Anti-Patterns e Sinais Vermelhos

| Sinal Vermelho                                     | Por que é problema                                         | Ação corretiva                                        |
|----------------------------------------------------|------------------------------------------------------------|-------------------------------------------------------|
| Teste que só passa se rodar sozinho                | Dependência oculta de estado compartilhado                  | Isolar dados; usar container/schema por teste         |
| Teste que usa `sleep()` para esperar resultado     | Flaky e lento                                              | Usar polling com timeout ou callback                  |
| Teste com mais de 50 linhas de setup               | Difícil de entender e manter                               | Extrair builder/factory; usar fixtures reutilizáveis  |
| Cobertura alta mas mutation score baixo             | Testes passam pelo código mas não validam comportamento     | Adicionar asserts significativos; usar mutation testing |
| Teste que valida implementação (não comportamento) | Quebra em refatoração sem mudar funcionalidade              | Testar via interface pública                          |
| Mock que retorna mock que retorna mock              | Teste impossível de entender; acoplado à implementação      | Redesign do código sob teste; reduzir aninhamento     |
| Teste ignorado/desabilitado há > 30 dias           | Dívida técnica esquecida                                   | Corrigir ou deletar; política de prazo máximo         |
| Suíte que leva > 15 min no PR                      | Feedback loop lento, desenvolvedores ignoram resultados     | Paralelizar; mover testes lentos para merge/nightly   |
| Flake rate > 5%                                    | Desenvolvedores perdem confiança na suíte                   | Quarentena agressiva; investir em correção            |
| Testes E2E > 20 por serviço                        | Manutenção proibitiva; suíte muito lenta                    | Reduzir para [5-10] cenários críticos de happy path   |

---

## 10. Glossário

| Termo                        | Definição                                                                                                |
|------------------------------|----------------------------------------------------------------------------------------------------------|
| **AAA**                      | Arrange / Act / Assert — padrão de estrutura de teste onde se prepara, executa e verifica.               |
| **Backoff exponencial**      | Estratégia de retry onde o intervalo entre tentativas cresce exponencialmente (ex.: 1s, 2s, 4s, 8s).     |
| **Breaking change**          | Alteração que torna incompatível a interface entre produtor e consumidor (ex.: remover campo obrigatório). |
| **Builder (test)**           | Objeto que facilita a criação de instâncias complexas para teste, com valores default sensatos.           |
| **Circuit breaker**          | Padrão de resiliência que interrompe chamadas a uma dependência após N falhas consecutivas.               |
| **Consistência eventual**    | Modelo onde os dados podem estar temporariamente inconsistentes, mas convergem para o estado correto.     |
| **Consumer-driven contract** | Teste de contrato onde o consumidor define suas expectativas e o produtor as valida.                      |
| **DLQ (Dead Letter Queue)**  | Fila para onde mensagens são enviadas após exceder o número máximo de retries.                            |
| **Defeito escapado**         | Bug que chegou a produção sem ser detectado pelos testes.                                                 |
| **Determinismo**             | Propriedade de um teste que sempre produz o mesmo resultado com os mesmos inputs.                         |
| **E2E (End-to-End)**         | Teste que exercita um fluxo completo do sistema, de ponta a ponta, com todos os componentes reais.        |
| **Fixture**                  | Dados pré-configurados usados como base para testes.                                                     |
| **Flaky test**               | Teste que falha intermitentemente sem que o código sob teste tenha mudado.                                |
| **Gate de qualidade**        | Critério que deve ser atingido para que o código avance para a próxima etapa do pipeline.                 |
| **GWT**                      | Given / When / Then — padrão comportamental equivalente ao AAA.                                          |
| **Idempotência**             | Propriedade onde executar a mesma operação N vezes produz o mesmo resultado que executá-la 1 vez.         |
| **Mock**                     | Objeto simulado que substitui uma dependência real, permitindo controlar seu comportamento no teste.      |
| **Mutation testing**         | Técnica que introduz pequenas modificações (mutantes) no código e verifica se os testes os detectam.      |
| **Pirâmide de testes**       | Modelo que sugere proporcionalmente mais testes unitários, menos integração e ainda menos E2E.            |
| **Polling**                  | Técnica de verificar repetidamente uma condição até que seja satisfeita ou um timeout expire.             |
| **Quarentena**               | Suíte separada onde testes flakey são isolados para não bloquear pipeline enquanto são corrigidos.        |
| **SAST**                     | Static Application Security Testing — análise de segurança no código-fonte sem executá-lo.               |
| **Schema**                   | Definição formal da estrutura de dados (campos, tipos, obrigatoriedade) de um payload ou tabela.          |
| **Seed**                     | Script que popula o banco com dados iniciais necessários para testes ou desenvolvimento.                  |
| **Smoke test**               | Teste mínimo e rápido que verifica se o sistema está funcional após deploy ("a fumaça passou?").          |
| **Stub**                     | Implementação simplificada de uma dependência que retorna respostas pré-definidas.                        |
| **Teardown**                 | Fase de limpeza após a execução de um teste, revertendo efeitos colaterais.                              |
| **Test double**              | Termo genérico para qualquer substituto de dependência em testes (mock, stub, fake, spy, dummy).          |

---

> **Nota final:** Este documento deve ser revisado trimestralmente. Os valores entre `[colchetes]` são placeholders que devem ser calibrados por cada time conforme a maturidade e criticidade do sistema. Em caso de conflito com outros documentos normativos, a versão mais recente prevalece.
