
# 🏗️ Fase 02 — Infraestrutura (VPS, Docker, DNS, SMTP)

> **Status:** ✅ Concluída
> **Output:** VPS provisionado com Docker, reverse proxy HTTPS, DNS configurado
> e SMTP transacional funcionando.

---

## 🎯 Objetivo da fase

Provisionar a infraestrutura base que vai hospedar todo o sistema:

- VPS Linux com recursos suficientes para Chatwoot + n8n + PostgreSQL + Redis
- Docker e Docker Compose para orquestração de containers
- Reverse proxy com SSL automático
- Domínio próprio com DNS configurado
- SMTP transacional para emails do Chatwoot (recuperação de senha, notificações)

---

## 🖥️ VPS — Hostinger KVM 2

### Especificações

```
Plano:       KVM 2 (Hostinger Cloud)
CPU:         2 vCores
RAM:         8 GB
Disco:       100 GB SSD
SO:          Ubuntu 24.04 LTS
Localização: Boston, USA
IPv4:        2.25.140.217
Hostname:    srv1710017.hstgr.cloud
Custo:       R$43,99/mês (plano anual) | ~2x no plano mensal avulso
```

### Justificativa do dimensionamento

- **8 GB de RAM**: Chatwoot recomenda mínimo de 4 GB, mas precisamos de espaço
  para PostgreSQL + Redis + Sidekiq + n8n futuro + sobra para picos
- **2 vCores**: suficiente para single-node com baixo a médio volume
- **100 GB SSD**: muito mais do que necessário para logs e mídia de WhatsApp
  (geralmente bastam 20-30 GB), mas dá margem confortável para crescimento

### Estratégia de POC (decisão importante)

O primeiro mês foi contratado no **plano mensal avulso** (custo
proporcionalmente maior) para validar a viabilidade técnica antes de
comprometer 12 meses.

```
Mês 1:    Plano mensal avulso (~R$80-90)
          ↓
          Validação técnica: stack instalada, Chatwoot operando,
          Meta API integrada, bug do "9" diagnosticado e corrigido,
          token permanente conquistado
          ↓
Mês 2:    Migração para plano anual (R$43,99/mês × 12)
```

**Justificativa:** validar antes de comprometer 12 meses adiantados é gestão
de risco básica de projeto. Pagar mais por 1 mês para evitar contrato anual
em stack não-validada é matemática simples e profissional.

---

## 🐳 Docker e Docker Compose

### Por que Docker?

- **Reprodutibilidade:** o Chatwoot oficial publica `docker-compose.yml`
  oficial. Seguir o caminho oficial reduz risco de incompatibilidades.
- **Isolamento:** cada serviço (Rails, Sidekiq, Redis, PostgreSQL) roda em
  seu próprio container, com volumes nomeados para persistência.
- **Atualização:** `docker compose pull && docker compose up -d` para
  atualizar versões.

### Containers em produção

```
chatwoot-rails-1       (aplicação web Rails)
chatwoot-sidekiq-1     (worker de jobs em background)
chatwoot-redis-1       (cache + fila do Sidekiq)
chatwoot-postgres-1    (banco PostgreSQL 16 com pgvector)
chatwoot-traefik-1     (reverse proxy + SSL automático)
```

### Localização dos arquivos no VPS

```
/root/chatwoot/
├── docker-compose.yml
├── .env                      ← variáveis sensíveis (FRONTEND_URL, SMTP, etc.)
└── data/                     ← volumes Docker (postgres, redis, storage)
```

---

## 🌐 Domínio e DNS

### Domínio

`recifeflatstemporada.com` — domínio próprio do cliente, já registrado e em uso.

### DNS — Netlify

O DNS é gerenciado pelo Netlify (escolha do cliente, anterior ao projeto).
Para o subdomínio dedicado ao Chatwoot:

```
Subdomínio:  chat.recifeflatstemporada.com
Tipo:        A
Aponta para: 2.25.140.217 (IPv4 do VPS)
TTL:         3600
```

### Por que subdomínio dedicado?

- Isola completamente o sistema de atendimento do site principal
- Permite trocar/migrar o site sem afetar o Chatwoot e vice-versa
- Certificado SSL específico (sem precisar de wildcard)
- Mais fácil de revogar/migrar se for necessário no futuro

---

## 🔒 Reverse Proxy — Traefik

### Por que Traefik?

- **SSL automático via Let's Encrypt** sem precisar de scripts de renovação
  manuais (Certbot, etc.) — o Traefik renova certificados sozinho
- **Service discovery automático** via labels do Docker
- **Configuração declarativa** dentro do mesmo `docker-compose.yml` do
  Chatwoot — uma única fonte de verdade

### Configuração relevante

O Traefik observa containers com labels específicos e roteia o tráfego
HTTPS para o serviço correto:

```yaml
# Trecho do docker-compose.yml (Chatwoot)
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.chatwoot.rule=Host(`chat.recifeflatstemporada.com`)"
  - "traefik.http.routers.chatwoot.entrypoints=websecure"
  - "traefik.http.routers.chatwoot.tls.certresolver=letsencrypt"
```

### Validação

```bash
curl -I https://chat.recifeflatstemporada.com
# HTTP/2 200
# Certificado válido emitido por Let's Encrypt R3
```

---

## 📧 SMTP — Brevo (transacional)

### Por que SMTP externo?

O Chatwoot precisa enviar emails para:

- Recuperação de senha de agentes
- Notificações de novas conversas (para atendentes que não estão online)
- Confirmação de novos cadastros
- Convites para membros da equipe

Enviar email direto do VPS é um problema porque IPs de VPS quase sempre estão
em listas de spam dos principais provedores (Gmail, Outlook). Solução: usar
um SMTP externo que tenha IPs com reputação validada.

### Por que Brevo (antigo Sendinblue)?

- **Free tier de 300 emails/dia** (mais que suficiente para o uso do Chatwoot)
- **Sender domain validation** simples (SPF + DKIM)
- **Painel de logs** fácil de inspecionar (vê se email saiu, abriu, etc.)
- Bom histórico de entregabilidade

### Configuração no Chatwoot (`.env`)

```bash
# Trecho do .env (valores reais omitidos)
MAILER_SENDER_EMAIL=admrecifeflatstemporada@gmail.com
SMTP_DOMAIN=smtp-relay.brevo.com
SMTP_ADDRESS=smtp-relay.brevo.com
SMTP_PORT=587
SMTP_USERNAME=<LOGIN_BREVO>
SMTP_PASSWORD=<SMTP_KEY_BREVO>
SMTP_AUTHENTICATION=login
SMTP_ENABLE_STARTTLS_AUTO=true
SMTP_OPENSSL_VERIFY_MODE=peer
```

### Validação

Teste via Rails console:

```bash
docker compose exec rails bundle exec rails runner "
  ActionMailer::Base.mail(
    from: 'admrecifeflatstemporada@gmail.com',
    to: 'teste@email.com',
    subject: 'Teste SMTP',
    body: 'Funcionando!'
  ).deliver_now
"
```

Email chegou na caixa de entrada (não em spam) em poucos segundos.

---

## 🐛 Problemas enfrentados nesta fase

### Bug do `FRONTEND_URL`

Durante a configuração inicial, o `FRONTEND_URL` no `.env` ficou apontado
para o hostname interno do servidor (`srv1710017.hstgr.cloud`) em vez do
domínio público (`chat.recifeflatstemporada.com`).

**Sintoma:** links em emails de recuperação de senha apontavam para o
hostname interno, que não tem certificado SSL válido externamente. Usuários
recebiam erro de certificado.

**Correção:**

```bash
nano ~/chatwoot/.env
# Alterar:
FRONTEND_URL=https://chat.recifeflatstemporada.com

# Reiniciar Chatwoot
docker compose restart rails sidekiq
```

Lição: variáveis de ambiente da aplicação web sempre devem refletir o
**endereço público**, não o hostname interno do servidor.

---

## ✅ Critérios de conclusão da fase

- [x] VPS provisionado e SSH funcionando
- [x] Docker e Docker Compose instalados
- [x] Domínio apontando corretamente para o VPS
- [x] Traefik rodando com SSL automático via Let's Encrypt
- [x] HTTPS funcionando em `chat.recifeflatstemporada.com`
- [x] SMTP Brevo configurado e testado
- [x] Backup pg_dump diário agendado (cron)

---

## 🧠 Aprendizados desta fase

1. **POC antes de plano anual.** Pagar mais por 1 mês para validar a stack
   inteira foi gestão de risco simples e eficaz. Recomendo sempre que houver
   diferença significativa entre plano mensal e anual.

2. **Subdomínio dedicado vale o esforço.** Isolar `chat.` em vez de colocar
   na raiz facilita manutenção futura — trocar provedor, migrar para outro
   subdomínio, ou descomissionar sem afetar o site principal.

3. **SMTP externo é mandatório.** Mesmo para baixo volume de email, IPs de
   VPS quase sempre são bloqueados pelos principais provedores. Usar
   serviço transacional com IPs reputados é o caminho.

4. **Variáveis de ambiente refletem URL pública, não hostname interno.**
   Bug do `FRONTEND_URL` foi pequeno mas ilustrativo: app web nunca deve
   referenciar o hostname interno do servidor.

---

[← Voltar ao README principal](../README.md)
