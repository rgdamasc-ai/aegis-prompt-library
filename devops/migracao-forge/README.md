---
nome: Migração Forge — Batch para Event-Driven (Cadeia de 3 Prompts)
descricao: Cadeia de três prompts encadeados para planejar e detalhar a migração do pipeline Forge de processamento batch para event-driven, de forma incremental e reversível
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [forge, migração, batch, event-driven, cadeia]
inputs:
  - nome: estado_atual_forge
    descricao: Descrição detalhada do estado atual do Forge (Passo 1 e 2)
  - nome: diagnostico_forge
    descricao: Saída do Passo 1 — diagnóstico completo (entrada do Passo 2)
  - nome: plano_migracao
    descricao: Saída do Passo 2 — plano faseado (entrada do Passo 3)
  - nome: fase_alvo
    descricao: Fase específica do plano para detalhar no Passo 3
---

# Migração Forge — Batch para Event-Driven — CP05

## Objetivo

Cadeia de três prompts encadeados para decompor uma migração técnica complexa em partes tratáveis:

| Passo | Arquivo | Entrada | Saída |
|-------|---------|---------|-------|
| 1 — Diagnóstico | [prompt-01-diagnostico.md](./prompt-01-diagnostico.md) | `{{estado_atual_forge}}` | Mapa de riscos + gaps + síntese |
| 2 — Plano | [prompt-02-plano.md](./prompt-02-plano.md) | `{{estado_atual_forge}}` + `{{diagnostico_forge}}` | Plano faseado reversível |
| 3 — Execução | [prompt-03-execucao.md](./prompt-03-execucao.md) | `{{plano_migracao}}` + `{{fase_alvo}}` | Passos executáveis + rollback |

## Como usar a cadeia

```
1. Rodar Passo 1 com o estado atual do Forge
   → Guardar a saída como {{diagnostico_forge}}

2. Rodar Passo 2 com o estado atual + saída do Passo 1
   → Guardar a saída como {{plano_migracao}}

3. Rodar Passo 3 com o plano + a fase que deseja detalhar
   → Resultado: passos executáveis com validações e rollback
```

## Quando usar

- Antes de propor uma migração técnica grande: gere o diagnóstico e veja se os riscos são aceitáveis
- Em reunião de planejamento: use a saída do Passo 2 como base para discussão
- No início de cada fase de execução: rode o Passo 3 para ter os passos detalhados

## Técnica utilizada

**Encadeamento de prompts (prompt chaining).**

Uma única pergunta "como migrar o Forge de batch para event-driven" resultaria em uma resposta genérica. Ao quebrar em três prompts encadeados — diagnóstico → plano → execução — cada elo recebe o resultado do anterior como contexto e pode ser mais específico e útil.

### Por que cadeia de prompts aqui?

A migração tem três camadas de raciocínio distintas:
1. **O que está em jogo** (diagnóstico): entender o estado atual antes de propor qualquer mudança
2. **O que fazer e em que ordem** (plano): sequenciar as fases sem big-bang
3. **Como fazer concretamente** (execução): passos, validações e rollback por fase

Misturar as três camadas em um único prompt resulta em output raso nas três. Separar permite profundidade em cada uma.

## Dados sensíveis a sanitizar

- Nomes internos de jobs (ex.: `forge-batch-ingest`) → manter (descritivos, não sensíveis)
- Nomes de sistemas dependentes (Sentinel, Cerebro, billing) → manter (contexto necessário)

## Execução — modelo: Claude Sonnet 4.6 (Claude Code, 2026-08-27)

### Passo 1 — Diagnóstico

**Entrada `{{estado_atual_forge}}`:**
```
Forge hoje:
- ingestão: um job em cron acorda a cada 60min (o "forge-batch-ingest")
- transformação: 14 etapas de processamento encadeadas (em Spark), ~40min no total
- destino: grava em tabelas no data warehouse, particionadas por hora
- ponto frágil: se um lote falha, o próximo acumula o dobro de volume
- quem depende do Forge: Sentinel (lê as tabelas agregadas), Cerebro (indexa os eventos
  transformados) e os relatórios de billing da Pepper (rodam de madrugada)
```

**Output:**

**1. Mapa de dependências**

| Consumidor | Dado consumido | Frequência | Impacto de atraso/parada |
|---|---|---|---|
| Sentinel | Tabelas agregadas de métricas/alertas | A cada ciclo de 60min | Alertas e dashboards desatualizados — clientes perdem visibilidade |
| Cerebro | Eventos transformados para indexação | A cada ciclo de 60min | Busca de logs desatualizada — investigação de incidentes prejudicada |
| Billing (Pepper) | Tabelas de uso por tenant | Diário (madrugada) | Relatórios de billing incorretos — impacto financeiro direto |

**2. Pontos de risco na migração**

| Risco | Probabilidade | Impacto |
|---|---|---|
| **Cascata de volume**: o event-driven recebe o backlog acumulado do Relay de uma vez ao iniciar | Alta — o Relay tem retenção de 4h de mensagens | Sobrecarga do novo consumer, podendo causar os mesmos problemas que o batch |
| **Processamento não-idempotente**: as 14 etapas Spark foram escritas assumindo batch — podem não ser idempotentes para eventos individuais | Alta — pipeline batch raramente é projetado para idempotência | Duplicação de dados nas tabelas do data warehouse |
| **Schema drift**: eventos chegando do Relay com schemas diferentes dos esperados pelas 14 etapas | Média — Relay recebe dados de múltiplos tenants | Falha silenciosa de transformação |
| **Degradação do Cerebro durante transição**: se o Forge mudar a frequência de entrega, o Cerebro pode receber rajadas em vez de fluxo contínuo | Média | Latência de indexação variável |
| **Relatórios de billing com granularidade errada**: billing depende de particionamento por hora — event-driven pode alterar esse padrão | Média — depende de como o destino é escrito | Relatórios inconsistentes ou quebrados |

**3. Gaps de pré-requisito**

- **Processamento idempotente**: as 14 etapas Spark precisam ser refatoradas para lidar com re-entrega de eventos sem duplicação
- **Gestão de consumer offsets**: precisa de mecanismo para rastrear onde o consumer parou, para reiniciar sem reprocessar ou pular
- **Schema registry / validação de schema**: garantir que eventos do Relay tenham schema validado antes de entrar nas etapas de transformação
- **Particionamento temporal do destino**: o destino atual é particionado por hora (batch). No event-driven, precisa de lógica de micro-batch ou streaming write para manter compatibilidade

**4. Avaliação de reversibilidade**

- Enquanto o sistema batch ainda roda: alta reversibilidade — o event-driven pode rodar em shadow mode sem afetar o batch
- Após o cut-over de um consumidor (ex. Cerebro): reversibilidade média — precisa de rollback coordenado entre Forge e Cerebro
- Após decommission do batch: irreversível sem re-deploy completo — esta janela deve ser mantida aberta por pelo menos 2 semanas após o último consumer migrar

**5. Síntese**

O Forge **não está pronto para migrar diretamente**. Os três gaps mais críticos são: (1) as 14 etapas Spark não são idempotentes — duplicação é certa sem refatoração; (2) sem gestão de consumer offsets, qualquer restart do novo consumer vai reprocessar ou perder dados; (3) o particionamento por hora do destino precisa ser preservado para não quebrar billing. O maior risco é o de cascata de volume ao iniciar o consumer: o Relay tem 4h de backlog que serão entregues de uma vez. A migração deve começar pelo shadow mode (consumer rodando em paralelo sem gravar no destino), apenas observando, antes de qualquer corte.

---

### Passo 2 — Plano de migração

**Entrada `{{diagnostico_forge}}`**: saída do Passo 1 acima

**Output:**

**Fase 1 — Consumer em modo shadow (semanas 1-2)**

*Objetivo*: Deploy do novo consumer event-driven lendo do Relay, mas sem gravar no destino. Valida que o consumer consegue acompanhar o throughput e que os schemas são compatíveis.

*Critério de avanço*: Consumer processa 100% dos eventos do Relay sem lag > 30s por 24h consecutivas, sem erros de schema.

*Critério de rollback*: Se o consumer acumular lag > 5min ou apresentar taxa de erros > 1%, desligar o consumer (não afeta o batch — o batch continua como fonte de verdade).

*Proteção dos consumidores*: Batch continua rodando normalmente. Shadow consumer não grava em lugar nenhum.

---

**Fase 2 — Refatoração de idempotência + escrita paralela (semanas 3-5)**

*Objetivo*: Refatorar as 14 etapas para serem idempotentes + configurar o consumer para gravar em tabelas de destino separadas (ex.: `forge_streaming_*`), paralelas às tabelas batch.

*Critério de avanço*: Dados das tabelas `forge_streaming_*` são estatisticamente equivalentes às tabelas batch (diferença < 0,1% em contagem de eventos e valores de métricas chave) por 48h.

*Critério de rollback*: Parar escrita nas tabelas `forge_streaming_*`. As tabelas batch continuam intactas — rollback sem impacto em consumidores.

*Proteção dos consumidores*: Sentinel e Cerebro continuam lendo das tabelas batch. Billing não é afetado.

---

**Fase 3 — Cut-over do Cerebro e Sentinel (semanas 6-8)**

*Objetivo*: Migrar Cerebro e Sentinel para ler das tabelas `forge_streaming_*` via feature flag. Manter batch rodando como fallback.

*Critério de avanço*: Cerebro e Sentinel operando normalmente a partir das tabelas de streaming por 72h sem alertas de divergência.

*Critério de rollback*: Feature flag reverte Cerebro e Sentinel para tabelas batch em < 5min.

*Proteção dos consumidores*: Billing continua lendo tabelas batch. Feature flag garante reversão rápida.

---

**Fase 4 — Decommission do batch e consolidação (semanas 9-10)**

*Objetivo*: Desligar o `forge-batch-ingest` após billing ser migrado e janela de observação passar.

*Critério de avanço*: Billing validado com dados de streaming por 2 ciclos completos. Nenhum alerta de divergência em 2 semanas.

*Critério de rollback*: Reativar o cron do `forge-batch-ingest`. Dados do batch estarão desatualizados pelo período offline — precisa de reprocessamento parcial.

*Risco residual*: Após decommission, rollback é custoso. Esta é a janela de menor reversibilidade.

---

### Passo 3 — Execução da Fase 1

**Entrada `{{fase_alvo}}`**: "Fase 1 — Consumer em modo shadow (semanas 1-2)"

**Output:**

**Pré-requisitos**

- [ ] Acesso ao Relay (configurações de consumer group no broker)
- [ ] Repositório do Forge com branch de feature criado
- [ ] Monitoring configurado para o novo consumer (métricas de consumer lag, erros de schema)
- [ ] Comunicação ao time sobre início da Fase 1

**Passos de execução**

1. **Criar consumer group dedicado** no broker do Relay com nome `forge-streaming-shadow`
   - Validação: `kafka-consumer-groups --describe --group forge-streaming-shadow` mostra offsets válidos
   - Tempo: 15 minutos

2. **Implementar consumer em shadow mode** (sem escrita no destino): consumer lê eventos, valida schema, registra métricas de throughput e erros — não persiste nada
   - Validação: consumer está rodando, logs mostram eventos sendo processados, sem erros de schema nos primeiros 100 eventos
   - Tempo: 4-8 horas de desenvolvimento + 1h de deploy

3. **Deploy do consumer em modo shadow** no ambiente de produção com recursos limitados (2 CPUs, 4GB RAM)
   - Validação: consumer aparece nos dashboards, consumer lag iniciando em algum ponto do histórico de retenção (4h)
   - Tempo: 30 minutos de deploy + até 4h para consumer alcançar o fim da fila

4. **Observar o catch-up** até o consumer estar processando eventos em tempo real (lag < 30s)
   - Validação: `kafka-consumer-groups --describe --group forge-streaming-shadow` mostra lag < 100 por partição
   - Tempo: até 4h (tamanho da retenção atual)

5. **Monitorar por 24h** em modo shadow: throughput, taxa de erro de schema, lag
   - Validação: lag < 30s por 24h consecutivas; taxa de erros de schema < 0,1%
   - Tempo: 24h de observação

**Checklist de avanço para Fase 2**

- [ ] Consumer processando 100% dos eventos (lag < 30s por 24h)
- [ ] Taxa de erros de schema < 0,1%
- [ ] Nenhum OOM ou restart do consumer em 24h
- [ ] Throughput do consumer compatível com o throughput do batch (mesma ordem de magnitude de eventos/hora)

**Procedimento de rollback**

1. Escalar o deployment do consumer para 0 réplicas: `kubectl scale deployment forge-streaming-shadow --replicas=0 -n forge-prod`
2. Verificar que não há consumers ativos no grupo: `kafka-consumer-groups --describe --group forge-streaming-shadow`
3. Notificar o time: rollback da Fase 1 executado, sem impacto em produção (batch continua)
4. **Tempo estimado**: < 5 minutos. **Impacto**: zero — batch nunca parou.

**Monitoramento durante a Fase 1**

| Métrica | Valor esperado | Alerta |
|---------|---------------|--------|
| consumer lag (forge-streaming-shadow) | < 30s após catch-up | > 5min por mais de 10min |
| schema errors/min | < 0,1% dos eventos | > 1% |
| consumer restarts | 0 | qualquer restart |
| heap do consumer | < 70% | > 85% |

---

## Curadoria

**Meta-prompting**: A diretriz foi: _"Crie uma cadeia de 3 prompts para planejar a migração do Forge. O Prompt 1 deve diagnosticar o estado atual, o 2 deve gerar o plano a partir do diagnóstico, e o 3 deve detalhar os passos executáveis para uma fase específica. O encadeamento é obrigatório — cada elo recebe a saída do anterior."_

**Refinamentos:**
1. A v1 tinha apenas 2 prompts (diagnóstico + plano). O Passo 3 foi adicionado após perceber que o plano ficava abstrato demais para ser executável diretamente.
2. O Passo 2 v1 não incluía "Proteção dos consumidores" por fase. Adicionado como seção obrigatória após perceber que era o aspecto mais crítico para a stakeholder Pepper Potts (billing).
3. O Passo 1 v1 não pedia "Avaliação de reversibilidade" — foi adicionado para forçar o diagnóstico a pensar nos pontos de retorno da migração.

## Limitações

- A cadeia assume que o sistema de mensageria é Kafka-like. Para outros brokers (RabbitMQ, Kinesis, Pub/Sub), os nomes de comandos e conceitos precisam ser adaptados.
- O Passo 3 detalha uma fase de cada vez — para detalhar todas as 4 fases, rodar o Passo 3 quatro vezes com `{{fase_alvo}}` diferente.
- Saída aberta — não testável por regex. Requer revisão humana ou LLM-judge para validação de qualidade.
