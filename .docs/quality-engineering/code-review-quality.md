# Code Review & Quality — Guia de Revisão de Código

> **Versão:** 1.0
> **Última atualização:** 2026-02-24
> **Público-alvo:** Todos os engenheiros que participam de code reviews
> **Documentos relacionados:** [Testing Strategy](testing-strategy.md) · [Test Automation Patterns](test-automation-patterns.md)

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Princípios de Code Review](#2-princípios-de-code-review)
3. [O que Revisar](#3-o-que-revisar)
4. [Checklist de Revisão por Área](#4-checklist-de-revisão-por-área)
5. [Padrões de Feedback](#5-padrões-de-feedback)
6. [Processo e Fluxo](#6-processo-e-fluxo)
7. [Métricas de Code Review](#7-métricas-de-code-review)
8. [Anti-Patterns](#8-anti-patterns)

---

## 1. Visão Geral

### 1.1 Objetivos do Code Review

| Objetivo                      | Descrição                                                                      |
|-------------------------------|--------------------------------------------------------------------------------|
| **Corretude**                 | O código faz o que deveria fazer? Cobre edge cases?                            |
| **Qualidade**                 | O código é legível, manutenível e segue padrões da equipe?                     |
| **Segurança**                 | O código introduz vulnerabilidades? Valida inputs? Protege dados sensíveis?    |
| **Disseminação de conhecimento** | O review espalha conhecimento do domínio e das decisões técnicas           |
| **Consistência**              | O código segue convenções e padrões estabelecidos pelo time?                   |
| **Mentoria**                  | Oportunidade de ensinar e aprender entre pares                                 |

### 1.2 O que Code Review **NÃO** é

| O que não é                    | Razão                                                                      |
|--------------------------------|----------------------------------------------------------------------------|
| Gatekeeping / poder sobre outros | Review deve ser colaborativo, não bloqueio por ego                       |
| Substituto de testes automatizados | Humanos falham em detectar bugs; automação é complementar               |
| Lugar para discutir arquitetura | Decisões de arquitetura devem ser feitas antes da implementação (ADR)     |
| Verificação de formatação      | Formatação deve ser automatizada (linters, formatters)                    |

---

## 2. Princípios de Code Review

| #  | Princípio                              | Descrição                                                                           |
|----|----------------------------------------|-------------------------------------------------------------------------------------|
| R1 | **Respeito e empatia**                 | Criticar o código, nunca a pessoa. Usar linguagem construtiva.                      |
| R2 | **Praise first**                       | Reconhecer o que está bom antes de apontar problemas.                               |
| R3 | **Perguntar antes de afirmar**         | "Você considerou X?" é melhor que "Isso está errado".                               |
| R4 | **Explicar o porquê**                  | Não basta dizer "mude isso" — explique o risco ou benefício.                        |
| R5 | **Tamanho pequeno**                    | PRs pequenos (< 400 linhas) recebem reviews melhores e mais rápidos.               |
| R6 | **Velocidade importa**                 | Review em ≤ 24 horas úteis. Review bloqueado = feature bloqueada.                   |
| R7 | **Automação primeiro**                 | Se pode ser detectado por linter/CI, não deveria ser comentário de review.          |
| R8 | **Nit vs. Blocker**                    | Diferenciar claramente comentários cosméticos de problemas que bloqueiam a aprovação.|
| R9 | **Decisões, não preferências**         | Comentários devem ser baseados em padrões do time, não em preferência pessoal.      |
| R10| **Trust but verify**                   | Confiar na competência do autor, mas verificar a lógica crítica.                    |

---

## 3. O que Revisar

### 3.1 Prioridade de Verificação

| Prioridade | Área                          | Descrição                                                     | Tempo |
|------------|-------------------------------|---------------------------------------------------------------|-------|
| **P0**     | Corretude / Lógica            | O código faz o que deveria? Cobre edge cases?                 | 40%   |
| **P1**     | Segurança                     | Inputs validados? Auth verificada? Dados protegidos?          | 20%   |
| **P2**     | Testes                        | Cenários adequados? Cobertura suficiente? Testes legíveis?    | 20%   |
| **P3**     | Design / Manutenibilidade     | Separação de responsabilidades? Acoplamento adequado?         | 15%   |
| **P4**     | Performance                   | Queries N+1? Loops desnecessários? Caching adequado?          | 5%    |

### 3.2 O que NÃO revisar manualmente

| Item                                 | Deve ser automatizado via                          |
|--------------------------------------|----------------------------------------------------|
| Formatação e indentação              | Prettier, gofmt, google-java-format, Black          |
| Convenções de nomenclatura           | ESLint, Checkstyle, golangci-lint, Ruff             |
| Imports não utilizados               | Linters                                              |
| Complexidade ciclomática             | SonarQube, CodeClimate                               |
| Vulnerabilidades conhecidas em deps  | Snyk, Dependabot, Trivy                              |
| Cobertura de testes                  | CI pipeline gates (JaCoCo, Istanbul, coverage.py)    |

---

## 4. Checklist de Revisão por Área

### 4.1 Lógica e Corretude

```markdown
- [ ] O código implementa corretamente os requisitos/história?
- [ ] Edge cases estão tratados (null, vazio, limite, overflow)?
- [ ] Condições de erro estão tratadas (timeout, indisponibilidade, dados inválidos)?
- [ ] Não há lógica duplicada (DRY sem over-engineering)?
- [ ] State machines têm todos os estados e transições válidas?
- [ ] Cálculos com dinheiro usam BigDecimal/Decimal (não float/double)?
- [ ] Comparações de string consideram case sensitivity e locale?
- [ ] Datas/horários consideram timezone corretamente?
```

### 4.2 Testes

```markdown
- [ ] Testes unitários cobrem toda nova lógica de negócio?
- [ ] Testes de integração cobrem toda nova interação com I/O?
- [ ] Cenários de erro/exceção estão testados?
- [ ] Testes de contrato atualizados (se API/evento alterado)?
- [ ] Nomes dos testes descrevem cenário e resultado esperado?
- [ ] Testes são independentes (sem dependência de ordem ou estado)?
- [ ] Sem sleep/wait fixo em testes assíncronos?
- [ ] Asserts são específicos (não apenas assertNotNull)?
- [ ] Dados de teste são criados pelo próprio teste (não pré-existentes)?
```

### 4.3 Segurança

```markdown
- [ ] Inputs são validados na borda (tipo, tamanho, formato)?
- [ ] Queries usam parâmetros (não concatenação de strings)?
- [ ] Autorização é verificada (não apenas autenticação)?
- [ ] Dados sensíveis não são logados (CPF, cartão, senha, token)?
- [ ] Secrets não estão hardcoded no código?
- [ ] Respostas de erro não expõem stack traces ou dados internos?
- [ ] Novos endpoints estão protegidos por autenticação?
- [ ] Rate limiting considerado para endpoints públicos?
```

### 4.4 Design e Manutenibilidade

```markdown
- [ ] Funções/métodos têm uma única responsabilidade?
- [ ] Nomes de variáveis, funções e classes são descritivos?
- [ ] Complexidade está adequada (funções < 30 linhas, classes < 300)?
- [ ] Acoplamento está minimizado (depende de abstrações, não implementações)?
- [ ] Código morto foi removido (não apenas comentado)?
- [ ] Comentários explicam "por quê", não "o quê"?
- [ ] DTOs/Records são usados nas fronteiras (não entidades internas)?
- [ ] Exceções são tipadas e específicas (não Exception genérica)?
```

### 4.5 Performance (quando aplicável)

```markdown
- [ ] Queries de banco estão otimizadas (sem N+1, com índices)?
- [ ] Paginação implementada em listagens?
- [ ] Caching usado para dados frequentemente acessados e raramente alterados?
- [ ] Operações pesadas são assíncronas/background?
- [ ] Conexões (DB, HTTP) usam pool com limites configurados?
- [ ] Sem alocações desnecessárias em hot paths?
- [ ] Timeouts configurados para chamadas externas?
```

### 4.6 Observabilidade

```markdown
- [ ] Logs adicionados em pontos de decisão importantes?
- [ ] Logs estruturados com contexto suficiente (correlationId, userId)?
- [ ] Métricas adicionadas para novos fluxos de negócio?
- [ ] Alertas configurados para novos pontos de falha?
- [ ] Health check atualizado (se novo componente/dependência)?
- [ ] Trace context propagado em chamadas entre serviços?
```

---

## 5. Padrões de Feedback

### 5.1 Classificação de Comentários

| Prefixo          | Significado                                                  | Bloqueia aprovação? |
|------------------|--------------------------------------------------------------|---------------------|
| **[blocker]**    | Problema que deve ser corrigido antes do merge               | ✅ Sim              |
| **[suggestion]** | Melhoria recomendada, mas não obrigatória                    | ❌ Não              |
| **[nit]**        | Cosmético, estilo — totalmente opcional                      | ❌ Não              |
| **[question]**   | Dúvida genuína — não é crítica, é curiosidade                | ❌ Não              |
| **[praise]**     | Destaque positivo — algo bem feito                           | ❌ Não              |
| **[follow-up]**  | Não precisa ser neste PR, mas deveria ser feito em seguida   | ❌ Não              |

### 5.2 Exemplos de Bom Feedback

```markdown
❌ Ruim: "Isso está errado."
✅ Bom: "[blocker] Essa query não usa o índice de clienteId, o que pode causar 
   full table scan em produção. Sugestão: adicionar WHERE clienteId = ? com 
   índice ou usar a query nomeada findByClienteId do Spring Data."

❌ Ruim: "Use var aqui."
✅ Bom: "[nit] Aqui var melhoraria legibilidade — o tipo já está claro pelo 
   lado direito da atribuição."

❌ Ruim: "Cadê o teste?"
✅ Bom: "[blocker] Essa regra de desconto tem 3 branches — seria ótimo ter 
   testes para cada: cliente VIP, cliente regular, e cliente inativo."

❌ Ruim: "Refatore isso."
✅ Bom: "[suggestion] Esse método tem 45 linhas com lógica de validação + 
   persistência + notificação. Extrair cada responsabilidade para um método 
   separado facilitaria testes unitários e leitura."

✅ Excelente: "[praise] Boa decisão usar consumer-driven contracts aqui. 
   Isso vai evitar breaking changes silenciosos entre os serviços."
```

### 5.3 Regra dos 2 Blockers

Se um PR recebe **mais de 2 blockers**, é sinal de que:
- O PR pode estar muito grande → dividir
- Precisa de design review prévio → ADR/RFC
- O autor precisa de pairing → oferecer ajuda

---

## 6. Processo e Fluxo

### 6.1 Fluxo Recomendado

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Autor   │ → │  CI      │ → │  Review  │ → │  Ajustes │ → │  Merge   │
│  abre PR │   │  passa   │   │  assign  │   │  (se     │   │          │
│          │   │  (auto)  │   │          │   │  necessário) │          │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

### 6.2 SLAs de Review

| Tipo de PR            | SLA de primeira review    | Justificativa                       |
|-----------------------|---------------------------|-------------------------------------|
| Hotfix / Incident     | ≤ 2 horas                | Impacto em produção                 |
| Bug fix               | ≤ 8 horas úteis          | Afeta qualidade                     |
| Feature (< 200 LOC)  | ≤ 24 horas úteis         | PR pequeno, feedback rápido         |
| Feature (200-400 LOC) | ≤ 24 horas úteis         | Tamanho aceitável                   |
| Feature (> 400 LOC)   | Considerar dividir o PR  | PR muito grande para review eficaz  |

### 6.3 Critérios de Aprovação

| Critério                                    | Obrigatório? |
|---------------------------------------------|--------------|
| CI pipeline verde (testes, lint, build)     | ✅ Sim        |
| ≥ 1 aprovação de reviewer                  | ✅ Sim        |
| 0 blockers pendentes                        | ✅ Sim        |
| Cobertura ≥ threshold configurado           | ✅ Sim        |
| Sem novos alertas de segurança (SAST/SCA)  | ✅ Sim        |
| 0 nits pendentes                            | ❌ Não (nice-to-have) |
| CODEOWNERS aprovaram (se aplicável)         | ✅ Sim        |

### 6.4 Tamanho Ideal do PR

| Tamanho (LOC alterados) | Qualidade do review     | Recomendação                               |
|--------------------------|-------------------------|--------------------------------------------|
| 1-100                    | Excelente               | PR ideal — foco e revisão rápida           |
| 100-200                  | Boa                     | Aceitável para a maioria dos casos          |
| 200-400                  | Razoável                | Considerar dividir se possível              |
| 400-800                  | Ruim                    | Dividir em PRs menores                      |
| > 800                    | Muito ruim              | Obrigatório dividir — review será superficial |

> **Exceções:** Renames em massa, migrations, geração de código podem ser grandes mas são low-risk.

---

## 7. Métricas de Code Review

### 7.1 Métricas de Eficiência

| Métrica                       | Meta              | Como medir                                    |
|-------------------------------|-------------------|-----------------------------------------------|
| **Time to first review**      | ≤ 24h úteis       | Tempo entre abertura do PR e primeiro comentário |
| **Time to merge (TTM)**       | ≤ 48h úteis       | Tempo entre abertura e merge                    |
| **Review cycles**             | ≤ 2               | Número de rounds de review/ajuste               |
| **PR size (LOC)**             | ≤ 200             | Linhas alteradas por PR                         |
| **Review thoroughness**       | ≥ 80% PRs com comentários | % de PRs que recebem feedback substantivo |

### 7.2 Métricas de Qualidade

| Métrica                       | Meta              | O que indica                                    |
|-------------------------------|-------------------|-------------------------------------------------|
| **Defeitos escapados**        | Tendência ↓       | Bugs que passaram pelo review e chegaram a prod  |
| **Blockers por PR**           | ≤ 2 média         | Qualidade do código antes do review              |
| **Nit/Suggestion ratio**      | > 50% construtivos| Proporção de feedback construtivo vs. cosmético  |

---

## 8. Anti-Patterns

| Anti-pattern                                | Problema                                              | Solução                                              |
|---------------------------------------------|-------------------------------------------------------|------------------------------------------------------|
| **Rubber stamping** (aprovar sem ler)       | Bugs e problemas passam despercebidos                  | Exigir pelo menos 1 comentário por review            |
| **Gatekeeping** (bloquear por estilo)       | Desmoraliza o time, atrasa entregas                    | Automatizar estilo; focar em lógica e corretude      |
| **PRs enormes** (> 800 LOC)                | Review superficial, alto risco de bugs                 | Limitar a 200-400 LOC; dividir em PRs temáticos      |
| **Review como arena** (debates infinitos)   | Paralisa o desenvolvimento                             | Time-box de 2 rounds; se persistir, reunião rápida  |
| **Sem follow-up**                           | Sugestões aceitas mas nunca implementadas               | Criar ticket para follow-ups; rastrear               |
| **Review apenas do tech lead**              | Gargalo; ponto único de falha                          | Distribuir reviews; todo dev sênior pode aprovar     |
| **Feedback ambíguo** ("ficou estranho")     | Autor não sabe o que corrigir                          | Ser específico: problema + sugestão + justificativa  |
| **Ignorar testes no review**               | Testes ruins/ausentes passam despercebidos              | Incluir testes no checklist obrigatório              |
| **Review tardio** (> 48h)                  | Contexto perdido, merge conflicts, frustração          | SLA de 24h; alertas automáticos                      |
| **Comentários em arquivo errado**          | Confusão; autor não sabe onde agir                     | Comentar inline no código específico                 |
