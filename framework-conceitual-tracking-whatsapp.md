# Framework Conceitual — Tracking de Campanhas → WhatsApp

**Versão:** v0.1 (conceitual, pré-playbook)
**Data:** 2026-05-20
**Escopo:** Anúncios Meta + Google Ads → WhatsApp (com ou sem LP intermediária)
**Cenários cobertos:** (A) Anúncio → CTWA direto · (B) Anúncio → LP → CTA WhatsApp
**Fora do escopo desta versão:** Form prévio antes do WhatsApp · entradas orgânicas/indicação · plataformas além de Meta/Google · playbook executável

---

## Sumário Executivo (1 página)

**O problema** — quando o lead vai pro WhatsApp em vez de um form, a cadeia de identidade entre o clique no anúncio e a venda fechada quebra. Sem identidade, a plataforma de mídia (Meta/Google) não recebe sinal de qualidade e a otimização da campanha degrada.

**A solução em uma frase** — capturar o click ID nativo da plataforma na **borda de entrada** (webhook do WhatsApp para Meta, redirect intermediário para Google), persistir no CRM atrelado ao lead, e devolver eventos do funil (Lead → MQL → SQL → DealWon/Lost) via API server-side (Meta CAPI for Business Messaging + Google Ads Offline Conversion Import).

**Os 5 pilares conceituais:**

| # | Pilar | Decisão crítica |
|---|---|---|
| 1 | **Identidade** | Meta = `ctwa_clid` nativo no webhook · Google = `gclid` via página intermediária |
| 2 | **Stack técnica** | BSP precisa expor `referral.ctwa_clid` — não é todo BSP que expõe |
| 3 | **Mapa de eventos** | Quais estágios do funil viram conversão pra plataforma (não é "todos" sem critério) |
| 4 | **Pipeline** | CRM → webhook → N8N → CAPI/Google Ads API (server-side, dedup com `event_id`) |
| 5 | **Compliance** | LGPD opt-in explícito + hash SHA-256 de PII + EMQ ≥ 7,0 como meta de done |

**Estado de maturidade desejada:**
- **N1 (mínimo)** — Lead chega com `ctwa_clid` no CRM, evento `Lead` devolve pra Meta via CAPI
- **N2 (funcional)** — Eventos `MQL` + `SAL` + `DealWon` devolvem com `event_id` consistente, EMQ ≥ 7,0
- **N3 (previsível)** — Google Ads paralelo via `gclid`, dedup cross-platform, dashboard de match rate

**Riscos estruturais já mapeados:**
- Cenário B (LP → WhatsApp) com Google Ads é **mais frágil** que Cenário A — depende de cookie `_gcl_aw` sobreviver até o clique no botão WhatsApp
- `ctwa_clid` só chega na **primeira mensagem** do lead — se BSP não captura imediatamente, perde
- Janela Meta encolheu em 2026 (28d view removida) — latência do evento agora importa mais

---

## Cap. 1 — Cenários de Entrada Canônicos

### 1A — Anúncio → CTWA direto (sem LP)
**Fluxo:** lead vê anúncio Meta com CTA "Enviar Mensagem" → app WhatsApp abre → primeira mensagem dispara webhook com `referral.ctwa_clid`

**Plataforma:** Meta only (Google Ads não tem equivalente nativo de CTWA — destino do anúncio é sempre uma URL HTTP)

**Vantagem estratégica:** 92% lower CPL e até 94% mais conversão vs. mesma campanha com LP intermediária (dados Meta LATAM/Brasil/Índia 2024-2025)

**Identidade:** `ctwa_clid` nativo, server-side, sem dependência de cookie ou JS

### 1B — Anúncio → LP → CTA WhatsApp
**Fluxo:** lead clica anúncio → cai em LP → clica botão WhatsApp → abre `wa.me/?text=...`

**Plataforma:** Meta + Google Ads (e qualquer outra)

**Vantagem estratégica:** permite pré-qualificar com copy + capturar dados de browsing (heatmap, scroll, tempo na página) antes do contato

**Identidade — Meta:**
- Anúncio Meta + LP: o tráfego é "website conversion", não CTWA → **não recebe `ctwa_clid`**
- Identidade depende de `fbclid` na URL → Pixel + CAPI capturam `fbc/fbp` → matching via hashed phone+email

**Identidade — Google:**
- `gclid` chega na URL da LP → Conversion Linker grava cookie `_gcl_aw` (TTL 90 dias)
- Botão WhatsApp precisa **anexar o `gclid` ao link `wa.me`** OU disparar conversão "Click on WhatsApp" antes do redirect
- Sem isso, o `gclid` é "lost in redirect" — caso mais comum de quebra de atribuição

### Resumo: O que muda entre A e B

| Aspecto | Cenário A (CTWA) | Cenário B (LP → WA) |
|---|---|---|
| Plataformas suportadas | Meta only | Meta + Google + qualquer outra |
| Click ID Meta | `ctwa_clid` (nativo) | `fbclid` + Advanced Matching |
| Click ID Google | N/A | `gclid` (cookie + parâmetro WA) |
| Onde capturar identidade | Webhook 1ª msg | LP (JS) + redirect ao WA |
| Fragilidade principal | BSP não expor `referral` | `gclid` perdido no redirect |
| Custo de implementação | Médio (BSP+CAPI) | Alto (LP+JS+CAPI+OCI) |

---

## Cap. 2 — Mecânica de Identidade por Plataforma

### 2.1 — Meta: o `ctwa_clid`

**O que é:** identificador único gerado por Meta para cada clique em anúncio CTWA. Equivalente conceitual do `gclid` do Google, mas exclusivo do contexto WhatsApp.

**Onde aparece:** dentro do objeto `referral` no payload do webhook que Meta envia pra Cloud API quando o lead manda a **primeira mensagem** após clicar no anúncio.

**Estrutura aproximada do objeto `referral`** (esquema documentado em fontes oficiais e BSPs):
```json
{
  "referral": {
    "source_url": "https://fb.me/...",
    "source_type": "ad",
    "source_id": "<ad_id>",
    "headline": "<ad headline>",
    "body": "<ad body copy>",
    "media_type": "image" | "video",
    "image_url": "...",
    "video_url": "...",
    "thumbnail_url": "...",
    "ctwa_clid": "<unique click id>"
  }
}
```

**Regras de captura críticas:**
1. O `referral` só vem na **primeira mensagem** do lead após o clique. Mensagens subsequentes do mesmo lead **não** trazem o objeto.
2. Se o BSP não persistir o `ctwa_clid` na hora, a janela está fechada — não há "voltar e buscar".
3. Casos reportados de `ctwa_clid` ausente em alguns webhooks existem (issue documentada nos developer forums da Meta) — exige monitoramento de match rate.

**O que devolver pra Meta (CAPI for Business Messaging):**
```json
{
  "data": [{
    "event_name": "Lead" | "Purchase" | "<CustomEvent>",
    "event_time": <unix timestamp>,
    "event_id": "<unique dedup id>",
    "action_source": "business_messaging",
    "messaging_channel": "whatsapp",
    "user_data": {
      "whatsapp_business_account_id": "<WABA_ID>",
      "ctwa_clid": "<from referral>",
      "ph": "<sha256 hash phone E.164>",
      "em": "<sha256 hash email>"
    },
    "custom_data": {
      "currency": "BRL",
      "value": <decimal>
    }
  }]
}
```

**Campos não-negociáveis:**
- `action_source: "business_messaging"` — se vier `"website"`, evento é aceito mas **não atribui à campanha**
- `messaging_channel: "whatsapp"` — distingue de Messenger/Instagram Direct
- `ctwa_clid` — sem ele, a atribuição cai pra "organic"

### 2.2 — Google Ads: o `gclid`

**O que é:** identificador único de cada clique em anúncio Google Ads. Vem como parâmetro na URL de destino.

**Como sobrevive até o WhatsApp:**
- O Conversion Linker (tag Google) lê o `gclid` da URL e grava em **cookie first-party `_gcl_aw`** (TTL 90 dias)
- Ao clicar no botão WhatsApp da LP, é preciso **injetar o `gclid` no link `wa.me`** como parâmetro de texto (ex: `wa.me/55119...?text=Quero%20info%20gclid:XYZ`)
- O lead manda a mensagem com o texto pré-preenchido → BSP extrai o `gclid` do texto → grava no CRM atrelado ao lead

**Fragilidades:**
- Se a LP redireciona via servidor sem preservar query string, o `gclid` se perde
- Se o usuário edita o texto antes de enviar, o `gclid` pode ser apagado
- Tracker temporal de 90 dias e indexação Google de 6h (não pode enviar conversão antes de 6h após o clique)

**O que devolver pra Google (Offline Conversion Import / Enhanced Conversions for Leads):**
- Via Google Ads API ou upload manual (CSV)
- Identificador: `gclid` (preferido) OU email/phone hashado (Enhanced Conversions for Leads)
- Conversion Action no Google Ads precisa estar criada para cada estágio do funil que se quer atribuir (Lead, MQL, SQL, Customer geralmente são 4 conversion actions separadas)
- Janela: `gclid` válido por 90 dias após o clique

### 2.3 — Identity matching e fallbacks (cross-platform)

Quando nem `ctwa_clid` nem `gclid` chegam (cenário degradado), o fallback é **identity matching** via:
- Phone hashado (E.164 normalizado, lowercase, SHA-256)
- Email hashado (lowercase, trim, SHA-256)
- IP + User Agent (Meta usa em Advanced Matching automatic)
- Cookies `_fbp`/`_fbc` (Meta) — só viabilizados em cenário B (com LP)

Meta usa esses sinais pra recompor probabilisticamente o match com o usuário. Quanto mais campos, maior o EMQ score (target ≥ 7,0).

---

## Cap. 3 — Stack Técnica: Comparativo de BSPs

### 3.1 — Critério-mestre: quem expõe `referral.ctwa_clid`?

**Confirmados (com `ctwa_clid` no webhook):**
- **Twilio** — campo `ReferralCtwaClid` no callback (documentação oficial 2024-2026)
- **WOZTELL** — documentação técnica explícita do CTWA conversion flow
- **360dialog** — acesso direto à Cloud API Meta sem markup → expõe payload nativo
- **Infobip / Sinch / MessageBird (Bird)** — confirmado em docs

**Provavelmente expõem (mas exige validação técnica):**
- **Take Blip** — BSP brasileiro autorizado Meta, infra robusta
- **Zenvia** — BSP autorizado Meta
- **Wati / AiSensy / Interakt** — mid-market, geralmente expõem webhook raw

**Risco de não expor (ou expor com defasagem):**
- **Z-API** — não é BSP oficial Meta, usa engenharia reversa do WhatsApp Web → **não tem acesso ao referral object da Cloud API** (limitação estrutural)
- **BotConversa** — opera majoritariamente com BSPs subjacentes; depende de qual está por baixo
- Wrappers no-code que normalizam payload → podem dropar campos que consideram "não-essenciais"

### 3.2 — Comparativo por critério

| BSP | `ctwa_clid` exposto | Pricing | Foco BR | Cloud API nativo |
|---|---|---|---|---|
| **360dialog** | Sim (raw payload) | Subscription €49+/mês, zero markup | Sim | Sim |
| **Twilio** | Sim (`ReferralCtwaClid`) | Pay-per-message + markup | Não específico | Sim |
| **Take Blip** | Provável (validar) | Enterprise BR | Sim (líder BR) | Sim |
| **Zenvia** | Provável (validar) | Enterprise BR | Sim | Sim |
| **Infobip** | Sim | Enterprise | Sim (escritório BR) | Sim |
| **Wati / Interakt / AiSensy** | Sim (mid-market) | SaaS-like, $40-200/mês | Não específico | Sim |
| **BotConversa** | Depende do BSP por trás | SaaS BR | Sim | Indireto |
| **Z-API** | **Não** (sem Cloud API) | Low-cost BR | Sim | **Não** (WhatsApp Web) |

### 3.3 — Recomendação por porte (preliminar — validar em playbook)

- **Small (≤ R$5k/mês de mídia, ≤ 1k msgs/mês)** → Twilio (pay-per-use, sem fixo) ou Wati/Interakt (SaaS turnkey)
- **Mid (R$5-50k/mês de mídia, 1-10k msgs/mês)** → 360dialog (subscription baixo, zero markup escala bem)
- **Enterprise / BR-first (>50k/mês, foco em Pix/LGPD/PT-BR full stack)** → Take Blip, Zenvia ou Infobip

**Anti-recomendação para tracking sério:** Z-API. Não é BSP Meta, não tem Cloud API, não vai expor `referral.ctwa_clid`. Pode servir pra atendimento básico, mas **quebra o framework**.

---

## Cap. 4 — Mapa de Eventos: Funil → Plataforma

### 4.1 — Estágios canônicos do funil

Eventos do core (Growth IA Ops v2.0):
1. `Lead` — primeira mensagem do lead (entrada no funil)
2. `MQL` — Marketing Qualified Lead (passou critério de fit mínimo)
3. `SAL` — Sales Accepted Lead (SDR aceitou trabalhar)
4. `SQL` — Sales Qualified Lead (proposta enviada / discovery feito)
5. `ProposalSent` — proposta formal enviada
6. `Negotiation` — em negociação
7. `DealWon` — venda fechada (com `value` em R$)
8. `DealLost` — perdido (com `reason`)
9. `NOICP` — fora do ICP (exclusion audience)

### 4.2 — Quais devolver pra Meta (CAPI)

**Recomendação:** devolver todos, mas **classificar** entre "Positive stages" (otimização) e "Other stages" (sinal):

| Estágio funil | Evento Meta | Tipo | Otimiza campanha? |
|---|---|---|---|
| `Lead` | `Lead` (standard) | Other | Sinal de volume |
| `MQL` | `Lead` (custom subtype) ou `CompleteRegistration` | Other → Positive | Pode otimizar campanha de prospecting |
| `SAL` | Custom event `SAL` | Positive | Sim |
| `SQL` | Custom event `SQL` | Positive | Sim |
| `ProposalSent` | Custom event `ProposalSent` | Positive | Sim |
| `DealWon` | `Purchase` (standard, com `value`) | Positive | **Sim — bidding por valor** |
| `DealLost` | Custom event `DealLost` | Other | Sinal pra exclusion |
| `NOICP` | Custom event `NOICP` | Other | Exclusion audience |

**Configuração Meta:** habilitar "Conversion Leads" no Ads Manager → mapear stages como Positive vs Other → escolher evento primário (geralmente `SQL` ou `DealWon` dependendo do ciclo).

### 4.3 — Quais devolver pra Google Ads (Offline Conversion Import)

Cada estágio = uma **Conversion Action** separada no Google Ads. Estrutura:

| Estágio | Conversion Action Google | Category | Optimization |
|---|---|---|---|
| `Lead` | "WhatsApp Lead" | Submit lead form | Conversion tracking |
| `MQL` | "WhatsApp MQL" | Qualified lead | Bidding |
| `SQL` | "WhatsApp SQL" | Qualified lead | Primary (B2B ciclo curto) |
| `DealWon` | "WhatsApp Customer" | Purchase | Primary (com value-based bidding) |

**Janela Google:** evento precisa ser uploadado dentro de **90 dias** do `gclid`, com latência mínima de **6h** após o clique (indexação).

### 4.4 — Custom data por estágio

| Evento | Custom data essencial |
|---|---|
| `Lead` | `source: "ctwa"`, `wa_phone_hash`, `event_id` |
| `MQL` | + `qualification_score`, `icp_match: true/false` |
| `SQL` | + `proposal_value` (estimado), `sales_rep_id` |
| `DealWon` | + `value` (R$), `currency: "BRL"`, `deal_id`, `cycle_days` |
| `DealLost` | + `loss_reason`, `competitor` (se aplicável) |
| `NOICP` | + `noicp_reason`, `flag_exclusion: true` |

---

## Cap. 5 — Arquitetura de Pipeline (CRM → N8N → Plataformas)

### 5.1 — Diagrama lógico

```
[Anúncio Meta] ──CTWA──→ [WhatsApp App] ──msg 1──→ [BSP webhook]
                                                         │
                                                         ▼
                                                  [N8N: captura ctwa_clid]
                                                         │
                                                         ▼
                                                  [CRM: cria Lead + grava ctwa_clid]
                                                         │
                                                         ▼
                                                  [CAPI Meta: evento "Lead"]

[Mudança de estágio no CRM (Lead → MQL → SQL → DealWon)]
                  │
                  ▼
        [CRM webhook → N8N]
                  │
                  ├──→ [Meta CAPI: custom event]
                  └──→ [Google Ads OCI: gclid + conversion action]
```

### 5.2 — Componentes do pipeline

**Borda de entrada (BSP → N8N):**
- BSP recebe webhook de mensagem do WhatsApp
- N8N nó "Webhook trigger" recebe payload bruto
- N8N parseia: extrai `wa_id` (telefone), `referral.ctwa_clid`, `text` (pra extrair `gclid` se cenário B)
- N8N cria/atualiza Lead no CRM com campos custom: `ctwa_clid`, `gclid`, `ad_id`, `ad_headline`, `consent_timestamp`

**Borda de saída (CRM → plataformas):**
- CRM dispara webhook em mudança de estágio (ou job N8N cron diário fazendo polling)
- N8N consome webhook, monta payload CAPI (Meta) + OCI (Google)
- N8N envia via HTTP node, recebe ACK, grava `event_id` no CRM pra dedup
- Retry policy: 3 tentativas com backoff exponencial, queue de falhas

### 5.3 — Deduplicação (`event_id`)

- **Meta:** `event_id` único por evento — se mandar 2× o mesmo, Meta deduplica
- **Google:** sem campo `event_id` nativo, mas evita duplicação por (gclid + conversion_action + timestamp)
- **Pattern recomendado:** `event_id = <lead_id>_<stage>_<timestamp_iso>` (estável e único)

### 5.4 — Latência aceitável

| Plataforma | Janela mínima | Janela ótima | Janela máxima |
|---|---|---|---|
| Meta CAPI | Real-time | < 1h | 7 dias (EMQ degrada) |
| Google OCI | 6h (indexação gclid) | < 24h | 90 dias (gclid expira) |

**Recomendação:** real-time pro `Lead`, batch a cada 1-6h pros eventos downstream (MQL/SQL/DealWon).

### 5.5 — Variáveis críticas a persistir no CRM

| Campo | Origem | Uso |
|---|---|---|
| `ctwa_clid` | Webhook BSP (1ª msg) | CAPI for Business Messaging |
| `gclid` | Texto da 1ª msg (cenário B) | Google Ads OCI |
| `fbclid` / `_fbp` / `_fbc` | LP (cenário B) | CAPI website (se aplicável) |
| `phone_e164` | WhatsApp `wa_id` | Hash → user_data |
| `email` | Discovery / form pós-WA | Hash → user_data |
| `consent_timestamp` + `consent_text_version` | 1ª msg + template policy | LGPD log |
| `ad_id` / `ad_headline` | Referral object | Reporting |
| `event_log[]` | Cada estágio + event_id enviado | Dedup + audit trail |

---

## Cap. 6 — Atribuição, Janelas e Edge Cases

### 6.1 — Janelas de atribuição (estado 2026)

**Meta (mudança importante em 2026):**
- Em **12 de janeiro de 2026**, Meta removeu permanentemente as janelas **7-day view** e **28-day view** da Ads Insights API
- Default atual: **7-day click + 1-day view**
- Em **3 de março de 2026**, Meta restringiu click-through a cliques reais (likes/comments/shares não contam mais como "click")
- **Impacto:** conversões reportadas caíram 15-40% overnight; ciclos de venda > 7 dias agora ficam invisíveis pra otimização

**Google Ads:**
- Janela padrão: **30 dias** (click-based)
- `gclid` válido por **90 dias** para upload de offline conversion
- Modelo padrão: data-driven attribution (DDA) desde 2023

### 6.2 — Modelo de atribuição

| Plataforma | Padrão | Alternativas |
|---|---|---|
| Meta | Last-click 7d / view 1d | Não há mais options de view 7/28d |
| Google | Data-Driven (DDA) | Last-click, First-click, Linear, Time decay, Position-based |

**Decisão recomendada:** aceitar os defaults e não mexer no modelo — energia maior em **EMQ + completude de eventos** vs. ajuste de modelo.

### 6.3 — Edge cases canônicos

| Cenário | Comportamento esperado | Mitigação |
|---|---|---|
| Lead manda 2 msgs antes do webhook processar | `ctwa_clid` só vem na 1ª — se BSP perde, é perda permanente | BSP confiável + monitoramento de match rate |
| Lead clica em 2 campanhas diferentes antes de mensagar | `ctwa_clid` da última prevalece (last-click) | Aceitar |
| Lead já existia no CRM antes da campanha | Criar `Lead` event novo + atrelar `ctwa_clid` ao deal | Política: re-atribuir só se mudança de estágio |
| Lead manda msg do anúncio mas conversa de outro número | Cross-device: phone hash matching no Meta resolve parcialmente | Aceitar gap; EMQ cobre |
| Ciclo de venda > 7 dias | Click-window Meta expira → conversão "não atribuída" | Custom event `DealWon` ainda chega via CAPI; otimização Meta sofre — ajustar ciclo na narrativa de campanha (otimizar pra SQL em vez de DealWon se ciclo > 7d) |
| Lead salva número direto e mensageia (sem clicar) | Não há `ctwa_clid` — entrada orgânica | Fora do escopo (cenário 1c/1d) |
| Texto da msg foi editado (cenário B) e `gclid` removido | Fallback pra identity matching via phone+email hash | Aceitar degradação |

### 6.4 — Health metrics do sistema

| Métrica | Target | Como medir |
|---|---|---|
| EMQ score (Meta CAPI) | ≥ 7,0 | Events Manager → Diagnostics |
| Match rate (% eventos com `ctwa_clid`) | ≥ 80% | N8N → contador de eventos com vs sem ctwa_clid |
| Latência média evento real → CAPI | < 1h pro Lead, < 6h downstream | Log timestamps no N8N |
| Taxa de dedup (eventos descartados por `event_id` duplicado) | < 2% | CAPI response logs |
| `gclid` válido no upload (não expirado) | 100% | Pré-validação no N8N (idade do gclid) |

---

## Cap. 7 — LGPD, Consent e Governança

### 7.1 — Princípio LGPD aplicado a WhatsApp

WhatsApp marketing exige **opt-in explícito**, **separado** do opt-in de email/SMS. Multas reais começaram em 2025: até 2% do faturamento (cap R$ 50M por violação).

### 7.2 — Estratégias de opt-in por cenário

**Cenário A (CTWA direto):**
- Lead clicou no anúncio (Meta) → ação ativa de iniciar conversa **conta como interesse manifesto** mas **não** substitui consentimento explícito pra marketing message subsequente
- Template de 1ª resposta deve ser **utility** (não-marketing) ou **session message** (24h após contato do lead)
- Pra marketing message fora da janela 24h, capturar opt-in explícito na 1ª conversa (ex: "Posso te mandar novidades? Responda SIM/NÃO")

**Cenário B (LP → WhatsApp):**
- Banner de consent na LP cobre cookies (Pixel, Conversion Linker) — não cobre marketing WhatsApp
- Opt-in pra WhatsApp pode ser capturado: (a) checkbox na LP antes do redirect, OU (b) prompt na 1ª msg

### 7.3 — Log de consentimento (obrigatório)

Pra cada lead, gravar no CRM:
- `consent_timestamp` (ISO 8601 UTC)
- `consent_source` (`lp_checkbox` | `whatsapp_prompt` | `ctwa_click_implicit`)
- `consent_text_version` (ID da versão do texto exato apresentado)
- `consent_ip` (IP no momento do consent — cenário B)
- `consent_status` (`granted` | `withdrawn`)

### 7.4 — Hash de PII

**Obrigatório antes de sair do CRM:**
- Email → lowercase + trim → SHA-256 hex
- Phone → E.164 (`+5511999999999`) → SHA-256 hex
- CPF/CNPJ → numeric-only → SHA-256 hex (não enviar, normalmente)

**Política:** PII em claro **nunca** sai do CRM. N8N hashpoints antes do envio pra CAPI/OCI.

### 7.5 — Retenção

- Dados conversados no WhatsApp: política do BSP (default 6-12 meses)
- Dados no CRM: política do projeto (típico 5 anos pra dados fiscais, 2 anos pra dados de marketing)
- Logs de consent: 5 anos mínimo (ANPD pode auditar)

### 7.6 — Meta template policy (relevante pra reativação)

- Marketing templates exigem opt-in registrado
- Utility templates (transactional) podem ser enviados sem opt-in se forem genuinamente transacionais
- Misclassification (marketing como utility) = principal causa de banimento de número Meta

---

## Anexo A — Decisões Pendentes (pra fechar antes do playbook)

1. **BSP final** — depende de teste real de webhook + custo + integração com CRM do projeto. Recomendação: começar testando 360dialog (raw payload) ou Twilio (ReferralCtwaClid documentado).
2. **CRM** — o framework assume CRM com webhook outbound em mudança de estágio. Validar se CRM em uso (Pipedrive, RD Station, HubSpot, etc.) suporta isso nativamente.
3. **Granularidade de eventos** — devolver todos os 9 eventos canônicos OU subset? Custom events além de 50 inflam Events Manager — recomendação preliminar: Lead, MQL, SQL, DealWon, DealLost (5 essenciais).
4. **Cenário B (LP) sem CTWA** — implementar `gclid` no `wa.me?text=` ou subdomain redirect? Trade-off entre UX (texto pré-preenchido editável) e captura confiável.
5. **EMQ target** — ≥ 7,0 é canônico. Atingir exige Advanced Matching com phone+email+IP+UA — validar quais campos o BSP já entrega.

---

## Anexo B — Próximos passos sugeridos

1. **Validar** este conceitual com o operador (este documento) — ajustes/redirecionamentos
2. **Construir Cap. 8 — Playbook executável** com:
   - Estrutura do workflow N8N (nó por nó)
   - Payloads JSON exatos (CAPI + OCI) com placeholders
   - Schema de campos custom no CRM
   - Checklist de QA pré-go-live (EMQ verificável, match rate, latência)
   - Plano de teste end-to-end (campanha de teste com R$50 → validar evento chegando no Events Manager)
3. **Considerar transformar em skill da biblioteca** (`whatsapp-tracking-architect`) via `god` skill — se o framework for usado em múltiplos clientes
4. **Cruzar com skills existentes** — verificar overlap/extensão com `measurement-architect`, `instrumentation-engineer`, `tracking-engineer` da Fundação Growth IA Ops v2.0

---

## Anexo C — Fontes

### Meta / CTWA / CAPI
- [Meta for Developers — Conversions API for Business Messaging (Onboarding Guide)](https://developers.facebook.com/documentation/ads-commerce/conversions-api/business-messaging)
- [Meta for Developers — Ads that Click to WhatsApp](https://developers.facebook.com/documentation/ads-commerce/marketing-api/ad-creative/messaging-ads/click-to-whatsapp)
- [Meta for Developers — Conversions API for CRM Integration](https://developers.facebook.com/docs/marketing-api/conversions-api/conversion-leads-integration/)
- [Meta for Developers — WhatsApp Cloud API Webhooks Components](https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks/components/)
- [Meta — API de Conversões para Business Messaging (PT-BR)](https://developers.facebook.com/docs/marketing-api/conversions-api/business-messaging?locale=pt_BR)
- [Click-to-WhatsApp Is Your WooCommerce Attribution Black Hole (Seresa)](https://seresa.io/blog/attribution-measurement/click-to-whatsapp-is-your-woocommerce-attribution-black-hole)
- [Click-to-WhatsApp Ads (CTWA): 2026 guide (AsisteClick)](https://asisteclick.com/en/blog/click-to-whatsapp-ads-ctwa-conversion-2026/)
- [Conversions API (CAPI) for WhatsApp Ads – WOZTELL](https://support.woztell.com/portal/en/kb/articles/wa-conversion-flow)
- [Meta CAPI: The Complete Guide (admove.ai)](https://www.admove.ai/blog/meta-capi-guide)
- [Meta Killed Its 28-Day View Attribution Window on January 12, 2026 (Seresa)](https://seresa.io/blog/attribution-measurement/meta-killed-its-28-day-view-attribution-window-on-january-12-2026)
- [Meta Ads Attribution Settings 2026 (jetfuel.agency)](https://jetfuel.agency/meta-ads-attribution-settings/)
- [Optimising Meta Ads for HubSpot CRM Conversion Events (GLO)](https://generateleads.online/optimising-meta-campaigns-for-hubspot-crm-conversion-events/)

### Google Ads / GCLID / OCI
- [Google Ads Help — Set up offline conversions using GCLID](https://support.google.com/google-ads/answer/7012522?hl=en)
- [Google Ads API — Upload offline conversion](https://developers.google.com/google-ads/api/samples/upload-offline-conversion)
- [Google Ads API — Manage offline conversions](https://developers.google.com/google-ads/api/docs/conversions/upload-offline)
- [GCLID Stripped by Redirects (Adnan Agic)](https://adnanagic.com/blog/gclid-stripped-by-redirects/)
- [Master Google Ads Enhanced Conversions with GTM 2026 (ClickSambo)](https://clicksambo.com/blog-detail/google-ads-conversion-tracking-gtm)
- [Google Ads Offline Conversion with API 2026 (ClickSambo)](https://clicksambo.com/blog-detail/google-ads-offline-conversion-with-api-2026-setup-guide)

### BSPs
- [Twilio — Click ID Callback Parameter for Inbound WhatsApp Messages](https://www.twilio.com/en-us/changelog/new--click-id--callback-parameter-for-inbound-whatsapp-messages-)
- [Twilio — 2026 guide to create ads that click to WhatsApp](https://www.twilio.com/en-us/blog/products/2026-guide-to-create-ads-that-click-to-whatsapp-with-twilio)
- [Best WhatsApp Business API Providers in Brazil 2026 (Message Central)](https://www.messagecentral.com/blog/best-whatsapp-business-api-providers-brazil)
- [WhatsApp BSP Comparison 2026 (EZContact)](https://ezcontact.ai/en/blog/whatsapp-bsp-comparison/)
- [Twilio vs 360Dialog (Kommunicate)](https://www.kommunicate.io/blog/twilio-vs-360dialog-a-comparison/)

### N8N / Pipeline
- [N8N — WhatsApp Business Cloud integrations](https://n8n.io/integrations/whatsapp-business-cloud/)
- [N8N — Webhook + WhatsApp Business Cloud](https://n8n.io/integrations/webhook/and/whatsapp-business-cloud/)
- [N8N integration with WhatsApp + CRM (Serverspace 2026)](https://serverspace.io/about/blog/how-to-set-up-n8n-integration-with-whatsapp-crm-and-online-store-to-sell-more-in-2026/)

### LGPD / Consent
- [Meta Business Help — LGPD compliance for Marketing Messages API for WhatsApp](https://www.facebook.com/business/business-messaging/compliance/whatsapp-lgpd-mm-api-for-whatsapp)
- [WhatsApp Marketing Brazil 2026 (Message Central)](https://www.messagecentral.com/blog/whatsapp-marketing-brazil)
- [WhatsApp Business API GDPR Compliance: Opt-In & Data (OnSync)](https://onsync.co/blog/compliance-whatsapp-business)
- [Data Protection Laws Brazil 2025-2026 (ICLG)](https://iclg.com/practice-areas/data-protection-laws-and-regulations/brazil)
