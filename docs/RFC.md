# RFC: Notificação outbound de mudança de status de pedidos via transactional outbox

Status: Rascunho
Autor: Larissa (Tech Lead)
Data: 2026-09-01
Revisores: Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança)
PRD relacionado: docs/PRD.md

## TL;DR

Proponho entregar a notificação de mudança de status com transactional outbox no MySQL
que já temos, sem infraestrutura nova (PRD-DEC-01, PRD-MET-04). O `changeStatus` grava o
evento na mesma transação em que muda o status, e um worker em processo Node separado faz
polling da outbox e entrega por HTTP POST assinado com HMAC-SHA256, com retry, backoff e
dead letter queue (PRD-ARQ-01).

O desenho troca latência mínima e complexidade operacional por consistência forte: se a
transação do pedido commitou, o evento existe. O preço está concentrado em três lugares
que este documento quer ver atacados no debate: a transação de pedidos fica mais pesada
e passa a poder falhar por causa de webhook, o worker vira um segundo processo a operar,
e a garantia de ordenação só vale enquanto o worker for único.

## Contexto e problema

Três clientes B2B pediram formalmente notificação de mudança de status e hoje dependem de
polling no `GET /orders` (PRD-PROB-01, PRD-PROB-02). O limiar de tempo real definido pelos
próprios clientes é de 10 segundos (PRD-OBJ-01), e há risco comercial declarado de perda de
conta caso a feature não saia (PRD-OBJ-02).

O fluxo é exclusivamente outbound: a plataforma envia, o cliente recebe, e não existe
webhook de entrada nesta feature (`[TRANSCRICAO 09:02]`, `[TRANSCRICAO 09:03]`). Isso
remove desta RFC todo o problema de autenticar chamadas inbound.

A restrição forte do contexto é organizacional antes de ser técnica: o time é pequeno e não
quer provisionar nem operar infraestrutura nova para esta entrega (PRD-MET-04,
`[TRANSCRICAO 09:07]`). Qualquer alternativa que exija um broker sai do orçamento de
operação disponível, não do orçamento de engenharia.

O ponto de integração é o `changeStatus` em `src/modules/orders/order.service.ts`
(PRD-CTX-08a). Esse método hoje já roda dentro de `this.prisma.$transaction`, e nessa
transação faz a leitura do pedido, o ajuste de `stockQuantity` dos produtos, o update do
pedido, a inserção no histórico e uma releitura final com relações. É uma transação já
carregada, e é ela que vamos onerar.

## Proposta técnica

**Publicação transacional.** O `changeStatus` passa a chamar `publishWebhookEvent`,
recebendo o client da transação corrente, em vez de receber um repository injetado
(PRD-ESC-02, `[TRANSCRICAO 09:41]`). A função consulta os webhooks ativos do customer,
aplica o filtro de status de interesse e, havendo inscrição, insere o evento na outbox
(PRD-ESC-03, PRD-FR-03d). O payload é gravado como snapshot no momento da inserção, não
recalculado no envio, para que o evento reflita o pedido como ele estava quando o status
mudou (PRD-FR-03c, `[TRANSCRICAO 09:52]`).

A consequência dessa escolha é deliberada e desconfortável: falha na inserção do evento
derruba a mudança de status junto (PRD-FR-03e, PRD-RISK-06). Aceitamos porque a alternativa
é ter status alterado sem evento, e o objetivo inteiro da outbox é justamente eliminar essa
janela.

**Worker.** Processo Node separado, com entry point próprio ao lado do `src/server.ts` já
existente e instância própria de PrismaClient (PRD-CTX-09, PRD-ESC-04b). Roda em polling a
cada 2 segundos sobre os eventos pendentes mais antigos, em lotes pequenos (PRD-ESC-04a).
A separação de processo existe para que um restart da API não interrompa o processamento
(PRD-DEC-02, `[TRANSCRICAO 09:11]`). Os dois processos não conversam entre si: sincronizam
apenas pelo banco (PRD-ARQ-11).

**Entrega.** POST HTTP para a URL cadastrada, com timeout de 10 segundos por tentativa
(PRD-ESC-05). Falha de resposta ou timeout reagenda o evento pela progressão de 1 minuto,
5 minutos, 30 minutos, 2 horas e 12 horas, cobrindo cerca de 15 horas, e o esgotamento move
o evento para uma tabela de dead letter separada da outbox (PRD-ESC-06, PRD-ESC-07). O
reprocessamento é manual, por endpoint administrativo restrito ao papel ADMIN, reusando o
`requireRole` já existente em `src/middlewares/auth.middleware.ts`, com log de auditoria de
quem executou (PRD-ESC-08a, PRD-ESC-08b, PRD-ESC-08c).

**Autenticidade.** HMAC-SHA256 sobre o corpo da requisição, com secret gerada pela
plataforma e única por endpoint, nunca uma secret global, para limitar o raio de um
vazamento a um cliente (PRD-ESC-09a, PRD-ESC-09b). A secret é rotacionável pela API, e a
anterior permanece válida por 24 horas (PRD-ESC-10). A URL do endpoint é obrigatoriamente
HTTPS (PRD-ESC-11).

**Garantia de entrega.** At-least-once, com identificador único por evento no header
`X-Event-Id` e deduplicação a cargo do cliente (PRD-ESC-13, PRD-DEC-05). Ordenação garantida
por pedido apenas em regime de worker único, registrada como limitação conhecida
(PRD-DEC-06).

**Reuso.** O módulo entra em `src/modules/webhooks` no padrão dos demais domínios
(PRD-CTX-07), herdando de `AppError` em `src/shared/errors/app-error.ts` com prefixo de
código próprio, e sendo tratado sem alteração pelo error middleware em
`src/middlewares/error.middleware.ts`, que já despacha `AppError`, `ZodError` e erros
conhecidos do Prisma. O logger é o Pino já configurado em `src/shared/logger/index.ts`
(PRD-ESC-16a, PRD-ESC-16b).

## Alternativas consideradas

**Disparo síncrono dentro da transação de mudança de status.** Descartado porque a transação
de `changeStatus` já é pesada e um cliente lento passaria a travar a mudança de status de
outros pedidos, além de não existir critério sensato para decidir rollback do pedido por
falha de entrega (PRD-FESC-06, `[TRANSCRICAO 09:04]`). Trade-off do descarte: perdemos a
entrega imediata e passamos a conviver com latência de disparo.

**Fila dedicada, do tipo Redis Streams, no lugar da outbox.** Resolveria o problema com
menos código próprio, mas exige provisionar e operar infraestrutura nova, considerada
desproporcional para o tamanho do time (PRD-FESC-07, `[TRANSCRICAO 09:07]`). Trade-off do
descarte: escrevemos e mantemos nós mesmos o retry, o backoff e a dead letter que um broker
entregaria pronto.

**Trigger de banco para acionar o worker de forma reativa.** Descartada porque o MySQL não
tem mecanismo nativo de notificação de processo externo equivalente ao do PostgreSQL, e
improvisar isso seria frágil (PRD-FESC-08, `[TRANSCRICAO 09:09]`). Trade-off do descarte:
aceitamos 2 segundos de latência de disparo no pior caso, folgados dentro dos 10 segundos.

**Retry com 3 tentativas em janela curta, e no extremo oposto retry indefinido.** As três
tentativas foram descartadas por matarem o evento durante indisponibilidade planejada do
cliente, com precedente real de 2 horas fora do ar; o retry indefinido foi descartado por
deixar evento pendurado para sempre caso o cliente desapareça (PRD-FESC-09, PRD-FESC-10,
`[TRANSCRICAO 09:15]`, `[TRANSCRICAO 09:16]`). Trade-off do descarte: um evento pode levar
cerca de 15 horas até ser declarado perdido.

**Dead letter como campo de estado na própria outbox.** Descartada em favor de tabela
separada, para manter a leitura da outbox principal limpa e o material de debug isolado
(PRD-FESC-12, `[TRANSCRICAO 09:18]`). Trade-off do descarte: uma tabela a mais e uma
movimentação de linha entre tabelas no caminho de falha.

## Questões em aberto

**Qual secret assina durante o grace period de rotação?** A reunião definiu que a secret
anterior continua válida por 24 horas, mas não disse se o worker assina com a nova, com a
antiga, ou envia mais de uma assinatura (PRD-OPEN-02, PRD-ARQ-15). Depende disso o contrato
de verificação do lado do cliente e o próprio sentido da janela de migração. Minha inclinação
é assinar sempre com a secret nova e manter a antiga válida apenas para não quebrar quem
ainda não migrou, mas isso só faz sentido se enviarmos as duas assinaturas, e essa é
exatamente a decisão que não foi tomada.

**Cinco tentativas ou cinco reagendamentos?** A fala de origem cita 5 tentativas e 5
intervalos, e a ambiguidade não foi resolvida (PRD-OPEN-05, `[TRANSCRICAO 09:17]`). Depende
disso a janela real de retenção do evento e o critério de aceite que valida o backoff.

**Rate limiting de saída.** A reunião registrou apenas observar e decidir depois, sem
classificar como incluso ou descartado (PRD-OPEN-01, `[TRANSCRICAO 09:39]`). Depende disso
o comportamento quando muitos pedidos do mesmo customer mudam de status em sequência, cenário
em que hoje simplesmente disparamos tudo e absorvemos a consequência pelo retry (PRD-RISK-04).

**A transação de `changeStatus` comporta a escrita a mais?** `NOVO`. O
`src/modules/orders/order.service.ts` chama `this.prisma.$transaction` sem opções, o que
mantém os limites padrão do Prisma para transação interativa. A publicação do evento
acrescenta pelo menos uma leitura de configuração de webhook e uma escrita a uma transação
que já faz leitura do pedido, updates de estoque item a item, update do pedido, insert de
histórico e releitura com relações. Depende disso decidir se ajustamos o tempo limite da
transação, se movemos a leitura da configuração para fora dela, ou se aceitamos o risco.

**Criação de pedido não gera evento.** `NOVO`. O método `create` do mesmo service insere o
pedido em `PENDING` e grava o histórico com origem nula sem passar pelo `changeStatus`, de
modo que a proposta como está não notifica o nascimento do pedido. A reunião falou sempre em
mudança de status e o PRD segue essa formulação, então trato isso como fora do escopo, mas
quero a decisão explícita e não por omissão: um cliente inscrito em `PENDING` hoje não
receberia nada.

**O que acontece quando o worker morre?** O PRD registra como não definido pela fonte o
comportamento esperado de restart, health check e alerta do processo do worker. `HIPÓTESE`:
sem isso, a falha do worker é silenciosa e só aparece como reclamação de cliente, porque
nenhum evento vai para a dead letter, todos ficam pendentes. Depende disso saber se a
observabilidade mínima da feature é suficiente.

**Versionamento do payload do evento.** O PRD registra que a estratégia de evolução do
schema do evento não foi definida. Depende disso a possibilidade de mudar o formato depois
que os três clientes estiverem integrados, o que na prática é uma decisão de contrato
público.

## Impacto e riscos

**No código existente.** A alteração invasiva é uma só, dentro do `changeStatus` em
`src/modules/orders/order.service.ts`. O restante da feature é módulo novo em
`src/modules/webhooks` e um entry point novo, sem tocar em `src/app.ts`, no error middleware
ou no logger. O `requireRole` e as classes de erro em `src/shared/errors/` são consumidos
como estão.

**Risco de acoplamento do caminho crítico.** A partir desta entrega, a mudança de status de
pedido passa a poder falhar por um motivo que não é do domínio de pedidos. É a contrapartida
consciente da garantia transacional (PRD-RISK-06), e o código de erro devolvido ao chamador
nesse cenário não foi definido pela fonte.

**Risco de vazamento de secret pelo log.** `NOVO`. O Pino em `src/shared/logger/index.ts`
tem redação configurada para authorization, cookie, password, passwordHash, token e
accessToken. Nenhum desses caminhos cobre a secret do webhook nem a assinatura, e o histórico
de entregas previsto expõe payload e resposta. A reunião citou precedente concreto de cliente
que vazou secret em log de aplicação (`[TRANSCRICAO 09:22]`), o que torna esse ponto mais
concreto do que teórico.

**Risco operacional do segundo processo.** Passamos a ter dois artefatos de execução em
produção com ciclos de deploy independentes (PRD-DEC-02). Some-se a isso a ausência de
definição de restart e alerta do worker, listada acima em aberto.

**Nos clientes já integrados.** Nenhuma quebra: a feature é aditiva e o `GET /orders` segue
funcionando para quem preferir continuar em polling. O impacto real é de contrato e não de
código: cada cliente precisa expor endpoint HTTPS, verificar HMAC e deduplicar por evento
(PRD-DEP-05), e a garantia at-least-once precisa estar documentada antes da integração, sob
pena de integrarem contra uma expectativa errada (PRD-DEP-03).

**Risco de ordenação ao escalar.** A garantia por pedido morre no instante em que subir o
segundo worker, e o particionamento que resolveria isso está deliberadamente adiado
(PRD-DEC-06, PRD-FESC-04).

## Decisões relacionadas

A preencher no fechamento do debate.
