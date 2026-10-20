# ADR-019: Contrato das capacidades privilegiadas do plugin core

Status: Aceito
Data: 2026-10-19

## Contexto
O ADR-004 e os ADR-003 e ADR-004 do `n3x-controller` deram ao plugin core capacidades exclusivas, mas deixaram as interfaces para o HLD do SDK.

## Decisão
O core implementa interfaces extras. O host e o executor só as aceitam de um plugin com ID `n3x.core` vindo do módulo `github.com/giustech/n3x-plugin-core`; em qualquer outro plugin, a subida é recusada.

**Controller**
| Capacidade | Quem implementa | Interface |
| --- | --- | --- |
| Rota pública de login | Core | `PublicGRPC(r)`, que registra só `n3x.plugins.n3x.core.v1.Login`, sem sessão |
| Autenticador de sessão | Core | `Authenticate(ctx, token) (Identity, error)` |
| Diretório de usuários | Core | `CheckUser(ctx, usuário, senha) (Identity, error)` e `ChangePassword(ctx, usuário, atual, nova) error` |
| Gravador de auditoria | Core | `WriteAudit(ctx, AuditEvent) error`, síncrono; se falhar, a ação do painel é recusada |
| Verificador de credencial | Host, entregue só ao core | `env.Credentials().Verify(ctx, serviço, segredo, ip) error` |
| Corte de sessões | Host, entregue só ao core | `env.Sessions().Revoke(usuário)` |

**Sidecar**
| Capacidade | Quem implementa | Interface |
| --- | --- | --- |
| Interceptor da conexão | Core | `ConnInterceptors() (grpc.UnaryClientInterceptor, grpc.StreamClientInterceptor)`, que anexa o token a toda chamada |

## Justificativa
- O mesmo modelo de interfaces opcionais do resto do contrato (ADR-015)
- A checagem por ID e por módulo impede que outro plugin se passe pelo core

## Trade-off
- Interfaces que só um plugin implementa ficam no contrato público do SDK
- O core continua concentrando autenticação, usuários e auditoria

## Requisitos relacionados
FR-002, FR-005, FR-009, FR-010, FR-011, FR-013 e FR-016; ADR-004.

## Atualizado em 2026-10-20
As interfaces seguem no contrato, mas o uso na v1 mudou com o HLD do `n3x-plugin-core`:
- `WriteAudit` fica reservado para a v2: o core da v1 não fornece o gravador de auditoria, e o host não recusa ações por falha de gravação (ADR-002 do `n3x-plugin-core`)
- `ChangePassword` pode devolver `ErrUnsupported`; o core da v1 sempre devolve, porque os usuários vêm só do `n3x.yaml` (ADR-003 do `n3x-plugin-core`)
- `env.Sessions().Revoke` fica sem uso na v1, porque não há gestão de usuários em runtime (ADR-003 do `n3x-plugin-core`)
