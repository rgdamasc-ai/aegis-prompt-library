---
nome: Endurecimento de NetworkPolicy do Sentinel
descricao: Converte manifesto de NetworkPolicy permissivo em política hardened com default-deny, e conduz verificação e refinamento iterativo da própria saída
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [kubernetes, networkpolicy, segurança, hardening, sentinel]
inputs:
  - nome: manifesto_permissivo
    descricao: Manifesto YAML da NetworkPolicy atual permissivo ou incorreto
  - nome: regras_padrao
    descricao: Regras do padrão de segurança da organização
  - nome: mapa_servicos
    descricao: Mapa de labels, namespaces e portas dos serviços
---

# Endurecimento de NetworkPolicy do Sentinel — CP06

## Objetivo

Recebe um manifesto de NetworkPolicy permissivo, as regras do padrão de segurança da organização e o mapa de serviços/labels, e produz em três fases: (1) a NetworkPolicy corrigida e hardened (v1), (2) uma verificação crítica da própria saída com perguntas de segurança, e (3) a versão refinada (v2) endereçando cada ponto identificado.

## Quando usar

- Antes de aplicar uma NetworkPolicy nova em produção: gere e verifique com este prompt
- Em revisão de segurança: substitui a revisão manual de manifesto por verificação sistemática
- Quando a Natasha pede revisão de um manifesto: use para gerar v1 → verificar → v2 antes de apresentar

## Técnica utilizada

**Refinamento iterativo com autocrítica (self-critique).**

O prompt força o modelo a gerar uma v1, depois criticar a própria v1 com perguntas específicas de segurança, e então produzir a v2 com as correções. Isso é mais robusto do que gerar em uma única passagem porque erros de primeira tentativa (especialmente em YAML com seletores complexos) são capturados antes de chegar ao usuário.

### Por que self-critique aqui?

NetworkPolicy tem sintaxe YAML específica onde um erro sutil (ex.: DNS só com UDP, esquecendo TCP; ou `podSelector: {}` errado) passa despercebido na primeira geração. A autocrítica com perguntas específicas (seletores corretos? DNS tem TCP e UDP? allow-all oculto?) captura esses erros de forma sistemática, sem depender de revisão humana detalhada. As iterações são documentadas como parte do valor da entrega.

## Dados sensíveis a sanitizar

- Labels de namespaces internos: manter (necessários para os seletores)
- Nomes de serviços internos: manter (contexto operacional)
- Não incluir IPs ou credenciais nos parâmetros do prompt

## Execução — modelo: Claude Sonnet 4.6 (Claude Code, 2026-08-27)

### Parâmetros usados:

**manifesto_permissivo**: o manifesto allow-all do playbook

**regras_padrao**: padrão da Aegis para sentinel-prod

**mapa_servicos**: labels e namespaces da Aegis

---

### Fase 1 — NetworkPolicy v1

```yaml
# NetworkPolicy hardened para sentinel-prod — v1
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-hardened
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel  # aplica somente aos pods do Sentinel, não a todos os pods do namespace
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Permite tráfego de entrada do Relay (consumo de eventos assíncronos)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: relay-prod
          podSelector:
            matchLabels:
              app: relay
    # Permite tráfego de entrada do API Gateway da plataforma
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: edge
          podSelector:
            matchLabels:
              app: api-gateway
  egress:
    # Permite saída para Forge (data warehouse PostgreSQL, porta 5432/TCP)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: forge-prod
          podSelector:
            matchLabels:
              app: forge
      ports:
        - protocol: TCP
          port: 5432
    # Permite saída para Cerebro (busca Elasticsearch, porta 9200/TCP)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: cerebro-prod
          podSelector:
            matchLabels:
              app: cerebro
      ports:
        - protocol: TCP
          port: 9200
    # Permite resolução DNS interna (kube-dns) — somente UDP
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

---

### Fase 2 — Verificação crítica da v1

**1. Seletores corretos?**
✅ `namespaceSelector` usa `kubernetes.io/metadata.name` que é o label padrão do Kubernetes para namespaces. `podSelector` usa os labels do mapa (`app: relay`, `app: api-gateway`, etc.).

**2. Escopo do pod correto?**
✅ `podSelector.matchLabels.app: sentinel` — aponta apenas para os pods do Sentinel, não para todos os pods do namespace.

**3. DNS: TCP e UDP?**
❌ **PROBLEMA**: a v1 inclui apenas `protocol: UDP` no DNS. DNS usa UDP para consultas padrão (até 512 bytes), mas cai automaticamente para TCP para respostas maiores (ex.: registros TXT longos, DNSSEC). Sem TCP/53, queries DNS grandes falham silenciosamente, causando timeouts de resolução de nome intermitentes.

**4. Allow-all oculto?**
✅ Não há `- {}` em nenhuma regra. As regras são explicitamente restritivas.

**5. Fluxos legítimos bloqueados?**
✅ Todos os fluxos das regras foram cobertos: Relay → Sentinel (ingress), API Gateway → Sentinel (ingress), Sentinel → Forge:5432 (egress), Sentinel → Cerebro:9200 (egress), Sentinel → DNS:53 (egress).

**6. Comentários em toda regra?**
✅ Toda regra tem comentário explicando o fluxo.

**Ponto a corrigir na v2**: adicionar `protocol: TCP, port: 53` junto com `protocol: UDP, port: 53` na regra de DNS.

---

### Fase 3 — NetworkPolicy v2

```yaml
# NetworkPolicy hardened para sentinel-prod — v2
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-hardened
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel  # aplica somente aos pods do Sentinel, não a todos os pods do namespace
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Permite tráfego de entrada do Relay (consumo de eventos assíncronos)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: relay-prod
          podSelector:
            matchLabels:
              app: relay
    # Permite tráfego de entrada do API Gateway da plataforma
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: edge
          podSelector:
            matchLabels:
              app: api-gateway
  egress:
    # Permite saída para Forge (data warehouse PostgreSQL, porta 5432/TCP)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: forge-prod
          podSelector:
            matchLabels:
              app: forge
      ports:
        - protocol: TCP
          port: 5432
    # Permite saída para Cerebro (busca Elasticsearch, porta 9200/TCP)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: cerebro-prod
          podSelector:
            matchLabels:
              app: cerebro
      ports:
        - protocol: TCP
          port: 9200
    # Permite resolução DNS interna (kube-dns) — UDP para queries padrão + TCP para respostas grandes
    # CORRIGIDO: v1 omitia TCP/53 — DNS cai para TCP em respostas > 512 bytes (DNSSEC, TXT longos)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**Mudança v1 → v2**: adicionado `protocol: TCP, port: 53` na regra de DNS. Todas as demais regras permanecem iguais.

---

## Curadoria

**Modelo**: Claude Sonnet 4.6 (Anthropic) para execução manual; `google:gemini-2.5-flash-lite` (Google) nos testes automatizados.

**Justificativa de modelo**: NetworkPolicy é YAML com sintaxe precisa — erros de primeira tentativa são comuns e difíceis de detectar visualmente. O `gemini-2.5-flash-lite` produziu YAML estruturalmente correto e passou todos os asserts. Para ambientes onde o manifesto contém informações de topologia de rede interna (namespaces reais, ranges de IP), prefer modelo hospedado internamente ou com garantia de não-treinamento (Vertex AI, Bedrock).

**Meta-prompting**: Diretriz usada: _"Crie um prompt que converte uma NetworkPolicy allow-all em uma versão hardened, e então faz a IA criticar a própria saída com perguntas específicas de segurança (seletores corretos, DNS TCP+UDP, allow-all oculto) antes de gerar a versão final. As iterações fazem parte do valor."_

**Refinamentos:**
1. A v1 do prompt não incluía as perguntas específicas na Fase 2. O modelo fazia uma verificação vaga ("parece correto"). Adicionadas 6 perguntas específicas (seletores, DNS, allow-all, fluxos bloqueados, comentários) para tornar a verificação sistemática.
2. A Fase 1 v1 do prompt não especificava "DNS deve ter TCP E UDP" explicitamente — o modelo frequentemente colocava apenas UDP. Adicionado explicitamente na instrução da Fase 1.
3. A Fase 3 v1 não pedia notas inline de correção (`# CORRIGIDO:`). Adicionado para rastreabilidade das mudanças.

## Limitações

- O prompt gera uma NetworkPolicy para um namespace específico. Para múltiplos namespaces, rodar o prompt uma vez por namespace.
- A verificação é feita pelo próprio modelo — não substitui uma revisão de segurança por humano antes de aplicar em produção.
- Para Kubernetes < 1.21, o label `kubernetes.io/metadata.name` pode não estar disponível nos namespaces — verificar e adicionar manualmente se necessário.
- O prompt não gera uma NetworkPolicy separada de "default-deny" — o default-deny é implícito pela presença de `policyTypes` sem regras `- {}`. Para ambientes que exigem uma NetworkPolicy de default-deny explícita e separada, criar manualmente.

## Testes

```bash
export GOOGLE_API_KEY="sua-chave"
promptfoo eval --config ./promptfooconfig.yaml
```

**Resultado real (2026-08-28, `google:gemini-2.5-flash-lite`):**

```
✓ 1 passed (100%) — 0 failed — 0 errors
Duration: ~7s (concurrency: 4)
```

| Entrada | Asserts verificados | Resultado |
|---------|---------------------|-----------|
| manifesto permissivo → hardened | kind:NetworkPolicy + Ingress/Egress + sem `-{}` + portas 5432/9200/53 + labels relay/sentinel/api-gw/forge/cerebro + TCP+UDP | ✅ PASS |

**O que foi ajustado durante os testes:**
- Assertion `cost` removida: provedores Google não reportam custo por chamada.
- `gemini-3.6-flash` substituído por `gemini-2.5-flash-lite` após 503 intermitentes.
- A regra "DNS deve ter TCP E UDP" foi reforçada no prompt durante a Fase 1 após o assert `contains "TCP"` falhar na v1 (modelo usava só UDP por padrão).

**Provedores ao longo do desafio (CP06):**
- **Execução manual (CP06)**: Claude Sonnet 4.6 via Claude Code (Anthropic)
- **Testes automatizados (CP08)**: `google:gemini-2.5-flash-lite` (Google)
- Dois fornecedores distintos: Anthropic + Google ✅

Ver [promptfooconfig.yaml](./promptfooconfig.yaml).
