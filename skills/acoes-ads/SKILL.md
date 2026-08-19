---
name: acoes-ads
description: "Executa mudanças em Amazon Ads pelo Radar — pausar ou reativar campanha, ajustar orçamento diário, mudar lance de palavra-chave ou de alvo, negativar termo ou alvo — e revisa a fila de propostas da automação (aprovar ou recusar). Use quando o usuário pedir para pausar/ligar campanha, subir ou baixar lance ou orçamento, negativar um termo que gasta sem vender, ou perguntar o que o Radar está sugerindo nos anúncios."
---

# Ações em Amazon Ads

Aqui a conversa **muda a conta de anúncios de verdade**. Para só entender o desempenho, use a skill `ads` — esta é o passo seguinte, quando o vendedor decidiu agir.

**Conta:** exige `seller_account_id` (via `whoami` do `radarscout`).

## A regra que não se quebra

Toda ferramenta de ação roda **em simulação por padrão**: sem `execute:true` ela devolve exatamente o que seria enviado à Amazon e nada muda lá.

1. **Leia antes.** Pegue o estado atual: `list_ad_campaigns` (id e orçamento diário da campanha), `list_ad_bids` (`keyword_id`/`target_id` e o lance atual), `list_search_terms` (o termo antes de negativar). **Nunca invente um id** nem chute o valor atual.
2. **Simule.** Chame a ferramenta sem `execute` e mostre ao vendedor o de-para: valor de hoje → valor proposto, e por quê.
3. **Peça confirmação explícita** desse valor, nessa campanha.
4. **Só então** repita a chamada com `execute:true`.

Um "pode subir o lance" solto no meio da conversa **não é confirmação** de um valor específico. Simule, mostre o número, confirme.

## As ações

| Quero… | Ferramenta | Precisa de |
| --- | --- | --- |
| Pausar campanha | `pause_campaign` | `campaignId` |
| Religar campanha | `resume_campaign` | `campaignId` |
| Mudar orçamento diário | `update_campaign_budget` | `campaignId`, `dailyBudget` |
| Mudar lance de palavra-chave | `update_keyword_bid` | `keywordId`, `bid` |
| Mudar lance de alvo | `update_target_bid` | `targetId`, `bid` |
| Bloquear uma busca | `create_negative_keyword` | `campaignId` (sempre), `keywordText`, `matchType`; `adGroupId` para valer só no grupo |
| Bloquear um produto/categoria | `create_negative_target` | `campaignId` (sempre), `expression`; `adGroupId` para valer só no grupo |

Negativação **sem** `adGroupId` vale para a campanha inteira — diga isso ao vendedor antes de executar; é o erro mais caro de reverter.

## As proteções (e como explicá-las)

O Radar segura mudanças bruscas. Vale explicar em português, não citar o nome técnico:

- **Mudança gradual:** orçamento até **30%** por vez, lance até **50%** por vez.
- **Faixas de segurança:** lance de **R$ 0,02 a R$ 500**; orçamento de **R$ 1 a R$ 100.000** — barram erro de digitação.
- **Tempo de observação:** depois de uma mudança, aquela campanha, palavra-chave ou alvo espera **7 dias** antes de aceitar outra (**14 dias** para negativações) — é o tempo de as vendas por atribuição amadurecerem e dar para ver o efeito. Pausar e religar contam como a mesma mudança.
- **Teto diário da conta:** pelo menos 100 alterações/dia no total e até 50 por campanha.

Quando uma chamada volta recusada, o motivo vem junto:

| Resposta | O que dizer ao vendedor |
| --- | --- |
| `dry_run` | Simulação: "é isso que eu mandaria — confirma?" |
| `ok` | Aplicado na Amazon. |
| `rejected_cooldown` | Essa campanha/lance mudou há pouco; faltam `daysRemaining` dias de observação. |
| `rejected_hard_limit` | O valor está fora da faixa permitida ou o salto é grande demais — mostre a `allowedRange`. |
| `rejected_rate_limit` | Bateu o teto de alterações do dia. |
| `api_error` | A Amazon recusou — repasse a mensagem, não repita a chamada às cegas. |
| `not_connected` | A conta de anúncios não está conectada ao Radar. |
| `not_enabled` | A execução de mudanças não está liberada para essa conta. |

**Forçar (`override_cooldown`, `override_rate_limit`) só depois** de a chamada ter sido recusada exatamente por esse motivo **e** com o vendedor confirmando que quer passar por cima. Nunca mande "forçar" preventivamente, nunca por conta própria.

## Fila de propostas da automação

O motor de Ads do Radar propõe mudanças; quem decide é o vendedor.

1. `list_ads_proposals` — traz as **pendentes** por padrão (`status:"all"` para todas). Cada uma vem com `reasoning`, `params` e `ageInHours`.
2. Apresente a proposta em linguagem do vendedor: o que muda, de quanto para quanto, e o motivo.
3. `approve_ads_proposal` (executa) ou `reject_ads_proposal` com `reason` — o motivo fica registrado.

Duas ressalvas:

- **Proposta com mais de 24 horas pode estar velha** — lances, orçamentos e status mudam. Confira o estado atual pelas ferramentas de leitura antes de aprovar.
- **A execução é assíncrona.** Aprovar não confirma o resultado: consulte `list_ads_proposals` de novo. `executed` = aplicada; `failed` = falhou; `unknown` = **o sistema não conseguiu confirmar** — investigue pelo `auditId` antes de tratar como feito.

## O que entregar

- Antes: o de-para (hoje → proposto), o motivo e o que a mudança afeta.
- Depois: o que efetivamente aconteceu — aplicado, recusado (com o motivo em português) ou aguardando confirmação.
- Quando o vendedor pedir uma decisão que a leitura não sustenta (poucos cliques, janela imatura), **diga isso antes de executar**. A skill `ads` tem as premissas de leitura.

## Exemplo

> **Usuário:** "Baixa o lance daquela keyword que tá queimando dinheiro."
> **Skill:** `list_ad_bids` (lance atual R$ 2,40) → simula `update_keyword_bid` com R$ 1,80.
> **Resposta:** "Hoje o lance de *capa de celular* é R$ 2,40. Simulei a redução para **R$ 1,80** (−25%, dentro do permitido) — nada foi enviado à Amazon ainda. Confirma que aplico? Depois de aplicar, essa palavra-chave fica 7 dias sem aceitar nova mudança, para dar tempo de ver o efeito."

## Cuidados

- **Nunca execute sem confirmação explícita do valor.** Simulação primeiro, sempre.
- **Nunca invente ids.** `campaignId`, `keywordId`, `targetId` vêm das ferramentas de leitura.
- **Nunca force** uma proteção sem recusa prévia por aquele motivo e sem o vendedor pedir.
- **Não negative um termo** em cima de poucos cliques ou de janela recente — veja `keyword_type` e volume na skill `ads`.
- **Não trate proposta aprovada como concluída** sem reler a fila; `unknown` pede investigação.
- Não decida a estratégia pelo vendedor: quanto cortar, quando pausar e o ACoS-alvo são dele. Sem estratégia declarada, exponha o trade-off e pergunte.
