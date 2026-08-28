---
nome: Nota de Triagem de Alerta
descricao: Transforma alertas crus do Sentinel em notas de triagem padronizadas com cinco campos obrigatórios, seguindo o padrão do time
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [alerta, triagem, nota, sre, padronização]
inputs:
  - nome: alerta_cru
    descricao: Texto bruto do alerta gerado pelo Sentinel, contendo timestamp, sistema afetado e sintomas observados
---

# Nota de Triagem de Alerta — CP02

## Objetivo

Recebe um alerta cru do Sentinel (texto livre, geralmente uma linha com timestamp, sistema e sintomas) e devolve uma nota de triagem padronizada com os cinco campos obrigatórios: ALERTA, IMPACTO, HIPÓTESE INICIAL, AÇÃO IMEDIATA e ESCALAR PARA. O formato é uniforme independentemente de quem está de plantão.

## Quando usar

- Ao abrir um incidente no Sentinel: cole o alerta cru e gere a nota estruturada
- Quando assumir um turno e precisar normalizar alertas do turno anterior
- Para padronizar alertas vindos de sistemas legados antes de registrá-los no histórico

## Técnica utilizada

**Few-shot learning (3-shot).**

Os três exemplos de notas boas do time estão embutidos no prompt como demonstrações de formato. O modelo generaliza o padrão a partir dos exemplos, sem precisar de instruções detalhadas sobre cada campo — os exemplos mostram melhor do que qualquer descrição.

### Por que few-shot aqui?

O desafio é ensinar um formato rígido de 5 campos com restrições de concisão (≤ 8 linhas) e um campo ESCALAR PARA com padrão específico (@handle + condição + prazo). Few-shot é a técnica mais eficiente para isso: os exemplos demonstram a estrutura, o tom e o nível de detalhe esperado sem ambiguidade. A alternativa (descrever cada campo em prosa) introduz mais margem para interpretação inconsistente.

### Decisão de método

O time considerou duas abordagens:
- **Few-shot** (escolhida): os 3 exemplos de boas notas já existem — basta incluí-los no prompt como demonstrações.
- **Prompt estruturado com descrição de cada campo**: mais verboso, mais sujeito a variação de interpretação.

Few-shot foi escolhida por ser mais direta e produzir formato mais consistente entre chamadas.

## Dados sensíveis a sanitizar

- Nomes de tenants (ex.: `stark-industries`, `wakanda-systems`) → substituir por `[TENANT]` antes de enviar a modelo externo
- Nomes de sistemas internos: manter (são relevantes para o contexto de triagem)

## Execução — modelo: Claude Sonnet 4.6 (Claude Code, 2026-08-27)

### Entrada 1 — autoscaler atingiu limite máximo de réplicas

**Alerta cru:**
```
2026-05-12 14:02:09 UTC [Sentinel] autoscaler hit max replicas (60/60) on sentinel-api,
queue depth on Relay growing 2k/min, CPU avg 88%, tenant stark-industries
sending 4x baseline volume after onboarding new region
```

**Output:**
```
ALERTA: Sentinel - autoscaler atingiu limite máximo de réplicas (60/60) em sentinel-api
IMPACTO: capacidade de processamento esgotada; fila no Relay crescendo 2k msgs/min; tenant [TENANT] gerando 4x o volume baseline após onboarding de nova região
HIPÓTESE INICIAL: onboarding de nova região do tenant disparou pico de volume que saturou o pool máximo de réplicas do sentinel-api
AÇÃO IMEDIATA: elevar provisoriamente o limite máximo de réplicas do HPA; avaliar throttling por tenant
ESCALAR PARA: @sentinel-core se fila não estabilizar após ajuste em 15min
```

### Entrada 2 — taxa de rejeição do Relay em 6% por 8 minutos

**Alerta cru:**
```
2026-05-13 03:11:00 UTC [Relay] ingest reject rate 6% for 8min, tenant wakanda-systems,
buffer saturated after deploy 02:55
```

**Output:**
```
ALERTA: Relay - taxa de rejeição de ingestão em 6% por 8min
IMPACTO: telemetry do tenant [TENANT] sendo rejeitado; dados perdidos ou atrasados
HIPÓTESE INICIAL: deploy executado às 02:55 saturou o buffer de ingestão do Relay; rejeições iniciaram às 03:11
AÇÃO IMEDIATA: rollback do deploy das 02:55 no Relay; monitorar taxa de rejeição após rollback
ESCALAR PARA: @relay-core se rejeição não cair abaixo de 2% em 10min após rollback
```

### Entrada 3 — lag crescente no consumer do Forge

**Alerta cru:**
```
2026-05-13 11:40:22 UTC [Forge] consumer lag 9min and climbing, batch forge-batch-ingest
delayed after previous job failure, downstream Sentinel starting to lag
```

**Output:**
```
ALERTA: Forge - lag do consumer em 9min e crescendo; batch forge-batch-ingest atrasado
IMPACTO: dados no Sentinel atrasando; relatórios downstream potencialmente afetados
HIPÓTESE INICIAL: falha no job anterior do Forge criou acúmulo de volume não processado, sobrecarregando o consumer atual
AÇÃO IMEDIATA: verificar e reiniciar job forge-batch-ingest falho; monitorar lag do consumer
ESCALAR PARA: @data-platform se lag não estabilizar abaixo de 5min em 20min
```

## Curadoria

**Modelo**: Claude Sonnet 4.6 (Anthropic) para execução manual; `google:gemini-2.5-flash-lite` (Google) nos testes automatizados.

**Justificativa de modelo**: Para notas de triagem (saída de formato fixo, baixa complexidade de raciocínio), um modelo rápido e barato é suficiente. O `gemini-2.5-flash-lite` responde em ~2s com tokens mínimos (~300 por chamada), mantendo custo por chamada abaixo de $0,001. Claude Sonnet 4.6 foi usado na execução manual por já estar disponível no ambiente de desenvolvimento — não seria o modelo de produção para esta tarefa.

**Meta-prompting**: O prompt foi gerado com a diretriz: _"Crie um prompt parametrizável que transforma alertas crus em notas padronizadas com 5 campos fixos. O time já tem 3 exemplos de notas boas — use-os como few-shot. O campo ESCALAR PARA deve sempre ter um handle @equipe e uma condição com prazo. A nota deve ter no máximo 8 linhas."_

**Refinamentos:**
1. A v1 incluía os exemplos após a instrução principal, mas o modelo frequentemente "misturava" o exemplo com o alerta a processar. A correção foi adicionar uma nota explícita: "Use-as como referência de formato — não são o alerta que você vai processar."
2. O campo HIPÓTESE INICIAL na v1 às vezes saía como certeza ("A causa é o deploy"). Adicionado: "é hipótese, não certeza."
3. O limite de 8 linhas era violado em alertas com muito contexto. O texto "Máximo 8 linhas no total" foi reforçado para "Máximo 8 linhas no total" na seção de regras.

## Limitações

- Com alertas muito sintéticos (menos de 10 palavras), a HIPÓTESE INICIAL pode ser genérica. Incluir contexto suficiente no alerta cru melhora a qualidade.
- O campo AÇÃO IMEDIATA assume que o plantonista tem acesso às ferramentas mencionadas. Para equipes sem acesso direto ao Argo CD ou kubectl, a ação gerada pode precisar de ajuste.
- O few-shot usa exemplos dos sistemas da Aegis (Relay, Forge, Sentinel, Cerebro). Para outros contextos, os exemplos devem ser trocados por demonstrações do novo domínio.

## Testes

```bash
export GOOGLE_API_KEY="sua-chave"
promptfoo eval --config ./promptfooconfig.yaml
```

**Resultado real (2026-08-28, `google:gemini-2.5-flash-lite`):**

```
✓ 3 passed (100%) — 0 failed — 0 errors
Duration: ~7s (concurrency: 4)
```

| Entrada | Asserts verificados | Resultado |
|---------|---------------------|-----------|
| 1 — autoscaler Sentinel | 5 labels + @handle + ≤8 linhas + Sentinel + Relay | ✅ PASS |
| 2 — rejeição Relay 6% | 5 labels + @handle + ≤8 linhas + Relay + deploy | ✅ PASS |
| 3 — lag Forge 9min | 5 labels + @handle + ≤8 linhas + Forge + job/batch | ✅ PASS |

**O que foi ajustado durante os testes:**
- `(?i)` (flag PCRE) é inválido em JavaScript — promptfoo usa `new RegExp(value)`. Todos os regex case-insensitive convertidos para `javascript` asserts com `/pattern/i.test(output)`.
- Assertion `cost` removida: provedores Google não reportam custo por chamada no promptfoo.
- `gemini-3.6-flash` substituído por `gemini-2.5-flash-lite` após 503 intermitentes de alta demanda.
- `gemini-2.5-flash` removido: modo thinking gera respostas em ~53s, excedendo o threshold de 15s.
- Regra anti-markdown adicionada ao prompt: Gemini envolvia a saída em blocos de código (`\`\`\``), quebrando o assert `contains "ALERTA:"`.

**Provedores ao longo do desafio (CP02):**
- **Execução manual (CP02)**: Claude Sonnet 4.6 via Claude Code (Anthropic)
- **Testes automatizados (CP08)**: `google:gemini-2.5-flash-lite` (Google)
- Dois fornecedores distintos: Anthropic + Google ✅

Ver [promptfooconfig.yaml](./promptfooconfig.yaml).
