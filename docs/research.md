# Radar Scout — Pesquisa & Blueprint do Plugin (MCP + Skills)

> **Status:** levantamento (research). Entregável desta fase = relatório citado + blueprint do novo repositório. As skills ainda **não** são escritas aqui — a próxima fase usa a skill `skill-creator` para gerá-las no novo repo.
> **Data da pesquisa:** 2026-06-12. **Knowledge cutoff do modelo:** jan/2026 — itens marcados **[verificar]** mudaram recentemente.
> **Convenção de idioma (igual ao resto do Radar):** prosa e strings voltadas ao usuário em pt-BR; identificadores, nomes de arquivo, JSON e nomes de tool em inglês.

---

## 0. Resumo executivo (decisões)

1. **O formato é "1 MCP server + N skills empacotados num plugin".** É exatamente o que ClickHouse, MongoDB e Amplitude fazem. O plugin do Claude Code é um diretório git com `.claude-plugin/plugin.json` + um `.mcp.json` + uma pasta `skills/`, publicado via um `.claude-plugin/marketplace.json`.

2. **NÃO empacotar o MCP do Radar como `.mcpb`/`.dxt`.** Esse formato é só para servidores **locais** (stdio). O MCP do Radar é **remoto** (Streamable HTTP + OAuth 2.1). A distribuição certa de um servidor remoto é: (a) config `{type:"http", url}` no plugin, (b) submissão ao **Anthropic Connector Directory**, (c) listagem no **MCP Registry**. (Ver §4.)

3. **Skills são portáveis "de graça".** O padrão `SKILL.md` da Anthropic (Agent Skills) virou padrão de fato: **Claude Code, Cursor 2.4, Windsurf/Devin Desktop e VS Code Copilot (preview)** leem o mesmo `SKILL.md`. Escreva uma vez, distribua para os quatro. (Ver §3.)

4. **Repo dedicado e público é o padrão dos concorrentes** (Apache/MIT), separado do monorepo do produto, com manifestos por cliente (`.claude-plugin/`, `.cursor-plugin/`, `.codex-plugin/`) servindo a partir de uma única pasta `skills/`. Modelo de referência: [`mongodb/agent-skills`](https://github.com/mongodb/agent-skills). (Ver §5 e §6.)

5. **Vantagem que já temos:** o monorepo `radar` já usa `.agents/skills/` — que é justamente um dos diretórios que **Cursor e Devin Desktop varrem nativamente**. E o MCP já expõe `/.well-known/oauth-protected-resource` (RFC 9728) e `/.well-known/oauth-authorization-server`, então está bem posicionado para o fluxo de connector. (Ver §4 e §7.)

6. **Skills para o vendedor (não para devs):** ~8 skills que orquestram os tools `radarscout_*` em fluxos do dia a dia (revisão de vendas, análise de lucro, diagnóstico do repricer, conferência de repasses, relatório semanal, glossário…). Spec em §8.

---

## 1. Como funciona um plugin do Claude Code

Todas as afirmações abaixo vêm da **documentação oficial** em `code.claude.com` (espelhada em `docs.claude.com/en/docs/claude-code/*`).

### 1.1 Manifesto `plugin.json`
- Fica em **`.claude-plugin/plugin.json`** na raiz do plugin e é **opcional** — sem ele, o Claude auto-descobre componentes nos diretórios padrão e deriva o nome do diretório. [plugins-reference](https://code.claude.com/docs/en/plugins-reference)
- Único campo **obrigatório**: `name` (kebab-case, sem espaços; usado para namespacing). `displayName` (legível) existe mas exige **v2.1.143+**. [verificar]
- Campos suportados: `name`, `displayName`, `version`, `description`, `author{name,email,url}`, `homepage`, `repository`, `license`, `keywords[]`, mais campos de caminho de componentes.
- `version` é opcional; **se você definir, precisa fazer bump** para usuários receberem updates — senão o Claude usa o SHA do commit. Campos desconhecidos viram **warning, não erro** (um mesmo `plugin.json` pode coexistir com outros manifestos).
- `$schema` (aceito, ignorado em runtime): `https://json.schemastore.org/claude-code-plugin-manifest.json`.

### 1.2 O que um plugin empacota (diretórios na **raiz**, não dentro de `.claude-plugin/`)
| Componente | Local padrão | Observação |
| --- | --- | --- |
| Skills | `skills/` (cada uma = pasta com `SKILL.md`) | **soma** ao padrão |
| Slash commands | `commands/*.md` | "use `skills/` para plugins novos" |
| Subagents | `agents/*.md` | `hooks`/`mcpServers`/`permissionMode` **proibidos** em agents de plugin |
| Hooks | `hooks/hooks.json` | tipos: command, http, mcp_tool, prompt, agent |
| MCP servers | `.mcp.json` | ver §1.3 |

Fonte: [plugins-reference](https://code.claude.com/docs/en/plugins-reference). Use `${CLAUDE_PLUGIN_ROOT}` para caminhos internos; arquivos fora da pasta do plugin **não** são copiados para o cache (`~/.claude/plugins/cache`) e quebram.

### 1.3 MCP server dentro do plugin
- Declarado em **`.mcp.json`** na raiz do plugin (ou inline em `plugin.json` sob `mcpServers`). Sobe automaticamente quando o plugin é habilitado. [mcp](https://code.claude.com/docs/en/mcp)
- Transportes: **stdio** (local), **http** (remoto, recomendado; `streamable-http` é alias de `http`), **sse** (**deprecado**), **ws**.
- Forma remota: `{"type":"http","url":"https://…","headers":{…}}`. OAuth é disparado quando o servidor responde **401/403**; o usuário autentica com o comando `/mcp`; tokens guardados no keychain e renovados automaticamente.
- Nome final do tool de plugin: `mcp__plugin_<plugin>_<server>__<tool>`.

### 1.4 Marketplace e instalação
- Catálogo = **`.claude-plugin/marketplace.json`** na raiz do repo. Campos obrigatórios: `name` (kebab), `owner{name}`, `plugins[]`. [plugin-marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
- Cada entrada de plugin: mínimo `name` + `source`; extras de marketplace: `category`, `tags`, `strict`, `defaultEnabled`. Tipos de `source`: relativo (`"./plugins/x"`), `github`, `url`, `git-subdir` (monorepo, sparse clone), `npm`.
- Fluxo do usuário (verbatim na doc): `/plugin marketplace add owner/repo` → `/plugin install plugin-name@marketplace-name`. UI: `/plugin` (abas Discover/Installed/Marketplaces/Errors). [discover-plugins](https://code.claude.com/docs/en/discover-plugins)
- `settings.json`: `enabledPlugins {"plugin@marketplace": true}` e `extraKnownMarketplaces`. CLI equivalente: `claude plugin install|validate|…` e `claude plugin marketplace add|…`. Marketplaces de terceiros **não** têm auto-update por padrão (só os oficiais da Anthropic).

---

## 2. Como funcionam as Agent Skills

Fontes oficiais: [overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) · [best-practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) · [skills no Claude Code](https://code.claude.com/docs/en/skills) · [skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md).

### 2.1 O arquivo `SKILL.md`
- Toda skill = pasta com um `SKILL.md` (frontmatter YAML + corpo Markdown). **Únicos campos obrigatórios: `name` e `description`.**
- `name`: ≤64 chars, só `[a-z0-9-]`, sem tags XML, **não pode conter "anthropic" nem "claude"**.
- `description`: não-vazia, ≤1024 chars, **em terceira pessoa** (é injetada no system prompt), deve dizer **o que faz E quando usar** (o "quando" é o gatilho de auto-carregamento).
- Recomenda-se nome no **gerúndio** (ex.: `processing-pdfs`) e evitar nomes vagos (`helper`, `utils`). *(Para o Radar vamos usar nomes de tarefa em pt-BR — ver §8 e a ressalva lá.)*

### 2.2 Progressive disclosure (3 níveis)
| Nível | Quando carrega | Custo | Conteúdo |
| --- | --- | --- | --- |
| 1 — Metadata | sempre (startup) | ~100 tokens/skill | `name` + `description` |
| 2 — Instruções | quando a descrição casa | corpo do `SKILL.md` (alvo <5k tokens; **<500 linhas**) | passo a passo |
| 3 — Recursos | sob demanda | praticamente ilimitado | arquivos/`scripts/` lidos ou executados via bash |

Boas práticas: referências **um nível de profundidade** a partir do `SKILL.md`; ToC em arquivos >100 linhas; deixar explícito "**Run** `x.py`" (executar) vs "**See** `x.py`" (ler). Em Claude Code, `${CLAUDE_SKILL_DIR}` resolve o diretório da própria skill.

### 2.3 Portabilidade entre superfícies (dentro do ecossistema Claude)
Mesmo `SKILL.md` em: claude.ai (settings), Claude API (`/v1/skills`, **beta** com headers datados **[verificar]**), Claude Code (`~/.claude/skills/`, `.claude/skills/`, `<plugin>/skills/`) e Agent SDK (`settingSources`). **Skills não sincronizam automaticamente entre superfícies.** Precedência no Claude Code: enterprise > personal > project; skills de plugin têm namespace `plugin:skill` (não colidem).

### 2.4 Skill vs command vs subagent vs MCP tool
- Em Claude Code, **commands foram fundidos em skills** (skill vence se houver colisão de nome). `disable-model-invocation: true` torna a skill manual (`/nome`).
- Skill = "manual de onboarding" carregado sob demanda; **MCP tool = capacidade/conexão**. Uma skill **orquestra** tools. Se a skill usa MCP, a doc recomenda **nome totalmente qualificado** `ServerName:tool_name`. *(Nuance multi-harness em §3.4.)*

---

## 3. Distribuição multi-harness (Claude Code · Cursor · Windsurf/Devin · VS Code)

**Manchete:** o `SKILL.md` virou padrão cross-harness, e `AGENTS.md` é o padrão universal de instruções (Linux Foundation / Agentic AI Foundation).

### 3.1 SKILL.md é lido nativamente por todos
| Harness | Diretórios de skills | Status |
| --- | --- | --- |
| **Claude Code** | plugin `skills/`; `~/.claude/skills/`; `.claude/skills/` | GA |
| **Cursor (2.4+)** | `.cursor/skills/`, **`.agents/skills/`** (+ globais `~/.cursor/skills/`, `~/.agents/skills/`) | GA — [cursor.com/docs/skills](https://cursor.com/docs/skills) |
| **Windsurf → Devin Desktop** | `.windsurf/skills/`; Devin Desktop também varre **`.agents/skills/`** e `.claude/skills/` | rebrand **02/jun/2026**; Cascade vira legado [verificar] |
| **VS Code Copilot** | `.github/skills/`, `.claude/skills/` (back-compat) | **PREVIEW**, gated por `chat.useAgentSkills` [verificar] |

> **Implicação prática:** `.agents/skills/` (que o monorepo `radar` já usa) é lido por Cursor **e** Devin Desktop; `.claude/skills/` cobre Claude + VS Code + Devin. Mantendo a fonte canônica em `skills/` no repo novo e copiando para `.agents/skills/`, pega-se o maior conjunto nativo.
> Divergências: token de invocação explícita difere — Cursor/Copilot `/skill`, Windsurf `@skill`. VS Code está em preview/gated (não confiar como default ainda).

### 3.2 Config do MCP remoto difere por harness (mesma URL, chaves diferentes)
| Harness | Arquivo | Chave | Campo da URL | OAuth |
| --- | --- | --- | --- | --- |
| Claude Code | `.mcp.json` (plugin) ou `claude mcp add --transport http` | `mcpServers` | `type:"http"` + `url` | 401→`/mcp` |
| Cursor | `.cursor/mcp.json` (proj) / `~/.cursor/mcp.json` | `mcpServers` | `url` | MCP OAuth |
| Windsurf/Devin | `~/.codeium/windsurf/mcp_config.json` | `mcpServers` | **`serverUrl`** | MCP OAuth |
| VS Code Copilot | `.vscode/mcp.json` | **`servers`** | `type:"http"` + `url` | MCP OAuth |

Fontes: [Cursor MCP](https://cursor.com/docs/context/mcp) · [Windsurf MCP](https://docs.windsurf.com/windsurf/cascade/mcp) · [VS Code MCP](https://code.visualstudio.com/docs/agent-customization/mcp-servers).

### 3.3 Deeplinks "Add to …" (instalação em 1 clique)
- **Cursor:** `cursor://anysphere.cursor-deeplink/mcp/install?name=$NAME&config=$BASE64` (config = JSON do `mcp.json` em base64); gerador de badge em [install-links](https://cursor.com/docs/context/mcp/install-links).
- **VS Code:** `vscode:mcp/install?{json-urlencoded}` (Insiders: `vscode-insiders:…`).
- **Windsurf/Devin:** `windsurf://windsurf-mcp-registry?serverName=<nome>` (abre a página do registry; menos flexível — exige listagem).
- **Claude:** não há deeplink equivalente; o "1 clique" do Claude é a **listagem no Directory** + `claude mcp add` / `/plugin install`.

### 3.4 Nuance: skills que chamam tools MCP em múltiplos harnesses
O nome qualificado do tool **muda por harness** (em plugin do Claude vira `mcp__plugin_<plugin>_<server>__<tool>`). Para manter as skills portáveis, **descreva o tool pela capacidade/propósito e pelo nome lógico do server (`radarscout`)**, não cole o namespace completo no corpo da skill. Assim a mesma skill funciona quando o agente resolve o tool correto em cada harness.

### 3.5 Fallback universal: `AGENTS.md`
Um `AGENTS.md` na raiz é lido nativamente por Cursor, Copilot, Codex, Gemini CLI, Aider, Windsurf/Devin, Zed etc. Bom para instruções-base onde `SKILL.md` ainda não está habilitado (ex.: VS Code sem o flag).

---

## 4. Distribuir o MCP **remoto** (HTTP + OAuth) para usuários finais

### 4.1 `.mcpb` (ex-`.dxt`) **não serve** — é só local
O bundle `.mcpb` empacota um **servidor local + `manifest.json`** ([modelcontextprotocol/mcpb](https://github.com/modelcontextprotocol/mcpb)). Para um servidor remoto OAuth como o do Radar, o caminho é config `http` + diretórios/registries. (Só usaríamos `.mcpb` se publicássemos um proxy stdio local — não recomendado.)

### 4.2 Caminhos de instalação no Claude
- **Claude Code (CLI):** `claude mcp add --transport http radarscout https://mcp.radarscout.com.br/mcp`. [mcp](https://code.claude.com/docs/en/mcp)
- **Claude Desktop / claude.ai:** Settings → **Connectors** → "+" → nome + URL. Connectors adicionados no claude.ai **aparecem automaticamente no Claude Code**.
- ⚠️ **O Claude conecta a partir da nuvem da Anthropic, não do device do usuário** — o endpoint precisa ser público e alcançável pelos IPs da Anthropic (sem VPN/firewall fechado). [custom connectors](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)

### 4.3 OAuth: o que o distribuidor precisa expor (para "conectar" sem fricção)
Fluxo MCP: 401 → **Protected Resource Metadata (RFC 9728)** em `/.well-known/oauth-protected-resource` → descobre o auth server → registra (DCR ou CIMD) → OAuth 2.1 + PKCE → bearer token.
- O **authorization server precisa servir sua própria metadata** (RFC 8414 ou OIDC Discovery) em `/.well-known/`.
- **[verificar — mudança recente]** A Anthropic passou a recomendar **CIMD** (Client ID Metadata Documents — usa uma URL HTTPS como `client_id`) **em vez de DCR**. O Claude só escolhe CIMD se a metadata anunciar `client_id_metadata_document_supported: true` **e** `"none"` em `token_endpoint_auth_methods_supported`; senão cai para **DCR** (RFC 7591). [CIMD post (2025-08-22)](https://blog.modelcontextprotocol.io/posts/client_registration/) · [Connectors auth](https://claude.com/docs/connectors/building/authentication)
- ⚠️ **[verificar]** Há bugs de OAuth abertos no Claude Code em 2026 (token não repassado após consent; "discovery poisoning"). **Testar o connect ao vivo** na versão atual.

### 4.4 Descoberta / registries
- **MCP Registry** (registry.modelcontextprotocol.io, **preview**): publica `server.json` com array `remotes:[{type:"streamable-http", url, headers}]` via CLI `mcp-publisher`. [registry](https://modelcontextprotocol.io/registry/remote-servers)
- **Anthropic Connector Directory** (claude.ai/directory): conectores revisados; submissão pelo portal admin (org Team/Enterprise). É o "1 clique" do ecossistema Claude. [submission guide](https://support.claude.com/en/articles/12922490-remote-mcp-server-submission-guide)

---

## 5. Prior art — como concorrentes empacotam (com exemplos reais)

| Vendor | O que envia | Read-only/governado | Skills? | Fonte |
| --- | --- | --- | --- | --- |
| **Snowflake** | MCP gerenciado: Cortex Analyst (NL→SQL), roda sob o role do usuário | governado por role | — | [blog](https://www.snowflake.com/en/blog/managed-mcp-servers-secure-data-agents/) |
| **Amplitude** | Plugin: MCP + **26 skills** (analista completo, incl. **briefings** diário/semanal) | — | **sim (26)** | [amplitude/mcp-marketplace](https://github.com/amplitude/mcp-marketplace) |
| **ClickHouse** | Plugin: MCP **read-only** (SQL) + skills de best-practice | **read-only** | sim | [clickhouse-claude-code-plugin](https://github.com/ClickHouse/clickhouse-claude-code-plugin) |
| **MongoDB** | **1 repo → Claude+Cursor+Codex+Gemini** (MCP + skills compartilhadas) | — | sim (7-8) | [mongodb/agent-skills](https://github.com/mongodb/agent-skills) |
| **Stripe** | MCP remoto `mcp.stripe.com` (OAuth); read-only via restricted keys | opcional read-only | — | [docs.stripe.com/mcp](https://docs.stripe.com/mcp) |
| **Sentry** | MCP remoto `mcp.sentry.dev` (OAuth) + plugin com sub-agent | — | sim | [mcp.sentry.dev](https://mcp.sentry.dev/) |

**Padrão vencedor recorrente:** repo público dedicado (Apache/MIT) com **um MCP server read-only/governado + N skills estreitas e nomeadas por tarefa (incluindo skills de "explicar/resumir")**, com `marketplace.json` por cliente, listado no Directory oficial. Anthropic mantém o índice ([`anthropics/claude-plugins-official`](https://github.com/anthropics/claude-plugins-official/blob/main/.claude-plugin/marketplace.json), 200+ plugins) apontando para esses repos via `git-subdir`/`url`.

**Lições de design para skills de analytics (do Amplitude/ClickHouse/Snowflake):**
1. Read-only/governado por padrão é a feature-manchete (segurança).
2. A `description` codifica o gatilho ("use quando o usuário…").
3. **Uma skill por tarefa de analista**, não uma mega-skill — progressive disclosure mantém o contexto pequeno.
4. **Skills de "explicar/resumir" ao lado dos tools de query** — usuários querem output interpretado (resumos, explicação de métrica), não linhas cruas. ↔ casa perfeitamente com `explain_price_change`, `get_sales_overview`, `get_profit_waterfall` do Radar.
5. MCP + skills no **mesmo plugin**.

---

## 6. Blueprint do novo repositório

Sugestão: repo público `radarscout/radar-scout-plugin` (ou `radar-scout-skills`), licença MIT/Apache-2.0, espelhando o layout do `mongodb/agent-skills` (verificado).

```
radar-scout-plugin/
├── .claude-plugin/
│   ├── marketplace.json          # catálogo (name, owner, plugins[])
│   └── plugin.json               # manifesto do plugin radar-scout
├── .cursor-plugin/               # manifesto Cursor (opcional, alcance extra)
│   └── plugin.json
├── .codex-plugin/                # manifesto Codex (opcional)
│   └── plugin.json
├── .mcp.json                     # MCP remoto: radarscout @ https://mcp.radarscout.com.br/mcp
├── skills/                       # FONTE CANÔNICA das skills (SKILL.md cross-harness)
│   ├── revisao-de-vendas/SKILL.md
│   ├── analise-de-lucro/SKILL.md
│   ├── diagnostico-do-repricer/SKILL.md
│   ├── conferencia-de-repasses/SKILL.md
│   ├── monitor-de-produtos/SKILL.md
│   ├── relatorio-semanal/SKILL.md
│   ├── primeiros-passos/SKILL.md
│   └── glossario-radar/SKILL.md
├── AGENTS.md                     # fallback universal de instruções
├── README.md                     # snippets de instalação por harness + badges/deeplinks
├── LICENSE
└── .github/workflows/validate.yml  # `claude plugin validate --strict` no CI
```

### 6.1 `.claude-plugin/marketplace.json`
```json
{
  "$schema": "https://json.schemastore.org/claude-code-marketplace.json",
  "name": "radar-scout",
  "description": "Plugin oficial do Radar Scout — converse com seus dados de vendas, lucro e repricer da Amazon.",
  "owner": { "name": "Radar Scout", "email": "suporte@radarscout.com.br" },
  "plugins": [
    {
      "name": "radar-scout",
      "source": ".",
      "description": "Skills + MCP para analisar vendas, lucro (MC1–MC3), repasses e repricer do Radar Scout.",
      "category": "analytics",
      "tags": ["amazon", "seller", "vendas", "lucro", "repricer", "repasses"]
    }
  ]
}
```

### 6.2 `.claude-plugin/plugin.json`
```json
{
  "$schema": "https://json.schemastore.org/claude-code-plugin-manifest.json",
  "name": "radar-scout",
  "displayName": "Radar Scout",
  "version": "0.1.0",
  "description": "Converse com seus dados do Radar Scout: vendas, lucro, repasses e repricer da sua operação Amazon.",
  "author": { "name": "Radar Scout", "url": "https://radarscout.com.br" },
  "homepage": "https://radarscout.com.br",
  "repository": "https://github.com/radarscout/radar-scout-plugin",
  "license": "MIT",
  "keywords": ["amazon", "seller", "analytics", "repricer", "vendas", "lucro", "repasses"]
}
```
> `skills/` e `.mcp.json` são auto-descobertos na raiz do plugin — não precisam ser listados no manifesto. `displayName` exige Claude Code **v2.1.143+** [verificar].

### 6.3 `.mcp.json`
```json
{
  "mcpServers": {
    "radarscout": {
      "type": "http",
      "url": "https://mcp.radarscout.com.br/mcp"
    }
  }
}
```
> Sem `headers`: a auth é OAuth (401 → `/mcp`). `mcp.radarscout.com.br` é a URL canônica em `apps/mcp/env.example` (`NEXT_PUBLIC_MCP_URL`).

### 6.4 Snippets de instalação para o README (por harness)
```bash
# Claude Code — marketplace + plugin (traz MCP + skills de uma vez)
/plugin marketplace add radarscout/radar-scout-plugin
/plugin install radar-scout@radar-scout
# … ou só o MCP:
claude mcp add --transport http radarscout https://mcp.radarscout.com.br/mcp
```
```jsonc
// Cursor — .cursor/mcp.json
{ "mcpServers": { "radarscout": { "url": "https://mcp.radarscout.com.br/mcp" } } }
```
```jsonc
// Windsurf / Devin Desktop — ~/.codeium/windsurf/mcp_config.json
{ "mcpServers": { "radarscout": { "serverUrl": "https://mcp.radarscout.com.br/mcp" } } }
```
```jsonc
// VS Code Copilot — .vscode/mcp.json
{ "servers": { "radarscout": { "type": "http", "url": "https://mcp.radarscout.com.br/mcp" } } }
```
> Skills (fora do plugin): instruir o usuário a copiar `skills/` para `.agents/skills/` (Cursor/Devin) ou `.claude/skills/` (Claude/VS Code), ou simplesmente instalar o plugin no Claude Code.

---

## 7. Prontidão do `apps/mcp` atual para distribuição

Já implementado (favorável):
- `/.well-known/oauth-protected-resource` (RFC 9728) e `/.well-known/oauth-authorization-server` — descoberta OK.
- Streamable HTTP em `/mcp`, read-only, seller-scoped, audience binding (RFC 8707), scope `radar:read`.
- 12 tools `radarscout_*` (ver §8).

A validar antes de submeter ao Directory **[verificar com o time/Clerk]**:
1. **CIMD vs DCR:** o auth server (Clerk) anuncia `client_id_metadata_document_supported` + `"none"`? Se não, garantir que **DCR** está habilitado para o connect zero-config.
2. **Alcance pela nuvem da Anthropic:** `mcp.radarscout.com.br` público, sem bloqueio de IP.
3. **Teste do fluxo de consent** ao vivo no Claude Desktop + Claude Code atuais (há bugs de OAuth abertos em 2026).
4. Texto do consent screen (nome do client + hostname do redirect) apresentável ao vendedor.

---

## 8. Skills a criar (spec — próxima fase com `skill-creator`)

Público: **vendedor Amazon** consultando seus dados via MCP. Cada skill orquestra tools `radarscout_*` e entrega **output interpretado em pt-BR** (não linhas cruas). Ressalva de naming: a doc recomenda gerúndio em inglês; como é produto pt-BR e o `name` vira o slash command do usuário, proponho **nomes de tarefa em pt-BR** (kebab-case, sem "claude"/"anthropic"). Descrições em **terceira pessoa** com gatilho ("Use quando…").

Tools disponíveis: `whoami`, `search_products`, `get_product_details`, `list_tracked_products`, `get_sales_overview`, `list_top_products`, `get_profit_waterfall`, `get_settlement_summary`, `get_repricer_summary`, `list_repricer_activity`, `explain_price_change`, `explain_concept`.

| Skill (`name`) | `description` (rascunho, 3ª pessoa + gatilho) | Tools orquestrados |
| --- | --- | --- |
| `revisao-de-vendas` | "Resume o desempenho de vendas de um período (GMV, pedidos, ticket médio, lucro MC3) e os produtos campeões. Use quando o usuário perguntar como foram as vendas, pedir um panorama de faturamento/desempenho ou comparar períodos." | whoami, get_sales_overview, list_top_products |
| `analise-de-lucro` | "Detalha a composição do lucro de um produto ou da operação via profit waterfall (receita → MC1 → MC2 → MC3, taxas e repasse). Use quando o usuário perguntar por que o lucro caiu, quiser entender margem/custos ou pedir o detalhamento de repasse." | whoami, get_profit_waterfall, get_product_details, explain_concept |
| `diagnostico-do-repricer` | "Avalia a saúde do repricer: atividade recente, vitórias/perdas de Buy Box, cooldown e safe mode, e explica mudanças de preço. Use quando o usuário perguntar se o repricer está funcionando, por que um preço mudou ou quiser revisar a reprecificação." | whoami, get_repricer_summary, list_repricer_activity, explain_price_change, explain_concept |
| `conferencia-de-repasses` | "Resume os repasses (settlements) da Amazon de um período e concilia com as vendas. Use quando o usuário perguntar quanto vai receber, sobre repasses/settlements ou a diferença entre vendas e valor repassado." | whoami, get_settlement_summary, get_sales_overview, explain_concept |
| `monitor-de-produtos` | "Encontra produtos monitorados e mostra detalhes (preço, Buy Box %, estoque, status) de um SKU/ASIN. Use quando o usuário quiser localizar um produto, ver detalhes de um item ou listar o que está sendo rastreado." | whoami, search_products, get_product_details, list_tracked_products |
| `relatorio-semanal` | "Gera um briefing diário/semanal do negócio (vendas, lucro, top produtos, alertas do repricer e repasses) em formato narrativo. Use quando o usuário pedir um relatório semanal/diário, um resumo geral ou um 'como está meu negócio'." | whoami, get_sales_overview, list_top_products, get_profit_waterfall, get_repricer_summary, get_settlement_summary |
| `primeiros-passos` | "Ajuda o usuário a conectar o Radar e descobrir suas contas de seller (seller_account_id). Use na primeira interação, quando o usuário não souber qual conta usar ou disser que não vê seus dados." | whoami |
| `glossario-radar` | "Explica conceitos do Radar e da operação Amazon (PMI, MC1–MC3, repasse, floor price, Buy Box %, safe mode). Use quando o usuário perguntar o que significa um termo, sigla ou métrica." | explain_concept |

**Opcionais futuras:** `comparar-periodos` (período vs período sobre `get_sales_overview`), `alertas-de-margem` (varre top produtos com MC3 abaixo de um piso). Considerar uma skill `disable-model-invocation:true` para relatórios disparados manualmente (`/relatorio-semanal`).

Cada `SKILL.md` deve: manter corpo <500 linhas; referir tools pela capacidade + server lógico `radarscout` (não colar o namespace, §3.4); incluir 1–2 exemplos; e, na fase de autoria, seguir o loop do `skill-creator` (capturar intenção → escrever → criar 3+ casos de avaliação → rodar com/sem skill → ajustar a `description`).

---

## 9. Riscos / itens em fluxo (a revalidar antes de publicar)
- **VS Code Agent Skills = preview**, gated por `chat.useAgentSkills` — não é default. Para VS Code, oferecer também `AGENTS.md` / prompt files como fallback.
- **Windsurf → Devin Desktop** (rebrand 02/jun/2026; Cascade virando legado). Paths `.windsurf/` ainda valem, mas `.agents/skills/` é a aposta mais segura.
- **CIMD vs DCR** e **bugs de OAuth do Claude Code (2026)** — testar o connect ao vivo.
- **Campos versionados:** `displayName` (v2.1.143+), `defaultEnabled` (v2.1.154+) — confirmar versão mínima alvo.
- **Marketplaces de terceiros não auto-atualizam** — documentar `/plugin marketplace update` para os usuários, e fazer **bump de `version`** a cada release.
- Vários docs oficiais (cursor.com, code.visualstudio.com, docs.windsurf.com) bloquearam fetch direto (403); citações desses três vêm de extratos de busca das **mesmas páginas oficiais** — alta confiança, mas não fetch primário limpo.

---

## 10. Próximos passos sugeridos
1. **Criar o repo** `radarscout/radar-scout-plugin` com o layout da §6 (posso scaffoldar).
2. **Escrever as skills** da §8 com `skill-creator` (uma a uma, com avaliações), iterando contra o MCP real (`Radar_Scout` já está conectado nesta sessão para testes ponta a ponta).
3. **Preparar a distribuição do MCP:** validar CIMD/DCR + alcance público (§7), publicar no MCP Registry e submeter ao Anthropic Connector Directory.
4. **README com badges/deeplinks** por harness (§6.4) e `AGENTS.md` (§3.5).
5. **CI** com `claude plugin validate --strict`.

---

## 11. Fontes (deduplicadas)

**Claude Code — plugins & skills**
- https://code.claude.com/docs/en/plugins-reference
- https://code.claude.com/docs/en/plugin-marketplaces
- https://code.claude.com/docs/en/discover-plugins
- https://code.claude.com/docs/en/mcp
- https://code.claude.com/docs/en/skills
- https://code.claude.com/docs/en/settings
- https://code.claude.com/docs/en/agent-sdk/skills

**Agent Skills (Anthropic)**
- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- https://github.com/anthropics/skills · skill-creator: https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md
- https://www.anthropic.com/news/skills · https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills

**MCP remoto / OAuth / distribuição**
- https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization
- https://blog.modelcontextprotocol.io/posts/client_registration/ (CIMD, 2025-08-22)
- https://claude.com/docs/connectors/building/authentication
- https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp
- https://support.claude.com/en/articles/12922490-remote-mcp-server-submission-guide
- https://modelcontextprotocol.io/registry/remote-servers · https://github.com/modelcontextprotocol/registry
- https://github.com/modelcontextprotocol/mcpb (`.mcpb`/ex-`.dxt`, local-only)

**Multi-harness**
- Cursor: https://cursor.com/docs/skills · https://cursor.com/docs/context/mcp · https://cursor.com/docs/context/mcp/install-links · https://cursor.com/docs/context/rules · https://cursor.com/changelog/2-4
- Windsurf/Devin: https://docs.windsurf.com/windsurf/cascade/skills · https://docs.windsurf.com/windsurf/cascade/mcp · https://docs.windsurf.com/windsurf/cascade/workflows · https://docs.windsurf.com/windsurf/cascade/agents-md
- VS Code: https://code.visualstudio.com/docs/agent-customization/agent-skills · https://code.visualstudio.com/docs/agent-customization/mcp-servers · https://code.visualstudio.com/docs/agent-customization/prompt-files · https://code.visualstudio.com/updates/v1_108

**Prior art**
- https://github.com/anthropics/claude-plugins-official/blob/main/.claude-plugin/marketplace.json
- https://github.com/mongodb/agent-skills · https://github.com/amplitude/mcp-marketplace · https://github.com/ClickHouse/clickhouse-claude-code-plugin · https://github.com/PostHog/ai-plugin · https://github.com/getsentry/sentry-mcp
- https://docs.stripe.com/mcp · https://github.com/github/github-mcp-server · https://www.snowflake.com/en/blog/managed-mcp-servers-secure-data-agents/ · https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp