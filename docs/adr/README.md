# ADRs da plataforma N3X

Decisões de arquitetura registradas no HLD de plataforma (`../HLD-plataforma.md`). Cada ADR traz contexto, decisão, justificativa, trade-off e requisitos relacionados do PRD (`../prd/PRD-N3X.md`).

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-controller-instancia-unica-por-ambiente.md) | Controller em instância única, sempre um por ambiente | Aceito (atualizado em 2026-10-20) |
| [ADR-002](ADR-002-armazenamento-efemero-ou-postgres.md) | Armazenamento efêmero ou Postgres com migrations isoladas por plugin | Substituído pelo ADR-002 do `n3x-controller` (nota de 2026-10-20) |
| [ADR-003](ADR-003-login-credencial-e-sessao-hmac.md) | Login por credencial de serviço e sessão HMAC sem estado | Aceito (atualizado em 2026-10-20) |
| [ADR-004](ADR-004-capacidades-privilegiadas-plugin-core.md) | Capacidades privilegiadas do plugin core | Aceito |
| [ADR-005](ADR-005-memoria-isolada-por-plugin.md) | Memória isolada por plugin no SDK | Aceito |
| [ADR-006](ADR-006-portas-17xxx-e-modo-istio.md) | Faixa de portas 17xxx e Istio no comando | Aceito |
| [ADR-007](ADR-007-sidecar-nativo-kubernetes.md) | Sidecar nativo no Kubernetes | Aceito |
| [ADR-008](ADR-008-exemplos-deploy-no-sdk.md) | Exemplos de deploy e CI cruzada em `n3x-sdk/deploy/` | Aceito |
| [ADR-009](ADR-009-sem-tracing-v1.md) | Sem tracing distribuído na v1 | Aceito |
| [ADR-010](ADR-010-metas-nao-funcionais-nivel-sistema.md) | Metas não funcionais no nível do sistema | Aceito |

## ADRs do SDK

Decisões registradas no HLD do SDK (`../HLD.md`).

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-011](ADR-011-modulo-unico-e-regras-de-dependencia.md) | Módulo único com pacotes por metade e regras de dependência | Aceito |
| [ADR-012](ADR-012-id-com-namespace.md) | ID do plugin no formato `<namespace>.<nome>` | Aceito |
| [ADR-013](ADR-013-manifesto-n3x-plugin-yaml-e-gerador.md) | Manifesto `n3x-plugin.yaml` e gerador `n3x-new` | Aceito |
| [ADR-014](ADR-014-pacote-secret-no-sdk.md) | Pacote `secret` no SDK, proibido aos plugins | Aceito |
| [ADR-015](ADR-015-interfaces-opcionais-e-config-tipada.md) | Hooks e capacidades como interfaces opcionais, com config tipada | Aceito |
| [ADR-016](ADR-016-controller-so-em-segundo-plano.md) | Controller só em segundo plano; conexões próprias permitidas na requisição | Aceito |
| [ADR-017](ADR-017-mapas-de-memoria-com-cota.md) | Memória isolada com mapas tipados, cota e descarte do mais antigo | Aceito |
| [ADR-018](ADR-018-telas-por-widgets.md) | Telas de plugin como árvore de widgets, sem HTML | Aceito |
| [ADR-019](ADR-019-contrato-das-capacidades-do-core.md) | Contrato das capacidades privilegiadas do plugin core | Aceito (atualizado em 2026-10-20) |
| [ADR-020](ADR-020-schema-da-struct-e-plugin-secret.md) | Schema gerado da struct, `plugin.Secret` e evolução só por adição | Aceito |
| [ADR-021](ADR-021-contrato-estavel-e-depreciacao.md) | Contrato estável desde a v1.0 e política de depreciação | Aceito |
| [ADR-022](ADR-022-metricas-por-plugin-sem-limite.md) | Métricas com prefixo por plugin e sem limite de cardinalidade | Aceito |
