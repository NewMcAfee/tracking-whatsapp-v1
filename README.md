# tracking-whatsapp-v1

Estudo sobre tracking de campanhas Meta + Google Ads quando o lead vai pro WhatsApp em vez de um formulário.

---

## Problema

Quando o anúncio manda o lead pro WhatsApp (CTWA direto ou via LP com botão), a cadeia de identidade entre o clique no anúncio e a venda fechada quebra. Sem identidade, a plataforma de mídia não recebe sinal de qualidade, a otimização degrada e o CPL sobe.

## Estrutura

Dois níveis: um framework conceitual agnóstico de stack, e playbooks executáveis por stack.

### 1. Framework conceitual (raiz)

| Arquivo | O que é |
|---|---|
| [framework-conceitual-tracking-whatsapp.md](framework-conceitual-tracking-whatsapp.md) | 5 pilares (identidade, stack técnica, mapa de eventos, pipeline, compliance), cenários A/B (CTWA direto vs LP intermediária), níveis de maturidade N1→N3. Agnóstico de stack — vale pra qualquer BSP. |

### 2. Playbook aplicado — Zendesk Sunshine + N8N

Aplicação do framework no cliente piloto Colina dos Sipes (Grupo Colina). Inbox (Zendesk Sunshine) e CRM (RD Station) são peças separadas, ligadas por N8N + Google Sheets. Pasta [`playbook-zendesk-sunshine/`](playbook-zendesk-sunshine/) — ver [README do playbook](playbook-zendesk-sunshine/README.md).

| Passo | Arquivo | Status |
|---|---|---|
| 1. Configurar gatilho no Zendesk Sunshine | [01-setup-zendesk-sunshine.md](playbook-zendesk-sunshine/01-setup-zendesk-sunshine.md) | ✅ Documentado |
| 2. Workflow N8N — captura lead + grava na Growthpack | [02-workflow-n8n-capturar-lead.md](playbook-zendesk-sunshine/02-workflow-n8n-capturar-lead.md) | ✅ Documentado |
| 3. Disparar evolução do funil (MQL/SQL/Ganho/Perdido) pro Meta CAPI | [03-disparo-capi-meta.md](playbook-zendesk-sunshine/03-disparo-capi-meta.md) | 🟡 Esqueleto |
| 4. Otimizar campanhas Meta por evento de conversão profundo | [04-otimizar-campanhas.md](playbook-zendesk-sunshine/04-otimizar-campanhas.md) | 🟡 Esqueleto |
| — | Template: [`workflow-capturar-lead.json`](playbook-zendesk-sunshine/templates/workflow-capturar-lead.json) (N8N) | ✅ Pronto pra importar |

### 3. Playbook aplicado — Kommo CRM

Playbook genérico (reutilizável, não amarrado a cliente). Cobre **Cenário A (CTWA direto)**. Diferença estrutural: o Kommo **unifica inbox WhatsApp + CRM + CAPI nativa** num só produto — por isso há duas trilhas (nativa no-code vs. híbrida com N8N). Pasta [`playbook-kommo/`](playbook-kommo/) — ver [README do playbook](playbook-kommo/README.md).

| Passo | Arquivo | Trilha | Status |
|---|---|---|---|
| 1. Conectar WhatsApp Cloud API + validar captura CTWA | [01-setup-kommo-whatsapp-ctwa.md](playbook-kommo/01-setup-kommo-whatsapp-ctwa.md) | A e B | ✅ Documentado |
| 2. CAPI nativo do Kommo (Lead + Purchase por estágio) | [02-capi-nativo-kommo.md](playbook-kommo/02-capi-nativo-kommo.md) | A (no-code) | ✅ Documentado |
| 3. Webhook Digital Pipeline → N8N → CAPI granular | [03-webhook-n8n-capi.md](playbook-kommo/03-webhook-n8n-capi.md) | B (híbrido) | 🟡 1 dependência a validar |
| 4. Otimizar campanhas Meta por evento profundo | [04-otimizar-campanhas.md](playbook-kommo/04-otimizar-campanhas.md) | A e B | 🟡 Esqueleto |

## Status

- **v0.1.**
  - **Zendesk Sunshine:** passos 1 e 2 executáveis; 3 e 4 em esqueleto. Pipeline end-to-end ainda não rodou em produção.
  - **Kommo:** Trilha A (CAPI nativo) documentável end-to-end — fecha N1/N2-simples sem N8N. Trilha B (híbrido) documentada com 1 dependência técnica a validar (acesso ao `ctwa_clid` via API).
- Fora do escopo: form prévio antes do WhatsApp · entradas orgânicas/indicação · plataformas além de Meta/Google · Cenário B (LP→WA) e Google Ads/`gclid` no playbook Kommo.

## O que falta

- **Kommo:** validar por teste técnico se o `ctwa_clid` é exposto via API/webhook (define a viabilidade da Trilha B); versionar o template N8N da Trilha B; fechar passo 4 com dados de campanha real.
- **Zendesk:** fechar passos 3 e 4 (payload CAPI com `ctwa_clid` + dedup `event_id` + EMQ ≥ 7,0).
- Decidir setup Google Ads: `wa.me?text=` com `gclid` direto vs página intermediária vs número dedicado.
- Validar quais BSPs além do Sunshine expõem `referral.ctwa_clid` no webhook.

## Convenções

- pt-BR.
- Markdown puro, sem build, sem CI.
- Status: `✅` = documentado · `🟡` = esqueleto, incompleto.
