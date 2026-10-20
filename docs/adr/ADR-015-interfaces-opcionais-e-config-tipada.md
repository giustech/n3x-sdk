# ADR-015: Hooks e capacidades como interfaces opcionais, com config tipada

Status: Aceito
Data: 2026-10-19

## Contexto
O executor do sidecar precisa saber, já na subida, quais hooks cada instância usa. Assim o body só cruza do Envoy para o Go quando alguém pede, e a meta de até 5 cruzamentos por requisição se mantém. O host do controller precisa do mesmo para as capacidades (gRPC, páginas, cartões e ciclo de vida).

## Decisão
- O plugin expõe uma função sem efeitos colaterais, chamada pelo registro que o builder gera (não há `init()` nem registro global):
  - `agent.Plugin() proxy.Plugin`, criado com `proxy.Define[C](New)`
  - `controller.Plugin() ctrl.Plugin`, criado com `ctrl.Define[C](New)`
- `New(cfg C, env Env) (Instance, error)` recebe a config já decodificada na struct `C` e validada pelo schema
- **Metade de proxy:** uma instância por `instance` do `n3x.yaml`. Hooks opcionais: `OnRequestHeaders`, `OnRequestBody`, `OnResponseHeaders` e `OnComplete`. O executor detecta por type assertion quais existem
- **Metade de controller:** uma instância por plugin. Capacidades opcionais: `Start`, `Stop`, `GRPC`, `Routes`, `Cards` e `Validate`. Os handlers gRPC são registrados pelo próprio plugin, com o stub gerado do `.proto`, e chamados pelo grpc-go por nome de método. Não existe um método genérico de execução
- O hook de body de resposta fica fora da v1 e entra depois como `OnResponseBody`, por adição

## Justificativa
- Um hook não implementado não custa nada
- Config tipada pelo compilador, sem `map[string]any`
- Modelo de registro igual ao do gRPC e do `net/http`, já conhecido pelos autores

## Trade-off
- As capacidades ficam menos visíveis na IDE do que numa interface única
- Generics deixam as mensagens de erro de compilação mais longas

## Requisitos relacionados
FR-004, FR-012 e FR-013.
