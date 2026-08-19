# Radar Scout — AGENTS.md

> Instruções-base para qualquer agente de código (Claude Code, Cursor, Codex, Gemini, Windsurf/Devin, VS Code) trabalhando neste repositório ou com o plugin Radar Scout.

## O que é o Radar Scout

Radar Scout é uma ferramenta para vendedores Amazon acompanharem **vendas, lucro (MC1–MC3), repasses (settlements) e repricer** da sua operação. Este repositório empacota o acesso a esses dados como um **plugin** (MCP + skills) instalável em agentes de código.

## O MCP `radarscout`

- Servidor MCP **remoto**: Streamable HTTP + OAuth 2.1 em `https://mcp.radarscout.com.br/mcp`.
- **Seller-scoped** — cada usuário enxerga apenas os próprios dados. Auth via OAuth (no Claude Code: 401 → `/mcp`).
- **Leitura + ação:** a maioria das tools é read-only; as tools de Amazon Ads **escrevem na conta do vendedor** (ver *Tools de ação*). O acesso a cada domínio depende do **entitlement** do plano (`radar` / `analytics` / `repricer`) — `whoami` devolve o resumo.
- Nome lógico do server: `radarscout`. O nome qualificado da tool **muda por harness**; refira as tools pela **capacidade** + server lógico `radarscout`, nunca cole o namespace completo (`mcp__plugin_...`).

### Tools de leitura (por capacidade)

- **Identidade:** `whoami` — contas de seller do usuário (`seller_account_id`) + entitlements.
- **Conexão:** `connect_amazon_sp_api` — gera o link de consentimento da Amazon para vincular uma nova `SellerAccount` (não conclui a conexão sozinho).
- **Produtos (catálogo / oportunidades):** `search_products`, `get_product_details`, `list_tracked_products`.
- **Ofertas do seller (SellerListing):** `list_seller_offers`, `get_seller_offer` — estoque, FBA/FBM/DBA, Buy Box, reposição (velocidade, dias de cobertura, status). Nunca conflatar com catálogo.
- **Vendas:** `get_sales_overview`, `list_top_products`.
- **Lucro:** `get_profit_waterfall`.
- **Repasses:** `get_settlement_summary`.
- **Repricer:** `get_repricer_summary`, `list_repricer_activity`, `explain_price_change`.
- **Ads:** `get_ads_overview`, `list_ad_campaigns`, `list_search_terms`, `list_ad_bids` — performance (investimento, vendas de anúncios, ACoS/ROAS) no total da conta, por campanha (com estado atual), por termo de busca e o lance atual de cada palavra-chave/alvo (`bid_source` indica se é próprio ou herdado do grupo).
- **Automação de Ads:** `list_ads_proposals` — fila de propostas do motor (padrão `pending`).
- **Simuladores (sem dado de seller):** `calculate_fees` (tarifas, lucro, margem, ROI para FBA/FBM/DBA), `estimate_sales_from_bsr` (unidades/mês por categoria + BSR).
- **Conceitos:** `explain_concept` — glossário Radar/Amazon (PMI, MC1–MC3, repasse, floor price, Buy Box %, safe mode, limites de alteração).

### Tools de ação (escrevem na conta do vendedor)

Amazon Ads, entitlement `analytics`:

- **Campanha:** `pause_campaign`, `resume_campaign`, `update_campaign_budget`.
- **Lances:** `update_keyword_bid`, `update_target_bid`.
- **Negativação:** `create_negative_keyword`, `create_negative_target`.
- **Propostas da automação:** `approve_ads_proposal`, `reject_ads_proposal`.

Contrato invariável dessas tools:

1. **Simulação por padrão.** Sem `execute:true` nada é enviado à Amazon — a resposta `dry_run` traz o que seria enviado. Só mande `execute:true` após confirmação explícita do vendedor **daquele valor**.
2. **Ler antes de escrever.** Ids e valores atuais vêm de `list_ad_campaigns` / `list_ad_bids` / `list_search_terms`. Nunca inventar id.
3. **Proteções do motor:** mudança gradual (orçamento até 30%, lance até 50% por vez), faixas de segurança (lance R$ 0,02–500; orçamento R$ 1–100.000), período de observação de 7 dias por campanha/lance (14 dias em negativações) e teto diário de alterações. Recusas voltam em `rejected_hard_limit` / `rejected_rate_limit` / `rejected_cooldown` com o motivo e a faixa permitida.
4. **`override_cooldown` / `override_rate_limit`** só após uma recusa por aquele motivo **e** com confirmação explícita do vendedor.
5. **`approve_ads_proposal` é assíncrona** — o resultado final se lê em `list_ads_proposals` (`executed` / `failed` / `unknown`; `unknown` exige investigar o `auditId`).

## Skills

12 skills do vendedor em `skills/`, cada uma orquestrando os tools `radarscout` e entregando output interpretado em pt-BR:

- `inicio` — conecta (inclusive gerando o link da Amazon) e resolve o `seller_account_id`.
- `vendas` — panorama de vendas (GMV, ticket, MC3, campeões).
- `lucro` — cascata financeira (MC1–MC3, CMV, tarifas, anúncios).
- `repricer` — saúde do repricer e auditoria de mudança de preço.
- `ads` — performance de Amazon Ads (ACoS/ROAS, campanhas, termos, lances) e premissas de leitura.
- `acoes-ads` — execução em Ads (pausar, orçamento, lances, negativação) e fila de propostas da automação.
- `repasses` — próximo repasse e conciliação com vendas.
- `produtos` — busca de catálogo e watchlist de Buy Box.
- `ofertas` — SKUs do seller: estoque, FBA/FBM, Buy Box e reposição.
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
