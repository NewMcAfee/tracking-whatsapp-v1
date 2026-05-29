# Playbook — Tracking WhatsApp via Kommo CRM

**Versão:** v0.1
**Última atualização:** 2026-05-29
**Escopo desta versão:** Cenário A (CTWA direto — anúncio Meta → WhatsApp, sem LP)
**Tipo:** Playbook genérico (reutilizável, parametrizável por cliente)
**Status:** Trilha A (CAPI nativo) documentável end-to-end · Trilha B (híbrido N8N) documentada com 1 dependência técnica a validar

---

## Objetivo

Capturar a identidade de leads que chegam via WhatsApp por anúncios CTWA (Click-to-WhatsApp) da Meta e devolver eventos de evolução do funil pra Meta, de forma que as campanhas otimizem por eventos profundos (Lead → ... → DealWon) em vez de só "início de conversa".

A diferença em relação ao [playbook Zendesk Sunshine](../playbook-zendesk-sunshine/README.md): o Kommo **unifica três peças num só produto** — é o inbox do WhatsApp (Cloud API nativa), é o CRM (pipeline/funil) **e** tem **Conversions API nativa embutida**. Isso abre um caminho no-code que o stack Zendesk+RD+N8N não tinha.

## A decisão arquitetural central — Trilha A vs. Trilha B

O Kommo tem CAPI nativo, mas ele **só dispara 2 eventos**: `Lead` e `Purchase`. Isso define duas trilhas:

| | **Trilha A — CAPI Nativo** | **Trilha B — Híbrido (Webhook + N8N)** |
|---|---|---|
| **Como funciona** | Mapeia 1 estágio do pipeline → evento `Lead`, e o estágio Close-Won → evento `Purchase`. Tudo dentro do Kommo. | Webhook do Digital Pipeline por estágio → N8N → CAPI for Business Messaging com eventos custom. |
| **Eventos cobertos** | `Lead` + `Purchase` (binário) | `Lead`, `MQL`, `SAL`, `SQL`, `DealWon` (`Purchase`), `DealLost`, `NOICP` — granularidade completa |
| **Infra extra** | Nenhuma (no-code) | N8N + lógica de hash/dedup/EMQ |
| **`event_id` / dedup** | Gerenciado pela Meta via integração nativa | Controlado por você (`event_id` estável) |
| **EMQ tuning** | Limitado ao que o Kommo envia | Total (phone+email+IP+UA hashados sob seu controle) |
| **Cobre maturidade** | **N1 + N2 simples** | **N2 completo + N3** |
| **Quando usar** | Começar rápido, validar CTWA, ciclo simples Lead→Compra | Funil B2B com estágios, value-based bidding granular, múltiplos eventos positivos |

> **Recomendação do playbook:** começar pela **Trilha A** (fecha N1/N2 sem infra), e migrar pra **Trilha B** quando precisar otimizar por evento intermediário (SQL/proposta) ou quando o EMQ nativo não atingir ≥ 7,0. As duas não são mutuamente exclusivas — dá pra rodar nativo pro `Purchase` e N8N pros eventos de meio de funil, desde que se respeite a dedup por `event_id`.

## Pré-requisitos do cliente

| Requisito | Por quê |
|---|---|
| **Kommo plano Pro ou superior** | CAPI nativa exige Pro+ (Base/Advanced só se já estava conectada antes) |
| **WhatsApp conectado via Kommo (Cloud API)** | Kommo é Meta Tech Partner / BSP oficial — conecta a Cloud API nativa, que carrega o `referral` object do CTWA |
| **Conta Meta Business Manager + acesso a CAPI** | Onde os eventos são devolvidos e a campanha otimiza |
| **Anúncios CTWA ativos** (CTA "Enviar mensagem" → WhatsApp) | Sem CTWA não há `ctwa_clid` — é o gatilho de identidade do Cenário A |
| **N8N (cloud ou self-hosted)** | **Só na Trilha B** — desnecessário na Trilha A |

## Os passos canônicos

| # | Etapa | Arquivo | Trilha | Status |
|---|---|---|---|---|
| 1 | Conectar WhatsApp Cloud API no Kommo + validar captura CTWA | [01-setup-kommo-whatsapp-ctwa.md](01-setup-kommo-whatsapp-ctwa.md) | A e B | ✅ Documentado |
| 2 | CAPI nativo do Kommo (Lead + Purchase por estágio) | [02-capi-nativo-kommo.md](02-capi-nativo-kommo.md) | A | ✅ Documentado |
| 3 | Webhook Digital Pipeline → N8N → CAPI granular | [03-webhook-n8n-capi.md](03-webhook-n8n-capi.md) | B | 🟡 1 dependência a validar |
| 4 | Otimizar campanhas Meta por evento de conversão profundo | [04-otimizar-campanhas.md](04-otimizar-campanhas.md) | A e B | 🟡 TODO |

## Arquitetura geral

```
┌─────────────────────────────────────────────┐
│  Anúncio Meta CTWA (CTA "Enviar mensagem")   │
└─────────────────────────────────────────────┘
                    │
                    ▼ 1ª mensagem WhatsApp (carrega referral.ctwa_clid)
        ┌───────────────────────────────┐
        │ Kommo                          │  ← Passo 1
        │ - WhatsApp Cloud API (nativo)  │
        │ - Inbox + lead criado          │
        │ - ctwa_clid mapeado p/ o lead  │
        │ - Digital Pipeline (funil)     │
        └───────────────────────────────┘
            │                       │
   TRILHA A │                       │ TRILHA B
            ▼                       ▼ webhook por estágio
  ┌──────────────────┐    ┌───────────────────────────┐
  │ CAPI nativo Kommo│    │ N8N                       │  ← Passo 3
  │ Lead + Purchase  │    │ - hash PII (SHA-256)      │
  │ (estágio→evento) │    │ - event_id / dedup        │
  │   ← Passo 2      │    │ - POST CAPI Business Msg  │
  └──────────────────┘    └───────────────────────────┘
            │                       │
            └───────────┬───────────┘
                        ▼
        ┌───────────────────────────────┐
        │ Meta Events Manager           │  ← Passo 4
        │ + otimização de campanha      │
        └───────────────────────────────┘
```

## Decisões estratégicas registradas

| Decisão | Escolha | Justificativa |
|---|---|---|
| Borda de entrada (inbox) | Kommo WhatsApp Cloud API nativo | Kommo é BSP oficial Meta → carrega o `referral` object do CTWA sem markup |
| CRM / storage de atribuição | Próprio Kommo (lead + custom fields) | Kommo já é o CRM; dispensa Sheets/Supabase intermediário do playbook Zendesk |
| Trilha inicial | A (CAPI nativo) | Fecha N1/N2 sem N8N; menor custo de implementação |
| Gatilho `Lead` | Estágio "Qualificado"/"Contatado" (não "início de chat") | Sinal de qualidade > volume bruto — Meta otimiza por lead que avançou |
| Gatilho `Purchase` | Estágio Close-Won, com `value` (R$) | Value-based bidding |
| Atribuição multi-clicks | Last-click | Default Meta; menor complexidade |
| Cenário | Só A (CTWA direto) nesta versão | Sem LP, sem Google Ads/`gclid` — fora do escopo v0.1 |

## Origens de lead identificáveis (Cenário A)

| Origem | Como detecta no Kommo | Status |
|---|---|---|
| **CTWA Meta** | Lead criado pelo canal WhatsApp com `referral.ctwa_clid` / ad source mapeado | ✅ (nativo) |
| **Orgânico** | Conversa WhatsApp sem `referral` (número salvo, indicação) | ✅ (ausência de CTWA) |
| **Google Ads / LP** | — | Fora do escopo v0.1 (ver framework Cenário B) |

## O que o Kommo preenche num lead de CTWA (investigado)

| Dado | Auto-preenchido? | Onde | Como usar |
|---|---|---|---|
| **UTMs** (`source`/`medium`/`campaign`/`content`) | ✅ Sim | Statistics → Tracking data | **Forma confiável de origem** — mas exige instrumentar o link Click-to-Chat com UTMs (ver [Passo 1.3](01-setup-kommo-whatsapp-ctwa.md)) |
| **`ctwa_clid`** | ⚠️ Só interno | Não visível no lead | Alimenta o CAPI nativo (Trilha A); **não comprovadamente legível via API** |
| **`ad_id` / nome do anúncio** | ❌ Não | — | Codificar no `utm_content` |

## Decisão pendente (a fechar com teste técnico)

1. **Acesso ao `ctwa_clid` via API (crítico pra Trilha B).** A investigação da documentação (2026-05-29) indica que o Kommo **expõe UTMs** mas **não expõe o `ctwa_clid`** como campo legível — ele o usa internamente pro CAPI nativo. Logo a Trilha B provavelmente atribui via **UTM + phone hash** (probabilística), não via `ctwa_clid` (forte). A confirmar com teste de API + ticket no suporte Kommo — ver [03-webhook-n8n-capi.md](03-webhook-n8n-capi.md), seção "Dependência crítica". A **Trilha A não é afetada** — usa o `ctwa_clid` por dentro.

## Fontes

- [Kommo — Ads that click to WhatsApp + automação](https://www.kommo.com/blog/click-to-whatsapp/)
- [Kommo — Set up Meta Conversions API (CAPI)](https://www.kommo.com/support/messenger-apps/capi-how-to-set-it-up/)
- [Kommo — Meta Conversions API overview](https://www.kommo.com/support/messenger-apps/capi-overview/)
- [Kommo — Connect WhatsApp Business (Cloud API)](https://www.kommo.com/support/messenger-apps/whatsapp-cloud-api-how-to-connect/)
- [Kommo — Track WhatsApp ad campaigns with UTMs](https://www.kommo.com/support/messenger-apps/whatsapp-analytics-utms/)
- [Kommo Developers — Webhooks in Digital Pipeline](https://developers.kommo.com/docs/webhooks-dp)
- [Kommo Developers — Webhooks (general)](https://developers.kommo.com/docs/webhooks-general)
- [Kommo Developers — Salesbot in Digital Pipeline](https://developers.kommo.com/docs/salesbot-dp)
- [Framework conceitual — Tracking de Campanhas → WhatsApp](../framework-conceitual-tracking-whatsapp.md)
</content>
</invoke>
