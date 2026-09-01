---
name: entrevista-prd
description: Conduz a entrevista estruturada que gera o PRD de uma feature, em 12 etapas, e produz o PRD em Markdown mais um export JSON opcional. Use quando o usuário pedir para criar, montar ou revisar um PRD de feature, ou ao rodar /entrevista-prd.
---

# Entrevista de PRD

## Prompt base

Leia `docs/prompts/entrevista-prd.md` por inteiro e siga aquele prompt como sua
instrução principal: papel, princípios de entrevista, as 12 etapas, a estrutura
JSON interna, as checagens de consistência, o estilo e o esqueleto de saída.

Esse arquivo é a fonte única. Não reproduza o conteúdo dele aqui e não improvise
uma versão resumida. O que segue nesta skill são apenas os modos de execução.

## Modos

O argumento da invocação escolhe o modo. Sem argumento, use o modo assistido.

### Modo assistido (padrão)

O usuário é o entrevistado. Siga o prompt base literalmente, incluindo uma
pergunta por vez e a confirmação do resumo ao final de cada etapa.

### Modo automatizado (`/entrevista-prd auto`)

O subagente `po-entrevistado` é o entrevistado. Ele responde ancorado em
`TRANSCRICAO.md` e em `src/`.

Loop de execução:

1. Abra o subagente uma única vez com a tool `Agent`
   (`subagent_type: "po-entrevistado"`), enviando a primeira etapa.
2. Nas etapas seguintes, continue **sempre** com `SendMessage` para
   `po-entrevistado`. Nunca abra um novo `Agent`: um spawn novo perde o histórico
   e o entrevistado passa a se contradizer entre etapas.
3. Agrupe as perguntas **por etapa** do Processo de Entrevista, não uma a uma.
   São 12 mensagens em vez de cerca de 40 idas e voltas. Numere as perguntas
   dentro da mensagem e peça respostas na mesma ordem.
4. Ao final de cada etapa, escreva o resumo de 3 a 6 linhas no chat, para o
   usuário humano, e siga adiante sem esperar confirmação do subagente. A
   confirmação automática do próprio entrevistado não tem valor de revisão.
5. Se uma resposta contradisser uma etapa anterior, mande a contradição de volta
   ao `po-entrevistado` citando as duas falas e só prossiga depois de resolvida.

### Rastreabilidade

No modo automatizado, o entrevistado devolve a origem de cada afirmação
(`[TRANSCRICAO 09:02]` ou um caminho em `src/`). Use essas citações para conferir
a fonte enquanto redige, e não como atribuição pronta: elas não trazem o nome de
quem falou, e vários timestamps têm mais de um falante.

Não persista essas origens em arquivo próprio e não as copie para o tracker. Quem
constrói o `docs/TRACKER.md` é o subagente `tracker-rastreabilidade`, que vai à
transcrição por conta própria. Essa segunda passagem independente é o que dá valor
ao tracker, e ela se perde se você entregar a ele a sua própria atribuição.

Quando o entrevistado classificar algo como ADIADO ou DESCARTADO, isso vai para
"Fora de escopo", nunca para requisitos. Quando classificar como NÃO DISCUTIDO,
aplique os Defaults Inteligentes do prompt base e marque como hipótese.

## Fechamento

Depois de gerar o PRD:

1. Salve em `docs/PRD.md`, respeitando o esqueleto exatamente.
2. Chame o subagente `revisor-prd` sobre o arquivo salvo.
3. Aplique os achados de gravidade bloqueador e ajuste, e mostre ao usuário o que
   ficou de fora e por quê.
4. Chame o subagente `tracker-rastreabilidade` sobre o PRD já corrigido, para
   acrescentar as linhas do documento ao `docs/TRACKER.md`. Este passo não é
   opcional: sem ele o pacote fica sem o artefato de rastreabilidade, e o PRD
   passa a ser a única evidência de si mesmo.
5. Cruze os dois relatórios. Nenhum item pode aparecer como `INVENTADO` no
   relatório do revisor e como rastreado no tracker. Se aparecer, um dos dois
   errou, e isso precisa ser resolvido antes de fechar.
6. Só então pergunte se ele quer o export JSON.
