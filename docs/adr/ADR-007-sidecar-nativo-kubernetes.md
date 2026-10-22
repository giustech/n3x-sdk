# ADR-007: Sidecar nativo no Kubernetes

Status: Aceito (atualizado em 2026-10-22)
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

## Atualização (2026-10-22)
Alinhamento com a clarificação das FDDs SDK-01 a SDK-16. O corpo acima foi mantido; o que mudou:
- A frase "sem `preStop` artificial" foi substituída: passa a existir um `preStop` opcional (H-20), apenas para aplicação que não pode chamar `/drain`.
- O `runAsUser` do pod passou para o contêiner `n3x`; a aplicação mantém o próprio UID.
- A chave AES e o `tls.key` vão por volume preenchido por init, com dono 17000 e modo 0440 (FDD SDK-15).
