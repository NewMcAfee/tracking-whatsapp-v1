# Passo 2 — Workflow N8N: capturar lead e armazenar na Growthpack

**Objetivo:** receber webhook do Zendesk Sunshine, normalizar dados do lead, identificar origem (CTWA Meta / Google Ads / orgânico), e gravar na planilha Growthpack (storage intermediário pra atribuição).

**Tempo estimado:** 30-45 min (com template já pronto, só ajustar credenciais + IDs)
**Quem executa:** Quem opera o N8N (operador)

---

## Pré-requisitos

| Item | Onde |
|---|---|
| **N8N rodando** | Cloud ou self-hosted |
| **Webhook Zendesk Sunshine configurado** | [Passo 1](01-setup-zendesk-sunshine.md) concluído |
| **Credencial Google Sheets no N8N** | Criada via OAuth2 (preferível Service Account em produção pra ter 300/min em vez de 60/min) |
| **Planilha "Growthpack"** | Compartilhada com a credencial do N8N |
| **Aba `leads`** na Growthpack | Cabeçalhos definidos (ver seção "Schema da aba") |

---

## Arquitetura do workflow

```
webhook_sunshine (recebe POST Sunshine)
   │
   ├──→ normalizando_webhook (Set): nome, phone, IDs base
   │     ├──→ Data Formatada (formata timestamp atual)
   │     │      └─→ normalizando_data (Set: data + phone)
   │     │             └─→ merge[1]
   │     │
   │     ├──→ procurar_leads (Sheets: lookup por phone)
   │     │      └─→ row_lead (Set: row_number do lead se existir)
   │     │             └─→ merge[0]
   │     │
   │     └──→ merge2[1] (combina enriquecimento)
   │
   └──→ enrich_source (Set: source/canal/ad_id placeholders)
          └─→ filter_whatsapp (filtra apenas leads WhatsApp)
                 └─→ merge2[0]
                        ↓
                 merge2 → merge1[0] ← merge[0,1]
                        ↓
                 lead_way (Switch: row_lead vazio?)
                        ├─ create (vazio) → criar_lead (Sheets: append)
                        └─ update (preenchido) → [TODO — branch ainda vazio]
```

## Caminhos do payload — referência rápida

| Dado | Caminho no payload Sunshine |
|---|---|
| Phone E.164 (sem +) | `body.events[0].payload.source.client.externalId` |
| Nome WhatsApp | `body.events[0].payload.source.client.raw.profile.name` |
| First name | `body.events[0].payload.user.profile.givenName` |
| Last name | `body.events[0].payload.user.profile.surname` |
| Sunshine User ID | `body.events[0].payload.user.id` |
| Conversation ID | `body.events[0].payload.conversation.id` |
| Integration ID (canal) | `body.events[0].payload.source.integrationId` |
| Source type | `body.events[0].payload.source.type` (= `whatsapp`) |
| Event type | `body.events[0].type` (= `conversation:create` ou `conversation:referral`) |
| **CTWA Click ID** | `body.events[0].payload.source.client.raw.referral.ctwa_clid` |
| **Ad ID** | `body.events[0].payload.source.client.raw.referral.source_id` |
| Headline anúncio | `body.events[0].payload.source.client.raw.referral.headline` |
| Body anúncio | `body.events[0].payload.source.client.raw.referral.body` |
| URL Meta | `body.events[0].payload.source.client.raw.referral.source_url` |

---

## Schema da aba `leads` na Growthpack

Colunas mínimas pra o template atual funcionar (na ordem que ele escreve):

| Coluna | Tipo | Origem | Notas |
|---|---|---|---|
| `lead_id` | string | `phone` (mesmo valor) | Identificador interno |
| `ad_id` | string | `referral.source_id` (futuro CTWA) | Vazio em orgânico |
| `create_at` | string | data atual formatada | `dd/MM/yyyy HH:mm:ss` |
| `canal` | string | placeholder (a popular conforme Switch) | "CTWA Meta" / "Google Ads" / "Orgânico" |
| `full_name` | string | `source.client.raw.profile.name` | Nome do WhatsApp |
| `first_name` | string | `user.profile.givenName` | |
| `last_name` | string | `user.profile.surname` | |
| `phone` | string | `source.client.externalId` | Chave do merge entre workflows |
| `execution_create_url` | string | URL da execução N8N | Debug |

**Colunas adicionais já mapeadas no schema (mas marcadas `removed: true` no template — placeholders pra futuro):**

- `update_at`, `first_win_at`, `last_win_at`
- `email`, `quality_score`, `total_value`, `wins`, `product_cart`, `purchased_products`
- `utm_campaign`, `utm_medium`, `utm_content`, `gclid`, `fbclid`, `li_fat_id`, `tkclid`
- `user_ip`, `user_agent`, `url`, `conversion_page`
- `country`, `state`, `city`, `street`, `cep`
- `company_name`, `company_phone`, `cnpj`, `cnae`, `regime_tributario`
- `owner_name`, `owner_phone`
- `first_message`, `qualificador_tamanho`, `tier_tamanho`, `qualificador_cargo`, `decision_committee`, `segmento`
- `person_id`, `company_id`

> Esses campos vão sendo populados conforme o framework expande (Passo 3, Passo 4, ou via integrações futuras).

---

## Detalhamento dos nós

### 1. `webhook_sunshine` (Webhook trigger)

| Campo | Valor |
|---|---|
| HTTP Method | POST |
| Path | `tracking-whatsapp-sunshine-<cliente>` |
| Authentication | None |
| Respond | `Immediately` |
| Options → Raw Body | **Desligado** (não vamos validar HMAC) |

**Output:** payload Sunshine completo (`body.events[]`, `headers`, etc.)

### 2. `normalizando_webhook` (Set)

Extrai dados básicos do payload:

```
growthpack_id    = "<ID da planilha hardcoded>"
group_id         = {{ $json.body.events[0].payload.source.integrationId }}
nome             = {{ $json.body.events[0].payload.source.client.raw.profile.name }}
first_name       = {{ $json.body.events[0].payload.user.profile.givenName }}
last_name        = {{ $json.body.events[0].payload.user.profile.surname }}
phone            = {{ $json.body.events[0].payload.source.client.externalId }}
execution        = {{ $execution.id link absoluto }}
```

> O `growthpack_id` é o ID da planilha Google Sheets (parte da URL entre `/d/` e `/edit`). Hardcoded no Set permite que os nós seguintes referenciem via `{{ $json.growthpack_id }}`.

### 3. `Data Formatada` (DateTime)

Formata timestamp atual:
- Input: `{{ $now }}`
- Format: `dd/MM/yyyy HH:mm:ss`
- Output field: `Data Formatada`

### 4. `normalizando_data` (Set)

Repassa `data do evento` (= Data Formatada) + `phone` (chave de merge).

### 5. `procurar_leads` (Google Sheets — Read)

Busca na aba `leads` por `phone`:
- Document ID: `{{ $json.growthpack_id }}`
- Sheet: `leads`
- Lookup column: `phone`
- Lookup value: `{{ $json.phone }}`

**`alwaysOutputData: true`** — garante que mesmo sem match retorna algo (importante pra Switch funcionar).

### 6. `row_lead` (Set)

Captura `row_number` da linha encontrada (ou vazio se não existe):
```
row_lead = {{ $json.row_number }}
phone    = {{ $('normalizando_webhook').item.json.phone }}
```

### 7. `enrich_source` (Set) — preparação CTWA/Google Ads

```
phone   = {{ ...source.client.externalId }}
ads_id  = (placeholder — será preenchido quando CTWA)
ad_id   = (vazio — preencher com referral.source_id)
canal   = (vazio — preencher conforme Switch)
source  = {{ ...source.type }} (= "whatsapp")
```

> 🟡 **TODO:** quando integrar o Switch de origem (CTWA Meta / Google Ads / Orgânico), preencher `ad_id` e `canal` aqui.

### 8. `filter_whatsapp` (Filter)

Garante que só processa eventos de canal `whatsapp`:
- Condição: `{{ $json.source }}` != `whatsApp` → bloqueia (note o casing — deve ser comparado com case real)

### 9. `merge` / `merge1` / `merge2` (Merge nodes)

Combinam fluxos paralelos via match por `phone`:
- `merge`: join `row_lead` + `normalizando_data`
- `merge1`: join `merge` + `merge2`
- `merge2`: join `filter_whatsapp` + `normalizando_webhook`

### 10. `lead_way` (Switch)

Decide criar ou atualizar:
- **update**: `{{ $json.row_lead }}` não vazio → lead já existe
- **create**: `{{ $json.row_lead }}` vazio → lead novo

🟡 **Estado atual:** branch `update` está vazio (não conectado). Apenas `create` está implementado.

### 11. `criar_lead` (Google Sheets — Append)

Cria linha nova na aba `leads`:

```yaml
operation: append
documentId: {{ $json.growthpack_id }}
sheetName: leads
columns:
  lead_id:                {{ $json.phone }}
  ad_id:                  {{ $json.ad_id }}
  create_at:              {{ $json['data do evento'] }}
  canal:                  {{ $json.canal }}
  full_name:              {{ $json.nome }}
  last_name:              {{ $json.last_name }}
  first_name:             {{ $json.first_name }}
  phone:                  {{ $json.phone }}
  execution_create_url:   {{ $json.execution }}
```

---

## Template do workflow

JSON exportável pronto pra importar no N8N:

→ [templates/workflow-capturar-lead.json](templates/workflow-capturar-lead.json)

**Ajustes a fazer após importar:**

1. **Credencial Google Sheets** — apontar pra credencial do projeto (substituir `vdwvKr06yxcLrIgM`)
2. **`growthpack_id`** no nó `normalizando_webhook` — substituir pelo ID da planilha do cliente
3. **Path do webhook** — alterar `tracking-whatsapp-sunshine-colina` pra `<cliente-target>`
4. **URL da execução** no Set — apontar pro N8N do projeto

---

## Limitações conhecidas (estado atual)

| Limitação | Impacto | Onde resolver |
|---|---|---|
| Branch `update` não implementado | Lead recorrente sobrescreve linha em vez de atualizar | Implementar nó Google Sheets "Update Row" no branch `update` do `lead_way` |
| `ad_id` e `canal` ainda não populados | Falta lógica condicional pra CTWA Meta / Google Ads / Orgânico | Adicionar Switch de origem após `enrich_source`, com 3 outputs (set `canal` e `ad_id` em cada branch) |
| `growthpack_id` hardcoded no Set | Trocar de cliente exige editar nó | Mover pra credencial/variável N8N se virar template multi-cliente |
| Google Sheets via OAuth user (60 req/min) | Bottleneck em alta volumetria | Migrar credencial pra Service Account (300 req/min) |
| Lead orgânico sem `referral` chega igual ao CTWA | Granularidade insuficiente pra atribuição | Lógica de Switch já desenhada — implementar |

---

## Como validar o workflow funcionando

1. Workflow N8N **Active**
2. **Mandar mensagem teste** no WhatsApp Comercial Web (número novo) → entra como lead orgânico
3. Conferir na Growthpack aba `leads` → deve aparecer linha nova com `phone`, `full_name`, `create_at`
4. Em Executions do N8N → execução verde (succeeded) no branch `create`

**Quando rodar campanha CTWA de teste (R$50-100):**
5. Mandar mensagem real do anúncio (de celular novo)
6. Conferir Growthpack → linha com `ad_id` populado + `canal` = "CTWA Meta"

---

## Rate limits relevantes (referência rápida)

| API | Limite | Compartilhamento |
|---|---|---|
| **Zendesk Support** | 400 req/min | Conta inteira (todos tokens + Comma + Ultimate + outras integrações) |
| **Google Sheets (OAuth user)** | 60 read + 60 write /min | Por usuário Google autenticado |
| **Google Sheets (Service Account)** | 300 read + 300 write /min | Por projeto Google Cloud |
| **Sunshine Conversations** | N/A — usamos webhook inbound (não fazemos query) | — |

Para Colina (estimativa pós-CTWA, 2-3k leads/mês): uso < 15% de qualquer limite. Sem stress.

---

## Próximo

→ [03-disparo-capi-meta.md](03-disparo-capi-meta.md)
