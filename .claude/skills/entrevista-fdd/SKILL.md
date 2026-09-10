---
name: entrevista-fdd
description: Conduz a entrevista estruturada que gera o FDD (Feature Design Doc) de uma feature a partir do PRD, da RFC e das ADRs já aceitas, e produz o FDD em Markdown mais um export JSON opcional. Use quando o usuário pedir para criar, montar ou revisar um FDD de feature, ou ao rodar /entrevista-fdd.
---

# Entrevista de FDD

## Prompt base

Leia `docs/prompts/entrevista-fdd.md` por inteiro e siga aquele prompt como sua
instrução principal: papel, princípios de entrevista, as 10 etapas, a estrutura
JSON interna e o esqueleto de saída.

Esse arquivo é a fonte única. Não reproduza o conteúdo dele aqui e não improvise
uma versão resumida. O que segue nesta skill são apenas os modos de execução.

## Pré-requisito

O FDD detalha a implementação do que já foi decidido em `docs/PRD.md`,
`docs/RFC.md` e em `docs/adrs/*.md`. Confirme que os três existem e estão
aceitos antes de começar. Se a RFC ainda tiver ADRs pendentes (`ADR-TBD`) ou
não estiver marcada como aceita, avise o usuário e pergunte se ele quer
prosseguir mesmo assim ou rodar `/rfc-close` primeiro.

## Modos

O argumento da invocação escolhe o modo. Sem argumento, use o modo assistido.

### Modo assistido (padrão)

O usuário é o entrevistado. Siga o prompt base literalmente, incluindo uma
pergunta por vez e a confirmação do resumo ao final de cada etapa.

### Modo automatizado (`/entrevista-fdd auto`)

O subagente `fdd-entrevistado` é o entrevistado. Ele responde ancorado
exclusivamente em `docs/PRD.md`, `docs/RFC.md` e `docs/adrs/*.md`, no papel do
Tech Lead. Ele não lê `TRANSCRICAO.md` nem `src/` diretamente.

Loop de execução:

1. Abra o subagente uma única vez com a tool `Agent`
   (`subagent_type: "fdd-entrevistado"`), enviando a primeira etapa.
2. Nas etapas seguintes, continue **sempre** com `SendMessage` para
   `fdd-entrevistado`. Nunca abra um novo `Agent`: um spawn novo perde o
   histórico e o entrevistado passa a se contradizer entre etapas.
3. O cabeçalho do FDD é a exceção. Responsável, versão e data são metadados do
   documento, não da feature, e não existem no PRD, na RFC nem nas ADRs.
   Pergunte esses ao **usuário humano**, nunca ao `fdd-entrevistado`, que
   devolveria a pergunta ao entrevistador. Produto/nome da feature podem vir
   do PRD ou da RFC normalmente.
4. Agrupe as perguntas **por etapa** do Processo de Entrevista, não uma a uma.
   Numere as perguntas dentro da mensagem e peça respostas na mesma ordem.
5. Ao final de cada etapa, escreva o resumo de 3 a 6 linhas no chat, para o
   usuário humano, e siga adiante sem esperar confirmação do subagente. A
   confirmação automática do próprio entrevistado não tem valor de revisão.
6. Se uma resposta contradisser uma etapa anterior, ou contradisser o que o
   PRD/RFC/ADR já registraram, mande a contradição de volta ao
   `fdd-entrevistado` citando as duas falas e só prossiga depois de resolvida.

### Rastreabilidade

No modo automatizado, o entrevistado devolve a origem de cada afirmação
(`[PRD seção N]`, `[RFC seção "..."]` ou `[ADR-00X]`). Use essas citações para
conferir a fonte enquanto redige, e não como atribuição pronta para o tracker:
quem constrói o `docs/TRACKER.md` é o subagente `tracker-rastreabilidade`, que
vai à `TRANSCRICAO.md` e ao código por conta própria. Essa segunda passagem
independente é o que dá valor ao tracker, e ela se perde se você entregar a
ele a sua própria atribuição.

Quando o entrevistado classificar algo como ADIADO ou DESCARTADO, isso vai
para "Escopo e exclusões" como excluído, nunca para os objetivos ou contratos
técnicos. Quando classificar como NÃO DISCUTIDO, aplique a regra de hipótese
do prompt base e marque como hipótese.

## Fechamento

Depois de gerar o FDD:

1. Salve em `docs/FDD.md`, respeitando o esqueleto exatamente.
2. Chame o subagente `revisor-fdd` sobre o arquivo salvo.
3. Aplique os achados de gravidade bloqueador e ajuste, e mostre ao usuário o
   que ficou de fora e por quê.
4. Chame o subagente `tracker-rastreabilidade` sobre o FDD já corrigido, para
   acrescentar as linhas do FDD ao `docs/TRACKER.md`. Este passo não é
   opcional: sem ele o FDD fica sem o artefato de rastreabilidade, e o
   documento passa a ser a única evidência de si mesmo. O `tracker-rastreabilidade`
   vai à `TRANSCRICAO.md` e ao código diretamente; ele não usa as citações de
   PRD/RFC/ADR que o `fdd-entrevistado` devolveu, e pode legitimamente
   rastrear um item do FDD até a mesma origem que já sustenta o PRD, a RFC ou
   uma ADR.
5. Cruze os dois relatórios. Nenhum item pode aparecer como `INVENTADO` no
   relatório do `revisor-fdd` e como rastreado no tracker. Se aparecer, um dos
   dois errou, e isso precisa ser resolvido antes de fechar.
6. Só então pergunte se ele quer o export JSON.
