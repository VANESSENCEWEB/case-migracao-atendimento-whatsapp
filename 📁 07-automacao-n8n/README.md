# Fase 07 — Automação do atendimento com n8n

> Status: 🟢 Em produção (núcleo funcional) · alguns testes de ponta a ponta
> ainda pendentes — ver seção "O que falta" no final.

Esta fase cobre a camada de automação que transforma o Chatwoot de uma caixa
de mensagens em um atendimento conversacional: reconhece se o hóspede já tem
reserva, consulta o sistema interno, explica e cobra a caução automaticamente,
e só aciona um atendente humano quando realmente precisa.

## Arquitetura

```
WhatsApp (Meta) → Chatwoot → webhook → n8n (Motor de Estado) → Postgres/Supabase
                                                ↳ InfinitePay (checkout da caução)
                                                ↳ API do Chatwoot (etiquetas + setor)
```

Dois workflows n8n compõem esta fase:

1. **Sprint 1B — MVP Saudação** — recebe toda mensagem do hóspede e decide o
   que responder.
2. **InfinitePay — Confirmação Caução** — recebe a confirmação de pagamento
   da caução e atualiza a conversa no Chatwoot.

## O "Motor de Estado"

O núcleo da automação é um único nó de código (JavaScript) de ~290 linhas
dentro do workflow Sprint 1B. Ele guarda, por conversa, um estado (em que
ponto da conversa o hóspede está) e os dados já coletados, usando o
armazenamento persistente nativo do n8n (`$getWorkflowStaticData`) — sem
precisar de tabela extra no banco só para isso.

Fluxos que o Motor de Estado conduz sozinho, sem intervenção humana:

- **Confirmação de reserva existente** — pede o código da reserva e consulta
  o Postgres/Supabase.
- **Nova reserva / disponibilidade** — recolhe check-in, número de pessoas,
  quartos e necessidade de estacionamento.
- **Caução** — explica o que é, responde dúvidas e objeções comuns
  automaticamente, e gera o link de pagamento (InfinitePay) na hora, sem
  precisar de um atendente.
- **Dúvidas gerais / FAQ** — coleta a dúvida e, se não conseguir resolver
  sozinho, escala para um atendente.
- **Transferência para atendente humano** — acionada explicitamente quando o
  hóspede pede, quando a reserva não é encontrada, ou depois de duas
  respostas seguidas que o robô não entendeu.

### Atendimento 24 horas (correção de 17/08/2026)

**Antes:** qualquer mensagem recebida fora do horário comercial (8h–20h,
horário de Recife) travava a conversa inteira logo na primeira mensagem —
o robô só dizia "estamos fora do horário" e não recolhia mais nada.

**Depois:** o robô conduz o fluxo completo (reserva, caução, disponibilidade,
dúvidas) 24 horas por dia. O aviso de horário comercial só aparece no
momento exato em que a conversa precisa de um atendente humano de verdade —
avisando que alguém vai responder assim que o expediente reabrir.

A mudança ficou concentrada em um único ponto do código (checagem logo antes
do retorno final, olhando se o novo estado é "transferido para humano"), em
vez de espalhada pelos ~9 pontos do fluxo que podem levar a uma transferência
— o que manteve o código pequeno e fácil de manter.

Validado simulando uma conversa de teste diretamente no editor do n8n
(sem afetar hóspedes reais): mensagem inicial fora do horário segue o fluxo
normal; o aviso aparece só na transferência; estados do meio do fluxo não
mostram o aviso indevidamente.

## Sincronização de etiquetas e setores do Chatwoot

O Chatwoot já tinha uma taxonomia própria configurada (4 setores, 31
etiquetas), mas o n8n só enviava mensagens de texto — nunca tocava nessa
organização. Foram adicionados nós de automação (via HTTP Request para a
API do Chatwoot, `/conversations/{id}/labels` e `/assignments`) nos dois
workflows, para que cada conversa seja etiquetada e roteada para o setor
certo automaticamente conforme o andamento (reserva confirmada, caução
paga, etc.), sem trabalho manual do atendente.

## Como importar o workflow

1. No n8n: **Workflows → Import from File**
2. Selecione `workflows/sprint-1b-mvp-saudacao.json` (nesta pasta)
3. Vincule as credenciais (Chatwoot API, Postgres, InfinitePay) aos nós
   correspondentes
4. Ative o workflow e configure o webhook de produção no Chatwoot
   (**Settings → Integrations → Webhooks**, evento `message_created`)

## O que falta

- Teste ponta a ponta completo da confirmação de pagamento da caução
  (a automação funciona; falta uma simulação completa e documentada de
  toda a cadeia).
- Acompanhar conversas reais fora do horário comercial nos primeiros dias
  para confirmar em produção o que já foi validado em simulação.
