---
nome: Triagem de Pods Kubernetes
descricao: Triagem da saúde dos pods de um cluster Kubernetes, identificando causa provável e próxima ação acionável para cada pod problemático
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [kubernetes, sre, triagem, pods, operacional]
inputs:
  - nome: snapshot_cluster
    descricao: Saída combinada de kubectl get pods, kubectl describe e kubectl logs de todos os pods do namespace monitorado
---

# Triagem de Pods Kubernetes — CP01

## Objetivo

Recebe um snapshot do cluster (saída de `kubectl get pods`, `kubectl describe` e `kubectl logs`) e devolve uma triagem estruturada por pod: status, causa provável (cruzando as três fontes) e próxima ação recomendada. Funciona tanto para clusters com problemas quanto para clusters saudáveis.

## Quando usar

- Ao início de um turno de plantão para verificar a saúde dos pods do Sentinel
- Quando um alerta de pod em falha é recebido
- Como ponto de partida para investigação de incidentes em Kubernetes

## Técnica utilizada

**Chain-of-Thought + saída estruturada.**

O prompt instrui o modelo a percorrer explicitamente as três fontes de evidência (status → eventos → logs) antes de concluir sobre a causa. Isso evita que o modelo repita o STATUS como "causa" (ex. "a causa é CrashLoopBackOff") e força a chegada ao motivo técnico subjacente. O formato de saída fixo (campos POD/STATUS/CAUSA/AÇÃO) garante leitura imediata em situação de incidente.

### Por que Chain-of-Thought aqui?

A triagem de pods exige correlação de múltiplas fontes heterogêneas em linguagem livre. O CoT explicita o raciocínio, tornando o output auditável: o plantonista pode checar se o modelo cruzou os logs com os eventos antes de confiar na recomendação.

## Dados sensíveis a sanitizar

Antes de enviar a um modelo externo (OpenAI, Google, etc.), substituir ou remover:
- Nomes de namespaces internos (ex.: `sentinel-prod` → `[NAMESPACE]`)
- Nomes de imagens com repositório privado (ex.: `registry.aegis.io/...` → `[REGISTRY]/...`)
- Hostnames internos de nodes

## Execução — modelo: Claude Sonnet 4.6 (Claude Code, 2026-08-27)

### Entrada 1 — pod em CrashLoopBackOff (OOMKilled)

**Output:**

```
---
POD: sentinel-api-7d9c8b6f4-h4m2t
STATUS: CrashLoopBackOff — 14 reinicializações (última há 90s)
CAUSA PROVÁVEL: O container está sendo morto pelo kernel por exceder o limite de memória de 512Mi (OOMKilled, exit code 137). Os logs confirmam: o cache de alertas de 6h carregou 410Mi de heap na inicialização, empurrando o uso para 498Mi/512Mi antes do processo ser finalizado. O pod reinicia, tenta carregar o mesmo cache e é morto novamente — loop determinístico de falha por pressão de memória.
AÇÃO RECOMENDADA: Aumentar o memory limit do container de 512Mi para pelo menos 768Mi (ex.: kubectl patch deployment sentinel-api -n sentinel-prod -p '{"spec":{"template":{"spec":{"containers":[{"name":"api","resources":{"limits":{"memory":"768Mi"}}}]}}}}'), ou reduzir o tamanho do cache de alertas na configuração do sentinel-api. Verificar se houve crescimento recente no volume de alertas que justifique o consumo adicional.
---
```

### Entrada 2 — dois pods não sobem (ImagePullBackOff + Pending)

**Output:**

```
---
POD: sentinel-api-7d9c8b6f4-zzp10
STATUS: ImagePullBackOff
CAUSA PROVÁVEL: A tag 2.9.2 da imagem não existe no registry privado (manifest unknown). Um deploy referenciou uma imagem que não foi publicada em registry.aegis.io — o kubelet não consegue fazer pull e entra em backoff exponencial.
AÇÃO RECOMENDADA: Verificar se a imagem foi publicada: docker manifest inspect registry.aegis.io/sentinel-api:2.9.2. Se não foi publicada, reverter o deployment para 2.9.1: kubectl set image deployment/sentinel-api api=registry.aegis.io/sentinel-api:2.9.1 -n sentinel-prod && kubectl rollout status deployment/sentinel-api -n sentinel-prod.
---

---
POD: sentinel-ingest-8f7a9c5b-4dkqm
STATUS: Pending — sem scheduling há 8 minutos
CAUSA PROVÁVEL: O pod solicita 2 CPUs, mas todos os 6 nós do cluster estão sem CPU disponível suficiente (0/6 nodes available: Insufficient cpu). O scheduler não consegue alocar o pod em nenhum nó.
AÇÃO RECOMENDADA: Verificar utilização atual com kubectl top nodes. Avaliar: (1) reduzir requests.cpu do sentinel-ingest se o valor está superdimensionado para a carga real; (2) acionar o cluster autoscaler para provisionar um nó adicional; (3) identificar pods ociosos que possam ser drenados para liberar CPU.
---
```

### Entrada 3 — cluster saudável

**Output:**

```
---
STATUS DO CLUSTER: Saudável
OBSERVAÇÃO: Todos os 4 pods estão em execução normal. O pod sentinel-worker-5b8f9c8d-jq7rs teve 1 reinicialização há 3 dias — dentro do limiar normal, sem recorrência. Os logs da API mostram health check OK sem erros nos últimos 5 minutos e p99 de latência em 120ms. Nenhuma ação necessária.
---
```

## Curadoria

**Modelo**: Claude Sonnet 4.6 (Anthropic) para execução manual; `google:gemini-2.5-flash-lite` (Google) nos testes automatizados.

**Justificativa de modelo**: Triagem de pods exige correlação de múltiplas fontes de dados (status + eventos + logs) com raciocínio estruturado — não é tarefa trivial para modelos menores. O `gemini-2.5-flash-lite` respondeu corretamente em ~14s mesmo sem modo thinking, mantendo custo baixo. Para uso em incidentes críticos com dados sensíveis internos (hostnames, nomes de tenants), considerar modelo local via Ollama ou Gemini Vertex AI com dados permanecem na região.

**Meta-prompting**: O prompt foi gerado com a seguinte diretriz à IA: _"Crie um prompt parametrizável para SRE que recebe um snapshot de kubectl (get pods + describe + logs) e produz triagem estruturada por pod: identifica problemas, cruza evidências das três fontes e recomenda ação concreta. O modelo não deve repetir o STATUS como causa — deve chegar ao motivo técnico subjacente."_

**Refinamentos após primeira versão:**
1. A v1 do prompt não incluía a instrução explícita "não repita o STATUS". O modelo estava escrevendo "CAUSA: CrashLoopBackOff" ao invés de explicar o OOMKilled. Adicionado: "não repita o STATUS — chegue ao motivo técnico subjacente."
2. A v1 não mencionava o caso saudável. Adicionado: "Se todos os pods estiverem saudáveis, declare isso explicitamente."
3. O formato de saída foi fixado em template após a v1 gerar outputs com formatação inconsistente entre chamadas.

## Limitações

- O prompt assume que o snapshot foi coletado com kubectl describe e kubectl logs — sem esses dados, a análise se torna superficial.
- Não analisa recursos além de pods (Deployments, Services, Ingress). Para análise completa de workload, usar o prompt `diagnosticar-erros-eks`.
- Reinicializações antigas (ex.: 1 restart há 3 dias) podem ser ruído — o plantonista deve avaliar o contexto.

## Testes

```bash
export GOOGLE_API_KEY="sua-chave"
promptfoo eval --config ./promptfooconfig.yaml
```

**Resultado real (2026-08-28, `google:gemini-2.5-flash-lite`):**

```
✓ 3 passed (100%) — 0 failed — 0 errors
Duration: ~14s (concurrency: 4)
```

| Entrada | Asserts verificados | Resultado |
|---------|---------------------|-----------|
| 1 — OOMKilled | pod h4m2t + OOMKilled/memória + POD/CAUSA/AÇÃO labels | ✅ PASS |
| 2 — ImagePullBackOff + Pending | pods zzp10 + 4dkqm + imagem 2.9.2 + cpu insuf. | ✅ PASS |
| 3 — cluster saudável | "saudável" + sem POD: + sem CrashLoopBackOff | ✅ PASS |

**O que foi ajustado durante os testes:**
- `(?i)` inválido em JavaScript: asserts `regex` com flag `(?i)` convertidos para `javascript` asserts com `/pattern/i.test(output)`.
- Assertion `cost` removida: provedores Google não reportam custo por chamada no promptfoo.
- `gemini-2.5-flash` removido (modo thinking ~53s excede threshold 15s); `gemini-3.6-flash` substituído por `gemini-2.5-flash-lite` (503 intermitentes).

**Provedores ao longo do desafio (CP01):**
- **Execução manual (CP01)**: Claude Sonnet 4.6 via Claude Code (Anthropic)
- **Testes automatizados (CP08)**: `google:gemini-2.5-flash-lite` (Google)
- Dois fornecedores distintos: Anthropic + Google ✅

Ver [promptfooconfig.yaml](./promptfooconfig.yaml) para os asserts de cada entrada.
