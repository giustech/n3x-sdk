# ADR-001: Controller em instância única, sempre um por ambiente

Status: Aceito
Data: 2026-10-19

## Contexto
O controller só observa os sidecars e fica fora do caminho crítico do tráfego. A meta é 99,5% de disponibilidade mensal e 500 sidecars em 1 vCPU e 512 MB. Era preciso definir quantas réplicas existem e como ambientes (staging, produção) convivem.

## Decisão
- A v1 roda o controller em **instância única**
- Existe **sempre um controller por ambiente**; o nome do serviço é a chave única dentro do ambiente e não há campo de ambiente no modelo

## Justificativa
- Com o controller fora do caminho crítico, a meta de 99,5% (cerca de 3,6 h por mês) se cumpre com o restart do orquestrador
- Sem eleição de líder, sem estado distribuído e sem colisão de nomes entre ambientes

## Trade-off
- Escala só vertical
- Réplicas no futuro exigem chave HMAC compartilhada (já possível via arquivo montado, ADR-003), sessões do painel fora da memória (ADR-006 do `n3x-controller`) e bancos de plugin compartilhados entre réplicas (ADR-002 do `n3x-controller`)

## Requisitos relacionados
FR-005, FR-006; disponibilidade e desempenho do controller nos requisitos não funcionais.

## Atualizado em 2026-10-20
A chave HMAC compartilhada entre réplicas passou a ser a `session.hmac_key` da seção do plugin core, não mais um arquivo montado no host (ADR-004 do `n3x-plugin-core`). O core da v1 guarda tudo em memória, então réplicas também exigem o banco da v2 (ADR-001 do `n3x-plugin-core`).
