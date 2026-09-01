---
name: rfc-draft
description: Escreve o rascunho da RFC do Sistema de Webhooks em docs/RFC.md, no papel do Tech Lead, a partir do PRD, do tracker, da transcrição e do código. Primeira etapa do fluxo de RFC. Use quando o usuário pedir para criar ou rascunhar a RFC, ou ao rodar /rfc-draft.
---

# Rascunho da RFC

## Prompt base

Leia `docs/prompts/rfc-fluxo.md` por inteiro e siga aquele arquivo como instrução
principal: fontes da verdade, regra de âncora, esqueleto da RFC, regra de altura e
estilo. Ele é a fonte única. Não reproduza o conteúdo dele aqui e não improvise uma
versão resumida.

## Papel

Você é o Tech Lead. A RFC é sua, você a assina e vai defendê-la no debate. Escreva
como quem propõe uma solução para ser atacada, não como quem relata o que a reunião
decidiu.

## Antes de escrever

1. Leia `docs/PRD.md` inteiro. Ele já fixou escopo, requisitos e fora de escopo.
2. Leia `docs/TRACKER.md`. Ele é o seu vocabulário de IDs e a origem já verificada
   de cada item do PRD.
3. Leia `TRANSCRICAO.md` inteira. É de lá que saem as alternativas descartadas e as
   questões em aberto, que o PRD não carrega no nível necessário.
4. Leia o código dos pontos de integração citados no PRD, em especial
   `src/modules/orders/order.service.ts`, os erros em `src/shared/errors/`, o
   `requireRole` em `src/middlewares/auth.middleware.ts` e o logger em
   `src/shared/logger/`. Confirme cada caminho antes de citá-lo.
5. Pergunte ao usuário humano o autor e a data da RFC. São metadados do documento,
   não fatos da feature, e não estão na transcrição. Os revisores, sim, saem da
   transcrição: são os participantes da reunião.

## Ao escrever

Siga o esqueleto do prompt base na ordem exata. Além dele:

- **Deixe arestas.** Um rascunho liso não dá o que debater. Onde a reunião não
  fechou, escreva a proposta e diga explicitamente que ela está em aberto, em vez
  de esconder a lacuna com linguagem vaga.
- **Alternativas consideradas** com no mínimo duas opções reais colocadas e
  derrubadas na reunião, cada uma com o trade-off que motivou o descarte. Não
  invente uma alternativa de livro que ninguém levantou.
- **Questões em aberto** com no mínimo dois pontos adiados ou não decididos na
  reunião. Formule cada um como pergunta a ser respondida, com o que depende dela.
- **Decisões relacionadas** fica com o texto "A preencher no fechamento do debate",
  porque os ADRs ainda não existem.
- Respeite a regra de altura. Toda vez que a mão for para o detalhe de contrato,
  pare: aquilo é FDD.
- Marque como `HIPÓTESE`, no próprio texto, qualquer afirmação sem origem nas
  quatro fontes.

## Fechamento

1. Salve em `docs/RFC.md` com `Status: Rascunho`.
2. Releia contra a regra de altura e corte o que for FDD.
3. Mostre ao usuário, em até dez linhas: as alternativas registradas, as questões
   em aberto, e o que você marcou como hipótese.
4. Pare e peça aprovação. Não chame `rfc-debate` por conta própria: debater em cima
   de um rascunho ruim desperdiça o debate inteiro.

Não chame o `tracker-rastreabilidade` nesta etapa. A RFC ainda vai mudar no debate,
e rastrear documento instável só gera linha para refazer.
