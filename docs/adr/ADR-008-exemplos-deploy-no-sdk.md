# ADR-008: Exemplos de deploy e CI cruzada em `n3x-sdk/deploy/`

Status: Aceito
Data: 2026-10-19

## Contexto
O HLD de plataforma precisa de um exemplo por topologia (docker compose, Kubernetes, ECS e Istio), e a CI usa esses exemplos nos testes de ambiente.

## Decisão
Os exemplos de referência e a orquestração da CI cruzada ficam em `n3x-sdk/deploy/`, junto do HLD de plataforma e do plugin de exemplo.

## Justificativa
- Um lugar só para exemplos, testes de ambiente e documentação de ponta a ponta
- Evita um sexto repositório para uma pessoa manter

## Trade-off
- O `n3x-sdk` passa a carregar artefatos de deploy, além da biblioteca
- Mudanças em sidecar ou controller exigem atualizar exemplos em outro repositório

## Requisitos relacionados
Critérios de aceitação de ambiente (Kubernetes, ECS, docker compose e Istio) e de serviço novo em 30 minutos.
