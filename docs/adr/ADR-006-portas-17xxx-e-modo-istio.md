# ADR-006: Faixa de portas 17xxx e Istio no comando

Status: Aceito
Data: 2026-10-19

## Contexto
O PRD citava `15021` como exemplo de health, porta que colide com o Istio (15000, 15001, 15006, 15020, 15021, 15090). Portas comuns (8080, 9090) colidiriam com a aplicação.

## Decisão
- Faixa própria fora do Istio:
  - Sidecar: entrada `17000`, admin `127.0.0.1:17001`, health `17021`, métricas `17090`
  - Controller: gRPC `17443`, painel HTTPS `17444`, métricas e health `17091`
- **No modo Istio, o Istio manda:** a imagem `proxyv2` injetada é substituída pela imagem N3X baseada nela, com o `libn3x.so`; o tráfego entra pelos listeners e portas do Istio e o `EnvoyFilter` insere o filtro do N3X. O N3X não abre `17000` nem `17021` e só acrescenta o que é dele, como as métricas do `.so`

## Justificativa
- Sem colisão com o Istio nem com portas usuais de aplicação
- No Istio, um único dono do ciclo de vida e das portas

## Trade-off
- Portas diferentes entre o modo standalone e o modo Istio, o que exige documentação clara
- Exige atualizar o exemplo `15021` do PRD (FR-003)

## Requisitos relacionados
FR-001, FR-003, FR-005, FR-008; compatibilidade e portabilidade nos requisitos não funcionais.
