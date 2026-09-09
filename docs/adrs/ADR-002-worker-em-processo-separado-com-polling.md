# ADR-002: Worker em processo separado com polling da outbox

Data: 2026-09-01
RFC de origem: docs/RFC.md
Decisores: Larissa (Tech Lead), Diego (Engenheiro Sênior, time de Plataforma), Bruno (Engenheiro Pleno, time de Pedidos)

## Status

Aceito

## Contexto

A ADR-001 fixa que o evento de mudança de status é gravado na `webhook_outbox` dentro da
transação do `changeStatus`. Falta decidir quem lê essa tabela e entrega o evento ao
cliente. O time não quer subir infraestrutura nova (PRD-MET-04), e o limiar de entrega
combinado com os clientes é menor que 10 segundos entre o commit da mudança de status e a
chegada da requisição (PRD-MET-01).

O minuto `[TRANSCRICAO 09:11]` tem mais de um falante e registra duas coisas distintas. A
primeira é a exigência de processo separado: "o worker tem que rodar como processo separado,
não dentro da mesma instância da API. Senão se a API reinicia, perde o worker". A segunda,
de outro participante no mesmo minuto, é a proposta de uma entry point nova no projeto, nos
moldes da que já existe para a API, com um script próprio para executá-la. O arquivo
`src/server.ts` existe hoje como precedente de entry point único do processo da API. O caminho
sugerido na fala para o worker não existe no repositório e é artefato a criar, assim como o
script correspondente, ausente hoje do `package.json`; por isso ele não é citado aqui como
caminho de código verificado.

`[TRANSCRICAO 09:09]` fixa o mecanismo de leitura: "Polling em loop. A cada 2 segundos,
busca os eventos pendentes mais antigos, processa, marca" (PRD-ESC-04a, PRD-DEC-03), com o
motivo de não haver reatividade nativa no MySQL equivalente ao NOTIFY/LISTEN do Postgres
(PRD-FESC-08). `[TRANSCRICAO 09:30]` fixa PrismaClient próprio por processo: "Separado.
PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é
outro processo Node" (PRD-ESC-04b, PRD-ARQ-11).

`[TRANSCRICAO 09:12]` fixa a garantia de ordenação assumida no desenho: "Se a gente tem um
único worker rodando, ele processa em ordem de created_at do outbox. Aí o cliente recebe em
ordem (...) single-worker e ordering implícita por order_id" (PRD-DEC-06).

O debate da RFC-001 (ARQ-03, DEV-06) mostrou que a RFC não dizia se o processamento de um
lote é sequencial ou concorrente, e que essa mesma fronteira é o que torna o worker
testável (PRD-TEST-07 antecipa expor "uma função de processamento de um lote, testável
isoladamente", marcado como inferência de arquitetura, não decisão da fonte). Serializar o
lote inteiro resolveria a ordenação, mas faria um cliente lento consumindo os 10 segundos de
timeout atrasar eventos de outros customers na mesma leva, sem ganho de ordenação, já que a
garantia acordada em PRD-DEC-06 é por `order_id`, não global. O tamanho do lote não foi
definido pela fonte (PRD-ESC-04a fala apenas em "lotes pequenos") e este ADR não inventa um
número.

## Decisão

O worker roda como processo Node separado do processo da API, com entry point próprio ao
lado de `src/server.ts`, e instância própria de `PrismaClient` conectada ao mesmo banco
(PRD-CTX-09, PRD-ESC-04b, PRD-ARQ-11). Os dois processos não se comunicam diretamente: a
única sincronização entre eles é o estado gravado no MySQL (PRD-NFR-COMPAT-03).

O worker consulta em loop, a cada 2 segundos, os eventos pendentes mais antigos, em lotes
pequenos (PRD-ESC-04a, PRD-DEC-03). O processamento de um lote fica isolado numa função
própria, separada do loop de polling, o que a torna testável fora do laço contínuo
(PRD-TEST-07). Dentro de um lote, a entrega respeita duas regras: entre eventos do mesmo
`order_id`, a entrega é sequencial, preservando a ordem de `created_at` da outbox
(PRD-DEC-06, `[TRANSCRICAO 09:12]`); entre eventos de `order_id` distintos, a entrega pode
ser concorrente, porque nenhuma das quatro fontes garante ordenação entre pedidos diferentes
e serializar tudo penalizaria clientes sem relação com o evento lento (PRD-MET-01).

## Alternativas Consideradas

**Worker embutido no mesmo processo da API.** Descartado porque um reinício da API
interromperia o processamento dos eventos pendentes, já que o ciclo de vida do worker
ficaria amarrado ao ciclo de vida do processo HTTP (PRD-DEC-02). `[TRANSCRICAO 09:11]`: "o
worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se
a API reinicia, perde o worker."

**Trigger de banco para acionamento reativo do worker.** Descartada porque o MySQL não
oferece mecanismo nativo de notificação de processo externo equivalente ao NOTIFY/LISTEN do
Postgres, e um trigger só executa SQL dentro do próprio banco, não avisa um processo de fora
(PRD-FESC-08). `[TRANSCRICAO 09:09]`: "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do
Postgres. Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só
executa SQL. Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo
ou bater num endpoint, fica esquisito."

## Consequências

### Positivas

- O processamento de eventos sobrevive a um restart ou deploy da API, porque roda em ciclo
  de vida próprio (PRD-DEC-02, `[TRANSCRICAO 09:11]`)
- Polling de 2 segundos cobre com folga o limiar de 10 segundos exigido pelos clientes, sem
  exigir nenhum mecanismo reativo novo no MySQL (PRD-DEC-03, PRD-MET-01)
- A fronteira entre o loop de polling e o processamento de um lote, isolada como função
  própria, é o que torna o worker testável sem exercitar o laço contínuo, e é a mesma
  fronteira que expressa a regra de sequencial-por-pedido e concorrente-entre-pedidos
  (PRD-TEST-07)

### Negativas

- Passa a existir um segundo artefato de execução em produção, com ciclo de deploy próprio,
  entry point próprio e instância própria de `PrismaClient`, tudo isso a operar além da API
  (PRD-DEC-02, PRD-ESC-04b)
- Nenhum mecanismo técnico impede duas instâncias do worker rodando ao mesmo tempo. A
  garantia de ordenação por `order_id` (PRD-DEC-06) depende de disciplina operacional, não
  de exclusão mútua: um claim atômico dos eventos pendentes exigiria algo como
  `UPDATE ... ORDER BY ... LIMIT` ou `SELECT ... FOR UPDATE SKIP LOCKED`, e não há
  precedente de SQL cru em `src/`, que hoje só usa o Prisma Client. Fixar esse mecanismo
  neste ADR seria decisão sem origem nas quatro fontes; permanece como questão em aberto da
  RFC, não decidida aqui
- O mecanismo de teste do worker e do ciclo de retry continua em aberto. Este ADR fixa a
  fronteira que torna o worker testável, mas não escolhe a ferramenta de transporte ou de
  simulação de tempo a usar nos testes; isso segue como questão em aberto da RFC

## Trade-off aceito

Troca-se simplicidade de um único processo por continuidade de processamento: aceita-se
operar um segundo artefato em produção, sem garantia técnica de instância única, em troca de
o worker sobreviver a reinícios da API e de a latência de disparo ficar limitada a poucos
segundos sem depender de infraestrutura reativa nova.
