# Playbook de IA Operacional — Aegis

[![Promptfoo Eval](https://github.com/rgdamasc-ai/aegis-prompt-library/actions/workflows/promptfoo-eval.yml/badge.svg)](https://github.com/rgdamasc-ai/aegis-prompt-library/actions/workflows/promptfoo-eval.yml)

Biblioteca de prompts parametrizáveis para a equipe de engenharia da Aegis, tratada como código: versionada, testada e pronta para uso em plantão.

> Entrega do capstone do Módulo 3 — AIOps & GenAI no Cotidiano Técnico.
> Construída sobre as convenções do [prompt-registry](https://github.com/fabricioveronez/prompt-registry).

---

## Como usar um prompt

1. Navegue até a categoria e pasta do prompt desejado.
2. Abra o `README.md` para entender objetivo, parâmetros e limitações.
3. Copie o conteúdo de `prompt.md` e substitua os placeholders `{{nome_variavel}}` pelos valores reais.
4. Cole no seu modelo preferido (Claude, Gemini, GPT-4o, etc.).

## Estrutura

```
prompt-library/
  devops/                     # Operações, SRE, Kubernetes, segurança
    triagem-de-pods/          # CP01 — saúde dos pods do cluster
    nota-de-triagem/          # CP02 — nota padronizada a partir de alerta cru
    causa-raiz/               # CP03 — análise de causa-raiz multi-artefato
    backpressure-relay/       # CP04 — estratégia de backpressure para o Relay
    migracao-forge/           # CP05 — migração batch→event-driven (cadeia)
    networkpolicy-sentinel/   # CP06 — NetworkPolicy hardened para o Sentinel
  .github/
    workflows/
      promptfoo-eval.yml      # CP10 — pipeline de testes em GitHub Actions
  PASSOS-MANUAIS.md           # Guia passo a passo para execução manual
  CLAUDE.md                   # Convenções internas
```

## Categorias

### [DevOps](./devops/)

Triagem de pods, notas de incidente, análise de causa-raiz, decisões de arquitetura, migração de pipeline e hardening de rede.

| Prompt | Checkpoint | Descrição |
|--------|-----------|-----------|
| [triagem-de-pods](./devops/triagem-de-pods/) | CP01 | Triagem da saúde dos pods com causa provável e ação recomendada |
| [nota-de-triagem](./devops/nota-de-triagem/) | CP02 | Nota padronizada de 5 campos a partir de alerta cru |
| [causa-raiz](./devops/causa-raiz/) | CP03 | Análise de causa-raiz cruzando config, métricas e logs |
| [backpressure-relay](./devops/backpressure-relay/) | CP04 | Comparação de estratégias de backpressure com recomendação |
| [migracao-forge](./devops/migracao-forge/) | CP05 | Cadeia de 3 prompts para migrar pipeline batch→event-driven |
| [networkpolicy-sentinel](./devops/networkpolicy-sentinel/) | CP06 | NetworkPolicy hardened com verificação e refinamento iterativo |

## Regras do playbook

1. **Todo prompt é parametrizável** — os dados variáveis entram por `{{placeholders}}`, nunca hardcoded.
2. **Prompts criados via meta-prompting** — a IA gerou e refinou os prompts; o playbook guarda o resultado curado.
3. **Teste é parte do prompt** — cada `promptfooconfig.yaml` viaja junto com o `prompt.md` na mesma pasta.
4. **Versionamento semântico** — cada mudança de prompt passa por commit semântico com escopo na categoria.

## Testes

```bash
# Rodar todos os testes (da pasta prompt-library/)
promptfoo eval --config devops/triagem-de-pods/promptfooconfig.yaml
promptfoo eval --config devops/nota-de-triagem/promptfooconfig.yaml
promptfoo eval --config devops/causa-raiz/promptfooconfig.yaml
promptfoo eval --config devops/networkpolicy-sentinel/promptfooconfig.yaml
```

Variáveis de ambiente necessárias: `GOOGLE_API_KEY`.
Ver [PASSOS-MANUAIS.md](./PASSOS-MANUAIS.md) para outros provedores e detalhes de configuração.

### Evidências de CI (CP10)

| Execução | PR | Resultado | Link |
|----------|----|-----------|------|
| Suíte completa — todos os prompts passando | [PR #1](https://github.com/rgdamasc-ai/aegis-prompt-library/pull/1) | ✅ 3/3 prompts | [Run #6](https://github.com/rgdamasc-ai/aegis-prompt-library/actions/runs/33131732493) |
| Regressão proposital — campo `ALERTA:` renomeado para `ALERTA_CRITICO:` | [PR #2](https://github.com/rgdamasc-ai/aegis-prompt-library/pull/2) | ❌ nota-de-triagem falhou | [Run #7](https://github.com/rgdamasc-ai/aegis-prompt-library/actions/runs/33131750029) |

O pipeline detectou a regressão antes do merge (PR #2 foi fechado sem merge). Ver [PASSOS-MANUAIS.md](./PASSOS-MANUAIS.md#5-decis%C3%B5es-de-design-do-pipeline-cp10--justificativa-estendida) para as 6 decisões de design com alternativas comparadas.

---

## Mapeamento para o prompt-registry (CP07)

Esta biblioteca foi construída sobre as convenções do [prompt-registry](https://github.com/fabricioveronez/prompt-registry). As decisões de mapeamento:

### Categoria única `devops/`

O playbook cobre seis prompts com cenários distintos (Kubernetes, alertas, Elasticsearch, Kafka-like, migração de pipeline, Kubernetes Network Policies), mas todos pertencem ao domínio de operações de plataforma. A opção por uma única categoria `devops/` em vez de subcategorias (`kubernetes/`, `alerting/`, `pipeline/`) seguiu a regra do template: "não aninhar categorias". Agrupar por domínio operacional é mais fácil de navegar do que por tecnologia específica.

### Nomes de pasta pelo resultado, não pela técnica

| Pasta | Técnica usada | Por que esse nome |
|-------|---------------|-------------------|
| `triagem-de-pods/` | Chain-of-Thought | O resultado é a triagem, não "o método CoT" |
| `nota-de-triagem/` | Few-shot 3-shot | O resultado é a nota padronizada |
| `causa-raiz/` | CoT + LLM-as-judge | O resultado é a análise de causa-raiz |
| `backpressure-relay/` | Comparative analysis | O resultado é a estratégia de backpressure |
| `migracao-forge/` | Prompt chaining (3 elos) | O resultado é a migração — a cadeia é implementação |
| `networkpolicy-sentinel/` | Self-critique iterativo | O resultado é a NetworkPolicy hardened |

### Parâmetros do prompt → `inputs` no frontmatter

Cada `{{placeholder}}` no corpo do `prompt.md` tem entrada correspondente em `inputs:` no frontmatter — com `nome` idêntico ao placeholder (sem chaves) e `descricao` explicando o que o parâmetro representa. Isso torna os parâmetros descobríveis por qualquer engenheiro sem precisar ler o prompt inteiro.

### `promptfooconfig.yaml` como terceiro arquivo

O template original define `prompt.md` + `README.md` por pasta. Para os 4 prompts de saída estruturada (CP01, CP02, CP06, CP03), o arquivo `promptfooconfig.yaml` foi adicionado na mesma pasta — o teste viaja junto com o prompt, como código de teste viaja com o código de produção.
