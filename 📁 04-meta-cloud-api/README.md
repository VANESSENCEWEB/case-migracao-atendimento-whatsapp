# 📡 Fase 04 — Integração Meta WhatsApp Cloud API

> **Status:** ✅ Concluída
> **Output:** Inbox WhatsApp no Chatwoot conectado à Meta Cloud API,
> recebendo e enviando mensagens bidirecionalmente com token permanente.

---

## 🎯 Objetivo da fase

Conectar o Chatwoot (já instalado na Fase 03) à API oficial do WhatsApp da
Meta, permitindo:

- Receber mensagens de hóspedes via WhatsApp dentro do Chatwoot
- Enviar respostas pelo Chatwoot que chegam no WhatsApp do cliente
- Manter um histórico unificado das conversas
- Usar o número novo como teste antes de migrar o número antigo da
  plataforma legada

---

## 🌐 Por que Meta Cloud API direta (e não via BSP)?

A Meta oferece **dois caminhos** para integração com WhatsApp Business:

1. **Cloud API direta** — você lida diretamente com a Meta
2. **BSP (Business Solution Provider)** — empresas intermediárias
   (360dialog, Gupshup, MessageBird, etc.) que abstraem a API

### Comparação

| Critério | Cloud API direta | BSP |
|----------|------------------|-----|
| Custo | Gratuito até cotas generosas | R$300-500+/mês |
| Setup | Mais técnico | Mais simples |
| Controle | Total | Limitado pelo BSP |
| Suporte | Documentação Meta | Suporte do BSP |
| Lock-in | Nenhum | Médio |

**Decisão:** Cloud API direta. O ganho de custo + autonomia compensa
amplamente o setup um pouco mais técnico.

---

## 🏢 Estrutura da conta Meta

A configuração na Meta envolve **vários objetos relacionados** que confundem
quem está começando. A estrutura final:

```
👤 Conta pessoal Facebook (Vanessa Rafaella)
   │
   ▼
🏢 Meta Business Account / Portfolio empresarial
   "Apartamento de Temporada"
   │
   ├─── 📱 App Meta for Developers (chatrecife)
   │         App ID: 1699467098052799
   │         Função: representa o software cliente que vai
   │         consumir a API
   │
   ├─── 💬 WhatsApp Business Account (WABA)
   │         WABA ID: 1755135172530822
   │         Função: a "conta WhatsApp" da empresa, contém
   │         os números de telefone
   │         │
   │         └─── 📞 Phone Number
   │                  Phone Number ID: 1094911290380821
   │                  Display: +55 81 9 8422-5358
   │
   └─── 👥 System Users
            "Chatwoot Recife Flats"
            Função: usuário não-humano que executa requisições à API
            Tokens: gerados em nome desse System User
```

### Por que essa complexidade?

Cada objeto existe por uma razão de segurança/escalabilidade:

- **Business Account** → permite agrupar múltiplos apps e WABAs sob a
  mesma entidade legal
- **App** → permite que o mesmo Business Account integre com múltiplos
  sistemas (Chatwoot, CRM, BI, etc.) sem misturar credenciais
- **System User** → permite tokens que não dependem da sessão de uma
  pessoa específica (importante quando alguém sai da empresa)
- **WABA** → permite que o mesmo Business Account opere com múltiplos
  números (escalabilidade)
- **Phone Number** → cada número é uma entidade independente, com
  qualidade própria e métricas próprias

---

## 📞 Provisionamento do número

### Estratégia: número NOVO para teste, mantendo o antigo na plataforma legada

A operação tem um número antigo já em uso (na plataforma SaaS) com
**alta qualidade na Meta** (reputação construída ao longo do tempo). Esse
número **não foi migrado de imediato** — manteve-se na plataforma legada
para garantir continuidade do atendimento.

Um **número novo** foi provisionado especificamente para a fase de testes
do Chatwoot:

```
Número antigo (Umbler, em produção):  +55 81 9 8658-6878
Número novo (Chatwoot, em teste):     +55 81 9 8422-5358
```

### Por que dois números?

```
✅ Zero risco de downtime no atendimento real durante a configuração
✅ Liberdade para fazer testes (incluindo mensagens "feias") sem afetar
   a qualidade do número de produção
✅ Período de 15-30 dias para validar estabilidade antes da migração
   definitiva
✅ Rollback fácil se algo der errado (basta voltar para a plataforma
   antiga)
```

---

## 🔌 Configuração do Inbox no Chatwoot

Com a estrutura Meta criada, foi configurado o inbox no Chatwoot:

```
Caixa de Entrada:        WhatsApp - Recife Flats (+5581984225358)
Inbox ID:                3
Provider:                WhatsApp Cloud
Phone Number:            +5581984225358 (com "9")
Phone Number ID:         1094911290380821
WhatsApp Business
Account ID (WABA):       1755135172530822
API Key (token):         <System User Token permanente>
Webhook Verify Token:    <gerado pelo Chatwoot, único>
```

> 💡 **Atenção:** o `phone_number` foi inserido com o "9" móvel brasileiro
> aqui — esse foi exatamente o ponto onde o bug da Fase 05 apareceu, e
> teve que ser corrigido posteriormente.

### Agentes atribuídos ao inbox

```
✅ Gabriela Rodrigues  (atendimento principal)
✅ Lucas Lima          (operacional / escalonamento)
✅ Vanessa Rafaella    (admin / técnico)
```

---

## 🔗 Webhook configurado na Meta

Para que mensagens recebidas no número WhatsApp cheguem ao Chatwoot, é
necessário configurar o webhook na Meta apontando para o endpoint público
da instalação.

### URL do webhook

```
https://chat.recifeflatstemporada.com/webhooks/whatsapp/+5581984225358
```

### Subscrições ativadas

Eventos que a Meta vai notificar via webhook:

| Evento | Função |
|--------|--------|
| `messages` | Mensagens recebidas (texto, mídia, etc.) |
| `message_template_status_update` | Atualização de status de templates aprovados |

### Validação do webhook

A Meta exige uma validação inicial (callback de verificação) antes de
ativar o webhook. O Chatwoot lida com isso automaticamente — basta que o
`verify_token` no inbox do Chatwoot bata com o `Verify Token` configurado
na Meta.

---

## 🔑 Tokens — jornada de 24h até permanente

### Token de teste (24h)

Para o primeiro setup, foi usado um **User Access Token** de curta duração
(24h), gerado diretamente pela interface da Meta. Suficiente para validar
o pipeline de envio/recepção.

### Token de produção (System User Token permanente)

Para operação real, é necessário um token que **não expire** — caso
contrário, o sistema pararia de funcionar a cada 24h, exigindo renovação
manual ou pipeline de refresh automatizado.

A geração do token permanente envolveu um **bloqueio inesperado de 2FA**,
detalhado em uma fase própria:

🔗 **Ver:** [06-recuperacao-2fa/](../06-recuperacao-2fa/) — investigação
de 3 dias com 3 caminhos investigados até a resolução.

---

## ✅ Critérios de conclusão da fase

- [x] Meta Business Account criada e verificada
- [x] App Meta for Developers configurado e vinculado ao Business
- [x] WABA criada e aprovada
- [x] Phone Number provisionado e ativo na Cloud API
- [x] System User criado para a integração
- [x] Token permanente gerado e armazenado em segurança
- [x] Webhook configurado na Meta apontando para o Chatwoot
- [x] Subscrições corretas (`messages` + `message_template_status_update`)
- [x] Inbox WhatsApp configurado no Chatwoot
- [x] 3 agentes atribuídos ao inbox
- [x] Teste bidirecional (envio + recepção) confirmado

> 🐛 **Atenção:** após esta fase, foi identificado um bug de matching de
> número brasileiro entre Meta e Chatwoot. Detalhamento em
> [05-bug-9-movel/](../05-bug-9-movel/).

---

## 🧠 Aprendizados desta fase

1. **Estrutura Meta é complexa por design.** Os múltiplos objetos
   (Business → App + WABA + System User → Phone Number) confundem no
   começo, mas refletem necessidades reais de segurança e escalabilidade
   da plataforma. Vale entender o porquê antes de só seguir tutorial.

2. **Sempre testar com número novo dedicado.** Nunca usar o número de
   produção como cobaia. O risco de afetar a qualidade do número
   (reputação Meta) é alto durante testes — uma vez "queimado", a
   recuperação é demorada.

3. **System User Token > User Access Token para produção.** User token
   está vinculado a uma pessoa (se ela sai da empresa, o token vira pó).
   System User Token vive enquanto a empresa quiser, independente de
   quem está logado.

4. **Webhook verify token tem que ser idêntico nos dois lados.** Erro
   sutil mas comum: copiar com espaço extra, com "/" no final, etc. Em
   caso de dúvida, regerar dos dois lados e colar com cuidado.

5. **Token de 24h é suficiente para o setup, não para produção.** Usar
   token curto no início para validar o pipeline é estratégia válida,
   mas planejar o token permanente como passo obrigatório antes de
   colocar em produção real.

---

[← Voltar ao README principal](../README.md)
