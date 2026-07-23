# Radar Scout — AGENTS.md

> Instruções-base para qualquer agente de código (Claude Code, Cursor, Codex, Gemini, Windsurf/Devin, VS Code) trabalhando neste repositório ou com o plugin Radar Scout.

## O que é o Radar Scout

Radar Scout é uma ferramenta para vendedores Amazon acompanharem **vendas, lucro (MC1–MC3), repasses (settlements) e repricer** da sua operação. Este repositório empacota o acesso a esses dados como um **plugin** (MCP + skills) instalável em agentes de código.

## O MCP `radarscout`

- Servidor MCP **remoto**: Streamable HTTP + OAuth 2.1 em `https://mcp.radarscout.com.br/mcp`.
- **Read-only**, seller-scoped — cada usuário enxerga apenas os próprios dados. Auth via OAuth (no Claude Code: 401 → `/mcp`).
- Nome lógico do server: `radarscout`. O nome qualificado da tool **muda por harness**; refira as tools pela **capacidade** + server lógico `radarscout`, nunca cole o namespace completo (`mcp__plugin_...`).

### Tools disponíveis (por capacidade)

- **Identidade:** `whoami` — contas de seller do usuário (`seller_account_id`).
- **Produtos:** `search_products`, `get_product_details`, `list_tracked_products`.
- **Vendas:** `get_sales_overview`, `list_top_products`.
- **Lucro:** `get_profit_waterfall`.
- **Repasses:** `get_settlement_summary`.
- **Repricer:** `get_repricer_summary`, `list_repricer_activity`, `explain_price_change`.
- **Ads:** `get_ads_overview`, `list_ad_campaigns` — performance de Amazon Ads (investimento, vendas de anúncios, ACoS/ROAS) no total da conta e por campanha, com o estado atual de cada uma.
- **Conceitos:** `explain_concept` — glossário Radar/Amazon (PMI, MC1–MC3, repasse, floor price, Buy Box %, safe mode).

## Skills

9 skills do vendedor em `skills/`, cada uma orquestrando os tools `radarscout` e entregando output interpretado em pt-BR:

- `inicio` — conecta e resolve o `seller_account_id`.
- `vendas` — panorama de vendas (GMV, ticket, MC3, campeões).
- `lucro` — cascata financeira (MC1–MC3, CMV, tarifas, anúncios).
- `repricer` — saúde do repricer e auditoria de mudança de preço.
- `ads` — performance de Amazon Ads (ACoS/ROAS, gasto por campanha) e premissas de leitura.
- `repasses` — próximo repasse e conciliação com vendas.
- `produtos` — busca de catálogo e watchlist de Buy Box.
- `relatorio` — briefing diário/semanal narrativo.
- `glossario` — explica conceitos (sem acessar dados do seller).

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
