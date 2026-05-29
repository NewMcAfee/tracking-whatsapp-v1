# Passo 4 — Otimizar campanhas Meta por evento de conversão profundo

**Objetivo:** uma vez que os eventos do funil chegam no Events Manager (Trilha A ou B), configurar a campanha CTWA pra otimizar pelo evento certo e ler o resultado.

**Status:** 🟡 TODO — esqueleto documentado, a detalhar com dados de campanha real
**Trilha:** vale pras duas (A e B)

---

## Princípio

A captura de identidade (Passos 1-3) só vira valor quando a campanha **otimiza pelo evento profundo**. Sem isso, você tem dados bonitos no Events Manager e a campanha continua otimizando por "início de conversa".

---

## Esqueleto do que detalhar

### 4.1 — Escolha do evento de otimização

| Cenário do cliente | Evento primário | Por quê |
|---|---|---|
| E-commerce / serviço transacional | `Purchase` | Ciclo curto, volume de compra suficiente |
| B2B ciclo ≤ 7 dias | `Lead` (estágio Qualificado) → migrar p/ `SQL` | Casa com a janela de atribuição Meta |
| B2B ciclo > 7 dias | `SQL` / `Qualificado` (evento intermediário) | `Purchase` cai fora da janela 7d-click (pós-jan/2026) |

> A janela de atribuição Meta encolheu pra **7-day click / 1-day view** em 2026 (framework Cap. 6.1). Ciclos longos devem otimizar por um evento que **acontece dentro de 7 dias**, não pela venda final.

### 4.2 — Volume mínimo pra sair do aprendizado

- Meta precisa de **~50 conversões do evento otimizado por semana por ad set** pra sair da fase de aprendizado
- Se `Purchase` não bate isso → otimizar por evento mais ao topo (Lead qualificado) até ter volume
- **A detalhar:** cálculo de volume esperado por verba do cliente

### 4.3 — Leitura de resultado

- **A detalhar:** dashboard de match rate (% eventos com `ctwa_clid`), CPL por evento profundo, CAC real (Close-Won / verba)
- Comparar CPL "início de conversa" vs. CPL "lead qualificado" — a métrica que importa muda

### 4.4 — Exclusion audiences (depende da Trilha B)

- `DealLost` + `NOICP` → custom audience de exclusão → não gastar verba em perfis que já não converteram
- Requer os eventos custom da Trilha B

---

## Pendências pra fechar este passo

1. Rodar uma campanha CTWA real e coletar os primeiros dados de conversão
2. Definir o dashboard de leitura (Meta nativo vs. planilha vs. BI)
3. Documentar o playbook de ajuste: quando trocar o evento de otimização, quando subir/baixar verba
4. Validar match rate real (% de leads com `ctwa_clid` capturado) — meta ≥ 80% (framework Cap. 6.4)

---

## Referências

- [Framework conceitual — Cap. 4 (Mapa de Eventos) e Cap. 6 (Atribuição/Janelas)](../framework-conceitual-tracking-whatsapp.md)
- [02-capi-nativo-kommo.md](02-capi-nativo-kommo.md) — origem dos eventos (Trilha A)
- [03-webhook-n8n-capi.md](03-webhook-n8n-capi.md) — origem dos eventos (Trilha B)
</content>
