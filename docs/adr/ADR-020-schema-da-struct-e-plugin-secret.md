# ADR-020: Schema gerado da struct, `plugin.Secret` e evolução só por adição

Status: Aceito (atualizado em 2026-10-22)
Data: 2026-10-19

## Contexto
O schema da config é usado na validação do `n3x.yaml`, no mascaramento de segredos enviado ao controller, nos nomes e descrições do painel e na config tipada que o plugin recebe. Também é preciso impedir que um segredo vaze por um log ou um `fmt` do autor, e que uma versão nova do plugin quebre o `n3x.yaml` existente.

## Decisão
- **Fonte única:** structs Go em `types/`, com as tags `json`, `n3x` (`required`, `min`, `max` e `enum`), `desc` e `default`. O `n3x-plugin build` gera o JSON Schema e o embute no manifesto do `.so` e do binário do controller
- **Segredo por tipo:** campos sensíveis usam `plugin.Secret`, que já marca o campo como segredo no schema. Log, `fmt`, JSON e erros mostram `***`, e o valor real só sai com `.Reveal()`. A análise estática recusa `.Reveal()` dentro de chamada de log, e o kit de testes confere que nenhum `Secret` aparece em logs
- **Evolução:** dentro da major do plugin, a config só muda por adição. Campo novo precisa ter `default` ou ser opcional; remover ou renomear exige major nova. A checagem roda na CI do plugin com `n3x-new schema-check --against <última tag>`

## Justificativa
- A struct que o plugin recebe é a mesma que valida, sem duas fontes divergindo
- O tipo protege em todos os caminhos de saída, não só no mascaramento enviado ao controller
- O operador atualiza a imagem sem precisar mudar o `n3x.yaml`

## Trade-off
- Só o subconjunto do JSON Schema expressável por tags
- `plugin.Secret` exige `.Reveal()` explícito em cada uso

## Requisitos relacionados
FR-001, FR-007, FR-012, FR-013 e FR-016; segurança nos requisitos não funcionais.

## Atualização (2026-10-22)
Decisões aprovadas na clarificação das FDDs SDK-01 a SDK-16:
- `desc` é obrigatória e `json` é explícito em snake_case
- Duração em texto Go; tamanho em base 1024
- `plugin.Secret` fica atrás de ponteiro para string e mostra `***` em qualquer saída
- AAD do controller: `n3x:v1:controller:<id do plugin>`; AAD do sidecar: `n3x:v1:<serviço>`
- A evolução é só por adição dentro da major do plugin; na série 0.x, a mesma linha é a mesma minor
- Código de build `N3X-B014`
