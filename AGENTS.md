# Radar Scout — AGENTS.md

> Instruções-base para qualquer agente de código (Claude Code, Cursor, Codex, Gemini, Windsurf/Devin, VS Code) trabalhando neste repositório ou com o plugin Radar Scout.

## O que é o Radar Scout

Radar Scout é uma ferramenta para vendedores Amazon acompanharem **vendas, lucro (MC1–MC3), repasses (settlements) e repricer** da sua operação. Este repositório empacota o acesso a esses dados como um **plugin** (MCP + skills) instalável em agentes de código.

## O MCP `radarscout`

- Servidor MCP **remoto**: Streamable HTTP + OAuth 2.1 em `https://mcp.radarscout.com.br/mcp`.
- **Seller-scoped** — cada usuário enxerga apenas os próprios dados. Auth via OAuth (no Claude Code: 401 → `/mcp`).
- **Leitura + ação:** a maioria das tools é read-only; as tools de Amazon Ads, do Repricer e de criação de oferta **escrevem na conta do vendedor** (ver *Tools de ação*). O acesso a cada domínio depende do **entitlement** do plano (`radar` / `analytics` / `repricer` / `performance`) — `whoami` devolve o resumo.
- **Modo preview:** as tools de **leitura** de Analytics e Performance respondem também em preview (sem plano ativo do app), com um aviso separado do dado: é o histórico que já está na plataforma, não atualizado. Escrita em Ads sempre exige plano ativo de Performance; Radar e Repricer não têm preview. Período de teste conta como ativo.
- Nome lógico do server: `radarscout`. O nome qualificado da tool **muda por harness**; refira as tools pela **capacidade** + server lógico `radarscout`, nunca cole o namespace completo (`mcp__plugin_...`).

### Tools de leitura (por capacidade)

- **Identidade:** `whoami` — contas de seller do usuário (`seller_account_id`) + entitlements.
- **Conexão:** `connect_amazon_sp_api` — gera o link de consentimento da Amazon para vincular uma nova `SellerAccount` (não conclui a conexão sozinho).
- **Produtos (catálogo / oportunidades):** `search_products`, `get_product_details`, `list_tracked_products`.
- **Ofertas do seller (SellerListing):** `list_seller_offers`, `get_seller_offer` — estoque, FBA/FBM/DBA, Buy Box, reposição (velocidade, dias de cobertura, status, compra sugerida já descontado o que está a caminho do FBA). Nunca conflatar com catálogo.
- **Estoque e reposição** (entitlement `analytics`): `get_inventory_health` (zerados, críticos, em atenção, parados, totais disponível e a caminho), `build_replenishment_plan` (lista de compra do mais urgente ao menos, com total de unidades).
- **Tráfego por item** (entitlement `analytics`, só Amazon): `get_sales_traffic_by_item` — sessões, visualizações, conversão e Buy Box por ASIN. O histórico por item começa quando a leitura entrou no ar (`data_since`).
- **Saúde da conta** (entitlement `radar`, só Amazon): `get_account_health` — situação da conta, nota de saúde, pedidos com defeito, cancelamento pelo vendedor (meta 2,5%), atrasos, rastreio, violações de política. Responde da última leitura salva com menos de 1 dia; se não, lê a Amazon na hora (até ~40 s); `stale` = leitura antiga enquanto a Amazon gera a nova.
- **Diagnóstico de anúncio** (entitlement `radar`, só Amazon, ao vivo): `check_listing` (um anúncio: pode ser comprado e por quê não, aparece na busca, `problems` × `suggestions`, logística, estoque, conteúdo da página) e `list_listing_problems` (anúncios da conta com aviso). É a visão da Amazon — não confundir com as ofertas (números do Radar).
- **Vendas:** `get_sales_overview`, `list_top_products`.
- **Pedidos:** `list_orders` — pedidos Amazon pedido a pedido (ou item a item com `granularity:"item"`), com valor decomposto (itens, frete, desconto, imposto) para conferir contra ERP. `items_total` **não** inclui frete; `status_group` padrão `all` **inclui cancelados**; `total_orders` conta pedidos e não é comparável ao `orders` de `get_sales_overview` (itens de pedido). Sem dado pessoal do comprador.
- **Lucro:** `get_profit_waterfall`.
- **Repasses:** `get_settlement_summary`.
- **Repricer:** `get_repricer_summary`, `list_repricer_activity`, `explain_price_change`.
- **Ads** (entitlement `performance`): `get_ads_overview`, `list_ads_daily_performance`, `list_ad_campaigns`, `list_search_terms`, `list_ad_bids` — performance (investimento, vendas de anúncios, ACoS/ROAS) no total da conta, dia a dia (até 400 dias; dia sem veiculação não tem linha), por campanha (com estado atual, orçamento e **portfólio**; filtro `portfolio_id` também no overview), por termo de busca e o lance atual de cada palavra-chave/alvo (`bid_source` indica se é próprio ou herdado do grupo; com `period_start`/`period_end` traz também o desempenho de cada item).
- **Automação de Ads:** `list_ads_proposals` — fila de propostas do motor (padrão `pending`).
- **Simuladores (sem dado de seller):** `calculate_fees` (tarifas, lucro, margem, ROI para FBA/FBM/DBA), `estimate_sales_from_bsr` (unidades/mês por categoria + BSR).
- **Conceitos:** `explain_concept` — glossário Radar/Amazon (PMI, MC1–MC3, repasse, floor price, Buy Box %, safe mode, limites de alteração).

### Tools de ação (escrevem na conta do vendedor)

Repricer, entitlement `repricer` (plano ativo; só contas Amazon na v1):

- **Custo e margem:** `update_product_cost_and_margin` — grava custo e/ou margem mínima, recalcula o preço mínimo que protege a margem e envia mínimo/máximo à Amazon. Custo vindo do ERP é recusado.
- **Regra:** `set_repricer_rule` — liga/desliga o Repricer no anúncio e escolhe a automação (`available_automations`); `armed`/`missing` dizem se o Repricer vai mexer no preço.
- **Preço:** `update_price` — mesmo caminho do Publicar do painel. Só com o Repricer ligado; recusa sem mínimo/máximo e abaixo do piso de margem; com o Repricer ligado a automação pode reajustar depois.

Seguem o contrato abaixo (simulação por padrão; `execute:true` só após confirmação daquele valor) e toda resposta traz `undo` com a chamada que volta ao valor anterior.

Ofertas, entitlement `radar` (plano ativo; só contas Amazon):

- **Criar oferta em produto existente:** `create_offer` — SKU novo num ASIN do catálogo, com preço, FBA ou envio próprio. No envio próprio a oferta nasce **sempre com estoque 0**: a quantidade vem do sistema de gestão (ERP) do vendedor — o Radar não escreve estoque. A simulação confere SKU livre, se a conta pode vender o produto (ou precisa de aprovação), o preço contra a margem (custo + tarifas da Amazon) e a validação da própria Amazon, sem criar nada. Sem custo, criar exige `confirm_without_cost:true`. No FBA a Amazon pede duas declarações do vendedor (baterias; produtos perigosos) — **pergunte, nunca suponha**. A oferta aparece no Radar na sincronização diária e entra no Repricer desligada. Não há `undo` automático: desfazer é encerrar a oferta no Seller Central.

Amazon Ads, entitlement `performance` (plano ativo — preview não basta):

- **Campanha:** `pause_campaign`, `resume_campaign`, `update_campaign_budget`, `archive_campaign` (**irreversível**; exige `confirm_archive:true` junto de `execute:true`).
- **Criação de estrutura:** `create_campaign_structure` (campanha SP inteira: grupos, produtos por SKU, keywords, alvos, negativas; `campaign.portfolioId` opcional), `create_portfolio`, `create_ad_group`.
- **Incremento em grupo existente:** `add_keywords`, `add_targets`, `add_product_ads` (até 100 por chamada).
- **Lances:** `update_keyword_bid`, `update_target_bid`, `update_ad_group_default_bid` (move todo item que herda o padrão do grupo).
- **Estado de keyword/alvo:** `update_keyword_state`, `update_target_state` (`PAUSED`/`ENABLED`; não arquiva; diferente de negativar).
- **Negativação:** `create_negative_keyword`, `create_negative_target`.
- **Propostas da automação:** `approve_ads_proposal`, `reject_ads_proposal`.

Contrato invariável dessas tools:

1. **Simulação por padrão.** Sem `execute:true` nada é enviado à Amazon — a resposta `dry_run` traz o que seria enviado. Só mande `execute:true` após confirmação explícita do vendedor **daquele valor**.
2. **Ler antes de escrever.** Ids e valores atuais vêm de `list_ad_campaigns` / `list_ad_bids` / `list_search_terms`. Nunca inventar id. Id da Amazon é só número — o nome da campanha não é id, e a tool recusa. Produto anunciado é por SKU (de `list_seller_offers`); ASIN só vale para conta vendor.
3. **Proteções do motor:** mudança gradual (orçamento até 30%, lance até 50% por vez), faixas de segurança (lance R$ 0,02–500; orçamento R$ 1–100.000), período de observação de 7 dias por campanha/lance (14 dias em negativações) e teto diário de alterações. Recusas voltam em `rejected_hard_limit` / `rejected_rate_limit` / `rejected_cooldown` com o motivo e a faixa permitida.
   - Lance padrão do grupo segue as regras de lance; pausar/reativar keyword ou alvo tem espera de 7 dias.
   - **Adições** (`add_*`, `create_ad_group`) não têm espera nem delta; item repetido no grupo recusa a chamada com a lista `duplicates`; aceite parcial volta `partial: true` com `created`/`failed`.
   - **Criação de campanha:** até 10 estruturas por seller/dia; nome de campanha (ou portfólio, ou grupo na campanha) repetido é recusado. Se um passo falhar no meio, o que foi criado fica **pausado** e a resposta diz o que existe (`partial`).
4. **`override_cooldown` / `override_rate_limit`** só após uma recusa por aquele motivo **e** com confirmação explícita do vendedor.
5. **`approve_ads_proposal` é assíncrona** — o resultado final se lê em `list_ads_proposals` (`executed` / `failed` / `unknown`; `unknown` exige investigar o `auditId`).

## Skills

13 skills do vendedor em `skills/`, cada uma orquestrando os tools `radarscout` e entregando output interpretado em pt-BR:

- `inicio` — conecta (inclusive gerando o link da Amazon) e resolve o `seller_account_id`.
- `vendas` — panorama de vendas (GMV, ticket, MC3, campeões) e conferência pedido a pedido.
- `lucro` — cascata financeira (MC1–MC3, CMV, tarifas, anúncios).
- `repricer` — saúde do repricer e auditoria de mudança de preço.
- `ads` — performance de Amazon Ads (ACoS/ROAS, evolução diária, campanhas, portfólios, termos, lances) e premissas de leitura.
- `acoes-ads` — execução em Ads (pausar, orçamento, lances, negativação, criar campanha/grupo/portfólio, incluir keywords/alvos/produtos, arquivar) e fila de propostas da automação.
- `repasses` — próximo repasse e conciliação com vendas.
- `produtos` — busca de catálogo e watchlist de Buy Box.
- `ofertas` — SKUs do seller: estoque, FBA/FBM, Buy Box e reposição.
- `anuncios` — diagnóstico e qualidade do anúncio pela visão da Amazon: pode ser comprado, aparece na busca, problemas × sugestões, conteúdo da página.
- `calculadora` — simulação de tarifas/lucro/ROI e estimativa de vendas por BSR.
- `relatorio` — briefing diário/semanal narrativo.
- `glossario` — explica conceitos (sem acessar dados do seller).

Divisão de responsabilidade: as skills entregam **premissas de leitura e segurança de execução**, não playbook de estratégia. Estrutura de campanha, ACoS-alvo e política de corte são do vendedor — pergunte em vez de assumir.

## Convenção de idioma

- **Prosa e strings voltadas ao usuário (vendedor):** pt-BR.
- **Identificadores, nomes de arquivo, JSON e nomes de tool:** inglês.

## Decisões registradas

Ver `docs/adr/`. Notável: **ADR-0001** — o MCP é distribuído como servidor **remoto** (config `http` + Connector Directory), **não** como bundle `.mcpb` local.

## Agent skills

### Issue tracker
Issues e PRDs vivem no GitHub Issues deste repo (via `gh`). Ver `docs/agents/issue-tracker.md`.

### Triage labels
Cinco papéis canônicos, strings default. Ver `docs/agents/triage-labels.md`.

### Domain docs
Single-context (`CONTEXT.md` + `docs/adr/` na raiz). Ver `docs/agents/domain.md`.
