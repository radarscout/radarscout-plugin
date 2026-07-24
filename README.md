# Radar Scout — Plugin (MCP + Skills)

Converse com seus dados do Radar Scout — **vendas, lucro (MC1–MC3), repasses, repricer e Amazon Ads** da sua operação Amazon — direto no seu agente de código.

> **Status:** v0.1.0 — MCP remoto + 9 skills do vendedor.

## O que vem aqui

- **MCP `radarscout`** — servidor remoto read-only (HTTP + OAuth) com 14 tools: `whoami`, busca/detalhe de produtos, panorama de vendas, profit waterfall, repasses, repricer, Amazon Ads e glossário.
- **Skills** — 9 fluxos do dia a dia do vendedor, que orquestram os tools e entregam output interpretado em pt-BR:

| Skill | Para quê |
| --- | --- |
| `radarscout:inicio` | Conecta e descobre a conta de seller |
| `radarscout:vendas` | Panorama de vendas (GMV, ticket, MC3, campeões) |
| `radarscout:lucro` | Cascata financeira (MC1–MC3, CMV, tarifas, anúncios) |
| `radarscout:repricer` | Saúde do repricer e "por que o preço mudou" |
| `radarscout:ads` | Performance de Amazon Ads (ACoS/ROAS, gasto por campanha) |
| `radarscout:repasses` | Quando e quanto a Amazon vai repassar |
| `radarscout:produtos` | Busca de catálogo e watchlist de Buy Box |
| `radarscout:relatorio` | Briefing diário/semanal do negócio |
| `radarscout:glossario` | Explica termos (PMI, MC3, safe mode…) |

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

As 9 skills são instaláveis em 70+ agentes (Claude Code, Cursor, Copilot, Windsurf, Gemini, Codex…) via [skills.sh](https://www.skills.sh):

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
read-only e seller-scoped: cada usuário enxerga apenas os próprios dados.

## Licença

MIT — ver [LICENSE](./LICENSE).
