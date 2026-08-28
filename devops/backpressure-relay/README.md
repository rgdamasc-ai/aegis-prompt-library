---
nome: Estratégia de Backpressure para Relay
descricao: Compara estratégias para controlar sobrecarga no barramento de eventos (backpressure), pesando prós e contras de cada abordagem antes de recomendar, respeitando SLAs e restrições do time
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [backpressure, relay, decisão, arquitetura, mensageria]
inputs:
  - nome: estado_relay
    descricao: Estado atual do barramento de eventos (throughput, pico, retenção, consumidores)
  - nome: restricoes_time
    descricao: Restrições da solução (SLAs, orçamento, restrições técnicas)
---

# Estratégia de Backpressure para Relay — CP04

## Objetivo

Recebe o estado atual do barramento de eventos e as restrições do time, e produz uma análise comparativa de quatro estratégias de backpressure (fila com prioridade, DLQ, particionamento por tenant, autoscaling) antes de recomendar a abordagem. O valor está no raciocínio comparativo — a recomendação final é consequência da análise, não um chute inicial.

## Quando usar

- Quando um pico de volume derrubou o barramento e o time precisa decidir como prevenir a recorrência
- Antes de uma reunião de arquitetura: o prompt gera o comparative analysis em minutos
- Ao avaliar trade-offs para uma decisão de infra cara (antes de gastar orçamento)

## Técnica utilizada

**Análise comparativa estruturada (Tree-of-Thoughts aplicado a decisão).**

O prompt força a avaliação de CADA estratégia em CADA dimensão antes de qualquer recomendação. Isso evita o viés de ancoragem (chegar na primeira solução que parece boa e não avaliar as demais). A estrutura é: avalia A, avalia B, avalia C, avalia D — só então recomenda.

### Por que análise comparativa aqui?

Backpressure é um problema com múltiplas soluções defensáveis e trade-offs reais (custo × latência × complexidade × SLA). Um prompt de resposta única tenderia a retornar a primeira solução "boa o suficiente" em vez de pesá-las contra as restrições específicas do time. A análise comparativa expõe o raciocínio, tornando a decisão auditável e contestável.

## Dados sensíveis a sanitizar

- Nomes de tenants específicos (ex.: stark-industries) → `[TENANT]`
- Percentuais de custo orçamentário podem ser sensíveis → manter como relatividade (ex.: "acima do previsto")

## Execução — modelo: Claude Sonnet 4.6 (Claude Code, 2026-08-27)

**Parâmetros usados:**

```
estado_relay:
  throughput sustentado: 180k msgs/s
  pico observado: 320k msgs/s por 25min (incidente da semana passada)
  retenção atual: 4h
  consumidores: Forge (ingestão), Sentinel (alerting em tempo real)

restricoes_time:
  alerting do Sentinel não pode atrasar mais que 60s (SLA com cliente)
  ingestão do Forge pode atrasar até 15min sem violar SLA
  orçamento de infra do trimestre já está 8% acima do previsto
  perda de telemetry é inaceitável (produto de observabilidade)
```

**Output:**

---

### Estratégia A — Fila com prioridade (consumidores priorizados)

**O que resolve**: Garante que o Sentinel consuma eventos antes do Forge quando o barramento está sob pressão. O Relay processa a fila do Sentinel com prioridade máxima; o Forge fica em fila secundária e atrasa quando a carga excede a capacidade.

**Prós**:
- Garante o SLA de alerting do Sentinel (60s) mesmo durante picos
- O Forge pode atrasar até 15min sem violação de SLA — a estratégia explora exatamente essa folga
- Mudança de configuração: sem nova infraestrutura, sem custo adicional
- Sem perda de mensagens: o Forge apenas atrasa, não perde

**Contras**:
- Se a sobrecarga durar mais de 15min (o incidente foi 25min), o Forge também viola SLA
- Não isola tenants: um tenant barulhento ainda afeta a fila geral do Forge
- Não resolve o problema estrutural de crescimento de volume

**Custo/complexidade**: Baixo — configuração de Kafka consumer groups com prioridade. Sem custo adicional de infra.

**Compatibilidade com SLAs**: ✅ Sentinel (60s garantido); ⚠️ Forge (15min garantido para picos de até 15min, violado em picos mais longos)

**Compatibilidade com perda**: ✅ Zero perda — Forge atrasa, não perde

---

### Estratégia B — Dead-letter queue (DLQ)

**O que resolve**: Quando o buffer do Relay satura, mensagens não processadas vão para uma DLQ separada em vez de serem perdidas. Um processo de replay as reprocessa após o pico.

**Prós**:
- Zero perda de mensagens — é a garantia mais forte para um produto de observabilidade
- Desacopla o problema de "processar agora" do problema de "não perder"
- Permite replay ordenado após recuperação

**Contras**:
- A DLQ pode crescer indefinidamente durante overloads prolongados
- O replay adiciona latência imprevisível (não atende SLA de 60s para as mensagens na DLQ)
- Precisa de mecanismo de replay + monitoring da DLQ — complexidade operacional adicional
- Não resolve o problema de throughput — sem priority queue, o Sentinel pode atrasar igualmente

**Custo/complexidade**: Médio — requer tópico/fila de DLQ, processo de replay e alertas específicos.

**Compatibilidade com SLAs**: ❌ Não garante 60s para Sentinel (mensagens podem ir para DLQ); ⚠️ Complementar à Estratégia A

**Compatibilidade com perda**: ✅ Zero perda (principal valor desta estratégia)

---

### Estratégia C — Particionamento por tenant

**O que resolve**: Isola tenants em partições dedicadas, impedindo que um tenant com pico de volume (ex.: stark-industries em onboarding) afete os demais.

**Prós**:
- Isolamento completo entre tenants — mais robusto estruturalmente
- Comportamento previsível por tenant
- Base para billing granular por tenant no futuro

**Contras**:
- Rearchitecting significativo: N partições por N tenants, schema de roteamento, migration de dados existentes
- Alta complexidade operacional: monitorar N partições em vez de 2-3 filas
- Custo adicional de infra (mais partições = mais recursos no Kafka/broker) — conflita com orçamento +8%
- Semanas a meses de implementação — não resolve o problema imediato
- Não garante o SLA de 60s por si só (depende da capacidade de cada partição)

**Custo/complexidade**: Alto — mudança arquitetural de meses.

**Compatibilidade com SLAs**: ⚠️ Não garante 60s sem combinação com prioridade por dentro de cada partição

**Compatibilidade com perda**: ✅ Pode garantir zero perda com DLQ por partição — mas aumenta a complexidade

---

### Estratégia D — Autoscaling de consumidores (HPA)

**O que resolve**: Escala o número de consumidores horizontalmente quando a carga sobe, aumentando a capacidade de processamento automaticamente.

**Prós**:
- Elástico para picos — aumenta a capacidade quando precisa
- Pode cobrir tanto Sentinel quanto Forge
- Padrão bem estabelecido no Kubernetes (HPA com métricas customizadas)

**Contras**:
- Latência de scale-up: HPA leva 1-3 min para detectar e escalar, pods levam 30-60s para iniciar. O SLA de 60s DO Sentinel pode ser violado durante o scale-up.
- O pico de 320k msgs/s durou 25min — o autoscaler pode não reagir a tempo para o SLA de 60s
- Custo adicional de infra diretamente (mais pods = mais compute) — conflita com orçamento +8%
- Autoscaling de Kafka consumers com múltiplas partições exige cuidado com rebalancing

**Custo/complexidade**: Médio-alto — HPA com custom metrics (Kafka consumer lag) e gestão de rebalancing.

**Compatibilidade com SLAs**: ❌ Não garante 60s durante o período de scale-up; ✅ Ajuda para picos sustentados após scale-up

**Compatibilidade com perda**: ⚠️ Não garante zero perda durante o scale-up sem DLQ complementar

---

### Decisão recomendada

**Recomendação: Estratégia A (prioridade) + Estratégia B (DLQ) — implementação imediata**

**Justificativa**:
- A combinação A+B atende todos os SLAs: prioridade garante o Sentinel em 60s; DLQ garante zero perda (o requisito mais crítico citado pelo Steve Rogers e impossível de relaxar para um produto de observabilidade)
- Custo incremental mínimo — sem nova infraestrutura computacional, respeitando o orçamento +8%
- Implementação em dias/semanas, não meses
- O pico de 25min viola o SLA do Forge com apenas A, mas com DLQ o dado não se perde e é reprocessado após o pico

**O que foi descartado e por quê**:
- **C (particionamento)**: rearchitecting de meses + custo adicional incompatível com orçamento. Válido para o roadmap do próximo trimestre quando orçamento normalizar.
- **D (autoscaling)**: a latência de scale-up (1-3 min) viola o SLA de 60s do Sentinel durante o período de escala. Pode ser complementar após A+B estabilizarem.

**Riscos residuais**:
- Com A+B, picos sustentados acima de 15min ainda violam o SLA do Forge (a DLQ acumula, mas o replay pode levar horas)
- A DLQ precisa de monitoramento próprio: se cresce indefinidamente, vira um problema secundário

**Próximos passos** (em ordem):
1. Configurar priority consumer groups no Relay (dias)
2. Criar tópico DLQ e processo de replay com alerta de volume (semana)
3. Testar com carga simulada de 320k msgs/s antes de promover para produção
4. Backlog: revisitar particionamento por tenant no próximo trimestre

---

## Curadoria

**Meta-prompting**: Diretriz usada: _"Crie um prompt de decisão arquitetural que force a avaliação de múltiplas estratégias contra restrições específicas antes de recomendar. O valor está na comparação, não na resposta única. As restrições (SLAs, orçamento, zero-loss) devem ser explicitamente checadas para cada estratégia."_

**Refinamentos:**
1. A v1 retornava apenas a estratégia recomendada sem avaliar as demais. A instrução "Para CADA estratégia, avalie explicitamente" foi adicionada para forçar a análise completa antes da recomendação.
2. A dimensão "zero perda de mensagens" não estava explícita como critério de avaliação na v1. Adicionada após perceber que o requisito de zero-loss é o mais diferenciador para um produto de observabilidade.
3. "Próximos passos" foi adicionado como seção final para tornar a decisão acionável — sem ela, a análise ficava como exercício teórico.

## Limitações

- O prompt avalia 4 estratégias específicas. Para barbeadores ou sistemas que não sejam Kafka-like (ex.: RabbitMQ, Kinesis), os nomes e mecanismos precisam ser adaptados.
- Saída aberta (não estruturada): não testável por regex — requer LLM-judge ou revisão humana para validação de qualidade.
- A análise de custo é qualitativa (baixo/médio/alto), não quantitativa. Para decisões que exigem número de dólares, complementar com estimativa de cost engineering.
