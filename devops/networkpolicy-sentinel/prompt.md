---
nome: Endurecimento de NetworkPolicy do Sentinel
descricao: Converte manifesto de NetworkPolicy permissivo em política hardened com default-deny, e conduz verificação e refinamento iterativo da própria saída
versao: 1.0.0
ferramenta usada: [Claude, Gemini, ChatGPT]
modelo: [gemini-2.5-flash-lite, claude-sonnet-4-6]
tags: [kubernetes, networkpolicy, segurança, hardening, sentinel]
inputs:
  - nome: manifesto_permissivo
    descricao: Manifesto YAML da NetworkPolicy atual, permissivo ou incorreto, que precisa ser corrigido
  - nome: regras_padrao
    descricao: Regras do padrão de segurança da organização que a NetworkPolicy deve seguir
  - nome: mapa_servicos
    descricao: Mapa de labels, namespaces e portas dos serviços envolvidos, para uso correto nos seletores
---

Você é um especialista em segurança Kubernetes com foco em controle de rede. Sua tarefa é analisar um manifesto de NetworkPolicy permissivo, produzir uma versão hardened seguindo as regras do padrão da organização, e então realizar uma verificação crítica da própria saída seguida de refinamento.

## Manifesto atual (permissivo — para corrigir)

```yaml
{{manifesto_permissivo}}
```

## Regras do padrão de segurança da organização

```
{{regras_padrao}}
```

## Mapa de serviços, namespaces e labels

```
{{mapa_servicos}}
```

---

## Fase 1 — Produzir a NetworkPolicy corrigida (v1)

Gere uma NetworkPolicy YAML que:
- Aplica ao namespace e pods corretos, usando `podSelector` com labels — não `podSelector: {}`
- Implementa default-deny implícito: declara `policyTypes: [Ingress, Egress]` sem regras abertas
- Libera ingress apenas para as origens especificadas nas regras, com `namespaceSelector` e `podSelector` corretos conforme o mapa
- Libera egress apenas para os destinos especificados, com protocolos e portas explícitas
- Inclui DNS (TCP e UDP, porta 53) no egress
- Adiciona um comentário `#` em cada regra explicando qual fluxo legítimo ela libera
- NÃO usa `- {}` em ingress ou egress

---

## Fase 2 — Verificação crítica da v1

Após gerar a v1, responda objetivamente a cada pergunta:

1. **Seletores**: os `namespaceSelector` e `podSelector` usam os labels corretos do mapa de serviços? Há algum label inventado?
2. **Escopo do pod**: o `podSelector` da política aponta para os pods corretos — não para `{}` (todos os pods)?
3. **DNS**: ambos TCP e UDP estão presentes na regra de DNS (porta 53)?
4. **Allow-all oculto**: há alguma regra que, na prática, libera tráfego mais amplo do que o pretendido?
5. **Fluxos bloqueados**: algum fluxo legítimo descrito nas regras ficou acidentalmente bloqueado?
6. **Comentários**: toda regra tem um comentário explicando o fluxo que libera?

Para cada ponto que identificar como problema, descreva o problema e a correção necessária.

---

## Fase 3 — Produzir a NetworkPolicy refinada (v2)

Com base nos pontos da verificação, gere a v2:
- Corrija todos os problemas identificados na Fase 2
- Para cada mudança em relação à v1, adicione uma nota inline `# CORRIGIDO: <o que mudou e por quê>`
- Se não houver mudanças necessárias, declare "v2 = v1 — nenhuma correção necessária" com justificativa
