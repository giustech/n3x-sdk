# Rascunho para a constituição do n3x-sdk

> Insumo para `/speckit-constitution`; não é a constituição. Fontes: PRD v1.6, HLD do SDK v1.2, HLD de plataforma v1.2, ADR-001 a ADR-022 e FDD-SDK-01 a FDD-SDK-16 (arquivos `.md`).

## 1. O que é

- SDK de plugins da plataforma N3X (Network Navigation Node eXtensible, pronúncia "Nex"), um middleware de rede extensível que funciona como camada de override sobre um proxy Envoy em modelo sidecar.
- Biblioteca Go sem runtime próprio. Roda dentro de dois processos que não são dele: o `libn3x.so` do sidecar (metade de proxy) e o binário do controller (metade de controller).
- Módulo `github.com/giustech/n3x-sdk`, licença Apache 2.0, publicado por tag semver no Go module proxy. Módulos aninhados com tags próprias: `example/`, `deploy/tools`, `deploy/e2e` (com `plugins/faults`) e `deploy/apps/eco`.
- Um dos cinco repositórios do N3X v1: `n3x-sidecar`, `n3x-controller`, `n3x-sdk`, `n3x-builder` e `n3x-plugin-core`.
- Situação atual: só documentação. Nenhuma fase do Spec Kit foi executada; `specs/README.md` planeja 10 especificações em ordem de dependência (01 a 08 prontas; 09 e 10 dependem de repositórios externos).

## 2. Problema de negócio

- Comportamentos de rede (rate limit, forward, redirects, observabilidade) são reimplementados em cada serviço e em cada linguagem. Bibliotecas por serviço geram retrabalho, service mesh é pesado e exige Kubernetes, e API gateways são difíceis de estender.
- O N3X transforma esses comportamentos em plugins compilados num único dynamic module Go dentro do Envoy, configurados no `n3x.yaml` de cada serviço, sem alterar o código da aplicação.
- O SDK resolve, no nível do contrato:
  - plugins que não quebram quando o SDK Go do Envoy muda (ele não tem versões com tag);
  - plugin testável sem Envoy e sem controller;
  - isolamento entre plugins (memória, token, namespace gRPC e rotas) imposto pelo contrato, não pela boa vontade do autor;
  - caminho quente sem alocação e com poucos cruzamentos Envoy→Go;
  - painel com visual, acessibilidade e segurança iguais em todas as telas, sem autor escrevendo HTML.

## 3. O que o repositório entrega

| Área | Pacotes e artefatos |
| --- | --- |
| Identidade e config | `plugin/` (ID, fases, políticas, manifesto, schema por tags, `plugin.Secret`), `yamlconv/`, `schemagen/` |
| Segredos | `secret/` (`ENC[v1:...]`, AES-256-GCM), de uso exclusivo da plataforma |
| Metade de proxy | `proxy/` (hooks, `Stream`, `RouteStream`, negação, `Env`), `telemetry/`, `memory/` |
| Metade de controller | `controller/` (pacote `ctrl`), `controller/ui` (widgets e renderizador), `controller/hostrules` |
| Plataforma | `chain/` (lógica pura da cadeia, compartilhada com o executor do sidecar), `internal/` |
| Testes | `sdktest/`, `sdktest/conformance`, `sdktest/ctrltest`, `sdktest/hostconformance`, `controller/ui/uitest` |
| Autor | `example/` (plugin `n3x.example`), `cmd/n3x-new` (`new` e `schema-check`), `docs/autor/` |
| Operação | `deploy/` (compose, Kubernetes, ECS, Istio), `deploytool`, harness `n3x-e2e`, `docs/operador/` |

## 4. Quem usa

- Autores de plugin: escrevem as metades, testam com o `sdktest` e criam repositórios com o `n3x-new`. Na v1, só o responsável pelo projeto escreve plugins; plugins de terceiros ficam fora da v1.
- Repositórios da plataforma:
  - `n3x-sidecar`: implementa `proxy.Env` e `proxy.Stream`; usa `plugin/`, `secret/` e `chain/`;
  - `n3x-controller`: implementa o host, os registradores e o painel; usa `controller/`, `controller/ui` e `secret/`;
  - `n3x-builder`: confere a faixa `sdk`, gera schema e registro e aplica a análise estática definida pelo SDK;
  - `n3x-plugin-core`: primeiro cliente do contrato e único dono das capacidades privilegiadas.
- Operadores (time de plataforma/SRE): usam os exemplos de `deploy/` e o guia do operador.
- Times de produto: beneficiários indiretos; põem o sidecar ao lado da aplicação sem mudar código.
- CIs de todos os repositórios: a CI cruzada é orquestrada a partir de `deploy/`.

## 5. Onde roda

- Metade de proxy: no `libn3x.so`, um por alvo (`envoy-1.40`, `istio-1.31`) e arquitetura (amd64, arm64), carregado do disco pelo Envoy; um único filtro executa todos os plugins.
- Metade de controller: no binário do controller, em instância única por ambiente.
- Topologias de referência: docker compose; Kubernetes 1.29+ com sidecar nativo; ECS `awsvpc` (Fargate ou EC2); Istio 1.31 (conteúdo reservado à FDD 49, de outro repositório, e bloqueante da v1.0.0).
- Portas: sidecar `17000` (entrada), `17021` (health), `17090` (métricas); controller `17443` (gRPC, TLS 1.3), `17444` (painel HTTPS), `17091` (métricas e health). Admin do Envoy só no socket `/tmp/n3x/admin.sock`.
- Tráfego HTTP/1.1, HTTP/2 e gRPC; aplicação em qualquer linguagem, escutando só em `127.0.0.1` fora do Istio.

## 6. Conceitos centrais do contrato

- Plugin com até duas metades, registrado por função gerada pelo builder (`agent.Plugin()`, `controller.Plugin()`), sem `init()`. Hooks e capacidades são interfaces opcionais; a config é tipada por generics (`Define[C](New)`).
- Oito fases fixas: `access`, `authn`, `route`, `authz`, `traffic`, `transform`, `upstream`, `observe` (ordem inversa na resposta). `SetRoute` só existe em `RouteStream`, nas três primeiras.
- Políticas de falha `open` e `closed`; toda requisição termina.
- ID `<namespace>.<nome>` (cada parte `^[a-z][a-z0-9_]{1,31}$`), derivado de forma mecânica para gRPC (`n3x.plugins.<ns>.<nome>.v<major>`), rotas (`/plugins/<ns>/<nome>/`) e métricas (`n3x_<ns>_<nome>_*`). Namespaces reservados: `n3x` (só módulos `github.com/giustech/`), `plugin`, `controller`, `sidecar`, `build` e `config`.
- Manifesto `n3x-plugin.yaml` na raiz do repositório do plugin; o nome `n3x.yaml` fica só para os arquivos do operador.
- Schema gerado da struct Go em `types/` (tags `json` em snake_case, `n3x`, `desc` obrigatória e `default`).
- Memória isolada por handle preso ao ID, com cota por plugin (16 MiB no sidecar, 64 MiB no controller) e descarte aproximado CLOCK.
- Papéis do `Caller`: `root`, leitura (`RoleRead`) e `sidecar`, mais o papel interno `RolePlugin` nas chamadas entre plugins, feitas por gRPC em processo e só para dependências declaradas em `depends_on`.
- Capacidades privilegiadas (login público, autenticador, diretório de usuários, interceptor da conexão, `CoreEnv`) aceitas só de `n3x.core` vindo de `github.com/giustech/n3x-plugin-core`.

## 7. Regras invioláveis (candidatas a princípios)

1. Independência de Envoy e hosts: sem cgo; nenhum pacote importa o SDK Go do Envoy nem internos do `n3x-sidecar` ou do `n3x-controller`; o Echo não aparece no contrato.
2. Regras de dependência, conferidas na CI do SDK e no `n3x-plugin build`: `proxy/` e `memory/` usam, de fora do SDK, só a stdlib e grpc-go; `proxy/` pode importar `telemetry/`, `memory/` e `plugin/` e nunca `controller/`; `controller/ui` não importa `ctrl`; `chain/` importa só stdlib, `plugin/` e `proxy/`; `yamlconv/` é o único pacote com YAML de terceiros (`go.yaml.in/yaml/v3`); código de plugin não importa `secret/`, `chain/`, `controller/hostrules` nem `internal/*`; o sidecar importa o SDK, nunca o inverso.
3. Controller fora do caminho da requisição: `env.Controller()` só em `env.Background()` e só no próprio namespace gRPC; dentro de `s.Go()` devolve `ErrRequestPath` sem tocar a rede.
4. Todo I/O tem prazo. Proibidos ao plugin: `go` cru, `time.Sleep`, `net.Dial`, `http.Get`, `context.Background()`, `context.TODO()`, `os.Exit`, `log.Fatal`, `init()` com efeito colateral e `unsafe`/`reflect` sobre tipos do SDK.
5. Desempenho é contrato: 0 alocações nos caminhos comuns (`AssertZeroAllocs` como portão de CI); hooks síncronos com p50 ≤ 0,05 ms; até 5 cruzamentos Envoy→Go por requisição na config de referência (core e exemplo, com body e um grupo com `Go()`), podendo passar de 5 com mais de um grupo com `Go()` ou body em vários pedaços.
6. Negação sempre ciente de gRPC e nunca com `grpc-status 0`; não existe resposta local crua. HTTP em JSON `{"code","message"}`; gRPC em trailers-only.
7. Segredos: campos sensíveis em `plugin.Secret` (`***` em qualquer saída, valor só com `.Reveal()`); `secret/` é o único código do formato `ENC[...]` e é proibido aos plugins; chave AES só em arquivo montado; nenhum segredo ou token em log, erro, métrica ou config enviada ao controller.
8. Nenhum comportamento do N3X configurado por variável de ambiente.
9. Isolamento entre plugins por handle preso ao ID, sem acessor global (memória, métricas, logs e estado).
10. Painel seguro por construção: plugin nunca gera HTML; todo texto é escapado; CSP sem JavaScript além do htmx; WCAG 2.1 AA; `Form` e `Action` com CSRF e exigindo root por padrão.
11. Contrato estável: semver estrito desde a `v1.0.0`, sem pacote `x/`; minor só acrescenta e major quebra; depreciação mínima de 6 meses; config e protos dos plugins evoluem só por adição (`n3x-new schema-check`, `buf breaking`).
12. Determinismo: schema, manifesto, renderização e gerador produzem os mesmos bytes para a mesma entrada.
13. Idioma: documentação e mensagens em pt-BR; identificadores, flags, rótulos e eventos em inglês.

## 8. Stack

- Go 1.26 ou superior (o mínimo só sobe em minor), com generics, sem cgo, em amd64 e arm64.
- grpc-go e protobuf; `go.yaml.in/yaml/v3` isolado em `yamlconv/`; `log/slog`; htmx 2.0.x só no renderizador; SVG no servidor para `Chart`; templ só no host do controller.
- buf v2, `govulncheck` e `gosec` na CI.
- Deploy e CI: Docker 24 e Compose v2.20, Kubernetes 1.29+ (kustomize 5.x, kubeconform), ECS, kind v0.24, Prometheus 2.53 e Grafana 10.4.

## 9. Qualidade e testes

- TDD; `go test -race` em tudo; testes de tabela, propriedade e golden (inclusive `go doc -all` da superfície pública); fuzzing de 5 min por alvo na CI noturna.
- Cobertura mínima: 90% nos pacotes do SDK; 85% em `example/` e `cmd/n3x-new`.
- Kit sem Envoy e sem controller, com as mesmas decisões de cadeia do executor (`chain/`) e suítes de conformidade que rodam no kit e nos hosts reais.
- Canário de segredos em toda saída capturada nos testes.
- Matriz de CI cruzada: no PR, compose amd64 com `envoy-1.40` e conformidade com o adaptador falso do executor; no noturno e no pré-release, arm64, kind, carga, longa duração e conformidade contra proxies reais; Istio em tier próprio (`kind-istio`, FDD 49); ECS em roteiro manual guiado.
- Release bloqueada se a negação gRPC chegar como sucesso; o Istio bloqueia a v1.0.0.

## 10. Versionamento e governança

- Pré-releases `v0.x` validadas com o core, o exemplo e um protótipo de rate limit antes da `v1.0.0`.
- O builder recusa plugin cuja faixa `sdk` (`>=X.Y.Z <A.B.C`) não inclua o SDK da imagem; API `// Deprecated:` gera só aviso.
- Precedência documental: PRD 1.6 para escopo e metas de negócio; HLD 1.2 e ADRs, com as atualizações datadas, como fonte técnica normativa; as FDDs detalham e, onde divergem, prevalecem HLD e ADR; o `.md` vale sobre o `.json` (os `.json` estão desatualizados).
- Exceções decididas na consolidação (2026-10-24): cruzamentos Envoy→Go seguem a FDD-SDK-05; a matriz de CI segue a FDD-SDK-16; a chave AES no Kubernetes e no kind segue a FDD-SDK-15 (`Secret` montado em volume `projected`, sem `n3x-files`).
- Uma pessoa mantém os cinco repositórios: escopo enxuto e automação de CI são restrições de projeto.

## 11. Fora da v1

- Plugins de comportamento de tráfego (além do exemplo), body de resposta (`OnResponseBody`, por adição), tracing distribuído, plugins de terceiros, `ui.External`, auditoria (v2), Helm, Terraform e CloudFormation, Windows sem WSL2.
