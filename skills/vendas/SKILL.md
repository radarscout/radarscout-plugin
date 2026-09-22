---
name: vendas
description: "Resume o desempenho de vendas de uma conta de seller num período — GMV (faturamento bruto), itens de pedido, ticket médio e lucro líquido MC3 — e destaca os produtos campeões. Também abre pedido a pedido, para conferir com o ERP. Use quando o usuário perguntar como foram as vendas, pedir um panorama de faturamento ou desempenho, falar em GMV/ticket médio, quiser comparar períodos, ou pedir a lista de pedidos de um dia."
---

# Revisão de vendas

Entrega um panorama interpretado das vendas de um período: indicadores de topo + produtos que mais venderam. Não despeja linhas cruas — resume e contextualiza em pt-BR.

## Conta e período (pré-requisitos)

- **Conta:** toda consulta exige `seller_account_id`. Se ainda não foi resolvido nesta conversa, obtenha via tool de identidade (`whoami`) do servidor `radarscout`; se houver mais de uma conta, pergunte qual (ver skill `inicio`). Reaproveite o id nas chamadas seguintes.
- **Período:** datas ISO `YYYY-MM-DD`, fuso America/Sao_Paulo, **fim exclusivo** (`period_end` não conta). Se o usuário não der período, use os **últimos 30 dias** e diga qual janela usou. "Este mês" = dia 1 até amanhã (exclusivo). "Semana passada" = segunda a domingo da semana anterior.

## Passo a passo

1. Resolva `seller_account_id` (ver acima) e a janela de datas.
2. Chame a tool de panorama de vendas (`get_sales_overview`) do `radarscout` com `seller_account_id`, `period_start`, `period_end`. Ela traz GMV, número de itens de pedido, ticket médio e **lucro líquido MC3** (já após CMV, tarifas Amazon e anúncios).
3. Chame a tool de ranking de produtos (`list_top_products`) com a mesma janela. Padrão: `sort=profit` (lucro líquido). Use `sort=margin` se o usuário falar em margem. Ela sinaliza SKUs com **margem negativa** (prejuízo).
4. Para comparar períodos, repita os passos 2–3 na janela anterior e mostre as variações (R$ e %).
5. Quando o vendedor quiser **pedido a pedido** — conferir com o ERP, investigar um pedido específico —, chame `list_orders`. Ela devolve um registro por pedido (SKUs em `items`), com número do pedido, data, status e o valor decomposto.

Três cuidados com `list_orders`, porque ela não bate com os outros números por construção:

- **`items_total` não inclui frete.** Frete, desconto promocional e imposto vêm em campos separados — some o que precisar, não presuma que já está lá.
- **Sem `status_group`, a lista inclui cancelados**, que o GMV dos outros relatórios não inclui.
- **`total_orders` conta pedidos; o `orders` do panorama conta itens de pedido**, e vem de outra fonte. Não são comparáveis, e apresentá-los lado a lado como se fossem gera desconfiança no resto do número.

## O que entregar

- Abertura com os números-chave do período (GMV, pedidos, ticket médio, MC3), em reais.
- 3–5 produtos campeões por lucro (ou margem, se pedido).
- **Alerta** se algum top produto estiver com margem negativa.
- Em comparação de períodos: variação de cada indicador e uma frase de leitura ("vendeu mais mas lucrou menos porque…").

## Exemplo

> **Usuário:** "Como foram as vendas esse mês?"
> **Skill:** resolve a conta, define a janela (01→amanhã), chama panorama + ranking.
> **Resposta:** "Em junho (até hoje): **GMV R$ 84.200**, 612 itens, ticket médio R$ 137, **lucro MC3 R$ 11.900**. Campeões por lucro: 1) Fone X (R$ 2.100)… ⚠️ O SKU *Cabo Y* está no top de vendas mas com **margem negativa**."

## Cuidados

- MC3 já é líquido — não some tarifas/anúncios "de novo" por cima.
- **Sem vendas** (GMV 0 / lista vazia): diga claramente que não há vendas nessa conta e período; ofereça outra conta (o usuário pode ter várias) ou outra janela. A ingestão pode atrasar dados muito recentes.
- **MC3 agregado negativo:** abra com o alerta de **prejuízo** — vendeu, mas o lucro líquido ficou negativo —, não só com o GMV. Encaminhe à skill `lucro` para achar a etapa que virou o resultado.
- `list_top_products` pagina por cursor; para um panorama, a primeira página (até 50) basta. Não varra todas as páginas sem o usuário pedir.
- `list_orders` também pagina por cursor. Para conferência de um dia, uma página costuma bastar; varrer o mês inteiro sem o vendedor pedir é gasto à toa.
- **Nunca prometa dado do comprador.** A ferramenta de pedidos não devolve CPF nem endereço, de propósito.
- Se o lucro vier estranho, pode faltar CMV cadastrado — encaminhe para a skill `lucro` (que reporta a cobertura de CMV).
