---
nome: Análise de Causa-Raiz de Degradação
descricao: Analisa artefatos de diagnóstico (configuração, métricas e logs) para identificar a causa-raiz de uma degradação de sistema, separando causas de consequências
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [causa-raiz, incidente, diagnóstico, sre, análise]
inputs:
  - nome: config_sistema
    descricao: Arquivo de configuração do sistema com parâmetros relevantes (memória, agendamentos, cache)
  - nome: metricas
    descricao: Série temporal de métricas durante a janela do incidente
  - nome: logs
    descricao: Trecho de logs da aplicação cobrindo a janela do incidente
---

# Análise de Causa-Raiz de Degradação — CP03

## Objetivo

Recebe três artefatos de diagnóstico (configuração, métricas e logs) e produz um relatório de causa-raiz estruturado em 6 passos: leitura independente de cada artefato, linha do tempo integrada, separação causa/consequência/correlação, declaração da causa-raiz com evidência, ações proporcionais e limites do diagnóstico.

## Quando usar

- Quando um incidente foi contido mas a causa-raiz ainda não está clara
- Para escalação: antes de escalar para engenharia de dados, apresentar uma hipótese fundamentada
- Como insumo para o postmortem: o relatório gerado pode ser o ponto de partida do documento

## Técnica utilizada

**Chain-of-Thought de 6 passos (CoT estruturado).**

O prompt force o modelo a percorrer etapas explícitas antes de declarar a causa-raiz. O passo crítico é o Passo 3 (separação causa × consequência × correlação), que evita o erro mais comum em análise de incidente: tratar um sintoma como causa (ex.: "o cache hit caiu" não é causa — é efeito da pressão de memória).

### Por que CoT aqui?

A correlação de três fontes heterogêneas (config YAML, tabela de métricas, logs com timestamps) exige raciocínio sequencial. Se o modelo não for guiado a percorrer as etapas, tende a listar sintomas em vez de chegar à causa-raiz. O CoT explicito também torna o raciocínio auditável — o engenheiro pode verificar se o modelo realmente cruzou os dados.

## Dados sensíveis a sanitizar

Antes de enviar a um modelo externo:
- `cerebro-node-3` → `[NODE]`
- `logs-2026.05` → `[INDEX]`
- Nomes de tenants (se presentes nos logs) → `[TENANT]`
- O GOOGLE_API_KEY ou qualquer credential que apareça em logs → remover completamente

## Execução — modelo: Claude Sonnet 4.6 (Claude Code, 2026-08-27)

### Cenário: degradação do Cerebro em 2026-05-13

**Artefatos**: cerebro.yaml + métricas 08:00–10:00 + logs Elasticsearch

**Output:**

---

**Passo 1 — Leitura independente de cada artefato**

**Config (cerebro.yaml):**
- `jvm_heap: 8g` — teto de memória da JVM. É o parâmetro mais crítico para este cenário.
- `reindex_job` — agendado às 02:00, duração média 90min, previsto para terminar às ~03:30.
- `query_cache.size_mb: 512` — cache de queries. Vulnerável a evicções sob pressão de memória.

**Métricas:**
- `search_p99_ms`: degradação progressiva, de 850ms (08:00) para 6700ms (10:00) — 8x de piora.
- `indexed_docs_per_s`: salta de ~4200 para 12400 às 10:00 — alta carga de escrita contínua.
- `heap_used_pct`: 61% (08:00) → 94% (10:00), crescimento constante.
- `cache_hit_pct`: 74% (08:00) → 29% (10:00) — cache sendo esvaziado progressivamente.
- Padrão: `heap_used_pct` começa a crescer às 08:00; as demais métricas se deterioram em cascata depois.

**Logs:**
- 08:02: reindex task em 38% (6h após início — deveria ter terminado às 03:30).
- 08:14: primeiro GC young gen com pausa de 620ms.
- 08:41: throttling de indexação no shard 7.
- 09:03–09:51: write thread pool enchendo progressivamente (150→188/200).
- 09:58: thread pool FULL (200/200), GC old gen 1.8s, circuit breaker a 96%, buscas com timeout.
- 10:01–10:05: CircuitBreakingException, falhas completas em queries.

---

**Passo 2 — Linha do tempo integrada**

| Horário | Evento |
|---------|--------|
| 02:00 | Reindexação inicia (agendada) |
| 03:30 | ❌ Previsto para terminar — NÃO aconteceu |
| 08:00 | Reindex em 38%, heap já em 61% (2x acima do baseline esperado após terminar) |
| 08:41 | Throttling do indexer no shard 7 — sistema começa a lutar |
| 09:03 | Thread pool com 75% de capacidade ocupada |
| 09:58 | Colapso: thread pool full, GC old gen 1.8s, circuit breaker 96% |
| 10:00 | Métricas: p99=6700ms, heap=94%, cache_hit=29% |
| 10:05 | Falhas totais em 3/20 queries |

---

**Passo 3 — Separação causa × consequência × correlação**

**CAUSA**: O job de reindexação agendado às 02:00 não concluiu dentro da janela esperada de 90 minutos (deveria terminar às 03:30). Às 08:02, estava apenas em 38% do progresso. Continuou rodando durante todo o horário comercial, gerando carga de escrita sustentada e consumindo heap continuamente.

**CONSEQUÊNCIA** (em cascata):
1. Heap crescendo continuamente → GC pressure crescente (young gen → old gen)
2. GC pressure → evicções do query cache (hit ratio 74% → 29%)
3. Evicções do cache → mais trabalho computacional por query → mais heap alocado
4. Carga de escrita sustentada → thread pool enchendo (150 → 200/200)
5. Thread pool full → bulk requests rejeitados (EsRejectedExecutionException)
6. Heap a 96% → circuit breaker dispara → queries bloqueadas ou com resultado parcial

**CORRELAÇÃO (não causal)**: O pico de `indexed_docs_per_s` (4200 → 12400) coincide com a piora das métricas, mas é consequência da reindexação contínua — não é uma causa separada.

---

**Passo 4 — Causa-raiz**

> A reindexação agendada às 02:00 não concluiu dentro da janela esperada de ~90 minutos (prevista para terminar às 03:30). Às 08:02, o job estava apenas em 38% do progresso, ainda ativo mais de 6 horas após o início. Esse job em execução gerou carga de escrita sustentada que saturou progressivamente o heap da JVM de 8g, desencadeando GC pressure → evicção do query cache → saturação do write thread pool → circuit breaker → falhas de busca.
>
> Evidência direta: log `[08:02:11] reindex task (scheduled 02:00) progress, created [3.8M]/[10M] docs (38%)` — 6h após início, taxa de progresso de ~4% por hora contra os 67%/hora esperados.

---

**Passo 5 — Ações recomendadas**

**Contenção imediata**:
- Cancelar o job de reindexação em execução: `POST /_tasks/88123/_cancel`
- Monitorar `heap_used_pct` — deve começar a cair em minutos após cancelamento
- Aguardar estabilização do GC antes de reiniciar buscas com alta carga

**Correção estrutural**:
- Adicionar throttling ao job de reindexação (`indices.store.throttle.max_bytes_per_sec: 100mb`) para limitar impacto no heap
- Configurar abort automático: se o job não concluir em 3x a duração média (4,5h), alertar e pausar
- Revisar `jvm_heap: 8g` — considerar 12g para acomodar reindexação + queries no mesmo nó

**Prevenção futura**:
- Alerta: se reindex job não concluir até 06:00, escalar para @search-infra antes do horário comercial
- Monitor: `heap_used_pct > 70%` deve disparar alerta (o problema se tornou crítico em 88%+)
- Investigar: por que esse job levou 8h+ quando a média é 90min (ver Passo 6)

---

**Passo 6 — Limites do diagnóstico**

Os artefatos disponíveis NÃO permitem concluir:
1. **Por que o job de reindexação atrasou tanto**: os logs não mostram a causa do atraso em si — pode ser volume de dados maior que o esperado, gargalo de I/O no disco, restart de nó que resetou progresso, ou outra causa. É necessário checar logs do nó de 02:00–08:00 para investigar.
2. **Se outros nós do cluster foram afetados**: os logs cobrem apenas `cerebro-node-3`. Outros nós podem ter degradado de forma diferente.
3. **Se o volume de dados para reindexar cresceu recentemente**: sem histórico de execuções anteriores do job, não é possível saber se 10M docs é normal ou anormal.

---

## Curadoria

**Modelo**: Claude Sonnet 4.6 (Anthropic) para execução manual; `google:gemini-2.5-flash-lite` (Google) como avaliado e juiz nos testes automatizados.

**Justificativa de modelo**: Análise de causa-raiz exige raciocínio de múltiplos passos com dados de três fontes heterogêneas — é a tarefa de maior complexidade do playbook. O `gemini-2.5-flash` (modo thinking) seria ideal em termos de qualidade, mas sua latência de ~53s excede o threshold operacional de 30s. O `gemini-2.5-flash-lite` sem thinking produziu análise igualmente correta (8/8 no juiz) em ~13s. Claude Sonnet 4.6 seria outra opção de qualidade, mas requer workspace-id em keys identity-linked — não configurável anonimamente no promptfoo. **Dados sensíveis**: logs com IPs internos, nomes de índices e configurações de heap devem ser sanitizados ou anonimizados antes de enviar a modelos externos.

**Meta-prompting**: O prompt foi gerado com a diretriz: _"Crie um prompt de análise de causa-raiz que recebe três artefatos (config, métricas, logs) e guia a IA a chegar à causa-raiz sem parar nos sintomas. O passo mais crítico é separar causa de consequência. O modelo deve ser honesto sobre o que não pode concluir."_

**Refinamentos:**
1. A v1 pedia análise livre sem estrutura. O modelo listava sintomas como causas. Solução: introduzir os 6 passos obrigatórios com o Passo 3 explícito para separação causa/consequência.
2. O Passo 6 (limites) não estava na v1 — o modelo tendia a fabricar certeza sobre a causa do atraso da reindexação. Adicionado explicitamente: "Não fabrique certeza onde há dúvida."
3. O Passo 1 foi refinado para pedir leitura **independente** por artefato, evitando que o modelo saltasse direto para conclusões sem analisar cada fonte.

## Gate de qualidade — LLM-as-judge (CP09)

**Rubrica** (4 critérios, escala 0–2 cada, total 0–8):
1. **Causa-raiz correta** (0-2): identifica a reindexação travada como causa, não os sintomas
2. **Correlação × causa** (0-2): separa explicitamente causas de consequências
3. **Ação proporcional** (0-2): menciona cancelar o job + correção estrutural
4. **Honestidade epistêmica** (0-2): reconhece a lacuna de diagnóstico (causa do atraso)

**Critério de aprovação**: total ≥ 6 AND nenhum critério com nota 0.

**Calibração do juiz** — comparação nota humana × nota do juiz (`google:gemini-2.5-flash-lite`):

| Critério | Nota humana | Nota do juiz | Δ | Justificativa do juiz |
|----------|-------------|-------------|---|----------------------|
| 1 — Causa-raiz correta | 2 | 2 | 0 | "identifica corretamente o job de reindexação como causa raiz, distinguindo-o dos sintomas como GC pressure e circuit breaker" |
| 2 — Correlação × causa | 2 | 2 | 0 | "separa explicitamente causa, consequência e correlação, listando claramente o job como causa e problemas de memória/thread pool como consequências" |
| 3 — Ação proporcional | 2 | 2 | 0 | "inclui contenção imediata (pausar/cancelar o job) e correções estruturais (otimizar o job, aumentar o heap)" |
| 4 — Honestidade epistêmica | 2 | 2 | 0 | "reconhece explicitamente as limitações do diagnóstico, como a causa exata do aumento da taxa de indexação e a razão do atraso do job" |
| **Total** | **8/8** | **8/8** | **0** | **PASSA (corte ≥ 6, nenhum critério zerado)** |

Δ máximo por critério: 0 — dentro do limite de calibração (≤1 ponto). O prompt do juiz não precisou de ajuste após a primeira calibração.

## Limitações

- O prompt assume que os artefatos cobrem a mesma janela temporal. Artefatos de janelas diferentes podem levar a análises incorretas.
- Para sistemas além do Elasticsearch/JVM, os exemplos do Passo 1 da análise (heap, GC, thread pool) não são diretamente aplicáveis — a descrição de como ler os logs deve ser adaptada.
- A análise não substitui um postmortem completo: não cobre impacto em SLA de clientes nem cronograma de remediação.

## Testes

```bash
export GOOGLE_API_KEY="sua-chave"
promptfoo eval --config ./promptfooconfig.yaml
```

**Resultado real (2026-08-28, `google:gemini-2.5-flash-lite`):**

```
✓ 1 passed (100%) — 0 failed — 0 errors
Duration: ~13s (concurrency: 4)
Total tokens: 10.380 (Eval: 5.872 | Grading: 4.508)
```

| Entrada | Gate | Score do juiz | Resultado |
|---------|------|---------------|-----------|
| Degradação Cerebro (reindexação) | llm-rubric ≥6, nenhum critério 0 | 8/8 (2+2+2+2) | ✅ PASS |

**Provedores ao longo do desafio (CP03 + CP09):**
- **Execução manual (CP03)**: Claude Sonnet 4.6 via Claude Code (Anthropic)
- **Testes automatizados (CP09)**: `google:gemini-2.5-flash-lite` como avaliado e como juiz (Google)
- Dois fornecedores distintos: Anthropic + Google ✅

Ver [promptfooconfig.yaml](./promptfooconfig.yaml).
