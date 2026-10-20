# ADR-017: Memória isolada com mapas tipados, cota e descarte do mais antigo

Status: Aceito
Data: 2026-10-19

## Contexto
O ADR-005 definiu o espaço de memória isolado por plugin, mas deixou para o HLD do SDK a API e a cota padrão.

## Decisão
- API: `memory.NewMap[K, V](env.Memory(), "nome")`, criada no `New`, presa ao ID do plugin, igual no sidecar e no controller. Operações `Get`, `Set` (com TTL opcional), `Delete` e `Len`
- Cota por plugin em bytes estimados (chave + valor): 16 MiB no sidecar e 64 MiB no controller, ajustável em `memory.max_bytes` na config do plugin
- Cota cheia: o item mais antigo é descartado (aproximação estilo CLOCK) e o novo é aceito; `n3x_plugin_memory_evictions_total` sobe. O plugin nunca recebe erro no caminho quente
- Implementação dividida em 64 partes, com trava por parte, e `Get` sem alocação para tipos de tamanho fixo
- Uso exposto em `n3x_plugin_memory_bytes`

## Justificativa
- Combina com dados voláteis (cache e contadores)
- Não exige tratamento de erro em cada `Set`, o que evita perda silenciosa por erro ignorado

## Trade-off
- Um plugin que precisa de todos os itens perde dados sem perceber; quem precisa de garantia usa o próprio banco
- A estimativa de bytes não é exata para tipos com ponteiros

## Requisitos relacionados
FR-012 e FR-013; ADR-005.
