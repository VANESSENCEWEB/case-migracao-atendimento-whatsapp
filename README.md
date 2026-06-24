# Case Técnico: Migração de Plataforma de Atendimento WhatsApp

> Migração completa do sistema de atendimento via WhatsApp de uma empresa do
> setor de hospedagem (curta temporada) — de plataforma SaaS proprietária
> brasileira (R$208/mês) para stack **open-source self-hosted** (a partir de
> ~R$44/mês no plano anual), com economia anual estimada de **R$1.968 (-79%)**
> e ganho total de autonomia técnica.

**Stack final:** Chatwoot v4.13.0 + Meta WhatsApp Cloud API + n8n (planejado)
em VPS Linux (Ubuntu 24.04) com Docker, Traefik e Let's Encrypt.

---

## 📌 Sobre este case

Este repositório documenta a migração técnica completa do sistema de atendimento
via WhatsApp de uma empresa cliente do setor de hospedagem (detalhes
anonimizados). É um trabalho **em andamento** — algumas fases estão concluídas,
outras estão em execução ou planejadas.

A documentação está organizada **por fases cronológicas** do projeto, e cada
fase tem seu próprio diretório com README detalhado.

> 💡 Este não é um exercício acadêmico: a infraestrutura aqui descrita está
> em operação atendendo uma empresa real, e cada decisão técnica tem
> consequência operacional.

---

## 🎯 O desafio

A empresa cliente operava o atendimento WhatsApp por uma plataforma SaaS
brasileira proprietária. Apesar de funcional, o sistema apresentava:

- **Custo recorrente alto** (R$208/mês = R$2.496/ano)
- **Dependência total da plataforma** (sem controle sobre dados ou automações)
- **Limitações de integração** com o sistema interno de gestão de reservas
- **Inflexibilidade** para automações personalizadas
- **Dados de hóspedes em servidor de terceiros** sem garantia de portabilidade

### Objetivos

1. Reduzir custos operacionais em pelo menos 70%
2. Migrar para stack open-source com soberania total dos dados
3. Manter funcionalidade igual ou superior à plataforma anterior
4. Construir infraestrutura para automações futuras (n8n + webhooks)
5. Integrar atendimento com sistema interno existente
6. Garantir **zero downtime** durante a transição

---

## 🏗️ Arquitetura da solução

```
┌─────────────────────────────────────────────────────────────┐
│                    META WHATSAPP CLOUD API                  │
└──────────────────────┬──────────────────────────────────────┘
                       │ Webhooks
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  VPS LINUX (Ubuntu 24.04)                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Traefik (Reverse Proxy + Let's Encrypt SSL)         │   │
│  └──────────────────────────────────────────────────────┘   │
│                       │                                     │
│  ┌────────────────────┴───────────────────┐                 │
│  │           CHATWOOT v4.13.0             │                 │
│  │  ┌──────────┐ ┌─────────┐ ┌─────────┐  │                 │
│  │  │  Rails   │ │ Sidekiq │ │  Redis  │  │                 │
│  │  └──────────┘ └─────────┘ └─────────┘  │                 │
│  │  ┌──────────────────────────────────┐  │                 │
│  │  │   PostgreSQL (pgvector/pg16)     │  │                 │
│  │  └──────────────────────────────────┘  │                 │
│  └────────────────────────────────────────┘                 │
│                       │                                     │
│  ┌────────────────────┴───────────────────┐                 │
│  │   n8n (planejado: automações)          │                 │
│  └────────────────────────────────────────┘                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│        SUPABASE (sistema interno de gestão existente)       │
└─────────────────────────────────────────────────────────────┘
```

---

## 📂 Documentação por fase

| # | Fase | Status | Documentação |
|---|------|--------|--------------|
| 01 | Discovery & decisão técnica | ✅ Concluída | [01-discovery/](./01-discovery/) |
| 02 | Infraestrutura (VPS + Docker + DNS) | ✅ Concluída | [02-infraestrutura/](./02-infraestrutura/) |
| 03 | Deploy do Chatwoot | ✅ Concluída | [03-chatwoot-deploy/](./03-chatwoot-deploy/) |
| 04 | Integração Meta WhatsApp Cloud API | ✅ Concluída | [04-meta-cloud-api/](./04-meta-cloud-api/) |
| 05 | ⭐ Bug do "9" móvel brasileiro | ✅ Resolvido | [05-bug-9-movel/](./05-bug-9-movel/) |
| 06 | ⭐ Recuperação de acesso 2FA | ✅ Resolvido | [06-recuperacao-2fa/](./06-recuperacao-2fa/) |
| 07 | Estruturação do atendimento | 🟡 Em andamento | _em construção_ |
| 08 | Automações via n8n | 🔵 Planejada | _roadmap_ |
| 09 | Migração final + descomissionamento | 🔵 Planejada | _roadmap_ |

**Legenda:** ✅ Concluída · 🟡 Em andamento · 🔵 Planejada

> ⭐ As fases 05 e 06 são problemas técnicos não-triviais encontrados durante
> a implementação, cuja investigação e resolução estão documentadas em
> profundidade — recomendo essas duas leituras para uma visão mais densa do
> trabalho técnico envolvido.

---

## 🛠️ Stack & decisões técnicas (resumo)

| Componente | Escolha | Justificativa |
|-----------|---------|---------------|
| Atendimento | **Chatwoot v4.13.0** | Open-source, multi-canal, multi-agente, API rica, mantido ativamente |
| Containers | **Docker Compose** | Simples para single-node, fácil de reproduzir, oficial pelo Chatwoot |
| Reverse proxy | **Traefik** | SSL automático via Let's Encrypt, integração nativa com Docker |
| Banco | **PostgreSQL 16 (pgvector)** | Default do Chatwoot, suporta busca semântica futura |
| Fila | **Redis + Sidekiq** | Default do Chatwoot |
| API WhatsApp | **Meta Cloud API (oficial)** | Estável, escalável, suportado oficialmente pela Meta |
| Automações | **n8n** | Open-source, alternativa ao Zapier/Make, hospedado no mesmo VPS |
| SMTP | **Brevo** | Free tier de 300 e-mails/dia, sender domain validation simples |
| DNS | **Netlify** | DNS gerenciado, já usado pelo cliente |
| VPS | **Hostinger KVM 2** | 8GB RAM / 100GB SSD — sobra para n8n e expansão (¹) |

> (¹) **Estratégia de POC:** o primeiro mês foi contratado avulso para validar
> a viabilidade técnica antes de comprometer 12 meses. Após a validação,
> renovação no plano anual baixa o custo para ~R$43,99/mês (no plano mensal
> avulso o custo é cerca de 2x maior). Justificativa detalhada em
> [02-infraestrutura/](./02-infraestrutura/).

---

## 📊 Resultados parciais

| Métrica | Antes | Depois | Variação |
|---------|-------|--------|----------|
| Custo mensal | R$208 | R$43,99 (²) | **−79%** |
| Custo anual | R$2.496 | R$527,88 | **−R$1.968** |
| Controle sobre dados | Limitado | Total | ✅ |
| Capacidade de automação | Restrita à plataforma | Ilimitada (n8n) | ✅ |
| Integração com sistema interno | Manual (copy-paste) | Via webhook automatizada (planejada) | ✅ |
| Multi-canal preparado | Não | Sim (Telegram, Instagram, e-mail) | ✅ |
| Dependência de fornecedor | Alta | Mínima (apenas Meta) | ✅ |

> (²) Custo do VPS no plano anual (Hostinger KVM 2). Durante a fase de POC
> o custo foi proporcionalmente maior, mas o objetivo era validar viabilidade
> técnica antes de comprometer-se com plano anual.

---

## 🧠 Aprendizados gerais do projeto

1. **Logs > suposições.** Bugs em ferramentas open-source maduras existem e
   raramente estão documentados — diagnóstico metódico via leitura de logs e
   source code é diferencial.

2. **Webhooks > APIs REST quando o assunto é payload bruto.** APIs refletem
   o estado armazenado; webhooks entregam o estado original. Para interceptar
   dados que serão descartados na ingestão, webhook é o único caminho.

3. **Diagnóstico vence persistência.** Em vez de repetir o mesmo fluxo várias
   vezes esperando resultado diferente, mapear o sistema e escolher o ponto
   certo de intervenção economiza tempo.

4. **Segredos nunca passam pelo terminal "visível".** Uso de `read -s` para
   carregar tokens em variáveis de ambiente sem que apareçam em histórico
   do bash. Boa prática básica de DevSecOps.

5. **Estratégia de migração paralela > migração direta.** Manter dois sistemas
   funcionando em paralelo por 30 dias custa pouco e elimina risco de downtime.

6. **POC antes de compromisso longo.** Contratar VPS no plano mensal avulso
   para validar a stack inteira antes de migrar para o plano anual — gestão
   de risco simples e eficaz aplicada a infra.

7. **Nem todo problema técnico tem solução técnica.** O case da recuperação
   de 2FA (fase 06) é um lembrete claro: três caminhos técnicos foram
   investigados, e a solução veio do canal de suporte da própria plataforma.
   Reconhecer a hora de pivotar e usar o canal certo é skill profissional
   tanto quanto código.

---

## 👋 Sobre

Projeto realizado por **Vanessa Rafaella**, estudante de Sistemas para
Internet, em 2026.

Diagnóstico técnico, arquitetura, implementação e debug executados por mim,
com apoio de IA (Claude) para revisão de comandos, discussão de trade-offs e
validação de hipóteses. Toda decisão técnica final e cada linha colocada em
produção foram responsabilidade minha.

Este é o registro público de um trabalho real, não um exercício acadêmico:
a infraestrutura aqui descrita está em operação atendendo uma empresa de
hospedagem, e cada um dos cases técnicos detalhados foi um problema encontrado
e resolvido durante a implementação.

**Contato:** [LinkedIn / e-mail]

---

> *Repositório em construção — fases pendentes serão documentadas conforme
> forem sendo concluídas.*
