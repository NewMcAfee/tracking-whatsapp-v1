# tracking-whatsapp-v1

> **Estudo conceitual.** Notas, esboços e experimentos sobre tracking de campanhas Meta + Google Ads quando o lead vai pro WhatsApp em vez de um formulário. Não é playbook, não é skill, não é produto. É raciocínio em voz alta antes de virar qualquer uma dessas coisas.

---

## Por que existe

Quando o anúncio manda o lead pro WhatsApp (direto via CTWA ou via LP com botão), a cadeia de identidade entre o **clique no anúncio** e a **venda fechada** quebra. Sem identidade, a plataforma de mídia não recebe sinal de qualidade, e a otimização da campanha degrada — você paga mais por lead pior.

Esse repo é o caderno de estudo onde eu destrincho o problema antes de propor solução de produção.

## O que tem aqui

| Arquivo | O que é |
|---|---|
| [framework-conceitual-tracking-whatsapp.md](framework-conceitual-tracking-whatsapp.md) | Documento principal — 5 pilares conceituais (identidade, stack técnica, mapa de eventos, pipeline, compliance), cenários A/B (CTWA direto vs LP intermediária), níveis de maturidade N1→N3. |
| [n8n-workflow-buscar-user-sunshine.json](n8n-workflow-buscar-user-sunshine.json) | Workflow N8N exploratório — busca de usuário na Sunshine Conversations API. Pedaço da camada de pipeline. |

## Status

- **v0.1 — conceitual, pré-playbook.** As ideias estão escritas mas não foram validadas em campo, não viraram skill executável, não tem instrumentação real rodando.
- Fora do escopo desta versão: form prévio antes do WhatsApp · entradas orgânicas/indicação · plataformas além de Meta/Google · playbook executável passo-a-passo.

## Próximos passos plausíveis (não-promessa)

- Validar `ctwa_clid` exposto pelo webhook de BSPs reais (nem todo BSP expõe).
- Mapear gclid via página intermediária com pré-preenchimento de mensagem.
- Definir EMQ ≥ 7,0 como meta de done pro CAPI.
- Talvez virar skill `tracking-whatsapp-engineer` no futuro — se o estudo virar conhecimento estável.

## Convenções

- Notas em pt-BR.
- Markdown, sem build, sem CI — repositório de estudo, não de software.
- Se você caiu aqui sem contexto: comece pelo [framework conceitual](framework-conceitual-tracking-whatsapp.md).
