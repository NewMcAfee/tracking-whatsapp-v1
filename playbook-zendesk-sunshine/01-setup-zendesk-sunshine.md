# Passo 1 — Configurar gatilho no Zendesk Sunshine

**Objetivo:** criar uma Custom Integration no Zendesk Sunshine que envia webhook outbound pro N8N toda vez que um lead chega no WhatsApp (1ª mensagem na conversa OU referral CTWA).

**Tempo estimado:** 15-20 minutos
**Quem executa:** Admin do Zendesk

---

## Pré-requisitos

| Item | Como verificar |
|---|---|
| **Plano Suite Professional+** | Admin Center → Account → Billing → Subscription. Deve aparecer "Suite Professional", "Enterprise" ou "Enterprise Plus" |
| **Sunshine Conversations ativo** | Admin Center → Apps and integrations → APIs → API de conversas (deve estar acessível) |
| **WhatsApp Business conectado via Sunshine** | Admin Center → Channels → Messaging and social → Messaging — deve aparecer o número WhatsApp ativo |
| **Permissão de Admin do Zendesk** | Sem isso, não consegue criar Integration nem API key |

---

## Etapa 1.1 — Criar chave API (App ID + Key ID + Secret)

Necessário pra Sunshine identificar a custom integration que vai receber webhooks.

1. Vai em **Admin Center** (`https://<subdomínio>.zendesk.com/admin/`)
2. Sidebar: **Aplicativos e integrações** → **API de conversas**
3. Clica em **Criar chave da API**
4. Nome: `growth-tracking` (ou nome que identifique o propósito)
5. Salva
6. **Copia as 3 credenciais** que aparecem (ficam visíveis apenas uma vez):
   - **App ID** (formato `app_xxxxxxxxxxxxxxxxxxxxxx`)
   - **Key ID** (formato similar)
   - **Secret** (string longa)
7. Guarda em local seguro (gerenciador de senhas, ou já vai colando no Set do N8N quando montar o workflow)

> **Nota:** essa chave é diferente do "Segredo compartilhado" do webhook (gerado mais à frente). São 2 credenciais distintas.

---

## Etapa 1.2 — Criar Custom Integration

Aqui você define o canal de comunicação Sunshine → N8N.

1. Ainda no **Admin Center**
2. Sidebar: **Aplicativos e integrações** → **Integrações de conversas**
3. Clica em **Criar integração**
4. Tipo: **Personalizado** (Custom)
5. Nome: `growth-tracking-webhook` (ou identificável)
6. Continua/Avança

---

## Etapa 1.3 — Configurar URL e parâmetros do webhook

Tela "Webhook do canal" / "Pontos de extremidade do webhook":

| Campo | Valor |
|---|---|
| **URL do ponto de extremidade do webhook** | URL do webhook N8N — ex: `https://<seu-n8n>/webhook/tracking-whatsapp-sunshine-<cliente>` |
| **Incluir usuário completo** | ✅ **Marcar** (essencial — sem isso, payload vem sem `payload.user`) |
| **Incluir origem completa** | ✅ **Marcar** (essencial — sem isso, payload vem sem `payload.source.client.raw`) |
| **Método de solicitação** | POST (default) |
| **Formato da solicitação** | JSON (default) |

> **Importante:** a URL do N8N deve ser a **Production URL**, não a Test URL. Test URL só funciona enquanto o operador clica "Listen for test event".

---

## Etapa 1.4 — Selecionar triggers (assinaturas de webhooks)

Na seção **Assinaturas de webhooks** da mesma tela:

### ✅ Marcar (essenciais)

| Trigger | Equivalente API | Quando dispara |
|---|---|---|
| **Conversa criada** | `conversation:create` | Lead 100% novo no canal — manda 1ª mensagem |
| **Indicação da conversa** | `conversation:referral` | Lead com conversa existente clica em novo anúncio CTWA |

### ❌ NÃO marcar

- **Mensagem da conversa** (`conversation:message`) — dispara em TODA msg recebida (alto volume desnecessário)
- Cliente removido / adicionado / atualizado
- Conversa apagada / lida
- Postbacks / Digitação / Êxito/Falha de entrega
- Falhas de controle de painel
- Participante entrou/saiu da conversa

> **Por que esses 2 cobrem tudo:**
> - `conversation:create` captura lead novo CTWA (referral já vem no payload)
> - `conversation:referral` captura lead recorrente CTWA (nova campanha em conversa existente)
> - Juntos cobrem 100% dos cenários CTWA sem volume excessivo

---

## Etapa 1.5 — Capturar Webhook ID e Segredo

Após salvar a integração, o Zendesk gera 2 strings:

| Item | Pra que serve | Onde guardar |
|---|---|---|
| **ID do webhook** | Identifica o webhook nos logs/audit | Anota só pra referência |
| **Segredo compartilhado** | Validação HMAC (opcional) | **Salvar com segurança** — só aparece UMA vez |

> **Atenção visual:** o input field do "Segredo compartilhado" pode parecer truncado na tela. Clica dentro do campo, faz **Ctrl+A → Ctrl+C** pra garantir que copia o valor completo (~85 chars).

> **Sobre validação HMAC:** o framework atual do Colina **não valida** (decisão consciente — URL privada já é proteção razoável). Se quiser validar em produção, o Sunshine envia o segredo no header `x-api-key` (compara direto com o segredo guardado).

---

## Etapa 1.6 — Concluir e ativar

1. Clica em **Avançar** / **Concluir**
2. A integração aparece em **Integrações de conversas** com status **Ativa**
3. O Sunshine **começa a enviar webhooks imediatamente** pra URL configurada quando os triggers dispararem

---

## Etapa 1.7 — Validação

Antes de declarar o passo 1 concluído:

1. **Workflow N8N receptor deve estar Active** (toggle no canto superior direito)
2. Mandar **1 mensagem de teste** no WhatsApp Comercial Web (de qualquer número novo)
3. Esperar 5-10 segundos
4. Conferir em **N8N → Executions** se apareceu execução nova
5. Abrir a execução e ver no nó Webhook se `body.events[0].type === "conversation:create"`

Se chegou e tem o `body` populado: **Passo 1 fechado.**

---

## Anatomia do payload recebido

Exemplo de payload de **lead orgânico** (sem CTWA):

```json
{
  "body": {
    "app": { "id": "<app_id>" },
    "webhook": { "id": "<webhook_id>", "version": "v2" },
    "events": [
      {
        "id": "<event_id>",
        "createdAt": "2026-05-21T04:23:57.258Z",
        "type": "conversation:create",
        "payload": {
          "conversation": {
            "id": "<conversation_id>",
            "type": "personal",
            "brandId": "<brand_id>",
            "activeSwitchboardIntegration": { "name": "ultimate", ... }
          },
          "user": {
            "id": "<sunshine_user_id>",
            "profile": {
              "givenName": "Nome",
              "surname": "Sobrenome"
            },
            "signedUpAt": "...",
            "identities": []
          },
          "creationReason": "message",
          "source": {
            "type": "whatsapp",
            "integrationId": "<integration_id>",
            "client": {
              "externalId": "55XXXXXXXXXXX",      ← phone E.164 sem +
              "displayName": "+55 XX XXXXX-XXXX",
              "raw": {
                "profile": { "name": "Nome" },
                "from": "55XXXXXXXXXXX"
              }
            }
          }
        }
      }
    ]
  }
}
```

**Quando for lead CTWA**, dentro de `client.raw` aparece o objeto `referral` adicional:

```json
"raw": {
  "profile": { "name": "..." },
  "from": "...",
  "referral": {
    "ctwa_clid": "ARxxxxxxxx...",      ← Click ID Meta
    "source_id": "120210xxxxxx",       ← Ad ID
    "source_type": "ad",
    "source_url": "https://fb.me/...",
    "headline": "...",
    "body": "...",
    "media_type": "image",
    "image_url": "..."
  }
}
```

---

## Troubleshooting comum

| Sintoma | Causa provável | Solução |
|---|---|---|
| Execução N8N falha com "No Respond to Webhook node found" | Webhook configurado pra usar Respond Node mas o node foi deletado | No Webhook N8N, troca **Respond** pra `Immediately` |
| Body chega vazio (`{}`) | `Raw Body` ligado no Webhook N8N | Desliga `Raw Body` (a não ser que vá validar HMAC) |
| Webhook não chega no N8N | Workflow N8N inativo OU URL errada no Zendesk | Conferir toggle Active no N8N + URL correta (production, não test) |
| Não tem campo `client.raw.referral` mesmo em CTWA | Número de teste já tinha conversado com o canal antes do anúncio | Testar com celular/número 100% novo (que nunca falou com o canal) |
| Falha "Filter parameter identities.email is required" ao tentar query API | Confundiu webhook inbound com Conversations API (`listUsers` não existe na v2) | Não fazer query — usar webhook outbound (esse passo 1) |

---

## Próximo

→ [02-workflow-n8n-capturar-lead.md](02-workflow-n8n-capturar-lead.md)
