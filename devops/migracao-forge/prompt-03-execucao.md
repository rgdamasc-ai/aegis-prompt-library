---
nome: Migração Forge — Passos Executáveis por Fase (Passo 3/3)
descricao: "Cadeia de migração batch→event-driven: detalha os passos executáveis e reversíveis para uma fase específica do plano (saída do Passo 2)"
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [forge, migração, batch, event-driven, execução]
inputs:
  - nome: plano_migracao
    descricao: Saída do Passo 2/3 — plano de migração faseado completo com fases, critérios e riscos
  - nome: fase_alvo
    descricao: Número e nome da fase para detalhar (ex. "Fase 1 — Consumer em modo shadow")
---

Você é um engenheiro de plataforma especializado em execução de migrações críticas. Com base no plano de migração e na fase especificada, detalhe os passos executáveis com validações e procedimento de rollback.

## Plano de migração (todas as fases para contexto)

```
{{plano_migracao}}
```

## Fase a detalhar

```
{{fase_alvo}}
```

## Detalhamento solicitado

### Pré-requisitos
O que precisa estar em vigor antes de iniciar esta fase: acessos, configurações, testes de sanidade prévios.

### Passos de execução
Lista numerada de passos concretos. Para cada passo:
- **Ação**: o que fazer (específico o suficiente para um engenheiro executar)
- **Validação**: como confirmar que o passo foi concluído com sucesso
- **Tempo estimado**: quanto tempo esperar para o passo se estabilizar

### Checklist de avanço
Checklist binário (sim/não) para verificar que a fase foi concluída com sucesso antes de avançar.
Cada item deve ser verificável objetivamente — sem "parece OK".

### Procedimento de rollback
Lista numerada de passos para reverter esta fase, na ordem inversa lógica. Inclua:
- Como verificar que o rollback foi bem-sucedido
- Quanto tempo o rollback deve levar
- Quem precisa ser notificado

### Monitoramento durante a fase
- Métricas-chave a observar (nomes específicos)
- Valores esperados vs. valores de alerta
- Tempo de estabilização esperado após cada passo
