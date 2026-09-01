---
name: rfc-adr
description: Registra as decisões acordadas no debate como ADRs em docs/adrs/, escritos pelo agente rfc-arquiteto a partir da ata, com numeração sequencial e checklist das decisões principais. Terceira etapa do fluxo de RFC. Use depois do debate fechado, ou ao rodar /rfc-adr.
---

# Registro das ADRs

## Prompt base

Leia `docs/prompts/rfc-fluxo.md` por inteiro e siga aquele arquivo: regra de âncora,
esqueleto de ADR, ciclo de vida do ADR e estilo. Ele é a fonte única. Esta skill
executa a fase de **abertura** do ciclo de vida; o aceite é do `rfc-close`.

## Pré-condição

`docs/debates/RFC-001/ata.md` existe, com a tabela "Decisões a registrar" e a RFC em
`Status: Em revisão`. Se não existir, pare e diga que falta rodar `rfc-debate`.
Nenhuma ADR nasce fora da ata: decisão que não passou pelo debate não tem por que
existir aqui.

## Papel

Quem escreve as ADRs é o agente `rfc-arquiteto`, que participou do debate e conhece
os pontos. Você coordena: aloca os números, define a fila e valida o resultado.

## Passo 1: fila e numeração

1. Leia a tabela "Decisões a registrar" da ata.
2. Deduplique. Dois pontos de eixos diferentes podem ser a mesma decisão. Uma ADR
   por decisão, não uma por ponto de debate.
3. Confira contra o checklist das decisões principais da reunião, listadas no
   `DESAFIO.md`:
   - Padrão Outbox no MySQL
   - Política de retry com backoff e DLQ
   - Autenticação HMAC-SHA256 com secret por endpoint
   - Garantia at-least-once com `X-Event-Id`
   - Worker em processo separado em polling
   - Reuso dos padrões existentes do projeto
   O conjunto precisa cobrir pelo menos cinco das seis. Se a ata não produziu uma
   delas, isso é buraco do debate: diga ao usuário qual ficou de fora antes de
   seguir, e escreva a ADR ancorada direto na transcrição.
4. O conjunto final tem entre cinco e oito ADRs. Decisão secundária que não couber
   fica registrada na ata e vai para o FDD, não vira ADR de enchimento.
5. Aloque os números você mesmo, em sequência, olhando o que já existe em
   `docs/adrs/`. Os agentes nunca escolhem o próprio número: dois agentes
   escolhendo em paralelo colidem no mesmo `ADR-003`.

Mostre a fila numerada ao usuário antes de escrever.

## Passo 2: escrita

Acione o `rfc-arquiteto` com `SendMessage`, uma ADR por mensagem, em sequência.
Cada mensagem carrega: o número alocado, o nome do arquivo, a decisão, os pontos de
origem na ata e as evidências já levantadas. Escrever em sequência, e não em
paralelo, é o que permite ao agente evitar que duas ADRs digam a mesma coisa com
palavras diferentes.

Exija em cada ADR:

- o esqueleto do prompt base, com todas as seções
- pelo menos uma alternativa considerada, com o motivo do descarte
- consequências positivas e negativas, e o trade-off aceito escrito
- seção `## Status` com o valor `Proposto` e o link de volta para `docs/RFC.md`.
  Nenhuma ADR nasce `Aceito`: a promoção é do `rfc-close`, para o conjunto inteiro
- toda afirmação ancorada em ID do tracker, timestamp ou caminho de arquivo
  verificado

Pelo menos uma ADR do conjunto precisa referenciar arquivos, módulos ou classes
reais do código base. A candidata natural é a de reuso dos padrões existentes.

## Passo 3: validação e rastreabilidade

1. Confira nome de arquivo (`ADR-NNN-titulo-em-kebab-case.md`), numeração sem
   buraco nem repetição, presença das seções obrigatórias (`Status`, `Contexto`,
   `Decisão`, `Alternativas Consideradas`, `Consequências` com os dois lados e
   `Trade-off aceito`), e que todo caminho de código citado existe.
2. Chame o `tracker-rastreabilidade` uma vez por ADR escrita, para acrescentar as
   linhas ao `docs/TRACKER.md`. Este passo não é opcional.
3. Não edite `docs/RFC.md` aqui. Trocar os `ADR-TBD` pelos links reais é trabalho do
   `rfc-close`, que valida os dois lados do link de uma vez.
4. Mostre ao usuário a lista das ADRs criadas e a cobertura do checklist das seis
   decisões.

Rodar esta skill de novo corrige as ADRs existentes. Não crie um segundo arquivo
para uma decisão que já tem ADR. ADR já `Aceito` não é reescrito para mudar a
decisão: abre-se um sucessor e o antigo vira `Substituído por ADR-NNN`, conforme o
ciclo de vida no prompt base.
