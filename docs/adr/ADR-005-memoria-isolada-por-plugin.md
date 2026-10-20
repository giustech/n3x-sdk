# ADR-005: Memória isolada por plugin no SDK

Status: Aceito
Data: 2026-10-19

## Contexto
Plugins precisam guardar valores em memória no sidecar e no controller (por exemplo, o token de sessão do plugin core). Os dados de um plugin não podem ser lidos nem alterados por outro.

## Decisão
- O SDK define um espaço de memória por plugin, no sidecar e no controller
- Cada plugin recebe um handle preso ao próprio ID; não existe acessor global nem forma de pedir o espaço de outro ID
- A análise estática do `n3x-plugin build` proíbe `unsafe` e `reflect` sobre tipos do SDK
- Cota de memória por plugin (valor padrão a definir no HLD do `n3x-sdk`)
- Dados voláteis: perdidos no restart e não compartilhados entre réplicas; não substituem o banco próprio de cada plugin (ADR-002 do `n3x-controller`)
- No sidecar, a API é segura para concorrência entre os workers do Envoy e respeita a meta de 0 alocações no caminho quente

## Justificativa
- Isolamento definido no contrato, igual para todos os plugins, inclusive o core
- Permite ao core guardar o token sem expô-lo

## Trade-off
- Isolamento por API, não por hardware: no mesmo processo Go, só a análise estática impede o contorno
- Mais uma superfície no SDK para manter estável

## Requisitos relacionados
FR-012, FR-013, FR-015; desempenho e confiabilidade nos requisitos não funcionais.
