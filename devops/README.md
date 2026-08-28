# DevOps

Prompts para operações de SRE, Kubernetes, segurança e arquitetura de plataforma.
Todos os prompts desta categoria são parametrizáveis e cobrem o ecossistema da Aegis (Relay, Forge, Sentinel, Cerebro).

## Prompts disponíveis

| Prompt | Checkpoint | Técnica | Testado |
|--------|-----------|---------|---------|
| [triagem-de-pods](./triagem-de-pods/) | CP01 | Chain-of-Thought + saída estruturada | ✅ promptfoo |
| [nota-de-triagem](./nota-de-triagem/) | CP02 | Few-shot (3-shot) | ✅ promptfoo |
| [causa-raiz](./causa-raiz/) | CP03 | Chain-of-Thought (6 passos) | ✅ LLM-as-judge |
| [backpressure-relay](./backpressure-relay/) | CP04 | Análise comparativa (Tree-of-Thoughts) | — |
| [migracao-forge](./migracao-forge/) | CP05 | Cadeia de prompts (3 elos) | — |
| [networkpolicy-sentinel](./networkpolicy-sentinel/) | CP06 | Refinamento iterativo com autocrítica | ✅ promptfoo |
