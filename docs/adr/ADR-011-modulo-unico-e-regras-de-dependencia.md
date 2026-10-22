# ADR-011: Módulo único com pacotes por metade e regras de dependência

Status: Aceito (atualizado em 2026-10-22)
Data: 2026-10-19

## Contexto
O SDK atende duas metades com necessidades opostas. A metade de proxy roda dentro do Envoy e precisa ser enxuta, com 0 alocações e sem dependências pesadas. A metade de controller precisa de gRPC, de widgets e de um renderizador de HTML. Uma pessoa só mantém os cinco repositórios, e o catálogo de alvos do builder precisa fixar uma combinação simples de versões.

## Decisão
- Um único módulo Go, `github.com/giustech/n3x-sdk`, com uma única versão semver
- Pacotes: `plugin/` (manifesto, IDs, fases, políticas e schema), `proxy/`, `controller/`, `controller/ui`, `memory/`, `secret/`, `sdktest/`, `example/` e `cmd/n3x-new`
- Regras de dependência, conferidas na CI do SDK e na análise estática do `n3x-plugin build`:
  - sem cgo
  - nenhum pacote importa o SDK Go do Envoy nem pacotes internos do `n3x-sidecar` ou do `n3x-controller`
  - `proxy/` e `memory/` usam só a biblioteca padrão e grpc-go (para a interface da conexão compartilhada)
  - `proxy/` nunca importa `controller/`
  - o Echo não aparece no contrato
  - código de plugin não importa `secret/` (ADR-014)

## Justificativa
- Uma versão só para manter e para fixar no catálogo de alvos
- As duas metades de um plugin sempre usam a mesma versão do contrato
- O Go só compila os pacotes importados, então o `.so` não carrega o renderizador de widgets

## Trade-off
- As dependências da metade de controller aparecem no `go.mod` de quem só usa a metade de proxy, ainda que não entrem no binário
- O isolamento entre `proxy/` e `controller/` depende de uma regra de CI, e não da fronteira de módulo

## Requisitos relacionados
FR-012, FR-013, FR-014 e FR-015; desempenho nos requisitos não funcionais.

## Atualização (2026-10-22)
Alinhamento com a clarificação das FDDs SDK-01 a SDK-16. O corpo acima foi mantido; o que mudou:
- O módulo do SDK continua único, mas passa a ter módulos aninhados: `example/` (`github.com/giustech/n3x-sdk/example`, com `go.mod` próprio, `go.work` e tags `example/vX.Y.Z`), `deploy/tools`, `deploy/e2e` (com `plugins/faults`) e `deploy/apps/eco`. Onde o corpo descreve um único `go.mod`, vale esta exceção.
- Pacotes novos no módulo: `yamlconv`, `schemagen`, `telemetry`, `chain`, `internal/{clock,lifecycle,privileged,scaffold,schemadiff}`, `sdktest/{conformance,ctrltest,hostconformance}` e `controller/{hostrules,ui,ui/uitest}`.
- Regras de dependência acrescentadas:
  - `yamlconv` é o único pacote com dependência de terceiros YAML (`go.yaml.in/yaml/v3`).
  - `proxy/` importa `telemetry/`, `memory/`, `plugin/` e grpc-go.
  - `chain/` importa a stdlib, `plugin/` e `proxy/`; é pacote de plataforma, fora do alcance de plugins (o sidecar importa o SDK, o inverso continua proibido).
  - `controller/ui` não importa `ctrl`.
  - `controller/hostrules` e `internal/*` ficam fora do alcance de plugins.
