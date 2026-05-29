# Passo 3 — Webhook Digital Pipeline → N8N → CAPI granular (Trilha B)

**Objetivo:** quando o funil tem estágios intermediários que valem otimização, devolver eventos profundos (`Lead`, `MQL`, `SAL`, `SQL`, `DealWon`, `DealLost`, `NOICP`) pra Meta via CAPI for Business Messaging, com `event_id` próprio e controle total de EMQ — usando o webhook nativo do Digital Pipeline do Kommo como gatilho.

**Tempo estimado:** 1-2h (com lógica de hash/dedup montada)
**Quem executa:** operador do N8N + Admin do Kommo
**Trilha:** B (híbrido). Cobre N2 completo + N3.
**Status:** 🟡 documentado, com **1 dependência crítica a validar** (acesso ao `ctwa_clid` via API).

---

## Dependência crítica — acesso ao `ctwa_clid`

A CAPI for Business Messaging atribui o evento à campanha pelo **`ctwa_clid`**. Na Trilha A, o Kommo usa esse valor **internamente** e não precisa expô-lo. Na Trilha B, o ideal é o **N8N ler o `ctwa_clid`** pra montar o payload de atribuição forte.

**O que a investigação da documentação já indica (2026-05-29):**
- O Kommo **expõe os UTMs** do lead (Statistics → Tracking data), legíveis via filtro e provavelmente via API. → use-os como sinal de origem.
- O Kommo **não documenta** o `ctwa_clid` como campo legível do lead — ele o usa internamente pro CAPI nativo. Logo, **a Trilha B provavelmente cai no Cenário C** abaixo, salvo confirmação via API/suporte.

Três cenários, a fechar com teste técnico:

| Cenário | Probabilidade | Plano |
|---|---|---|
| **A — `ctwa_clid` exposto** como campo via API | Baixa (não documentado) | N8N lê o campo via API e usa direto — atribuição forte |
| **B — capturável via Salesbot `widget_request`** na 1ª msg | **Baixa** | A doc do `widget_request` mostra que o payload só carrega os campos que você configura — **não expõe o objeto `referral` bruto** da mensagem. Logo, pescar o `ctwa_clid` na entrada via Salesbot **provavelmente não é viável** |
| **C — não exposto** (mais provável) | **Alta** | **Atribuição via UTM + identity matching:** usar `utm_campaign`/`utm_content` (legíveis em Tracking data) + phone/email hashados no `user_data`. EMQ menor, atribuição probabilística. Alternativa recomendada: deixar `Lead`+`Purchase` na Trilha A (nativa, que tem o `ctwa_clid` por dentro) e usar a Trilha B só pros eventos de meio de funil (SAL/SQL/Proposta) que a nativa não cobre |

> **Ação antes de implementar a Trilha B:** criar um lead CTWA real com link instrumentado (Passo 1.3/1.5), abrir via [API do Kommo](https://developers.kommo.com/reference) (`GET /api/v4/leads/{id}?with=contacts`) e inspecionar (a) se o `ctwa_clid` aparece em algum campo e (b) se os UTMs vêm na resposta. Registrar o resultado aqui — define se a Trilha B é "forte" (Cenário A/B) ou "probabilística" (Cenário C). **Em paralelo, abrir ticket no suporte Kommo** perguntando explicitamente se o `ctwa_clid` é acessível via API.

---

## Pré-requisitos

| Item | Onde |
|---|---|
| Passo 1 concluído | [01-setup-kommo-whatsapp-ctwa.md](01-setup-kommo-whatsapp-ctwa.md) |
| Dependência do `ctwa_clid` resolvida (cenário A/B/C acima) | Esta página |
| N8N rodando (cloud/self-hosted) | — |
| Token de integração privada do Kommo (long-lived) | Kommo → Configurações → Integrações → criar integração privada |
| Acesso de envio à CAPI (token do dataset/pixel) | Meta Events Manager |

---

## Etapa 3.1 — Borda de saída: webhook por estágio no Kommo

O Kommo dispara webhook nativo quando o lead muda de estágio no Digital Pipeline:

1. **Leads → Configurar** (Digital Pipeline) → no estágio desejado, **adicionar ação automática** → **API → Enviar webhook**
2. URL: a Production URL do webhook N8N — ex.: `https://<seu-n8n>/webhook/tracking-whatsapp-kommo-<cliente>`
3. Selecionar o(s) evento(s)/estágio(s) que disparam
4. Repetir pros estágios que viram evento: Qualificado (`MQL`), SAL, SQL, Proposta, Close-Won (`DealWon`), Close-Lost (`DealLost`)

> **Requisito do Kommo:** o endpoint precisa responder **HTTP 200 em até 2 segundos**. No N8N, use o nó Webhook com **Respond: Immediately** (não bloqueie no mesmo fluxo síncrono — processe depois do respond).

**Formato do payload (Digital Pipeline):** o webhook envia campos do lead — incluindo `id` do lead, `status_id` (estágio atual), `pipeline_id`, e os IDs de estágio/pipeline anteriores. Estrutura geral `{"leads":{"status":[{...}]}}` / `{"lead":{"event":{...}}}` conforme o tipo de gatilho.

---

## Etapa 3.2 — N8N: arquitetura do workflow

```
webhook_kommo (recebe POST do Digital Pipeline)
   │  └─ responde 200 imediatamente
   ▼
parse_estagio (Set): lead_id, status_id, pipeline_id → mapeia estágio → event_name
   ▼
get_lead (HTTP → Kommo API): GET /api/v4/leads/{lead_id}?with=contacts
   │   → lê phone, email, value, ctwa_clid (se cenário A), custom fields
   ▼
hash_pii (Code/Crypto): phone E.164 → SHA-256 · email lowercase+trim → SHA-256
   ▼
montar_event_id (Set): event_id = `${lead_id}_${event_name}_${timestamp_iso}`
   ▼
checar_dedup (compara com event_log do lead) ── já enviado? ─→ skip
   ▼
post_capi (HTTP POST → Meta CAPI for Business Messaging)
   ▼
gravar_log (HTTP PATCH → Kommo): append event_id em event_log + grava ACK
```

### Mapa estágio → evento

| Estágio Kommo | `event_name` Meta | Tipo | Otimiza? |
|---|---|---|---|
| Qualificado | `Lead` (ou custom `MQL`) | Positive | Prospecting |
| SAL | custom `SAL` | Positive | Sim |
| SQL / Proposta | custom `SQL` / `ProposalSent` | Positive | Sim (B2B ciclo curto) |
| Close-Won | `Purchase` (com `value`) | Positive | **Sim — value-based** |
| Close-Lost | custom `DealLost` | Other | Sinal p/ exclusion |
| Fora do ICP | custom `NOICP` | Other | Exclusion audience |

---

## Etapa 3.3 — Payload CAPI for Business Messaging

```json
{
  "data": [{
    "event_name": "Purchase",
    "event_time": 1748500000,
    "event_id": "<lead_id>_Purchase_<iso>",
    "action_source": "business_messaging",
    "messaging_channel": "whatsapp",
    "user_data": {
      "whatsapp_business_account_id": "<WABA_ID>",
      "ctwa_clid": "<do lead, se exposto>",
      "ph": "<sha256 phone E.164>",
      "em": "<sha256 email>"
    },
    "custom_data": {
      "currency": "BRL",
      "value": 1990.00,
      "deal_id": "<lead_id>"
    }
  }]
}
```

**Campos não-negociáveis** (idênticos ao framework, Cap. 2.1):
- `action_source: "business_messaging"` — se vier `"website"`, o evento é aceito mas **não atribui** à campanha
- `messaging_channel: "whatsapp"`
- `ctwa_clid` — sem ele, cai pra "organic" (a menos que o fallback de matching por phone/email cubra)

---

## Etapa 3.4 — Deduplicação e idempotência

- **`event_id` estável:** `<lead_id>_<event_name>_<timestamp_iso>` — mesmo padrão do framework (Cap. 5.3)
- Antes de enviar, checar o campo `event_log` do lead (criado no Passo 1.3); se o `event_id` já está lá, **skip**
- Após ACK da Meta, **append** o `event_id` no `event_log` via PATCH na API do Kommo
- Se rodar Trilha A **e** B ao mesmo tempo: NÃO mapeie o mesmo estágio nas duas (ex.: deixe `Purchase` só na nativa OU só no N8N) pra não duplicar. A Meta deduplica por `event_id`, mas o nativo gera o seu próprio — alinhe a fronteira.

---

## Etapa 3.5 — LGPD / hash de PII (obrigatório)

Idêntico ao framework (Cap. 7):
- **Phone** → E.164 (`+5511999999999`) → SHA-256 hex
- **Email** → lowercase + trim → SHA-256 hex
- PII em claro **nunca** sai do Kommo — o N8N hasheia antes do POST
- Gravar `consent_status` + `consent_timestamp` no lead (campos do Passo 1.3); só enviar marketing fora da janela de 24h com opt-in registrado

---

## Validação

1. Mover lead de teste até cada estágio mapeado → conferir execução verde no N8N
2. **Events Manager → Test Events** → cada evento chega com `action_source: business_messaging` + `event_id` único
3. **Diagnostics** → EMQ ≥ 7,0 (Trilha B deve bater isso com phone+email hashados)
4. Reenviar o mesmo evento → confirmar que a dedup por `event_id` descarta (taxa de dedup < 2%)

---

## Limitações conhecidas (estado v0.1)

| Limitação | Impacto | Onde resolver |
|---|---|---|
| Acesso ao `ctwa_clid` via API não confirmado | Bloqueia atribuição precisa se cenário C | Teste técnico (topo desta página) |
| Template N8N ainda não versionado nesta pasta | Implementação manual | Adaptar do [template do playbook Zendesk](../playbook-zendesk-sunshine/templates/workflow-capturar-lead.json) — a borda de saída é nova, a lógica CAPI é a mesma |
| Janela Meta 7d-click (pós-jan/2026) | Ciclo > 7d perde otimização por `Purchase` | Otimizar por `SQL` se ciclo longo (ver framework Cap. 6.3) |

---

## Próximo

→ [04-otimizar-campanhas.md](04-otimizar-campanhas.md)
</content>
