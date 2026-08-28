---
nome: Migração Forge — Plano de Migração Faseado (Passo 2/3)
descricao: "Cadeia de migração batch→event-driven: gera um plano faseado e reversível com base no diagnóstico do estado atual (saída do Passo 1)"
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [forge, migração, batch, event-driven, plano]
inputs:
  - nome: estado_atual_forge
    descricao: Descrição do estado atual do Forge (mesma entrada do Passo 1, repetida para contexto)
  - nome: diagnostico_forge
    descricao: Saída do Passo 1/3 — diagnóstico completo com riscos, gaps e síntese
---

Você é um arquiteto de dados especializado em migrações incrementais de sistemas críticos. Com base no diagnóstico fornecido, elabore um plano de migração faseado, reversível e seguro do Forge de batch para event-driven.

## Estado atual do sistema

```
{{estado_atual_forge}}
```

## Diagnóstico do estado atual (saída do Passo 1)

```
{{diagnostico_forge}}
```

## Restrições obrigatórias do plano

- **Sem big-bang**: nenhuma fase corta o sistema atual antes do novo estar validado em produção
- **Fases independentemente reversíveis**: cada fase pode ser revertida sem comprometer a fase anterior
- **Proteção dos consumidores**: Sentinel, Cerebro e billing não podem ser interrompidos em nenhuma fase
- **Critérios mensuráveis**: cada fase tem um critério de avanço verificável, não subjetivo

## Plano de migração solicitado

Para cada fase do plano, documente:

**Título e objetivo**: o que esta fase entrega em termos de valor e redução de risco

**Duração estimada**: janela realista para execução e validação

**Ações principais**: o que precisa acontecer (visão de alto nível — o Passo 3 detalha)

**Critério de avanço**: como saber objetivamente que está pronto para a próxima fase (métrica ou verificação mensurável)

**Critério de rollback**: qual condição dispara o rollback e qual é o procedimento de volta ao estado anterior

**Risco residual**: o que ainda pode dar errado nesta fase específica

**Nota de proteção dos consumidores**: como Sentinel, Cerebro e billing são protegidos durante esta fase
