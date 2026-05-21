# tracking-whatsapp-v1

Estudo sobre tracking de campanhas Meta + Google Ads quando o lead vai pro WhatsApp em vez de um formulário.

---

## Problema

Quando o anúncio manda o lead pro WhatsApp (CTWA direto ou via LP com botão), a cadeia de identidade entre o clique no anúncio e a venda fechada quebra. Sem identidade, a plataforma de mídia não recebe sinal de qualidade, a otimização degrada e o CPL sobe.

## Estrutura

Dois níveis:

### 1. Framework conceitual (raiz)

| Arquivo | O que é |
|---|---|
| [framework-conceitual-tracking-whatsapp.md](framework-conceitual-tracking-whatsapp.md) | 5 pilares (identidade, stack técnica, mapa de eventos, pipeline, compliance), cenários A/B (CTWA direto vs LP intermediária), níveis de maturidade N1→N3. Agnóstico de stack — vale pra qualquer BSP. |

### 2. Playbook aplicado — Zendesk Sunshine + N8N

Aplicação do framework no cliente piloto Colina dos Sipes (Grupo Colina). Pasta [`playbook-zendesk-sunshine/`](playbook-zendesk-sunshine/) — ver [README do playbook](playbook-zendesk-sunshine/README.md) pra arquitetura, decisões registradas e particularidades do piloto.

| Passo | Arquivo | Status |
|---|---|---|
| 1. Configurar gatilho no Zendesk Sunshine | [01-setup-zendesk-sunshine.md](playbook-zendesk-sunshine/01-setup-zendesk-sunshine.md) | ✅ Documentado |
| 2. Workflow N8N — captura lead + grava na Growthpack | [02-workflow-n8n-capturar-lead.md](playbook-zendesk-sunshine/02-workflow-n8n-capturar-lead.md) | ✅ Documentado |
| 3. Disparar evolução do funil (MQL/SQL/Ganho/Perdido) pro Meta CAPI | [03-disparo-capi-meta.md](playbook-zendesk-sunshine/03-disparo-capi-meta.md) | 🟡 Esqueleto |
| 4. Otimizar campanhas Meta por evento de conversão profundo | [04-otimizar-campanhas.md](playbook-zendesk-sunshine/04-otimizar-campanhas.md) | 🟡 Esqueleto |
| — | Template: [`workflow-capturar-lead.json`](playbook-zendesk-sunshine/templates/workflow-capturar-lead.json) (N8N) | ✅ Pronto pra importar |

## Status

- **v0.1.** Passos 1 e 2 documentados com instruções executáveis. Passos 3 e 4 têm esqueleto conceitual mas não foram fechados. Pipeline end-to-end ainda não rodou em produção.
- Fora do escopo: form prévio antes do WhatsApp · entradas orgânicas/indicação · plataformas além de Meta/Google · BSPs que não sejam Zendesk Sunshine · setup Google Ads (pendente).

## O que falta

- Fechar passos 3 e 4 do playbook (payload CAPI com `ctwa_clid` + dedup `event_id` + EMQ ≥ 7,0).
- Decidir setup Google Ads: `wa.me?text=` com `gclid` direto vs página intermediária vs número dedicado.
- Validar quais BSPs além do Sunshine expõem `referral.ctwa_clid` no webhook.

## Convenções

- pt-BR.
- Markdown puro, sem build, sem CI.
- Status: `✅` = documentado · `🟡` = esqueleto, incompleto.
