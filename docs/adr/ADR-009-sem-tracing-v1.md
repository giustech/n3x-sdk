# ADR-009: Sem tracing distribuído na v1

Status: Aceito
Data: 2026-10-19

## Contexto
A observabilidade do tráfego da aplicação está fora da v1. Restava decidir sobre tracing da operação do próprio N3X (chamadas entre sidecar e controller).

## Decisão
- Sem tracing distribuído na v1
- Correlação por `request_id` nos metadados gRPC, registrado nos logs JSON do sidecar e do controller

## Justificativa
- Custo zero dentro do Envoy e nenhuma dependência nova
- Evita aproximar a v1 da observabilidade de tráfego, que está fora de escopo

## Trade-off
- Depuração entre sidecar e controller depende de cruzar logs manualmente
- OpenTelemetry fica para uma versão futura

## Requisitos relacionados
Observabilidade nos requisitos não funcionais.
