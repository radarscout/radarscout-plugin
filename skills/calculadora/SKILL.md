---
name: calculadora
description: "Simula tarifas, lucro e viabilidade de um produto na Amazon Brasil antes de comprar ou de mudar o preço — tarifa de venda, logística FBA/FBM/DBA, armazenagem, impostos, lucro líquido, margem e ROI — e estima vendas mensais a partir do BSR e da categoria. Use quando o usuário perguntar quanto sobra vendendo a X reais, se vale a pena um produto, qual a tarifa da Amazon, qual preço mínimo cobrir custos, ou quantas unidades por mês um BSR representa."
---

# Calculadora e estimativa

Duas simulações que **não leem dados da conta** — funcionam para qualquer produto, inclusive um que o vendedor ainda não vende:

- `calculate_fees` — tarifas e rentabilidade de um preço.
- `estimate_sales_from_bsr` — vendas mensais estimadas a partir do BSR.

Não pedem `seller_account_id`.

## Calcular tarifas e lucro

`calculate_fees` do `radarscout`. Único parâmetro obrigatório: `price`.

| Informe | Para quê |
| --- | --- |
| `cost` | Custo de compra — sem ele **não há ROI nem lucro real**, só tarifas. |
| `category` | Rótulo ("Eletrônicos") ou ID Amazon — define a tarifa de venda. **Sem categoria, a tool assume 15%.** |
| `weight_g`, `length_cm`, `width_cm`, `height_cm` | Peso e dimensões da embalagem — definem a logística FBA/DBA. |
| `fulfillment` | `fba` (padrão), `fbm` (você envia — informe `shipping_cost`) ou `dba` (informe `dba_region`). |
| `tax_rate_pct` | Impostos sobre o preço (ex.: `12` = 12%). |
| `estimated_sales` | Projeta o lucro total para um volume. |

Retorna tarifa de venda, logística, armazenagem, imposto, custo total, **lucro líquido, margem (%) e ROI (%)**.

**Pergunte antes de assumir.** Custo, categoria e peso mudam o resultado por completo. Se o usuário não deu, pergunte — ou calcule assumindo e **diga em voz alta o que assumiu**.

## Estimar vendas pelo BSR

`estimate_sales_from_bsr` — informe `ranking` (BSR atual) e `category_id` (código da categoria Amazon, preferível) ou `category_name`.

Retorna a estimativa de unidades/mês da categoria. Leia com cuidado:

- `is_below_known_range: true` — o BSR é **melhor que o topo da tabela** da categoria; a estimativa é um piso, não um número fechado.
- `estimated_avg_sales: null` — a categoria não tem faixa para esse BSR. Diga que não dá para estimar; não invente por interpolação.
- É **média da categoria**, não desse produto. Sazonalidade, variações e cauda de listagem distorcem.

## Juntar as duas

A pergunta "vale a pena?" quase sempre é lucro × volume: `calculate_fees` dá o lucro por unidade, `estimate_sales_from_bsr` dá a ordem de grandeza do volume. Multiplique **rotulando como estimativa** — nenhuma das duas é dado da conta do vendedor.

Para um produto do catálogo, `get_product_details` (skill `produtos`) já traz preço, BSR e tarifas FBA — use-o para alimentar a calculadora em vez de pedir tudo ao usuário.

## Exemplo

> **Usuário:** "Comprando a R$ 40 e vendendo a R$ 99 em eletrônicos, compensa?"
> **Skill:** chama `calculate_fees` (`price` 99, `cost` 40, `category` "Eletrônicos", FBA) e pergunta o peso.
> **Resposta:** "A R$ 99, com tarifa de venda de R$ 14,85 e logística FBA de R$ 15,50 (estimada para 400 g): sobram **R$ 28,65** por unidade — margem 28,9%, ROI 71,6%. Sem impostos no cálculo, você não passou a alíquota. Se o peso real for maior, a logística sobe e a margem cai — me diga o peso da embalagem para fechar a conta."

## Cuidados

- **É simulação, não dado da conta.** Nada aqui vem das vendas reais — para lucro realizado use a skill `lucro` (`get_profit_waterfall`).
- **Nunca esconda as premissas.** Categoria assumida (15%), peso estimado ou imposto zerado mudam a conclusão — diga o que foi assumido.
- **Sem `cost` não fale em ROI nem em "compensa"** — sem custo só existem tarifas.
- BSR estimado é **média de categoria**; não apresente como "esse produto vende X".
- FBM só faz sentido com `shipping_cost`; DBA só com `dba_region` — senão a logística sai errada.
