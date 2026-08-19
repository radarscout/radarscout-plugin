# Radar Scout — Plugin (MCP + Skills)

Converse com seus dados do Radar Scout — **vendas, lucro (MC1–MC3), repasses, repricer e Amazon Ads** da sua operação Amazon — direto no seu agente de código. E, em Amazon Ads, **execute as mudanças** pelo próprio chat, com simulação e confirmação antes de qualquer alteração.

> **Status:** v0.2.0 — MCP remoto + 12 skills do vendedor.

## O que vem aqui

- **MCP `radarscout`** — servidor remoto (HTTP + OAuth) com 31 tools: identidade e conexão da conta, catálogo, ofertas do seller, vendas, profit waterfall, repasses, repricer, Amazon Ads (performance, termos de busca, lances), calculadora/estimativas e glossário — mais as **ações de Ads** (pausar, orçamento, lances, negativação e a fila de propostas da automação).
- **Skills** — 12 fluxos do dia a dia do vendedor, que orquestram os tools e entregam output interpretado em pt-BR:

| Skill | Para quê |
| --- | --- |
| `radarscout:inicio` | Conecta a conta Amazon e descobre a conta de seller |
| `radarscout:vendas` | Panorama de vendas (GMV, ticket, MC3, campeões) |
| `radarscout:lucro` | Cascata financeira (MC1–MC3, CMV, tarifas, anúncios) |
| `radarscout:repricer` | Saúde do repricer e "por que o preço mudou" |
| `radarscout:ads` | Performance de Amazon Ads (ACoS/ROAS, campanhas, termos, lances) |
| `radarscout:acoes-ads` | Pausar, ajustar orçamento/lance, negativar e revisar propostas |
| `radarscout:repasses` | Quando e quanto a Amazon vai repassar |
| `radarscout:produtos` | Busca de catálogo e watchlist de Buy Box |
| `radarscout:ofertas` | Seus SKUs: estoque, FBA/FBM, Buy Box e reposição |
| `radarscout:calculadora` | Tarifas, lucro e ROI de um preço; vendas estimadas por BSR |
| `radarscout:relatorio` | Briefing diário/semanal do negócio |
| `radarscout:glossario` | Explica termos (PMI, MC3, safe mode…) |

### Mudanças em Amazon Ads são sempre confirmadas

As tools de ação rodam **em simulação por padrão**: o agente mostra o de-para (valor de hoje → valor proposto) e nada é enviado à Amazon sem a sua confirmação explícita. Além disso, o Radar aplica mudança gradual (orçamento até 30%, lance até 50% por vez), faixas de segurança de valor, um período de observação entre alterações e um teto diário — as proteções só são ignoradas se você pedir.

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
acessa depende do seu plano.

## Licença

MIT — ver [LICENSE](./LICENSE).
