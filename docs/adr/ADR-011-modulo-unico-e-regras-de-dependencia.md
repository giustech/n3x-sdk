# ADR-011: Módulo único com pacotes por metade e regras de dependência

Status: Aceito
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
