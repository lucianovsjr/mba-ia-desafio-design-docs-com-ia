---
name: rfc-close
description: Fecha a RFC: troca os ADR-TBD pelos links reais, valida a coerência entre RFC, ADRs, ata e PRD com o agente rfc-revisor, roda o tracker e marca a RFC como Aceita. Quarta e última etapa do fluxo de RFC. Use depois das ADRs escritas, ou ao rodar /rfc-close.
---

# Fechamento da RFC

## Prompt base

Leia `docs/prompts/rfc-fluxo.md` por inteiro e siga aquele arquivo: regra de âncora,
esqueleto da RFC, ciclo de vida do ADR, regra de altura e estilo. Ele é a fonte
única. Esta skill executa a fase de **aceite** do ciclo de vida do ADR.

## Pré-condição

`docs/RFC.md` em `Status: Em revisão`, `docs/debates/RFC-001/ata.md` escrita e as
ADRs presentes em `docs/adrs/`, em `Status: Proposto`. Se faltar alguma, pare e diga
qual etapa falta.

## Passo 1: fechar os links

1. Troque cada `ADR-TBD-NN` da seção "Decisões relacionadas" pelo link real do
   arquivo, com o título da decisão. Nenhum `TBD` pode sobrar em lugar nenhum do
   documento.
2. Confira o link nos dois sentidos: cada ADR citada existe como arquivo, e cada
   ADR aponta de volta para `docs/RFC.md`. Link de mão única é defeito.
3. Confirme que todo `DIVERGENTE` e todo `EM ABERTO` da ata está em "Questões em
   aberto", e que nada classificado como `FORA DE ESCOPO` entrou na RFC.
4. Mantenha o mínimo do desafio: duas alternativas com trade-off, duas questões em
   aberto e pelo menos dois links de ADR.

## Passo 2: revisão

Chame o `rfc-revisor` sobre `docs/RFC.md` e sobre `docs/adrs/`.

Aplique todos os achados de gravidade `bloqueador`. Para os de `ajuste` e `nota`,
aplique o que for correto e mostre ao usuário o que ficou de fora e por quê.

Se o revisor apontar `INVENTADO` em algum item, resolva antes de seguir: ou você
encontra a origem nas quatro fontes, ou o item sai do documento. Não existe terceira
saída.

## Passo 3: rastreabilidade e cruzamento

1. Chame o `tracker-rastreabilidade` sobre a `docs/RFC.md` já corrigida, para
   acrescentar as linhas do documento ao `docs/TRACKER.md`.
2. Cruze os dois relatórios. Nenhum item pode aparecer como `INVENTADO` no relatório
   do revisor e como rastreado no tracker. Se aparecer, um dos dois errou, e isso
   precisa ser resolvido antes de fechar. O cruzamento só vale porque o revisor e o
   tracker leem a fonte de forma independente: não entregue ao tracker a atribuição
   que o revisor já fez.

## Passo 4: fechar

1. Confira, ADR a ADR, as cinco condições de aceite do ciclo de vida: revisor sem
   `bloqueador` aberto, nenhum item `INVENTADO`, link de mão dupla sem `TBD`, todo
   caminho de código citado existente e linhas já no `docs/TRACKER.md`. ADR que
   falhar em qualquer uma continua `Proposto`, e a RFC não fecha.
2. Cumpridas as cinco em todo o conjunto, troque `Proposto` por `Aceito` na seção
   `## Status` de cada ADR, e só então troque o status da RFC para `Aceita`.
3. Mostre ao usuário o resumo final: alternativas, questões em aberto, ADRs
   linkadas, achados aplicados, achados recusados e as linhas acrescentadas ao
   tracker.
4. Diga o que ficou pendente para o FDD, a partir dos pontos do debate que foram
   adiados por altura. Não escreva o FDD aqui.

Rodar esta skill de novo revalida e corrige. Não duplique linha no tracker nem
seção na RFC.
