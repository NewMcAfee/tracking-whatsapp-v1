# tracking-whatsapp-v1

> **Caderno de estudo.** Como fazer tracking de campanhas Meta + Google Ads quando o lead vai pro WhatsApp em vez de um formulário — começa conceitual e desce até um playbook aplicado num cliente piloto real. Não é produto, não é skill estável; é raciocínio em voz alta evoluindo conforme o problema é destrinchado.

---

## Por que existe

Quando o anúncio manda o lead pro WhatsApp (CTWA direto ou via LP com botão), a cadeia de identidade entre o **clique no anúncio** e a **venda fechada** quebra. Sem identidade, a plataforma de mídia não recebe sinal de qualidade, a otimização degrada e você paga mais por lead pior.

Esse repo é onde eu destrincho o problema — primeiro no abstrato, depois aplicado num caso concreto.

## Como o conteúdo está organizado

São dois níveis de profundidade:

### 1. Framework conceitual (raiz)

| Arquivo | O que é |
|---|---|
| [framework-conceitual-tracking-whatsapp.md](framework-conceitual-tracking-whatsapp.md) | Documento principal — 5 pilares (identidade, stack técnica, mapa de eventos, pipeline, compliance), cenários A/B (CTWA direto vs LP intermediária), níveis de maturidade N1→N3. **Agnóstico de stack** — vale pra qualquer BSP. |

### 2. Playbook aplicado — Zendesk Sunshine + N8N

O framework conceitual rodando num cliente real (Colina dos Sipes / Grupo Colina). Pasta [`playbook-zendesk-sunshine/`](playbook-zendesk-sunshine/) — começa pelo [README do playbook](playbook-zendesk-sunshine/README.md) que tem arquitetura ASCII, decisões registradas e particularidades do piloto.

| Passo | Arquivo | Status |
|---|---|---|
| 1. Configurar gatilho no Zendesk Sunshine | [01-setup-zendesk-sunshine.md](playbook-zendesk-sunshine/01-setup-zendesk-sunshine.md) | ✅ Documentado |
| 2. Workflow N8N — captura lead + grava na Growthpack | [02-workflow-n8n-capturar-lead.md](playbook-zendesk-sunshine/02-workflow-n8n-capturar-lead.md) | ✅ Documentado |
| 3. Disparar evolução do funil (MQL/SQL/Ganho/Perdido) pro Meta CAPI | [03-disparo-capi-meta.md](playbook-zendesk-sunshine/03-disparo-capi-meta.md) | 🟡 Esqueleto |
| 4. Otimizar campanhas Meta por evento de conversão profundo | [04-otimizar-campanhas.md](playbook-zendesk-sunshine/04-otimizar-campanhas.md) | 🟡 Esqueleto |
| — | Template: [`workflow-capturar-lead.json`](playbook-zendesk-sunshine/templates/workflow-capturar-lead.json) (N8N) | ✅ Pronto pra importar |

## Status

- **v0.1.** Passos 1 e 2 do playbook documentados com instruções executáveis; passos 3 e 4 têm esqueleto conceitual mas ainda não foram fechados. Nada disso rodou em produção end-to-end — o piloto Colina está na largada da migração Lead Form → CTWA.
- **Fora do escopo desta versão:** form prévio antes do WhatsApp · entradas orgânicas/indicação · plataformas além de Meta/Google · BSPs que não sejam Zendesk Sunshine · canal Google Ads (depende de decisão de setup ainda pendente).

## O que falta destrinchar

- Fechar passos 3 e 4 do playbook (payload CAPI com `ctwa_clid` + dedup `event_id` + EMQ ≥ 7,0 como meta de done).
- Decidir setup Google Ads (`wa.me?text=` com `gclid` direto vs página intermediária vs número dedicado) e quantificar o trade-off.
- Validar em campo quais BSPs além do Sunshine expõem `referral.ctwa_clid` no webhook — sem isso, esse repo só serve pra clientes em Sunshine.
- Se o playbook ficar estável depois do piloto, talvez vire skill `tracking-whatsapp-engineer`. Por enquanto, é caderno.

## Convenções

- pt-BR.
- Markdown puro, sem build, sem CI — repositório de estudo, não de software.
- Sem mascarar a realidade: status `🟡` quer dizer "incompleto e eu sei", `✅` quer dizer "documentado", nenhum dos dois quer dizer "rodando em prod".
- Se você caiu aqui sem contexto: leia primeiro o [framework conceitual](framework-conceitual-tracking-whatsapp.md), depois o [README do playbook](playbook-zendesk-sunshine/README.md).
