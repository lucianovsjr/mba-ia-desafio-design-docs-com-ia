# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante fica em um arquivo próprio, no formato MADR
definido em `docs/prompts/rfc-fluxo.md`.

## Nome do arquivo

`ADR-NNN-titulo-em-kebab-case.md`, com `NNN` sequencial e sem buraco.
Exemplo: `ADR-001-outbox-no-mysql.md`.

## Seções obrigatórias

`Status`, `Contexto`, `Decisão`, `Alternativas Consideradas`, `Consequências`
(positivas e negativas) e `Trade-off aceito`. O esqueleto completo está em
`docs/prompts/rfc-fluxo.md`, seção "Esqueleto de ADR", que é a fonte única.

## Ciclo de vida

Um ADR nasce de uma decisão registrada na ata do debate em `docs/debates/`, é
escrito pela skill `rfc-adr` e é aceito no fechamento pela skill `rfc-close`.
O fluxo completo, da abertura ao aceite, está em `docs/prompts/rfc-fluxo.md`,
seção "Ciclo de vida de um ADR".
