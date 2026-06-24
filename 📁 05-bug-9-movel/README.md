
# 🐛 Bug do "9" móvel brasileiro no Chatwoot v4.13.0

> **Status:** ✅ Resolvido (workaround aplicado)
> **Severidade:** Crítica — bloqueava recepção de TODAS as mensagens WhatsApp
> **Versão afetada:** Chatwoot v4.13.0 (provavelmente versões anteriores também)
> **Plataforma:** WhatsApp via Meta Cloud API
> **Região afetada:** Brasil (todos os números móveis brasileiros)

---

## 📋 Resumo executivo

Após configurar uma inbox do Chatwoot conectada à Meta WhatsApp Cloud API com
um número brasileiro, **nenhuma mensagem chegava ao inbox** — apesar da Meta
confirmar entrega via webhook (status `200 OK` no log). O diagnóstico revelou
uma **inconsistência de formatação** entre como a Meta envia o
`display_phone_number` (sem o "9" móvel brasileiro) e como o Chatwoot grava o
canal no banco (com o "9"). O webhook job não encontrava o canal correspondente
e descartava silenciosamente todas as mensagens recebidas.

A correção foi aplicada via Rails console, atualizando o `phone_number` do
canal no banco para o formato esperado pela Meta.

---

## 🎯 Contexto

Durante a configuração inicial da inbox WhatsApp no Chatwoot v4.13.0
self-hosted, todos os passos foram seguidos conforme a documentação oficial:

1. Criação do App Meta for Developers e configuração da WhatsApp Business
   Account (WABA)
2. Provisionamento de número novo brasileiro no Meta Cloud API
3. Configuração da inbox no Chatwoot com:
   - Provider: WhatsApp Cloud
   - Phone Number: `+55 81 9 8422-5358` (formato canônico brasileiro com "9")
   - Phone Number ID: provisionado pela Meta
   - Business Account ID: WABA
   - Token de API: válido (24h, mas suficiente para o teste)
4. Webhook URL configurado na Meta apontando para o endpoint correto
5. Subscriptions ativadas (`messages` + `message_template_status_update`)

Tudo aparentemente correto. Teste de envio (Chatwoot → WhatsApp) funcionava.
Mas o caminho de recepção (WhatsApp → Chatwoot) **não funcionava**.

---

## 🔍 Sintoma observado

### O que se esperava

```
Cliente envia "Oi" via WhatsApp
        ↓
Meta entrega webhook ao Chatwoot
        ↓
Mensagem aparece no inbox "WhatsApp - Recife Flats"
```

### O que acontecia

```
Cliente envia "Oi" via WhatsApp
        ↓
Meta entrega webhook ao Chatwoot
        ↓
Chatwoot responde 200 OK (recebeu)
        ↓
[silêncio — mensagem NÃO aparecia no inbox]
```

Nenhum erro visível na interface. Nenhuma notificação. A mensagem simplesmente
não existia para o sistema.

---

## 🧪 Diagnóstico passo a passo

### Passo 1: Confirmar que o webhook estava chegando

A primeira suspeita foi que o webhook não estaria chegando ao servidor.
Inspeção dos logs do container Rails:

```bash
docker compose logs rails --tail=200 | grep -i whatsapp
```

Resultado: webhooks estavam chegando, e o status de resposta era `200 OK`.
**O webhook não era o problema.**

### Passo 2: Investigar logs do Sidekiq (jobs em background)

A próxima suspeita: talvez o job de processar a mensagem estivesse falhando
silenciosamente.

```bash
docker compose logs sidekiq --tail=200 | grep -i whatsapp
```

Resultado: foi encontrada a seguinte linha repetida:

```
Inactive WhatsApp channel: unknown - +5581984225358
```

**Pista crítica:** o sistema dizia que o canal estava "inactive" ou "unknown",
mesmo o canal existindo no banco e estando configurado.

### Passo 3: Inspecionar o canal pelo Rails console

```bash
docker compose exec rails bundle exec rails console
```

```ruby
inbox = Inbox.find_by(name: 'WhatsApp - Recife Flats')
puts inbox.channel.phone_number
# => "+5581984225358"

puts inbox.channel.provider_config['phone_number_id']
# => "1094911290380821"
```

O canal **existia** com o `phone_number` `+5581984225358`. Tudo certinho do
ponto de vista da configuração feita.

### Passo 4: Investigar o payload bruto da Meta

Voltando aos logs do Rails para ver o body do webhook que chegava:

```bash
docker compose logs rails --tail=500 | grep -A 20 "display_phone_number"
```

Trecho do payload (anonimizado):

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "1755135172530822",
    "changes": [{
      "value": {
        "metadata": {
          "display_phone_number": "558184225358",
          "phone_number_id": "1094911290380821"
        }
      }
    }]
  }]
}
```

**🎯 Bingo.** A Meta enviava o número **sem o dígito "9" móvel**.

### Passo 5: Encontrar o código que faz o match

Localização do código fonte do Chatwoot que processa o evento:
`app/jobs/webhooks/whatsapp_events_job.rb`

Trecho relevante:

```ruby
def get_channel_from_wb_payload(wb_params)
  phone_number = "+#{wb_params[:entry].first[:changes].first.dig(:value, :metadata, :display_phone_number)}"
  channel = Channel::Whatsapp.find_by(phone_number: phone_number)
  return channel if channel.present?

  # ... [outros fallbacks também usam phone_number como chave de busca]
end
```

O método monta o `phone_number` adicionando `+` na frente do `display_phone_number`
recebido da Meta — resultado: `"+558184225358"`. Em seguida, busca no banco
por `Channel::Whatsapp.find_by(phone_number: "+558184225358")`.

**Mas no banco o canal está gravado como `"+5581984225358"`** (com "9").

O método retorna `nil`. O job interpreta como "canal inativo/desconhecido"
e descarta a mensagem.

---

## 🇧🇷 Por que isso só acontece com números brasileiros?

O Brasil é um dos poucos países que adicionou um dígito extra (o "9") a
**todos os números móveis** em uma reforma do plano de numeração (2010-2016
gradualmente em cada estado).

- **Formato canônico brasileiro:** `+55 [DDD] 9 [8 dígitos]` → 13 dígitos
  (ex.: `+5581984225358`)
- **Formato E.164 internacional (sem 9):** `+55 [DDD] [8 dígitos]` → 12 dígitos
  (ex.: `+558184225358`)

A Meta usa o formato **sem o "9"** no campo `display_phone_number` (
provavelmente seguindo padrão E.164 internacional), mas o Chatwoot, ao
provisionar a inbox, espera/grava com o "9" (formato local brasileiro). O
mismatch é silencioso porque ambos os formatos são "válidos" — apenas não
batem na busca exata.

Países cujos números nunca tiveram dígito adicional inserido (a maioria do
mundo) não enfrentam esse problema.

---

## 🔧 Solução aplicada

A correção foi atualizar o `phone_number` do canal no banco para o formato
**sem o "9" móvel**, fazendo match exato com o payload da Meta.

### Via Rails console (workaround)

```bash
docker compose exec rails bundle exec rails runner "
  inbox = Inbox.find_by(name: 'WhatsApp - Recife Flats')
  ch = inbox.channel
  puts \"Antes: #{ch.phone_number}\"
  ch.update!(phone_number: '+558184225358')
  puts \"Depois: #{ch.phone_number}\"
"
```

**Resultado:**
```
Antes: +5581984225358
Depois: +558184225358
```

### Validação imediata

Após a alteração, um teste de mensagem recebida via WhatsApp passou a aparecer
no inbox em segundos. Bidirecional (envio + recepção) confirmado.

---

## 🧭 Reproduzibilidade

Para reproduzir esse bug em um ambiente Chatwoot v4.13.0 self-hosted:

1. Provisione um número WhatsApp Business brasileiro na Meta Cloud API
2. Configure a inbox no Chatwoot inserindo o número com o "9" móvel:
   `+55 [DDD] 9 [8 dígitos]`
3. Configure o webhook na Meta apontando para a URL do Chatwoot
4. Envie uma mensagem de teste do seu celular pessoal para o número configurado
5. Observe os logs do Sidekiq:
   `docker compose logs sidekiq | grep -i whatsapp`
6. O log mostrará `Inactive WhatsApp channel: unknown - +55XXXXXXXXXXX`
7. A mensagem **não** aparecerá no inbox

---

## ⚠️ Implicações para a comunidade

Esse bug provavelmente afeta **todos os usuários do Chatwoot self-hosted no
Brasil** que usam Meta Cloud API como provider — uma população não-trivial.

### Soluções de longo prazo possíveis (sugestões à comunidade)

1. **Normalização no momento do `find_by`:** o método `get_channel_from_wb_payload`
   poderia tentar variações do número (com/sem "9" para números brasileiros)
   antes de retornar `nil`.

2. **Normalização ao gravar:** o Chatwoot poderia normalizar o `phone_number`
   ao formato E.164 estrito (sem o "9" para números brasileiros) ao provisionar
   a inbox, evitando o mismatch desde o início.

3. **Validação na configuração da inbox:** alertar o usuário no momento da
   configuração se o número parece estar no formato local em vez de E.164
   internacional.

Pretendo abrir issue/PR no repositório oficial do Chatwoot relatando o bug e
sugerindo correção. Até lá, o workaround documentado aqui resolve o problema.

---

## 🧠 Aprendizados técnicos

1. **Webhook respondendo `200 OK` não significa que a mensagem foi processada.**
   O ACK é apenas "recebi o payload" — o que acontece depois é responsabilidade
   da lógica de negócio. Sempre verifique também os logs dos jobs em background.

2. **Padrões de formato de dados são contratos implícitos.** A Meta segue um
   padrão E.164 estrito; o Chatwoot tem flexibilidade na entrada. Ambos
   funcionam isoladamente, mas a interação revela o mismatch.

3. **Ler o source code é parte do trabalho.** A documentação oficial não
   menciona esse caso. O bug só foi diagnosticado por leitura direta do
   `whatsapp_events_job.rb` e correlação com o payload recebido.

4. **Logs detalhados salvam horas de diagnóstico.** A linha
   `Inactive WhatsApp channel: unknown - +55XXXXXXXXXXX` foi o que apontou
   o caminho. Sem isso, eu poderia ter perdido muito mais tempo suspeitando
   de problemas de rede, token, webhook, etc.

5. **Reprodução manual via Rails console é diagnóstico de alto nível.**
   Conseguir abrir o Rails console e simular a query exata que o job faz
   (`Channel::Whatsapp.find_by(phone_number: ...)`) confirmou a hipótese em
   segundos.

---

## 📚 Referências

- [Chatwoot source — whatsapp_events_job.rb](https://github.com/chatwoot/chatwoot/blob/master/app/jobs/webhooks/whatsapp_events_job.rb)
- [Documentação oficial Meta — Webhook payload format](https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks/payload-examples)
- [Padrão E.164 — ITU-T Recommendation](https://en.wikipedia.org/wiki/E.164)
- [Reforma do plano de numeração móvel no Brasil — ANATEL](https://www.gov.br/anatel/pt-br)

---

[← Voltar ao README principal](../../README.md)
