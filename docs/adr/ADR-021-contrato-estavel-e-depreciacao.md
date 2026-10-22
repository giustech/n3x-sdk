# ADR-021: Contrato estável desde a v1.0 e política de depreciação

Status: Aceito (atualizado em 2026-10-22)
Data: 2026-10-19

## Contexto
Todos os plugins dependem do SDK. Avaliou-se lançar as APIs ainda não maduras num pacote `x/`, fora da promessa do semver. A escolha foi não ter APIs experimentais.

## Decisão
- Todos os pacotes são estáveis desde a `v1.0.0`; não existe pacote `x/`
- Semver estrito: patch corrige, minor só acrescenta (widget novo, hook novo, capacidade nova) e major quebra
- Depreciação: a API marcada com `// Deprecated:` continua funcionando até a próxima major, por no mínimo 6 meses. O `n3x-plugin build` mostra um aviso, sem falhar
- Cada imagem do builder traz uma versão do SDK, e o build recusa plugin cuja faixa `sdk` do `n3x-plugin.yaml` não a inclua (FR-015)
- O SDK exige a versão de Go das imagens do builder (1.26 ou superior), e o mínimo só sobe em minor
- Antes da `v1.0.0`, pré-releases `v0.x` validadas com as telas do core, o plugin de exemplo e um protótipo de rate limit

## Justificativa
- Um contrato só, sem dúvida sobre o que é seguro usar
- Os plugins não quebram por mudanças em APIs "quase estáveis"

## Trade-off
- Um erro de design descoberto depois da `v1.0.0` só se corrige numa major
- As pré-releases `v0.x` atrasam a `v1.0.0`

## Requisitos relacionados
FR-012, FR-013 e FR-015; compatibilidade nos requisitos não funcionais.

## Atualização (2026-10-22)
Decisões aprovadas na clarificação das FDDs SDK-01 a SDK-16:
- A regra da série `0.x` (quebra permitida entre minors) vale também para a checagem de schema da config do PLUGIN (mesma linha = mesma minor), além da faixa `sdk`
- `chain/` e os demais pacotes de plataforma seguem o semver do módulo, mas ficam fora do contrato de plugin. Isso qualifica "Todos os pacotes são estáveis desde a `v1.0.0`"
- Módulos aninhados (`example/` etc.) têm tags próprias com prefixo
