# Análise do push de estoque em tempo real

## Pontos que podem impedir o envio
- **Debounce por quantidade**: a chave gerada por `tdml_ml_debounce_key()` inclui a quantidade atual, então alterações repetidas que resultem no mesmo valor serão ignoradas pelo `tdml_ml_push_stock_debounced_for_user()` por 10 segundos, registrando `DEBOUNCE skip` no log. Isso evita loops, mas faz parecer que nada foi enviado quando o estoque não mudou de fato.
- **Variação sem `variation_id` no Mercado Livre**: `tdml_ml_push_stock_for_user()` pula (`SKIP var não existe no ML`) quando não encontra o `variation_ml_id` para a variação. Se o reindex automático falhar, é necessário reexportar o anúncio ou ajustar o mapa `_tdml_ml_variation_ids`/`_tdml_ml_variation_skus`.
- **Anúncio fechado**: respostas que contenham `status closed` ou `variations is not modifiable` fazem o fluxo retornar sucesso sem atualizar o estoque. Verifique o status do item no ML.
- **Token ou item ausente**: faltando `_td_ml_access_token` ou `_tdml_ml_item_ids`, a função retorna erro ou não encontra a conta correta. É preciso garantir que o produto esteja vinculado à conta ML e que o token esteja válido.
- **Hooks não disparados**: o sync é acionado por `updated_post_meta`/`added_post_meta` para `_stock` e `_stock_status`, e por hooks de estoque do WooCommerce (`woocommerce_product_set_stock`, etc.). Se outro plugin altera o estoque por fora desses hooks, o push não será chamado. Confirme que as atualizações passam pelas APIs do Woo.
- **Fallback por posição falhando**: o reindex tenta casar variações por SKU/atributos e, se não conseguir, usa a ordem. Se as contagens não batem, nenhuma variação é associada e o push subsequente não encontra IDs, gerando skips.

## Recomendações rápidas de diagnóstico
1. Ative o log (`TDML_DEBUG_STOCK=true`) e force uma atualização de estoque para ver mensagens `DEBOUNCE skip`, `SKIP var não existe no ML` ou `ERRO HTTP ...`.
2. Confira o mapa de item e variações nos metas `_tdml_ml_item_ids`, `_tdml_ml_variation_ids` e `_tdml_ml_variation_skus` do produto/variações que falham.
3. Valide o token da conta em `_td_ml_access_token` e o `seller_id` em `_td_ml_user_id`.
4. Teste uma alteração de estoque via interface do WooCommerce (não direto no banco) para garantir que os hooks de disparo estão sendo executados.
5. Se persistir, reexporte o anúncio/variações para repopular os IDs de variação e refaça o reindex.
