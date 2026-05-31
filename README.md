

#  📚 Case: Migração de Plataforma de Atendimento WhatsApp para empresa de aluguel de temporada em Recife - PE 🏖️

> Caso real de migração de plataforma SaaS de atendimento WhatsApp 
> para stack open-source self-hosted, com economia de 81% nos custos 
> mensais e adição de capacidades inexistentes na plataforma anterior.

[![Status](https://img.shields.io/badge/status-em%20desenvolvimento-blue)]()
[![Stack](https://img.shields.io/badge/stack-Docker%20|%20PostgreSQL%20|%20n8n-blue)]()
[![Economia](https://img.shields.io/badge/economia-R$%202.028%2Fano-success)]()

## 📊 Resultados (parciais)

| Métrica | Antes | Depois |
|---|---|---|
| Custo mensal | R$ 208 | R$ 39 |
| Plataforma | SaaS proprietário | Open-source self-hosted |
| Automações | Básicas | Avançadas (em construção) |
| Integração com BD | Não tinha | Em desenvolvimento |

**Economia anual: R$ 2.028 (81% redução de custos)**

## 🎯 Sobre o projeto

Projeto real para empresa do setor de hospedagem por temporada, 
com 4 unidades operadas. O cliente utilizava plataforma SaaS apenas 
como caixa de entrada compartilhada de WhatsApp, sem aproveitar 
capacidades de automação ou integração de dados.

## 🛠️ Stack

- **VPS Hostinger** (Ubuntu 24.04, KVM 2)
- **Docker + Traefik** (reverse proxy com HTTPS automático)
- **Chatwoot 4.13** (atendimento omnichannel)
- **PostgreSQL + pgvector** (Supabase)
- **n8n** (motor de automação visual)
- **Meta WhatsApp Cloud API** (oficial)

## 📚 Documentação

- [Discovery e análise do problema](./01-discovery/)
- [Arquitetura da solução](./02-arquitetura/)
- [Fases de implementação](./03-implementacao/)
- [Design conversacional (Persona "Cecília")](./04-design-conversacional/)

## 👩‍💻 Sobre

Projeto desenvolvido como estudo prático de Sistemas para Internet, 
explorando arquitetura de aplicações distribuídas, integração de 
APIs e design conversacional.

---
🚧 Documentação em construção. Atualizado conforme as fases avançam.
