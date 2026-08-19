---
name: produtos
description: "Localiza produtos e entrega detalhes do catálogo Amazon (preço, estimativas, tarifas FBA) e a watchlist de Buy Box do seller (produtos rastreados). Use quando o usuário quiser encontrar um produto por ASIN/EAN ou filtro, ver o detalhe de um item do catálogo, ou revisar o que está rastreando/monitorando de Buy Box."
---

# Monitor de produtos

Cobre três coisas distintas — não as confunda:
- **Catálogo Amazon** (entidade Product, por ASIN): busca e detalhe.
- **Produtos rastreados** (watchlist de Monitorar Buy Box do usuário).
- (Os *anúncios* do seller — SellerListing — são outra entidade, com estoque e reposição: skill `ofertas`. Estas tools tratam do **catálogo** e da **watchlist**, não dos seus anúncios de venda.)

## Conta

- Exige `seller_account_id` (via `whoami` do `radarscout`) — aqui só para atribuição; a busca é no catálogo, não nas suas vendas.

## Passo a passo

### Encontrar / detalhar um produto do catálogo
1. **Buscar:** tool de busca de catálogo (`search_products`) do `radarscout` — por lista de `asins`/`eans` (busca exata) ou por filtros (`brand`, `category_ids`, `price_min/max`, `reviews_min/max`, `rating_min/max`). Retorna linhas compactas (ASIN, título, preço, marca, sinais). Pagina por cursor.
2. **Detalhar:** tool de detalhe (`get_product_details`) por `asin` — metadados de marketplace, estatísticas D90, **tarifas FBA** e estimativas de unidades/receita.

### Revisar a watchlist de Buy Box
3. Tool de produtos rastreados (`list_tracked_products`) do `radarscout` — lista a watchlist com **preço atual**, **variação recente** de preço e a flag de notificação. Use `response_format=detailed` para o histórico recente. Pagina por cursor.

### Avaliar se vale a pena
4. Para simular lucro, margem e ROI de um preço, ou estimar vendas mensais pelo BSR, passe para a skill `calculadora` (`calculate_fees`, `estimate_sales_from_bsr`) — alimentada com o preço, a categoria e o BSR que o detalhe já trouxe.

## O que entregar

- Em busca: uma tabela enxuta dos achados + destaque do mais relevante ao critério do usuário.
- Em detalhe: preço, tarifas FBA e estimativa de vendas, com uma leitura ("margem apertada por causa da tarifa FBA alta").
- Em watchlist: itens monitorados com movimento de preço; destaque quedas/altas relevantes.

## Exemplo

> **Usuário:** "Me mostra o que eu tô monitorando."
> **Skill:** chama produtos rastreados.
> **Resposta:** "Você rastreia 12 produtos. Movimentos recentes: *Fone X* caiu R$ 8 (−6%), *Mouse Y* subiu R$ 3. 3 itens com notificação ligada. Quer o histórico de algum?"

## Cuidados

- **Catálogo ≠ seus anúncios.** Deixe claro que busca/detalhe são do catálogo Amazon, não das suas vendas. Se o usuário quis dizer os SKUs dele (estoque, reposição), a skill é `ofertas`.
- **Rastreado ≠ favorito** — são entidades diferentes.
- **Título nulo** num produto rastreado: identifique pelo **ASIN** em vez de omitir o item.
- Não varra todas as páginas de cursor sem o usuário pedir; mostre a primeira e ofereça mais.
