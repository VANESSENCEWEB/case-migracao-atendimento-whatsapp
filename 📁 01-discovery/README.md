# 📋 Fase 01 — Discovery e decisão técnica

> **Status:** ✅ Concluída
> **Período:** Início do projeto
> **Output:** Decisão de stack documentada com justificativa

---

## 🎯 Objetivo da fase

Antes de escrever uma linha de código ou provisionar qualquer infraestrutura,
era necessário responder com clareza:

1. Por que migrar?
2. Migrar para o quê?
3. Quanto vai custar? (mensal e total ao longo do tempo)
4. Quais são os riscos?
5. Em quanto tempo o ROI é positivo?

Essa fase produziu as respostas a essas cinco perguntas.

---

## 🔍 Análise da situação atual

A empresa cliente operava com uma plataforma SaaS brasileira de atendimento
multicanal, focada em WhatsApp. O sistema atendia bem do ponto de vista
operacional, mas tinha problemas estruturais relevantes:

### Custos

- **R$208/mês** = R$2.496/ano só pela plataforma
- Custo crescente conforme volume aumenta (mais atendentes, mais conversas)
- Sem possibilidade de negociar valores ou suspender temporariamente

### Limitações funcionais

- Automações restritas ao que a plataforma oferecia (sem extensibilidade real)
- Integração com sistema interno via **copy-paste manual** entre interfaces
- Bot/fluxos limitados a triggers pré-definidos
- Sem webhooks bidirecionais para integração com Supabase (sistema interno)

### Riscos estratégicos

- **Dados de hóspedes** em servidor de terceiros
- Sem garantia de portabilidade dos dados em caso de cancelamento
- Dependência total do roadmap e estabilidade da plataforma
- Sem controle sobre uptime ou capacidade

---

## 🛠️ Alternativas avaliadas

Foram avaliadas 5 categorias de alternativas:

### A. Continuar com a plataforma atual

**Prós:** zero esforço, equipe já treinada.
**Contras:** mantém todos os problemas estruturais.
**Decisão:** descartado.

### B. Twilio + frontend próprio

**Prós:** API maduríssima, escalável, robusta.
**Contras:** preço por mensagem em volume mensal pode ficar caro; lock-in
significativo; precisaria construir frontend de atendimento (esforço enorme).
**Decisão:** descartado por custo total e esforço de desenvolvimento.

### C. Z-API / Evolution API (WhatsApp Web não oficial)

**Prós:** baratíssimo, sem aprovação da Meta necessária.
**Contras:** **alto risco de banimento** (não usa API oficial), instabilidade,
sem suporte oficial, problemas legais em conformidade LGPD.
**Decisão:** descartado por risco operacional inaceitável.

### D. 360dialog ou outros BSPs (Business Solution Providers)

**Prós:** API oficial Meta, suporte profissional.
**Contras:** preço elevado para o porte da operação; ainda exige construir
ou contratar frontend separado.
**Decisão:** descartado por custo desproporcional ao porte.

### E. Chatwoot self-hosted + Meta Cloud API direta

**Prós:** open-source, multi-canal, API rica, comunidade ativa, integração
oficial com Meta, hospedagem própria = soberania total.
**Contras:** requer conhecimento de DevOps para manutenção; responsabilidade
de uptime fica com a equipe.
**Decisão:** ✅ **escolhida**.

---

## 💰 Análise financeira comparativa

Projeção para 12 meses (sem considerar crescimento de volume):

| Opção | Custo mensal | Custo anual | Esforço inicial | Soberania de dados |
|-------|--------------|-------------|------------------|---------------------|
| A. Plataforma atual | R$208 | R$2.496 | Zero | ❌ Não |
| B. Twilio + frontend | ~R$150-400+ | R$1.800-4.800+ | Alto (dev) | ⚠️ Parcial |
| C. WhatsApp Web não-oficial | R$30-80 | R$360-960 | Médio | ⚠️ Risco alto |
| D. 360dialog | R$300-500+ | R$3.600+ | Médio | ⚠️ Parcial |
| **E. Chatwoot + Meta** | **R$43,99** ¹ | **R$527,88** | Médio (infra) | ✅ **Total** |

> ¹ Plano anual Hostinger KVM 2. No plano mensal avulso o valor é
> aproximadamente o dobro.

### Economia projetada

```
Plataforma atual:     R$2.496/ano
Chatwoot + Meta:      R$  528/ano
─────────────────────────────────
Economia anual:       R$1.968 (-79%)
```

---

## ⏱️ Análise de ROI

| Marco | Quando |
|-------|--------|
| Início do projeto | Mês 0 |
| Stack funcionando em paralelo (sem cancelar antigo) | Mês 1 |
| Validação completa do novo sistema | Mês 2 |
| Cancelamento do sistema antigo | Mês 3 |
| Break-even (economia compensa esforço inicial) | ~Mês 6 |
| ROI positivo claro | Mês 12+ |

**Esforço inicial estimado:** ~60-80h de trabalho técnico (provisioning,
configuração, debug, integração, testes).

Considerando custo de oportunidade do tempo, o ROI ainda é amplamente positivo
em 12 meses — sem contar os benefícios não-monetários (autonomia, controle de
dados, capacidade de automação futura).

---

## ⚠️ Riscos identificados e mitigações planejadas

| Risco | Mitigação |
|-------|-----------|
| Migração gerar downtime no atendimento | Estratégia de operação paralela 30 dias antes de descomissionar o antigo |
| Bug ou instabilidade no Chatwoot self-hosted | Versão LTS estável + backup pg_dump diário + monitoramento |
| Equipe (Gabriela, Lucas) não se adaptar à nova interface | Treinamento via docs internos + período de transição com os dois sistemas |
| Esforço técnico maior que estimado | POC com plano mensal avulso de VPS antes de comprometer 12 meses |
| Meta bloquear/banir o número durante setup | Usar número novo dedicado para testes, manter número antigo na plataforma atual até validação |

---

## ✅ Decisão final

**Stack escolhida:** Chatwoot v4.13.0 self-hosted + Meta WhatsApp Cloud API
(oficial) + n8n para automações + Supabase (sistema interno já existente).

**Justificativa principal:** é a única opção que combina:
- Custo significativamente menor (-79%)
- Soberania total dos dados
- API oficial Meta (sem risco de banimento)
- Extensibilidade via n8n para automações
- Comunidade open-source ativa (manutenção e features contínuas)
- Custo previsível (VPS fixo) versus custo variável (por mensagem ou por agente)

**Próximas fases:** infraestrutura (Fase 02) → deploy Chatwoot (Fase 03) →
integração Meta (Fase 04).

---

[← Voltar ao README principal](../README.md)
