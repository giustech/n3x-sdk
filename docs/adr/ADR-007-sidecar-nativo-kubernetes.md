# ADR-007: Sidecar nativo no Kubernetes

Status: Aceito
Data: 2026-10-19

## Contexto
A meta de zero requisições cortadas em rolling update depende da ordem de subida e descida entre sidecar e aplicação.

## Decisão
No Kubernetes, o sidecar entra como **sidecar nativo**: `initContainers` com `restartPolicy: Always`, exigindo Kubernetes 1.29 ou superior.

## Justificativa
- O sidecar sobe antes da aplicação e desce depois dela, sem `preStop` artificial
- A drenagem de 20 s no SIGTERM fica previsível

## Trade-off
- Clusters anteriores ao 1.29 não são suportados oficialmente
- O modo Istio segue o ciclo de vida definido pelo Istio

## Requisitos relacionados
FR-003; disponibilidade e compatibilidade nos requisitos não funcionais.
