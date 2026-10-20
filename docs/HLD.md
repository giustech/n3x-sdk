### HLD: n3x-sdk (contratos de plugin, kit de testes e plugin de exemplo)

Versão: 1.1
Data: 2026-10-20
Responsável: Guilherme Giusti

---

### Objetivo técnico
Definir os contratos Go estáveis que todo plugin do N3X implementa (metade de proxy e metade de controller), sem cgo e sem depender do Envoy nem do host, além do kit de testes, do plugin de exemplo, do gerador de plugins e da documentação para autores. A implementação do executor da cadeia (sidecar) e do host de plugins (controller) fica fora: este HLD define só o que eles oferecem ao plugin e o que exigem dele.

Este HLD é dono destes contratos (conforme `docs/hld/HLDs-N3X.md`):
- Interfaces de plugin de proxy e de controller, incluindo as capacidades privilegiadas do plugin core
- As oito fases fixas, as políticas de falha e o padrão de IDs com namespace
- Manifesto do plugin (`n3x-plugin.yaml`) e estrutura padrão do repositório de plugin
- Widgets do painel, memória isolada, schema da config e tipo `plugin.Secret`
- Formato `ENC[...]` como pacote (`secret/`), mantendo o formato definido no HLD do `n3x-sidecar`
- Regras da análise estática (implementadas pelo `n3x-builder`)
- Política de versionamento e depreciação do SDK

Problemas técnicos endereçados
- Os plugins não podem quebrar quando o SDK Go do Envoy muda, porque ele não tem versões com tag
- O plugin precisa ser testável sem Envoy e sem controller
- O isolamento entre plugins (memória, token, namespace gRPC e rotas) precisa estar no contrato, e não depender da boa vontade do autor
- O caminho quente do sidecar exige 0 alocações e até 5 cruzamentos Envoy→Go por requisição, independentemente do número de plugins
- O painel precisa manter visual, acessibilidade e segurança iguais em todas as telas de plugin, sem nenhum autor escrevendo HTML
- Os documentos anteriores usavam três formatos de ID e três arquivos chamados `n3x.yaml`

Base: `docs/prd/PRD-N3X.md` versão 1.5, de 2026-10-20 (atualizada com as decisões deste HLD e do HLD do `n3x-plugin-core`). Requisitos cobertos: FR-012, FR-013 e FR-014, mais os pontos de contrato de FR-004, FR-011, FR-015 e FR-016 e os requisitos não funcionais de desempenho, segurança e observabilidade no lado do plugin.

Dependências com outros sistemas
- `n3x-sidecar`: implementa o contrato de proxy (executor, adaptador por alvo e conexão compartilhada) e usa `plugin/` e `secret/`
- `n3x-controller`: implementa o contrato de controller (host, registradores e renderizador no painel) e usa `controller/ui` e `secret/`
- `n3x-builder`: confere a faixa `sdk` do manifesto, gera o schema e o registro dos plugins e aplica a análise estática definida aqui
- `n3x-plugin-core`: primeiro cliente do contrato e único dono das capacidades privilegiadas
- Plugins futuros (ex.: `giustech.ratelimit`), em repositórios próprios
- grpc-go e protobuf; buf na CI dos plugins

---

### Arquitetura geral
O SDK é uma **biblioteca Go sem runtime próprio**, publicada por tag semver no GitHub como um único módulo (ADR-011). Ele roda dentro de dois processos que não são dele: o `libn3x.so` do sidecar (metade de proxy) e o binário do controller (metade de controller).

```
n3x-sdk/                          github.com/giustech/n3x-sdk
├── plugin/        ID, fases, políticas, manifesto, schema por tags, plugin.Secret
├── proxy/         contrato de proxy: Define, hooks, Stream, Deny, Go, Env
├── controller/    contrato de controller: Define, capacidades, Caller, State
│   └── ui/        widgets e renderizador (HTML, htmx, SVG)
├── memory/        mapas isolados por plugin, com cota
├── secret/        ENC[...] (AES-256-GCM); proibido aos plugins
├── sdktest/       kit de testes sem Envoy e sem controller
├── example/       plugin n3x.example, com as duas metades
├── cmd/n3x-new/   gerador de plugin e schema-check
├── deploy/        exemplos de deploy e CI cruzada (ADR-008)
└── docs/          HLD de plataforma, este HLD, ADRs e guia do autor
```

**Regras de dependência** (ADR-011), conferidas na CI do SDK e no `n3x-plugin build`
- Sem cgo
- Nenhum pacote importa o SDK Go do Envoy nem pacotes internos do `n3x-sidecar` ou do `n3x-controller`
- `proxy/` e `memory/` usam só a biblioteca padrão e grpc-go
- `proxy/` nunca importa `controller/`
- O Echo não aparece no contrato: só `net/http`, `context` e widgets
- Código de plugin não importa `secret/` (ADR-014)

**Padrão de IDs** (ADR-012)
| Onde | Regra | Exemplo |
| --- | --- | --- |
| ID | `<namespace>.<nome>`, minúsculas, letras, números e `_`, cada parte com 2 a 32 caracteres | `giustech.ratelimit` |
| `n3x.yaml` do serviço e do controller | O ID inteiro | `giustech.ratelimit` |
| Serviço gRPC | `n3x.plugins.<namespace>.<nome>.v<major>` | `n3x.plugins.giustech.ratelimit.v1` |
| Rotas do painel | `/plugins/<namespace>/<nome>/` | `/plugins/giustech/ratelimit/` |
| Métricas do plugin | `n3x_<namespace>_<nome>_<métrica>` | `n3x_giustech_ratelimit_blocked_total` |
| Rótulo `plugin` | O ID inteiro | `plugin="giustech.ratelimit"` |

- Namespace `n3x` reservado aos módulos `github.com/giustech/` (`n3x.core` e `n3x.example`); o login do core é `n3x.plugins.n3x.core.v1.Login`
- ID duplicado faz o build falhar

**Fases e políticas** (contrato; o comportamento do executor está no HLD do `n3x-sidecar`)
- Oito fases fixas, nesta ordem: `access`, `authn`, `route`, `authz`, `traffic`, `transform`, `upstream` e `observe`. Na resposta, a ordem é a inversa
- Só as fases até `route` recebem a API de alteração de rota (`s.SetRoute`); as seguintes não a têm
- Políticas de falha: `open` (segue) e `closed` (503 ou `UNAVAILABLE`), com o padrão declarado no manifesto e sobrescrito por instância no `n3x.yaml`
- Mesma fase e mesmo `order` formam um grupo paralelo (ADR-005 do `n3x-sidecar`)

**Estrutura padrão de um repositório de plugin** (FR-014)
```
n3x-plugin-ratelimit/                 github.com/giustech/n3x-plugin-ratelimit
├── n3x-plugin.yaml   manifesto (ADR-013)
├── agent/            metade de proxy: Plugin(), New e hooks
├── controller/       metade de controller: Plugin(), New e capacidades
├── types/            structs de config (fonte do schema) e tipos compartilhados
├── proto/            .proto em n3x.plugins.<namespace>.<nome>.v1 e stubs gerados
├── connectors/       clients do plugin (ex.: réplica Redis), quando houver
└── Makefile          test, generate (buf) e lint
```
- Plugin só de proxy ou só de controller omite a pasta da outra metade; plugin que não fala com o controller dispensa `proto/`

**Manifesto `n3x-plugin.yaml`** (ADR-013)
```yaml
id: giustech.ratelimit
version: 1.0.0
sdk: ">=1.0.0 <2.0.0"          # faixa de SDK aceita; o build confere
halves: [proxy, controller]
proxy:
  phases: [traffic]             # fases permitidas no n3x.yaml do serviço
  default_phase: traffic
  default_order: 50
  default_failure: open         # open | closed
controller:
  required: false
  depends_on: []                # IDs; o build falha se faltar algum
```

Ambiente de implantação
- Não se implanta sozinho: é compilado dentro do `libn3x.so` (por alvo e arquitetura) e do binário do controller pelo `n3x-plugin build`
- Publicação: tag semver no GitHub, via Go module proxy; pré-releases `v0.x` até a `v1.0.0` (ADR-021)
- Exemplos de deploy e CI cruzada em `deploy/` (ADR-008 de plataforma)

Tecnologias principais
- Go 1.26 ou superior, sem cgo, com generics
- grpc-go e protobuf (interface da conexão compartilhada e registrador gRPC)
- templ e htmx só dentro do renderizador de widgets, nunca no contrato
- SVG gerado no servidor para `Chart`
- buf (`generate` e `breaking`) na CI dos plugins
- `log/slog` para logs

Padrões adotados
- Plugin de duas metades, compilado estaticamente e registrado por função gerada (sem `init()`)
- Hooks e capacidades como interfaces opcionais, detectadas por type assertion (ADR-015)
- Config tipada por generics, com schema gerado da struct (ADR-020)
- Registro no estilo gRPC e `net/http`: o host chama o método de registro, e o roteamento é por nome
- UI declarativa por árvore de widgets, no estilo do Flutter (ADR-018)
- Isolamento por handle preso ao ID (memória, métricas, logs e estado)
- Capacidades privilegiadas por interface, aceitas só do `n3x.core` (ADR-019)
- Prazo obrigatório em todo I/O e controller só em segundo plano (ADR-016)

---

### Componentes e responsabilidades
| Componente | Responsabilidades | Dependências |
| ----------- | ----------------- | ------------ |
| `plugin/` | Validação do ID; fases e políticas; leitura do `n3x-plugin.yaml`; geração do JSON Schema a partir das tags; decodificação estrita da config (campo desconhecido é erro, `default` aplicado); tipo `plugin.Secret` | Biblioteca padrão |
| `proxy/` | `Define[C](New)`; hooks `OnRequestHeaders`, `OnRequestBody`, `OnResponseHeaders` e `OnComplete`; `Stream` (path, método, headers, método gRPC, `SetRoute` até a fase `route`, `Deny`, `Go()`); respostas prontas de negação ciente de gRPC; `Env` com `Controller()`, `Background()`, `Dial()`, clientes HTTP e TCP com prazo, `Sleep()`, `Memory()`, `Metrics()`, `Log()` e `Target()` | `plugin/`, `memory/`, grpc-go |
| `controller/` | `Define[C](New)`; capacidades `Start`, `Stop`, `GRPC`, `Routes`, `Cards` e `Validate`; `Caller(ctx)`; `State()` com `ok` e `degradado`; interfaces privilegiadas do core | `plugin/`, `memory/`, `controller/ui`, grpc-go |
| `controller/ui` | Catálogo de widgets; renderizador que gera página inteira ou fragmento htmx, SVG dos gráficos, paginação, filtros, exportação, CSRF de `Form` e `Action` e acessibilidade WCAG 2.1 AA | templ (interno) |
| `memory/` | `NewMap[K, V]` preso ao ID; cota em bytes estimados; descarte do mais antigo; 64 partes com trava por parte | Biblioteca padrão |
| `secret/` | `ENC[v1:...]` com AES-256-GCM e AAD por serviço; `Encrypt`, `Decrypt` e `KeyGen`; zera buffers por melhor esforço | Biblioteca padrão |
| `sdktest/` | Requisição, `Stream`, config, métricas e logs falsos; agendador determinístico; host falso de controller; `bufconn`; `AssertHonorsCancel`, `AssertZeroAllocs` e `AssertCounter`; alvos simulados | `proxy/`, `controller/` |
| `example/` | Plugin `n3x.example`: adiciona header e nega um path em HTTP e gRPC, conta as negações e as envia ao controller, que mostra um cartão | Todos os pacotes de contrato |
| `cmd/n3x-new` | `n3x-new <id>` cria a estrutura a partir do `example/`; `schema-check --against <tag>` confere a evolução da config | `plugin/` |
| Executor do sidecar (outro repositório) | Implementa `proxy.Env` e `proxy.Stream` sobre o adaptador de cada alvo | `proxy/` |
| Host do controller (outro repositório) | Implementa `ctrl.Env`, os registradores e as checagens de namespace, rotas e capacidades privilegiadas | `controller/`, `controller/ui` |

Quem persiste dados: ninguém no SDK; o banco é sempre do plugin. Quem faz cache: só o `memory/`, volátil e por plugin. Quem orquestra: o executor do sidecar (requisição) e o host do controller (ciclo de vida); o SDK só define o contrato.

**Contrato de proxy** (FR-012, ADR-015 e ADR-016)
```go
package agent

type Config = types.ProxyConfig            // struct em types/, fonte do schema

func Plugin() proxy.Plugin { return proxy.Define[Config](New) }   // chamado pelo registro gerado

func New(cfg Config, env proxy.Env) (proxy.Instance, error) {
    i := &Example{cfg: cfg, denies: env.Metrics().Counter("denies_total", "Negações", "path")}
    i.count = memory.NewMap[string, int64](env.Memory(), "denies")
    if conn, err := env.Controller(); err == nil {                 // ErrStandalone sem controller
        cli := examplev1.NewExampleClient(conn)
        env.Background("report", func(ctx context.Context) {       // controller só em segundo plano
            for ctx.Err() == nil {
                cli.Report(ctx, &examplev1.ReportReq{Denies: i.total()})
                proxy.Sleep(ctx, 10*time.Second)
            }
        })
    }
    return i, nil
}

func (i *Example) OnRequestHeaders(s proxy.Stream) proxy.Action {
    if strings.HasPrefix(s.Path(), i.cfg.DenyPath) {
        i.denies.With(i.cfg.DenyPath).Inc()
        return s.Deny(proxy.Forbidden("bloqueado"))   // HTTP 403 ou gRPC PERMISSION_DENIED em trailers-only
    }
    s.Headers().Set(i.cfg.Header, "1")
    return proxy.Continue
}
```
- Ações: `proxy.Continue`, `s.Deny(...)` e `s.Go(func(ctx) proxy.Result)`; `proxy.Fail(err)` aplica a política de falha
- Negação sempre ciente de gRPC; resposta local crua não existe na API
- `Target().Has(proxy.CapX)` informa a capacidade ausente no alvo, e o plugin desliga a funcionalidade de forma explícita
- Conexões: `env.Controller()` só em `Background()` (dentro de `s.Go()` devolve `proxy.ErrRequestPath`); `env.Dial()` permitido na requisição, com o prazo do `Go()`
- Limites por plugin: 8 tarefas de fundo e 16 conexões em `Dial()` (ajustáveis na config)
- Body de resposta fora da v1; entra como `OnResponseBody` por adição

**Contrato de controller** (FR-013, ADR-015, ADR-018 e ADR-019)
```go
package controller

func Plugin() ctrl.Plugin { return ctrl.Define[Config](New) }

func New(cfg Config, env ctrl.Env) (ctrl.Instance, error) { return &Example{env: env}, nil }

func (p *Example) GRPC(r ctrl.GRPCRegistrar) { examplev1.RegisterExampleServer(r, p) }  // só n3x.plugins.n3x.example.v1

func (p *Example) Report(ctx context.Context, r *examplev1.ReportReq) (*examplev1.Ok, error) {
    who := ctrl.Caller(ctx)                                   // {sidecar, pedidos, pedidos-7d9f-xk2lp}
    p.soma(who.Service, r.Denies)
    return &examplev1.Ok{}, nil
}

func (p *Example) Cards(c ctrl.Cards) { c.Add("negacoes", ui.Stat{Label: "Negações", Value: p.total()}) }

func (p *Example) Routes(r ctrl.Routes) {
    r.Page("/negacoes", ctrl.RoleRead, func(ctx context.Context, q ui.Query) (ui.Widget, error) {
        return ui.Page{Title: "Negações", Body: ui.Column{Children: []ui.Widget{
            ui.Row{Children: []ui.Widget{ui.Stat{Label: "Total", Value: p.total()}}},
            ui.Section{Title: "Por serviço", Child: ui.Refresh{Every: 5 * time.Second, Child:
                ui.Table{
                    Columns: []ui.Column{{Key: "service", Label: "Serviço"}, {Key: "denies", Label: "Negações"}},
                    Rows:    p.porServico(q),
                    Export:  []ui.Format{ui.CSV, ui.JSON},
                },
            }},
        }}}, nil
    })
}
```
- Uma instância de controller por plugin, criada uma vez; o host chama as capacidades de registro e, depois, o grpc-go e o roteador HTTP chamam os handlers por nome
- `env` traz config tipada, logger, métricas, memória isolada e `State()`
- Estado: o plugin declara `ok` ou `degradado` com motivo (banner no painel); `falho` é decidido só pelo host
- O banco é do plugin, configurado na seção dele no `n3x.yaml` do controller (ADR-002 do `n3x-controller`)

**Catálogo de widgets da v1** (ADR-018)
| Grupo | Widgets |
| --- | --- |
| Layout | `Page`, `Column`, `Row`, `Grid`, `Section` e `Tabs` |
| Conteúdo | `Text`, `Stat`, `Badge`, `Table`, `Chart` (linha e barra, em SVG), `Detail`, `Code` (YAML e diff) e `Empty` |
| Interação | `Refresh` (polling htmx), `Link`, `Form` e `Action` (CSRF automático; sem papel declarado, exige root) |

**Capacidades privilegiadas do core** (ADR-019), aceitas só de `n3x.core` vindo de `github.com/giustech/n3x-plugin-core`
| Capacidade | Lado | Quem implementa | Interface |
| --- | --- | --- | --- |
| Rota pública de login | Controller | Core | `PublicGRPC(r)`, só `n3x.plugins.n3x.core.v1.Login` |
| Autenticador de sessão | Controller | Core | `Authenticate(ctx, token) (Identity, error)` |
| Diretório de usuários | Controller | Core | `CheckUser(ctx, usuário, senha) (Identity, error)` e `ChangePassword(...)`, que pode devolver `ErrUnsupported` (o core da v1 devolve, ADR-003 do `n3x-plugin-core`) |
| Gravador de auditoria | Controller | Core | `WriteAudit(ctx, AuditEvent) error`, síncrono; reservado para a v2, o core da v1 não fornece (ADR-002 do `n3x-plugin-core`) |
| Verificador de credencial | Controller | Host, entregue só ao core | `env.Credentials().Verify(ctx, serviço, segredo, ip) error` |
| Corte de sessões | Controller | Host, entregue só ao core | `env.Sessions().Revoke(usuário)`; sem uso na v1, que não tem gestão de usuários em runtime |
| Interceptor da conexão | Sidecar | Core | `ConnInterceptors() (unary, stream)` |

**Kit de testes** (FR-014)
```go
func TestNegaPath(t *testing.T) {
    p := sdktest.NewProxy(t, agent.Plugin(), `{"header":"X-N3X","deny_path":"/admin"}`)
    p.Request(sdktest.GET("/admin")).AssertDenied(t, 403)
    p.Request(sdktest.GRPC("/pedidos.v1.Pedidos/Criar")).AssertHeader(t, "X-N3X", "1")
}

func TestRespeitaCancelamento(t *testing.T) { sdktest.AssertHonorsCancel(t, agent.Plugin(), cfg) }
```
| Peça | Para quê |
| --- | --- |
| `NewProxy`, `GET`, `POST` e `GRPC` | Requisição, config e `Stream` falsos, com as mesmas regras do executor (fases, grupos e negação gRPC) |
| Agendador determinístico | Roda `Go()` e `Background()` passo a passo, sem `time.Sleep` |
| Métricas e logs falsos | `AssertCounter` e checagem de que nenhum `plugin.Secret` aparece em logs |
| `NewHost` | Host falso de controller: `Start`, `GRPC`, `Routes`, `Cards`, papéis e renderização de widgets |
| `bufconn` | Liga a metade de proxy à de controller no mesmo teste |
| `AssertHonorsCancel` e `AssertZeroAllocs` | Portões obrigatórios: prazo respeitado e 0 alocações |
| Alvos simulados | `envoy-1.40` e `istio-1.31`, para testar capacidades ausentes |

Tudo roda com `go test -race`, sem Envoy e sem controller.

---

### Fluxo de requisições e de dados
**Fluxo de requisição**
- **Ciclo do autor:**
  - `go run github.com/giustech/n3x-sdk/cmd/n3x-new@v1 giustech.ratelimit --halves proxy,controller`
  - implementação das metades
  - `go test -race ./...` com o `sdktest`
  - `buf generate` e `buf breaking`
  - `n3x-new schema-check --against <última tag>`
  - tag semver
- **Inclusão:** `RUN n3x-plugin add github.com/giustech/n3x-plugin-ratelimit@v1.0.0` nos Dockerfiles do sidecar e do controller (FR-015); seção do plugin no `n3x.yaml` do serviço e no do controller; redeploy
- **Build** (execução no `n3x-builder`, regras do SDK):
  - lê o `n3x-plugin.yaml` e confere `id`, `halves`, pastas e a faixa `sdk`
  - gera o JSON Schema das structs de `types/`
  - gera o registro chamando `agent.Plugin()` e `controller.Plugin()`
  - aplica a análise estática (ver Segurança)
- **Subida no sidecar:**
  - para cada `instance` do `n3x.yaml`, decodifica a `config` na struct `C`
  - valida pelo schema e chama `New(cfg, env)`
  - detecta os hooks por interface e entrega a instância ao executor
  - qualquer erro faz o Envoy não subir (FR-004)
- **Requisição:**
  - o caminho é Envoy → adaptador do alvo → executor → `Stream` reaproveitado de um pool (0 alocações) → hook
  - o hook devolve `Continue`, `Deny` ou `Go()`
  - dentro do `Go()`, `env.Controller()` devolve `ErrRequestPath` e `env.Dial()` funciona com o prazo do `Go()`
  - no fim, `OnComplete` é chamado em todas as instâncias que rodaram
- **Segundo plano:**
  - `env.Background()` abre uma goroutine do core com `recover` e rótulos de pprof
  - um panic gera log e `panics_total`, e a tarefa reinicia com backoff
  - no desligamento, o `ctx` é cancelado e há 2 s para terminar
- **Subida no controller:**
  - decodifica a seção do plugin, roda `Validate` e chama `New`
  - roda `Start` com prazo e depois `GRPC`, `Routes` e `Cards`
  - o host confere o namespace, as rotas e as capacidades privilegiadas
- **Página no painel:**
  - o host aplica sessão, CSRF e papel e chama a função da página com `ctx` e prazo (`runtime.plugin_call_timeout`)
  - a função devolve a árvore de widgets
  - o renderizador do SDK gera a página inteira, ou o fragmento quando há `HX-Request`

**Fluxo de dados**
- Struct Go em `types/` → JSON Schema → manifesto embutido no `.so` e no binário → validação do `n3x.yaml`, mascaramento por `plugin.Secret` e rótulos do painel
- `n3x.yaml` do serviço (`config` com `ENC[...]`) → `secret/` no `.so` → struct tipada com `plugin.Secret` → `New`
- Proxy → `Background()` → conexão compartilhada (interceptor do core anexa o token) → serviço `n3x.plugins.<namespace>.<nome>.v1` → banco do plugin
- Banco do plugin → réplica de leitura (replicação do próprio banco) → `env.Dial()` no sidecar, durante a requisição (ADR-016)
- `env.Metrics()` → stats do Envoy (`17090`) ou Prometheus do controller (`17091`); `env.Log()` → JSON em stderr

---

### Modelo de dados (alto nível)
Entidades principais
- **Manifesto do plugin** (`n3x-plugin.yaml`): `id`, `version`, `sdk`, `halves`, fases permitidas, `default_phase`, `default_order`, `default_failure`, `required` e `depends_on`
- **Manifesto embutido:** manifestos dos plugins + JSON Schema de cada config + alvo + hash (formato do HLD do `n3x-builder`)
- **Config tipada:** struct Go com tags `json`, `n3x` (`required`, `min`, `max` e `enum`), `desc` e `default`, e campos sensíveis em `plugin.Secret`
- **Instância de proxy:** objeto devolvido pelo `New`, um por `instance` do `n3x.yaml` do serviço
- **Instância de controller:** objeto devolvido pelo `New`, um por plugin
- **Identidade (`Caller`):** papel `root`, `leitura` ou `sidecar`, mais usuário ou serviço e instância
- **Evento de auditoria** (reservado para a v2): ator, ação, alvo, IP e horário; montado pelo host e gravado pelo core
- **Estado do plugin:** `ok` e `degradado` (declarados pelo plugin) e `falho` (só pelo host), com motivo e horário
- **Widget:** árvore de `ui.Widget`, gerada a cada requisição e nunca guardada
- **Mapa de memória:** `memory.Map[K, V]`, preso ao ID e com cota por plugin

Relações
- Plugin 1:1 manifesto; plugin 1:N instâncias de proxy; plugin 0:1 instância de controller
- Plugin 0:N mapas de memória, todos dentro da mesma cota
- Plugin 0:1 serviço gRPC por major; plugin 0:N páginas e cartões
- Página 1:1 árvore de widgets por requisição

Fonte de verdade
- Repositório do plugin: `n3x-plugin.yaml`, structs de `types/` e `.proto`
- SDK: fases, políticas, papéis, padrão de IDs e catálogo de widgets
- `n3x.yaml` do serviço e `n3x.yaml` do controller: os valores da config de cada plugin
- O SDK não persiste nada; a memória é volátil e o banco é sempre do plugin

Versionamento e retenção
- Config do plugin: só por adição dentro da major do plugin, conferida na CI com `n3x-new schema-check` (ADR-020)
- Protos: `n3x.plugins.<namespace>.<nome>.v<major>`, só por adição, com `buf breaking`
- SDK: semver estrito, estável desde a `v1.0.0` (ADR-021)
- Nada é retido pelo SDK

---

### Interfaces públicas
| Nome | Tipo | Protocolo | Exposição | SLAs/Limites |
| ---- | ---- | ---------- | --------- | ------------- |
| `proxy/` | SDK | API Go | Externa (autores de plugin) | 0 alocações nos caminhos comuns; hooks síncronos do core e do exemplo em p50 ≤ 0,05 ms (orçamento do sidecar); 8 tarefas de fundo e 16 conexões por plugin |
| `controller/` | SDK | API Go | Externa (autores de plugin) | Página com o prazo do host (10 s); cartão com prazo de 1 s |
| `controller/ui` | SDK | API Go (widgets) | Externa (autores de plugin) | Tabela de 500 linhas renderizada em até 5 ms; paginação de 50 linhas por padrão; WCAG 2.1 AA |
| `memory/` | SDK | API Go | Externa | 16 MiB no sidecar e 64 MiB no controller por plugin; seguro entre os workers do Envoy |
| `plugin/` | SDK | API Go + tags de struct | Externa | Schema gerado; config só evolui por adição |
| `secret/` | SDK | API Go | Interna (sidecar, controller e builder) | Proibido aos plugins |
| `sdktest/` | SDK | API Go | Externa | `go test -race`, sem Envoy e sem controller |
| `n3x-plugin.yaml` | SDK | YAML | Externa | Validado pelo `n3x-plugin add` |
| Convenção gRPC dos plugins | API | Protobuf | Externa (autores) → interna (runtime) | `n3x.plugins.<namespace>.<nome>.v<major>`, com `buf breaking` |
| `n3x-new` (`new` e `schema-check`) | SDK | CLI | Externa | Criação e CI do plugin |

---

### Considerações de escalabilidade e disponibilidade
Abordagem geral
- O SDK não escala sozinho: ele escala junto com o sidecar (1:1 com a aplicação) e com o controller (instância única). O papel dele é não gastar o orçamento desses dois
- O controller nunca entra no caminho da requisição: a conexão com ele só funciona em segundo plano (ADR-016)
- Todo I/O do plugin tem prazo e limite, e um backend lento vira "política de falha aplicada", e não sidecar travado

Técnicas aplicadas
- **Caminho quente sem alocação:**
  - `Stream` reaproveitado por pool
  - acessores que não copiam (`s.Path()`, `Headers().Get`)
  - respostas de negação pré-montadas (`proxy.Forbidden`, `proxy.Unavailable`) e sem `fmt` no caminho quente
- **Métricas por handle**, criadas no `New`, sem busca por nome por requisição
- **`memory.Map`:**
  - dividido em 64 partes com trava por parte
  - `Get` sem alocação para tipos fixos
  - descarte aproximado (CLOCK) quando a cota enche
- **Limites por plugin:**
  - 8 tarefas de fundo (a 9ª falha no `New`)
  - 16 conexões em `Dial()` (a 17ª espera vaga até o prazo do `Go()`)
- **Renderizador de widgets** com buffer reaproveitado e paginação no servidor, para o painel ficar em p95 < 300 ms com 500 sidecars
- **Reinício com backoff** das tarefas de fundo após panic
- **Portões de CI:** `AssertZeroAllocs` nos acessores, no `Deny` e no `Map.Get` do exemplo, e benchmark do renderizador

Meta de disponibilidade
- O SDK não tem meta própria de uptime; ele herda as metas do sidecar e do controller
- Zero requisições da aplicação com erro causado pelo controller, garantido pelo contrato (ADR-016)
- Toda chamada de plugin termina: prazo em `Go()`, `Dial()`, clientes do SDK, páginas e cartões

---

### Segurança
Autenticação
- O SDK não autentica nada: entrega `ctrl.Caller(ctx)` já preenchido pelo host, que conferiu o token do sidecar ou a sessão do painel
- O token de sessão do sidecar fica só na memória do core e é anexado pelo interceptor do core (ADR-019); nenhum outro plugin o enxerga

Autorização
- Páginas, cartões, `Form` e `Action` declaram o papel mínimo (`RoleRead` ou `RoleRoot`); `Form` e `Action` sem papel declarado exigem root
- Capacidades privilegiadas aceitas só de `n3x.core` vindo de `github.com/giustech/n3x-plugin-core` (ADR-019)
- Namespace `n3x` reservado aos módulos `github.com/giustech/` (ADR-012)
- Registro gRPC fora de `n3x.plugins.<namespace>.<nome>.v<major>` ou rota fora de `/plugins/<namespace>/<nome>/`: recusado na subida
- Alteração de rota só nas fases até `route`
- Estado `falho` só pelo host; o plugin não consegue se desligar sozinho

Proteção de dados
- `plugin.Secret` mostra `***` em log, `fmt`, JSON e erros; o valor real só sai com `.Reveal()` (ADR-020)
- O mascaramento da config enviada ao controller usa o mesmo tipo
- O renderizador escapa todo texto, e nenhum plugin gera HTML (sem XSS vindo de plugin); `Form` e `Action` com CSRF automático; CSP do painel preservada (`Chart` em SVG, sem JavaScript)
- Memória isolada por handle, sem acessor global (ADR-005 e ADR-017)
- O SDK não trata PII; o que for PII é responsabilidade do plugin e do banco dele

**Análise estática** (regras definidas aqui, implementadas pelo `n3x-builder`)
| Regra | Motivo |
| --- | --- |
| `go` cru, `os.Exit`, `log.Fatal` e `init()` com efeito colateral | Goroutine sem `recover` ou processo encerrado derruba o Envoy |
| `unsafe` e `reflect` sobre tipos do SDK | Furam o isolamento de memória |
| cgo e import do SDK Go do Envoy | Contrato estável e sem crash em C |
| Import de `secret/` e de pacotes internos do sidecar ou do controller | Chave e internals fora do alcance do plugin |
| `context.Background()` e `context.TODO()` | Toda operação precisa de prazo (ADR-006 do `n3x-sidecar`) |
| `time.Sleep`, `net.Dial`, `http.Get` e chamadas bloqueantes sem `ctx` | Use `proxy.Sleep`, `env.Dial` e os clientes do SDK |
| `.Reveal()` dentro de chamada de log | Vazamento de segredo |
| API marcada `// Deprecated:` | Só aviso, sem falhar (ADR-021) |

Gestão de segredos
- Segredos da config em `ENC[v1:...]` no `n3x.yaml`, decifrados pelo sidecar ou pelo controller com `secret/` antes do `New`
- A chave AES nunca chega ao plugin
- `secret/` usa só a biblioteca padrão e é o único código do formato `ENC[...]` (ADR-014)

---

### Observabilidade
Logs
- `env.Log()` devolve um `*slog.Logger` com saída JSON e campos fixos: `ts`, `level`, `component=plugin`, `service`, `instance`, `plugin_id` e `request_id` quando houver requisição
- `plugin.Secret` sempre mascarado
- Stack trace de panic amostrado em 1 por segundo por instância, feito pelo core do sidecar ou pelo host

Métricas
- `env.Metrics()` com counter, gauge e histogram, por handle, prefixo `n3x_<namespace>_<nome>_` e rótulo `instance` automático (ADR-022)
- Sem limite de cardinalidade; a documentação mostra exemplos certos e errados
- Publicadas pelo SDK, sem esforço do autor:
  - `n3x_plugin_duration_seconds` por hook
  - `n3x_plugin_memory_bytes`
  - `n3x_plugin_memory_evictions_total`
- Os contadores de falha, panic, disjuntor e zumbis continuam no executor (HLD do `n3x-sidecar`)

Tracing
- Sem tracing distribuído na v1 (ADR-009 de plataforma)
- `request_id` propagado nos metadados gRPC pela conexão compartilhada e disponível em `s.RequestID()`

Dashboards e alertas
- No painel: banner por estado (`degradado` ou `falho`) e cartões dos plugins em widgets
- Alertas sugeridos na documentação:
  - `rate(n3x_plugin_memory_evictions_total[5m]) > 0`, ou seja, cota apertada
  - número de séries por plugin acima do esperado, ou seja, cardinalidade
  - `n3x_plugin_duration_seconds` p99 por hook acima do orçamento
- No kit de testes: `AssertCounter`, captura de logs e checagem de segredos

---

### Riscos arquiteturais e mitigação
#### Contrato congelado na v1.0 com um erro de design
- **Probabilidade:** média
- **Impacto:** major nova cedo, com todos os plugins recompilados
- **Mitigação:**
  - Pré-releases `v0.x` validadas com o core, o exemplo e um protótipo de rate limit antes da `v1.0.0`
  - Revisão do contrato inteiro antes da tag
- **Plano de contingência:** `v2` com camada de compatibilidade para a `v1` durante uma major

#### Goroutine de biblioteca de connector dá panic fora do `recover`
- **Probabilidade:** média
- **Impacto:** derruba o Envoy e o serviço perde a entrada até o restart
- **Mitigação:**
  - Lista de bibliotecas aprovadas para connectors
  - Teste de longa duração e de injeção de falha com o connector real
  - Documentação do risco no guia do autor
- **Plano de contingência:** tirar o plugin da imagem e redeployar

#### Rótulo de métrica com cardinalidade alta
- **Probabilidade:** média
- **Impacto:** memória do sidecar estoura até o OOM
- **Mitigação:**
  - Documentação com exemplos certos e errados
  - Alerta sugerido sobre o número de séries por plugin
- **Plano de contingência:** tirar a instância do `n3x.yaml` e redeployar

#### Catálogo de widgets não cobre uma tela, sem escape para HTML
- **Probabilidade:** média
- **Impacto:** plugin espera uma minor do SDK
- **Mitigação:**
  - Telas do core (sidecars, diff de config e lista de usuários; auditoria só na v2) feitas só com widgets antes da `v1.0.0`
- **Plano de contingência:** minor rápida com o widget novo

#### 0 alocações difícil de manter com uma API ergonômica
- **Probabilidade:** média
- **Impacto:** latência do sidecar acima do orçamento
- **Mitigação:**
  - `AssertZeroAllocs` como portão de CI
  - Handles pré-criados e respostas pré-montadas
- **Plano de contingência:** variantes de baixo nível só para o caminho quente

#### Análise estática deixa passar um contorno do isolamento
- **Probabilidade:** baixa
- **Impacto:** plugin lê memória ou token de outro
- **Mitigação:**
  - Corpus de plugins "maliciosos" nos testes do builder
  - Isolamento por handle, sem acessor global
- **Plano de contingência:** corrigir a regra e remover o plugin da imagem

#### Comportamento diferente entre `envoy-1.40` e `istio-1.31` sob o mesmo contrato
- **Probabilidade:** média
- **Impacto:** plugin funciona num alvo e falha no outro
- **Mitigação:**
  - Suíte de conformidade do SDK nos dois alvos (seguir, parar, retomar, negar e panic)
  - Capacidades ausentes informadas por `Target().Has`
- **Plano de contingência:** desligar o recurso naquele alvo

#### Uma pessoa para manter SDK, gerador, kit e widgets
- **Probabilidade:** alta
- **Impacto:** atrasos na `v1.0.0`
- **Mitigação:**
  - Gerador derivado do `example/`
  - Catálogo de widgets enxuto
- **Plano de contingência:** adiar `Tabs` e `Chart` de barra

---

### ADRs e próximos passos
ADRs associados
- [ADR-011: Módulo único com pacotes por metade e regras de dependência](adr/ADR-011-modulo-unico-e-regras-de-dependencia.md)
- [ADR-012: ID do plugin no formato `<namespace>.<nome>`](adr/ADR-012-id-com-namespace.md)
- [ADR-013: Manifesto `n3x-plugin.yaml` e gerador `n3x-new`](adr/ADR-013-manifesto-n3x-plugin-yaml-e-gerador.md)
- [ADR-014: Pacote `secret` no SDK, proibido aos plugins](adr/ADR-014-pacote-secret-no-sdk.md)
- [ADR-015: Hooks e capacidades como interfaces opcionais, com config tipada](adr/ADR-015-interfaces-opcionais-e-config-tipada.md)
- [ADR-016: Controller só em segundo plano; conexões próprias permitidas na requisição](adr/ADR-016-controller-so-em-segundo-plano.md)
- [ADR-017: Memória isolada com mapas tipados, cota e descarte do mais antigo](adr/ADR-017-mapas-de-memoria-com-cota.md)
- [ADR-018: Telas de plugin como árvore de widgets, sem HTML](adr/ADR-018-telas-por-widgets.md)
- [ADR-019: Contrato das capacidades privilegiadas do plugin core](adr/ADR-019-contrato-das-capacidades-do-core.md)
- [ADR-020: Schema gerado da struct, `plugin.Secret` e evolução só por adição](adr/ADR-020-schema-da-struct-e-plugin-secret.md)
- [ADR-021: Contrato estável desde a v1.0 e política de depreciação](adr/ADR-021-contrato-estavel-e-depreciacao.md)
- [ADR-022: Métricas com prefixo por plugin e sem limite de cardinalidade](adr/ADR-022-metricas-por-plugin-sem-limite.md)
- Herdados da plataforma (mesma pasta): ADR-004 (capacidades do core), ADR-005 (memória isolada), ADR-008 (exemplos de deploy) e ADR-009 (sem tracing)

Decisões pendentes
- Lista de bibliotecas aprovadas para connectors (Redis, Postgres e HTTP): definir com o protótipo de rate limit
- Lista exata de capacidades por alvo (`proxy.CapX`): depende da prova de carga do `.so` no `istio-1.31` (HLD do `n3x-sidecar`)
- Formato do manifesto embutido e do hash: HLD do `n3x-builder`
- Resolvida: formato da config efetiva enviada ao controller, com mascaramento por `plugin.Secret` e hash (ADR-006 do `n3x-plugin-core`)
- Renderização interna dos widgets (templ ou geração direta de HTML) e tema visual do painel: LLD do `controller/ui`, junto com o HLD do `n3x-controller`

Próximos passos
- Implementar `plugin/` (ID, manifesto, schema por tags e `plugin.Secret`) com testes de tabela e fuzzing
- Implementar `proxy/` e `memory/` com TDD e `AssertZeroAllocs`
- Mover `core/secret` do `n3x-sidecar` para `secret/`
- Implementar `controller/` e `controller/ui`, começando por `Page`, `Column`, `Row`, `Stat`, `Table` e `Detail`
- Escrever o `sdktest` e o `example/` com as duas metades, e então o `n3x-new`
- Publicar `v0.1.0` e validar com o plugin core e um protótipo de rate limit
- Escrever o guia do autor: criar, testar, adicionar ao Dockerfile e configurar no `n3x.yaml`
- Escrever o FDD/LLD do `proxy/` e do renderizador de widgets
