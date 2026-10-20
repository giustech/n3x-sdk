# ADR-022: Métricas com prefixo por plugin e sem limite de cardinalidade

Status: Aceito
Data: 2026-10-19

## Contexto
Os plugins precisam publicar métricas próprias sem colidir com as do core (`n3x_plugin_*`) nem com as de outros plugins, e sem alocar no caminho quente. Avaliou-se limitar a 100 valores por rótulo.

## Decisão
- `env.Metrics()` oferece counter, gauge e histogram, criados no `New` e usados por handle (`m.With("burst").Inc()`), sem busca por nome a cada requisição
- Nome sempre com o prefixo `n3x_<namespace>_<nome>_` (ADR-012) e rótulo `instance` automático
- No sidecar viram stats do Envoy (`17090`); no controller, métricas Prometheus (`17091`)
- O SDK publica sozinho `n3x_plugin_duration_seconds` por hook, `n3x_plugin_memory_bytes` e `n3x_plugin_memory_evictions_total`
- Não há limite de cardinalidade nos rótulos; a responsabilidade é do autor

## Justificativa
- Nenhuma colisão de nomes entre plugins
- Sem custo de checagem de cardinalidade no caminho quente e sem perda de dados com `other`

## Trade-off
- Um rótulo com valores vindos do usuário (ex.: IP) cria séries sem fim e pode levar o sidecar ao OOM. Mitigação: documentação com exemplos certo e errado e alerta sugerido sobre o número de séries por plugin

## Requisitos relacionados
FR-012 e FR-013; observabilidade e desempenho nos requisitos não funcionais.
