# Guia rápido — n8n + Chatwoot (MVP Saudação)

> Para quem está travado no Code node. Este guia usa **zero JavaScript**.

## Arquitetura (o que você já tem)

```
WhatsApp (Meta) → Chatwoot → webhook → n8n → API Chatwoot → WhatsApp
```

Você **não precisa** falar com a Meta API no n8n. O Chatwoot já faz isso.
O n8n só precisa:

1. Receber o webhook quando chega mensagem (`message_created`)
2. Responder pela API do Chatwoot

## Opção 1 — Importar workflow pronto (recomendado)

1. No n8n: **Workflows → Import from File**
2. Arquivo: `workflows/sprint-1b-mvp-saudacao.json`
3. Abra o nó **Responde no Chatwoot** e vincule a credencial **Chatwoot API**
4. Ative o workflow
5. Copie a URL de produção do Webhook (`https://n8n.recifeflatstemporada.com/webhook/chatwoot-mvp`)

### Configurar webhook no Chatwoot

1. Chatwoot → **Settings → Integrations → Webhooks**
2. Add webhook
3. URL: a URL de produção do n8n
4. Evento: **message_created**
5. Salvar

## Opção 2 — Montar manualmente (sem Code node)

| # | Nó | Função |
|---|-----|--------|
| 1 | Webhook | POST `/chatwoot-mvp` |
| 2 | IF | `body.event` = `message_created` AND `body.message_type` = `incoming` |
| 3 | Edit Fields | Extrai phone, message, conversation_id, account_id, first_name |
| 4 | Edit Fields | Monta texto da saudação |
| 5 | HTTP Request | POST mensagem na API do Chatwoot |

### Expressões do nó "Extrai dados" (Edit Fields)

| Campo | Expressão |
|-------|-----------|
| phone_raw | `{{ $json.body.sender.phone_number }}` |
| message | `{{ $json.body.content }}` |
| conversation_id | `{{ $json.body.conversation.id }}` |
| account_id | `{{ $json.body.account.id }}` |
| sender_name | `{{ $json.body.sender.name }}` |
| first_name | `{{ $json.body.sender.name.split(' ')[0] }}` |

### HTTP Request — Responde no Chatwoot

- **Method:** POST
- **URL:** `https://chat.recifeflatstemporada.com/api/v1/accounts/{{ $json.account_id }}/conversations/{{ $json.conversation_id }}/messages`
- **Auth:** Header `api_access_token` = seu token do Chatwoot
- **Body (JSON):**

```json
{
  "content": "{{ $json.resposta }}",
  "message_type": "outgoing",
  "private": false
}
```

## Por que o Code node falhou

No Code node v2, o modo **Run Once for Each Item** exige:

```javascript
return { json: dados };  // objeto, SEM array
```

Se você usa `return [{ json: dados }]`, o nó mostra sucesso com **0 itens** e o fluxo para.

**Solução:** não use Code node para extração simples. Use Edit Fields com expressões.

## Evitar loop infinito

O nó IF filtra só `message_type = incoming`. Sem isso, quando o bot responde,
o Chatwoot dispara outro webhook e o bot responde de novo para sempre.

## Próximo passo — buscar dados no Supabase

Depois que a saudação funcionar, adicione um nó **HTTP Request** entre
"Extrai dados" e "Monta resposta":

- **GET** `https://SEU_PROJETO.supabase.co/rest/v1/hospedes?telefone=eq.{{ $json.phone_raw }}`
- Headers: `apikey` e `Authorization: Bearer SEU_TOKEN`

Use o resultado para personalizar a mensagem (nome da reserva, apartamento, etc.).

## Templates oficiais n8n

- [Multichannel AI Assistant + Chatwoot](https://n8n.io/workflows/8260) — base para IA
- Busque em n8n.io/workflows por "Chatwoot"

## Alternativas se n8n continuar difícil

| Opção | Quando usar |
|-------|-------------|
| **Macros do Chatwoot** | Respostas fixas, sem código |
| **Captain AI (Chatwoot)** | FAQ automatizado nativo |
| **Make.com / Zapier** | Mais fácil, mas pago |
| **Supabase Edge Function** | Se preferir código TypeScript |

Para o seu caso (saudação + dados do hóspede), n8n + Edit Fields é o caminho
mais simples e gratuito.
