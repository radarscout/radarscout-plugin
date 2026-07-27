---
name: ads
description: "Analisa a performance de Amazon Ads de uma conta de seller — investimento, vendas de anúncios, ACoS/ROAS, cliques e conversão, no total da conta, por campanha (com o estado atual de cada uma) e por termo de busca — e executa alterações com confirmação: pausar/reativar campanha, ajustar lance e orçamento, negativar termos, e revisar/aprovar propostas do motor de automação. Use quando o usuário perguntar como estão os anúncios, quanto gastou em Ads, qual o ACoS, quais campanhas gastam sem vender, quais termos dispararam os anúncios, quiser revisar o desempenho — ou pedir para pausar, mudar lance/orçamento, negativar um termo ou revisar propostas de automação."
---

# Performance de Amazon Ads

Responde duas perguntas: **"quanto investi e o que voltou?"** (conta) e **"quais campanhas puxam e quais queimam?"** (por campanha) — e, quando o vendedor decidir agir, executa a alteração com segurança (ver **Alterações na conta**).

Esta skill dá **premissas de leitura e de execução**, não uma estratégia de campanha. Estruturas e táticas variam por operação — pergunte a do vendedor em vez de assumir uma.

## Conta e período

- **Conta:** exige `seller_account_id` (via `whoami` do `radarscout`).
- **Período:** ISO `YYYY-MM-DD`, **fim exclusivo**. Sem período, use os **últimos 30 dias** (Ads precisa de volume para o número significar algo) e diga a janela.
- **Atribuição:** as conversões de Ads amadurecem ao longo da janela de atribuição — os **~3 dias mais recentes costumam vir subestimados**. As tools devolvem exatamente o período pedido e sinalizam isso em `attribution_note`; **desconte os dias recentes ao decidir** e não confunda com queda real.

## Passo a passo

1. Chame `get_ads_overview` do `radarscout` com a janela: investimento, vendas de anúncios, impressões, cliques, pedidos e os derivados **ACoS, ROAS, CTR, CVR, CPC**. Cobre **todas** as campanhas que gastaram no período — inclusive as pausadas ou arquivadas depois.
2. Chame `list_ad_campaigns` para a mesma leitura **por campanha**, cada uma com o **estado atual**. Por padrão traz `ENABLED` + `PAUSED`; `ARCHIVED` só se pedirem explicitamente.
3. Para descer ao **termo de busca** — o que o cliente realmente digitou —, chame `list_search_terms` (opcionalmente filtrando por campanha). Veja a seção **Termos de busca**.
4. Para julgar **lucratividade** (e não só ACoS), veja as premissas abaixo.

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

## Veiculação zero: campanha ENABLED sem impressões

Uma campanha `ENABLED` com investimento e impressões zerados não é "sem dados" — é um veredito que precisa de diagnóstico, na ordem certa. **Nunca assuma a causa** antes de checar os sinais abaixo, nessa ordem:

1. **Primeiro, `data_freshness`.** Antes de julgar a campanha, cheque se os dados chegaram: `get_ads_overview` traz `data_freshness` (`last_ingested_at`, `latest_window_end`, `latest_window_records`) e `freshness_note`. Importação parada ou ausente é conversa de **pipeline**, não de campanha; importação recente com zero linhas confirma que a campanha realmente não veiculou.
2. **Depois, o `serving_status` da campanha.** `list_ad_campaigns` devolve o veredito da Amazon no nível da campanha (ex.: orçamento esgotado, falha de pagamento, início pendente). Um status desses já explica o silêncio sem precisar olhar os anúncios.
3. **Depois, `ad_serving_summary.serving_status_counts` por anúncio.** Para campanhas `ENABLED` com zero impressões, a tool também devolve a contagem de status por anúncio — é aqui que aparecem problemas de estoque e elegibilidade. Status diferentes levam a conversas diferentes: sem estoque é reposição, orçamento é dimensionamento, suspensão é política — e nenhum desses é ajuste de Ads.
4. **Todos os status saudáveis e zero exibições.** Se os anúncios estão todos elegíveis mas ainda assim não exibiram, o problema é **competitividade de lance/segmentação** — estão perdendo todos os leilões. Nenhum status vai apontar isso; não force uma causa "com cara de status" quando na verdade é disputa de leilão.
5. **`ENABLED` ≠ elegível.** O estado da campanha é a intenção do vendedor; o `serving_status` (campanha ou anúncio) é o veredito da Amazon — os dois podem divergir. Se `ad_serving_summary.truncated` vier `true`, as contagens são um **piso**, não o total (a lista de anúncios foi cortada). Ao conversar com o vendedor, use linguagem simples — "status de veiculação", "não exibiu anúncios" — nunca o jargão bruto da API.

## Alterações na conta (write tools)

Além da leitura, existem tools de **alteração**: pausar/reativar campanha (`pause_campaign`, `resume_campaign`), ajustar orçamento (`update_campaign_budget`), ajustar lances (`update_keyword_bid`, `update_target_bid`) e negativar (`create_negative_keyword`, `create_negative_target`). Os limites exatos de cada uma (faixas de valores, variação máxima por passo, período de espera) estão nas próprias descriptions das tools — não os repita de memória.

1. **Simule primeiro, sempre.** Toda tool de alteração aceita `execute: false` (o padrão): monta o pedido e valida sem mudar nada na Amazon. Mostre ao vendedor o que mudaria e obtenha **confirmação explícita** antes de repetir com `execute: true`.
2. **Valores absolutos, nunca delta.** As tools recebem o valor final (`bid: 1.20`), não "aumente 10%". Ao ouvir um pedido relativo, calcule sobre o valor atual — o orçamento atual vem em `daily_budget` no `list_ad_campaigns`; para lance atual de keyword/target ainda não há leitura direta (uma simulação rejeitada por variação revela o valor em `previous`).
3. **Rejeição é conversa, não erro.** `rejected_hard_limit` → a resposta traz `allowedRange`; proponha um valor dentro da faixa. `rejected_cooldown` → existe um período de espera entre alterações na mesma campanha/keyword (`daysRemaining` diz quanto falta). `rejected_rate_limit` → limite diário de segurança de alterações da conta. Explique o motivo em linguagem simples e **nunca tente contornar por conta própria**.
4. **Forçar é decisão do vendedor, nunca sua.** As flags `override_cooldown`/`override_rate_limit` só entram depois de uma rejeição, com o motivo explicado e a confirmação explícita do vendedor.
5. **Propostas do motor de automação** (`list_ads_proposals` → revisar → `approve_ads_proposal`/`reject_ads_proposal`): proposta com mais de ~24h pode não refletir o estado atual — confira antes de aprovar. Status `unknown` = desfecho não confirmado pelo sistema; investigue (via `auditId`) antes de tratar como feito. Ao rejeitar, registre o motivo em `reason`.
6. **Rastro.** Toda tentativa de alteração (inclusive simulação e rejeição) gera um `auditId` — cite-o ao reportar o que foi feito.
7. **Linguagem com o vendedor:** "simulação", "período de espera", "faixa permitida", "limite diário" — nunca o jargão bruto da API (dry-run, cooldown, rate limit).

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
- Campanha **sem entrega** (`ENABLED` com tudo zerado) não é erro de dado nem tem causa única presumida — siga a ordem de diagnóstico em **Veiculação zero** (`data_freshness` → `serving_status` da campanha → `serving_status_counts` dos anúncios → competitividade de lance/segmentação).
- **Alterar exige o fluxo de Alterações na conta**: simulação, confirmação explícita do vendedor e só então execução — nunca execute direto. **Criar campanha** ainda não é possível por aqui; essa, sim, é feita no console da Amazon.
- **Sem dados no período:** verifique se o Amazon Ads está conectado e se a importação já rodou (a nota da tool indica qual é o caso).
