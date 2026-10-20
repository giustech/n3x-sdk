# ADR-014: Pacote `secret` no SDK, proibido aos plugins

Status: Aceito
Data: 2026-10-19

## Contexto
O pacote que cifra e decifra `ENC[...]` morava em `n3x-sidecar/core/secret`. O controller importava o repositório do sidecar só para usá-lo nos subcomandos `encrypt` e `keygen` e para decifrar as seções dos plugins, o que criava uma dependência controller → sidecar.

## Decisão
- O pacote passa a ser `github.com/giustech/n3x-sdk/secret`, usado pelo entrypoint e pelo `.so` do sidecar, pelo controller e pelo builder
- O formato não muda: `ENC[v1:<base64(nonce || texto cifrado || tag)>]` com AES-256-GCM, AAD `n3x:v1:<service>` e chave de 32 bytes em arquivo montado (contrato do HLD do `n3x-sidecar`)
- Usa só a biblioteca padrão (`crypto/aes`, `crypto/cipher` e `crypto/rand`)
- A análise estática do `n3x-plugin build` recusa código de plugin que importe `secret/`: o plugin recebe a config já decifrada, com os campos sensíveis em `plugin.Secret` (ADR-020)

## Justificativa
- O sidecar e o controller já dependem do SDK, então não entra dependência nova
- Remove a dependência estranha do controller sobre o sidecar

## Trade-off
- O formato `ENC[...]` passa a seguir o semver do SDK
- O HLD do `n3x-sidecar` e o do `n3x-controller` precisam trocar a referência a `core/secret`

## Requisitos relacionados
FR-001, FR-011 e os requisitos de segurança (AES-256-GCM e chave em arquivo).
