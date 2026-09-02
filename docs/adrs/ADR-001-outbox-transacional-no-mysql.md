# ADR-001: Outbox transacional no MySQL dentro do changeStatus

Data: 2026-09-01
RFC de origem: docs/RFC.md
Decisores: Larissa (Tech Lead), Diego (Engenheiro Sênior, time de Plataforma), Bruno (Engenheiro Pleno, time de Pedidos)

## Status

Proposto

## Contexto

Três clientes B2B pedem notificação de mudança de status de pedido em até 10 segundos
(PRD-OBJ-01, PRD-MET-01), e o time não quer provisionar infraestrutura nova para atender
esse pedido (PRD-MET-04). O ponto de integração é o `changeStatus` em
`src/modules/orders/order.service.ts`, que já roda dentro de `this.prisma.$transaction`,
fazendo leitura do pedido, ajuste de `stockQuantity` item a item, update do pedido, insert
de histórico e uma releitura final com relações (`src/modules/orders/order.service.ts`).
É essa transação, já carregada, que passa a ganhar mais uma escrita.

`[TRANSCRICAO 09:06]` registra a explicação do padrão: "quando o status do pedido muda,
dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também
insere uma linha numa tabela tipo webhook_outbox com o evento (...) Garante que se a
transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some
junto. Não tem inconsistência possível." `[TRANSCRICAO 09:40]` e `[TRANSCRICAO 09:41]`
confirmam a integração no `changeStatus` e a função `publishWebhookEvent` recebendo o
client `tx` da transação corrente, no lugar de um repository injetado (PRD-ESC-02).
`[TRANSCRICAO 09:52]` fixa que o payload é gravado como snapshot no momento da inserção,
e não recalculado no envio (PRD-FR-03c).

O debate da RFC-001 (ARQ-05, DEV-02) apontou que a RFC apresentava essa integração como
proposta fechada, mas deixava em aberto se a transação comporta a escrita a mais, sem
tomar nenhuma posição mínima, e que faltava dizer em que ponto exato da transação a
publicação entra. Nenhuma das quatro fontes traz medição de tempo de transação, volume de
pedidos concorrentes ou limite de contenção aceitável: a evidência de que a transação fica
mais pesada é estrutural, pela leitura do código, não quantitativa.

## Decisão

O `changeStatus` publica o evento na `webhook_outbox` dentro da mesma transação SQL que já
executa hoje, chamando `publishWebhookEvent(tx, order, fromStatus, toStatus)` com o client
da transação corrente (PRD-ESC-02, `[TRANSCRICAO 09:41]`). A chamada entra depois do
`tx.order.update` que grava o novo status e antes do fim da transação, para que o payload
gravado como snapshot reflita o pedido já no status de destino (PRD-FR-03c,
`[TRANSCRICAO 09:52]`; posicionamento fixado no debate como ARQ-05/DEV-02).

Havendo webhook ativo do customer inscrito no status de destino, o evento é inserido
(PRD-FR-03b, PRD-FR-03d). Falha na inserção do evento propaga como falha da transação
inteira, derrubando também a mudança de status (PRD-FR-03e, PRD-RISK-06).

Aceita-se, de forma declarada e não quantificada, que essa transação passa a fazer mais uma
leitura de configuração de webhook e mais uma escrita, sobre uma transação que já lê o
pedido, ajusta estoque item a item, atualiza o pedido, insere histórico e relê com
relações. Não há número de fonte para o tamanho aceitável dessa contenção adicional; o
aceite é qualitativo, sujeito a revisão com dados de produção depois do lançamento.

## Alternativas Consideradas

**Disparo síncrono de HTTP dentro da transação de mudança de status.** Descartado porque um
cliente lento no endpoint de destino travaria a transação e, por consequência, a mudança de
status de outros pedidos, e não havia critério sensato para decidir rollback do pedido só
porque a entrega HTTP falhou (PRD-FESC-06). `[TRANSCRICAO 09:04]`: "a transação de mudança
de status hoje já é pesada (...) Se a gente acrescentar um HTTP call no meio disso, qualquer
cliente lento vai travar mudança de status pra outros pedidos", e "se o cliente tiver fora
do ar, o que a gente faz, dá rollback na mudança de status? Não dá."

**Fila dedicada do tipo Redis Streams, publicada fora da transação.** Removeria a escrita
extra da transação de pedidos, mas exigiria provisionar e operar um broker novo, considerado
desproporcional ao tamanho do time (PRD-FESC-07). O minuto `[TRANSCRICAO 09:07]` tem mais de
um falante: nele a alternativa é levantada como algo que exigiria subir mais infraestrutura, e
em seguida derrubada com o argumento de que o time é pequeno e um cluster novo para isso seria
overengineering.

## Consequências

### Positivas

- Garantia forte de consistência entre status e evento: se a transação do pedido commitou,
  o evento existe na outbox; se ela sofreu rollback, o evento não existe (PRD-MET-03,
  `[TRANSCRICAO 09:06]`)
- Nenhuma infraestrutura nova a provisionar ou operar, dentro da restrição organizacional do
  time (PRD-MET-04, PRD-DEP-04)
- O payload como snapshot no momento da inserção evita depender de um estado do pedido que
  pode ter mudado entre a gravação e o envio (PRD-FR-03c)

### Negativas

- Falha na inserção do evento na outbox derruba a mudança de status junto, mesmo quando a
  causa da falha não tem nenhuma relação com o domínio de pedidos (PRD-FR-03e, PRD-RISK-06).
  A mudança de status passa a poder falhar por um motivo alheio ao pedido
- A transação de `changeStatus`, já carregada, ganha mais uma leitura e uma escrita, sem que
  exista medição de fonte sobre o efeito no tempo de lock ou na taxa de conflito sob volume
  concorrente; o aceite dessa contenção é qualitativo, não numérico
- `src/app.ts` precisa ser tocado para fiar a nova dependência na construção do
  `OrderService`, contrariando a leitura inicial de que a alteração ficaria isolada dentro
  do `order.service.ts` (achado do debate, DEV-01)

## Trade-off aceito

Troca-se latência mínima de entrega e isolamento do domínio de pedidos por consistência
forte e ausência de infraestrutura nova: aceita-se que a transação de `changeStatus` fique
mais pesada e possa falhar por causa de uma feature de notificação, em vez de aceitar uma
janela em que o status muda sem que o evento correspondente exista.
