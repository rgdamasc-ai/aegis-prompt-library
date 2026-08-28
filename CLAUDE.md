# CLAUDE.md — Playbook de IA Operacional da Aegis

Convenções internas para criação e manutenção dos prompts desta biblioteca.
Baseadas no template [prompt-registry](https://github.com/fabricioveronez/prompt-registry).

## Estrutura obrigatória

```
<categoria>/
  <nome-do-prompt>/
    prompt.md              # frontmatter + texto do prompt com {{placeholders}}
    README.md              # mesmo frontmatter + documentação humana
    promptfooconfig.yaml   # testes (obrigatório para prompts de saída estruturada)
```

- **Categoria**: pasta na raiz, em `kebab-case`. Não aninhar categorias.
- **Nome do prompt**: pasta dentro da categoria, descrevendo o **resultado**, não a técnica.
- **`prompt.md`**: apenas frontmatter + texto. Sem explicações sobre o prompt — o README faz isso.
- **`README.md`**: mesmo frontmatter + documentação (objetivo, quando usar, execução, limitações).
- **`promptfooconfig.yaml`**: obrigatório para prompts de saída estruturada (CP01, CP02, CP06). Para saída aberta, usar LLM-as-judge (CP03). Prompts de cadeia e decisão (CP04, CP05) documentam limitações.

## Frontmatter padrão

```yaml
---
nome: Nome humano do prompt
descricao: Uma linha descrevendo o objetivo
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [tag1, tag2, tag3]
inputs:
  - nome: variavel_um
    descricao: O que essa variável representa
---
```

- `versao`: semver, começa em `1.0.0`.
- `inputs`: lista todos os `{{placeholders}}` presentes no corpo do prompt.
- Frontmatter idêntico em `prompt.md` e `README.md`.

## Convenções de conteúdo

- Português (pt-BR) em tudo.
- Placeholders no formato `{{nome_variavel}}`.
- Sem comentários explicando "o que o prompt faz" dentro do `prompt.md`.
- Dados sensíveis (hostnames internos, nomes de tenants, credentials) nunca entram nos prompts — sanitizar antes.

## Git

- Commits semânticos com escopo na categoria: `feat(devops): adiciona prompt de triagem de pods`
- Versão inicial de todo prompt: `1.0.0`
- Toda mudança de comportamento do prompt: incrementar `MINOR` ou `MAJOR` no frontmatter + commit semântico
