# Credencial headless para agentes autônomos no MCP `radarscout`

> Pesquisa para a issue radarscout/radarscout-plugin#13. Data da coleta: 2026-11-07.
> Fontes: apenas documentação oficial / spec / referência first-party. Tudo que não foi possível confirmar em fonte primária está marcado **[não verificado]**.
> Escopo: **levantar opções**, não decidir. A decisão de identidade do Agente fica para um ticket posterior, com humano.

## Contexto

- O MCP `radarscout` é remoto (Streamable HTTP + OAuth 2.1), com Clerk como authorization server (DCR + refresh via `offline_access`), sem scopes granulares hoje e controle por audience binding (RFC 8707).
- A spec MCP 2025-11-25 exige que o cliente envie `resource` (RFC 8707) e que o servidor valide que o token foi emitido para ele como audience — <https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization> (seções "Resource Parameter Implementation" e "Token Audience Binding and Validation").
- Objetivo: um agente autônomo (primeiro alvo Grok/xAI, mas agnóstico) operando ~50 contas de seller de um mesmo usuário **sem clique humano por sessão**.

---

## 1. Lado cliente: o que cada cliente autônomo aceita

### Tabela cliente × mecanismo

Legenda: ✅ suportado e documentado · ❌ documentado como não suportado · ⚠️ parcial/beta/condicionado · — não documentado

| Cliente | OAuth interativo c/ refresh persistido pelo cliente | Bearer estático (`Authorization`) | Header custom | `client_credentials` | Quem renova o token | Fonte |
|---|---|---|---|---|---|---|
| **xAI API — Remote MCP tools** (native SDK, Responses compatível, Voice Agent API) | — (não mencionado) | ✅ campo `authorization` | ✅ `headers` (`extra_headers` no SDK nativo) | — | **O integrador** (token é passado a cada request; doc não fala de refresh) | <https://docs.x.ai/developers/tools/remote-mcp> |
| **Grok app — Connectors custom** (grok.com/connectors) | ⚠️ conectores built-in/catálogo usam OAuth; para custom a doc diz só "complete any required authentication" | — | — | — | — | <https://docs.x.ai/grok/connectors> |
| **Grok Tasks / Automations** (agendado) | **[não verificado]** — nenhuma doc oficial encontrada sobre uso de conectores custom em execuções agendadas | — | — | — | — | (só fontes secundárias) |
| **OpenAI Responses API — tool `mcp`** | ❌ OpenAI não guarda nem renova; "OAuth client registration and authorization must be handled separately by your application" | ✅ campo `authorization` (não armazenado; reenviar a cada request) | — (doc só documenta `authorization`) | — | **O integrador** | <https://developers.openai.com/api/docs/guides/tools-connectors-mcp> |
| **OpenAI Agents SDK** (`MCPServerStreamableHttp`) | ⚠️ aceita `params["auth"]` (objeto `httpx.Auth`); OAuth/refresh não documentados na página — usar o provider OAuth do SDK MCP seria inferência **[não verificado]** | ✅ `params["headers"]` | ✅ `params["headers"]` | — | **O integrador** | <https://openai.github.io/openai-agents-python/mcp/> |
| **ChatGPT (Apps SDK / connectors)** | ✅ OAuth 2.1 auth code + PKCE S256; CIMD (preferido), DCR ou cliente pré-definido | ❌ não apresenta "custom API keys" | — | ❌ "ChatGPT does **not** support machine-to-machine OAuth grants" | ChatGPT (doc diz que tokens "can still expire or rotate normally"; onde/como renova não é detalhado) | <https://developers.openai.com/apps-sdk/build/auth> |
| **Anthropic Messages API — MCP connector** | ❌ "API consumers are expected to handle the OAuth flow ... and to refresh the token as needed" | ✅ `authorization_token` | — | — | **O integrador** | <https://platform.claude.com/docs/en/agents-and-tools/mcp-connector> |
| **Claude Managed Agents** (vaults) | ✅ credencial `mcp_oauth` com bloco `refresh` (`token_endpoint`, `client_id`, `scope`, `refresh_token`, `token_endpoint_auth`) — **Anthropic renova**; o 1º token o integrador obtém | ✅ credencial `static_bearer` | — | — | **Anthropic** (com `refresh`); falha emite `vault_credential.refresh_failed` | <https://platform.claude.com/docs/en/managed-agents/vaults> |
| **claude.ai / Desktop / Cowork / Claude Code — connectors** | ✅ `oauth_dcr`, `oauth_cimd`, `oauth_anthropic_creds`; Claude renova reativamente no 401 e proativamente até 5 min antes do vencimento; pede `offline_access` se anunciado | ⚠️ `static_headers` (Owner da org cola API key/bearer) — **beta, conjunto limitado de orgs**; credencial é da org, não do usuário | ⚠️ mesmo `static_headers` (`authorization`, `x-api-key`, `x-auth-token`; outros nomes precisam de aprovação) | ❌ "A machine-to-machine `client_credentials` grant isn't supported" | **Claude** | <https://claude.com/docs/connectors/building/authentication> |
| **Claude scheduled tasks (ex-routines)** | ✅ usam os connectors da conta claude.ai do usuário; rodam autonomamente na nuvem e agem "as you" | herda dos connectors | herda | herda (❌) | **Claude** (mesma infra de connectors) | <https://code.claude.com/docs/en/routines> |
| **n8n — MCP Client Tool** | ✅ credencial MCP OAuth2 (DCR ligado por padrão; ou client id/secret manual; Resource URL opcional/auto-descoberta) | ✅ Bearer | ✅ generic header / multiple headers | ⚠️ a credencial OAuth2 genérica do n8n lista "Client Credentials grant type"; se a credencial MCP OAuth2 aceita esse grant **[não verificado]** | n8n — refresh automático **[não verificado]** em doc oficial | <https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/>, <https://docs.n8n.io/integrations/builtin/credentials/mcp/>, <https://docs.n8n.io/integrations/builtin/credentials/httprequest/> |

### Observações por cliente

- **xAI/Grok (API).** O único mecanismo documentado é token estático em `authorization` ou `headers`; `connector_id` e `require_approval` "are not currently supported" na Responses API compatível. Não há OAuth gerenciado pela xAI — se o token expira, quem chama a API precisa obter outro. <https://docs.x.ai/developers/tools/remote-mcp>
- **Grok (app).** Conectores built-in "authenticate via OAuth"; para custom MCP o texto é genérico. Em Grok Business/Enterprise "a team admin must first provision a connector in the cloud console". Uso de conectores em Tasks/Automations: **[não verificado]**. <https://docs.x.ai/grok/connectors>
- **OpenAI Responses.** "the Responses API does not store the value you provide in the `authorization` field" → o integrador reenvia o token a cada `responses.create`. <https://developers.openai.com/api/docs/guides/tools-connectors-mcp>
- **ChatGPT.** Além de não aceitar M2M nem API key custom, apresenta certificado mTLS da OpenAI e IPs de egress publicados para identificar o cliente. <https://developers.openai.com/apps-sdk/build/auth>
- **Anthropic Managed Agents.** Vault = conjunto de credenciais de um usuário final, referenciado por sessão via `vault_ids`. **Uma credencial ativa por `mcp_server_url` por vault** (duplicata → 409); entre vaults, "the first vault with a match wins". Ou seja, 50 contas de seller atrás da **mesma URL** não podem ter 50 credenciais distintas no mesmo vault — ou um token cobre as 50 contas, ou são vaults/sessões separados. `refresh.token_endpoint` não pode ser alterado depois de criado. Valores sensíveis são write-only. <https://platform.claude.com/docs/en/managed-agents/vaults>
- **Claude connectors.** Requisitos de servidor relevantes: retornar `invalid_grant` quando refresh token não vale mais; para clientes públicos (DCR/CIMD) a spec MCP exige rotacionar ou sender-constrain refresh tokens; endpoint de token tem 10 s (troca) / 30 s (refresh). Existe também "Enterprise Managed Auth" (auto-connect sem consent por usuário) — fora do escopo desta leitura. <https://claude.com/docs/connectors/building/authentication>
- **Claude scheduled tasks.** "Anything a scheduled task does through your connected GitHub identity or connectors appears as you." O token de API trigger é só para disparar a tarefa, não para o MCP. <https://code.claude.com/docs/en/routines>

### Leitura transversal

- **Denominador comum dos clientes de API (xAI, OpenAI Responses, Anthropic Messages, Agents SDK, n8n):** um **bearer** que o integrador fornece. Nenhum deles faz o OAuth/refresh por conta própria, exceto Managed Agents (com bloco `refresh`) e n8n (OAuth2 credential).
- **Clientes "de consumo" (ChatGPT, claude.ai, Grok app):** OAuth interativo com refresh mantido pelo cliente; `client_credentials` explicitamente **não** suportado em ChatGPT e Claude.
- **`client_credentials` no MCP:** existe a extensão oficial opcional `io.modelcontextprotocol/oauth-client-credentials` (client secret ou JWT assertion RFC 7523), mas nenhum dos clientes acima documenta suporte a ela. <https://modelcontextprotocol.io/extensions/auth/oauth-client-credentials>, <https://modelcontextprotocol.io/extensions/auth/overview>

---

## 2. Lado servidor: o que o Clerk oferece hoje para auth de máquina/agente

Visão geral oficial: três tipos de token de máquina — OAuth access tokens, M2M tokens e API keys. <https://clerk.com/docs/guides/development/machine-auth/overview>

### 2.1 OAuth (o que já usamos)

- Grants anunciados: `authorization_code`, `refresh_token` e, quando disponível, `urn:ietf:params:oauth:grant-type:device_code`. <https://clerk.com/docs/guides/configure/auth-strategies/oauth/how-clerk-implements-oauth>
- **`client_credentials`: não suportado.** "Clerk does not yet support it, but we are aiming to add support for it in the near future." <https://clerk.com/docs/guides/development/machine-auth/overview>
- **Device Authorization Grant (RFC 8628)** — lançado 2026-09-08; mesmo scopes/tokens/consent/claims de org do auth code; cliente público ou confidencial. Ainda exige um humano aprovar uma vez num navegador. <https://clerk.com/changelog/2026-09-08-device-authorization-grant>
- **Custom OAuth scopes** — lançado 2026-08-21 (ex.: `messages:read`, `tools:execute`); atribuídos por OAuth application, anunciáveis no metadata; o enforcement é do nosso servidor verificando os scopes do token. Apagar um scope não revoga tokens já emitidos; refresh token com scope apagado falha com `invalid_scope`. <https://clerk.com/changelog/2026-08-21-custom-oauth-scopes>, <https://clerk.com/docs/guides/configure/auth-strategies/oauth/how-clerk-implements-oauth>
- **Default scopes para DCR/CIMD** configuráveis (clientes como ChatGPT e Claude podem omitir `scope`). <https://clerk.com/docs/guides/configure/auth-strategies/oauth/how-clerk-implements-oauth>
- **Step-up de scopes** para MCP (`requireAdditionalScopes()`). <https://clerk.com/docs/nextjs/guides/ai/mcp/step-up-scopes.md> (título/descrição vistos no índice `https://clerk.com/docs/llms.txt`; conteúdo **[não verificado]**)
- **Organizations no OAuth:** scope `user:org:read` → seletor de org no consent e claim `org_id` no access token. <https://clerk.com/changelog/2026-05-14-oauth-organizations>
- **Vida útil:** access token **1 dia**; refresh token **nunca expira**; authorization code 10 min; device/user code 10 min. <https://clerk.com/docs/guides/configure/auth-strategies/oauth/how-clerk-implements-oauth>
  - Nota: a página "scoped access" ainda mostra `expires_in: 7200` num exemplo — conflito entre páginas; a página "how Clerk implements OAuth" é a mais específica. <https://clerk.com/docs/guides/configure/auth-strategies/oauth/scoped-access>
  - Rotação de refresh token pelo Clerk: **[não verificado]** (não descrita).
- **Formato e revogação:** JWT (padrão, sem revogação imediata) ou **opaco** (verificação via rede, "can be revoked immediately"). Endpoint de revogação/listagem de consentimentos por usuário: **[não verificado]**. <https://clerk.com/docs/guides/development/machine-auth/token-formats>
- **Consent:** obrigatório quando DCR está ligado. <https://clerk.com/docs/guides/configure/auth-strategies/oauth/how-clerk-implements-oauth>
- **CIMD** GA desde 2026-09-17. <https://clerk.com/changelog/2026-09-17-client-id-metadata-documents-ga>

### 2.2 API keys (GA desde 2026-04-06)

<https://clerk.com/docs/guides/development/machine-auth/api-keys>, <https://clerk.com/changelog/2026-04-17-api-keys-ga>

- Criadas pelo usuário (aba **API Keys** em `<UserProfile />` / `<OrganizationProfile />` ou componente `<APIKeys />`) ou pelo backend (`apiKeys.create()`).
- `subject` obrigatório: `user_xxx` **ou** `org_xxx` — a chave age em nome do usuário/org.
- `scopes` (array de strings livres) e `claims` (objeto livre) por chave → **restrição por chave é possível, mas o significado e o enforcement são do nosso servidor** (ex.: guardar em `claims` a lista de `seller_account_id` permitidos). A doc sugere usar scopes/claims para rejeitar chaves criadas pelo usuário via Frontend API quando só chaves emitidas pelo backend devem valer.
- Expiração: "By default, API keys never expire"; opcional `secondsUntilExpiration`.
- Revogação individual imediata (opacas). Segredo exibido uma única vez.
- Desligar a feature no Dashboard faz as chaves falharem na verificação sem revogá-las.
- Verificação no servidor: `acceptsToken` aceita array, p.ex. `['session_token', 'oauth_token', 'api_key']` — o mesmo endpoint pode aceitar OAuth e API key. <https://clerk.com/docs/expressjs/guides/development/verifying-api-keys.md>
- Preço: 1.000 criações/mês grátis, depois US$ 0,001; 100.000 verificações/mês grátis, depois US$ 0,00001. Plano Hobby limitado ao free tier.
- Audience/resource binding (RFC 8707) em API keys: não mencionado — **[não verificado]**.

### 2.3 M2M tokens

<https://clerk.com/docs/guides/development/machine-auth/m2m-tokens>

- Para "requests between different backend services within your own infrastructure". Cada máquina tem um secret; token criado com `m2m.createToken()`.
- `subject` = máquina (`mch_...`); `scopes` = **ids de outras máquinas** que ela pode chamar (não são permissões de domínio). Sem vínculo a usuário.
- `claims` opcional; expiração opcional (padrão: não expira); formato opaco (revogável) ou JWT (não revogável, não listável).
- Preço: 2.500 criações/mês grátis, depois US$ 0,001; verificação US$ 0,00001 após 100.000.
- Encaixe: serve para serviço-nosso→serviço-nosso, não para "usuário delega a agente de terceiro".

### 2.4 Novidades de agente (fora do caminho principal)

- **Clerk for Agent Auth** — private beta (2026-10-30), baseado em OAuth + Personal Agent Protocol. Capacidades concretas: **[não verificado]**. <https://clerk.com/changelog/2026-10-30-clerk-for-agent-auth>
- **Browser sessions for agents** (2026-11-06): link de uso único que abre o app logado como o usuário — voltado a agentes com navegador, não a MCP headless. <https://clerk.com/changelog/2026-11-06-agent-browser-sessions>

### Resumo do que o Clerk habilita

| Necessidade | Clerk hoje |
|---|---|
| Token sem humano nenhum (`client_credentials`) | ❌ não suportado (roadmap) |
| Um consentimento humano, depois refresh indefinido | ✅ OAuth auth code ou device grant + `offline_access`; refresh nunca expira |
| Credencial estática por usuário para colar em cliente | ✅ API key (`subject=user_`), sem expiração por padrão, revogável |
| Scopes granulares | ✅ custom OAuth scopes; scopes livres em API keys (enforcement nosso) |
| Restrição por recurso (ex.: subset de seller accounts) | ⚠️ só via `claims`/`scopes` livres + enforcement nosso; em OAuth, `org_id` por org |
| Revogação individual | ✅ API key; ✅ OAuth opaco; ❌ JWT (só expira) |
| Expiração configurável | ✅ API key/M2M (`secondsUntilExpiration`); OAuth fixo (1 dia / refresh sem expiração) |

---

## 3. Padrões de mercado (servidores MCP remotos multi-tenant)

| Servidor | Interativo | Headless / agente | Como restringe | Fonte |
|---|---|---|---|---|
| **Stripe** (`mcp.stripe.com`) | OAuth (escolhe contas/sandboxes e permissões por ambiente; sessões revogáveis pelo usuário e por admin) | **Agent API key** = restricted key marcada como de agente, passada como `Authorization: Bearer`. A partir de dez/2026 o MCP deixa de aceitar secret key full-access e restricted key sem tag Agent | Permissões da restricted key + **approval rules** automáticas para ações sensíveis (payouts, refunds…) + "Human confirmation" por ação. Para Connect: OAuth **não** age em connected accounts → agent key da plataforma + header `Stripe-Account: acct_...` | <https://docs.stripe.com/mcp>, <https://docs.stripe.com/keys#agent-keys> |
| **Linear** (`mcp.linear.app/mcp`) | OAuth 2.1 + DCR | OAuth token ou **API key** direto em `Authorization: Bearer` (inclui agir como usuário `app`) | Endpoint `/mcp/readonly`; scope OAuth `read`; API key só com permissão `Read`. Cada workspace = contexto de auth próprio | <https://linear.app/docs/mcp> |
| **GitHub** (`api.githubcopilot.com/mcp/`) | OAuth (só scopes aprovados; políticas de org) | **PAT** em `Authorization: Bearer`; recomendado fine-grained | Fine-grained PAT: **um resource owner por token**, seleção de repos, permissões granulares, expiração (políticas podem limitar), aprovação pela org, revogação; headers `X-MCP-Toolsets`, `X-MCP-Tools`, `X-MCP-Readonly` | <https://github.com/github/github-mcp-server/blob/main/docs/remote-server.md>, <https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp-in-your-ide/set-up-the-github-mcp-server>, <https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens> |
| **Sentry** (`mcp.sentry.dev`) | "All connections use OAuth" | Sem caminho headless documentado | Escopo por **URL**: `/mcp/{org}` e `/mcp/{org}/{project}` | <https://mcp.sentry.dev/> |
| **Shopify** (Customer Accounts MCP) | OAuth auth code + PKCE, descoberto via `/.well-known/customer-account-api` | — | Scopes do app + requisitos de dados protegidos | <https://shopify.dev/docs/apps/build/customer-accounts-mcp> (Storefront MCP sem auth: só fonte secundária — **[não verificado]**) |

Padrões recorrentes:

1. **Dupla via**: OAuth para cliente interativo + **chave estática com escopo** para agente/cliente sem OAuth (Stripe, Linear, GitHub).
2. **A chave para agente é um tipo próprio e mais restrito** que a chave "admin" (Stripe agent key; GitHub fine-grained PAT; Linear key read-only).
3. **Restrição por recurso** aparece de três formas: no token (GitHub: 1 owner + repos), na URL (Sentry org/projeto; Linear `/readonly`) ou em header por chamada (Stripe `Stripe-Account`; GitHub `X-MCP-*`).
4. **Governança de escrita** separada da autenticação: approval rules / human confirmation (Stripe).
5. **Revogação visível ao usuário**: lista de sessões OAuth / tokens revogáveis (Stripe, GitHub).

---

## 4. Implicações para a decisão de identidade do Agente

> Opções e trade-offs apenas. **Não é decisão** — fica para humano em ticket posterior.

**Opção A — Manter só OAuth; o operador do agente faz um consentimento inicial e persiste o refresh token.**
- Funciona hoje com Clerk (refresh nunca expira; device grant evita redirect para bots sem navegador).
- Claude Managed Agents renova sozinho (`mcp_oauth` + `refresh`); claude.ai/scheduled tasks também. Para xAI, OpenAI Responses, Anthropic Messages e Agents SDK, **o integrador precisa rodar o refresh** e reenviar o access token (1 dia) — mais código do lado do cliente.
- Refresh token sem expiração é credencial de longa duração de fato; rotação pelo Clerk **[não verificado]**. Sem scopes granulares hoje, o token tem acesso total do usuário (custom scopes existem no Clerk e poderiam ser adotados).

**Opção B — Aceitar Clerk API keys (`subject=user_…`) como bearer estático, além de OAuth.**
- Maior compatibilidade: é exatamente o que xAI (`authorization`/`headers`), OpenAI Responses, Anthropic Messages/`static_bearer`, Agents SDK e n8n aceitam sem código extra. Alinha com Stripe/Linear/GitHub.
- Restrição por chave via `scopes`/`claims` (ex.: subset de `seller_account_id`, read-only vs. escrita em Ads) — **enforcement 100% nosso**. Revogação individual imediata; expiração opcional.
- Trade-offs: chave estática vaza mais fácil; não há audience binding RFC 8707 documentado; ChatGPT não aceita API key; `static_headers` no Claude é beta e é credencial **da org**, não do usuário. Custo por criação/verificação acima do free tier.
- Exige decidir se o usuário cria a chave (UI `<APIKeys />`) ou só o backend emite (doc recomenda scopes/claims para invalidar chaves criadas pelo frontend).

**Opção C — Esperar/adotar `client_credentials` (extensão MCP oficial).**
- Clerk ainda não suporta; ChatGPT e Claude declaram não suportar; xAI/OpenAI API não documentam. Hoje é bloqueado nos dois lados.

**Opção D — Identidade de máquina (M2M tokens do Clerk).**
- Projetado para serviços internos; scopes = ids de máquinas, sem vínculo a usuário. Para um agente de terceiro agindo por um usuário, exigiria mapear máquina→usuário por conta própria. Baixo encaixe.

**Eixo transversal — granularidade "1 credencial × N sellers":**
- Um token por usuário cobrindo as 50 contas (como hoje) é o que cabe em Managed Agents (uma credencial por `mcp_server_url` por vault) e no Claude/ChatGPT.
- Restringir por seller pode ser feito no token (claims/scopes da API key; análogo ao fine-grained PAT), na URL (análogo ao Sentry `/mcp/{org}`) ou por header/argumento (análogo ao `Stripe-Account`; já temos `seller_account_id` nas tools). Cada lugar muda o que o cliente precisa suportar (URL e header custom não estão disponíveis em todos os clientes — ver tabela).
- Ações de escrita em Ads já têm dry-run/`execute:true` e limites do motor; o mercado (Stripe) adiciona **approval rules** específicas para credencial de agente — pergunta em aberto se credencial de agente deve ter política de escrita diferente da humana.

### Perguntas em aberto para o ticket de decisão

1. Qual o conjunto mínimo de clientes que precisa funcionar no dia 1 (só xAI API? n8n? Managed Agents?) — isso define se bearer estático é obrigatório.
2. A credencial do agente deve poder escrever em Ads, ou começar read-only?
3. Restrição por subset de sellers é requisito ou "todas as contas do usuário" basta?
4. Aceitamos credencial de longa duração (API key sem expiração / refresh sem expiração) ou exigimos expiração + rotação?
5. Quem emite a credencial: o usuário via UI, ou nosso backend via fluxo próprio?

## Itens [não verificado]

- Grok Tasks/Automations: suporte a conectores MCP custom em execuções agendadas.
- Grok app: mecanismo exato de auth de conector custom (OAuth/DCR? header?).
- OpenAI Agents SDK: OAuth/refresh via `params["auth"]` com provider do SDK MCP.
- n8n: refresh automático da credencial MCP OAuth2 e suporte a `client_credentials` nela.
- Clerk: rotação de refresh token; endpoint/UX de revogação de consentimento OAuth; audience binding em API keys; detalhes de "Clerk for Agent Auth" e step-up scopes.
- Shopify Storefront MCP sem autenticação.
