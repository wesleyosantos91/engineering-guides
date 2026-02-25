# Kubestronaut — Visão Geral e Estratégia

## O que é Kubestronaut?

O título **Kubestronaut** é concedido pela CNCF a profissionais que passam em **todas as 5 certificações Kubernetes**:

| # | Certificação | Sigla | Nível | Foco |
|---|-------------|-------|-------|------|
| 1 | Kubernetes and Cloud Native Associate | **KCNA** | Associate | Conceitos fundamentais |
| 2 | Certified Kubernetes Application Developer | **CKAD** | Practitioner | Desenvolvimento de apps |
| 3 | Certified Kubernetes Administrator | **CKA** | Practitioner | Administração de clusters |
| 4 | Certified Kubernetes Security Specialist | **CKS** | Specialist | Segurança |
| 5 | Kubernetes and Cloud Native Security Associate | **KCSA** | Associate | Fundamentos de segurança |

## Benefícios do Kubestronaut

- **Reconhecimento** — título exclusivo da CNCF.
- **Comunidade** — acesso ao grupo de Kubestronauts.
- **Merchandise** — jacket exclusiva da CNCF.
- **KubeCon** — convites para eventos exclusivos.
- **Carreira** — diferencial competitivo significativo.
- **Credly badges** — todas as 5 certificações no perfil.

## Ordem Recomendada de Estudo

```
Fase 1 (Fundamentos):
  KCNA → Conceitos cloud native, K8s basics
  ↓
Fase 2 (Practitioner):
  CKAD → Desenvolver apps no K8s
  CKA  → Administrar clusters K8s
  ↓
Fase 3 (Segurança):
  KCSA → Fundamentos de segurança
  CKS  → Segurança avançada (requer CKA ativo)
```

### Timeline Sugerida
| Fase | Duração | Cert |
|------|---------|------|
| Mês 1-2 | 4-6 semanas | KCNA |
| Mês 3-4 | 6-8 semanas | CKAD |
| Mês 5-7 | 8-10 semanas | CKA |
| Mês 8-9 | 4-6 semanas | KCSA |
| Mês 10-12 | 8-10 semanas | CKS |

**Total**: ~10-12 meses de estudo dedicado.

## Formato dos Exames

| Cert | Formato | Duração | Passing Score | Prática? |
|------|---------|---------|:---:|:---:|
| KCNA | Multiple choice | 90 min | 75% | ❌ |
| CKAD | Hands-on (terminal) | 2h | 66% | ✅ |
| CKA | Hands-on (terminal) | 2h | 66% | ✅ |
| CKS | Hands-on (terminal) | 2h | 67% | ✅ |
| KCSA | Multiple choice | 90 min | 75% | ❌ |

## Ambiente do Exame (Hands-on)

### PSI Bridge Exam Environment
- **Browser-based terminal** — não é sua máquina.
- **Webcam e microfone** obrigatórios.
- **Uma aba extra** permitida para documentação oficial:
  - kubernetes.io/docs
  - kubernetes.io/blog
  - github.com/kubernetes
  - helm.sh/docs
  - falco.org/docs (CKS)
  - aquasecurity.github.io/trivy (CKS)
  - gitlab.com/apparmor (CKS)

### Dicas para o Ambiente
- Pratique com **kubectl** extensivamente — velocidade importa.
- Configure **aliases** no início do exame:
  ```bash
  alias k=kubectl
  alias kn='kubectl config set-context --current --namespace'
  export do='--dry-run=client -o yaml'
  export now='--force --grace-period=0'
  ```
- Use **`kubectl explain`** para referência rápida de campos.
- Use **`kubectl create`** e **`kubectl run --dry-run`** para gerar YAML.
- **Copie/cole** da documentação oficial — não memorize YAML.

## Recursos Gerais

### Plataformas de Prática
| Plataforma | Descrição | Preço |
|-----------|-----------|-------|
| **killer.sh** | Simuladores oficiais (incluso no voucher) | Incluso |
| **KodeKloud** | Cursos + labs | ~$20/mês |
| **Killercoda** | Cenários interativos gratuitos | Grátis |
| **Play with Kubernetes** | Labs gratuitos | Grátis |
| **Katacoda alternatives** | Ambientes hands-on | Vários |

### Cursos Recomendados
| Plataforma | Instrutor | Certs |
|-----------|-----------|-------|
| **KodeKloud** | Mumshad Mannambeth | CKA, CKAD, CKS |
| **Udemy** | Mumshad Mannambeth | CKA, CKAD, CKS |
| **Linux Foundation** | Oficial | Todos |
| **A Cloud Guru** | Vários | CKA, CKAD |
| **Kubernetes The Hard Way** | Kelsey Hightower | CKA/CKS deep |

## Dicas Gerais para Todas as Provas

1. **Pratique, pratique, pratique** — hands-on é 80% da preparação.
2. **killer.sh** — faça o simulador 2x antes do exame.
3. **Tempo** — gerencie o tempo rigorosamente. Pule questões difíceis.
4. **Documentação** — saiba navegar kubernetes.io rapidamente.
5. **kubectl** — domine imperative commands para velocidade.
6. **YAML** — gere com dry-run, não escreva do zero.
7. **Contexto** — sempre verifique o namespace/cluster correto.
8. **Pesos** — priorize questões de maior peso.
9. **Revisão** — reserve 15-20 min para revisar.
10. **Flag** — marque questões para revisão e siga em frente.

## Validade

- **CKA/CKAD/CKS**: 2 anos.
- **KCNA/KCSA**: 3 anos.
- Para manter o título **Kubestronaut**, todas devem estar ativas.
- Planeje **renovações** antes dos vencimentos.
