# ADR-008: Exemplos de deploy e CI cruzada em `n3x-sdk/deploy/`

Status: Aceito (atualizado em 2026-10-22)
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

## Atualização (2026-10-22)
Alinhamento com a clarificação das FDDs SDK-01 a SDK-16. O corpo acima foi mantido; o que mudou:
- Há um exemplo por topologia (docker compose, Kubernetes, ECS e Istio). O Istio é conteúdo que bloqueia a v1.0.0 e depende da FDD 49, de outro repositório.
- A CI cruzada passa a ter o harness `n3x-e2e` em `deploy/e2e` e a ferramenta `deploytool` em `deploy/tools`, ambos módulos aninhados.
- A conformidade com o proxy real roda no noturno e no pré-release; o adaptador falso roda em todo PR (FDD SDK-16).
