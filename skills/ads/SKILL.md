---
name: ads
description: "Analisa a performance de Amazon Ads de uma conta de seller — investimento, vendas de anúncios, ACoS/ROAS, cliques e conversão, no total da conta e por campanha, com o estado atual de cada uma. Use quando o usuário perguntar como estão os anúncios, quanto gastou em Ads, qual o ACoS, quais campanhas gastam sem vender, ou quiser revisar o desempenho das campanhas."
---

# Performance de Amazon Ads

Responde duas perguntas: **"quanto investi e o que voltou?"** (conta) e **"quais campanhas puxam e quais queimam?"** (por campanha).

Esta skill dá **premissas de leitura**, não uma estratégia de campanha. Estruturas e táticas variam por operação — pergunte a do vendedor em vez de assumir uma.

## Conta e período

- **Conta:** exige `seller_account_id` (via `whoami` do `radarscout`).
- **Período:** ISO `YYYY-MM-DD`, **fim exclusivo**. Sem período, use os **últimos 30 dias** (Ads precisa de volume para o número significar algo) e diga a janela.
- **Atribuição:** as conversões de Ads amadurecem ao longo da janela de atribuição — os **~3 dias mais recentes costumam vir subestimados**. As tools devolvem exatamente o período pedido e sinalizam isso em `attribution_note`; **desconte os dias recentes ao decidir** e não confunda com queda real.

## Passo a passo

1. Chame `get_ads_overview` do `radarscout` com a janela: investimento, vendas de anúncios, impressões, cliques, pedidos e os derivados **ACoS, ROAS, CTR, CVR, CPC**. Cobre **todas** as campanhas que gastaram no período — inclusive as pausadas ou arquivadas depois.
2. Chame `list_ad_campaigns` para a mesma leitura **por campanha**, cada uma com o **estado atual**. Por padrão traz `ENABLED` + `PAUSED`; `ARCHIVED` só se pedirem explicitamente.
3. Para julgar **lucratividade** (e não só ACoS), veja as premissas abaixo.

## Premissas de leitura

### ACoS não é lucro

ACoS compara o investimento com a **receita** do anúncio — é cego às **tarifas da Amazon**, que comem uma fatia relevante de cada venda. Duas campanhas com o mesmo ACoS podem ter lucros opostos, dependendo da categoria e do tamanho/peso do produto.

Para avaliar lucratividade de verdade, desça ao produto: `get_product_details` + `calculate_fees` estimam a tarifa daquele item. Se o vendedor também usa o Analytics do Radar (com custo/CMV configurado), `get_profit_waterfall` traz a margem real.

### O total da conta ≥ a soma das campanhas listadas

`get_ads_overview` é **sem filtro de estado** — é o gasto real do período, incluindo campanhas arquivadas depois. `list_ad_campaigns` é um **subconjunto filtrado por estado**. Não tente reconciliar por soma: a diferença é esperada e corresponde justamente ao que está fora do filtro.

### Estado é informação, não filtro

Cada campanha vem com seu estado, e nenhuma é escondida. Uma `PAUSED` com gasto no período é um fato histórico legítimo; uma `ENABLED` que não gastou nada é sinal (lance/orçamento insuficiente, ou sem entrega). As duas importam.

### Quando o estado não pôde ser consultado

Se `state_filter_applied` vier `false`, a lista veio **sem** o filtro de estado — ou a consulta de estado falhou, ou a conta não tem Ads conectado (a nota diz qual). Nesse caso trate os estados como desconhecidos e diga isso.

## O que entregar

- Veredito do período: investimento, retorno e ACoS/ROAS, com a **janela explícita** e a ressalva dos dias recentes.
- Para onde o dinheiro está indo: as campanhas que mais gastam e, principalmente, as que **gastam sem retorno**.
- Quando pedirem decisão (cortar, subir lance, pausar), **use a estratégia do vendedor**. Sem uma declarada, exponha o trade-off em vez de inventar um playbook.

## Exemplo

> **Usuário:** "Como foram meus anúncios esse mês?"
> **Skill:** chama `get_ads_overview` (últimos 30 dias) e `list_ad_campaigns`.
> **Resposta:** "Nos últimos 30 dias: R$ 4.180 investidos e R$ 21.400 em vendas de anúncios — ACoS 19,5%, ROAS 5,1. Duas campanhas concentram 62% do investimento. A `SP_AUTO_DISCOVERY` gastou R$ 620 sem venda atribuída — vale olhar. Obs.: os ~3 dias mais recentes ainda estão amadurecendo, então o ACoS real tende a ficar um pouco melhor."

## Cuidados

- **Não trate ACoS como lucro.** Sem tarifa (e sem custo) não dá para dizer se a campanha ganha dinheiro.
- **Não conclua "caiu" olhando os últimos dias** — pode ser só atribuição imatura.
- Campanha **sem entrega** (`ENABLED` com tudo zerado) não é erro de dado: normalmente é lance ou orçamento insuficiente.
- **Somente leitura.** Estas tools não criam campanha, não pausam e não alteram lances — se pedirem execução, diga que a ação ainda é feita no console da Amazon.
- **Sem dados no período:** verifique se o Amazon Ads está conectado e se a importação já rodou (a nota da tool indica qual é o caso).
