---
name: inicio
description: "Conecta o usuário ao Radar Scout, descobre suas contas de seller Amazon (seller_account_id) e gera o link para conectar uma conta nova pela Amazon. Use na primeira interação, quando o usuário não souber qual conta usar, quando disser que não vê seus dados, quando quiser conectar/adicionar outra conta Amazon, ou quando outra skill precisar de um seller_account_id e ainda não houver um."
---

# Primeiros passos no Radar Scout

Resolve **quem é o usuário** e **qual conta de seller** usar. Toda consulta de dados do Radar (vendas, lucro, repasses, repricer, produtos) exige um `seller_account_id` — este é o ponto de partida que o fornece.

## Quando usar

- Primeira interação do usuário com o Radar Scout.
- Usuário diz "não vejo meus dados", "não conectou", "qual conta?".
- Usuário quer **conectar uma conta Amazon nova** (ou uma segunda loja).
- Outra skill precisa de `seller_account_id` e nenhum foi resolvido ainda nesta conversa.

## Passo a passo

1. Chame a tool de identidade (`whoami`) do servidor `radarscout`. Ela não pede parâmetros e retorna o usuário Radar autenticado + as contas de seller (SellerAccounts) conectadas.
2. Trate o resultado:
   - **Nenhuma conta:** o usuário ainda não conectou uma conta Amazon. Explique em pt-BR que sem isso não há dados a consultar e ofereça conectar agora (veja **Conectar uma conta Amazon**).
   - **Uma conta:** confirme qual é (nome + id) e diga que vai usá-la nas próximas consultas. Guarde o `seller_account_id` para reaproveitar na conversa.
   - **Várias contas:** liste os nomes de forma legível e pergunte qual o usuário quer usar. Só depois siga para a consulta.
3. Se o usuário chegou com uma pergunta de dados (ex.: "como foram minhas vendas?"), encadeie: resolva a conta aqui e então deixe a skill apropriada (`vendas`, `lucro`, etc.) continuar com o `seller_account_id` resolvido.

## Conectar uma conta Amazon

Quando não houver conta conectada — ou o usuário quiser adicionar outra loja —, chame `connect_amazon_sp_api` do `radarscout`. Ele **não conecta nada sozinho**: devolve o link de autorização da Amazon que o usuário precisa abrir e aprovar no Seller Central.

- `region`: `NA` (padrão, inclui o Brasil), `EU` ou `FE`.
- `target_app`: `radar` (padrão), `analytics` ou `repricer` — a área do Radar à qual a conta será vinculada. Na dúvida, pergunte o que ele quer usar em vez de assumir.

Entregue o link com o que acontece depois: ele autoriza na Amazon, volta ao Radar e a conta aparece. **Só depois disso** o `whoami` vai listá-la — rode de novo para confirmar, e não afirme que conectou antes disso.

Se o usuário preferir a web, o mesmo fluxo existe em radarscout.com.br.

## O que entregar

- Confirmação clara de quem está logado e qual conta será usada.
- Se houver fricção (sem conta / login expirado), oriente o próximo passo em linguagem de vendedor, sem jargão de OAuth.

## Exemplo

> **Usuário:** "Oi, acho que não tô vendo nada aqui."
> **Skill:** chama `whoami`. Retorna 2 contas ("Loja Principal", "Loja Outlet").
> **Resposta:** "Você está conectado como João. Encontrei 2 contas de seller: **Loja Principal** e **Loja Outlet**. Qual delas você quer consultar?"

## Cuidados

- Nunca invente um `seller_account_id`. Ele vem sempre do `whoami`.
- **O link de autorização não conclui a conexão.** Enquanto o usuário não completar o fluxo na Amazon, a conta não existe — confirme com um novo `whoami` antes de dizer que está conectada.
- O id é interno (`SellerAccount.id`) — use-o nas chamadas, mas mostre ao usuário o **nome** da conta, não o id cru.
