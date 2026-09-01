---
name: rfc-debate
description: Conduz a análise independente e o debate da RFC com os agentes rfc-arquiteto, rfc-dev e rfc-seguranca, produz a ata em docs/debates/RFC-001/ e aplica os ajustes acordados na RFC. Segunda etapa do fluxo de RFC. Use depois do rascunho aprovado, ou ao rodar /rfc-debate.
---

# Debate da RFC

## Prompt base

Leia `docs/prompts/rfc-fluxo.md` por inteiro e siga aquele arquivo: regra de âncora,
taxonomia dos pontos, formato da análise, formato da ata, regra de altura e estilo.
Ele é a fonte única.

## Papel

Você conduz o debate no papel do Tech Lead, autor da RFC. Você responde a cada
ponto levantado e classifica cada um. Você não delega a classificação aos
debatedores: a sugestão deles é insumo, a decisão é sua.

## Pré-condição

`docs/RFC.md` existe e está em `Status: Rascunho`, aprovado pelo usuário. Se não
estiver, pare e diga que falta rodar `rfc-draft`.

## Fase 1: análises independentes

Abra os três agentes com a tool `Agent`, em paralelo, na mesma mensagem:
`rfc-arquiteto`, `rfc-dev` e `rfc-seguranca`. Cada um recebe o mesmo pedido: ler o
prompt base e a RFC, produzir a análise no formato definido e salvá-la em
`docs/debates/RFC-001/analise-<papel>.md`.

Regra dura: diga a cada agente que ele não pode ler a análise dos outros. As três
análises são escritas às cegas. É isso que evita que os três repitam o primeiro que
escreveu, e é a razão de esta fase existir separada do debate.

Quando as três voltarem, leia os três arquivos e monte a lista consolidada de
pontos, sem editar as análises.

## Fase 2: debate

No máximo três rodadas. Continue **sempre** com `SendMessage` para os mesmos três
agentes. Nunca abra um `Agent` novo: um spawn novo perde o histórico e o debatedor
passa a se contradizer entre rodadas.

Em cada rodada:

1. Mande a cada agente os pontos dos outros dois que tocam o eixo dele, mais a sua
   resposta como autor a cada ponto dele: aceito, rejeitado com motivo, ou preciso
   de mais evidência.
2. Peça que sustente ou recue, com evidência. Recuo é resultado legítimo.
3. Rejeitar um ponto é legítimo, mas rejeição sem motivo escrito não é. E ponto
   rejeitado não some: ele vira `DIVERGENTE` na ata.

Encerre antes das três rodadas se nenhum ponto novo com evidência aparecer. Encerre
na terceira mesmo com divergência aberta: divergência que sobrevive a três rodadas é
questão em aberto da RFC, não motivo para uma quarta.

Ao final, todo ponto precisa estar classificado como `ACORDADO`, `DIVERGENTE`,
`EM ABERTO` ou `FORA DE ESCOPO`. Ponto sem classificação impede o encerramento.

## Fase 3: consolidação

1. Escreva `docs/debates/RFC-001/ata.md` no formato do prompt base, incluindo a
   tabela "Decisões a registrar" com os provisórios `ADR-TBD-NN` derivados dos
   pontos `ACORDADO`. Agrupe pontos que são a mesma decisão sob um único `TBD`.
2. Edite `docs/RFC.md`:
   - aplique os ajustes de texto dos pontos `ACORDADO` que não viram ADR
   - leve todo `DIVERGENTE` e todo `EM ABERTO` para "Questões em aberto", nomeando
     as duas posições quando houver divergência
   - preencha "Decisões relacionadas" com a lista de `ADR-TBD-NN` e o título
     provisório de cada
   - troque o status para `Em revisão`
   - respeite a regra de altura ao aplicar os ajustes. Um ponto do debate que só se
     resolve com contrato detalhado vira questão em aberto ou entra na fila do FDD,
     não vira parágrafo de implementação na RFC
3. Mostre ao usuário: quantos pontos por classificação, quais viraram `ADR-TBD` e
   quais divergências ficaram registradas.
4. Pare e peça aprovação antes de `rfc-adr`.

Não chame o `tracker-rastreabilidade` nesta etapa. As análises e a ata são
artefatos de processo, não documentos de design: elas não entram no tracker, e a
RFC só é rastreada depois de fechada.
