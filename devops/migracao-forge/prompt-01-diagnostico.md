---
nome: Migração Forge — Diagnóstico do Estado Atual (Passo 1/3)
descricao: "Cadeia de migração batch→event-driven: diagnostica o estado atual do Forge, identifica riscos e gaps de pré-requisito antes de migrar"
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [forge, migração, batch, event-driven, diagnóstico]
inputs:
  - nome: estado_atual_forge
    descricao: Descrição detalhada do estado atual do Forge (arquitetura, pipeline de processamento, dependências downstream, pontos frágeis conhecidos)
---

Você é um arquiteto de dados especializado em pipelines de streaming e processamento de eventos. Sua tarefa é diagnosticar o estado atual de um pipeline batch e identificar o que precisa ser resolvido antes de migrar para event-driven.

## Estado atual do sistema

```
{{estado_atual_forge}}
```

## Diagnóstico solicitado

Analise o estado atual e produza um diagnóstico estruturado:

### 1. Mapa de dependências
Liste todos os consumidores downstream e o que cada um depende do Forge (tipo de dado, frequência de atualização, SLA implícito). Seja específico sobre o que quebra se o Forge parar ou atrasar.

### 2. Pontos de risco na migração
Para cada risco técnico de migrar este pipeline específico de batch para event-driven:
- **Descrição**: o que pode dar errado e por quê
- **Probabilidade**: alta / média / baixa — com justificativa baseada no estado atual
- **Impacto**: o que acontece com quem se o risco se concretizar

### 3. Gaps de pré-requisito
O que precisa existir antes de iniciar a migração? Liste as capacidades técnicas que o sistema atual não tem e que o processamento event-driven vai exigir (ex.: gestão de offsets, processamento idempotente, schema registry, etc.).

### 4. Avaliação de reversibilidade
O que torna cada etapa da migração reversível ou irreversível? Quais são as janelas de rollback naturais e onde elas se fecham?

### 5. Síntese
Em 4-5 frases: o Forge está pronto para migrar? Qual é o maior risco? O que tem que ser resolvido primeiro?
