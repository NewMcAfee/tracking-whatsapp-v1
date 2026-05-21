# Playbook — Tracking WhatsApp via Zendesk Sunshine Conversations

**Versão:** v0.1
**Última atualização:** 2026-05-21
**Cliente piloto:** Colina dos Sipes (Grupo Colina)
**Status:** Em construção — Passos 1 e 2 fechados; 3 e 4 pendentes

---

## Objetivo

Capturar identidade de leads que chegam via WhatsApp (CTWA Meta, Google Ads, orgânico) e devolver eventos de evolução do funil para as plataformas de mídia (Meta CAPI + Google Ads Offline Conversion), de forma que as campanhas otimizem por eventos profundos (Lead → MQL → SQL → DealWon) em vez de apenas "início de conversa".

## Pré-requisitos do cliente

| Requisito | Por quê |
|---|---|
| **Zendesk Suite Professional ou superior** | Único plano que dá acesso à Conversations API + Sunshine Conversations |
| **WhatsApp Business API conectado via Sunshine** | Permite receber webhooks com `referral` object (CTWA) |
| **N8N (cloud ou self-hosted)** | Orquestração dos workflows |
| **Google Sheets** (planilha "Growthpack") | Storage intermediário pra atribuição (alternativa a Supabase) |
| **CRM com webhook outbound** (RD, HubSpot, Pipedrive) | Sinaliza mudança de estágio do funil → dispara CAPI |
| **Conta Meta Business Manager com acesso a CAPI for Business Messaging** | Onde os eventos são devolvidos |

## Os 4 passos canônicos

| # | Etapa | Arquivo | Status |
|---|---|---|---|
| 1 | Configurar gatilho no Zendesk Sunshine | [01-setup-zendesk-sunshine.md](01-setup-zendesk-sunshine.md) | ✅ Documentado |
| 2 | Workflow N8N — capturar lead + armazenar na Growthpack | [02-workflow-n8n-capturar-lead.md](02-workflow-n8n-capturar-lead.md) | ✅ Documentado |
| 3 | Disparar evolução do lead (MQL/SQL/Ganho/Perdido) pro Meta CAPI | [03-disparo-capi-meta.md](03-disparo-capi-meta.md) | 🟡 TODO |
| 4 | Otimizar campanhas Meta por evento de conversão profundo | [04-otimizar-campanhas.md](04-otimizar-campanhas.md) | 🟡 TODO |

## Arquitetura geral

```
┌─────────────────────────────────────────────────────────────┐
│  Anúncio Meta CTWA / Google Ads / Orgânico                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼ msg WhatsApp
              ┌─────────────────────────┐
              │ Zendesk Sunshine        │
              │ + custom integration    │  ← Passo 1
              │   (webhook outbound)    │
              └─────────────────────────┘
                            │
                            ▼ webhook
              ┌─────────────────────────┐
              │ Workflow N8N            │  ← Passo 2
              │ - Recebe webhook         │
              │ - Normaliza dados       │
              │ - Lookup Growthpack     │
              │ - Cria/atualiza lead    │
              └─────────────────────────┘
                            │
                            ▼ append
              ┌─────────────────────────┐
              │ Growthpack (Sheets)     │
              │ aba: leads              │
              └─────────────────────────┘
                            │
                            ▼ (quando CRM evolui estágio)
              ┌─────────────────────────┐
              │ Workflow N8N            │  ← Passo 3 (futuro)
              │ - Webhook do CRM        │
              │ - Lookup ctwa_clid      │
              │ - POST Meta CAPI         │
              └─────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │ Meta Events Manager     │  ← Passo 4 (futuro)
              │ + otimização campanha   │
              └─────────────────────────┘
```

## Decisões estratégicas registradas

| Decisão | Escolha | Justificativa |
|---|---|---|
| Storage intermediário | Google Sheets (Growthpack) | Operador já tem; menos infra extra que Supabase/Postgres |
| Validação HMAC do webhook Sunshine | **Pulada** (skip) | Decisão consciente do operador — URL privada, risco baixo, ganho mínimo |
| Atribuição em multi-clicks | Last-click | Default Meta; menor complexidade |
| Disparo CAPI Lead em re-engagement | Custom event `Reengaged` (não `Lead` de novo) | Não infla métricas; campanha de retargeting otimiza por evento próprio |
| Eventos downstream (MQL/SQL/DealWon) | Last-click no momento do evento | Consistência cross-evento |
| Validação de webhook por API key | Pulada — usa URL como segurança por obscuridade | Consciente; risco baixo |

## Origens de lead identificáveis

| Origem | Como detecta no payload | Status |
|---|---|---|
| **CTWA Meta** | Presença de `events[0].payload.source.client.raw.referral.ctwa_clid` | ✅ Implementado |
| **Google Ads** | Depende do setup (link `wa.me?text=` com `gclid`, página intermediária, ou número dedicado) | 🟡 Pendente decisão do setup |
| **Orgânico** | Ausência dos critérios acima | ✅ Implementado |

## Cliente piloto — particularidades

- **Setup Zendesk:** Suite Professional, 110 agentes, 22 packs Sunshine MAUs (~56k MAUs/mês)
- **Volume Meta hoje:** R$80k/mês, 1.000 leads/mês via Lead Form (CPL ~R$80)
- **Migração:** Lead Form → CTWA, em paralelo por ~2 semanas
- **Stack:**
  - WhatsApp via Sunshine (canal `Comercial Web`: +55 11 2190-6000)
  - Switchboard ativo: `ultimate` (chatbot AI)
  - CRM: RD Station CRM
  - N8N: self-hosted em `n8n-imob.imob.collieassociados.com`
  - Google Sheets: Growthpack como storage intermediário
  - Outros canais Sunshine: Colina 0077, ColinaClin (não cobertos por esse playbook ainda)
