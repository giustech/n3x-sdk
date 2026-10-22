# ADR-018: Telas de plugin como árvore de widgets, sem HTML

Status: Aceito (atualizado em 2026-10-22)
Data: 2026-10-19

## Contexto
O PRD previa telas de plugin como componentes templ. Assim, cada autor escreveria HTML, e o painel perderia a identidade visual, a acessibilidade (WCAG 2.1 AA) e a segurança (CSP e escape) a cada plugin. Expor o `echo.Context` também prenderia o contrato à versão do Echo.

## Decisão
- O plugin monta páginas e cartões como uma árvore de widgets, um dentro do outro, no estilo do Flutter. O renderizador do SDK gera o HTML com a identidade do painel
- Registro: `r.Page(path, papel, func(ctx, ui.Query) (ui.Widget, error))` e `c.Add(id, widget)`. O contrato usa só `net/http` e `context`, nunca o Echo
- Catálogo da v1 (`controller/ui`):
  - Layout: `Page`, `Column`, `Row`, `Grid`, `Section` e `Tabs`
  - Conteúdo: `Text`, `Stat`, `Badge`, `Table`, `Chart` (linha e barra, em SVG gerado no servidor), `Detail`, `Code` (YAML e diff) e `Empty`
  - Interação: `Refresh` (polling htmx), `Link`, `Form` e `Action`, com CSRF automático e papel mínimo
- O renderizador resolve layout, filtros, paginação no servidor (50 linhas por padrão), ordenação, exportação JSON e CSV, fragmentos htmx e acessibilidade
- Não há escape para HTML ou templ próprio na v1. O templ fica só dentro do SDK e do host. Widget novo entra no SDK numa versão minor
- Cartão com prazo de 1 s; se estourar, mostra "indisponível" sem travar os outros

## Justificativa
- Visual, acessibilidade e segurança garantidos num lugar só, e ninguém escreve HTML
- Todo texto é escapado pelo renderizador, sem risco de XSS vindo de plugin
- O Echo pode ser atualizado sem quebrar plugins

## Trade-off
- Uma tela que o catálogo não cobre espera uma minor do SDK
- O catálogo de widgets vira parte do contrato estável (ADR-021)

## Requisitos relacionados
FR-008, FR-011 e FR-013; acessibilidade e segurança do painel.

## Atualização (2026-10-22)
Decisões aprovadas na clarificação das FDDs SDK-01 a SDK-16:
- `Cards.Add(id, papel, fn CardFunc)` substitui `c.Add(id, widget)`. A função é chamada com prazo de 1 s e atualização
- O renderizador escreve HTML direto no SDK (sem templ no SDK; templ só no host). Isso substitui "O templ fica só dentro do SDK e do host"
- O catálogo da v1 ganha `Pipeline` (com `Stage`) e `Code` com texto, YAML, JSON e diff (antes, só YAML e diff)
- `ui.Column` é só layout; a coluna de tabela é `ui.TableColumn`. `Stat.Value` é string. `Table` e `Refresh` exigem `ID`
- `Table.Export` é vazio por padrão; exportar exige root, e `ExportAllowRead` libera a leitura. `ui.External` não existe na v1
- O polling não renova a sessão. Horas em UTC. Listas acima dos limites são truncadas com o aviso `list_truncated`
- Controles de paginação, ordenação e filtro têm IDs estáveis, e há `role=status`/`aria-live`
- `Form` e `Action` têm `AllowRead bool` no widget e o papel em `Routes.Action` (papel zero = root)
- Widget novo continua entrando por minor, mas `Pipeline` e as ampliações acima entraram na própria v1
