<!--
Relatório de Impacto da Sincronização
- Versão: template sem versão → 1.0.0 (primeira adoção)
- Fonte: DRAFT_CONSTITUITION.md, consolidado em 2026-10-24 a partir do PRD 1.6, dos HLDs 1.2,
  dos ADR-001 a ADR-022 e das FDD-SDK-01 a FDD-SDK-16
- Princípios (placeholders [PRINCIPLE_1..5] substituídos por 9 princípios):
  - I. Independência do Envoy e dos hosts
  - II. Isolamento entre plugins por construção
  - III. Controller fora do caminho da requisição e todo I/O com prazo
  - IV. Desempenho é contrato
  - V. Toda requisição termina e toda negação é ciente de gRPC
  - VI. Segredos nunca vazam
  - VII. Painel seguro e acessível por construção
  - VIII. Contrato estável e evolução só por adição
  - IX. Testes primeiro, sem Envoy e com saídas determinísticas
- Seções adicionadas: Restrições Técnicas e de Plataforma; Fluxo de Desenvolvimento e Portões
  de Qualidade; Governança preenchida
- Seções removidas: nenhuma
- Templates e documentos:
  - ✅ .specify/templates/plan-template.md: sem alteração; o /speckit-plan preenche a
    "Constitution Check" a partir desta constituição
  - ✅ .specify/templates/spec-template.md: sem alteração
  - ✅ .specify/templates/checklist-template.md: sem alteração
  - ✅ .specify/templates/tasks-template.md: testes passam a ser obrigatórios (princípio IX) e
    a convenção de caminhos inclui a biblioteca Go deste repositório
  - ✅ README.md: o mapa de documentação aponta para esta constituição
  - ⚠ .claude/skills/speckit-tasks/SKILL.md e .agents/skills/speckit-tasks/SKILL.md: dizem
    "Tests are OPTIONAL" salvo pedido de TDD; esta constituição exige TDD, o que aciona a
    exceção. Não foram editados (arquivos gerenciados pelo Spec Kit)
  - ⚠ .specify/templates/commands/*.md: diretório inexistente; nada a validar
- Pendências (TODO): nenhuma
-->

# Constituição do n3x-sdk

O `n3x-sdk` é o SDK Go de plugins da plataforma N3X: uma biblioteca sem runtime próprio que roda
dentro do `libn3x.so` do sidecar (metade de proxy) e do binário do controller (metade de
controller). Ela define os contratos estáveis que todo plugin implementa, o kit de testes, o
plugin de exemplo, o gerador `n3x-new` e os exemplos de deploy e a CI cruzada da plataforma.

## Princípios Fundamentais

### I. Independência do Envoy e dos hosts

- O SDK e o código de plugin NÃO DEVEM usar cgo nem importar o SDK Go do Envoy ou pacotes
  internos do `n3x-sidecar` e do `n3x-controller`. O Echo NÃO DEVE aparecer no contrato, que usa
  só `net/http`, `context` e widgets.
- As regras de dependência entre pacotes DEVEM ser conferidas com `go list -deps` na CI do SDK e
  no `n3x-plugin build`:
  - `proxy/` e `memory/` usam, de fora do SDK, só a biblioteca padrão e grpc-go;
  - `proxy/` pode importar `telemetry/`, `memory/` e `plugin/` e NUNCA `controller/`;
  - `controller/ui` não importa `ctrl`;
  - `chain/` importa só a biblioteca padrão, `plugin/` e `proxy/`;
  - `yamlconv/` é o único pacote com dependência YAML de terceiros (`go.yaml.in/yaml/v3`);
  - o `n3x-sidecar` importa o SDK, e o SDK NUNCA importa o sidecar nem o controller.

Justificativa: o SDK Go do Envoy não tem versões com tag e difere entre `envoy-1.40` e
`istio-1.31`. Um contrato próprio mantém os plugins estáveis entre alvos e testáveis sem Envoy.

### II. Isolamento entre plugins por construção

- Memória, métricas, logs e estado DEVEM ser entregues por handle preso ao ID do plugin. NÃO DEVE
  existir acessor global nem forma de pedir o recurso de outro ID.
- A memória tem cota por plugin (16 MiB no sidecar, 64 MiB no controller, ajustável em
  `runtime.plugin_limits["<id>"].memory.max_bytes`). Cota cheia descarta itens (CLOCK) sem
  devolver erro no caminho quente.
- Um plugin só registra serviço gRPC em `n3x.plugins.<namespace>.<nome>.v<major>` e rotas em
  `/plugins/<namespace>/<nome>/`; fora disso, o host recusa na subida.
- Um plugin só acessa dados de outro pela API pública dele (gRPC em processo, papel interno
  `RolePlugin`) e só para IDs declarados em `depends_on`.
- As capacidades privilegiadas (rota pública de login, autenticador de sessão, diretório de
  usuários, interceptor da conexão e `CoreEnv`) DEVEM ser aceitas só de `n3x.core` vindo do
  módulo `github.com/giustech/n3x-plugin-core`; qualquer outro plugin que as declare tem a subida
  recusada.
- O namespace `n3x` é reservado a módulos `github.com/giustech/`; `plugin`, `controller`,
  `sidecar`, `build` e `config` são reservados à plataforma. ID duplicado faz o build falhar.
- Código de plugin NÃO DEVE importar `secret/`, `chain/`, `controller/hostrules` nem `internal/*`,
  nem usar `unsafe` ou `reflect` sobre tipos do SDK.

Justificativa: todos os plugins rodam no mesmo processo. O isolamento é feito por API e por
análise estática, não por hardware, então precisa estar no contrato e não na boa vontade do autor.

### III. Controller fora do caminho da requisição e todo I/O com prazo

- `env.Controller()` DEVE funcionar só dentro de `env.Background()` e só para o próprio namespace
  gRPC do plugin, com teto de 5 s por chamada. Dentro de `s.Go()`, DEVE devolver
  `proxy.ErrRequestPath` na hora, sem tocar a rede.
- Conexões próprias do plugin (`env.Dial()` e os clientes HTTP e TCP do SDK) são permitidas na
  requisição, sempre com o prazo do `Go()` (200 ms por padrão) e a política de falha da instância.
- Toda operação de I/O, página e cartão DEVE ter prazo. A análise estática DEVE recusar no
  plugin: `go` cru, `time.Sleep`, `net.Dial`, `http.Get`, chamadas bloqueantes sem `ctx`,
  `context.Background()`, `context.TODO()`, `os.Exit`, `log.Fatal` e `init()` com efeito colateral.
- Limites por plugin: 8 tarefas de fundo e 16 conexões em `Dial()`, ajustáveis em
  `runtime.plugin_limits["<id>"]`.

Justificativa: a meta é zero requisições com erro causado pelo controller. Um backend lento DEVE
virar política de falha aplicada, nunca sidecar travado.

### IV. Desempenho é contrato

- Os caminhos comuns do SDK (acessores do `Stream`, `Deny`, métricas por handle, `Map.Get`) DEVEM
  ter 0 alocações por operação, com `AssertZeroAllocs` como portão de CI.
- Hooks síncronos do core e do exemplo DEVEM ficar em p50 ≤ 0,05 ms.
- A config de referência (core e exemplo, com body e um grupo com `Go()`) DEVE ficar em até 5
  cruzamentos Envoy→Go por requisição. Configurações com mais de um grupo com `Go()` ou com body em
  vários pedaços podem passar de 5. Hook não implementado NÃO DEVE custar cruzamento nem alocação.
- O SDK NÃO DEVE estourar as metas do sistema: latência adicionada pelo sidecar em p50 ≤ 0,5 ms e
  p99 ≤ 2 ms a 1.000 rps, sidecar ocioso com até 100 MB e 0,05 vCPU, e painel em p95 < 300 ms com
  500 sidecars.

Justificativa: os plugins rodam dentro do Envoy em toda requisição. Regressão de desempenho é
falha de contrato, não detalhe de implementação.

### V. Toda requisição termina e toda negação é ciente de gRPC

- O panic de um plugin DEVE ser contido: com a política `open`, a cadeia segue; com `closed`, a
  requisição é negada (503 ou `UNAVAILABLE`). Nos dois casos, `panics_total` sobe.
- Uma negação NUNCA sai com `grpc-status 0`, e não existe resposta local crua na API. Em HTTP, o
  corpo é JSON `{"code":<status>,"message":"<mensagem>"}` com
  `content-type: application/json; charset=utf-8`; em gRPC, a resposta é trailers-only com
  `grpc-status` e `grpc-message`.
- As oito fases são fixas: `access`, `authn`, `route`, `authz`, `traffic`, `transform`,
  `upstream` e `observe`, em ordem inversa na resposta. `SetRoute` existe só em
  `proxy.RouteStream`, nas fases `access`, `authn` e `route`.
- O uso indevido da API NUNCA é silencioso: vira falha com erro sentinela, sem mudança parcial.

Justificativa: uma negação gRPC que chega ao cliente como sucesso é falha de segurança e bloqueia
a release.

### VI. Segredos nunca vazam

- Campos sensíveis DEVEM usar `plugin.Secret`, que mostra `***` em log, `fmt`, JSON e erros. O
  valor só sai com `.Reveal()`, e a análise estática DEVE recusar `.Reveal()` dentro de chamada
  de log.
- `secret/` é o único código do formato `ENC[v1:...]` (AES-256-GCM, com AAD `n3x:v1:<serviço>`
  no sidecar e `n3x:v1:controller:<id do plugin>` no controller) e é proibido aos plugins, que
  recebem a config já decifrada.
- A chave AES DEVE vir só de arquivo montado, NUNCA de variável de ambiente. Envelope que não
  decifra impede a subida, sem logar o valor.
- Nenhum segredo ou token de sessão pode aparecer em log, mensagem de erro, métrica, evento ou
  config enviada ao controller. O kit de testes DEVE varrer toda saída capturada com um canário.
- Nenhum comportamento do N3X é configurado por variável de ambiente.

Justificativa: um segredo passa pelo operador, pelo entrypoint, pelo `.so` ou host, pelo plugin e
pelo core. O tipo protege todos os caminhos de saída, não só o mascaramento.

### VII. Painel seguro e acessível por construção

- As telas de plugin DEVEM ser árvores de widgets do `controller/ui`. Nenhum plugin escreve HTML,
  e não existe escape para HTML nem templ próprio na v1.
- O renderizador DEVE escapar todo texto, respeitar a CSP do painel (nenhum JavaScript além do
  htmx; `Chart` em SVG gerado no servidor) e atender WCAG 2.1 AA.
- `Form` e `Action` têm CSRF automático e, sem papel declarado, exigem root. A exportação de
  tabela exige root por padrão.
- A autorização DEVE ser aplicada pelo controller antes de qualquer handler de plugin, que recebe
  `ctrl.Caller(ctx)` já preenchido (`root`, leitura ou `sidecar`, mais o interno `RolePlugin`).

Justificativa: visual, acessibilidade e segurança ficam garantidos num lugar só, sem XSS vindo de
plugin e sem prender o contrato à versão do Echo.

### VIII. Contrato estável e evolução só por adição

- O SDK segue semver estrito desde a `v1.0.0`, sem pacote `x/`: patch corrige, minor só
  acrescenta (widget, hook ou capacidade) e major quebra.
- Uma API marcada `// Deprecated:` DEVE continuar funcionando até a próxima major, por no mínimo
  6 meses; o build mostra aviso sem falhar.
- A config de plugin evolui só por adição dentro da major do plugin (campo novo com `default` ou
  opcional), conferida com `n3x-new schema-check`. Os protos em
  `n3x.plugins.<namespace>.<nome>.v<major>` evoluem só por adição, conferidos com `buf breaking`.
  Na série `0.x`, a mesma linha é a mesma minor.
- A faixa `sdk` do `n3x-plugin.yaml` usa a gramática restrita `>=X.Y.Z <A.B.C`; o builder recusa
  plugin cuja faixa não inclua o SDK da imagem.
- Antes da `v1.0.0`, as pré-releases `v0.x` DEVEM ser validadas com o core, o exemplo e um
  protótipo de rate limit.
- Os pacotes de plataforma (`chain/`, `controller/hostrules`, `internal/*`) seguem o semver do
  módulo, mas ficam fora do contrato de plugin. Módulos aninhados têm tags com prefixo.

Justificativa: todos os plugins dependem do SDK, e o operador precisa atualizar imagens sem mudar
o `n3x.yaml`.

### IX. Testes primeiro, sem Envoy e com saídas determinísticas

- TDD é obrigatório: os testes são escritos e DEVEM falhar antes da implementação, em especial no
  contrato de proxy, na cadeia (`chain/`) e na contenção de falhas.
- Tudo DEVE rodar com `go test -race`. Os testes incluem tabela, propriedade e golden (inclusive o
  golden de `go doc -all` da superfície pública), fuzzing de 5 min por alvo na CI noturna e
  benchmarks com `-benchmem`.
- O kit (`sdktest` e `sdktest/ctrltest`) DEVE rodar sem Envoy e sem controller e tomar as mesmas
  decisões de cadeia do executor, pelo pacote `chain/`. As suítes de conformidade rodam no kit e
  nos hosts reais.
- Cobertura mínima: 90% nos pacotes do SDK; 85% em `example/` e `cmd/n3x-new`.
- Schema, manifesto, renderização de widgets e gerador DEVEM produzir os mesmos bytes para a mesma
  entrada. O kit NUNCA chama `time.Sleep`: usa um agendador determinístico.

Justificativa: os riscos mais caros (ignorar o `ctx`, alocar no caminho quente, negação gRPC
errada, segredo em log) só apareceriam em produção sem esses portões.

## Restrições Técnicas e de Plataforma

- **Linguagem e alvos:** Go 1.26 ou superior (o mínimo só sobe em minor), com generics e sem cgo,
  em amd64 e arm64, para os alvos `envoy-1.40` e `istio-1.31`.
- **Dependências:** biblioteca padrão, grpc-go, protobuf e `go.yaml.in/yaml/v3` (só em
  `yamlconv/`); `log/slog` para logs; htmx 2.0.x só dentro do renderizador; templ só no host do
  controller. buf v2, `govulncheck` e `gosec` na CI. Toda dependência de terceiros nova DEVE ser
  justificada na "Complexity Tracking" do plano.
- **Natureza:** o SDK é biblioteca sem runtime próprio e não persiste nada. O banco, quando
  existe, é sempre do plugin.
- **Módulos:** `github.com/giustech/n3x-sdk`, com os módulos aninhados `example/`,
  `deploy/tools`, `deploy/e2e` (com `plugins/faults`) e `deploy/apps/eco`.
- **Nomes:** ID `<namespace>.<nome>`, com cada parte em `^[a-z][a-z0-9_]{1,31}$`, e derivação
  mecânica para gRPC, rotas e métricas (`n3x_<namespace>_<nome>_*`, com rótulo `instance`
  automático). O manifesto do plugin é `n3x-plugin.yaml`; `n3x.yaml` é só arquivo do operador.
  Config tipada com tags `json` em snake_case e `desc` obrigatória.
- **Portas:** sidecar `17000`, `17021` e `17090`; controller `17443`, `17444` e `17091`; admin do
  Envoy só no socket `/tmp/n3x/admin.sock`.
- **Topologias de referência em `deploy/`:** docker compose, Kubernetes 1.29+ com sidecar nativo,
  ECS `awsvpc` e Istio (bloqueante da v1.0.0). O mesmo `n3x.yaml` vale byte a byte em todas; o
  controller roda em instância única por ambiente; fora do Istio, a aplicação escuta só em
  `127.0.0.1`. A chave AES vai por volume preenchido por `n3x-files` no compose, por `Secret` em
  volume `projected` no Kubernetes e no kind, e por `n3x-files` lendo do Secrets Manager e do
  Parameter Store no ECS.
- **Idioma:** documentação, especificações, planos, tarefas e mensagens em pt-BR; identificadores,
  flags, rótulos, eventos, nomes de arquivos e branches em inglês.
- **Fora da v1:** plugins de comportamento de tráfego (além do exemplo), body de resposta
  (`OnResponseBody`, por adição), tracing distribuído, plugins de terceiros, `ui.External`,
  auditoria (v2), Helm, Terraform, CloudFormation e Windows sem WSL2.
- **Capacidade:** uma pessoa mantém os cinco repositórios do N3X. Escopo enxuto e automação de CI
  são restrições de projeto.

## Fluxo de Desenvolvimento e Portões de Qualidade

- **Ordem de trabalho:** as fases do Spec Kit (`specify`, `clarify`, `plan`, `tasks`,
  `implement`) seguem a ordem de `specs/README.md` (01 a 10); cada especificação depende só das
  anteriores.
- **Precedência documental:** o PRD 1.6 vale para escopo e metas de negócio. O HLD 1.2 e os ADRs,
  com as atualizações datadas, são a fonte técnica normativa. As FDDs detalham e, onde divergem,
  prevalecem o HLD e o ADR. O `.md` vale sobre o `.json`.
- **Exceções registradas em 2026-10-24:**
  - cruzamentos Envoy→Go seguem a FDD-SDK-05;
  - a matriz de CI segue a FDD-SDK-16;
  - a chave AES no Kubernetes e no kind segue a FDD-SDK-15.
- **Portões de todo PR:**
  - `go vet`, `go test -race` e `go list -deps` (regras do princípio I);
  - golden de `go doc -all`, `AssertZeroAllocs` num passo sem `-race` e cobertura mínima;
  - `govulncheck`, `gosec` e busca de segredos em logs, erros e config;
  - conformidade com o adaptador falso do executor e integração em compose amd64 com
    `envoy-1.40`.
- **Noturno e pré-release:**
  - arm64, kind, carga open-loop e longa duração;
  - fuzzing e conformidade contra proxies reais;
  - Istio no tier próprio `kind-istio`;
  - ECS por roteiro manual guiado.
- **Portões de release:**
  - uma negação gRPC que chega como sucesso bloqueia a release;
  - todos os cenários bloqueantes DEVEM passar em amd64 e arm64;
  - o Istio bloqueia a `v1.0.0`, e só uma decisão registrada no PRD permite adiar o alvo;
  - cada pré-release do SDK só sai com o exemplo verde;
  - mudança em `deploy/`, `docs/operador/` ou `docs/autor/` exige os roteiros manuais aprovados.
- **Revisão:** todo PR DEVE verificar a conformidade com esta constituição. Todo plano registra a
  "Constitution Check", e qualquer violação só é aceita com justificativa na
  "Complexity Tracking".

## Governança

- Esta constituição prevalece sobre qualquer outra prática do repositório. Um conflito entre ela
  e uma spec, um plano ou uma lista de tarefas é sempre CRITICAL e se resolve ajustando o
  artefato, nunca diluindo o princípio.
- **Emendas:** só por PR que altera `.specify/memory/constitution.md`, com o Relatório de Impacto
  da Sincronização atualizado, a nova versão e a propagação aos templates de `.specify/templates/`.
  Uma emenda que muda um contrato compartilhado com outro repositório do N3X DEVE vir acompanhada
  de ADR novo ou de atualização datada de ADR existente.
- **Versionamento desta constituição:**
  - MAJOR para remoção ou redefinição incompatível de princípio;
  - MINOR para princípio ou seção nova, ou expansão material de orientação;
  - PATCH para esclarecimento de redação.
- **Conformidade:**
  - o `/speckit-plan` preenche a "Constitution Check" antes da pesquisa e a reavalia depois do
    design;
  - o `/speckit-analyze` trata violação desta constituição como CRITICAL;
  - a revisão de conformidade é repetida a cada release do SDK.
- **Guia de desenvolvimento em runtime:** `CLAUDE.md` (regras de idioma das saídas do Spec Kit) e
  `README.md` (visão do projeto e das topologias).

**Versão**: 1.0.0 | **Ratificada em**: 2026-10-24 | **Última emenda**: 2026-10-24
