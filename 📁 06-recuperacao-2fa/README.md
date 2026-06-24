
# 🔐 Recuperação de acesso 2FA — investigação técnica e resolução

> **Status:** ✅ Resolvido (token permanente "Nunca expira" gerado)
> **Duração da investigação:** 3 dias / ~15h de trabalho técnico
> **Caminhos investigados:** 3 (API REST, webhook receiver customizado, suporte oficial)
> **Caminho que resolveu:** Suporte oficial da Meta (com insight do bot de IA)

---

## 📋 Resumo executivo

Para integração estável com a Meta WhatsApp Cloud API, é necessário gerar um
**System User Token permanente** ("Never expires"). O token padrão expira a
cada 24h, o que tornaria a operação insustentável em produção (renovação
manual diária ou pipeline de refresh automático).

A geração do token permanente exige verificação 2FA adicional via **WhatsApp/SMS
para um número cadastrado no perfil pessoal da Meta**. No meu caso, o número
cadastrado era antigo — chip cancelado há tempos, sem acesso possível. O 2FA
do login (via app authenticator) funcionava normalmente, mas o fluxo específico
de geração de tokens permanentes **não oferecia método alternativo de
verificação** — só a opção do número descontinuado.

Três caminhos técnicos foram investigados durante 3 dias. A resolução veio
do **bot de IA do suporte oficial da Meta**, que identificou uma característica
de UX do sistema desconhecida publicamente: **remover o número de telefone do
perfil pessoal libera o fallback automático para o email cadastrado.**

Este documento descreve as três investigações e o aprendizado central.

---

## 🎯 O contexto

Após o sistema Chatwoot estar funcionando bidirecionalmente com a Meta Cloud
API (ver [05-bug-9-movel/](../05-bug-9-movel/)), o último passo era estabilizar
a integração com **token permanente** ao invés do token de 24h.

### O bloqueio

```
business.facebook.com → Usuários do sistema → Chatwoot Recife Flats
                                          ↓
                                  "Gerar novo token"
                                          ↓
                          "Para sua segurança, vamos verificar
                           sua conta pelo WhatsApp"
                                          ↓
                          "Código enviado para +55 81 *********78"
                                          ↓
                              [número descontinuado]
                                          ↓
                                  🚫 IMPOSSÍVEL
```

A tela de verificação **não oferecia método alternativo** — nem email, nem
app authenticator, nem nada. Apenas o número antigo.

### Observação importante

```
🟢 Login normal no Facebook    → funciona (2FA via app authenticator)
🟢 Login no Business Manager    → funciona
🟢 Operações administrativas    → funcionam
🔴 Geração de token permanente  → bloqueada (verificação extra independente)
```

A verificação para gerar tokens permanentes é uma **camada extra de segurança**
que opera independentemente do 2FA padrão de login. Ela consulta o número do
perfil diretamente, sem oferecer o método configurado no 2FA principal.

---

## 🔬 Tentativa 1 — API REST da plataforma legada

**Hipótese:** a plataforma SaaS antiga (que ainda operava como fallback)
recebia as mensagens da Meta no número antigo (via WhatsApp Business API),
mas exibia "conteúdo não suportado" no painel da interface. Se o conteúdo
do webhook estivesse armazenado no banco, a API REST poderia retorná-lo.

### Investigação

```bash
# Descoberta do organizationId via endpoint /me
curl -H "Authorization: Bearer $TOKEN_PLATAFORMA" \
     "https://api-da-plataforma.com/v1/me"

# Consulta da mensagem armazenada
curl -H "Authorization: Bearer $TOKEN_PLATAFORMA" \
     "https://api-da-plataforma.com/v1/messages/{messageId}/?organizationId={orgId}"
```

### Resultado

A API retornava o objeto, mas com **campos vazios**:

```json
{
  "id": "ajgrDqIJPGQskeP9",
  "messageType": "Unsupported",
  "content": null,
  "file": null,
  "thumbnail": null,
  "contacts": [],
  "buttons": [],
  "eventAtUTC": "2026-06-21T18:18:52Z"
}
```

**Diagnóstico:** a plataforma legada descarta o conteúdo de mensagens de tipo
"Unsupported" antes de persistir. Apenas metadados (ID, timestamp, tipo) são
mantidos. O payload bruto da Meta nunca foi armazenado.

**Conclusão:** caminho inviável. A informação não existe mais nesse sistema.

---

## 🔬 Tentativa 2 — Webhook receiver customizado

**Hipótese:** se a plataforma legada descarta o conteúdo ao armazenar, então
o payload bruto ainda passa pelo webhook **antes** de ser descartado. Se eu
interceptar o payload no webhook (não no armazenamento), eu pego o conteúdo
original.

### Implementação

Construído um receptor HTTP em Python (`http.server` da stdlib) hospedado no
mesmo VPS do Chatwoot:

- **Porta:** 5555 (escuta local)
- **Captura:** payload bruto de qualquer POST recebido
- **Persistência:** headers + body salvos em arquivo de log estruturado
- **Detector regex:** identifica códigos de 6 dígitos automaticamente
  (`\b\d{6}\b`)
- **Resposta ao caller:** `200 OK` com body `{"status":"received"}`

```python
# Trecho do receptor (simplificado)
class WebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(content_length).decode('utf-8')

        # Persistir tudo
        with open(LOG_FILE, 'a', encoding='utf-8') as f:
            f.write(f"BODY:\n{body}\n{'='*80}\n")

        # Detectar códigos de 6 dígitos
        codigos = re.findall(r'\b\d{6}\b', body)
        if codigos:
            print(f"🚨 CÓDIGOS DETECTADOS: {codigos}", flush=True)

        self.send_response(200)
        self.send_header('Content-Type', 'application/json')
        self.end_headers()
        self.wfile.write(b'{"status":"received"}')
```

### Exposição via HTTPS

Exposição pública via **ngrok** (tunnel HTTPS gratuito), apontando para a
porta 5555 local. Webhook configurado na plataforma legada com evento
"Mensagem recebida" ativado, URL apontando para o tunnel.

### Validação

Mensagens reais do WhatsApp foram capturadas com sucesso — o pipeline
end-to-end estava funcionando:

```
================================================================================
TIMESTAMP: 2026-06-22T20:03:30.608992
PATH: /umbler
BODY:
{
  "Type": "Message",
  "Payload": {
    "Content": "Oii",
    "MessageType": "Text",
    ...
  }
}
================================================================================
```

O webhook receiver foi validado bidirecionalmente e estava pronto para capturar
qualquer código de 6 dígitos que a Meta enviasse.

### Por que essa tentativa também não resolveu (sozinha)

O webhook ficou pronto, mas naquele dia eu já tinha solicitado o código várias
vezes ao Meta — atingi o rate limit, e a Meta bloqueou novos envios por
algumas horas. O plano era voltar no dia seguinte com o pipeline pronto e
solicitar o código uma única vez para capturá-lo.

**Mas no dia seguinte, decidi pivotar para o caminho do suporte (Tentativa 3)
em vez de insistir tecnicamente.** O webhook receiver foi desligado sem ter
sido necessário para o uso final.

> 💡 **Esse trabalho não foi desperdiçado.** Funcionou como diagnóstico
> validador (a hipótese técnica estava correta) e como exercício de pipeline
> webhook end-to-end. Mas se eu fosse refazer o projeto, teria ido direto
> para a Tentativa 3 muito antes.

---

## ✅ Tentativa 3 — Suporte oficial da Meta (a que resolveu)

Depois de 3 dias batendo cabeça com soluções técnicas, decisão estratégica:
**parar de tentar resolver tecnicamente e usar o canal oficial.**

### O ticket

Mensagem escrita de forma objetiva e estruturada (contexto, problema, dados
da conta para identificação, pedido específico), enviada via formulário do
Meta Business Help Center.

**O respondente foi um bot de IA da Meta** — não um humano. Ele leu o ticket,
analisou o histórico da conta, e me deu uma instrução que **não está
documentada publicamente em lugar nenhum**:

> *"O número de telefone está cadastrado em mais de um lugar na sua conta.
> Para que outras opções de verificação apareçam quando você gerar tokens,
> você precisa remover o número do **perfil pessoal** na Central de Contas
> (não no 2FA — lá ele não está, você usa app authenticator). O sistema
> consulta o número do perfil em primeiro lugar; sem ele, as opções
> alternativas (email cadastrado) aparecem automaticamente."*

### Execução

```
1. Central de Contas → Perfis e dados pessoais → Vanessa Rafaella
2. Telefone → Remover número antigo (+55 81 *********78)
3. Salvar (não pede confirmação porque é remoção)
4. Voltar para Business Manager → Gerar token
5. Tela de verificação aparece COM nova opção: "Enviar para email"
6. Selecionar email
7. Receber código no email (que sempre esteve cadastrado, mas nunca tinha
   sido oferecido)
8. Inserir código
9. Configurar token: "Never expires" (Nunca expira)
10. Token gerado ✅
```

**Tempo total da Tentativa 3, do ticket à conclusão:** ~30 minutos.

---

## 🔑 Insight central — UX layered de verificação na Meta

O que aprendi de fato durante todo o processo:

```
┌──────────────────────────────────────────────────────────┐
│                  META — VERIFICAÇÃO POR CAMADA           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Camada 1 — LOGIN normal                                 │
│    Métodos disponíveis: senha + 2FA (app authenticator)  │
│    Status: ✅ funcionava                                  │
│                                                          │
│  Camada 2 — Operações administrativas no Business        │
│    Métodos: sessão ativa do Camada 1                     │
│    Status: ✅ funcionava                                  │
│                                                          │
│  Camada 3 — Geração de TOKEN PERMANENTE                  │
│    Verificação EXTRA, INDEPENDENTE das camadas 1 e 2     │
│    Consulta: número do PERFIL primeiro                   │
│    Fallback: email (se número do perfil não existir)     │
│    Status: 🔴 bloqueava                                   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

A Meta tem **camadas independentes de verificação** para operações de risco
crescente. Cada camada consulta dados diferentes da conta. Saber em qual
camada o bloqueio está acontecendo é metade do diagnóstico.

---

## 🧠 Aprendizados

### Técnicos

1. **Webhooks expõem o payload original; APIs REST expõem o estado armazenado.**
   Quando um sistema descarta dados na ingestão, o webhook é o único ponto
   onde a informação completa existe.

2. **Detector regex em pipeline customizado vale a pena para padrões previsíveis.**
   Códigos de 6 dígitos têm formato muito específico — `\b\d{6}\b` é simples,
   rápido, e funciona em qualquer payload, sem precisar parsear estrutura.

3. **Variáveis de ambiente via `read -s` para credenciais.** Boa prática
   básica de DevSecOps. O token nunca fica visível no terminal nem em
   histórico do bash.

### Estratégicos

4. **Sistemas têm UX em camadas independentes.** O 2FA principal funcionar não
   significa que outras verificações vão funcionar. Cada camada de risco pode
   ter regras próprias e fallbacks próprios.

5. **A documentação oficial não cobre todos os edge cases.** Características
   de UX como "remova o número do perfil para liberar o email como fallback"
   só são conhecidas internamente — nem o suporte humano sabe disso
   imediatamente, apenas o sistema (e a IA dele) conhece.

6. **Nem todo problema técnico tem solução técnica.** A Tentativa 2 (webhook
   receiver) foi tecnicamente correta, executada com sucesso, e teria
   provavelmente funcionado para capturar o código. Mas o caminho mais
   rápido para o objetivo final era um ticket bem escrito. Saber a hora
   de pivotar é tão importante quanto saber implementar.

7. **Bots de suporte às vezes sabem mais que humanos.** A IA da Meta tem
   acesso ao comportamento real do sistema, não a um runbook humano. Para
   problemas que envolvem regras internas do sistema, é frequentemente
   melhor que o suporte humano.

---

## 📊 O que cada tentativa custou

| Tentativa | Tempo investido | Resultado | Aproveitável |
|-----------|------------------|-----------|--------------|
| 1 — API REST | ~2h | ❌ Caminho fechado | Conhecimento da API da plataforma legada |
| 2 — Webhook receiver | ~6h | ⏸️ Funcionou mas não foi usado | Pipeline reutilizável + skill de pipeline customizado |
| 3 — Suporte Meta | ~30min | ✅ Resolveu | Token permanente + insight do sistema |

**Total:** ~8h+ de trabalho técnico. **Em retrospecto, a Tentativa 3 sozinha
teria resolvido em ~30 minutos.** Lição valiosa para próximos projetos:
**sempre que houver um canal oficial de suporte, vale testá-lo em paralelo
com investigação técnica — não depois.**

---

## 🔐 Sobre o token gerado

- **Tipo:** System User Token
- **Expiração:** "Never expires" (permanente)
- **Permissões:** `whatsapp_business_messaging` + `whatsapp_business_management`
- **Armazenamento:** salvo em gerenciador de credenciais; configurado no
  Chatwoot via interface web (campo "API Key" da inbox WhatsApp)
- **Validação:** teste bidirecional (envio + recepção) confirmado após
  atualização do token

---

[← Voltar ao README principal](../README.md)
