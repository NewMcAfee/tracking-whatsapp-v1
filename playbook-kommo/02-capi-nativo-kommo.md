# Passo 2 — CAPI nativo do Kommo (Trilha A)

**Objetivo:** configurar a Conversions API nativa do Kommo pra devolver os eventos `Lead` e `Purchase` pra Meta, mapeados a estágios do Digital Pipeline — sem nenhuma infra externa.

**Tempo estimado:** 15-25 min
**Quem executa:** Admin do Kommo com acesso ao Meta Business Manager
**Trilha:** A (no-code). Fecha N1 e N2-simples.

---

## O que a CAPI nativa do Kommo faz (e o que não faz)

| Cobre | Não cobre |
|---|---|
| Evento **`Lead`** ao atingir um estágio escolhido | Eventos intermediários distintos (MQL/SAL/SQL/ProposalSent como eventos próprios) |
| Evento **`Purchase`** no Close-Won, com `value` | `event_id` sob seu controle (a Meta gerencia a dedup) |
| `action_source` de business messaging gerenciado | Tuning fino de EMQ (campos de matching que você adiciona manualmente) |
| Dedup gerenciada pela integração | Mais de 2 eventos no funil |

> Se o funil é **Lead → Compra** (ciclo simples, e-commerce, serviço transacional), a Trilha A é suficiente. Se o funil tem **estágios intermediários que você quer otimizar** (B2B com MQL/SQL/proposta), vá pra [Trilha B](03-webhook-n8n-capi.md) — ou combine as duas.

---

## Pré-requisitos

| Item | Onde |
|---|---|
| Passo 1 concluído (WhatsApp Cloud API conectado) | [01-setup-kommo-whatsapp-ctwa.md](01-setup-kommo-whatsapp-ctwa.md) |
| Plano **Pro+** | Configurações → Assinatura |
| Acesso ao **Meta Events Manager** | business.facebook.com → Events Manager |
| Estágios do Digital Pipeline definidos | Ex.: Novo → Contatado → Qualificado → Proposta → Close-Won / Close-Lost |

---

## Etapa 2.1 — Ativar a integração CAPI

1. No Kommo: **Configurações → Integrações** → buscar **"Conversions API"** (Meta CAPI) → **Instalar/Conectar**
2. Autorizar com a conta Meta (a mesma do Business Manager do Passo 1)
3. Selecionar a **página do Facebook** e o **dataset** da conta de anúncios que vai receber os eventos
4. Confirmar que as integrações de mensageria (WhatsApp, e opcionalmente FB/IG) estão vinculadas

> **Qual dataset usar — o mesmo de sempre.** Não precisa de dataset/pixel dedicado pra CTWA. A Meta unificou pixel → dataset: um único dataset recebe web, app, offline **e** business messaging. O que distingue o evento de WhatsApp dos eventos de site **não é o dataset**, é o `action_source: "business_messaging"` + `messaging_channel: "whatsapp"` + `ctwa_clid` (o Kommo monta isso por baixo). Reusar o dataset existente é o caminho recomendado — dataset separado só faz sentido por organização de relatório, nunca por atribuição.
>
> **Pré-requisito:** a WABA precisa estar conectada a uma **página do Facebook** (é o que amarra o WhatsApp ao dataset/conta de anúncios).

---

## Etapa 2.2 — Mapear o estágio → evento `Lead`

**Decisão estratégica:** não mapeie o evento `Lead` pro estágio de "início de conversa". Use um estágio que represente **interesse qualificado** — assim a Meta otimiza por leads que avançam, não por volume bruto de chats.

1. Na configuração da CAPI, seção **Lead event**
2. Selecionar o(s) estágio(s) que disparam `Lead` — recomendado: **"Qualificado"** ou **"Contatado"**
   - Alternativa: um passo inicial do Salesbot que confirma interesse (ex.: lead respondeu à qualificação)
3. Salvar

> **Por quê não o 1º estágio:** "abriu conversa" inclui curiosos, enganos e spam. Mapear `Lead` a "Qualificado" ensina o algoritmo a buscar pessoas parecidas com quem **demonstrou fit**, não com quem só clicou.

---

## Etapa 2.3 — Mapear o Close-Won → evento `Purchase`

1. Seção **Purchase event**
2. Selecionar o estágio **Close-Won** (venda fechada)
3. Vincular o **campo de valor** do lead (`value` / orçamento da venda) → vai como `value` + `currency: BRL` no evento
4. Salvar

Isso habilita **value-based bidding**: a Meta passa a otimizar pra leads com maior valor esperado, não só por quantidade.

---

## Etapa 2.4 — Configurar "Conversion Leads" no Meta Ads

Pra a campanha realmente otimizar pelos eventos vindos do CRM:

1. **Meta Ads Manager** → criar/editar campanha de **mensagens** (CTWA)
2. Em otimização, habilitar **Conversion Leads** (otimizar por leads de conversão do CRM)
3. Escolher o **evento primário** de otimização:
   - Ciclo curto / e-commerce → `Purchase`
   - Ciclo com qualificação → `Lead` (no estágio "Qualificado") até ter volume de `Purchase`
4. Publicar

---

## Etapa 2.5 — Validação

1. Mover manualmente um lead de teste até o estágio mapeado como `Lead`
2. **Meta Events Manager → Test Events / Eventos** → confirmar o evento `Lead` chegando com:
   - `action_source` = business messaging
   - origem WhatsApp
3. Mover um lead até **Close-Won** com valor preenchido → confirmar `Purchase` com `value`
4. **Events Manager → Diagnostics** → checar o **EMQ score** (meta ≥ 7,0)

Se `Lead` e `Purchase` chegam com a origem correta e EMQ aceitável: **Trilha A fechada (N1/N2-simples).**

---

## Health metrics da Trilha A

| Métrica | Target | Onde medir |
|---|---|---|
| EMQ score | ≥ 7,0 | Events Manager → Diagnostics |
| `Lead` chegando por estágio | 100% dos leads que atingem o estágio | Events Manager → Eventos |
| `Purchase` com `value` | 100% dos Close-Won | Events Manager (parâmetro `value` presente) |
| Latência estágio → evento | < 1h | Comparar timestamp do estágio vs. evento |

---

## Quando migrar pra Trilha B

Migre (ou complemente) quando bater num destes limites:

- Precisa otimizar por um **evento intermediário** (ex.: SQL/proposta enviada), não só Lead/Purchase
- EMQ nativo **não atinge 7,0** e você quer adicionar mais campos de matching (email, IP, UA hashados)
- Quer **`event_id` estável próprio** pra auditoria/dedup cross-sistema
- Quer enviar **`DealLost`/`NOICP`** pra alimentar exclusion audiences

→ [03-webhook-n8n-capi.md](03-webhook-n8n-capi.md)

---

## Próximo

→ [04-otimizar-campanhas.md](04-otimizar-campanhas.md) (otimização vale pras duas trilhas)
</content>
