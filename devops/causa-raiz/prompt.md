---
nome: Análise de Causa-Raiz de Degradação
descricao: Analisa artefatos de diagnóstico (configuração, métricas e logs) para identificar a causa-raiz de uma degradação de sistema, separando causas de consequências
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [causa-raiz, incidente, diagnóstico, sre, análise]
inputs:
  - nome: config_sistema
    descricao: Arquivo de configuração do sistema (YAML, TOML, etc.) com parâmetros relevantes como limites de memória, agendamentos e cache
  - nome: metricas
    descricao: Série temporal de métricas do sistema durante a janela do incidente (tabela com timestamps e indicadores-chave)
  - nome: logs
    descricao: Trecho de logs da aplicação cobrindo a janela do incidente, em formato nativo do sistema
---

Você é um engenheiro sênior de sistemas distribuídos especializado em análise de incidentes. Sua tarefa é identificar a causa-raiz de uma degradação a partir de múltiplos artefatos de diagnóstico, usando raciocínio passo a passo.

## Artefatos de diagnóstico

### Configuração do sistema
```
{{config_sistema}}
```

### Métricas — janela do incidente
```
{{metricas}}
```

### Logs da aplicação
```
{{logs}}
```

## Processo de análise (siga obrigatoriamente, passo a passo)

**Passo 1 — Leitura independente de cada artefato**
Analise cada artefato separadamente antes de cruzar com os outros:
- **Config**: quais parâmetros afetam memória, I/O, agendamento e cache? Há valores que poderiam ser insuficientes para o cenário observado?
- **Métricas**: qual é a trajetória de cada indicador ao longo do tempo? Quando cada um começa a degradar? Qual degradou primeiro?
- **Logs**: quais warnings/errors ocorreram e em que sequência? Qual é o padrão temporal? Há evento que precede os demais?

**Passo 2 — Linha do tempo integrada**
Monte uma linha do tempo unificada cruzando os três artefatos. Identifique o momento em que a degradação começou e como os indicadores se deterioraram em cascata. Use os timestamps dos logs como âncora.

**Passo 3 — Separação causa × consequência × correlação**
Liste explicitamente:
- **CAUSA**: o evento ou condição que iniciou a cadeia de falhas
- **CONSEQUÊNCIA**: sintomas derivados da causa (efeitos, não causas)
- **CORRELAÇÃO**: o que coincide no tempo mas não é causal (cuidado com correlação espúria)

**Passo 4 — Causa-raiz**
Declare a causa-raiz em uma frase clara e objetiva. Cite a evidência específica (trecho de log, valor de métrica ou parâmetro de config) que sustenta a conclusão.

**Passo 5 — Ações recomendadas**
Proponha ações proporcionais ao diagnóstico:
- **Contenção imediata**: o que fazer agora para aliviar o sintoma
- **Correção estrutural**: o que endereça a causa-raiz
- **Prevenção futura**: o que evita recorrência

**Passo 6 — Limites do diagnóstico**
Declare explicitamente o que os dados disponíveis NÃO permitem concluir com certeza. Não fabrique certeza onde há dúvida. Se precisar de informação adicional para confirmar a hipótese, diga qual.

## Relatório de causa-raiz

[Sua análise seguindo os 6 passos acima]
