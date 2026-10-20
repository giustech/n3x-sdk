# ADR-002: Armazenamento efêmero ou Postgres com migrations isoladas por plugin

Status: Substituído pelo ADR-002 do `n3x-controller` (controller sem armazenamento, cada plugin traz o próprio banco)
Data: 2026-10-19

> Esta decisão não vale mais. Na entrevista do HLD do `n3x-controller` ficou decidido que o host do controller não tem armazenamento: cada plugin que guarda dados traz a própria conexão de banco, e o plugin core guarda inventário, configs, usuários de leitura e auditoria (ADR-002 e ADR-003 do `n3x-controller`). O texto abaixo fica só como histórico.

## Contexto
O PRD deixou o armazenamento do controller para o HLD. O controller guarda estado observado, usuários de leitura e auditoria, e os plugins podem precisar de dados próprios (FR-011). O N3X é uma camada de sobrescrita de rede, não uma plataforma de dados.

## Decisão
- Dois modos, escolhidos no `n3x.yaml` do controller:
  - **Efêmero (padrão):** se o controller desligar, tudo se perde
  - **Postgres:** persistência; backup e retenção física são responsabilidade do operador
- No modo Postgres, o controller tem as próprias migrations e cada plugin pode trazer as suas
- **Isolamento entre plugins:** as migrations e o acesso a dados de um plugin nunca manipulam as tabelas de outro plugin. Um plugin acessa dados de outro só pela API pública dele

## Justificativa
- Quem precisa de consistência e retenção usa Postgres; o padrão continua leve (um docker compose, sem dependência extra)
- O estado observado se reconstrói sozinho, porque todo sidecar refaz login e registro ao reconectar

## Trade-off
- No modo efêmero, usuários de leitura e auditoria somem no restart; a retenção de 90 dias (FR-010) e a trilha de LGPD valem só no modo Postgres
- Mais um modo para testar na CI

## Pendências (HLD do `n3x-controller`)
- Mecanismo de isolamento (hipótese: um schema e um role Postgres por plugin, dono só do próprio schema, com o core em `n3x_core`)
- Comportamento na falha de migration (segue a regra de plugin obrigatório e opcional do FR-011)

## Requisitos relacionados
FR-006, FR-007, FR-009, FR-010, FR-011; compliance e confiabilidade nos requisitos não funcionais.

## Atualizado em 2026-10-20
O HLD do `n3x-plugin-core` decidiu que, na v1, o core também não usa banco: inventário, configs, eventos e usuários ficam em memória, atrás de uma porta `Store` que recebe um banco real na v2 (ADR-001 do `n3x-plugin-core`). A auditoria formal saiu da v1 (ADR-002 do `n3x-plugin-core`), e os usuários de leitura vêm do `n3x.yaml` (ADR-003 do `n3x-plugin-core`).
