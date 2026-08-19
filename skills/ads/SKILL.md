---
name: ads
description: "Analisa a performance de Amazon Ads de uma conta de seller — investimento, vendas de anúncios, ACoS/ROAS, cliques e conversão, no total da conta, por campanha (com o estado atual de cada uma), por termo de busca e por lance atual de palavra-chave/alvo. Use quando o usuário perguntar como estão os anúncios, quanto gastou em Ads, qual o ACoS, quais campanhas gastam sem vender, quais termos de busca dispararam os anúncios (e quais gastam sem converter), ou quanto está pagando por clique em cada palavra-chave."
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
3. Para descer ao **termo de busca** — o que o cliente realmente digitou —, chame `list_search_terms` (opcionalmente filtrando por campanha). Veja a seção **Termos de busca**.
4. Para ver **quanto se está pagando** em cada palavra-chave ou alvo, chame `list_ad_bids`. Veja a seção **Lances atuais**.
5. Para julgar **lucratividade** (e não só ACoS), veja as premissas abaixo.

Quando o vendedor decidir **agir** (pausar, mudar lance ou orçamento, negativar), passe para a skill `acoes-ads` — as ferramentas de execução estão lá, com simulação e confirmação obrigatórias.

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

## Termos de busca

`list_search_terms` mostra os **termos que o cliente realmente buscou** e que acionaram os anúncios, cada um com o **alvo que o capturou** e o **tipo** desse alvo (`keyword_type`) — que é a primeira coisa a ler:

- **`BROAD` / `PHRASE` / `EXACT`** — o termo veio de uma **palavra-chave que o vendedor cadastrou**.
- **`TARGETING_EXPRESSION` / `TARGETING_EXPRESSION_PREDEFINED`** — veio de **segmentação automática ou por produto** (a Amazon escolheu onde mostrar).

A distinção muda a conclusão: um termo que converte sob **segmentação automática** é candidato natural a **virar palavra-chave própria**; o mesmo termo já sob uma **keyword exata** não é — já está cadastrado. **Leia o `keyword_type` antes de decidir qualquer coisa** sobre o termo.

Três premissas ao ler termos:

- **Atribuição vale aqui também.** O termo só ganha significado com o período somado, e os ~3 dias recentes vêm imaturos. Nunca conclua "esse termo não converte" — muito menos o negative — olhando poucos dias.
- **Fee-aware, não ACoS.** Um termo "lucrativo" pelo ACoS pode não ser depois da tarifa da Amazon. Antes de chamar um termo de bom, desça ao produto (`get_product_details` + `calculate_fees`, ou `get_profit_waterfall` se houver custo configurado).
- **Volume antes de veredito.** Termo com cliques e **sem** venda não é automaticamente desperdício: veja se houve **cliques suficientes** para concluir. Poucos cliques são ruído, não sinal.

Os filtros da tool (`min_clicks`, `min_cost`, `has_sales`, `keyword_types`) são **primitivas de consulta neutras** — recortam a leitura, não são um limiar de decisão. O que fazer com um termo (negativar, colher para keyword, subir lance) é **estratégia do vendedor**, não da skill.

## Lances atuais

`list_ad_bids` lista palavras-chave e alvos de produto/categoria com o **lance que a Amazon usa nos leilões agora**. Em conta grande, filtre por `campaign_id` (ou `ad_group_id`); `type` recorta entre `keyword`, `target` ou ambos.

O campo decisivo é o **`bid_source`**:

- **`próprio`** — o item tem lance definido nele mesmo.
- **`padrão do grupo`** — o item **não tem lance próprio** e está herdando o padrão do grupo de anúncios. Mudar o lance desse item cria um lance próprio e o desliga do padrão do grupo — e mexer no padrão do grupo mudaria **todos** os itens que ainda herdam. Diga qual dos dois o vendedor quer antes de propor um número.

Os ids que vêm aqui (`keyword_id` / `target_id`) são exatamente os que as ferramentas de mudança de lance recebem — leia daqui, nunca invente.

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
- **Não negative um termo em cima de janela imatura nem de poucos cliques** — e cheque o `keyword_type` antes (colher faz sentido no automático, não numa keyword já exata).
- Campanha **sem entrega** (`ENABLED` com tudo zerado) não é erro de dado: normalmente é lance ou orçamento insuficiente.
- **Esta skill é de leitura.** Ela não pausa campanha, não muda lance nem orçamento — quem executa é a skill `acoes-ads`, sempre com simulação e confirmação do vendedor antes.
- **Sem dados no período:** verifique se o Amazon Ads está conectado e se a importação já rodou (a nota da tool indica qual é o caso).
