### HLD: N3X Plataforma (visão de sistema e contratos transversais)

Versão: 1.1
Data: 2026-10-20
Responsável: Guilherme Giusti

---

### Objetivo técnico
Dar a visão de sistema do N3X (contexto e containers) e definir os contratos que atravessam todos os repositórios e não têm um dono único: autenticação de ponta a ponta entre sidecar e controller, topologias de deploy, mapa de portas, metas não funcionais do sistema inteiro, regras de versionamento entre repositórios e matriz de CI entre componentes.

Contratos com dono (SDK, plugin core, sidecar, controller e builder) são apenas referenciados aqui, conforme `docs/hld/HLDs-N3X.md`. As metas não funcionais ficam registradas no nível do sistema; a divisão por componente fica no HLD de cada repositório (ADR-010).

Problemas técnicos endereçados
- Temas transversais (autenticação, portas, versionamento, CI) duplicados ou contraditórios entre os cinco HLDs de repositório
- Falta de uma visão única de como os cinco repositórios se compõem em runtime e em build
- Metas não funcionais do PRD sem um dono no nível do sistema

Base: `docs/prd/PRD-N3X.md` versão 1.5, de 2026-10-20. Requisitos cobertos: requisitos não funcionais do sistema; FR-002, FR-005, FR-011 e FR-016 na autenticação de ponta a ponta.

Dependências com outros sistemas
- Repositórios N3X: `n3x-sidecar`, `n3x-controller`, `n3x-sdk`, `n3x-builder` e `n3x-plugin-core`
- Aplicação do cliente, em qualquer linguagem, que recebe tráfego só pelo sidecar
- Orquestradores: Kubernetes (1.29 ou superior), ECS e docker compose
- Istio 1.31, apenas como compatibilidade
- Imagens base `envoyproxy/envoy:v1.40.x` e `istio/proxyv2:1.31.x`, fixadas por digest
- Registro de containers do operador, Go module proxy e GitHub (build)
- Bancos dos plugins que guardam dados; o controller não tem armazenamento próprio, e o plugin core da v1 guarda tudo em memória (ADR-001 do `n3x-plugin-core`)
- Prometheus, opcional, para as métricas operacionais

---

### Arquitetura geral
O N3X tem dois planos.

- **Plano de runtime:** um sidecar Envoy por aplicação, no mesmo namespace de rede dela, carregando um único `libn3x.so` (dynamic module Go) com um único filtro que executa os plugins em oito fases fixas. Cada sidecar abre uma conexão gRPC com TLS 1.3 para um controller central, que só observa. O controller roda em **instância única, sempre um por ambiente** (ADR-001), fora do caminho crítico: se ele cair, o tráfego e os plugins continuam.
- **Plano de build:** o `n3x-builder` compila os plugins via Dockerfile multi-stage e gera o `libn3x.so` (por alvo e arquitetura) e o binário do controller, ambos com manifesto embutido.

Camadas de responsabilidade: aplicação → sidecar (Envoy + core do `.so` + plugins) → controller (servidor gRPC + host de plugins + painel, sem armazenamento próprio) → banco de cada plugin que guarda dados.

**Regras de versionamento entre repositórios**
- Cada repositório e cada imagem têm semver próprio (ADR-001 do `n3x-builder`). O manifesto grava a versão do `core/` ou do `host/` compilado, e o runtime aceita a mesma major com minor da imagem igual ou maior; o README do `n3x-builder` publica os pares testados. O catálogo de alvos fixa a combinação suportada de SDK, alvo e imagens
- SDK em semver: quebra de contrato só em major. O `n3x-plugin build` recusa plugin fora da faixa de SDK da imagem (FR-015)
- Protos de plugin em `n3x.plugins.<namespace>.<nome>.v<major>`, evoluindo só por adição dentro da major, com `buf breaking` em todo PR
- Sidecar e controller: o proto do core é aceito na mesma major; fora dela, `FAILED_PRECONDITION` (FR-016)
- Listas de plugins diferentes entre sidecar e controller são detectadas pelo manifesto e alertadas no painel (FR-006)

**Matriz de CI entre repositórios** (orquestrada a partir de `n3x-sdk/deploy/`, ADR-008)

| Teste | Repositórios | Alvos | Arquiteturas | Quando |
| --- | --- | --- | --- | --- |
| Unitários com `-race`, fuzzing, benchmarks com 0 alocações | Todos | n/a | amd64 | Todo PR |
| Conformidade do SDK (seguir, parar, retomar, negar, panic) | sdk, sidecar | `envoy-1.40`, `istio-1.31` | amd64, arm64 | Todo PR |
| Build e carga do `.so` (cenários de falha do `n3x-plugin build`) | builder | `envoy-1.40`, `istio-1.31` | amd64, arm64 | Todo PR |
| Integração ponta a ponta (core + plugin de exemplo + controller, HTTP/1.1, HTTP/2, gRPC) | Todos | `envoy-1.40`, `istio-1.31` | amd64 no PR, arm64 à noite | PR e noturno |
| Compatibilidade de protos (`buf breaking`) | plugin-core e plugins | n/a | n/a | Todo PR |
| Segurança (`govulncheck`, `gosec`, busca de segredos em logs, erros e config) | Todos | n/a | amd64 | Todo PR |
| Ambientes: docker compose, kind, kind + Istio | sdk (deploy) | ambos | amd64 | Compose no PR, kind à noite |
| Carga open-loop (1.000 rps) e 500 sidecars simulados | sidecar, controller | `envoy-1.40` | amd64 | Noturno, máquina dedicada |
| Longa duração (vazamento de goroutines) | sidecar | `envoy-1.40` | amd64 | Noturno e pré-release |
| Próxima versão candidata do Envoy e cada patch do Istio | sidecar, builder | candidatos | amd64 | Agendado |
| ECS | sdk (deploy) | `envoy-1.40` | amd64 | Manual guiado, pré-release |

Ambiente de implantação
- Agnóstico: cloud ou on-premises, onde houver containers, sem recurso exclusivo de orquestrador
- **docker compose:** controller como serviço próprio; aplicação com `network_mode: "service:<sidecar>"`, escutando em `127.0.0.1`; `n3x.yaml`, chave AES e CA montados por volume
- **Kubernetes (1.29 ou superior):** sidecar nativo (`initContainers` com `restartPolicy: Always`), que sobe antes e desce depois da aplicação (ADR-007); probes no health `17021`; `n3x.yaml` por ConfigMap e chave por Secret montado como arquivo
- **ECS:** sidecar e aplicação na mesma task (`awsvpc`); aplicação com `dependsOn` do sidecar; health check do container no `17021`; chave por arquivo montado (hipótese: volume populado por secret)
- **Istio:** o Istio manda. A imagem `proxyv2` injetada é trocada pela imagem N3X baseada nela, com o `.so` dentro; o tráfego entra pelos listeners e portas do Istio e um `EnvoyFilter` estático insere o filtro do N3X; o `n3x.yaml` é montado no pod (ADR-006)
- Regra comum: a aplicação nunca fica exposta; fora do Istio, ela escuta só em `127.0.0.1`
- Exemplos de referência de cada topologia ficam em `n3x-sdk/deploy/` (ADR-008)

Tecnologias principais
- Envoy 1.40 e istio-proxy 1.31 com dynamic modules
- Go 1.26 ou superior, sem cgo nos plugins
- grpc-go, protobuf e buf
- Echo v5 + templ + htmx no painel
- Banco próprio por plugin que guarda dados (o plugin core não usa banco na v1, ADR-001 do `n3x-plugin-core`)
- Docker buildx (amd64 e arm64) e imagens OCI fixadas por digest
- Prometheus (opcional)

Padrões adotados
- Sidecar como única porta de entrada da aplicação
- Plugin de duas metades (proxy e controller), compilado estaticamente
- Cadeia de filtros em oito fases fixas dentro de um único filtro
- Controller fora do caminho crítico, com degradação graciosa
- Config estática e imutável, alterada só por redeploy
- Contrato gRPC por plugin, num único servidor e numa única conexão compartilhada
- Isolamento por plugin: memória (ADR-005), banco próprio (ADR-002 do `n3x-controller`) e namespace gRPC e HTTP
- Login com troca de credencial por token de sessão sem estado (ADR-003)

---

### Componentes e responsabilidades
| Componente | Responsabilidades | Dependências |
| ----------- | ----------------- | ------------ |
| `n3x-sidecar` | Entrada única da aplicação; bootstrap por `n3x.yaml`; decifra `ENC[...]`; executa a cadeia de plugins; contém falhas por plugin; mantém a conexão gRPC compartilhada; drena no desligamento | Envoy ou istio-proxy, `libn3x.so`, chave AES em arquivo, controller (opcional) |
| `n3x-controller` | Servidor gRPC único (TLS 1.3, interceptor de autorização); host de plugins; painel; sessão do painel e root; verificador de credencial de serviço; sem armazenamento próprio | Plugins compilados |
| `n3x-sdk` | Contratos de proxy e de controller; memória isolada por plugin; kit de testes; plugin de exemplo; exemplos de deploy e CI cruzada | Nenhuma em runtime (biblioteca) |
| `n3x-builder` | Imagens `sidecar-builder` e `controller-builder` por alvo e arquitetura; `n3x-plugin add/build`; análise estática; catálogo de alvos; manifesto embutido | SDK, Go module proxy, GitHub, imagens base |
| `n3x-plugin-core` | Login e renovação da sessão; interceptor que anexa o token a toda chamada; autenticador do servidor gRPC; registro, heartbeat, eventos, inventário e configs; chave HMAC dos tokens; usuários de leitura vindos do `n3x.yaml`, sem gestão pelo painel; auditoria só na v2 | SDK, conexão compartilhada, memória própria na v1 (ADR-001 a ADR-004 do `n3x-plugin-core`) |
| Aplicação | Recebe tráfego só via sidecar | Sidecar |
| Orquestrador | Coloca sidecar e aplicação no mesmo namespace de rede; reinicia; executa probes; monta arquivos de config e chaves | Imagens no registro |

Quem persiste: só os plugins, cada um com o próprio banco; o host do controller não persiste nada (ADR-002 do `n3x-controller`). Quem faz cache: ninguém externo; cada plugin tem apenas a própria memória isolada. Quem orquestra a requisição: o executor da cadeia no `.so`.

---

### Fluxo de requisições e de dados
**Fluxo de requisição**
- O cliente conecta na entrada do sidecar (`17000`, ou o listener do Istio no modo Istio)
- O Envoy normaliza o path e chama o filtro único do N3X
- O `.so` executa os plugins nas fases `access`, `authn`, `route`, `authz`, `traffic`, `transform`, `upstream` e `observe`; uma negação sai com status HTTP e `grpc-status` corretos
- O Envoy encaminha para a aplicação em `127.0.0.1:<porta>`, mantendo o protocolo
- A resposta volta pelas fases em ordem inversa e todos os plugins são notificados no fim

**Fluxo de controle (sempre em segundo plano, nunca no caminho da requisição)**
- O `.so` sobe e abre a conexão gRPC com TLS 1.3, conferindo o certificado do controller com o bundle de CA montado
- O plugin core chama `n3x.plugins.n3x.core.v1.Login` com serviço, instância e segredo do serviço
- O controller confere o segredo contra o hash bcrypt e o plugin core devolve um token de sessão assinado com HMAC-SHA256 (serviço, instância, emitido em, expira em e nonce; ADR-005 do `n3x-plugin-core`)
- O plugin core guarda o token no próprio espaço de memória e o interceptor de cliente o anexa a toda chamada de qualquer plugin
- O plugin core registra o sidecar (manifesto, config mascarada e hash), mantém o heartbeat a cada `controller.heartbeat_interval` (padrão de 10 s), envia eventos e avisa quando sai
- Uma goroutine renova o token em 80% do prazo, com jitter; em `UNAUTHENTICATED`, faz login de novo com jitter e reenvia o registro
- Enquanto o login não completa ou o controller está fora, as chamadas dos plugins esperam no buffer limitado (descarta os mais antigos) ou recebem `UNAVAILABLE`

**Fluxo de dados**
- `n3x.yaml` do serviço → entrypoint (valida, decifra `ENC[...]`) → bootstrap do Envoy → config dos plugins no `.so`
- Config efetiva → mascaramento pelo schema → hash → controller → plugin core → memória do core → painel (formato no ADR-006 do `n3x-plugin-core`)
- Eventos do sidecar → controller → inventário e painel
- Dockerfile → `sidecar-builder` e `controller-builder` → `n3x-plugin add/build` → `libn3x.so` e binário com manifesto → imagens finais no registro

---

### Modelo de dados (alto nível)
Entidades principais
- Serviço: nome (chave única no ambiente) e hashes bcrypt da credencial
- Instância de sidecar: identidade (nome do pod ou ID do container), alvo, versão do Envoy, estado e "sessão emitida em"
- Manifesto: plugins, versões, alvo e hash, embutido no `.so` e no controller
- Config efetiva: hash e conteúdo com segredos mascarados
- Evento: panic, disjuntor aberto, plugin desligado, erro de config, início de drenagem e segredo em claro (lista no ADR-007 do `n3x-plugin-core`)
- Usuário de leitura, declarado na seção do core no `n3x.yaml` do controller; Registro de auditoria só na v2 (ADR-002 do `n3x-plugin-core`)
- Sessão: não é armazenada; existe só como token assinado em memória no sidecar

Relações
- Serviço 1:N Instância
- Instância 1:1 Manifesto e 1:1 Config efetiva atual
- Instância 1:N Evento
- Usuário 1:N Registro de auditoria (v2)

Fonte de verdade
- `n3x.yaml` de cada serviço: config do sidecar e dos plugins
- `n3x.yaml` do controller: root, hashes das credenciais e config de cada plugin, inclusive a conexão de banco dele; na seção do core, usuários de leitura e chave HMAC opcional
- Memória do plugin core na v1: estado observado, configs e eventos, reconstruídos pelos sidecars após restart (ADR-001 do `n3x-plugin-core`)
- Isolamento: um controller por ambiente, então o nome do serviço basta como chave (ADR-001); cada plugin com o próprio banco, e um plugin acessa dados de outro só pela API pública dele (ADR-002 do `n3x-controller`)

---

### Interfaces públicas
| Nome | Tipo | Protocolo | Exposição | SLAs/Limites |
| ---- | ---- | ---------- | --------- | ------------- |
| Entrada do sidecar (`17000`) | API | HTTP/1.1, HTTP/2, gRPC | Externa | Latência adicionada p50 ≤ 0,5 ms e p99 ≤ 2 ms a 1.000 rps |
| Health do sidecar (`17021`) | API | HTTP | Interna (orquestrador) | Pronto em até 5 s após o container subir |
| Métricas do sidecar (`17090`) | API | HTTP (Prometheus) | Interna | n/a |
| Admin do Envoy (`127.0.0.1:17001`) | API | HTTP | Só localhost | n/a |
| Login (`n3x.plugins.n3x.core.v1.Login`, porta `17443`) | API | gRPC, TLS 1.3 | Interna, única rota sem sessão | Limite de tentativas por IP e por serviço |
| Servidor gRPC do controller (`n3x.plugins.<namespace>.<nome>.v1`, `17443`) | API | gRPC, TLS 1.3 | Interna | 500 sidecars em 1 vCPU e 512 MB |
| Painel do controller (`17444`) | API | HTTPS | Interna | p95 < 300 ms |
| Métricas e health do controller (`17091`) | API | HTTP (Prometheus) | Interna | n/a |

No modo Istio, entrada e health do sidecar são as do próprio Istio; o N3X não abre `17000` nem `17021` e só acrescenta o que é dele, como as métricas do `.so` (ADR-006). Os contratos de cada interface pertencem aos HLDs do sidecar, do controller e do plugin core.

---

### Considerações de escalabilidade e disponibilidade
Abordagem geral
- Sidecar escala 1:1 com a aplicação, sem estado compartilhado
- Controller em instância única por ambiente, com escala vertical; fora do caminho crítico do tráfego
- O estado observado se reconstrói sozinho: todo sidecar refaz login e registro ao reconectar

Técnicas aplicadas
- Overload manager do Envoy, `GOMEMLIMIT` e `GOMAXPROCS` ≥ workers do Envoy
- Buffer limitado no sidecar como backpressure para o controller
- Limites de conexões e streams no servidor gRPC (`RESOURCE_EXHAUSTED`)
- Jitter no novo login e na reconexão, evitando rajadas após restart do controller
- Chave HMAC opcional na seção do core (`session.hmac_key`): sessões sobrevivem ao restart e podem ser validadas por mais de uma réplica no futuro (ADR-004 do `n3x-plugin-core`)
- Cota de memória por plugin
- Drenagem de 20 s no SIGTERM e sidecar nativo no Kubernetes

Meta de disponibilidade
- Zero requisições da aplicação com erro causado pelo controller
- Controller com 99,5% de disponibilidade mensal, apoiado no restart do orquestrador
- Zero requisições cortadas em rolling update, dentro do prazo de drenagem
- Sidecar de volta ao painel em até 30 s após o controller voltar, cerca de 10 s com `session.hmac_key` configurada no core; sem ela, o restart força novo login de todos, com cerca de um bcrypt por serviço graças ao cache do host, e a meta vale até cerca de 200 serviços (ADR-004 do `n3x-plugin-core`)
- Na v1 nada do core é persistido: o estado observado se reconstrói pelos sidecars; persistência, retenção e backup voltam com o banco da v2

---

### Segurança
Autenticação
- Sidecar: credencial por serviço `{service, secret}`. No `n3x.yaml` do sidecar o segredo fica como `ENC[...]` (AES-256-GCM, chave em arquivo montado, nunca em variável de ambiente); no `n3x.yaml` do controller fica só o hash bcrypt com custo ≥ 12 (ADR-003)
- Login em `n3x.plugins.n3x.core.v1.Login`, única rota sem sessão; o host do controller confere o segredo, com 5 falhas por IP e por serviço bloqueando por 15 min, até 2 bcrypt simultâneos e cache de 1 h por serviço (ADR-004 do `n3x-controller`)
- Sessão: token assinado com HMAC-SHA256, ligado a serviço e instância, com TTL de 1 h e renovação em 80% do prazo (ADR-005 do `n3x-plugin-core`). O controller não guarda tokens; valida assinatura e validade
- Chave HMAC: do plugin core, em `session.hmac_key` na seção dele quando configurada; sem ela, gerada aleatoriamente no `Start` e mantida só em memória (ADR-004 do `n3x-plugin-core`)
- Painel: root vindo do `n3x.yaml` com hash bcrypt e usuários de leitura com hash bcrypt declarados na seção do core no `n3x.yaml`, sem gestão pelo painel na v1 (FR-009, ADR-003 do `n3x-plugin-core`); sessão do painel só em memória no controller (ADR-006 do `n3x-controller`)
- Transporte: TLS 1.3 no gRPC e no painel; o sidecar confere o certificado do controller com um bundle de CA montado

Autorização
- Papéis root, leitura e sidecar, aplicados pelo controller antes de qualquer handler de plugin (FR-005, FR-011)
- Capacidades privilegiadas exclusivas do plugin core: interceptor na conexão compartilhada, autenticador do servidor gRPC, rota pública de login e diretório de usuários (ADR-004 e ADR-003 do `n3x-controller`); o gravador de auditoria fica reservado para a v2
- Nenhum plugin enxerga o token de sessão; ele fica no espaço de memória privado do core
- Isolamento por plugin: memória (ADR-005), banco próprio (ADR-002 do `n3x-controller`), namespace gRPC `n3x.plugins.<namespace>.<nome>.v1` e rotas `/plugins/<namespace>/<nome>/`

Proteção de dados
- Nenhum segredo ou token em log, mensagem de erro ou config enviada ao controller; campos de segredo mascarados pelo schema do plugin
- Admin do Envoy só em localhost e aplicação nunca exposta
- LGPD: na v1 não há trilha de auditoria; login, logout e bloqueio do painel ficam só no log JSON operacional do controller, com a retenção do coletor de logs. A trilha formal, com IP e usuário tratados como dados pessoais e retenção aplicada pelo core, fica para a v2 (ADR-002 do `n3x-plugin-core`)
- Análise estática no build proíbe cgo, `unsafe` e `reflect` sobre tipos do SDK; `govulncheck` em todo build; imagens base fixadas por digest

Gestão de segredos
- Formato único `ENC[...]` (AES-256-GCM) para segredos que precisam ser lidos: credencial no sidecar, chave HMAC do core e senhas de banco dos plugins no `n3x.yaml` do controller
- bcrypt para tudo que só é conferido: root, usuários de leitura e credenciais de serviço
- Chave AES sempre em arquivo montado pelo orquestrador; a chave HMAC, quando configurada, vai cifrada na seção do core
- Ferramentas: `n3x keygen` e `n3x encrypt` no sidecar (ADR-003 do `n3x-sidecar`); `hash`, `credential new`, `encrypt` e `keygen` no controller (ADR-009 do `n3x-controller`)
- Rotação da credencial: um único hash por serviço (ADR-005 do `n3x-controller`); troca do hash, restart do controller e redeploy do serviço; durante a troca os sidecars do serviço somem do painel, sem afetar o tráfego

---

### Observabilidade
Logs
- JSON estruturado no entrypoint, no core do `.so`, nos plugins (via SDK) e no controller
- Campos fixos: `ts`, `level`, `service`, `instance`, `plugin_id` e `request_id` quando houver
- Nenhum segredo ou token em log

Métricas
- Prefixo `n3x_`, expostas em Prometheus
- Sidecar: estado da conexão com o controller, idade da sessão, logins e renovações por resultado, `panics_total` por plugin, plugins desligados automaticamente, tamanho e descartes do buffer, uso de memória por plugin em relação à cota
- Controller (prefixo `n3x_controller_`, detalhe no HLD do `n3x-controller`): chamadas gRPC por método e status, logins por resultado e serviço, logins bloqueados, sessões ativas e estado de cada plugin (`ok`, `degradado`, `falho`)
- Plugin core (prefixo `n3x_n3x_core_`, regra do ADR-022 do `n3x-sdk`, detalhe no ADR-010 do `n3x-plugin-core`): sidecars por estado, divergências, conflitos de identidade, registros, logins, eventos, memória usada e estado degradado
- Observabilidade do tráfego da aplicação está fora da v1

Tracing
- Sem tracing distribuído na v1 (ADR-009)
- Correlação por `request_id` nos metadados gRPC entre sidecar e controller, registrado nos logs dos dois lados

Dashboards e alertas
- Painel do controller: idade da sessão por instância, sidecars por estado, divergência de config e de manifesto, eventos
- Alertas: sidecar sem conexão há mais de N minutos, panics e plugin desligado, falhas de login em sequência, divergência de config ou de manifesto, plugin degradado ou falho (por exemplo, core acima do teto de memória)
- Health do sidecar (`17021`) e do controller (`17091`) usados pelo orquestrador

---

### Riscos arquiteturais e mitigação
#### Matriz de CI cruzada cara demais para uma pessoa manter
- **Probabilidade:** alta
- **Impacto:** CI lenta e alvos sem teste, principalmente `istio-1.31` e arm64
- **Mitigação:**
  - Só amd64 e testes rápidos no PR; arm64, carga e longa duração à noite
  - CI cruzada centralizada em `n3x-sdk/deploy/`
- **Plano de contingência:** adiar o alvo `istio-1.31` e entregar primeiro `envoy-1.40`

#### Plugin core concentra autenticação, interceptor e registro
- **Probabilidade:** média
- **Impacto:** um bug no core corta a comunicação de todos os plugins com o controller; o tráfego segue normalmente
- **Mitigação:**
  - TDD e testes de conformidade dedicados ao login, renovação e interceptor
  - Capacidades privilegiadas restritas ao core pelo SDK
- **Plano de contingência:** modo standalone (sem controller) até o core ser corrigido e redeployado

#### Login como rota pública vira alvo de força bruta ou de DoS de CPU por bcrypt
- **Probabilidade:** média
- **Impacto:** controller saturado e painel lento
- **Mitigação:**
  - Limite de tentativas por IP e por serviço
  - Limite de concorrência de verificações bcrypt
  - Porta gRPC só em rede interna
- **Plano de contingência:** restringir a origem por rede no orquestrador e subir CPU do controller

#### Rajada de logins após restart do controller
- **Probabilidade:** baixa
- **Impacto:** painel vazio ou lento por alguns minutos
- **Mitigação:**
  - Jitter no novo login
  - `session.hmac_key` configurada no core, mantendo as sessões válidas após restart
- **Plano de contingência:** escala vertical do controller

#### Plugin core acima do teto de memória
- **Probabilidade:** baixa
- **Impacto:** usuários de leitura não entram e novas versões de config são recusadas; o tráfego não é afetado (ADR-009 do `n3x-plugin-core`)
- **Mitigação:**
  - Plugin core descarta histórico e eventos antigos antes de se marcar como degradado, e o painel mostra o banner
  - Root entra mesmo assim
  - Métrica e alerta de estado por plugin
- **Plano de contingência:** reiniciar o controller ou subir `memory_limit`; o estado observado se reconstrói pelos sidecars

#### Banco ou migration de um plugin falha na subida
- **Probabilidade:** média
- **Impacto:** controller não sobe (plugin obrigatório) ou plugin sobe desligado (opcional)
- **Mitigação:**
  - Cada plugin com o próprio banco, sem tocar nos dados de outro (ADR-002 do `n3x-controller`)
  - Política de obrigatório e opcional do FR-011
- **Plano de contingência:** rollback da imagem do controller para a versão anterior

#### Isolamento de memória entre plugins contornado
- **Probabilidade:** baixa
- **Impacto:** um plugin lê dados de outro, inclusive do core
- **Mitigação:**
  - Handle de memória preso ao ID do plugin, sem acessor global
  - Análise estática proíbe `unsafe` e `reflect` sobre tipos do SDK
- **Plano de contingência:** remover o plugin da imagem e redeployar

#### Versões desencontradas entre os cinco repositórios
- **Probabilidade:** média
- **Impacto:** falha de build ou painel inconsistente
- **Mitigação:**
  - Catálogo de alvos fixando a combinação suportada
  - `buf breaking`, checagem de SDK no build e alerta de manifesto no painel
- **Plano de contingência:** refazer o build com a mesma lista e versões nos dois lados

Riscos com dono, só referenciados: erro fatal do Go no `.so`, iteração de filtros, negação gRPC e overhead (HLD do `n3x-sidecar`); quebra do Istio e do SDK Go do Envoy (HLD do `n3x-builder`).

---

### ADRs e próximos passos
ADRs associados
- [ADR-001: Controller em instância única, sempre um por ambiente](adr/ADR-001-controller-instancia-unica-por-ambiente.md)
- [ADR-002: Armazenamento efêmero ou Postgres com migrations isoladas por plugin](adr/ADR-002-armazenamento-efemero-ou-postgres.md) (substituído pelo ADR-002 do `n3x-controller`: controller sem armazenamento, cada plugin traz o próprio banco)
- [ADR-003: Login por credencial de serviço e sessão HMAC sem estado](adr/ADR-003-login-credencial-e-sessao-hmac.md) (atualizado em 2026-10-20: TTL e chave HMAC no `n3x-plugin-core`)
- [ADR-004: Capacidades privilegiadas do plugin core](adr/ADR-004-capacidades-privilegiadas-plugin-core.md)
- [ADR-005: Memória isolada por plugin no SDK](adr/ADR-005-memoria-isolada-por-plugin.md)
- [ADR-006: Faixa de portas 17xxx e Istio no comando](adr/ADR-006-portas-17xxx-e-modo-istio.md)
- [ADR-007: Sidecar nativo no Kubernetes](adr/ADR-007-sidecar-nativo-kubernetes.md)
- [ADR-008: Exemplos de deploy e CI cruzada em `n3x-sdk/deploy/`](adr/ADR-008-exemplos-deploy-no-sdk.md)
- [ADR-009: Sem tracing distribuído na v1](adr/ADR-009-sem-tracing-v1.md)
- [ADR-010: Metas não funcionais no nível do sistema](adr/ADR-010-metas-nao-funcionais-nivel-sistema.md)

Decisões pendentes
- Resolvida: TTL da sessão de 1 h e renovação em 80% do prazo (ADR-005 do `n3x-plugin-core`)
- Cota de memória padrão por plugin e API do espaço de memória: HLD do `n3x-sdk`
- Contrato das capacidades novas do core (diretório de usuários, uso do verificador de credencial; gravador de auditoria reservado para a v2) e das APIs de cartões e de estado de plugin: HLD do `n3x-sdk`
- Resolvida: banco do plugin core. Na v1 tudo em memória, com teto e degradação (ADR-001 e ADR-009 do `n3x-plugin-core`); banco real na v2
- Resolvida: formato da config efetiva, mascaramento e hash (ADR-006 do `n3x-plugin-core`)
- Resolvidas: vários hashes por serviço (um só, ADR-005 do `n3x-controller`); isolamento no Postgres (não se aplica, ADR-002 do `n3x-controller`); ferramenta de `ENC[...]` (ADR-003 do `n3x-sidecar`); semver independente por repositório e por imagem (ADR-001 do `n3x-builder`)

Próximos passos
- Escrever o HLD do `n3x-sdk`, depois o do `n3x-plugin-core`, depois sidecar e controller em paralelo e, por fim, o `n3x-builder`
- Criar os exemplos em `n3x-sdk/deploy/` (docker compose, Kubernetes, ECS e Istio) assim que o schema do `n3x.yaml` estiver fechado no HLD do sidecar
- Montar o esqueleto da CI cruzada a partir de `n3x-sdk/deploy/`
