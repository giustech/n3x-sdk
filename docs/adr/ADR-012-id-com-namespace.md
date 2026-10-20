# ADR-012: ID do plugin no formato `<namespace>.<nome>`

Status: Aceito
Data: 2026-10-19

## Contexto
O ID do plugin aparece no `n3x.yaml` do serviço, no `n3x.yaml` do controller, no nome do serviço gRPC, nas rotas do painel, nas métricas e nos logs. Os HLDs anteriores usavam três formatos diferentes (`n3x.example/header`, `n3x.core` e `core`).

## Decisão
- Formato: `<namespace>.<nome>`, em minúsculas, com letras, números e `_`, e cada parte com 2 a 32 caracteres. Exemplos: `n3x.core`, `n3x.example` e `giustech.ratelimit`
- Derivações:

| Onde | Regra | Exemplo |
| --- | --- | --- |
| `n3x.yaml` do serviço e do controller | O ID inteiro | `giustech.ratelimit` |
| Serviço gRPC | `n3x.plugins.<namespace>.<nome>.v<major>` | `n3x.plugins.giustech.ratelimit.v1` |
| Rotas do painel | `/plugins/<namespace>/<nome>/` | `/plugins/giustech/ratelimit/` |
| Métricas do plugin | Prefixo `n3x_<namespace>_<nome>_` | `n3x_giustech_ratelimit_blocked_total` |
| Rótulo `plugin` em métricas e logs | O ID inteiro | `plugin="giustech.ratelimit"` |

- O namespace `n3x` fica reservado aos plugins oficiais. O build recusa um ID `n3x.*` vindo de um módulo fora de `github.com/giustech/`
- Dois plugins com o mesmo ID fazem o build falhar
- O login do plugin core passa a ser `n3x.plugins.n3x.core.v1.Login`

## Justificativa
- Um formato só, derivável de forma mecânica em todos os lugares
- O namespace evita colisão entre autores quando houver plugins de outras origens

## Trade-off
- Nomes gRPC e rotas mais longos
- Exige atualizar o PRD e os HLDs de plataforma, sidecar e controller, que usavam `n3x.plugins.core.v1` e `/plugins/<id>/`

## Requisitos relacionados
FR-004, FR-005, FR-011, FR-012, FR-013, FR-015 e FR-016.
