# ADR-004: Capacidades privilegiadas do plugin core

Status: Aceito
Data: 2026-10-19

## Contexto
Com o login e a sessão (ADR-003), alguém precisa anexar o token a toda chamada de todos os plugins e validar a sessão no controller antes de qualquer handler. O princípio do N3X é que tudo é plugin, inclusive o registro.

## Decisão
O plugin core (obrigatório e sempre incluído) é o único com estas capacidades, recusadas pelo SDK e pelo controller para qualquer outro plugin:
- Instalar o **interceptor de cliente** na conexão compartilhada do sidecar; ele **sempre** sobrescreve as chamadas ao controller para anexar o token
- Fornecer o **autenticador** usado pelo interceptor do servidor gRPC do controller
- Expor a **única rota pública**: `n3x.plugins.n3x.core.v1.Login`

Enquanto o login não completa, as chamadas dos outros plugins esperam no buffer limitado ou recebem `UNAVAILABLE`.

## Justificativa
- Mantém o modelo "tudo é plugin" sem dar a nenhum outro plugin acesso ao token
- Um único ponto de autenticação, aplicado antes de qualquer plugin

## Trade-off
- O plugin core concentra autenticação, interceptor e registro: um bug nele corta a comunicação de todos os plugins com o controller (o tráfego não é afetado)
- Exceção explícita à regra "nenhuma rota pública de plugin" do FR-011

## Requisitos relacionados
FR-002, FR-005, FR-011, FR-013, FR-016.
