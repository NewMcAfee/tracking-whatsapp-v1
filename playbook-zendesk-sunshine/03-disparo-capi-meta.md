# Passo 3 — Disparar evolução do lead pro Meta CAPI

**Status:** 🟡 TODO — a desenhar
**Objetivo:** quando o lead evoluir no funil do CRM (Lead → MQL → SQL → DealWon/Lost), devolver o evento pro Meta com `ctwa_clid` correto, permitindo otimização da campanha por conversão profunda.

---

## Conceito

```
Lead muda estágio no CRM (RD Station, HubSpot, etc.)
   ↓
CRM dispara webhook → N8N
   ↓
N8N consulta Growthpack pelo phone do lead
   ↓
Se tem ctwa_clid → monta payload Meta CAPI for Business Messaging
   ↓
POST pro endpoint do Dataset Business Messaging
   ↓
Meta atribui o evento ao anúncio que gerou o lead
```

---

## Pré-requisitos (a configurar antes do workflow)

### 1. Criar Dataset Business Messaging no Meta Events Manager

1. `business.facebook.com/events_manager`
2. **Conjuntos de dados** → **Conectar fonte de dados** → **Business Messaging**
3. Conectar à **WhatsApp Business Account** do cliente
4. Anota o **Dataset ID** que aparece

### 2. Gerar Access Token com permissão `business_messaging`

1. `developers.facebook.com` → System Users do Business Manager
2. Cria System User dedicado (ex: `growth-tracking-capi`)
3. Atribui permissão à **WABA** com role apropriado
4. Gera **Access Token** com scope `business_messaging`
5. Guarda token (provavelmente long-lived, mas confirma TTL)

### 3. Definir mapping estágios CRM → eventos Meta

| Estágio funil CRM | Evento Meta CAPI | Tipo |
|---|---|---|
| Lead (entrada) | `Lead` (já disparado pelo Workflow A do passo 2 — futuro) | Standard |
| MQL | `Lead` (com custom subtype) OU custom event `MQL` | Custom |
| SAL | Custom event `SAL` | Custom |
| SQL | Custom event `SQL` | Custom |
| ProposalSent | Custom event `ProposalSent` | Custom |
| **DealWon** | **`Purchase` com value em BRL** | Standard ★ |
| DealLost | Custom event `DealLost` | Custom |
| NOICP (fora do ICP) | Custom event `NOICP` | Custom (pra exclusion audience) |

> Recomendação operacional: focar em `Lead` + `SQL` + `Purchase` (DealWon) no MVP — depois expande granularidade.

---

## Estrutura do payload Meta CAPI for Business Messaging

```json
POST https://graph.facebook.com/v18.0/{dataset_id}/events?access_token={token}

{
  "data": [
    {
      "event_name": "Purchase",
      "event_time": <unix_timestamp>,
      "event_id": "<unique_dedup_id>",
      "action_source": "business_messaging",
      "messaging_channel": "whatsapp",
      "user_data": {
        "whatsapp_business_account_id": "<WABA_ID>",
        "ctwa_clid": "<da Growthpack>",
        "ph": "<sha256(phone_E164)>",
        "em": "<sha256(email)>"
      },
      "custom_data": {
        "currency": "BRL",
        "value": 1500.00,
        "deal_id": "<id no CRM>"
      }
    }
  ]
}
```

**Campos não-negociáveis:**
- `action_source: "business_messaging"` — senão Meta trata como web e não atribui ao CTWA
- `messaging_channel: "whatsapp"`
- `ctwa_clid` — sem isso, não há atribuição

---

## Workflow N8N (sketch — a construir)

```
[Webhook trigger - RD CRM "estágio mudou"]
   ↓
[Set - normaliza dados do evento]
   ↓
[Google Sheets - lookup Growthpack por phone]
   ↓
[IF - tem ctwa_clid?]
   ├─ não → para (lead orgânico ou Lead Form, não atribuível ao CTWA)
   └─ sim → segue
       ↓
   [IF - já disparou Lead pra essa pessoa? (capi_lead_disparado)]
       ├─ não (1ª vez) → event_name = "Lead"
       └─ sim (recorrente) → event_name = "Reengaged"
       ↓
   [Switch - estágio do funil]
       ├─ MQL    → event_name custom
       ├─ SQL    → event_name custom
       ├─ DealWon → event_name = "Purchase" + value BRL
       └─ ...
       ↓
   [Set - monta payload CAPI completo]
       - action_source, messaging_channel, ctwa_clid
       - hash phone + email
       - custom_data (value, currency, deal_id)
       - event_id único (idempotência)
   ↓
   [HTTP Request POST Meta CAPI]
   ↓
   [Google Sheets - atualiza Growthpack]
       - capi_lead_disparado = TRUE (se foi 1ª vez)
       - capi_lead_disparado_em = NOW
       - capi_lead_event_id = event_id usado
```

---

## Decisões pendentes pra fechar Passo 3

| Decisão | Opções | Default sugerido |
|---|---|---|
| Como o CRM RD dispara webhook? | Nativo, Zapier, automation custom | Webhook nativo RD (operador já faz) |
| Cada estágio CRM = 1 evento Meta? | 1:1 ou batch? | 1:1 (real-time, EMQ mais alto) |
| Valor monetário pra `Purchase` | Valor real do deal OU valor médio? | Valor real (vem do CRM) |
| Política `Reengaged` ativada? | Sim/não | Sim — separa aquisição de reativação |

---

## Health metrics a monitorar

| Métrica | Target | Onde |
|---|---|---|
| **EMQ Score** | ≥ 7,0 | Meta Events Manager → Diagnostics |
| **Match rate** (% eventos com `ctwa_clid` válido) | ≥ 80% | Growthpack contagem manual |
| **Latência CRM event → CAPI POST** | < 1h pro `Lead`, < 6h pros downstream | N8N execution timestamps |
| **Taxa de dedup** (eventos descartados Meta) | < 2% | Meta API response logs |

---

## Edge cases já mapeados

| Cenário | Tratamento |
|---|---|
| Lead chegou via Lead Form (sem ctwa_clid), depois fechou venda | Sem atribuição CAPI — esse fluxo já é coberto pela integração Meta-RD nativa do Lead Form (lead_id) |
| Lead chegou via CTWA, mas conversa fechou sem virar contato no CRM | Lead orgânico no CRM, mas tem ctwa_clid na Growthpack. Sem evento downstream pra disparar. OK. |
| Lead manda mensagem mas Comma/Ultimate não cria ticket | Growthpack tem o ctwa_clid mas CRM nunca recebe — sem trigger de Passo 3. Aceitar perda. |
| Mesmo deal muda estágio várias vezes (Lead → MQL → SQL → MQL → SQL → DealWon) | Disparar `MQL` 2× é OK pra Meta (deduplicação por `event_id`). Disparar `Lead` 2× = forçar idempotência via flag. |

---

## Próximo passo

Depois do **Passo 2 fechado e validado em produção com leads CTWA reais**, construir esse workflow.

Skill aplicável: `instrumentation-engineer` + `tracking-engineer` da Fundação Growth IA Ops v2.0 (ver se faz sentido invocar).
