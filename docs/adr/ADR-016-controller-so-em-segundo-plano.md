# ADR-016: Controller só em segundo plano; conexões próprias permitidas na requisição

Status: Aceito
Data: 2026-10-19

## Contexto
O PRD exige zero erros da aplicação causados pelo controller. Ao mesmo tempo, plugins como um rate limit global precisam consultar dados compartilhados com baixa latência. O modelo de referência é este: o sidecar bloqueia um IP e avisa o controller em segundo plano; a metade de controller grava no Redis master; os sidecars leem uma réplica de leitura, que pode ser um container ao lado do sidecar.

## Decisão
| Conexão | Durante a requisição (`s.Go()`) | Em segundo plano (`env.Background()`) |
| --- | --- | --- |
| Controller (`env.Controller()`, conexão compartilhada) | Proibida: devolve `proxy.ErrRequestPath` na hora | Permitida, com teto de 5 s por chamada |
| Conexão própria do plugin (`env.Dial()`, ex.: réplica Redis) | Permitida, com o prazo do `Go()` (200 ms por padrão) e a política de falha da instância | Permitida |

- `go` cru é proibido. O SDK oferece `s.Go()` (preso à requisição), `env.Background(nome, fn)` (tarefa longa, com `recover`, reinício com backoff após panic e `ctx` cancelado no desligamento) e `proxy.Sleep(ctx, d)`
- `env.Dial()` e os clientes HTTP e TCP do SDK sempre aplicam prazo no transporte
- Limites por plugin, ajustáveis na config: até 8 tarefas de fundo (a 9ª falha no `New`) e até 16 conexões no `env.Dial()` (a 17ª espera vaga até o prazo do `Go()`)
- Endereços e senhas dos connectors ficam na `config` do plugin no `n3x.yaml` do serviço, com `ENC[...]`

## Justificativa
- O controller nunca entra no caminho do tráfego
- Dados compartilhados com baixa latência continuam possíveis, por réplicas locais
- Os limites transformam um backend lento em "plugin aplicando a política de falha", em vez de derrubar o sidecar

## Trade-off
- Bibliotecas de connector (ex.: client Redis) criam goroutines próprias, fora do `recover` do SDK; um panic nelas derruba o Envoy. Mitigação: lista de bibliotecas aprovadas e teste de longa duração
- O plugin decide pela memória local ou pela réplica, com dados possivelmente defasados

## Requisitos relacionados
FR-002, FR-004 e FR-012; disponibilidade nos requisitos não funcionais.
