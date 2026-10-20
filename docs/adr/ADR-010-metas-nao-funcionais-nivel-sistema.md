# ADR-010: Metas não funcionais no nível do sistema

Status: Aceito
Data: 2026-10-19

## Contexto
O PRD define metas de latência, recursos, disponibilidade e segurança para o sistema inteiro. Era preciso decidir se o HLD de plataforma dividiria essas metas por componente.

## Decisão
O HLD de plataforma registra as metas só no nível do sistema. A divisão por componente (por exemplo, quanto do p99 de 2 ms cabe ao core do `.so` e a cada plugin) fica no HLD de cada repositório.

## Justificativa
- HLD de plataforma curto, sem contratos que têm dono
- Cada repositório conhece melhor o próprio orçamento

## Trade-off
- A soma dos orçamentos por componente precisa ser conferida contra a meta do sistema em cada HLD de repositório

## Requisitos relacionados
Todos os requisitos não funcionais do PRD.
