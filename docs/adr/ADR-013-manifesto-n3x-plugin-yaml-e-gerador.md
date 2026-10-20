# ADR-013: Manifesto `n3x-plugin.yaml` e gerador `n3x-new`

Status: Aceito
Data: 2026-10-19

## Contexto
O PRD chamava de `n3x.yaml` três arquivos diferentes: a config do serviço, a config do controller e o manifesto na raiz do repositório do plugin. Além disso, criar um plugin à mão exige escrever o ID em cinco lugares (manifesto, `proto/`, rotas, métricas e `go.mod`), o que costuma fazer o primeiro build falhar.

## Decisão
- O manifesto na raiz do repositório do plugin passa a se chamar `n3x-plugin.yaml`. Os dois arquivos do operador continuam como `n3x.yaml`
- Campos do manifesto: `id`, `version`, `sdk` (faixa semver aceita), `halves` (`proxy` e `controller`), `proxy.phases`, `proxy.default_phase`, `proxy.default_order`, `proxy.default_failure`, `controller.required` e `controller.depends_on`
- O SDK traz o gerador `cmd/n3x-new`:
  - `n3x-new <id> --halves proxy,controller` cria a estrutura a partir do `example/`, com o ID, o pacote gRPC, as rotas e os testes já preenchidos
  - `n3x-new schema-check --against <tag>` compara o schema da config com o de uma versão anterior, na CI do plugin (ADR-020)

## Justificativa
- O nome deixa claro que o arquivo é do plugin, e o `n3x-plugin add` procura exatamente esse arquivo
- O gerador elimina o erro mais comum, que é esquecer de renomear o namespace

## Trade-off
- Uma ferramenta a mais para manter no SDK
- Muda uma linha do FR-014 do PRD

## Requisitos relacionados
FR-014 e FR-015.
