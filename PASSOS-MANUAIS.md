# Passos Manuais — Execução e Configuração

Este guia cobre o que não pode ser automatizado no pipeline de CI: configuração de API keys, execução local do promptfoo, adição de novos provedores e upload para o GitHub.

---

## Estado atual dos testes (2026-08-28)

### Execução real — ✅ Todos passando (100%)

Todos os 4 configs foram executados com sucesso contra a API real (`GOOGLE_API_KEY` válida, `google:gemini-2.5-flash-lite`):

```
✅ devops/nota-de-triagem/promptfooconfig.yaml:        3/3 passed (100%) — duração: ~7s
✅ devops/triagem-de-pods/promptfooconfig.yaml:         3/3 passed (100%) — duração: ~14s
✅ devops/networkpolicy-sentinel/promptfooconfig.yaml:  1/1 passed (100%) — duração: ~7s
✅ devops/causa-raiz/promptfooconfig.yaml:              1/1 passed (100%) — duração: ~13s (inclui llm-rubric)
```

### Provider atual: `google:gemini-2.5-flash-lite`

O provider final configurado em todos os configs é `google:gemini-2.5-flash-lite`. Motivo da escolha:
- `gemini-2.0-flash-001` — **deprecado** (retorna 404, redireciona para gemini-3.6-flash)
- `gemini-3.6-flash` — **intermitente** (503 high demand com frequência)
- `gemini-2.5-flash` — **lento demais** para modo thinking (~53s, excede threshold de 15s)
- `gemini-2.5-flash-lite` — **disponível e rápido** (7-14s por eval, sem thinking forçado)

### Problemas corrigidos durante os testes

| Problema | Causa | Correção |
|----------|-------|----------|
| `(?i)` inválido em regex | promptfoo usa `new RegExp(value)` JS — sem suporte a PCRE `(?i)` | Convertido para `javascript` asserts com `/regex/i.test(output)` |
| `cost` assertion falhava | Gemini não reporta custo por chamada no promptfoo | Removido de todos os configs |
| gemini-2.5-flash timeout | Thinking mode (~53s) excede threshold de 15s | Removido dos configs determinísticos, mantido só no llm-rubric |
| gemini-3.6-flash 503 | Alta demanda no endpoint | Substituído por gemini-2.5-flash-lite |

### Rodar os testes

```bash
export GOOGLE_API_KEY="AIzaSy..."
cd módulo-3-desafios/prompt-library/

promptfoo eval --config devops/nota-de-triagem/promptfooconfig.yaml
promptfoo eval --config devops/triagem-de-pods/promptfooconfig.yaml
promptfoo eval --config devops/networkpolicy-sentinel/promptfooconfig.yaml
promptfoo eval --config devops/causa-raiz/promptfooconfig.yaml
```

### Outputs documentados nos README de cada prompt

- [triagem-de-pods/README.md](./devops/triagem-de-pods/README.md) — Entradas 1, 2 e 3
- [nota-de-triagem/README.md](./devops/nota-de-triagem/README.md) — Entradas 1, 2 e 3
- [causa-raiz/README.md](./devops/causa-raiz/README.md) — Análise completa do Cerebro
- [networkpolicy-sentinel/README.md](./devops/networkpolicy-sentinel/README.md) — v1, verificação e v2

---

## 1. Configurar API Keys

### Google AI Studio (Gemini) — já configurado localmente

```bash
export GOOGLE_API_KEY="sua-chave-aqui"
```

Para persistir:
```bash
echo 'export GOOGLE_API_KEY="sua-chave-aqui"' >> ~/.bashrc
source ~/.bashrc
```

### Adicionar Anthropic (Claude)

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

E atualizar os `promptfooconfig.yaml` para adicionar o provider:
```yaml
providers:
  - id: google:gemini-2.5-flash-lite
    config:
      temperature: 0.0
  - id: anthropic:claude-haiku-4-5-20251001
    config:
      temperature: 0.0
```

### Adicionar OpenAI (GPT)

```bash
export OPENAI_API_KEY="sk-..."
```

Provider no config:
```yaml
providers:
  - id: openai:gpt-4o-mini
    config:
      temperature: 0.0
```

### Usar modelos locais via Ollama

```bash
ollama pull llama3.2
```

Provider no config:
```yaml
providers:
  - id: ollama:llama3.2
    config:
      temperature: 0.0
```

---

## 2. Executar promptfoo localmente

### Pré-requisitos

```bash
# Verificar instalação
promptfoo --version

# Se não instalado:
npm install -g promptfoo@latest
```

### Rodar um config específico

```bash
# Da pasta prompt-library/
cd módulo-3-desafios/prompt-library/

promptfoo eval --config devops/triagem-de-pods/promptfooconfig.yaml
promptfoo eval --config devops/nota-de-triagem/promptfooconfig.yaml
promptfoo eval --config devops/causa-raiz/promptfooconfig.yaml
promptfoo eval --config devops/networkpolicy-sentinel/promptfooconfig.yaml
```

### Ver resultados no browser

```bash
promptfoo eval --config devops/nota-de-triagem/promptfooconfig.yaml
promptfoo view
```

Abre `http://localhost:15500` com os resultados.

### Rodar com saída JSON (para CI/CD manual)

```bash
promptfoo eval \
  --config devops/nota-de-triagem/promptfooconfig.yaml \
  --output results/nota-de-triagem.json \
  --no-cache
```

---

## 3. Executar os prompts CP04 e CP05 (sem promptfoo)

Os prompts de **decisão** (CP04 - backpressure) e **cadeia** (CP05 - migração) têm saída aberta e não são testáveis por promptfoo de forma determinística. Execute manualmente:

### CP04 — Backpressure Relay

1. Abrir o arquivo `devops/backpressure-relay/prompt.md`
2. Copiar o texto do prompt (abaixo do frontmatter)
3. Substituir `{{estado_relay}}` e `{{restricoes_time}}` pelos valores reais
4. Colar no modelo escolhido (Claude, Gemini, GPT-4o, etc.)

Exemplo de valores:
```
{{estado_relay}}:
  throughput sustentado: 180k msgs/s
  pico observado: 320k msgs/s por 25min
  retenção atual: 4h
  consumidores: Forge (ingestão), Sentinel (alerting em tempo real)

{{restricoes_time}}:
  alerting do Sentinel: SLA de 60s máximo de atraso
  ingestão do Forge: pode atrasar até 15min
  orçamento: já 8% acima do previsto no trimestre
  perda de telemetry: inaceitável (produto de observabilidade)
```

### CP05 — Migração Forge (cadeia de 3)

Rodar em sequência, passando a saída de cada passo como entrada do próximo:

**Passo 1:**
```
prompt: devops/migracao-forge/prompt-01-diagnostico.md
{{estado_atual_forge}}: [descrição do Forge atual]
→ guardar saída como DIAGNOSTICO
```

**Passo 2:**
```
prompt: devops/migracao-forge/prompt-02-plano.md
{{estado_atual_forge}}: [mesma entrada do Passo 1]
{{diagnostico_forge}}: [saída do Passo 1 = DIAGNOSTICO]
→ guardar saída como PLANO
```

**Passo 3:**
```
prompt: devops/migracao-forge/prompt-03-execucao.md
{{plano_migracao}}: [saída do Passo 2 = PLANO]
{{fase_alvo}}: "Fase 1 — Consumer em modo shadow"
→ resultado: passos executáveis com validações e rollback
```

---

## 4. Configurar GitHub Actions (CP10)

### Configurar secrets no repositório

No GitHub, acessar **Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Valor |
|--------|-------|
| `GOOGLE_API_KEY` | Chave do Google AI Studio |
| `ANTHROPIC_API_KEY` | Chave da Anthropic (se usar Claude) |
| `OPENAI_API_KEY` | Chave da OpenAI (se usar GPT) |

### Criar o repositório público no GitHub

```bash
cd /home/damaceno/AIOps-GenAI-Cotidiano-Tecnico/módulo-3-desafios/prompt-library/

# Inicializar git (se ainda não é um repositório)
git init
git add .
git commit -m "feat(devops): inicializa playbook de IA operacional da Aegis com CP01-CP10"

# Criar repositório no GitHub e fazer push
gh repo create aegis-prompt-library --public --source=. --remote=origin --push
```

Ou via interface:
1. Criar repositório em github.com com nome `aegis-prompt-library`
2. `git remote add origin https://github.com/SEU-USUARIO/aegis-prompt-library.git`
3. `git push -u origin main`

### Verificar que o workflow está funcionando

Após o push, acessar:
- **Actions** no repositório → ver o workflow `Promptfoo Eval`
- Na primeira execução, pode ser necessário habilitar Actions nas Settings

---

## 5. Decisões de design do pipeline (CP10) — Justificativa estendida

Esta seção compara explicitamente pelo menos duas alternativas para cada decisão de design do pipeline. O objetivo é documentar o raciocínio — não apenas a escolha final.

---

### Decisão 1 — Quando acionar o pipeline (trigger)

**Alternativa A (escolhida): trigger em PR + push para main, filtrado por paths**

- Custo: só executa quando arquivos de prompt são modificados.
- Feedback: bloqueia merge antes de chegar a main.
- Fallback: push direto para main roda tudo (sem filtro de paths).
- Contras: configuração adicional com `dorny/paths-filter`; mudanças transversais (ex.: nova variável de ambiente) não disparam testes de prompts não alterados.

**Alternativa B: trigger em todo push, qualquer branch**

- Custo: executa para cada commit em cada branch, inclusive branches de documentação ou README — ~$0,05 por execução desnecessária.
- Feedback: detecta regressões antes do PR abrir.
- Contras: gasto elevado em repositórios ativos; demorar o feedback de um commit simples em 2 minutos de CI desincentiva commits frequentes.

**Alternativa C: schedule diário (cron)**

- Custo: execução fixa 1×/dia, independente de volume de commits.
- Detecta regressões causadas por mudanças do provider externo (ex.: nova versão do Gemini retorna formato diferente).
- Contras: não bloqueia merge — um prompt regredido pode entrar em main e permanecer até o próximo job diário (até 24h de janela cega).

**Decisão**: Alternativa A. O paths-filter elimina execuções desnecessárias sem perder cobertura nos merges para main. O schedule diário seria complementar (não substituto), mas está fora do escopo deste MVP.

---

### Decisão 2 — Escopo: quais prompts rodar por execução

**Alternativa A (escolhida): paths-filter — rodar só os prompts alterados no PR**

- Um PR que toca só `nota-de-triagem/` gasta ~$0,002 (3 asserts, ~1k tokens cada) em vez de ~$0,05 (suíte completa).
- Fallback: push para main sempre roda todos — regressão transversal é detectada no merge.
- Contras: se uma mudança afeta múltiplos prompts indiretamente (ex.: alteração em variável de ambiente compartilhada), só os prompts explicitamente alterados são testados.

**Alternativa B: rodar sempre todos os prompts, em todo PR**

- Cobertura total: nenhuma regressão transversal escapa, independente de quais arquivos foram modificados.
- Custo: ~$0,05/PR fixo, independente do tamanho da mudança. Em repositórios com 10+ PRs/dia, isso é ~$0,50/dia só em CI de testes de prompt.
- Contras: torna CI lento e caro para PRs de documentação ou refatoração sem mudança de comportamento de prompt.

**Alternativa C: seleção manual via `workflow_dispatch` input**

- Engenheiro seleciona quais prompts testar antes de abrir o PR.
- Controle total sobre custo por execução.
- Contras: sem garantia de cobertura — engenheiro pode esquecer de selecionar um prompt afetado. Remove a "rede de segurança" que o CI deve prover.

**Decisão**: Alternativa A. O fallback de push para main garante que a suíte completa rode pelo menos uma vez antes de qualquer mudança chegar a produção.

---

### Decisão 3 — Gate: o que falha o build

**Alternativa A (escolhida): falha total — qualquer assert falhado OU score do LLM-judge < 6 bloqueia o merge**

- Garantia: nenhum prompt regredido entra em main.
- Risco: o juiz LLM não é determinístico — pode reprovar um build válido (falso positivo). Mitigação: threshold conservador (6/8 = 75%) + temperature=0 + 4 critérios independentes (improvável falhar por flutuação aleatória).
- Contras: bloqueante — um false positive do juiz exige rerun manual.

**Alternativa B: apenas warn — CI loga resultado mas nunca falha o build**

- Sem risco de falso positivo do juiz; PR nunca bloqueado por causa do CI de prompts.
- Requer disciplina manual: engenheiro precisa verificar os logs de CI antes de fazer merge.
- Contras: regressões silenciosas chegam a main. Em incidente real, um prompt que "passou no CI" pode não produzir a análise esperada.

**Alternativa C: falha somente em asserts determinísticos; LLM-judge emite relatório mas não bloqueia**

- Melhor equilíbrio no papel: formato/latência são objetivos → bloqueiam. Qualidade subjetiva → avisa mas não bloqueia.
- Problema: regressão de qualidade (prompt regrediu de "causa-raiz correta" para "lista de sintomas") nunca bloqueia merge. Para um playbook de incidente, qualidade é o requisito primário — não um nice-to-have.
- Contras: qualidade de análise é o que diferencia um prompt útil de um inútil; não bloqueá-la derrota o propósito do LLM-judge.

**Decisão**: Alternativa A. A consequência de um prompt de causa-raiz ruim em plantão (análise incorreta levando a ação errada) é maior que a de um PR bloqueado por falso positivo do juiz (rerun em 2 minutos).

---

### Decisão 4 — Modelo do juiz LLM (CP09)

**Alternativa A (escolhida): mesmo provider que o avaliado (`google:gemini-2.5-flash-lite`)**

- Configuração simples: apenas um secret (`GOOGLE_API_KEY`), nenhum segundo provider.
- Custo: sem segundo custo de API — o mesmo budget cobre tanto a avaliação quanto o julgamento.
- Risco: juiz e avaliado são do mesmo fornecedor. Se o Gemini tende a validar sua própria lógica interna ("auto-consistência"), as notas podem ser sistematicamente mais altas do que um avaliador independente daria.
- Contras: em caso de outage do Google, tanto o avaliado quanto o juiz ficam indisponíveis simultaneamente.

**Alternativa B: provider diferente (ex.: GPT-4o-mini como juiz para avaliações Gemini)**

- Elimina o viés de auto-consistência: juiz com lógica independente do avaliado.
- Melhor para produção: a calibração humana × juiz mostrou Δ=0 neste playbook, mas um juiz de provider diferente reduziria esse risco estruturalmente.
- Contras: segundo secret no repositório (`OPENAI_API_KEY`); segundo custo de provider; complexidade de manutenção quando um dos dois providers tem outage — o CI pode falhar por razão de infraestrutura, não de qualidade de prompt.

**Alternativa C: rubrica estática sem LLM — verificação por regex de palavras-chave na saída**

- Determinístico: sem risco de falso positivo por flutuação do juiz.
- Sem custo adicional de API para o gate de qualidade.
- Contras: detecta apenas presença de palavras, não qualidade da análise. Um prompt pode mencionar "causa-raiz" e "reindexação" sem ter feito a correlação correta. O valor do LLM-judge é exatamente avaliar o raciocínio, não apenas a presença de termos.

**Decisão**: Alternativa A para o MVP. Para produção real com alto volume de merges, migrar para Alternativa B (provider diferente) para eliminar o viés estrutural.

---

### Decisão 5 — Threshold de latência

**Alternativa A (escolhida): 15s para prompts simples, 30s para LLM-judge com raciocínio**

- Alinhado com expectativa operacional: um plantonista aguardando triagem de pods em incidente real não espera mais de 15s por resposta.
- Detecta regressões de latência antes que impactem usuário: se uma mudança de prompt faz o modelo gerar resposta 3x mais longa, o threshold pega.
- Contras: picos momentâneos de latência da API do Gemini podem gerar falso positivo. Observado em testes: `gemini-2.5-flash` atingiu ~53s por thinking mode — threshold de 15s teria reprovado.

**Alternativa B: sem threshold de latência**

- Sem risco de falso positivo por pico de API.
- Contras: uma mudança no prompt que triplique o tempo de resposta passa sem alerta. O usuário de plantão só percebe a lentidão em incidente real — quando o contexto de urgência torna qualquer atraso mais custoso.

**Alternativa C: threshold alto e único (60s)**

- Elimina quase todos os falsos positivos por pico de API — 60s é improvável de ser atingido por `gemini-2.5-flash-lite` em condições normais.
- Contras: uma regressão de 3x na latência (ex.: prompt novo induz thinking mode) passa sem alerta até atingir 60s. Para o caso `gemini-2.5-flash` (53s), o threshold de 60s não teria capturado o problema que foi identificado manualmente.

**Decisão**: Alternativa A (15s/30s). O custo de um falso positivo ocasional (rerun do CI) é menor que o custo de não detectar regressão de latência em um prompt operacional.

---

### Decisão 6 — fail-fast na matrix de prompts

**Alternativa A (escolhida): `fail-fast: false` — continua todos os jobs da matrix mesmo se um falha**

- Ver todos os falhos de uma vez é mais útil para diagnóstico. Se 3 de 4 prompts falham simultaneamente, é provável que seja problema de provider ou de variável de ambiente — não de prompt individual.
- Contras: gasta tokens dos prompts que rodam após o primeiro falho. Em um cenário de falha total, pode gastar ~$0,05 desnecessariamente.

**Alternativa B: `fail-fast: true` — cancela a matrix ao primeiro falho**

- Economiza tokens quando há falha óbvia (ex.: GOOGLE_API_KEY inválida → falha no primeiro prompt → cancela os demais).
- Contras: perde contexto sobre quais outros prompts estavam passando. Se 1 falha e 3 passam, o diagnóstico aponta para o prompt específico — não para infraestrutura. Sem ver os outros resultados, não é possível fazer essa distinção.

**Alternativa C: jobs sequenciais sem matrix (um job por prompt, dependência `needs`)**

- Controle total de ordem e dependências: pode, por exemplo, rodar nota-de-triagem só se triagem-de-pods passou.
- Contras: sem paralelismo — tempo total é a soma das durações individuais (~7s + ~14s + ~7s + ~13s = ~41s sequencial vs ~14s paralelo). YAML mais verboso e difícil de manter ao adicionar novos prompts.

**Decisão**: Alternativa A (`fail-fast: false`). Diagnóstico completo em uma execução > economia marginal de tokens em cenário de falha.

---

### O que falha o build — resumo

**Asserts determinísticos** (CP08): qualquer falha bloqueia o merge imediatamente. São objetivos e não sofrem de não-determinismo:
- Formato (campos obrigatórios presentes)
- Latência (< 15s para prompts simples, < 30s para LLM-judge)
- Ausência de padrões proibidos (ex.: `- {}` na NetworkPolicy)

**LLM-judge** (CP09): falha o build se nota total < 6 OR qualquer critério zerado.

### Custo estimado por execução completa

| Config | Tokens estimados | Custo estimado |
|--------|-----------------|----------------|
| triagem-de-pods (6 chamadas) | ~3k tokens/chamada | ~$0,01 |
| nota-de-triagem (6 chamadas) | ~1k tokens/chamada | ~$0,002 |
| causa-raiz + judge (2 chamadas) | ~8k tokens/chamada | ~$0,03 |
| networkpolicy-sentinel (2 chamadas) | ~2k tokens/chamada | ~$0,005 |
| **Total por execução** | | **~$0,05** |

Gemini 2.0 Flash: ~$0,075/1M tokens input. Os números acima são estimativas conservadoras.

---

## 6. Adicionar novos prompts à biblioteca

1. Criar a pasta na categoria correta: `devops/<nome-do-prompt>/`
2. Criar `prompt.md` com frontmatter + texto + `{{placeholders}}`
3. Criar `README.md` com mesmo frontmatter + documentação + execução
4. (Se saída estruturada) Criar `promptfooconfig.yaml` com asserts
5. Adicionar o novo prompt à tabela do `devops/README.md`
6. Adicionar ao `README.md` raiz
7. Adicionar o config ao workflow `.github/workflows/promptfoo-eval.yml` (detect-changes + matrix)
8. Commit semântico: `feat(devops): adiciona prompt de <objetivo>`

---

## 7. Considerações sobre privacidade e modelo

Para prompts com dados potencialmente sensíveis (CP03 - logs com hostnames internos, CP01/02 com nomes de tenants), seguir o protocolo:

1. Sanitizar antes de enviar ao modelo externo (ver seção "Dados sensíveis" em cada README)
2. Para dados altamente sensíveis (PII, credenciais), usar modelos locais via Ollama
3. Para o pipeline de CI, os exemplos nos `promptfooconfig.yaml` já estão sanitizados — são os dados fictícios do playbook

**Modelo recomendado por tipo de tarefa:**

| Tarefa | Modelo recomendado | Justificativa |
|--------|-------------------|---------------|
| Triagem rápida de pods | `gemini-2.5-flash-lite` | Baixa latência, formato simples |
| Nota de triagem | `gemini-2.5-flash-lite` | Formato fixo, pouco raciocínio |
| Causa-raiz | `gemini-2.5-flash-lite` ou `claude-sonnet-4-6` | Raciocínio complexo, mais capaz |
| NetworkPolicy | `gemini-2.5-flash-lite` ou `claude-sonnet-4-6` | YAML preciso exige mais qualidade |
| Backpressure / Migração | `gemini-2.5-flash` ou `claude-opus-4-8` | Análise comparativa + decisão |

> **Nota (2026-08-28)**: `gemini-2.0-flash-001` foi deprecado. `gemini-3.6-flash` é o sucessor oficial mas tem instabilidade de 503 na API pública. `gemini-2.5-flash-lite` é a alternativa estável testada com sucesso em todos os configs deste playbook.
