# ADR-003: Login por credencial de serviço e sessão HMAC sem estado

Status: Aceito
Data: 2026-10-19

## Contexto
O PRD previa um token por serviço enviado em toda chamada, com identidade por serviço e não por instância. Era desejável que a credencial longa trafegasse o mínimo possível, que cada instância tivesse identidade própria e que o painel mostrasse a idade da sessão de cada sidecar, sem o controller guardar segredos nem tokens.

## Decisão
- **Credencial por serviço** `{service, secret}`:
  - `n3x.yaml` do sidecar: `secret` como `ENC[...]` (AES-256-GCM, chave em arquivo montado)
  - `n3x.yaml` do controller: só o hash bcrypt (custo ≥ 12); o controller nunca guarda o segredo
- **Login** em `n3x.plugins.n3x.core.v1.Login`, única rota sem sessão, por TLS 1.3, com limite de tentativas por IP e por serviço
- O controller confere o bcrypt e devolve um **token de sessão assinado com HMAC-SHA256**, contendo serviço, instância, emitido em e expira em
- **O controller não guarda tokens**: valida assinatura e validade a cada chamada e registra só o "emitido em" no inventário da instância, para exibir a idade da sessão no painel
- **Chave HMAC:** lida de arquivo montado por volume quando configurada; sem ela, gerada aleatoriamente na subida e mantida só em memória
- No sidecar, o token fica no espaço de memória privado do plugin core (ADR-005) e é renovado por uma goroutine antes de expirar; em `UNAUTHENTICATED`, novo login com jitter e novo registro

## Justificativa
- Identidade por instância, credencial longa trafegando só no login
- Nada sensível persistido no controller
- Agnóstico de orquestrador: igual em Kubernetes, ECS e docker compose
- Chave em arquivo permite sessões sobreviverem ao restart e, no futuro, serem aceitas por mais de uma réplica

## Trade-off
- Sem revogação de uma sessão específica antes de expirar; a defesa é TTL curto (hipótese: 1 h). Para cortar um serviço, troca-se o hash e as sessões morrem na próxima renovação
- Sem chave em arquivo, um restart do controller derruba todas as sessões e gera uma rajada de logins (mitigada por jitter e escala vertical)
- Login vira rota pública, exceção ao FR-011

## Pendências
- TTL e janela de renovação: HLD do `n3x-plugin-core`
- Vários hashes por serviço durante a rotação: resolvido no HLD do `n3x-controller` com um único hash por serviço (ADR-005 do `n3x-controller`)

## Requisitos relacionados
FR-002, FR-005, FR-011, FR-016; segurança nos requisitos não funcionais. Exige atualização do PRD nesses FRs.

## Atualizado em 2026-10-20
O HLD do `n3x-plugin-core` fechou as pendências deste ADR:
- Pendência de TTL resolvida: token de 1 h, renovado em 80% do prazo (cerca de 48 min) com jitter de 10%; o token também leva um nonce (ADR-005 do `n3x-plugin-core`)
- A chave HMAC passou a ser do plugin core, não do host: padrão de 32 bytes aleatórios gerados no `Start`, ou `session.hmac_key: ENC[...]` opcional na seção do core no `n3x.yaml` do controller; `keys.hmac_file` sai do `n3x.yaml` do host (ADR-004 do `n3x-plugin-core`). Onde este ADR diz "arquivo montado", leia essa chave
- Sem a chave configurada, o restart força novo login de todos; com o cache por serviço do host, a meta de 30 s vale até cerca de 200 serviços
