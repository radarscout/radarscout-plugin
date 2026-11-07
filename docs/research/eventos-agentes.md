# Eventos para agentes autônomos: push MCP, webhooks e fontes Amazon

> Pesquisa para a issue radarscout/radarscout-plugin#16. Data da coleta: 2026-11-07.
> Fontes: só spec / documentação oficial / código-fonte first-party. O que não foi confirmado em fonte primária está marcado **[não verificado]**. Inferências minhas vêm marcadas como *(inferência)*.
> Escopo: **levantar opções**, não decidir.

## Contexto

- Objetivo: agentes autônomos (primeiro alvo Grok/xAI, mas sem depender de plataforma) operando várias contas de seller de um mesmo usuário e **reagindo a gatilhos** (Buy Box perdida, estoque acabando, ACoS disparando, repasse caiu, proposta pendente) sem polling constante.
- O MCP `radarscout` é remoto (Streamable HTTP + OAuth 2.1); ver ADR-0001 no repo.
- Pergunta: (1) o que o MCP oferece de push servidor→cliente e quem suporta; (2) padrões fora do MCP para acordar um agente; (3) o que a Amazon já empurra.

---

## 1. Protocolo MCP: push servidor→cliente

### 1.1 Duas revisões em vigor no ecossistema

A spec "latest" hoje é a **2026-07-28**; a anterior é a **2025-11-25**. <https://modelcontextprotocol.io/specification/latest>, <https://modelcontextprotocol.io/specification/2026-07-28/changelog>

| Mecanismo | 2025-11-25 | 2026-07-28 |
|---|---|---|
| Canal servidor→cliente fora de uma requisição | `GET` no endpoint MCP abre stream SSE; servidor "MAY send JSON-RPC requests and notifications" nele, sem relação com requisição em andamento. <https://modelcontextprotocol.io/specification/2025-11-25/basic/transports> | **Removido.** Substituído por `subscriptions/listen`: "a single long-lived POST-response stream for opted-in server-to-client change notifications". <https://modelcontextprotocol.io/specification/2026-07-28/changelog> |
| Sessão | `MCP-Session-Id` opcional, atribuído no `initialize`. <https://modelcontextprotocol.io/specification/2025-11-25/basic/transports> | **Removida** (sem `initialize`, sem `Mcp-Session-Id`); estado entre chamadas via handles explícitos passados como argumento de tool. <https://modelcontextprotocol.io/specification/2026-07-28/changelog> |
| Assinatura de recurso | `resources/subscribe` → `notifications/resources/updated`; capability `resources.subscribe`. <https://modelcontextprotocol.io/specification/2025-11-25/server/resources> | `subscriptions/listen` com `notifications.resourceSubscriptions: [uri…]`; servidor confirma com `notifications/subscriptions/acknowledged` e marca cada notificação com `io.modelcontextprotocol/subscriptionId`. <https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions> |
| `list_changed` | `notifications/{tools,prompts,resources}/list_changed` (capability `listChanged`). <https://modelcontextprotocol.io/specification/2025-11-25/server/resources> | Mesmas notificações, mas só se o cliente pedir no filtro (`toolsListChanged`, `promptsListChanged`, `resourcesListChanged`); "The server MUST NOT send notification types the client has not explicitly requested." <https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions> |
| Tipos de evento customizados | Não há tipo de notificação de domínio na spec (só os acima + progress/log/cancel) *(inferência a partir das páginas citadas)* | O filtro de `subscriptions/listen` tem só os 4 campos acima — não há evento de domínio arbitrário. <https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions> |
| Retomada após queda | `Last-Event-ID` + IDs de evento SSE permitem replay. <https://modelcontextprotocol.io/specification/2025-11-25/basic/transports> | **Removida** ("A broken response stream loses the in-flight request"). Em stdio, ao reconectar o cliente "MUST re-send `subscriptions/listen`… the server holds no subscription state across reconnections". <https://modelcontextprotocol.io/specification/2026-07-28/changelog>, <https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions> |
| Cache p/ reduzir polling | — | Resultados de `tools/list`, `resources/list`, `resources/read` etc. trazem `ttlMs` e `cacheScope` obrigatórios ("allowing clients to cache responses and reduce polling"). <https://modelcontextprotocol.io/specification/2026-07-28/changelog> |

Consequência prática *(inferência)*: no MCP, push só existe **enquanto o cliente mantém uma conexão aberta**; não há entrega durável nem "acordar" um cliente desconectado. A notificação `resources/updated` carrega só a `uri` — o cliente ainda precisa fazer `resources/read`.

### 1.2 Tasks (assíncrono)

- Saiu do core e virou extensão oficial `io.modelcontextprotocol/tasks` (SEP-2663): servidor devolve `taskId`; cliente faz polling com `tasks/get`, envia input com `tasks/update`. <https://modelcontextprotocol.io/specification/2026-07-28/changelog>, <https://modelcontextprotocol.io/extensions/tasks/overview>
- Push de status: "Servers can push status updates via `notifications/tasks`. Clients opt into these through the `subscriptions/listen` mechanism… Polling is the default." <https://modelcontextprotocol.io/extensions/tasks/overview>
- Serve para operação **iniciada pelo cliente** que demora (ex.: `approve_ads_proposal` assíncrona), não para evento originado no servidor *(inferência)*.
- Suporte de clientes: a página "Extension Support Matrix" existe, mas a coluna Tasks não pôde ser lida (marcas não renderizadas no fetch) — **[não verificado]**. <https://modelcontextprotocol.io/extensions/client-matrix>

### 1.3 Webhooks no MCP: só proposta

- O **Tasks and Events WG** (ex-Triggers and Events, renomeado em 2026-10-16) tem como missão especificar "a standardized callback mechanism—webhooks or similar—that lets servers push notifications when new data is available, with defined ordering guarantees". Item "SEP: Events in MCP v1 RFC" está em **Ideating**, sem SEP aceita. <https://modelcontextprotocol.io/community/working-groups/tasks-events>
- Repo de incubação declara: "Contents are exploratory and do not represent official MCP specifications". <https://github.com/modelcontextprotocol/experimental-ext-triggers-events>
- Logo, **não existe hoje** mecanismo padrão MCP de webhook/callback para acordar um agente.

### 1.4 O que os clientes autônomos suportam

Legenda: ✅ documentado/suportado · ❌ documentado como não suportado · ⚠️ parcial/beta/preview · — não documentado

| Cliente | Tools MCP | Notificações servidor→cliente (`list_changed`, `resources/updated`, `subscriptions/listen`) | Tasks | Fonte |
|---|---|---|---|---|
| **xAI API — Remote MCP tools** | ✅ "Only Streaming HTTP and SSE transports are supported"; `authorization`, `headers`, `allowed_tools` | — (página só fala de tools) | — | <https://docs.x.ai/developers/tools/remote-mcp> |
| **Grok (app) — custom connector** | ✅ "Grok will discover the tools your MCP server exposes" | — | — | <https://docs.x.ai/grok/connectors> |
| **Grok Bot** | ✅ "A Bot can also add an MCP server that isn't in the catalog, by URL" | — | — | <https://docs.x.ai/grok-bot/computer-and-apps> |
| **OpenAI Responses API — tool `mcp`** | ✅ Streamable HTTP ou HTTP/SSE; só tools (`mcp_list_tools`, `mcp_call`) | — | — | <https://developers.openai.com/api/docs/guides/tools-connectors-mcp> |
| **OpenAI Agents SDK** | ✅ `HostedMCPTool`, `MCPServerStreamableHttp`, `MCPServerSse`, `MCPServerStdio`; também lista prompts e resources | — (página não menciona; tem `cache_tools_list` + `invalidate_tools_cache()` manual) | — | <https://openai.github.io/openai-agents-python/mcp/> |
| **OpenAI Agents API** (beta, `OpenAI-Beta: agents=v1`) | ✅ tool MCP com `server_url` | — | — | <https://developers.openai.com/api/docs/guides/agents-api/overview> |
| **Anthropic Messages API — MCP connector** (beta) | ✅ | ❌ "only tool calls are currently supported" | ❌ (idem) | <https://platform.claude.com/docs/en/agents-and-tools/mcp-connector> |
| **Claude Managed Agents** | ✅ MCP servers declarados no agente, credencial via vault | — | — | <https://platform.claude.com/docs/en/agents-and-tools/mcp-connector> |
| **Claude Code — channels** (research preview) | ✅ | ⚠️ um servidor MCP "pushes events into your running Claude Code session"; só sessão local já aberta, plugin em allowlist, `--channels`; não registra canal que negocia a revisão 2026-07-28 | — | <https://code.claude.com/docs/en/channels> |
| **n8n — MCP Client / MCP Client Tool** | ✅ só tools (docs); transporte `httpStreamable` ou `sse` no código-fonte | — (nenhum handler de notificação encontrado em `nodes/mcp/shared/utils.ts`) | — | <https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcpclient>, <https://github.com/n8n-io/n8n/blob/master/packages/@n8n/nodes-langchain/nodes/mcp/shared/descriptions.ts>, <https://github.com/n8n-io/n8n/blob/master/packages/@n8n/nodes-langchain/nodes/mcp/shared/utils.ts> |

Resumo: **nenhum cliente autônomo hospedado documenta consumo de notificações MCP**. Todos tratam o servidor MCP como fornecedor de tools chamado sob demanda dentro de um turno. O único push MCP→agente documentado (Claude Code channels) é local, em preview e exige sessão aberta.

---

## 2. Fora do MCP: acordar o runtime do agente

### 2.1 Gatilhos de entrada (algo externo inicia um run)

| Runtime | Agendado | Webhook/API de entrada | Payload | Limites/observações | Fonte |
|---|---|---|---|---|---|
| **xAI API** | — | — (há Deferred Completions e Batch, ambos polling) | — | Sem trigger documentado na API | <https://docs.x.ai/developers/advanced-api-usage/deferred-chat-completions>, <https://docs.x.ai/llms.txt> |
| **Grok Bot — routines** | ✅ ("every weekday at 8:00 AM", "every 2 hours"; mín. 5 min entre execuções) | ✅ webhook: URL + `Authorization: Bearer <key>`; também Slack, GitHub, Linear, Sentry, PagerDuty, e-mail | "optionally, a JSON body; the Bot receives the body with the routine instruction"; `200` = run iniciado, não concluído | 50 routines/Bot; 300 runs/h no total e 200/h por routine; run que estoura não é re-tentado; cada run consome uso | <https://docs.x.ai/grok-bot/skills-routines-and-automations> |
| **Grok Build (CLI)** | ⚠️ `/loop` (mín. 60 s, expira em 7 dias, máx. 50) — sessão local | ⚠️ "monitors": cada linha de um script vira notificação na conversa — local | — | Ferramenta de dev, não runtime hospedado | <https://docs.x.ai/build/features/background-tasks> |
| **OpenAI Agents API — automations** (beta) | ✅ `periodic` (cron + timezone) ou `once` (`run_at`) | — inbound não documentado; a app pode criar automation `once` em reação a evento próprio | `target.input` (nova sessão) ou `target.instructions` (sessão existente) | `completed` = sessão aceitou o input, não terminou | <https://developers.openai.com/api/docs/guides/agents-api/sessions/automations> |
| **ChatGPT workspace agents** | [não verificado] (só snippet de busca) | ✅ `POST https://api.chatgpt.com/v1/workspace_agents/{id}/trigger` com token de Workspace Agent | `input` (string), `conversation_key`, header `Idempotency-Key` | Status de run em beta; resposta do agente "cannot currently be retrieved through the API" | <https://developers.openai.com/workspace-agents/trigger-runs> |
| **Claude Managed Agents** | ✅ scheduled deployments (cron + IANA tz; jitter até 15% do intervalo; máx. 1.000/org; sem backfill ao despausar) | ⚠️ não há "webhook trigger" pronto; o receptor do integrador cria sessão (`POST /v1/sessions` com `initial_events`) ou envia `user.message` a sessão existente; `deployments/{id}/run` dispara manual sem payload documentado | `user.message` com conteúdo livre | beta `managed-agents-2026-04-01` | <https://platform.claude.com/docs/en/managed-agents/scheduled-deployments>, <https://platform.claude.com/docs/en/managed-agents/sessions> (`initial_events` até 50; `POST /v1/sessions/{id}/events`) |
| **Claude Code scheduled tasks** (ex-routines; research preview) | ✅ | ✅ `POST …/v1/claude_code/routines/{trig}/fire` com bearer; também GitHub | `text` livre, "is not parsed"; chega rotulado como dado não confiável e o prompt precisa optar por agir sobre ele | Beta `experimental-cc-routine-2026-04-01`; só usuários claude.ai; usa connectors MCP da conta | <https://code.claude.com/docs/en/routines> |
| **n8n** | ✅ Schedule Trigger (segundos…cron; workflow publicado) | ✅ Webhook node (Basic, Header, JWT, OAuth2 n8n user) | corpo HTTP arbitrário | URL de produção registrada ao publicar | <https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/>, <https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/> |
| **Zapier** | — (não pesquisado) | ✅ Webhooks by Zapier "Catch Hook" (URL única por Zap; até 10 MB) / "Catch Raw Hook" (2 MB, com headers) | JSON parseado; array dispara 1 vez por item | Não disponível no plano Free; Zapier como runtime de agente (Agents) [não verificado] | <https://help.zapier.com/hc/en-us/articles/8496288690317-Trigger-Zaps-from-webhooks> |

### 2.2 Webhooks de saída das plataformas (observar o agente)

- OpenAI: webhooks de saída (`response.completed` em background, batch, fine-tuning); "OpenAI will send an HTTP POST request to the URL specified" — não iniciam run. <https://developers.openai.com/api/docs/guides/webhooks>. Agents API: "use webhooks to learn when the agent finishes or needs input". <https://developers.openai.com/api/docs/guides/agents-api/overview>
- Anthropic Managed Agents: eventos `session.*`, `deployment_run.*` etc.; payload só com `type` + `id` ("fetch the object directly"); até 3 tentativas, depois "the event is dropped… Webhooks aren't a durable log"; sem ordem garantida; assinatura Standard Webhooks. <https://platform.claude.com/docs/en/managed-agents/webhooks>

### 2.3 Como SaaS comparáveis expõem eventos

- **Stripe — event destinations**: webhook, Amazon EventBridge ou Azure Event Grid; recomenda **thin events** (tipo + id do objeto, sem estado) para novas integrações, porque o snapshot "can be stale"; sem ordem garantida; dedupe por id de evento; retries por até 3 dias; recuperação via List Events (30 dias). <https://docs.stripe.com/event-destinations>
- **Composio** (plataforma de tools para agentes; lido via resumo de busca dos docs oficiais, não página a página): "triggers" por evento × conta conectada; entrega por WebSocket (`subscribe()`) em dev ou webhook único por projeto em produção, com verificação de assinatura; emite também eventos de ciclo de vida (`composio.connected_account.expired`, `composio.trigger.disabled`). <https://docs.composio.dev/docs/setting-up-triggers/subscribing-to-events>, <https://docs.composio.dev/docs/triggers>
- **Anthropic Managed Agents** (acima): mesmo padrão "thin" (type + id) + reconciliação por API. <https://platform.claude.com/docs/en/managed-agents/webhooks>
- Padrão comum *(inferência)*: evento enxuto com id → consumidor busca estado atual via API; entrega at-least-once, fora de ordem, com reconciliação.

---

## 3. Fontes de evento da Amazon

### 3.1 SP-API Notifications API

Mecânica (todas em <https://developer-docs.amazon.com/sp-api/docs/notifications-api> e páginas ligadas):

- Destino: **Amazon SQS** (fila standard; "does not support delivery to FIFO queues") ou **Amazon EventBridge** (partner event source `aws.partner/sellingpartnerapi.amazon.com/{AWS Account Id}/{Application Id}`), conforme o tipo. <https://developer-docs.amazon.com/sp-api/docs/set-up-notifications-with-amazon-sqs>, <https://developer-docs.amazon.com/sp-api/docs/set-up-notifications-with-amazon-eventbridge>
- Operações de destino são *grantless* (nível app); a assinatura é feita em nome do seller/vendor. <https://developer-docs.amazon.com/sp-api/docs/notifications-api>
- Semântica SQS: sem ordem garantida, pode duplicar; dedupe por `notificationId`. <https://developer-docs.amazon.com/sp-api/docs/set-up-notifications-with-amazon-sqs>
- "As a best practice, we recommend that you have a means to retrieve needed information in the event of an unexpected outage or delay in notification delivery." <https://developer-docs.amazon.com/sp-api/docs/notifications-api>
- Filtros: `eventFilter` (só `ANY_OFFER_CHANGED`, `ORDER_CHANGE`, `SHIPMENT_TRACKING_MILESTONE_CHANGED`; inclui `aggregationTimePeriod` `FiveMinutes`/`TenMinutes`) ou `filterExpression` CEL (não combináveis). <https://developer-docs.amazon.com/sp-api/docs/set-up-notifications-with-amazon-sqs>
- Brasil está no endpoint NA (`sellingpartnerapi-na.amazon.com`, `us-east-1`). <https://developer-docs.amazon.com/sp-api/docs/sp-api-endpoints>

Tipos relevantes (descrições de <https://developer-docs.amazon.com/sp-api/docs/notification-type-values>):

| Tipo | Quando dispara | Destino | Gatilho Radar Scout |
|---|---|---|---|
| `ANY_OFFER_CHANGED` | mudança em qualquer das top 20 ofertas por condição, preço externo, "change to which offer wins the BuyBox, or a change to the BuyBox price"; agregação 5/10 min | SQS | Buy Box perdida |
| `PRICING_HEALTH` | oferta do seller "ineligible to be the Featured Offer (Buy Box offer) because of an uncompetitive price", com preços de referência | SQS | Buy Box perdida (causa preço) |
| `B2B_ANY_OFFER_CHANGED` | mudança nas top 20 ofertas B2B | SQS | Buy Box B2B |
| `FBA_INVENTORY_AVAILABILITY_CHANGES` | mudança nas quantidades FBA; snapshot do estoque FBA da região | SQS | Estoque acabando (FBA) |
| `LISTINGS_ITEM_MFN_QUANTITY_CHANGE` | mudança na quantidade disponível do anúncio FBM | EventBridge | Estoque acabando (FBM) |
| `ITEM_INVENTORY_EVENT_CHANGE` / `ITEM_SALES_EVENT_CHANGE` | horários (5 min após a hora), por ASIN, com atraso de até 24 h | SQS | Velocidade/estoque (agregado) |
| `ORDER_CHANGE` | mudança importante no pedido (status, cancelamento pedido pelo comprador) | SQS | Vendas/velocidade |
| `LISTINGS_ITEM_STATUS_CHANGE` | anúncio criado, removido ou "its buyability changes" | EventBridge | Oferta deixou de ser comprável |
| `LISTINGS_ITEM_ISSUES_CHANGE` | issue criada/corrigida/atualizada (supressão, inativação) | EventBridge | Oferta suprimida |
| `ACCOUNT_STATUS_CHANGED` | status entre `NORMAL`, `AT_RISK`, `DEACTIVATED` | SQS | Saúde da conta |
| `TRANSACTION_UPDATE` | "new transaction posted to the seller's account" | SQS | Proxy financeiro (ver 3.3) |
| `REPORT_PROCESSING_FINISHED` | relatório "that you have requested" chega a `DONE`/`CANCELLED`/`FATAL` | SQS | Pipeline interno |
| `FEE_PROMOTION` | promoção de tarifa ativa | SQS | Oportunidade |
| `FBA_OUTBOUND_SHIPMENT_STATUS` | criação/cancelamento de remessa FBA Onsite — "available only in the Amazon Brazil store" | SQS | Operação BR |
| `TAX_INVOICE_*` (3 tipos) | status de nota fiscal — só BR e MX | SQS | Fiscal BR |

### 3.2 Amazon Ads — Amazon Marketing Stream

- "a push-based messaging system that delivers hourly Amazon Ads campaign metrics" e "information on campaign changes in near real-time"; disponível para quem integra a Amazon Ads API. <https://advertising.amazon.com/library/guides/amazon-marketing-stream>
- Entrega em JSON para **SQS** ou **Amazon Data Firehose** na conta AWS do integrador; assinatura por dataset via Ads API. <https://aws.amazon.com/blogs/industries/unlock-real-time-advertising-insights-with-amazon-marketing-stream-and-aws/>
- Datasets (lista do exemplo de referência da Amazon, regiões NA/EU/FE): `sp-traffic`, `sp-conversion`, `budget-usage`, `sd-traffic`, `sd-conversion`, `sb-traffic`, `sb-conversion`, `sb-clickstream`, `sb-rich-media`, `campaigns`, `adgroups`, `ads`, `targets`, `sponsored-ads-campaign-diagnostics-recommendations`, `sp-budget-recommendations`. Com SQS é preciso confirmar a assinatura; com Firehose não. <https://github.com/amzn/amazon-marketing-stream-examples/blob/main/README.md>
- `sp-budget-recommendations`: recomendações "in near real-time when a campaign goes out of budget" (release notes, via busca; página da doc não renderizou no fetch) — **[não verificado]** em leitura direta. <https://advertising.amazon.com/API/docs/en-us/release-notes/index>
- Schemas/latência por dataset: a doc oficial (`/API/docs/en-us/amazon-marketing-stream/data-guide`) é SPA e não pôde ser lida — **[não verificado]**.
- Brasil no endpoint/região NA da Ads API e disponibilidade do Stream para BR — **[não verificado]**.

Mapeamento *(inferência)*: ACoS disparando = `sp-traffic` (custo) + `sp-conversion` (vendas) por hora → ACoS calculado pelo Radar Scout; `budget-usage` / `sp-budget-recommendations` = orçamento estourando; `campaigns`/`adgroups`/`targets` = mudança de entidade feita fora do Radar Scout.

### 3.3 Lacunas na Amazon

- **Repasse**: não há notificação de settlement. "Settlement reports cannot be requested or scheduled. They are automatically scheduled by Amazon. You can search for these reports using the `getReports` operation." <https://developer-docs.amazon.com/sp-api/docs/report-type-values-settlement>. Como `REPORT_PROCESSING_FINISHED` cobre relatórios "that you have requested", relatórios de settlement provavelmente não disparam *(inferência; [não verificado])*. `GET_V2_SETTLEMENT_REPORT_DATA_FLAT_FILE` e `…_XML` têm remoção em 2026-11-11; fica o `…_FLAT_FILE_V2`. <https://developer-docs.amazon.com/sp-api/docs/sp-api-deprecations>. Se `TRANSACTION_UPDATE` cobre o fechamento ou o depósito do repasse: **[não verificado]**.
- **Proposta pendente**: evento interno do motor de automação de Ads do Radar Scout; não tem fonte Amazon.
- **"Estoque acabando"** e **"ACoS disparando"** são limiares que o Radar Scout calcula: a Amazon envia mudança de quantidade e métricas, não o alerta *(inferência)*.

---

## 4. Matriz mecanismo × suporte por cliente

✅ documentado · ⚠️ parcial/beta/preview/local · ❌ documentado como não suportado · — não documentado

| Mecanismo | xAI API | Grok Bot | OpenAI Responses / Agents SDK | OpenAI Agents API | ChatGPT workspace agents | Anthropic Messages (MCP connector) | Claude Managed Agents | Claude Code (cloud tasks / channels) | n8n | Zapier |
|---|---|---|---|---|---|---|---|---|---|---|
| Tools MCP remotas (pull) | ✅ | ✅ | ✅ | ✅ | [não verificado] | ✅ | ✅ | ✅ | ✅ | [não verificado] |
| Notificações MCP (`list_changed`, `resources/updated`, `subscriptions/listen`) | — | — | — | — | — | ❌ | — | ⚠️ channels (local, preview) | — | — |
| Extensão MCP Tasks | — | — | — | — | — | ❌ | — | — | — | — |
| Execução agendada | — | ✅ | — | ✅ beta | [não verificado] | — | ✅ beta | ✅ preview | ✅ | [não verificado] |
| Webhook/API de entrada que inicia run com payload | — | ✅ | — | ⚠️ indireto (criar automation `once`) | ✅ `input` | — | ⚠️ indireto (receptor cria sessão) | ✅ `/fire` `text` (preview) | ✅ | ✅ |
| Webhook de saída da plataforma | — | — | ✅ (`response.completed`) | ✅ | — | — | ✅ beta | — | n/a | n/a |

Fontes: as das seções 1.4 e 2.1/2.2.

---

## 5. Implicações (opções e trade-offs — sem decisão)

1. **Só pull + agendamento do lado do agente** (status quo + cron do runtime: Grok Bot routine, Managed Agents deployment, Agents API automation, n8n Schedule).
   - Prós: funciona hoje em todos os clientes; nada novo no servidor além, talvez, de uma tool de "resumo de alertas".
   - Contras: latência = intervalo do cron; custo de LLM a cada run mesmo sem novidade (Grok Bot avisa que agenda curta "can use a week of usage in a day"). <https://docs.x.ai/grok-bot/skills-routines-and-automations>

2. **Push MCP nativo** (`resources/updated` via `subscriptions/listen`, ou GET SSE na 2025-11-25).
   - Prós: alinhado à spec; não precisa de canal novo.
   - Contras: nenhum cliente autônomo hospedado documenta suporte; exige conexão aberta; a 2026-07-28 tirou retomada (evento perdido na queda); só há tipos genéricos (`uri` mudou), não evento de domínio; a mudança de transporte entre revisões obriga a suportar as duas. Webhooks MCP ainda em "Ideating".

3. **Webhooks de saída do Radar Scout → endpoint de wake do runtime** (Grok Bot webhook, Claude `/fire`, ChatGPT `trigger`, n8n Webhook, Zapier Catch Hook; Managed Agents/Agents API via receptor do integrador).
   - Prós: única forma que acorda agente hoje em vários fornecedores; agnóstico de cliente MCP.
   - Contras: cada destino tem contrato próprio (Grok: Bearer + JSON; Claude `/fire`: `text` string não parseada; ChatGPT: `input` string + `Idempotency-Key`), então provavelmente precisa de adaptadores ou de payload texto *(inferência)*. Também é preciso gerir assinaturas (por usuário × seller × tipo de evento), possivelmente expostas como tools MCP no estilo Composio. O payload vira entrada de prompt (risco de injeção; a Anthropic já rotula `text` como não confiável). Há limites de taxa e custo no destino (Grok: 300 runs/h). O padrão "thin event" (tipo + id + `seller_account_id`, sem estado) reduz dado velho e vazamento, mas obriga o agente a chamar tools de leitura em seguida.

4. **Feed/inbox de eventos consultável** (tool tipo "listar eventos desde cursor").
   - Prós: complementa 1 e 3 como reconciliação, que todas as fontes recomendam (Amazon "backup mechanism", Anthropic "aren't a durable log", Stripe List Events); barato de consultar; idempotente por id de evento.
   - Contras: sozinho continua sendo polling.

5. **Lado da fonte: consumir SP-API Notifications + Marketing Stream vs. continuar consultando a Amazon.**
   - Prós: latência menor (Buy Box/estoque quase em tempo real; Ads por hora); menos cota de API.
   - Contras: infra AWS (SQS/EventBridge/Firehose) e assinatura por seller; dedupe e ordem por conta própria; volume de `ANY_OFFER_CHANGED` (mitigado por agregação de 5/10 min); repasse continua dependendo de polling de `getReports`; "proposta pendente" é evento interno. Essa escolha independe de 1–4: a fonte de eventos (Amazon → Radar Scout) e a entrega (Radar Scout → agente) são decisões separadas *(inferência)*.

Pontos transversais a decidir em outro ticket: escopo da assinatura (usuário com várias contas vs. seller), semântica de entrega (at-least-once + dedupe), assinatura/verificação (ex.: Standard Webhooks, usado pela Anthropic), e como o agente autenticado (ver issue #13) busca o detalhe após o evento.
