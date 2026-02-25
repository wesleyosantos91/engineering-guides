# Estratégia de Testes e Qualidade de Software

> **Versão:** 2.0
> **Última atualização:** 2026-02-24
> **Público-alvo:** Times de engenharia (back-end, front-end, plataforma, QA, SRE)
> **Classificação:** Documento normativo — decisões aqui são obrigatórias para novos componentes e fortemente recomendadas para legado.
> **Documentos relacionados:** [Performance Testing](performance-testing.md) · [Security Testing](security-testing.md) · [Test Automation Patterns](test-automation-patterns.md) · [Code Review & Quality](code-review-quality.md) · [Observability & Quality](observability-quality.md)

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Pirâmide de Testes (versão moderna)](#2-pirâmide-de-testes-versão-moderna)
3. [Padrão de Escrita: Cenário → Ação → Resultado](#3-padrão-de-escrita-cenário--ação--resultado)
4. [Estratégia por Camada/Componente](#4-estratégia-por-camadacomponente)
5. [Estratégia para Assíncrono e Resiliência](#5-estratégia-para-assíncrono-e-resiliência)
6. [Test Doubles — Quando Usar Cada Tipo](#6-test-doubles--quando-usar-cada-tipo)
7. [Estratégias Avançadas de Teste](#7-estratégias-avançadas-de-teste)
8. [Testes para Front-end](#8-testes-para-front-end)
9. [Testes em Arquitetura de Microsserviços](#9-testes-em-arquitetura-de-microsserviços)
10. [Dados de Teste e Ambientes](#10-dados-de-teste-e-ambientes)
11. [CI/CD e Gates de Qualidade](#11-cicd-e-gates-de-qualidade)
12. [Métricas e Observabilidade de Testes](#12-métricas-e-observabilidade-de-testes)
13. [Ferramentas e Frameworks Recomendados](#13-ferramentas-e-frameworks-recomendados)
14. [Definition of Done (DoD) e Checklists](#14-definition-of-done-dod-e-checklists)
15. [Glossário](#15-glossário)

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

- Testes de performance/carga (ver [Performance Testing](performance-testing.md)).
- Testes de segurança/penetração (ver [Security Testing](security-testing.md)).
- Padrões avançados de automação e receitas (ver [Test Automation Patterns](test-automation-patterns.md)).
- Detalhes de configuração de ferramentas específicas (cada time define suas ferramentas).
- Testes exploratórios manuais (cobertos no processo de QA do time).

### 1.3 Shift-Left Testing

> **Conceito:** Mover atividades de qualidade para as fases mais iniciais do ciclo de desenvolvimento.

Em vez de tratar qualidade como uma fase posterior ao desenvolvimento, ela deve ser integrada **desde o design**:

```
Tradicional:  Design → Código → Testes → Deploy → Monitoramento
                                  ^^^^^
                            Qualidade aqui

Shift-Left:   Design → Código → Testes → Deploy → Monitoramento
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                        Qualidade em TODAS as fases
```

| Fase           | Atividade de Qualidade                                                        |
|----------------|-------------------------------------------------------------------------------|
| **Design**     | Revisão de contratos, definição de critérios de aceitação, threat modeling      |
| **Código**     | TDD/BDD, pair programming, análise estática, pré-commit hooks                  |
| **Build**      | Testes unitários, integração, contrato, SAST, cobertura                        |
| **Deploy**     | Smoke tests, canary analysis, feature flags                                   |
| **Produção**   | Observabilidade, alertas, chaos engineering, synthetic monitoring              |

### 1.4 Princípios Fundamentais

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

## 6. Test Doubles — Quando Usar Cada Tipo

### 6.1 Taxonomia de Test Doubles

| Tipo      | Definição                                                                                      | Quando usar                                         |
|-----------|-----------------------------------------------------------------------------------------------|-----------------------------------------------------|
| **Dummy** | Objeto passado como parâmetro mas nunca usado. Apenas preenche a assinatura.                  | Parâmetros obrigatórios que não afetam o teste       |
| **Stub**  | Retorna respostas pré-definidas. Não verifica interações.                                     | Isolar dependência fornecendo dados controlados      |
| **Spy**   | Registra chamadas recebidas para verificação posterior. Pode delegar ao objeto real.           | Verificar que uma interação ocorreu (ex.: evento publicado) |
| **Mock**  | Objeto com expectativas pré-programadas. Verifica interações automaticamente.                 | Validar orquestração e ordem de chamadas             |
| **Fake**  | Implementação funcional simplificada (ex.: banco in-memory, fila local).                      | Testes de integração leves, sem infra externa        |

### 6.2 Regras de Uso

| Regra | Descrição                                                                                          |
|-------|----------------------------------------------------------------------------------------------------|
| TD1   | **Prefira Stubs a Mocks.** Stubs tornam o teste menos acoplado à implementação.                    |
| TD2   | **Use Fakes para integração leve.** Um `FakeRepository` in-memory é mais legível que um mock complexo. |
| TD3   | **Mocks apenas para verificar interações críticas.** Ex.: "evento FOI publicado", não "método X foi chamado". |
| TD4   | **Nunca mock de Value Objects.** Se é simples criar o real, use o real.                            |
| TD5   | **Mock no limite arquitetural.** Mock em portas/interfaces, nunca em classes concretas internas.   |
| TD6   | **Se o mock é complexo demais, redesenhe o código.** Mock complicado = design smell.               |

### 6.3 Exemplos Comparativos

#### Stub (Java — JUnit + Mockito)

```java
// Stub: controla o retorno, não verifica interação
@Test
void calcularFrete_cepValido_retornaValorCalculado() {
    // Arrange
    var tabelaFrete = mock(TabelaFretePort.class);
    when(tabelaFrete.buscarPorCep("01310-100")).thenReturn(new Frete(15.90));
    var calculadora = new CalculadoraFrete(tabelaFrete);

    // Act
    var resultado = calculadora.calcular("01310-100", 2.5);

    // Assert
    assertThat(resultado.getValor()).isEqualByComparingTo("39.75");
}
```

#### Mock com verificação (Java — JUnit + Mockito)

```java
// Mock: verifica que a interação ocorreu
@Test
void criarPedido_sucesso_publicaEventoPedidoCriado() {
    // Arrange
    var eventPublisher = mock(EventPublisherPort.class);
    var repository = mock(PedidoRepository.class);
    var useCase = new CriarPedidoUseCase(repository, eventPublisher);
    var comando = new CriarPedidoCommand("cli-123", List.of(item("SKU-1", 2)));

    // Act
    useCase.executar(comando);

    // Assert — verifica que o evento foi publicado com dados corretos
    verify(eventPublisher).publish(argThat(event ->
        event.getClienteId().equals("cli-123") &&
        event.getType().equals("PEDIDO_CRIADO")
    ));
}
```

#### Fake (Go)

```go
// Fake: implementação funcional simplificada
type FakeOrderRepository struct {
    orders map[string]*Order
}

func NewFakeOrderRepository() *FakeOrderRepository {
    return &FakeOrderRepository{orders: make(map[string]*Order)}
}

func (r *FakeOrderRepository) Save(ctx context.Context, order *Order) error {
    r.orders[order.ID] = order
    return nil
}

func (r *FakeOrderRepository) FindByID(ctx context.Context, id string) (*Order, error) {
    order, ok := r.orders[id]
    if !ok {
        return nil, ErrNotFound
    }
    return order, nil
}

func TestCreateOrder_ValidData_PersistsCorrectly(t *testing.T) {
    repo := NewFakeOrderRepository()
    svc := NewOrderService(repo)

    order, err := svc.Create(ctx, CreateOrderInput{CustomerID: "cust-1"})

    require.NoError(t, err)
    saved, _ := repo.FindByID(ctx, order.ID)
    assert.Equal(t, "cust-1", saved.CustomerID)
    assert.Equal(t, StatusPending, saved.Status)
}
```

### 6.4 Anti-Patterns com Test Doubles

| Anti-pattern                              | Sintoma                                           | Solução                                                  |
|-------------------------------------------|---------------------------------------------------|----------------------------------------------------------|
| Mock retorna mock que retorna mock        | Teste ilegível, 10+ linhas de setup de mocks     | Redesenhar o código — Law of Demeter violada             |
| Verificar ordem exata de chamadas         | Teste quebra ao reordenar código que não muda resultado | Verificar resultado final, não sequência interna     |
| Mock de classe concreta interna           | Acoplado à implementação, não ao comportamento    | Extrair interface/port e mock na fronteira               |
| Stub que replica lógica real              | Duplica comportamento, diverge com o tempo        | Usar Fake com implementação simplificada testada         |
| Não verificar argumentos do mock          | Teste passa mesmo com dados errados               | Usar `argThat()`, `ArgumentCaptor` ou equivalente        |

---

## 7. Estratégias Avançadas de Teste

### 7.1 Property-Based Testing (PBT)

> **Conceito:** Em vez de testar com valores específicos (exemplos), definir **propriedades** que devem ser verdadeiras para **qualquer** input válido. O framework gera centenas de inputs aleatórios automaticamente.

| Aspecto            | Detalhe                                                                            |
|--------------------|------------------------------------------------------------------------------------|
| **Quando usar**    | Funções puras, serialização/deserialização, parsers, codecs, algoritmos            |
| **Quando evitar**  | Lógica com muitas dependências externas, código com poucos inputs possíveis        |
| **Benefício**      | Encontra edge cases que humanos jamais pensariam em testar                          |
| **Frameworks**     | jqwik (Java), fast-check (JS/TS), Hypothesis (Python), gopter (Go)                |

**Exemplo — Java com jqwik:**

```java
@Property
void serializarEDeserializar_qualquerPedido_mantemDadosIntactos(
    @ForAll @StringLength(min = 1, max = 50) String clienteId,
    @ForAll @Positive BigDecimal valor
) {
    var pedido = new Pedido(clienteId, valor);

    // Propriedade: serialize → deserialize deve retornar objeto equivalente
    var json = serializer.toJson(pedido);
    var restaurado = serializer.fromJson(json, Pedido.class);

    assertThat(restaurado).isEqualTo(pedido);
}
```

**Exemplo — TypeScript com fast-check:**

```typescript
import fc from 'fast-check';

test('sort é idempotente — ordenar duas vezes dá o mesmo resultado', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const sorted1 = [...arr].sort((a, b) => a - b);
      const sorted2 = [...sorted1].sort((a, b) => a - b);
      expect(sorted1).toEqual(sorted2);
    })
  );
});
```

### 7.2 Snapshot Testing

> **Conceito:** Capturar a saída de um componente/função e salvar como "snapshot". Em execuções futuras, comparar a saída atual com o snapshot salvo. Qualquer diferença falha o teste.

| Aspecto            | Detalhe                                                                            |
|--------------------|------------------------------------------------------------------------------------|
| **Quando usar**    | Componentes UI, respostas de API, schemas, HTML/JSON gerado                        |
| **Quando evitar**  | Dados que mudam frequentemente (timestamps, IDs aleatórios)                         |
| **Cuidado**        | Snapshots devem ser revisados no PR — "aprovar cegamente" anula o valor             |
| **Frameworks**     | Jest (JS/TS), approval tests (Java/C#), snapshot crates (Rust)                     |

```typescript
// Jest snapshot — componente React
test('renderiza corretamente o card de produto', () => {
  const tree = renderer.create(
    <ProductCard name="Notebook" price={2999.90} inStock={true} />
  ).toJSON();

  expect(tree).toMatchSnapshot();
});
```

**Regras para snapshots:**
- Snapshot novo ou alterado DEVE ser revisado manualmente no PR
- Excluir campos dinâmicos (timestamps, UUIDs) via serializers customizados
- Manter snapshots pequenos e focados — evitar snapshots de páginas inteiras

### 7.3 Mutation Testing

> **Conceito:** Introduzir pequenas modificações (mutantes) no código-fonte e verificar se os testes detectam. Se um mutante sobrevive, significa que os testes não validam aquele comportamento.

| Mutante típico                  | Exemplo                                        | O que revela se sobrevive             |
|--------------------------------|------------------------------------------------|---------------------------------------|
| Troca operador relacional      | `>` → `>=`                                     | Assert não valida valor limite         |
| Remove chamada void            | Remove `eventPublisher.publish(...)`           | Efeito colateral não verificado       |
| Troca retorno por constante    | `return calculado` → `return 0`                | Assert genérico (ex.: `assertNotNull`) |
| Nega condição                  | `if (valid)` → `if (!valid)`                   | Caminho de erro não testado           |
| Remove exceção                 | Remove `throw new ...`                          | Cenário de erro não testado           |

**Ferramentas:** PITest (Java), Stryker (JS/TS/C#), mutmut (Python), go-mutesting (Go)

**Meta recomendada:** ≥ 60% mutation score em código de domínio/negócio.

### 7.4 Chaos Engineering (Testes em Produção)

> **Conceito:** Introduzir falhas controladas em sistemas em produção para verificar resiliência.

| Prática                     | Descrição                                                                   | Ferramenta exemplo    |
|-----------------------------|-----------------------------------------------------------------------------|-----------------------|
| **Kill pod aleatório**      | Verificar que o sistema se recupera com réplicas restantes                  | Chaos Monkey, Litmus  |
| **Injetar latência**        | Inserir delay em chamadas entre serviços                                   | Istio fault injection |
| **Simular falha de zona**   | Desabilitar uma AZ e verificar failover                                    | AWS FIS, Gremlin      |
| **CPU/Memory stress**       | Saturar recursos de um pod e verificar autoscaling                         | stress-ng, Litmus     |
| **Network partition**       | Simular perda de conectividade entre serviços                              | tc (traffic control)  |

**Pré-requisitos obrigatórios:**
- Observabilidade completa (métricas, logs, traces)
- Runbook de rollback documentado
- Blast radius controlado (começar com % mínimo de tráfego)
- Game day agendado e comunicado

---

## 8. Testes para Front-end

### 8.1 Pirâmide de Testes Front-end

```
          ╱  ╲
         ╱ E2E╲              ← Cypress, Playwright (poucos)
        ╱ Visual ╲
       ╱──────────╲
      ╱ Integração  ╲        ← Testing Library (componente + contexto)
     ╱────────────────╲
    ╱    Componente      ╲   ← Testing Library (componente isolado)
   ╱──────────────────────╲
  ╱       Unitário          ╲ ← Lógica pura, hooks, utils (Jest/Vitest)
 ╱────────────────────────────╲
```

### 8.2 Estratégia por Tipo

| Tipo                    | O que testar                                                     | Ferramentas             | Proporção |
|-------------------------|------------------------------------------------------------------|-------------------------|-----------|
| **Unitário**            | Hooks customizados, funções utilitárias, state machines, reducers | Jest, Vitest            | ~40%      |
| **Componente**          | Renderização, interação do usuário, estados (loading, error, empty) | Testing Library        | ~30%      |
| **Integração**          | Fluxo entre componentes, contexto/providers, rotas               | Testing Library + MSW   | ~20%      |
| **Visual Regression**   | Aparência do componente não mudou (screenshot diff)              | Chromatic, Percy, Loki  | ~5%       |
| **E2E**                 | Fluxos críticos completos (login → ação → resultado)             | Playwright, Cypress     | ~5%       |

### 8.3 Princípios de Teste Front-end

| Princípio                                 | Descrição                                                                      |
|-------------------------------------------|--------------------------------------------------------------------------------|
| **Teste como o usuário usa**              | Buscar por texto, role, label — nunca por classe CSS ou ID de implementação    |
| **Evite testar detalhes de implementação**| Não teste state interno, métodos privados, ou chamadas de lifecycle             |
| **Prefira queries acessíveis**            | `getByRole`, `getByLabelText`, `getByText` > `getByTestId` > nunca `querySelector` |
| **Mock na fronteira de rede**             | Usar MSW (Mock Service Worker) para interceptar chamadas HTTP                  |
| **Teste estados do componente**           | Loading, success, error, empty — cada estado é um cenário                      |

### 8.4 Exemplos — React com Testing Library

#### Teste de componente isolado

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ProductCard } from './ProductCard';

describe('ProductCard', () => {
  it('exibe nome, preço e botão de adicionar ao carrinho', () => {
    render(<ProductCard name="Notebook" price={2999.90} />);

    expect(screen.getByRole('heading', { name: 'Notebook' })).toBeInTheDocument();
    expect(screen.getByText('R$ 2.999,90')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /adicionar/i })).toBeEnabled();
  });

  it('desabilita botão quando produto fora de estoque', () => {
    render(<ProductCard name="Notebook" price={2999.90} inStock={false} />);

    expect(screen.getByRole('button', { name: /adicionar/i })).toBeDisabled();
  });
});
```

#### Teste de integração com MSW

```tsx
import { rest } from 'msw';
import { setupServer } from 'msw/node';
import { render, screen, waitFor } from '@testing-library/react';
import { OrderHistory } from './OrderHistory';

const server = setupServer(
  rest.get('/api/orders', (req, res, ctx) =>
    res(ctx.json([
      { id: '1', product: 'Notebook', status: 'delivered' },
      { id: '2', product: 'Mouse', status: 'shipping' },
    ]))
  )
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('exibe lista de pedidos do usuário', async () => {
  render(<OrderHistory userId="user-123" />);

  expect(screen.getByText(/carregando/i)).toBeInTheDocument();

  await waitFor(() => {
    expect(screen.getByText('Notebook')).toBeInTheDocument();
    expect(screen.getByText('Mouse')).toBeInTheDocument();
  });
});

test('exibe mensagem de erro quando API falha', async () => {
  server.use(
    rest.get('/api/orders', (req, res, ctx) => res(ctx.status(500)))
  );

  render(<OrderHistory userId="user-123" />);

  await waitFor(() => {
    expect(screen.getByText(/erro ao carregar/i)).toBeInTheDocument();
  });
});
```

### 8.5 Anti-Patterns Front-end

| Anti-pattern                                | Problema                                        | Solução                                            |
|---------------------------------------------|--------------------------------------------------|----------------------------------------------------|
| Buscar por className ou CSS selector        | Quebra com mudança de estilo                     | Usar `getByRole`, `getByText`, `getByLabelText`   |
| Testar state interno do componente          | Acoplado à implementação                         | Testar o que o usuário vê/interage                 |
| Snapshot de componente inteiro              | Diff enorme, aprovação cega                      | Snapshots pequenos e focados, ou visual regression |
| E2E para tudo                               | Lento, frágil, caro                              | E2E apenas para happy paths críticos               |
| Não testar acessibilidade                   | Problemas a11y chegam a produção                 | `jest-axe`, `@axe-core/playwright` em todo componente |

---

## 9. Testes em Arquitetura de Microsserviços

### 9.1 Desafios Específicos

| Desafio                            | Descrição                                                                         |
|------------------------------------|-----------------------------------------------------------------------------------|
| **Ownership distribuído**          | Cada serviço é de um time diferente — quem testa a integração?                    |
| **Versões independentes**          | Serviço A pode deployar sem Serviço B estar pronto                                |
| **Consistência eventual**          | Dados propagam via eventos — estado inconsistente é temporariamente normal         |
| **Explosão combinatória**          | N serviços × M versões = testes E2E exponenciais                                  |
| **Ambiente complexo**              | Subir todos os serviços localmente é impraticável                                 |

### 9.2 Estratégia Recomendada

```
┌─────────────────────────────────────────────────────────────┐
│                   MICROSSERVIÇOS                             │
│                                                              │
│  Serviço A          Serviço B          Serviço C            │
│  ┌──────────┐       ┌──────────┐       ┌──────────┐        │
│  │ Unit     │       │ Unit     │       │ Unit     │        │
│  │ Integ    │       │ Integ    │       │ Integ    │        │
│  │ Contract │◄─────►│ Contract │◄─────►│ Contract │        │
│  └──────────┘       └──────────┘       └──────────┘        │
│        │                  │                  │               │
│        └──────────────────┴──────────────────┘               │
│                           │                                  │
│                    ┌──────────────┐                          │
│                    │ E2E / Smoke  │  ← Poucos, pós-deploy   │
│                    │ (Staging)    │                          │
│                    └──────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

**Princípio:** Cada serviço é testável **isoladamente**. A integração entre serviços é validada via **testes de contrato**, não via E2E.

### 9.3 Consumer-Driven Contract Testing (CDCT)

| Passo | Ator         | Ação                                                                      |
|-------|--------------|---------------------------------------------------------------------------|
| 1     | Consumidor   | Define suas expectativas (quais campos, tipos, valores espera da API/evento) |
| 2     | Consumidor   | Gera um "contrato" (arquivo JSON/Pact) e publica no broker               |
| 3     | Produtor     | Baixa o contrato e roda contra sua implementação real                     |
| 4     | Produtor     | Se passa → compatível. Se falha → breaking change detectada antes de prod |

**Ferramentas:** Pact (multi-linguagem), Spring Cloud Contract (Java), Specmatic (OpenAPI-based)

### 9.4 Service Virtualization

Quando testar *seu* serviço mas a dependência ainda não está pronta (ou é cara):

| Abordagem                | Descrição                                               | Ferramenta exemplo      |
|--------------------------|---------------------------------------------------------|-------------------------|
| **WireMock/Mountebank**  | Servidor HTTP que simula APIs com respostas configuradas | WireMock, Mountebank    |
| **Testcontainers**       | Containers efêmeros com mocks de infraestrutura         | Testcontainers          |
| **LocalStack**           | Simulação de serviços AWS localmente                    | LocalStack              |
| **MSW**                  | Mock a nível de rede para front-end/Node                | Mock Service Worker      |

### 9.5 Padrões de Teste por Tipo de Comunicação

| Comunicação           | Padrão de teste                                                              |
|-----------------------|------------------------------------------------------------------------------|
| **REST síncrono**     | Contract test (OpenAPI/Pact) + Integration test com WireMock                 |
| **gRPC**              | Contract test (proto validation) + Integration test com servidor in-process  |
| **Eventos (Kafka/SQS)** | Contract test (schema registry) + Integration test com Testcontainers     |
| **GraphQL**           | Contract test (schema) + Integration test via HTTP                           |
| **Saga/Coreografia**  | Teste de cada step isolado + E2E do fluxo completo em staging               |

---

## 10. Dados de Teste e Ambientes

### 10.1 Estratégia de Test Data

| Estratégia               | Quando usar                                     | Cuidados                                              |
|--------------------------|-------------------------------------------------|-------------------------------------------------------|
| **Builders / Factories** | Unitários e integração — criar objetos de domínio| Defaults sensatos; sobrescrever apenas o relevante     |
| **Fixtures (JSON/YAML)** | Dados complexos, snapshots, contratos            | Versionar junto ao código; não compartilhar entre suítes |
| **Seeds (scripts DDL/DML)** | Popular banco para testes de integração       | Executar no setup da suíte; limpar no teardown         |
| **Dados sintéticos**     | Testes E2E, staging, dados sensíveis             | Gerar via algoritmo determinístico; nunca usar dados reais de produção |
| **Copy de produção (anonimizado)** | Testes de carga, validação de migração  | Obrigatório anonimização; considerar volume             |

### 10.2 Regras de Dados

| Regra  | Descrição                                                                                              |
|--------|--------------------------------------------------------------------------------------------------------|
| D1     | **Nunca** usar dados de produção reais em ambientes de teste (LGPD/GDPR/compliance).                  |
| D2     | Cada teste deve criar **seus próprios dados**. Proibido depender de dados pré-existentes no banco.     |
| D3     | Usar **identificadores únicos** por teste (UUID, sufixo timestamp) para evitar colisão em paralelismo. |
| D4     | Dados de teste são **código** — versionados, revisados e mantidos.                                     |
| D5     | Preferir **builders com defaults** sobre fixtures estáticas — builders são mais flexíveis e legíveis.  |

### 10.3 Isolamento e Paralelismo

| Mecanismo                           | Descrição                                                               |
|--------------------------------------|-------------------------------------------------------------------------|
| **Container efêmero por suíte**      | Cada suíte de integração sobe seu próprio container de banco/cache      |
| **Schema/namespace isolado**         | Cada execução paralela usa um schema/namespace diferente                |
| **Transação revertida**              | Cada teste roda dentro de uma transação que faz rollback ao final       |
| **Nomes com sufixo único**           | Filas, tópicos e tabelas temporárias usam sufixo único por execução     |
| **Clock determinístico**             | Relógio injetável permite testar lógica temporal sem depender de `now()` |

### 10.4 Ambientes

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

## 11. CI/CD e Gates de Qualidade

### 11.1 Pipeline Recomendado

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

### 11.2 Gates de Qualidade

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

### 11.3 Política de Quarentena para Testes Flakey

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

## 12. Métricas e Observabilidade de Testes

### 12.1 Métricas Obrigatórias

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

### 12.2 Observabilidade nos Testes

| Prática                               | Descrição                                                                         |
|----------------------------------------|-----------------------------------------------------------------------------------|
| **Logs estruturados no teste**         | Cada teste deve gerar logs com `testName`, `testId` e contexto suficiente          |
| **Trace ID no teste de integração**    | Propagar trace ID nos testes de integração para rastrear chamadas entre componentes |
| **Screenshot/Snapshot em falha**       | Testes E2E devem capturar estado visual ou dump do sistema ao falhar               |
| **Diff de estado**                     | Em falhas de integração, logar o estado esperado vs. obtido em formato diff        |
| **Métricas de tempo por teste**        | Identificar testes lentos (> [2]s para unitário, > [30]s para integração)          |
| **Rerun automático com log enriquecido** | Na primeira falha, re-executar com nível de log aumentado para capturar mais contexto |

### 12.3 Dashboard Recomendado

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

## 13. Ferramentas e Frameworks Recomendados

### 13.1 Por Linguagem/Plataforma

| Linguagem     | Unit Test            | Mocking              | Integration/Container   | Property-Based      | Mutation           |
|---------------|----------------------|----------------------|-------------------------|---------------------|--------------------||
| **Java**      | JUnit 5              | Mockito, MockK (Kotlin) | Testcontainers, Spring Boot Test | jqwik          | PITest             |
| **Kotlin**    | JUnit 5, Kotest      | MockK                | Testcontainers          | Kotest property     | PITest             |
| **Go**        | testing (stdlib)     | testify/mock, gomock | Testcontainers-Go       | gopter, rapid       | go-mutesting       |
| **TypeScript**| Jest, Vitest         | jest.mock, ts-mockito| Testcontainers-Node     | fast-check          | Stryker            |
| **Python**    | pytest               | unittest.mock, pytest-mock | Testcontainers-Python | Hypothesis       | mutmut             |
| **C#/.NET**   | xUnit, NUnit         | Moq, NSubstitute     | Testcontainers-dotnet   | FsCheck             | Stryker.NET        |

### 13.2 Por Tipo de Teste

| Tipo de Teste          | Ferramentas recomendadas                                                       |
|------------------------|--------------------------------------------------------------------------------|
| **Contract Testing**   | Pact, Spring Cloud Contract, Specmatic                                         |
| **API Testing**        | REST Assured (Java), Supertest (Node), httptest (Go)                           |
| **E2E / Browser**      | Playwright, Cypress                                                            |
| **Visual Regression**  | Chromatic, Percy, Loki, BackstopJS                                             |
| **Load/Performance**   | k6, Gatling, JMeter, Locust                                                   |
| **Security (SAST)**    | SonarQube, Semgrep, CodeQL, Snyk Code                                          |
| **Security (DAST)**    | OWASP ZAP, Burp Suite                                                          |
| **Chaos Engineering**  | Litmus, Chaos Monkey, Gremlin, AWS FIS                                         |
| **Coverage**           | JaCoCo (Java), Istanbul/c8 (JS/TS), coverage.py (Python), go tool cover (Go)  |
| **Accessibility**      | axe-core, jest-axe, @axe-core/playwright, pa11y                               |

### 13.3 Infraestrutura de Teste

| Ferramenta            | Propósito                                                                    |
|-----------------------|---------------------------------------------------------------------------------|
| **Testcontainers**    | Containers efêmeros para banco, fila, cache — multi-linguagem                  |
| **LocalStack**        | Simulação de serviços AWS localmente (S3, SQS, DynamoDB, Lambda, etc.)        |
| **WireMock**          | Servidor HTTP simula APIs externas com respostas configuráveis                 |
| **MSW**               | Mock Service Worker — intercepta requisições HTTP a nível de rede (front-end)   |
| **MailHog / MailPit** | Servidor SMTP fake para testar envio de e-mails                                |
| **MinIO**             | Servidor S3-compatível para testes de storage                                  |
| **Toxiproxy**         | Proxy programável para simular falhas de rede (latência, timeout, disconnect) |

### 13.4 Análise Estática como Gate de Qualidade

| Ferramenta        | O que verifica                                                                 |
|-------------------|--------------------------------------------------------------------------------|
| **SonarQube**     | Code smells, bugs, vulnerabilities, cobertura, duplicação                     |
| **ESLint/Biome**  | Padrões de código, erros comuns (JS/TS)                                        |
| **Checkstyle**    | Convenções de código (Java)                                                    |
| **golangci-lint** | Linters agregados para Go                                                      |
| **Ruff**          | Linter ultra-rápido para Python                                                |
| **Semgrep**       | Análise estática customizável, foco em segurança                                |
| **Trivy/Snyk**    | Vulnerabilidades em dependências e imagens Docker                               |

> **Regra:** Análise estática deve rodar **antes** dos testes na pipeline. Código que não passa no lint não merece ter testes executados.

---

## 14. Definition of Done (DoD) e Checklists

### 14.1 Definition of Done — Features

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

### 14.2 Definition of Done — Bugs

| #   | Critério                                                                    |
|-----|-----------------------------------------------------------------------------|
| B1  | Bug reproduzido com teste que falha **antes** do fix.                       |
| B2  | Fix implementado e teste agora passa.                                       |
| B3  | Teste adicionado permanentemente à suíte (não removido após fix).           |
| B4  | Root cause documentado no ticket.                                           |
| B5  | Verificado se o mesmo padrão de bug existe em outros módulos (busca ampla). |

### 14.3 Checklist de Pull Request

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

### 14.4 Anti-Patterns e Sinais Vermelhos

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

## 15. Glossário

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
| **Testcontainers**           | Biblioteca que provê containers Docker efêmeros para testes de integração.                               |
| **Shift-left**               | Prática de mover atividades de qualidade para fases mais iniciais do ciclo de desenvolvimento.           |
| **Property-based testing**   | Técnica que gera inputs aleatórios e verifica propriedades invariantes do código, em vez de exemplos fixos. |
| **Snapshot testing**         | Técnica que salva a saída de um componente e compara com versões futuras para detectar mudanças.          |
| **Chaos engineering**        | Disciplina de experimentar em sistemas distribuídos para construir confiança na resiliência.              |
| **MSW (Mock Service Worker)**| Ferramenta que intercepta requisições HTTP a nível de service worker para testes front-end.               |
| **CDCT**                     | Consumer-Driven Contract Testing — abordagem onde o consumidor define suas expectativas de contrato.      |
| **Spy**                      | Test double que registra chamadas para verificação posterior, podendo delegar ao objeto real.             |
| **Fake**                     | Implementação funcional simplificada usada em testes (ex.: repositório in-memory).                       |
| **Dummy**                    | Objeto passado como parâmetro mas nunca utilizado, apenas para satisfazer assinatura.                    |
| **Visual regression test**   | Teste que compara screenshots de componentes/páginas para detectar mudanças visuais inesperadas.         |
| **Service virtualization**   | Simulação de sistemas externos (APIs, serviços) para permitir testes independentes.                      |
| **Synthetic monitoring**     | Testes automatizados que simulam interações de usuário em produção para detectar problemas proativamente. |

---

> **Nota final:** Este documento deve ser revisado trimestralmente. Os valores entre `[colchetes]` são sugestões que devem ser calibrados por cada time conforme a maturidade e criticidade do sistema. Em caso de conflito com outros documentos normativos, a versão mais recente prevalece. Para tópicos complementares, consulte os documentos relacionados listados no cabeçalho.
