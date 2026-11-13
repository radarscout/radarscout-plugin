---
name: anuncios
description: "Diagnostica os anúncios do seller na Amazon e avalia a qualidade de cada um a partir do que a própria Amazon informa — se pode ser comprado e por quê não, se aparece na busca, problemas que impedem a venda, sugestões de qualidade do catálogo, logística, estoque e o conteúdo da página (título, tópicos, descrição, imagens). Use quando o usuário perguntar por que um anúncio não vende, não aparece ou está inativo, quais anúncios têm problema, o que corrigir primeiro, ou pedir uma avaliação da qualidade de um anúncio."
---

# Diagnóstico e qualidade do anúncio

Responde três perguntas, nesta ordem de importância:

1. **Pode ser vendido?** (`buyable`) — se não, por quê.
2. **Aparece na busca?** (`discoverable`)
3. **A página está bem feita?** — conteúdo e sugestões de qualidade da Amazon.

A leitura vem **da Amazon, na hora**. Os números do Radar sobre o anúncio (vendas, estoque, Buy Box, preço mínimo, reposição) são outra coisa: skill `ofertas`.

## Conta

- Exige `seller_account_id` (via `whoami` do `radarscout`). Hoje só contas **Amazon**.

## Passo a passo

### Quais anúncios têm problema
1. Tool `list_listing_problems` do `radarscout`. Vem **problemas** (`problems`, impedem venda ou exibição) separados de **sugestões** (`suggestions`, qualidade do catálogo, não impedem). Ordem: com problema → não compráveis → só sugestão. `only_problems: true` deixa de fora os só com sugestão. Até 100 anúncios; `truncated` avisa que há mais.

### Um anúncio a fundo
2. Tool `check_listing` do `radarscout` por `offer_id`, `sku` ou `asin`: tudo da lista mais logística (`FBA`/`FBM`), estoque, conteúdo completo da página (título, marca, tópicos, descrição como a Amazon guarda, quantidade de imagens), preço, condição, tipo de produto e datas.
3. Para vendas, ritmo, cobertura e compra sugerida do mesmo SKU, complemente com `get_seller_offer` (skill `ofertas`). Para a saúde da conta inteira (defeitos, cancelamentos, violações), `get_account_health`.

## Como ler

### Não pode ser comprado (`not_buyable_reason`)

| Valor | O que dizer |
| --- | --- |
| `marketplace_problem` | Há problema da Amazon em `problems` — é ele que trava. Explique a mensagem dela. |
| `out_of_stock` | Está **sem estoque**. Não é problema do anúncio: é reposição (skill `ofertas` / `build_replenishment_plan`). |
| `unknown` | A Amazon não diz o motivo. **Não invente.** Sugira conferir no Seller Central. |

### Problema × sugestão
- **Problema (ERROR):** impede a venda ou a exibição. Corrigir primeiro.
- **Sugestão (WARNING)**, como o código 18448 ("faltam alguns atributos principais"): **não impede** a venda. Em revenda, esses atributos costumam ser do catálogo da marca, e o vendedor nem sempre consegue mudá-los — e o Seller Central pode nem mostrar esse aviso na tela de estoque. Não trate como urgência.

### Qualidade da página (regras públicas da Amazon)
Avalie só com o que `check_listing` traz, citando que são orientações da Amazon:
- **Título:** presente e descritivo (marca, produto, característica principal, tamanho/quantidade). A Amazon limita o tamanho por categoria; título longo demais ou com promoção ("frete grátis", "promoção") vai contra as regras dela.
- **Tópicos (`bullets`):** a Amazon aceita até **5**. Menos de 5 é espaço não usado.
- **Descrição:** presente. Pode vir com HTML — é como a Amazon guarda.
- **Imagens (`image_count`):** mais de uma, mostrando o produto por ângulos e em uso. A tool não verifica fundo branco nem resolução — não afirme.
- **Sugestões de atributo:** liste os atributos citados (`attribute_names`) como o que completaria o cadastro.

Entregue uma **nota em três faixas**, sem inventar pontuação:
- **Corrigir já** — não pode ser comprado por problema da Amazon, ou não aparece na busca.
- **Ajustar** — vende, mas a página tem lacunas (menos de 5 tópicos, sem descrição, poucas imagens) ou há sugestões de atributo.
- **Pronto** — vende, aparece e a página está completa.

## Revenda × marca própria

Quem revende **não é dono** do conteúdo da página (título, tópicos, descrição, imagens são do catálogo da marca). Nesse caso a avaliação de conteúdo serve para **escolher o que vender**, não para editar; o que o revendedor controla é preço, estoque e logística. Se não souber se é marca própria, **pergunte** antes de recomendar editar a página.

## O que entregar

- Na lista: primeiro o que **trava a venda**, depois o que está **sem estoque**, e só então um resumo das sugestões ("26 anúncios com aviso: 1 com problema, 2 sem estoque, 23 só com sugestão de catálogo").
- No detalhe: a faixa (Corrigir já / Ajustar / Pronto), a causa em uma frase e o que fazer.

## Exemplo

> **Usuário:** "Por que o RDX0015 não está vendendo?"
> **Skill:** chama `check_listing` (sku RDX0015) e `get_seller_offer`.
> **Resposta:** "O RDX0015 está **sem estoque** no FBA — por isso não pode ser comprado. Ele vendia 3 por dia nos últimos 30 dias; a sugestão de compra é de 155 unidades. Há também uma sugestão da Amazon de completar peso e faixa etária no cadastro, mas isso **não** impede a venda e, por ser revenda, depende da marca. Prioridade: repor."

## Cuidados

- **Sem estoque não é problema de anúncio.** Nunca atribua "não comprável" a uma sugestão de atributo.
- **Não confunda** este diagnóstico (visão da Amazon) com `ofertas` (números do Radar).
- Não varra a lista inteira sem o usuário pedir; mostre o que exige ação e ofereça o resto.
- Esta skill **só lê e sugere**. Editar o anúncio não está disponível ainda.
