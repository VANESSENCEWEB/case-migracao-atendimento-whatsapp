# 🚀 Fase 03 — Deploy do Chatwoot

> **Status:** ✅ Concluída
> **Output:** Chatwoot v4.13.0 instalado, configurado, com estrutura
> organizacional pronta para receber o primeiro inbox.

---

## 🎯 Objetivo da fase

Instalar e configurar o Chatwoot self-hosted no VPS já provisionado, deixando
a aplicação pronta para receber canais (WhatsApp, eventualmente outros) e
operar como sistema central de atendimento.

---

## 📦 Versão escolhida

**Chatwoot v4.13.0** — última versão estável na época da implementação.

### Justificativa

- Versão recente o suficiente para ter as features mais novas (multi-canal,
  AI assistant, bots nativos)
- Madura o suficiente para já ter passado por bugfixes pós-lançamento
- Compatível com PostgreSQL 16 (versão default do compose oficial)
- Suporte a `pgvector` (extensão usada pelo Chatwoot para busca semântica)

---

## 🐳 Instalação via Docker Compose oficial

A instalação seguiu o caminho oficial do Chatwoot, que disponibiliza um
`docker-compose.yml` pronto para produção.

### Passos da instalação

```bash
# 1. Clonar repositório oficial
cd /root
git clone https://github.com/chatwoot/chatwoot.git
cd chatwoot

# 2. Copiar e configurar variáveis de ambiente
cp .env.example .env
nano .env
# Editar: FRONTEND_URL, SECRET_KEY_BASE, POSTGRES_PASSWORD, SMTP_*

# 3. Subir containers
docker compose up -d

# 4. Rodar migrations e setup inicial
docker compose exec rails bundle exec rails db:chatwoot_prepare
```

### Containers em produção

Após `docker compose up -d`, os seguintes containers ficam rodando:

```
chatwoot-rails-1       Aplicação web (Rails) — porta 3000 interna
chatwoot-sidekiq-1     Worker de jobs em background (webhooks, emails)
chatwoot-redis-1       Cache + fila do Sidekiq
chatwoot-postgres-1    PostgreSQL 16 com pgvector
chatwoot-traefik-1     Reverse proxy + SSL (configurado na Fase 02)
```

---

## ⚙️ Variáveis de ambiente críticas

O arquivo `.env` contém configurações essenciais. As mais importantes:

```bash
# URL pública (não o hostname interno do servidor!)
FRONTEND_URL=https://chat.recifeflatstemporada.com

# Banco PostgreSQL (com credentials geradas)
POSTGRES_HOST=postgres
POSTGRES_DATABASE=chatwoot
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=<senha_forte_gerada>

# Redis
REDIS_URL=redis://redis:6379

# Secret keys (geradas via Rails)
SECRET_KEY_BASE=<gerada_com_rails_secret>

# SMTP (configurado na Fase 02)
SMTP_ADDRESS=smtp-relay.brevo.com
SMTP_PORT=587
SMTP_USERNAME=<login_brevo>
SMTP_PASSWORD=<smtp_key>

# Habilitar registros via convite apenas (mais seguro)
ENABLE_ACCOUNT_SIGNUP=false
```

> 🛡️ **Segurança:** o `.env` tem permissões restritivas (`chmod 600`) e
> nunca foi commitado em controle de versão. Backups do `.env` ficam em
> local separado com criptografia.

---

## 👤 Configuração inicial

### Conta admin

Após a primeira execução, foi criada a conta master via interface web:

```
Nome:    Vanessa Rafaella
Email:   <email_admin>
Senha:   <senha_forte_via_password_manager>
Role:    Administrator
```

### Conta da empresa

Dentro da instalação Chatwoot, foi criada a "Account" (entidade que pode ter
múltiplos canais, agentes e configurações):

```
Account: Recife flats temporada
```

---

## 👥 Agentes adicionados

Três agentes foram cadastrados na plataforma:

| Agente | Função | Permissão |
|--------|--------|-----------|
| Vanessa Rafaella | Owner / técnica | Administrator |
| Gabriela Rodrigues | Atendimento WhatsApp | Agent |
| Lucas Lima | Operacional (check-ins, manutenção) | Agent |

Cada agente recebeu convite por email (validando o SMTP da Fase 02) e
configurou senha própria via link do convite.

---

## 🏢 Estrutura organizacional

### Teams (setores)

Foram criados 4 teams para organizar o atendimento por área:

| Team | Cor | Função |
|------|-----|--------|
| **Vendas e reservas** | 🔵 Azul | Leads, perguntas de pré-reserva, fechamento |
| **Hóspedes ativos** | 🟢 Verde | Atendimento durante a estadia (WiFi, ar, problemas) |
| **Manutenção & suporte** | 🟠 Laranja | Solicitações operacionais que envolvem Lucas |
| **Pós check-out & feedback** | 🟣 Roxo | Recuperação de feedback, reviews, retorno comercial |

### Por que essa divisão?

A operação tem **fluxos distintos** com necessidades diferentes:

- Lead (pré-reserva) → resposta rápida + comercial
- Hóspede ativo → resposta prioritária + resolutiva
- Manutenção → escalável pra equipe operacional (Lucas)
- Pós check-out → cadência diferente, foco em fidelização

Misturar tudo no mesmo inbox sem separação seria caótico em volume e
prioridade.

---

## 🏷️ Etiquetas (labels)

Etiquetas foram criadas para classificar conversas por apartamento (já que
a empresa opera vários):

```
🏷️ administrativo
🏷️ apto-105
🏷️ apto-203
🏷️ apto-804
🏷️ apto-1006
```

Cada conversa pode receber múltiplas etiquetas (ex.: `apto-105` +
`administrativo`), facilitando filtros e relatórios futuros.

---

## 📡 Inboxes (canais de comunicação)

Dois canais inicialmente configurados:

| Canal | Tipo | Status |
|-------|------|--------|
| Recife Flats Temporada | Website (widget de chat) | ✅ Ativo |
| WhatsApp - Recife Flats | WhatsApp Cloud API | ✅ Ativo (configurado na Fase 04) |

O canal de WhatsApp é detalhado na próxima fase, que envolve toda a
integração com a Meta Cloud API.

---

## 🛡️ Hardening básico de segurança

Após a instalação inicial, foram aplicados ajustes de segurança:

```bash
# 1. Desabilitar signup público (apenas convites)
ENABLE_ACCOUNT_SIGNUP=false

# 2. Senhas fortes para Rails Secret Key Base
SECRET_KEY_BASE=$(docker compose exec rails bundle exec rails secret)

# 3. Backup pg_dump diário via cron
0 3 * * * docker compose -f /root/chatwoot/docker-compose.yml \
  exec -T postgres pg_dump -U postgres chatwoot \
  > /root/backups/chatwoot_$(date +\%Y\%m\%d).sql

# 4. Retenção de backups (manter últimos 14 dias)
find /root/backups -name "chatwoot_*.sql" -mtime +14 -delete
```

---

## ✅ Critérios de conclusão da fase

- [x] Chatwoot v4.13.0 instalado e rodando
- [x] HTTPS funcionando em `chat.recifeflatstemporada.com`
- [x] Login admin operacional
- [x] SMTP testado (convites de agentes funcionando)
- [x] 3 agentes cadastrados (Vanessa, Gabriela, Lucas)
- [x] 4 teams criados
- [x] Etiquetas criadas (uma por apartamento)
- [x] Widget de chat (website channel) configurado
- [x] Backup pg_dump diário automatizado
- [x] Signup público desabilitado

---

## 🧠 Aprendizados desta fase

1. **Caminho oficial > caminho otimizado.** Tentar otimizar o
   `docker-compose.yml` antes de validar a stack é cilada — primeiro fazer
   funcionar pelo padrão oficial, depois otimizar se for necessário.

2. **Conta admin via interface web, não via Rails console.** Cria a primeira
   conta pela interface pra garantir que SMTP, sessão, e fluxo de senha
   estão funcionando ponta a ponta.

3. **Estrutura organizacional vem ANTES do canal.** Teams, etiquetas e
   agentes devem estar prontos antes de conectar canais externos — assim
   as primeiras conversas já chegam organizadas.

4. **Backup automatizado desde o dia 1.** Não esperar "depois" para
   configurar backup. Tem que estar rodando desde o primeiro dia de
   operação real.

5. **Desabilitar signup público é não-negociável.** Em qualquer
   self-hosted exposto na internet, signup aberto é convite a abuso.
   Apenas convites por email.

---

[← Voltar ao README principal](../README.md)
