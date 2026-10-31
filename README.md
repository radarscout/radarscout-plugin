# Radar Scout — Plugin (MCP + Skills)

Converse com seus dados do Radar Scout — **vendas, pedidos, lucro (MC1–MC3), repasses, repricer e Amazon Ads** da sua operação Amazon — direto no seu agente de código. E, em Amazon Ads, **gerencie as campanhas** pelo próprio chat — de ajustes de lance a campanhas novas —, com simulação e confirmação antes de qualquer alteração.

> **Status:** v0.3.0 — MCP remoto com 43 ferramentas (leitura + ações de Ads) e 12 skills do vendedor.

## O que vem aqui

- **MCP `radarscout`** — servidor remoto (HTTP + OAuth) em `https://mcp.radarscout.com.br/mcp` com **43 ferramentas**: 23 de leitura (identidade e conexão da conta, catálogo, ofertas do seller, vendas, pedidos, Cascata de Lucro, repasses, repricer, Amazon Ads, calculadora/estimativas e glossário) e 20 **ações de Amazon Ads** (pausar/reativar/arquivar campanha, orçamento, lances, estado de palavra-chave/alvo, negativação, criação de campanha, grupo e portfólio, inclusão de palavras-chave/alvos/produtos e a fila de propostas da automação). A lista completa está em [Ferramentas do MCP](#ferramentas-do-mcp).
- **Skills** — 12 fluxos do dia a dia do vendedor, que orquestram as ferramentas e entregam output interpretado em pt-BR:

| Skill | Para quê |
| --- | --- |
| `radarscout:inicio` | Conecta a conta Amazon e descobre a conta de seller |
| `radarscout:vendas` | Panorama de vendas (GMV, ticket, MC3, campeões) e pedidos para conferir com o ERP |
| `radarscout:lucro` | Cascata financeira (MC1–MC3, CMV, tarifas, anúncios) |
| `radarscout:repricer` | Saúde do repricer e "por que o preço mudou" |
| `radarscout:ads` | Performance de Amazon Ads (ACoS/ROAS, evolução diária, portfólios, campanhas, termos, lances) |
| `radarscout:acoes-ads` | Ajustar, pausar, negativar, criar campanhas/grupos/portfólios e revisar propostas |
| `radarscout:repasses` | Quando e quanto a Amazon vai repassar |
| `radarscout:produtos` | Busca de catálogo e watchlist de Buy Box |
| `radarscout:ofertas` | Seus SKUs: estoque, FBA/FBM, Buy Box e reposição |
| `radarscout:calculadora` | Tarifas, lucro e ROI de um preço; vendas estimadas por BSR |
| `radarscout:relatorio` | Briefing diário/semanal do negócio |
| `radarscout:glossario` | Explica termos (PMI, MC3, safe mode…) |

### Mudanças em Amazon Ads são sempre confirmadas

O MCP vai além de consultas: as 20 ferramentas de ação **mudam seus anúncios na Amazon**. Por isso elas rodam **em simulação por padrão**: o agente mostra o de-para (valor de hoje → valor proposto) e nada é enviado à Amazon sem a sua **confirmação explícita** — tecnicamente, a chamada só aplica com `execute:true`, que o agente envia apenas depois que você confirma aquele valor. Além disso, o Radar aplica mudança gradual (orçamento até 30%, lance até 50% por vez), faixas de segurança de valor, um período de observação entre alterações e um teto diário (inclusive de até 10 campanhas criadas por dia) — as proteções só são ignoradas se você pedir. Arquivar campanha, que não tem volta na Amazon, pede uma confirmação a mais. As ações exigem o app Performance ativo e a conta de anúncios conectada; sem isso, o agente avisa.

## Ferramentas do MCP

Você não precisa decorar nada disto: pergunte em português e o agente escolhe a ferramenta. A descrição em linguagem de vendedor, com exemplos do que pedir, está em [docs.radarscout.com.br/docs/ai-assistant/tools](https://docs.radarscout.com.br/docs/ai-assistant/tools) — é a fonte de verdade desta lista. As ferramentas que leem dados de uma conta pedem o `seller_account_id`, que o agente descobre sozinho com `radarscout_whoami`.

### Leitura (23)

| Assunto | Ferramentas |
| --- | --- |
| Identidade & conexão | `radarscout_whoami`, `radarscout_connect_amazon_sp_api` |
| Produtos & descoberta | `radarscout_search_products`, `radarscout_get_product_details`, `radarscout_list_top_products`, `radarscout_list_tracked_products`, `radarscout_estimate_sales_from_bsr` |
| Suas ofertas | `radarscout_list_seller_offers`, `radarscout_get_seller_offer` |
| Financeiro | `radarscout_get_sales_overview`, `radarscout_get_profit_waterfall`, `radarscout_get_settlement_summary`, `radarscout_list_orders`, `radarscout_calculate_fees` |
| Amazon Ads — leitura | `radarscout_get_ads_overview`, `radarscout_list_ads_daily_performance`, `radarscout_list_ad_campaigns`, `radarscout_list_ad_bids`, `radarscout_list_search_terms`, `radarscout_list_ads_proposals` |
| Repricer | `radarscout_get_repricer_summary`, `radarscout_list_repricer_activity`, `radarscout_explain_price_change` |
| Conceitos | `radarscout_explain_concept` |

### Ações em Amazon Ads (20) — simulação por padrão, aplicam só com `execute:true`

| Assunto | Ferramentas |
| --- | --- |
| Campanha | `radarscout_pause_campaign`, `radarscout_resume_campaign`, `radarscout_update_campaign_budget`, `radarscout_archive_campaign` (irreversível; pede confirmação extra) |
| Criação de estrutura | `radarscout_create_campaign_structure`, `radarscout_create_portfolio`, `radarscout_create_ad_group` |
| Incremento em grupo existente | `radarscout_add_keywords`, `radarscout_add_targets`, `radarscout_add_product_ads` |
| Lances | `radarscout_update_keyword_bid`, `radarscout_update_target_bid`, `radarscout_update_ad_group_default_bid` |
| Estado de palavra-chave / alvo | `radarscout_update_keyword_state`, `radarscout_update_target_state` |
| Negativação | `radarscout_create_negative_keyword`, `radarscout_create_negative_target` |
| Propostas do Scout AI | `radarscout_approve_ads_proposal`, `radarscout_reject_ads_proposal` |

Contas do **Mercado Livre** aparecem na lista de contas e podem ser consultadas no Repricer. Vendas, lucro, repasses, ofertas, busca de produtos e anúncios são, por enquanto, só da **Amazon**.

## Instalação

### Claude Code (plugin = MCP + skills de uma vez)

```bash
/plugin marketplace add radarscout/radarscout-plugin
/plugin install radarscout@radarscout
```

…ou só o MCP:

```bash
claude mcp add --transport http radarscout https://mcp.radarscout.com.br/mcp
```

### Claude Desktop / claude.ai

Settings → **Connectors** → "+" → nome `radarscout`, URL `https://mcp.radarscout.com.br/mcp`.
Connectors adicionados no claude.ai aparecem automaticamente no Claude Code.

### Cursor — `.cursor/mcp.json`

```jsonc
{ "mcpServers": { "radarscout": { "url": "https://mcp.radarscout.com.br/mcp" } } }
```

### Windsurf / Devin Desktop — `~/.codeium/windsurf/mcp_config.json`

```jsonc
{ "mcpServers": { "radarscout": { "serverUrl": "https://mcp.radarscout.com.br/mcp" } } }
```

### VS Code Copilot — `.vscode/mcp.json`

```jsonc
{ "servers": { "radarscout": { "type": "http", "url": "https://mcp.radarscout.com.br/mcp" } } }
```

### Skills em qualquer agente — skills.sh (`npx skills`)

As 12 skills são instaláveis em 70+ agentes (Claude Code, Cursor, Copilot, Windsurf, Gemini, Codex…) via [skills.sh](https://www.skills.sh):

```bash
# todas as skills do repo
npx skills add radarscout/radarscout-plugin

# só algumas, num agente específico, sem prompt interativo
npx skills add radarscout/radarscout-plugin --skill vendas --skill lucro -a claude-code -y

# experimentar uma sem instalar
npx skills use radarscout/radarscout-plugin@relatorio | claude
```

> ⚠️ **`npx skills` instala só as skills** (os `SKILL.md` de `skills/`). O **MCP `radarscout` é configurado à parte** (seções acima, por harness) — sem ele conectado, as skills não têm de onde ler os dados.
> Requer o repositório **público** (o `add` clona via GitHub).

O primeiro acesso dispara o login OAuth (no Claude Code: comando `/mcp`). O MCP é
seller-scoped: cada usuário enxerga apenas os próprios dados, e o que cada conta
acessa depende do seu plano (Radar, Analytics, Repricer, Performance). Em modo preview,
Analytics e Performance respondem à leitura do histórico; ações em Ads exigem plano ativo.

## Licença

MIT — ver [LICENSE](./LICENSE).
