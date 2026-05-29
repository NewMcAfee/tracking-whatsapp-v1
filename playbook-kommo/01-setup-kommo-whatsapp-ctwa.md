# Passo 1 — Conectar WhatsApp Cloud API no Kommo + validar captura CTWA

**Objetivo:** conectar o WhatsApp Business (Cloud API) no Kommo via integração nativa, garantir que leads de anúncios CTWA entram no CRM com a origem do anúncio mapeada, e validar que o `referral.ctwa_clid` é capturado na primeira mensagem.

**Tempo estimado:** 30-45 min (depende de a conta Meta/WABA já estar verificada)
**Quem executa:** Admin do Kommo + acesso ao Meta Business Manager
**Vale pra:** Trilha A e Trilha B (é a borda de entrada comum às duas)

---

## Pré-requisitos

| Item | Como verificar |
|---|---|
| **Kommo plano Pro+** | Configurações → Assinatura. CAPI nativa (Passo 2) exige Pro |
| **Meta Business Manager verificado** | business.facebook.com → Configurações do negócio → Informações do negócio |
| **Número de telefone disponível** | Não pode estar registrado em outra conta WhatsApp (pessoal ou API) |
| **Permissão de Admin no Kommo** | Sem isso, não cria integração de canal |
| **Anúncio CTWA já criado (ou a criar)** | Meta Ads → objetivo de mensagens → destino WhatsApp → CTA "Enviar mensagem" |

---

## Etapa 1.1 — Conectar o WhatsApp (Cloud API) no Kommo

O Kommo é **Meta Tech Partner / BSP oficial**, então a conexão é direta com a Cloud API da Meta — é isso que garante que o `referral` object do CTWA chega no payload (diferente de soluções não-oficiais tipo WhatsApp Web, que não têm acesso ao Cloud API).

1. No Kommo: **Leads → Configurar** (Digital Pipeline) → coluna da esquerda **"Conectar"** → **WhatsApp**
   - Ou: **Configurações → Integrações → WhatsApp (Cloud API)**
2. Escolher **WhatsApp Cloud API** (não "WhatsApp Lite" / QR-code — esse não é Cloud API e **não** carrega o `referral`)
3. Login com a conta Meta que administra o Business Manager
4. Selecionar/criar a **WhatsApp Business Account (WABA)**
5. Selecionar/registrar o **número de telefone**
6. Concluir a verificação (SMS/ligação no número)
7. O canal aparece como **Ativo** no Digital Pipeline

> **Por que Cloud API e não QR/Lite:** só a Cloud API recebe o webhook da Meta com o objeto `referral` (que contém `ctwa_clid`, `source_id` do anúncio, headline etc.). Conexões por QR-code (engenharia reversa do WhatsApp Web) **não têm** esse objeto — quebram o Cenário A. Ver Cap. 3 do [framework conceitual](../framework-conceitual-tracking-whatsapp.md).

---

## Etapa 1.2 — Habilitar o vínculo com anúncios CTWA

Pra que o Kommo associe a conversa ao anúncio de origem:

1. **Configurações → Integrações → Meta / Facebook** — conectar a página do Facebook e a conta de anúncios ao Kommo
2. Garantir que a WABA usada no anúncio CTWA é a **mesma** conectada no Passo 1.1 **e que ela está no mesmo Meta Business Manager da conta de anúncios** (sem isso, o Kommo não recupera os dados de origem)
3. No gerenciador de anúncios, confirmar que o anúncio usa **CTA "Enviar mensagem" → WhatsApp** (não Messenger/Instagram Direct)

Quando o lead clica e manda a 1ª mensagem, o Kommo:
- cria um **lead novo** (ou reabre o existente) no Digital Pipeline
- recupera os **UTMs do link** (ver Etapa 1.3 — é a forma confiável de marcar origem)
- mapeia internamente o **`ctwa_clid`** — usado pelo CAPI nativo (Passo 2), mas **não exposto como campo legível do lead** (ver "O que o Kommo realmente preenche" abaixo)

---

## Etapa 1.3 — Instrumentar a origem via UTM no link Click-to-Chat ⭐

**Esta é a forma confiável de identificar a origem no Kommo (Cenário A).** Diferente de outros stacks, o Kommo **não** expõe `ctwa_clid` nem `ad_id` como campos do lead. O que ele lê e mostra são **UTMs** — mas só se você os colocar no link.

1. Montar o link Click-to-Chat com UTMs:
   ```
   https://wa.me/55XXXXXXXXXXX?utm_source=facebook&utm_medium=cpc&utm_campaign=<campanha>&utm_content=<criativo/ad>
   ```
2. Usar **esse** link como destino no anúncio (objetivo Mensagens/Leads/Vendas → destino Click to WhatsApp)
3. Codificar a identificação do anúncio no `utm_campaign` / `utm_content` (já que o `ad_id` da Meta **não** vem como campo)
4. **Conceder ao Kommo permissão pra ler metadata do anúncio** (pré-requisito, senão os UTMs não chegam): WhatsApp Business integration → **Accounts** → botão **"Continue with Facebook"** → completar CAPTCHA → selecionar a conta de anúncios (mesmo Business Manager da WABA)
5. Quando o lead clica, o Kommo recupera os UTMs via metadata da Meta e os grava no lead

**Onde aparece:** aba **Statistics → Tracking data** do lead card. Dá pra filtrar leads por UTM em **Leads → Search & filter**.

> **Limitação atual do Kommo:** vincular UTM a conversões de CRM ("qual campanha gerou a venda") ainda **não é nativo** — por isso a Trilha B (N8N) existe pra fechar esse loop.

### O que o Kommo realmente preenche (corrige hipótese anterior)

| Dado | Auto-preenchido? | Onde | Observação |
|---|---|---|---|
| **UTMs** (`source`/`medium`/`campaign`/`content`) | ✅ Sim | Statistics → Tracking data | Só se você os colocou no link Click-to-Chat |
| **`ctwa_clid`** | ⚠️ Interno | Não visível no lead | Usado pelo CAPI nativo; não comprovadamente legível via API |
| **`ad_id` / nome do anúncio** | ❌ Não | — | Codificar no `utm_content` |
| **`ad_headline` / body** | ❌ Não | — | — |

---

## Etapa 1.4 — Definir campos custom complementares

Pra auditoria e pra preparar a Trilha B, criar em **Configurações → Campos personalizados → Leads**:

| Campo custom | Tipo | Preenchido por | Uso |
|---|---|---|---|
| `ctwa_clid` | Texto | N8N (Trilha B), **se** acesso confirmado | Reenvio CAPI Business Messaging |
| `consent_status` | Lista (`granted`/`withdrawn`) | Salesbot/atendente | Log LGPD |
| `consent_timestamp` | Data/hora | Salesbot/atendente | Log LGPD |
| `event_log` | Texto longo | N8N (Trilha B) | `event_id` enviados (dedup/audit) |

> Os UTMs já vêm nativos (Etapa 1.3) — não precisam de campo custom. O `ad_id`/`headline` viram parte do `utm_content`/`utm_campaign`, então também dispensam campo próprio.

---

## Etapa 1.5 — Validação da borda de entrada

Antes de declarar o Passo 1 fechado:

1. **Número de teste 100% novo** (celular que **nunca** falou com o número comercial — senão o `referral` não vem)
2. Rodar um **anúncio CTWA de teste** (orçamento mínimo, R$50-100) com o **link Click-to-Chat instrumentado com UTMs** (Etapa 1.3)
3. Clicar no CTA "Enviar mensagem" e mandar a 1ª mensagem
4. No Kommo, conferir:
   - ✅ Lead novo apareceu no Digital Pipeline
   - ✅ Em **Statistics → Tracking data**, os **UTMs** do anúncio aparecem (`utm_campaign`/`utm_content` que você definiu)
5. Mandar também uma mensagem de **número orgânico** (sem clicar no anúncio) → deve entrar **sem** UTMs

Se o lead CTWA chega com os UTMs e o orgânico chega "limpo": **Passo 1 fechado.**

> **Teste extra (pré-Trilha B):** abrir o lead via API (`GET /api/v4/leads/{id}?with=contacts`) e inspecionar se o `ctwa_clid` aparece em algum campo. Registrar o resultado em [03-webhook-n8n-capi.md](03-webhook-n8n-capi.md) — define o plano da Trilha B.

---

## Anatomia da identidade no Cenário A

| Dado | Origem | Onde no Kommo | Legível? |
|---|---|---|---|
| Phone E.164 | WhatsApp `wa_id` | Campo telefone do contato | ✅ |
| Nome WhatsApp | Profile name | Nome do contato/lead | ✅ |
| **UTMs** | Link Click-to-Chat (você instrumenta) | Statistics → Tracking data | ✅ **forma confiável de origem** |
| **`ctwa_clid`** | `referral.ctwa_clid` (1ª msg) | Mapeado internamente p/ CAPI nativo | ⚠️ não comprovado via API |
| **`ad_id`** | `referral.source_id` | — (codificar no `utm_content`) | ❌ |
| Headline/Body | `referral.headline`/`body` | — | ❌ |

> **Regra de ouro do Cenário A:** o `referral` (com `ctwa_clid`) só chega na **primeira** mensagem do lead após o clique. Como o Kommo não expõe esse objeto bruto, a **identificação de origem prática depende dos UTMs** que você colocou no link — instrumente o link **antes** de subir o anúncio, senão a origem chega vazia e não há "voltar e buscar".

---

## Troubleshooting comum

| Sintoma | Causa provável | Solução |
|---|---|---|
| Lead CTWA chega sem UTMs no Tracking data | Link do anúncio não tinha UTMs | Instrumentar o link Click-to-Chat (Etapa 1.3) e re-subir o anúncio |
| UTMs não aparecem mesmo com link instrumentado | WABA e conta de anúncios em Business Managers diferentes | Etapa 1.2 — mesma WABA + mesmo BM da conta de anúncios |
| Conexão não oferece "Cloud API" | Plano/permissão ou WABA não verificada | Verificar WABA no Business Manager primeiro |
| `referral`/CTWA não funciona | Conexão via QR/Lite, não Cloud API | Reconectar via Cloud API |
| Espera ver `ctwa_clid` no lead e não acha | Kommo usa internamente, não expõe como campo | Normal — use UTMs p/ origem; `ctwa_clid` serve o CAPI nativo (Passo 2) |
| Evento não otimiza campanha depois | `action_source` errado no envio | Garantir `business_messaging` (Passo 2/3) |

---

## Próximo

- **Trilha A (no-code):** → [02-capi-nativo-kommo.md](02-capi-nativo-kommo.md)
- **Trilha B (híbrido):** → [03-webhook-n8n-capi.md](03-webhook-n8n-capi.md)
</content>
