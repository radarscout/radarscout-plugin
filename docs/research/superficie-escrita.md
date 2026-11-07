# Superfície de escrita do Radar sem tool no MCP (issue radarscout/radarscout-plugin#15)

> **Tipo:** levantamento (research). **Data:** 2026-11-07.
> **Escopo:** ações de escrita/configuração que o vendedor faz hoje no app web do Radar Scout e que **não** têm tool no MCP `radarscout`. Este documento **não** decide quais ações devem virar tool — essa é uma decisão humana posterior.
> **Convenção:** prosa em pt-BR; identificadores, nomes de tool e campos em inglês. Item sem fonte primária = **[não verificado]**.

## Fontes usadas

| Id | Fonte | O que é |
| --- | --- | --- |
| R-README | `README.md` | Lista das 43 tools (23 leitura + 20 ações de Ads) |
| R-AGENTS | `AGENTS.md` | Contrato das tools e das ações de Ads |
| R-SKILLS | `skills/*/SKILL.md` | Uso das tools pelas skills |
| R-CONTEXT | `CONTEXT.md` | Glossário (Agente, Delegação, Agência…) |
| MCP-LIVE | Registro de tools do conector `Radar_Scout` nesta sessão (nomes + JSON Schema de 5 tools) | Confirma as mesmas 43 tools do README; nenhuma tool de escrita fora de Ads |
| D-TOOLS | https://docs.radarscout.com.br/docs/ai-assistant/tools | "Ferramentas Disponíveis" (fonte de verdade, segundo R-README); lista as mesmas 43 |
| D-AI | https://docs.radarscout.com.br/docs/ai-assistant | "Como Conectar" |
| D-REP | https://docs.radarscout.com.br/docs/repricer | Repricer: visão geral |
| D-REP-LIST | https://docs.radarscout.com.br/docs/repricer/listings | Repricer: Ofertas |
| D-REP-STRAT | https://docs.radarscout.com.br/docs/repricer/strategies | Repricer: Estratégias & Automação |
| D-REP-PMI | https://docs.radarscout.com.br/docs/repricer/pricing-mode | Repricer: Modo de Precificação & PMI |
| D-REP-SAFE | https://docs.radarscout.com.br/docs/repricer/safe-mode | Repricer: Modo simulação |
| D-REP-BULK | https://docs.radarscout.com.br/docs/repricer/bulk-update | Repricer: Importar em massa |
| D-REP-CFG | https://docs.radarscout.com.br/docs/repricer/configuracoes | Repricer: Configurações |
| D-REP-OV | https://docs.radarscout.com.br/docs/repricer/overview | Repricer: Visão gerencial & Ask Scout |
| D-REP-PROMO | https://docs.radarscout.com.br/docs/repricer/promotions | Repricer: Central de Promoções (Mercado Livre) |
| D-REP-AI | https://docs.radarscout.com.br/docs/repricer/scout-ai | Repricer: Scout AI |
| D-REP-BRAND | https://docs.radarscout.com.br/docs/repricer/brand | Repricer: Visão da Marca |
| D-AN-CFG | https://docs.radarscout.com.br/docs/features/analytics/settings | Analytics: Configurações |
| D-AN-COGS | https://docs.radarscout.com.br/docs/features/analytics/cogs | Analytics: Custos de Produto |
| D-AN-LIST | https://docs.radarscout.com.br/docs/features/analytics/listings | Analytics: Ofertas & Reposição |
| D-PERF | https://docs.radarscout.com.br/docs/performance | Performance: visão geral |
| D-PERF-CFG | https://docs.radarscout.com.br/docs/performance/configuracoes | Performance: Configurações |
| D-PERF-CAMP | https://docs.radarscout.com.br/docs/performance/campanhas | Performance: Campanhas |
| D-PERF-KW | https://docs.radarscout.com.br/docs/performance/palavras-chave | Performance: Palavras-chave |
| D-PERF-REC | https://docs.radarscout.com.br/docs/performance/recomendacoes | Performance: Recomendações |
| D-TRACK | https://docs.radarscout.com.br/docs/features/tracking | Scout: Monitorar BuyBox |
| D-LISTS | https://docs.radarscout.com.br/docs/features/lists | Scout: Listas & Workspace |
| D-NOTES | https://docs.radarscout.com.br/docs/features/notes | Scout: Notas |
| D-SCAN | https://docs.radarscout.com.br/docs/features/product-scanner | Scout: Scanner |
| D-PLAN | https://docs.radarscout.com.br/docs/features/planner | Scout: Planejamento |
| D-SUPP | https://docs.radarscout.com.br/docs/features/suppliers | Scout: Fornecedores |
| D-IMG | https://docs.radarscout.com.br/docs/features/listing-images | Ferramentas: Imagens de Listing |
| D-PA | https://docs.radarscout.com.br/docs/features/product-analysis | Scout: Análise de Produto |
| D-SCOUTAI | https://docs.radarscout.com.br/docs/features/scout-ai | Scout AI no Scout |
| D-CONN | https://docs.radarscout.com.br/docs/connect-accounts | Conectar Contas |
| D-INTEG | https://docs.radarscout.com.br/docs/account/integrations | Integrações (ERP) |
| D-NOTIF | https://docs.radarscout.com.br/docs/features/notifications | Notificações |
| D-ACC | https://docs.radarscout.com.br/docs/account | Assinaturas & Planos |
| D-PART | https://docs.radarscout.com.br/docs/account/partners | Programa de Parceiros |

Método: crawl das 92 páginas `/docs/*` de docs.radarscout.com.br (exceto `/docs/legal/*`) e leitura das páginas de feature/configuração de cada app; leitura do repositório do plugin; inspeção do schema das tools ao vivo. O site radarscout.com.br é renderizado no cliente e só trouxe texto de marketing (preços, nomes dos apps) — sem informação de ações.

## Achados de enquadramento (antes da tabela)

1. **Toda escrita do MCP hoje é Amazon Ads.** As 20 tools de ação são todas de Ads; as outras 23 são leitura (R-README, R-AGENTS, D-TOOLS, MCP-LIVE).
2. **A doc oficial declara Repricer e Analytics como somente leitura pelo assistente externo.** "Preços do Repricer e configurações do Analytics não mudam pelo assistente: ele só consulta. Para alterar, use o app." (D-AI). "Pelo Assistente IA conectado ao Claude ou ao ChatGPT, o Repricer é somente leitura" (D-REP-AI).
3. **No Performance, a relação se inverte: o MCP é o único caminho de escrita de campanha.** "O painel não altera campanhas diretamente. Pausar, reativar, mudar orçamento ou lance e negativar termos não são botões do Performance." (D-PERF). O que o painel edita é só configuração do Scout AI e objetivo/meta por campanha (D-PERF) — e é exatamente isso que falta no MCP.
4. **O assistente *interno* (Scout AI dentro do app) já escreve no Repricer e no Scout.** No Repricer ele cria/remove automação, vincula/desvincula ofertas, liga/pausa reprecificação, liga/desliga simulação (oferta ou conta), define mínimo e máximo, sincroniza ofertas, liga PMI, atualiza margem mínima em lote e faz atualização em massa (D-REP-AI). No Scout ele cria listas, adiciona produtos, liga/desliga o Monitorar BuyBox, cria/edita notas e gera imagens de listing (D-SCOUTAI). Ou seja: as operações já existem no backend com interface de agente; só não estão expostas no MCP. Se dá para reaproveitar, só o monorepo responde (ver última seção).
5. **"Radar" × "Scout".** O briefing chama de "Radar" o app de catálogo/watchlist; a doc chama esse app de **Scout** (D-TRACK, D-ACC), e o entitlement do MCP é `radar` (R-AGENTS). Na tabela uso "Scout (entitlement `radar`)".

## Legenda de risco

- **reversível** — dá para desfazer pelo próprio app sem perda além do tempo.
- **afeta preço** — muda (direta ou indiretamente, via piso/teto/estratégia/promoção) o preço publicado no marketplace.
- **afeta dinheiro** — muda gasto (Ads), cobrança/assinatura/comissão, ou o custo/lucro que o Radar reporta (CMV, impostos).
- **irreversível** — a doc diz que não dá para desfazer, ou não mostra caminho de desfazer (marcado [não verificado] quando inferido).
- **operacional** — dispara rotina (sincronização, varredura) sem efeito direto em preço ou dinheiro. Rótulo extra além dos quatro pedidos, para não forçar esses itens nas outras categorias.

"Existe no MCP?": **Não** = nenhuma tool; **Parcial** = existe tool relacionada que não cobre a ação inteira; **Sim** = coberta (listada só para contraste).

## Tabela — app × ação × MCP × risco × fonte

### Repricer (Amazon + Mercado Livre; entitlement `repricer`)

| # | Ação no app web | Existe no MCP? | Risco | Fonte |
| --- | --- | --- | --- | --- |
| RP-01 | Ligar/desligar o **motor do Repricer** da conta inteira (Status do Repricer) | Não (só leitura: `get_repricer_summary`, `list_repricer_activity`, `explain_price_change`) | afeta preço · reversível | D-REP-CFG; R-README |
| RP-02 | Ligar/desligar **Modo simulação da conta** (desligar = passa a publicar preços) | Não | afeta preço · reversível | D-REP-CFG; D-REP-SAFE |
| RP-03 | Ligar/desligar **Modo simulação por oferta** | Não | afeta preço · reversível | D-REP-SAFE; D-REP-LIST |
| RP-04 | Ligar/**pausar a reprecificação por oferta** (interruptor Ativado/Pausado) | Não | afeta preço · reversível | D-REP-LIST; D-REP |
| RP-05 | **Retomar ajustes** após pausa automática por Disputa de preço | Não | afeta preço · reversível | D-REP-LIST; D-REP-SAFE |
| RP-06 | Definir/editar **Preço mínimo manual** (na linha ou na página da oferta) e "Aplicar alterações" | Não (`get_seller_offer` só lê limites/PMI) | afeta preço · reversível | D-REP-PMI; D-REP-LIST; MCP-LIVE |
| RP-07 | Definir/editar **Preço máximo**; "Sugerir valor máximo" (+40% do mínimo) | Não | afeta preço · reversível | D-REP-PMI; D-REP-LIST |
| RP-08 | Trocar o **modo de precificação** Manual ↔ **PMI** por oferta | Não | afeta preço · reversível | D-REP-PMI |
| RP-09 | Editar insumos do PMI por oferta: **margem mínima**, **custo do produto**, **custo de envio** e a **origem** de cada um (Automático/Valor fixo) | Não | afeta preço · afeta dinheiro (custo é compartilhado com o CMV do Analytics) · reversível | D-REP-PMI; D-AN-COGS |
| RP-10 | **Editar atributos** da oferta Amazon (comprimento, largura, altura, peso) — entram na tarifa e no PMI | Não | afeta preço · reversível | D-REP-LIST |
| RP-11 | **Criar automação** (cartões Conservador/Equilibrado/Agressivo/Liquidação/Scout AI; formulário; "Criar com Scout AI"; "Configuração rápida: Buy Box" que **cria e vincula todas as ofertas**) | Não | afeta preço (a configuração rápida já vincula tudo) · reversível | D-REP-STRAT |
| RP-12 | **Editar automação**: referência (vencedor da Buy Box / menor preço / preço competitivo), posição (abaixo/igual/acima, % ou R$), preço diferente p/ Amazon, sem concorrência, regra após ganhar Buy Box, Buy Box suprimida, filtros de concorrente (mesmo envio, ignorar Amazon, ignorar minhas contas, whitelist/blacklist de sellers); no ML, "fora do catálogo" | Não | afeta preço · reversível | D-REP-STRAT |
| RP-13 | Escrever o **Prompt personalizado** da estratégia Scout AI (experimental, com aviso de responsabilidade) | Não | afeta preço · reversível | D-REP-STRAT; D-REP-AI |
| RP-14 | **Duplicar** automação | Não | reversível (sem efeito até vincular) | D-REP-STRAT |
| RP-15 | **Remover** automação (as ofertas ficam sem regra) | Não | afeta preço · irreversível [não verificado: a doc não mostra desfazer; dá para recriar à mão] | D-REP-STRAT |
| RP-16 | **Vincular/desvincular** oferta a uma automação | Não | afeta preço · reversível | D-REP-STRAT; D-REP-LIST |
| RP-17 | **Importar em massa** (CSV/JSON): preço, mínimo, máximo, margem mínima %, custo do produto (alimenta o histórico do Analytics), custo de envio (vira valor fixo); remover importação do histórico | Não | afeta preço (inclui a coluna **Preço** de venda) · afeta dinheiro (custo) · reversível só reimportando valores antigos [não verificado] | D-REP-BULK |
| RP-18 | **Alíquota efetiva de impostos** da conta (compartilhada com o Analytics) | Não | afeta preço (PMI) · afeta dinheiro (lucro reportado) · reversível | D-REP-CFG; D-AN-CFG |
| RP-19 | **Custo de preparação** por unidade (compartilhado com o Analytics) | Não | afeta preço · afeta dinheiro · reversível | D-REP-CFG; D-AN-CFG |
| RP-20 | **Margem mínima da conta** (Mercado Livre; também pelo "Aplicar" do simulador da Central de Promoções); remover pede confirmação | Não | afeta preço · reversível | D-REP-CFG; D-REP-PROMO |
| RP-21 | **Região de origem do frete** (tabela DBA) | Não | afeta preço (PMI de ofertas DBA) · reversível | D-REP-CFG |
| RP-22 | **Sincronizar ofertas** do Repricer | Não | operacional | D-REP-CFG |
| RP-23 | Ask Scout: ligar **rotinas da Visão gerencial/Ask Scout** e **e-mail de propostas** > R$ 500 | Não | reversível | D-REP-CFG; D-REP-OV |
| RP-24 | Ask Scout: **Varrer agora** | Não | operacional | D-REP-OV |
| RP-25 | Ask Scout: **aprovar / ignorar / reabrir / desfazer propostas** (subir piso, baixar piso, reativar Repricer, baixar preço máximo, esperar e avisar, revisar custos) | Não (as tools de proposta do MCP — `list_ads_proposals`, `approve_ads_proposal`, `reject_ads_proposal` — são só de **Ads**) | afeta preço · reversível ("desfazer" existe) | D-REP-OV; D-REP-AI; R-AGENTS |
| RP-26 | No preço mínimo: **Baixar o piso até igualar** (em lote e pelo "Aplicar" do simulador) | Não | afeta preço · reversível [não verificado: desfazer não descrito] | D-REP-OV |
| RP-27 | No preço mínimo: **Atualizar custo** em lote (em ofertas com PMI o piso é recalculado e enviado) | Não | afeta preço · afeta dinheiro · reversível | D-REP-OV |
| RP-28 | Criar/encerrar **aviso de concorrente** ("Avisar quando subir") | Não | reversível | D-REP-OV |
| RP-29 | Central de Promoções (ML): **ligar/desligar promoções** por oferta ou em lote | Não | afeta preço · afeta dinheiro (tarifa) · reversível | D-REP-PROMO; D-REP-LIST |
| RP-30 | Central de Promoções (ML): **entrar / sair de campanha / voltar ao automático** (até 500 por vez); "Ligar" e "Entrar" na aba Elegíveis | Não | afeta preço · afeta dinheiro · reversível [não verificado: reentrada depende da janela da campanha no ML] | D-REP-PROMO |
| RP-31 | Conectar nova conta (Amazon ou ML) pelo seletor do cabeçalho | Parcial (ver CT-01/CT-02) | — | D-REP |

Fora da lista: a Visão da Marca (D-REP-BRAND) e Concorrentes são telas de leitura; não achei escrita própria nelas além de RP-28. A exportação de ofertas é leitura.

### Analytics (Amazon; entitlement `analytics`)

| # | Ação no app web | Existe no MCP? | Risco | Fonte |
| --- | --- | --- | --- | --- |
| AN-01 | **Custo do produto por SKU com histórico**: adicionar período (data de início + valor; fecha o anterior), **editar** ou **excluir** período | Não (`get_profit_waterfall` só aponta SKUs sem custo; `get_seller_offer` só lê o CMV) | afeta dinheiro (reescreve o lucro de períodos passados quando se edita um período antigo) · afeta preço (mesmo histórico do PMI) · excluir período: irreversível [não verificado: sem desfazer descrito] | D-AN-COGS; D-AN-LIST; R-SKILLS (`skills/lucro/SKILL.md`) |
| AN-02 | **Alíquota efetiva de impostos** | Não | = RP-18 (mesmo campo) | D-AN-CFG |
| AN-03 | **Custo de preparação** | Não | = RP-19 (mesmo campo) | D-AN-CFG |
| AN-04 | **Lead time de reposição** da conta (1–365, padrão 21) | Não (`list_seller_offers` só lê `replenishment_status`) | reversível (muda só a sugestão de compra) | D-AN-CFG; R-SKILLS (`skills/ofertas/SKILL.md`) |
| AN-05 | **Cobertura desejada** da conta (1–365, padrão 30) e **por SKU**, com "voltar ao padrão" | Não | reversível | D-AN-CFG; D-AN-LIST |
| AN-06 | **Dados em tempo real** (liga/desliga; habilita o aviso Nova venda) | Não | reversível | D-AN-CFG |
| AN-07 | **Sincronizar ofertas** do Analytics | Não | operacional | D-AN-CFG |
| AN-08 | Origem do custo/ERP: **Pausar / Retomar / Sincronizar agora / Conferir** (também no Repricer) | Não | afeta dinheiro · afeta preço · reversível | D-AN-CFG; D-REP-CFG |
| AN-09 | **Gerar imagens de listing** (Ferramentas › Listings; aparece com o Analytics ativo) | Não | reversível [não verificado se publica algo na Amazon; a doc só fala em gerar e na galeria] | D-IMG |

Observação: a doc diz que "não há importação de custo por CSV dentro do Analytics"; o caminho em massa é o Importar do Repricer (RP-17) ou o ERP (CT-05) (D-AN-COGS).

### Performance (Amazon Ads; entitlement `performance`)

| # | Ação no app web | Existe no MCP? | Risco | Fonte |
| --- | --- | --- | --- | --- |
| PF-01 | **Ativar Scout AI** (primeira vez): define a meta de ACoS da conta e os nomes da marca; **a ativação também libera as mudanças pelo assistente de IA** nesta conta | Não | afeta dinheiro (habilita automação que move lances; habilita as próprias tools de ação do MCP) · reversível depois (pausar) | D-PERF; D-PERF-CFG |
| PF-02 | **Ligar/pausar o Scout AI** (pausado: aprovar fica desligado, rejeitar continua) | Não | afeta dinheiro · reversível | D-PERF-CFG; D-PERF-REC |
| PF-03 | **Modo do Scout AI**: "Você aprova" ↔ "Automático"; **Continuar aprovando cada mudança** (cancela a transição automática do 14º dia) | Não | afeta dinheiro (no Automático, lances e negativações vão à Amazon sem aprovação) · reversível | D-PERF-CFG; D-PERF; D-PERF-CAMP |
| PF-04 | **Meta de ACoS da conta** (em branco: o Scout AI só negativa e não mexe em lance) | Não | afeta dinheiro · reversível | D-PERF-CFG |
| PF-05 | **Nomes da marca** (nunca negativados) | Não | afeta dinheiro (indireto) · reversível | D-PERF-CFG; D-PERF-REC |
| PF-06 | **Objetivo por campanha** (Descoberta / Desempenho / Defesa / Disputa); "Definir" em Riscos | Não | afeta dinheiro (decide se o Scout AI move lances) · reversível | D-PERF-CAMP; D-PERF |
| PF-07 | **Meta de ACoS por campanha** (até 200%; em branco herda a da conta) | Não | afeta dinheiro · reversível | D-PERF-CAMP |
| PF-08 | **Aprovar / rejeitar recomendação** do Scout AI | Sim (`approve_ads_proposal`, `reject_ads_proposal`) | afeta dinheiro | R-AGENTS; D-TOOLS |
| PF-09 | **Aprovar/rejeitar em lote** (até 100, com resumo agregado) | Parcial — `approve_ads_proposal` aceita um único `proposal_id` | afeta dinheiro | D-PERF-REC; MCP-LIVE |
| PF-10 | **Conectar / Reconectar conta de anúncios** (Amazon Ads) e **escolher profile** | Não (`connect_amazon_sp_api` só gera link SP-API; `target_app` aceita `radar`/`analytics`/`repricer`, não `performance`) | reversível | D-PERF-CFG; D-CONN; MCP-LIVE |
| PF-11 | **Sincronizar ofertas** do Performance (para o Scout AI reconhecer a marca nos termos) | Não | operacional | D-PERF-CFG |
| PF-12 | Pausar/reativar/arquivar campanha, orçamento, lances, estado de keyword/alvo, negativar, criar campanha/grupo/portfólio, incluir keywords/alvos/produtos | Sim (20 tools) — e **não** são botões do app: o painel delega ao assistente | afeta dinheiro; arquivar = irreversível | R-README; D-PERF; D-PERF-CAMP; D-PERF-KW |

Sobre leitura de configuração (complementar): o schema ao vivo de `list_ad_campaigns` não menciona objetivo nem meta de ACoS da campanha nos campos descritos, e nenhuma tool lê o modo/meta do Scout AI (MCP-LIVE). Se a resposta traz esses campos sem documentar é **[não verificado]**.

### Scout (catálogo e watchlist; entitlement `radar`)

| # | Ação no app web | Existe no MCP? | Risco | Fonte |
| --- | --- | --- | --- | --- |
| SC-01 | **Adicionar produto ao Monitorar BuyBox** (por ASIN, pela análise, pelas Oportunidades, pelo Workspace ou pelo Scout AI) | Não (só `list_tracked_products`) | reversível | D-TRACK; MCP-LIVE |
| SC-02 | **Remover** produto monitorado | Não | reversível [não verificado se o histórico de preço se perde] | D-TRACK |
| SC-03 | Interruptor **Notificar** por produto monitorado | Não (o MCP só lê a flag) | reversível | D-TRACK; D-NOTIF |
| SC-04 | **Listas/Workspace**: criar, editar, fixar, remover lista; adicionar, remover e mover produtos entre listas | Não | reversível; remover lista [não verificado] | D-LISTS |
| SC-05 | **Notas**: criar, editar, excluir (ligadas a produto, categoria, marca, lista, fornecedor, oferta ou automação do Repricer) | Não | reversível; excluir [não verificado] | D-NOTES |
| SC-06 | **Scanner**: importar lote (CSV/JSON até 5 MB), remover lote | Não | reversível; remover lote [não verificado] | D-SCAN |
| SC-07 | **Planejamento**: criar/editar simulação por SKU, importar cotação/parâmetros | Não | reversível (simulação, sem efeito externo) | D-PLAN |
| SC-08 | **Indicar fornecedor** (passa por revisão da equipe) | Não | irreversível [não verificado: a doc não descreve retirar a indicação] | D-SUPP |
| SC-09 | **Sugerir produto** (ASIN) para o Radar acompanhar | Não | irreversível [não verificado] | D-PA; D-SCOUTAI |

### Conta / Minha Conta (transversal a todos os apps)

| # | Ação no app web | Existe no MCP? | Risco | Fonte |
| --- | --- | --- | --- | --- |
| CT-01 | **Conectar conta Amazon (SP-API)** | Parcial — `connect_amazon_sp_api` gera o link; o usuário conclui no Seller Central | reversível | R-SKILLS (`skills/inicio/SKILL.md`); D-CONN; MCP-LIVE |
| CT-02 | **Conectar conta Mercado Livre** (OAuth) | Não | reversível | D-CONN; MCP-LIVE |
| CT-03 | **Desconectar conta** / revogar acesso | Não — e **também não existe no app**: é pedido ao suporte (ou revogado no Seller Central/ML) | irreversível [não verificado] | D-CONN |
| CT-04 | **Adicionar/remover colaborador** numa conta de seller (a "Delegação" do glossário) | Não — e **também não existe no app**: só pela equipe do Radar via suporte | — | D-CONN; R-CONTEXT |
| CT-05 | **Integração ERP** (Olist ERP, VHSYS): conectar com token, Conferência, **Aplicar e ligar automático**, trocar token, "Qual custo usar" (último/médio), Pausar/Retomar/Sincronizar, adicionar/remover conta, Desconectar | Não | afeta dinheiro (CMV) · afeta preço (PMI) · reversível (desconectar mantém os custos como valor fixo); envolve token secreto do ERP | D-INTEG |
| CT-06 | **Preferências de notificação** (cada aviso × no app / por e-mail) | Não | reversível | D-NOTIF |
| CT-07 | **Mudar plano / adicionar app / montar pacote** (cobrança proporcional e ativação imediata) | Não | afeta dinheiro · reversível só no próximo ciclo (descer de plano vale no ciclo seguinte) | D-ACC |
| CT-08 | **Cancelar assinatura** (conduzido pelo Scout AI no chat) | Não | afeta dinheiro · irreversível em parte [não verificado: o teste grátis "vale uma vez por conta"] | D-ACC |
| CT-09 | **Trocar cartão / atualizar pagamento** (portal Stripe) | Não | afeta dinheiro | D-ACC |
| CT-10 | **Programa de Parceiros**: ativar (código **não pode ser trocado depois**), emitir/revogar vouchers, **solicitar saque PIX** | Não | afeta dinheiro · ativação irreversível | D-PART |

## Resumo por app

| App | Ações de escrita sem tool | Com tool parcial | Observação |
| --- | --- | --- | --- |
| Repricer | 30 (RP-01…RP-30) | RP-31 (via CT-01) | 100% da configuração e do preço sem tool; a doc chama o MCP de "somente leitura" no Repricer |
| Analytics | 9 (AN-01…AN-09; AN-02/03 = mesmos campos de RP-18/19) | — | Custo por SKU (AN-01) é a lacuna que as próprias skills apontam ("liste os SKUs a cadastrar") |
| Performance | 9 (PF-01…PF-07, PF-10, PF-11) | PF-09 (lote) | Ações de campanha cobertas; falta toda a configuração do Scout AI e de objetivo/meta |
| Scout | 9 (SC-01…SC-09) | — | Watchlist só tem leitura |
| Conta | 8 sem tool (CT-02, CT-05…CT-10) + 2 sem caminho nem no app (CT-03, CT-04) | CT-01 | Colaborador/Delegação é pré-requisito do cenário "agência" do CONTEXT.md e hoje só existe via suporte |

## Lacunas que só o monorepo `radar` responde (inacessível nesta sessão)

1. **Reaproveitamento das tools do assistente interno.** Quais handlers/serviços o Scout AI do Repricer e do Scout já usa para escrever (D-REP-AI, D-SCOUTAI), com que validações, e se dá para expô-los no MCP sem reimplementar.
2. **Completude da superfície.** Se há ações de escrita no app que a doc não descreve (flags de beta, telas por convite como a Visão da Marca, configurações administrativas).
3. **Reversibilidade real** dos itens marcados [não verificado]: remover automação (RP-15), excluir período de custo (AN-01), remover lista/nota/lote/monitorado (SC-02, SC-04–SC-06) — soft delete ou exclusão física? Existe histórico/auditoria?
4. **Trilha de auditoria por origem.** No Performance as mudanças levam origem (Scout AI ou MCP) em Atividades (D-PERF); se as Atividades do Repricer e o histórico de custo do Analytics registram a origem de quem mudou config/custo, e se já existe origem "MCP" nesses domínios.
5. **Gate "mudanças liberadas na sua conta"** (`not_enabled` em R-SKILLS `skills/acoes-ads/SKILL.md`): onde é definido, e se a ativação do Scout AI (PF-01) é o único caminho (D-PERF diz que a ativação "também libera as mudanças pelo assistente").
6. **Limites de plano no backend** por ação (tetos de automações 3/10/30, ofertas com preço automático 50/200/500, produtos monitorados 10/50/ilimitado, itens em listas 30/200/ilimitado — D-REP, D-TRACK, D-LISTS): se são aplicados na API ou só na UI.
7. **Identificadores estáveis** para escrita: ids de automação, de proposta do Ask Scout, de aviso de concorrente, de período de custo; se as propostas do Ask Scout usam o mesmo modelo `AutomationProposal` das de Ads (o schema de `approve_ads_proposal` exige `AutomationProposal.id` de 24 hex — MCP-LIVE).
8. **Semântica da coluna "Preço" no Importar** (RP-17): se escreve o preço direto no marketplace ignorando a automação/simulação, ou só quando a reprecificação está ativa e fora da simulação (D-REP-BULK só diz isso de forma geral).
9. **Escrita no Mercado Livre**: contrato e rate limits da Central de Promoções (RP-29/RP-30) e da publicação de preço no ML.
10. **Modelo de Delegação** (CT-04): se já existe no código algo de "dono × colaborador" self-service, ou só a operação manual da equipe (D-CONN, R-CONTEXT).
11. **Campos de leitura de configuração**: se `list_ad_campaigns`/`get_ads_overview` já devolvem objetivo, meta de ACoS ou modo do Scout AI sem documentar (para "ler antes de escrever" nos itens PF-03…PF-07).
12. **Lote em propostas de Ads** (PF-09): se o backend da aprovação em lote do app pode ser exposto, ou se o agente tem de aprovar uma por vez.

## Discrepâncias notadas de passagem (fora do escopo; sem ação)

- D-AI diz que o plugin para Claude Code traz "oito fluxos prontos"; o repositório tem 12 skills (R-README, R-AGENTS).
- D-TOOLS descreve `add_product_ads` como "um ASIN ou SKU"; R-AGENTS diz que ASIN só vale para conta vendor e que conta de seller anuncia por SKU.
