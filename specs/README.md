# Índice das especificações do n3x-sdk

Este índice relaciona cada diretório de `specs/` às FDDs que ele cobre, na ordem planejada de execução. Cada especificação depende apenas das anteriores.

Os diretórios estão vazios. Ainda não foi executada nenhuma fase do GitHub Spec Kit (`specify`, `clarify`, `plan`, `tasks`, `implement`).

## Documentos de contexto (valem para todas as especificações)

| Documento | Caminho |
| --- | --- |
| PRD (v1.6) | [`docs/prd/PRD-N3X.md`](../docs/prd/PRD-N3X.md) |
| HLD do SDK (v1.2) | [`docs/HLD.md`](../docs/HLD.md) |
| HLD de plataforma (v1.2) | [`docs/HLD-plataforma.md`](../docs/HLD-plataforma.md) |
| ADRs | [`docs/adr/README.md`](../docs/adr/README.md) |

Cada FDD tem, perto do início, os blocos "Decisões aprovadas na clarificação" e "Pendências não bloqueantes". Eles registram o que foi decidido na clarificação e o que continua aberto.

## Visão geral

| # | Diretório | FDDs | Situação |
| --- | --- | --- | --- |
| 01 | [`01_identidade_e_config`](01_identidade_e_config) | SDK-01, SDK-02 | Pronto |
| 02 | [`02_segredos_e_memoria`](02_segredos_e_memoria) | SDK-03, SDK-04 | Pronto |
| 03 | [`03_contrato_proxy_e_ambiente`](03_contrato_proxy_e_ambiente) | SDK-05, SDK-06 | Pronto |
| 04 | [`04_contrato_controller_e_core`](04_contrato_controller_e_core) | SDK-08, SDK-09 | Pronto |
| 05 | [`05_widgets_e_painel`](05_widgets_e_painel) | SDK-10, SDK-11 | Pronto |
| 06 | [`06_kit_testes_proxy`](06_kit_testes_proxy) | SDK-07 | Pronto |
| 07 | [`07_kit_testes_controller`](07_kit_testes_controller) | SDK-12 | Pronto |
| 08 | [`08_plugin_exemplo_e_gerador`](08_plugin_exemplo_e_gerador) | SDK-13, SDK-14 | Pronto |
| 09 | [`09_deploy_e_guias`](09_deploy_e_guias) | SDK-15 | Escopo definido, depende de repositórios externos |
| 10 | [`10_integracao_e_ci_cruzada`](10_integracao_e_ci_cruzada) | SDK-16 | Escopo definido, depende de repositórios externos |

## Especificações

### 01. `01_identidade_e_config`

Identidade do plugin, fases, manifesto e configuração tipada com schema gerado.

- Entrega: `plugin/` (ID, fases, políticas de falha, manifesto, config tipada), `yamlconv/` e `schemagen/`.
- Depende de: nenhuma.
- FDDs:
  - [FDD-SDK-01: Identidade do plugin, fases, políticas de falha e manifesto](../docs/fdd/FDD-SDK-01-identidade-fases-e-manifesto.md)
  - [FDD-SDK-02: Config tipada, schema gerado da struct e decodificação estrita](../docs/fdd/FDD-SDK-02-config-tipada-e-schema.md)

### 02. `02_segredos_e_memoria`

Tratamento de segredos e memória isolada por plugin.

- Entrega: `plugin.Secret`, `secret/` (formato `ENC[...]`) e `memory/`.
- Depende de: 01.
- FDDs:
  - [FDD-SDK-03: Segredos, `plugin.Secret` e `ENC[...]`](../docs/fdd/FDD-SDK-03-segredos-plugin-secret-e-enc.md)
  - [FDD-SDK-04: Memória isolada por plugin](../docs/fdd/FDD-SDK-04-memoria-isolada.md)

### 03. `03_contrato_proxy_e_ambiente`

Contrato da metade de proxy e o ambiente que o plugin recebe.

- Entrega: `proxy/` (hooks, `Stream`, ações, negação), `proxy.Env` e `telemetry/`.
- Depende de: 01, 02.
- FDDs:
  - [FDD-SDK-05: Contrato de proxy, hooks, `Stream` e ações](../docs/fdd/FDD-SDK-05-contrato-de-proxy.md)
  - [FDD-SDK-06: Ambiente do proxy, `proxy.Env`](../docs/fdd/FDD-SDK-06-ambiente-do-proxy.md)

### 04. `04_contrato_controller_e_core`

Contrato da metade de controller e as capacidades privilegiadas do plugin core.

- Entrega: `controller/` (pacote `ctrl`), capacidades, `Caller`, estado do plugin, API entre plugins, `CoreEnv`, `Sidecar`, `HostValidator`.
- Depende de: 01, 02, 03.
- FDDs:
  - [FDD-SDK-08: Contrato de controller, capacidades, `Caller` e estado do plugin](../docs/fdd/FDD-SDK-08-contrato-de-controller.md)
  - [FDD-SDK-09: Capacidades privilegiadas do plugin core](../docs/fdd/FDD-SDK-09-capacidades-privilegiadas-do-core.md)

### 05. `05_widgets_e_painel`

Widgets do painel, renderizador, interação, gráficos e exportação.

- Entrega: `controller/ui` e `controller/ui/uitest`.
- Depende de: 01, 04.
- FDDs:
  - [FDD-SDK-10: Widgets do painel e renderizador](../docs/fdd/FDD-SDK-10-widgets-e-renderizador.md)
  - [FDD-SDK-11: Widgets interativos, gráficos e exportação](../docs/fdd/FDD-SDK-11-widgets-interativos-graficos-e-exportacao.md)

### 06. `06_kit_testes_proxy`

Kit de testes da metade de proxy, sem Envoy e sem controller.

- Entrega: `sdktest/`, `chain/` (lógica pura da cadeia, compartilhada com o executor do sidecar) e `sdktest/conformance`.
- Depende de: 03, 04.
- FDDs:
  - [FDD-SDK-07: Kit de testes da metade de proxy](../docs/fdd/FDD-SDK-07-kit-de-testes-do-proxy.md)

### 07. `07_kit_testes_controller`

Kit de testes da metade de controller, com host falso e gRPC em memória.

- Entrega: `sdktest/ctrltest`, `sdktest/hostconformance` e `controller/hostrules`.
- Depende de: 04, 05, 06.
- FDDs:
  - [FDD-SDK-12: Kit de testes da metade de controller](../docs/fdd/FDD-SDK-12-kit-de-testes-do-controller.md)

### 08. `08_plugin_exemplo_e_gerador`

Plugin de exemplo `n3x.example` e o gerador de plugins com checagem de evolução do schema.

- Entrega: `example/` (módulo aninhado `github.com/giustech/n3x-sdk/example`) e `cmd/n3x-new` (`new` e `schema-check`).
- Depende de: 03, 04, 05, 06, 07.
- FDDs:
  - [FDD-SDK-13: Plugin de exemplo `n3x.example`](../docs/fdd/FDD-SDK-13-plugin-de-exemplo.md)
  - [FDD-SDK-14: Gerador `n3x-new` e `schema-check`](../docs/fdd/FDD-SDK-14-gerador-n3x-new-e-schema-check.md)

### 09. `09_deploy_e_guias`

Topologias de deploy de referência e os guias do operador e do autor.

- Entrega: `deploy/` (docker compose, Kubernetes, ECS e Istio), `deploy/tools` (`deploytool`), guias.
- Depende de: 08.
- Dependências externas: imagens e binários do sidecar, controller e builder; conteúdo do Istio (FDD 49); estágio `out` e `compat/pairs.json` (FDD 46).
- FDDs:
  - [FDD-SDK-15: Topologias de deploy e guias do operador e do autor](../docs/fdd/FDD-SDK-15-topologias-de-deploy-e-guias.md)

### 10. `10_integracao_e_ci_cruzada`

Integração ponta a ponta e CI cruzada entre os repositórios.

- Entrega: harness `n3x-e2e` (`deploy/e2e`), cenários, matriz de CI e gate de release.
- Depende de: 09 e das especificações anteriores.
- Dependências externas: FDD 49 (Istio, bloqueia a v1.0.0), FDD 46, FDD 51 (carga simulada), repasse de `GOTRACEBACK=crash` pelo sidecar.
- FDDs:
  - [FDD-SDK-16: Integração ponta a ponta e CI cruzada](../docs/fdd/FDD-SDK-16-integracao-ponta-a-ponta-e-ci-cruzada.md)

## Referência cruzada: FDD para especificação

| FDD | Arquivo | Especificação |
| --- | --- | --- |
| SDK-01 | [`FDD-SDK-01-identidade-fases-e-manifesto.md`](../docs/fdd/FDD-SDK-01-identidade-fases-e-manifesto.md) | 01 |
| SDK-02 | [`FDD-SDK-02-config-tipada-e-schema.md`](../docs/fdd/FDD-SDK-02-config-tipada-e-schema.md) | 01 |
| SDK-03 | [`FDD-SDK-03-segredos-plugin-secret-e-enc.md`](../docs/fdd/FDD-SDK-03-segredos-plugin-secret-e-enc.md) | 02 |
| SDK-04 | [`FDD-SDK-04-memoria-isolada.md`](../docs/fdd/FDD-SDK-04-memoria-isolada.md) | 02 |
| SDK-05 | [`FDD-SDK-05-contrato-de-proxy.md`](../docs/fdd/FDD-SDK-05-contrato-de-proxy.md) | 03 |
| SDK-06 | [`FDD-SDK-06-ambiente-do-proxy.md`](../docs/fdd/FDD-SDK-06-ambiente-do-proxy.md) | 03 |
| SDK-07 | [`FDD-SDK-07-kit-de-testes-do-proxy.md`](../docs/fdd/FDD-SDK-07-kit-de-testes-do-proxy.md) | 06 |
| SDK-08 | [`FDD-SDK-08-contrato-de-controller.md`](../docs/fdd/FDD-SDK-08-contrato-de-controller.md) | 04 |
| SDK-09 | [`FDD-SDK-09-capacidades-privilegiadas-do-core.md`](../docs/fdd/FDD-SDK-09-capacidades-privilegiadas-do-core.md) | 04 |
| SDK-10 | [`FDD-SDK-10-widgets-e-renderizador.md`](../docs/fdd/FDD-SDK-10-widgets-e-renderizador.md) | 05 |
| SDK-11 | [`FDD-SDK-11-widgets-interativos-graficos-e-exportacao.md`](../docs/fdd/FDD-SDK-11-widgets-interativos-graficos-e-exportacao.md) | 05 |
| SDK-12 | [`FDD-SDK-12-kit-de-testes-do-controller.md`](../docs/fdd/FDD-SDK-12-kit-de-testes-do-controller.md) | 07 |
| SDK-13 | [`FDD-SDK-13-plugin-de-exemplo.md`](../docs/fdd/FDD-SDK-13-plugin-de-exemplo.md) | 08 |
| SDK-14 | [`FDD-SDK-14-gerador-n3x-new-e-schema-check.md`](../docs/fdd/FDD-SDK-14-gerador-n3x-new-e-schema-check.md) | 08 |
| SDK-15 | [`FDD-SDK-15-topologias-de-deploy-e-guias.md`](../docs/fdd/FDD-SDK-15-topologias-de-deploy-e-guias.md) | 09 |
| SDK-16 | [`FDD-SDK-16-integracao-ponta-a-ponta-e-ci-cruzada.md`](../docs/fdd/FDD-SDK-16-integracao-ponta-a-ponta-e-ci-cruzada.md) | 10 |

## Observações

- As FDDs que citam números acima de 16 (por exemplo "FDD 17" ou "FDD 49") referem-se a FDDs de outros repositórios do N3X (sidecar, controller, builder e plugin core), que não estão neste repositório.
- Os arquivos `.json` ao lado das FDDs são duplicatas e não são fonte da verdade. O `.md` é a fonte.
