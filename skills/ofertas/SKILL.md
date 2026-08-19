---
name: ofertas
description: "Consulta as ofertas (anúncios) da conta do seller na Amazon — SKUs com preço, estoque, modalidade FBA/FBM/DBA, Buy Box e reposição (velocidade de venda, dias de cobertura, risco de ruptura). Use quando o usuário perguntar dos SEUS anúncios/SKUs, quanto tem de estoque, quantos dias de estoque restam, o que precisa repor, quais SKUs estão perdendo a Buy Box, ou pedir o detalhe de um SKU específico."
---

# Ofertas do seller

Trata dos **anúncios do próprio vendedor** (os SKUs da conta conectada): preço, estoque, modalidade, Buy Box e reposição.

**Não confunda com catálogo.** Buscar produto por ASIN/marca/preço é a skill `produtos` (oportunidades do catálogo Amazon). Aqui é o que o vendedor **já vende**.

## Conta e período

- **Conta:** exige `seller_account_id` (via `whoami` do `radarscout`).
- **Período:** ISO `YYYY-MM-DD`, **fim exclusivo**, opcional. Ele define a janela de **unidades vendidas e velocidade** usadas na leitura de reposição — não filtra quais ofertas aparecem. Sem período, a tool usa os **últimos 30 dias**; diga a janela ao apresentar velocidade ou cobertura.

## Passo a passo

1. **Listar:** `list_seller_offers` do `radarscout`. Filtros: `search` (ASIN, SKU ou título), `fulfillment` (`fba`/`fbm`/`dba`), `buybox` (`winning`/`losing`), `active_only` (padrão: só ofertas ativas). Pagina por cursor (`next_cursor`).
2. **Detalhar:** `get_seller_offer` por `listing_id` **ou** `sku` — acrescenta limites de preço/PMI, custo (CMV) e a reposição do item.
3. Se o usuário quiser saber **por que o preço mudou** naquele SKU, passe para a skill `repricer`; se quiser **lucro**, para a skill `lucro`.

## Como ler a reposição

Cada oferta vem com `velocity_per_day` (média de unidades/dia na janela), `days_of_cover` (quantos dias o estoque atual dura nesse ritmo) e `replenishment_status`:

| Status | O que significa |
| --- | --- |
| `critical` | O estoque acaba **antes** de o fornecedor conseguir repor — risco real de ruptura. |
| `attention` | Ainda dá tempo, mas está abaixo do estoque-alvo. |
| `ok` | Cobertura confortável. |
| `no_movement` | Tem estoque e **não vendeu** na janela — parado, não é urgência de compra. |
| `out_of_stock` | Zerado e com venda no período — está perdendo venda agora. |

`no_movement` e `out_of_stock` são situações opostas: a primeira é capital parado, a segunda é venda perdida. Nunca trate as duas como "problema de estoque" genérico.

## O que entregar

- Em lista: as ofertas que **exigem ação** primeiro (ruptura, crítico, Buy Box perdida), não a lista inteira em ordem de banco.
- Em detalhe: preço x limites, estoque x velocidade, Buy Box, com uma leitura em uma frase ("13 dias de cobertura e o fornecedor leva 20 — precisa comprar agora").
- **Sempre a janela usada** quando falar de velocidade ou cobertura.

## Exemplo

> **Usuário:** "O que eu preciso repor?"
> **Skill:** chama `list_seller_offers` na conta.
> **Resposta:** "De 84 ofertas ativas, 6 pedem atenção. **Críticas (3):** *Fone X* (SKU FN-01) tem 9 dias de cobertura vendendo 4,2 un/dia; … **Zeradas com venda (1):** *Cabo Y* está sem estoque e vendia 2,8 un/dia — é venda perdida agora. Números da janela de 30 dias."

## Cuidados

- **Oferta ≠ produto do catálogo.** `list_seller_offers` é o que o vendedor vende; `search_products` é o catálogo Amazon. Não misture as duas na mesma tabela.
- **Sem ofertas retornadas** não quer dizer sem anúncios: os dados vêm do sync do Analytics/Repricer. Verifique se a conta está conectada e sincronizada antes de dizer que não há nada.
- `active_only` vem ligado por padrão — se o usuário procura um SKU antigo/inativo e não acha, desligue antes de dizer que não existe.
- **Cobertura depende da janela.** Uma janela curta em produto sazonal distorce a velocidade; diga a janela em vez de dar o número solto.
- Não varra todas as páginas de cursor sem o usuário pedir.
