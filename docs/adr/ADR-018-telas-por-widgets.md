# ADR-018: Telas de plugin como árvore de widgets, sem HTML

Status: Aceito
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
