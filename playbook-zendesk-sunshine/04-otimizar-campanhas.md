# Passo 4 — Otimizar campanhas Meta por evento de conversão profundo

**Status:** 🟡 TODO — a definir após Passo 3 funcionar
**Objetivo:** configurar as campanhas Meta pra otimizarem por eventos profundos (Lead → SQL → DealWon) em vez de apenas "Início de conversa". Foco é deixar o algoritmo Meta encontrar leads que **fecham**, não só leads que **mandam mensagem**.

---

## Por que isso importa

| Cenário | Comportamento Meta |
|---|---|
| Sem CAPI configurado | Otimiza por "Mensagens iniciadas" (sinal raso) — muitos leads sem qualificação |
| Com CAPI mas otimizando por `Lead` | Otimiza por "lead chegou" — melhora um pouco, mas ainda raso |
| **Com CAPI + otimização por `SQL` ou `Purchase`** | **Otimiza por quem realmente vira oportunidade/venda** — algoritmo encontra perfis com maior LTV |

Diferença real esperada (dados Meta LATAM): redução de **40-60% no CPL qualificado** quando otimiza por evento profundo vs início de conversa.

---

## Pré-requisitos (a satisfazer antes)

| Item | Status atual no Colina |
|---|---|
| Passo 1 (Sunshine) funcionando | ✅ |
| Passo 2 (workflow captura) funcionando | 🟡 em validação |
| Passo 3 (CAPI Meta) disparando eventos profundos | 🟡 TODO |
| **Volume mínimo de eventos pra otimização sair de "learning phase"** | 🟡 Depende — Meta precisa ~50 eventos/semana do tipo otimizado |
| EMQ score ≥ 7,0 em produção | 🟡 Validar após Passo 3 ativo |

---

## Decisões estratégicas a tomar

### 1. Qual evento usar como "Conversion Event" da campanha?

Meta permite escolher **1 evento primário** + opcionalmente eventos "supporting" pra ajudar learning.

**Hierarquia de decisão (depende do ciclo de venda):**

| Ciclo de venda do produto | Evento primário recomendado | Eventos secundários (supporting) |
|---|---|---|
| Curto (≤ 3 dias entre lead e fechamento) | **`Purchase`** (DealWon) | Lead, SQL |
| Médio (4-14 dias) | **`SQL`** | Lead, MQL, Purchase |
| Longo (15-30 dias) | **`MQL`** | Lead, SQL |
| Muito longo (>30 dias — fora da janela Meta de 7d click) | **`Lead`** com qualificação alta + segmentação pré-tráfego | — (otimizar pra volume e filtrar via lookalike depois) |

**Pro Colina (Plano funerário):**
- Ciclo provavelmente **curto** (venda emocional, decisão rápida)
- Recomendação preliminar: **otimizar por `Purchase`** OU se volume baixo, por `SQL`

### 2. Conversion Leads vs Highest Volume optimization

| Modelo | Quando usar |
|---|---|
| **Highest Volume** | Quando quer escala máxima e CPL baixo (mesmo aceitando qualidade variável) |
| **Conversion Leads** (otimiza por valor downstream) | Quando tem CAPI configurado + quer leads que avancem no funil |

**Recomendação:** **Conversion Leads** assim que CAPI estiver entregando eventos com EMQ ≥ 7,0.

### 3. Bidding strategy

| Estratégia | Quando |
|---|---|
| Highest Volume | Default Meta. Pra começar e gerar baseline. |
| Cost per Result Goal | Quando tiver dados históricos pra setar meta de CPL/CPS |
| ROAS goal | Quando `Purchase` event tiver `value` em BRL trafegando consistentemente |

---

## Configuração no Ads Manager

### Passo a passo (a executar quando Passo 3 estiver vivo)

1. **Ads Manager** → criar campanha → objetivo **Vendas** ou **Engajamento** com WhatsApp
2. No nível **Conjunto de anúncios**:
   - **Local de conversão:** WhatsApp
   - **Evento de conversão:** seleciona o evento criado via CAPI (ex: `Purchase` ou `SQL`)
   - **Otimização:** Conversion Leads (não "Conversas")
3. **Audience:** lookalike do `DealWon` audience (criado a partir da Growthpack ou CRM) — mais quente que lookalike de `Lead`
4. **Janela de atribuição:** 7d click + 1d view (default 2026 — após Meta remover 28d view)

---

## Audience strategies (recomendadas conforme volume cresce)

### Curto prazo (primeiras 4 semanas)

- **Public open:** geo + idade + interesses funeral
- Lookalike 1% de Lead Form histórico (não tem ainda DealWon via CAPI)

### Médio prazo (8-12 semanas com CAPI estável)

- **Lookalike 1% de DealWon** (audience custom alimentada via Growthpack export → Meta Custom Audience)
- **Lookalike 1% de SQL** (segundo melhor sinal)
- **Exclusion:** customers atuais + `NOICP` (audience de leads que não fecharam o ICP)

### Longo prazo (após 3+ meses)

- **Value-based lookalike:** quando `Purchase` event tiver `value` real trafegando, Meta consegue achar perfis de **alto LTV**, não só "quem compra"
- **Engaged audiences:** retargeting de quem chegou até MQL mas não fechou — campanha de **Reengagement** dedicada

---

## Health metrics da otimização

| Métrica | Target | Sinal |
|---|---|---|
| **CPL `Lead`** | Baseline atual R$80 (Lead Form) — esperado cair pra R$30-50 | Eficiência aquisição |
| **CPL `SQL`** | A definir após histórico | Qualidade de leads |
| **CPS `Purchase`** | A definir | Bottom line |
| **Conversion rate Lead → SQL** | A medir histórico CRM | Qualidade Meta delivery |
| **Conversion rate SQL → DealWon** | A medir histórico CRM | Eficácia comercial (não Meta) |
| **EMQ Score** | ≥ 7,0 | Qualidade do tracking |
| **Saiu de Learning Phase** | ≥ 50 eventos/semana do tipo otimizado | Estabilidade |

---

## Cuidados operacionais

| Risco | Mitigação |
|---|---|
| Trocar evento primário em campanha que já saiu de learning → reset | Não mudar evento sem clonar campanha. Testar em campanha nova. |
| Volume baixo do evento profundo → ficar em learning eternamente | Começar com evento de volume médio (MQL/SQL) e migrar pra Purchase quando volume estabilizar |
| EMQ baixo (<6) reduz eficiência do algoritmo | Validar matching keys (phone hash, email hash, IP, UA) antes de ativar Conversion Leads |
| Atribuição cross-platform (Google Ads + Meta brigando pelo mesmo lead) | Definir política clara: last-click default Meta pra leads CTWA; last-click default Google pra leads via Search; Growthpack registra qual foi |

---

## Próximo (futuro)

Depois do Passo 4 ativo e validado:

- **Workflow batch de enrichment Meta** — buscar campaign_name + adset_name via Marketing API (1x/dia) pra reports humanos na Growthpack
- **Dashboard de pipeline CTWA** — Looker/Sheets com funil Lead → MQL → SQL → DealWon segmentado por campanha
- **A/B "Iniciar conversas" vs "WhatsApp Flows form"** — testar se form intermediário sobe ou desce qualificação líquida (CPL × conversion rate)
- **Expansão pra Google Ads** — replicar pipeline com gclid (depende do setup de link)
- **Expansão pra outros canais Sunshine** — Colina 0077, ColinaClin, etc. (mesma arquitetura, integrationId distinto)
