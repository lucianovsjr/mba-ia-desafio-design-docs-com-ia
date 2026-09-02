# RFC: Notificação outbound de mudança de status de pedidos via transactional outbox

Status: Em revisão
Autor: Larissa (Tech Lead)
Data: 2026-09-01
Revisores: Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança)
PRD relacionado: docs/PRD.md

## TL;DR

Proponho entregar a notificação de mudança de status com transactional outbox no MySQL que já
temos, sem infraestrutura nova (PRD-DEC-01, PRD-MET-04). O `changeStatus` grava o evento na mesma
transação em que muda o status, e um worker em processo Node separado faz polling da outbox e
entrega por HTTP POST assinado com HMAC-SHA256, com retry, backoff e dead letter queue
(PRD-ARQ-01).

O desenho troca latência mínima e complexidade operacional por consistência forte: se a transação
do pedido commitou, o evento existe. O preço está em três lugares que este documento não esconde:
a transação de pedidos fica mais pesada e passa a poder falhar por causa de webhook, o worker vira
um segundo processo a operar, e a ordenação por pedido não é imposta por mecanismo técnico nenhum.

## Contexto e problema

Três clientes B2B pediram formalmente notificação de mudança de status e hoje dependem de polling
no `GET /orders` (PRD-PROB-01, PRD-PROB-02). O limiar de tempo real definido pelos próprios
clientes é de 10 segundos (PRD-OBJ-01), e há risco comercial declarado de perda de conta caso a
feature não saia (PRD-OBJ-02).

O fluxo é exclusivamente outbound (`[TRANSCRICAO 09:02]`), o que remove desta RFC todo o problema
de autenticar chamadas inbound. A restrição forte do contexto é organizacional antes de ser
técnica: o time é pequeno e não quer provisionar nem operar infraestrutura nova (PRD-MET-04,
`[TRANSCRICAO 09:07]`), então qualquer alternativa que exija um broker sai do orçamento de
operação disponível, não do de engenharia.

O ponto de integração é o `changeStatus` em `src/modules/orders/order.service.ts` (PRD-CTX-08a),
que já roda dentro de `this.prisma.$transaction` fazendo leitura do pedido, ajuste de
`stockQuantity` item a item, update do pedido, inserção no histórico e releitura final com
relações. É uma transação já carregada, que já escala com o número de itens, e é ela que vamos
onerar.

## Proposta técnica

**Publicação transacional.** O `changeStatus` passa a chamar `publishWebhookEvent`, recebendo o
client da transação corrente, em vez de receber um repository injetado (PRD-ESC-02, `[TRANSCRICAO
09:41]`). A função consulta os webhooks ativos do customer, aplica o filtro de status de interesse
e, havendo inscrição, insere o evento na outbox (PRD-ESC-03, PRD-FR-03d). A publicação ocorre
depois do update de status e antes do fim da transação, e o payload é gravado como snapshot
naquele momento, não recalculado no envio (PRD-FR-03c, `[TRANSCRICAO 09:52]`).

Assumo a posição, e não a deixo em aberto: mantemos a publicação dentro da transação e aceitamos a
contenção adicional. O aceite é qualitativo e declarado como tal, sem medição por trás: a
transação já faz uma consulta e um update por item antes desta feature, e as duas operações
acrescentadas são proporcionalmente pequenas nesse conjunto. Revisamos com dados de produção, e o
eventual ajuste de tempo limite é matéria de FDD.

A consequência dessa escolha é deliberada e desconfortável: falha na inserção do evento derruba a
mudança de status junto (PRD-FR-03e, PRD-RISK-06). Aceitamos porque a alternativa é ter status
alterado sem evento, e o objetivo inteiro da outbox é eliminar essa janela.

**Worker.** Processo Node separado, com entry point próprio ao lado do `src/server.ts` e instância
própria de PrismaClient (PRD-CTX-09, PRD-ESC-04b), em polling a cada 2 segundos sobre os eventos
pendentes mais antigos, em lotes pequenos de tamanho não definido pela fonte (PRD-ESC-04a). A
separação existe para que um restart da API não interrompa o processamento (PRD-DEC-02,
`[TRANSCRICAO 09:11]`), e os dois processos sincronizam apenas pelo banco (PRD-ARQ-11).

O processamento de um lote fica em uma função isolável do loop de polling, e não embutido nele
(PRD-TEST-07). Dentro do lote, a entrega é sequencial entre eventos do mesmo pedido e concorrente
entre pedidos distintos. Isso não é preferência de estilo: a garantia acordada é ordenação por
`order_id` em ordem de `created_at` (PRD-DEC-06, `[TRANSCRICAO 09:12]`), e serializar o lote
inteiro faria um cliente lento consumindo os 10 segundos de timeout atrasar eventos de outros
customers sem ganho de ordenação nenhum, contra PRD-MET-01.

**Entrega.** POST HTTP para a URL cadastrada, com timeout de 10 segundos por tentativa
(PRD-ESC-05). Falha de resposta ou timeout reagenda o evento pela progressão de 1 minuto, 5
minutos, 30 minutos, 2 horas e 12 horas, cobrindo cerca de 15 horas, e o esgotamento move o evento
para uma tabela de dead letter separada da outbox (PRD-ESC-06, PRD-ESC-07). O reprocessamento é
manual, por endpoint administrativo restrito ao papel ADMIN, reusando o `requireRole` já existente
em `src/middlewares/auth.middleware.ts` (PRD-ESC-08a, PRD-ESC-08b). O replay preserva o
identificador de evento original: como a deduplicação é responsabilidade do cliente e se apoia
nesse identificador (PRD-ESC-13), gerar um novo no reprocessamento faria o cliente tratar a
reentrega como evento inédito e reaplicar uma mudança possivelmente já superada.

A trilha de auditoria de quem executou o replay (PRD-ESC-08c) é peça nova e consultável do módulo,
não uma linha solta no log compartilhado: não há model nem utilitário de auditoria no projeto para
reusar. O replay é a única operação da feature com autorização reforçada, e uma trilha diluída no
volume geral de log esvaziaria o único controle que existe.

**Autenticidade.** HMAC-SHA256 sobre o corpo da requisição, com secret gerada pela plataforma e
única por endpoint, nunca uma secret global, para limitar o raio de um vazamento a um cliente
(PRD-ESC-09a, PRD-ESC-09b). A secret é rotacionável pela API, e a anterior permanece válida por 24
horas (PRD-ESC-10). A URL do endpoint é obrigatoriamente HTTPS (PRD-ESC-11).

A secret não é exposta fora do momento de emissão. Ela aparece na resposta da criação e na da
rotação, e nunca nas respostas de leitura da configuração. No log, a redação do Pino configurada
em `src/shared/logger/index.ts` é estendida para cobrir secret e assinatura, que hoje não estão
entre os caminhos redigidos.

**Configuração.** O CRUD é autenticado pelo JWT já existente e não tem restrição por papel nesta
entrega: qualquer usuário autenticado pode cadastrar e alterar a URL de destino de um customer
(PRD-DEC-08, `[TRANSCRICAO 09:37]`). Registro isso na proposta em vez de deixá-lo implícito,
porque omitir uma decisão não é o mesmo que aceitá-la.

**Garantia de entrega.** At-least-once, com identificador único por evento e deduplicação a cargo
do cliente (PRD-ESC-13, PRD-DEC-05).

**Reuso.** O módulo entra em `src/modules/webhooks` no padrão dos demais domínios (PRD-CTX-07). Os
erros estendem as subclasses HTTP de `src/shared/errors/http-errors.ts` quando existe equivalente,
como já fazem `InvalidStatusTransitionError` e `InsufficientStockError`, reservando a herança
direta de `AppError` para o que não tiver, e o error middleware os trata sem alteração. O worker
usa a factory `createLogger()` com serviço próprio, não o singleton da API, para que os logs dos
dois processos sejam distinguíveis (PRD-ESC-16a, PRD-ESC-16b).

## Alternativas consideradas

**Disparo síncrono dentro da transação de mudança de status.** Descartado porque um cliente lento
passaria a travar a mudança de status de outros pedidos, e não existe critério sensato para
decidir rollback do pedido por falha de entrega (PRD-FESC-06, `[TRANSCRICAO 09:04]`). Trade-off:
perdemos a entrega imediata e convivemos com latência de disparo.

**Fila dedicada, do tipo Redis Streams, no lugar da outbox.** Resolveria o problema com menos
código próprio, mas exige infraestrutura nova, desproporcional para o tamanho do time
(PRD-FESC-07, `[TRANSCRICAO 09:07]`). Trade-off: escrevemos e mantemos nós mesmos o retry, o
backoff e a dead letter que um broker entregaria pronto.

**Trigger de banco para acionar o worker de forma reativa.** Descartada porque o MySQL não tem
notificação nativa de processo externo, e improvisar seria frágil (PRD-FESC-08, `[TRANSCRICAO
09:09]`). Trade-off: 2 segundos de latência de disparo no pior caso, folgados dentro dos 10
segundos.

**Worker embutido no mesmo processo da API.** Descartado porque um restart da API interromperia o
processamento pendente (PRD-DEC-02, `[TRANSCRICAO 09:11]`). Trade-off: perdemos o artefato único
de deploy e passamos a operar dois processos independentes.

**Retry com 3 tentativas em janela curta, e no extremo oposto retry indefinido.** Três tentativas
matariam o evento durante indisponibilidade planejada, com precedente real de 2 horas fora do ar;
o retry indefinido deixaria evento pendurado para sempre se o cliente desaparecesse (PRD-FESC-09,
PRD-FESC-10, `[TRANSCRICAO 09:15]`, `[TRANSCRICAO 09:16]`). Trade-off: um evento pode levar cerca
de 15 horas até ser declarado perdido.

**Dead letter como campo de estado na própria outbox.** Descartada em favor de tabela separada,
para manter a leitura da outbox limpa e o material de debug isolado (PRD-FESC-12, `[TRANSCRICAO
09:18]`). Trade-off: uma tabela a mais e uma movimentação de linha no caminho de falha.

## Questões em aberto

**O material assinado deve incluir o timestamp?** Divergência do debate. Segurança sustenta que
assinar só o corpo torna qualquer entrega capturada reproduzível para sempre, e que o agravante é
concreto: com duas instâncias do worker, duas entregas legítimas com timestamps diferentes ficam
indistinguíveis de uma reentrega maliciosa. Como autor, concordo com o risco e recuso resolvê-lo
aqui: o PRD fixou o HMAC sobre o corpo (PRD-ESC-09a) e a RFC não sobrepõe o PRD. Depende disso o
contrato de verificação, e a decisão é do PM com a segurança.

**Quem pode cadastrar o destino, e o destino deve ser validado?** Uma questão só, porque as duas
metades se agravam mutuamente. O CRUD fica aberto a qualquer papel autenticado, permissão que a
própria segurança registrou como provisória (PRD-DEC-08, PRD-FESC-05), e o worker dispara outbound
automaticamente para a URL cadastrada, que hoje só é validada quanto ao esquema e portanto pode
apontar para rede privada ou para o endereço de metadados de nuvem. Esta segunda metade é
`HIPÓTESE`, cenário não discutido na reunião, e ficou divergente: segurança encaminhou como
questão em aberto, arquitetura sustentou como bloqueador por ser composição de duas decisões já
tomadas e não risco de borda. Mantenho aqui porque a mitigação não vem de nenhuma das quatro
fontes.

**Qual secret assina durante o grace period, e o que acontece com uma secret comprometida?** A
reunião definiu que a anterior continua válida por 24 horas, mas não disse qual assina, nem
distinguiu rotação de rotina de rotação motivada por vazamento (PRD-OPEN-02, PRD-ARQ-15). Como
está, uma secret sabidamente comprometida segue válida por um dia depois de o cliente ter pedido a
troca. Depende disso o contrato de verificação e o sentido da janela de migração.

**O que garante que o worker é único?** Nada, hoje. A ordenação por `order_id` (PRD-DEC-06) se
apoia em disciplina operacional: um rolling restart já produz duas instâncias sem ninguém ter
decidido escalar. Qualquer claim atômico na leitura do lote exigiria SQL cru, técnica sem uma
única ocorrência em `src/` hoje. Depende disso decidir se estreamos SQL cru no projeto ou se
aceitamos a garantia como operacional.

**Como testamos o worker e o retry?** O projeto só sabe testar por supertest contra a app real,
sem nock, msw ou fake timers, e nem um retry de 12 horas nem um processo sem Express cabem nesse
padrão. Depende disso o teste de recálculo da assinatura exigido em PRD-TEST-03.

**Cinco tentativas ou cinco reagendamentos?** A fonte cita 5 tentativas e 5 intervalos, e a
ambiguidade não foi resolvida (PRD-OPEN-05, `[TRANSCRICAO 09:17]`). Depende disso a janela real de
retenção e o critério de aceite que valida o backoff.

**Rate limiting de saída.** A reunião registrou apenas observar e decidir depois (PRD-OPEN-01,
`[TRANSCRICAO 09:39]`). Depende disso o comportamento quando muitos pedidos do mesmo customer
mudam de status em sequência (PRD-RISK-04).

**Criação de pedido não gera evento.** `NOVO`. O método `create` insere o pedido em `PENDING` sem
passar pelo `changeStatus`, então um cliente inscrito em `PENDING` não receberia nada. Trato como
fora do escopo, já que a fonte fala sempre em mudança de status, mas quero a decisão explícita e
não por omissão.

**O que acontece quando o worker morre?** O PRD registra restart, health check e alerta como não
definidos pela fonte. `HIPÓTESE`: sem isso, a falha é silenciosa, porque nenhum evento vai para a
dead letter, todos ficam pendentes. Depende disso a visibilidade operacional no dia 1.

**Versionamento do payload do evento.** A estratégia de evolução do schema não foi definida pela
fonte. Depende disso poder mudar o formato depois que os três clientes estiverem integrados, o que
é decisão de contrato público.

## Impacto e riscos

**No código existente.** A alteração invasiva é uma só, no `changeStatus`, mas não é a única
mudança: `buildControllers` em `src/app.ts` monta `OrderService` por injeção manual e precisa ser
alterado para fiar a nova dependência, e `tests/setup.ts` precisa das tabelas novas na limpeza. O
restante é módulo novo mais um entry point novo. O `requireRole` e o error middleware são
consumidos como estão; o logger ganha uma extensão de redação.

**Risco de acoplamento do caminho crítico.** A mudança de status de pedido passa a poder falhar
por um motivo que não é do domínio de pedidos. É a contrapartida consciente da garantia
transacional (PRD-RISK-06), e o código de erro devolvido ao chamador nesse cenário não foi
definido pela fonte.

**Risco de vazamento de secret pelo log.** `NOVO`. A redação do Pino em
`src/shared/logger/index.ts` não alcança a secret do webhook nem a assinatura, e a reunião citou
precedente de cliente que vazou secret em log (`[TRANSCRICAO 09:22]`). A proposta trata disso, mas
o risco só se fecha quando a extensão existir.

**Risco de unicidade do worker.** A ordenação por pedido é convenção operacional, não mecanismo.
Duas instâncias coexistindo, inclusive por acidente de deploy, quebram a ordenação e agravam a
exposição a reentrega. O risco existe desde o dia 1, e some-se a ele a operação de dois artefatos
com ciclos de deploy independentes (PRD-DEC-02) sem definição de restart e alerta do worker.

**Risco de crescimento da outbox.** O arquivamento das linhas entregues ficou fora desta entrega
(PRD-FESC-03), então a tabela cresce indefinidamente e a consulta de pendentes, a cada 2 segundos,
compete com uma tabela cada vez maior.

**Nos clientes já integrados.** Nenhuma quebra: a feature é aditiva e o `GET /orders` segue
funcionando para quem preferir polling. O impacto é de contrato: cada cliente precisa expor
endpoint HTTPS, verificar HMAC e deduplicar por evento (PRD-DEP-05), e a garantia at-least-once
precisa estar documentada antes da integração (PRD-DEP-03).

**Risco de ordenação ao escalar.** A garantia por pedido morre no segundo worker deliberado, e o
particionamento que resolveria isso está adiado (PRD-DEC-06, PRD-FESC-04).

## Decisões relacionadas

Provisórios vindos da ata em `docs/debates/RFC-001/ata.md`, a serem substituídos pelos ADRs no
fechamento:

- `ADR-TBD-01`: publicação do evento dentro da transação do `changeStatus`, depois do update
  de status, com aceite declarado e não quantificado da contenção adicional
- `ADR-TBD-02`: processamento da outbox por função de lote isolável do loop de polling, com
  entrega sequencial por `order_id` e concorrente entre pedidos distintos
- `ADR-TBD-03`: replay de item da dead letter preserva o identificador de evento original
- `ADR-TBD-04`: secret do webhook não exposta fora do momento de emissão, com redação no log
  e ausência nas respostas de leitura
- `ADR-TBD-05`: trilha de auditoria do replay como peça própria do módulo, consultável, sem
  exigência de retenção
