### PRD: Order Management System (OMS) - Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-08-31
Responsável: Marcos (Product Manager)

---

### Resumo

Três clientes B2B integrados ao OMS pediram formalmente para serem notificados quando o status de seus pedidos muda. Hoje eles descobrem essa mudança fazendo polling no `GET /orders`, o que torna a integração deles lenta e cara. Esta feature entrega notificação outbound: quando o status de um pedido muda, o OMS envia um HTTP POST assinado para o endpoint HTTPS cadastrado pelo cliente, usando transactional outbox no MySQL já existente e um worker em processo separado, sem introduzir nenhuma infraestrutura nova.

Objetivo de negócio
- Eliminar a dependência de polling caro e lento no `GET /orders` por parte dos clientes B2B integradores, entregando a notificação de mudança de status dentro do limiar de 10 segundos que os próprios clientes definiram como tempo real. A motivação é comercial e concreta: a Atlas Comercial sinalizou risco de migrar para um concorrente caso a feature não seja entregue.

---

### Contexto e problema

Público-alvo
- Clientes B2B integradores da plataforma que consomem a API de pedidos, com os casos citados de Atlas Comercial, MaxDistribuição e Nova Cargo
- Operador interno com papel ADMIN, responsável por reprocessar manualmente eventos que falharam de forma definitiva

Cenários de uso chave
- O cliente cadastra um endpoint de webhook e passa a receber notificação a cada mudança de status de pedido, em vez de fazer polling
- O cliente restringe a notificação apenas aos status que lhe interessam, por exemplo SHIPPED e DELIVERED
- O cliente consulta o histórico de entregas dos webhooks recebidos para auditar sua própria integração
- O operador ADMIN reprocessa manualmente um evento que caiu na dead letter queue

Onde essa feature será implantada
- Sistema existente. A feature entra como um novo módulo `src/modules/webhooks`, seguindo o padrão já estabelecido nos demais domínios da codebase, com controller, service, repository, routes e schemas. A alteração no código existente ocorre dentro do `changeStatus` em `src/modules/orders/order.service.ts`, que hoje atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`. O worker de entrega roda como processo separado, com entry point próprio ao lado do `src/server.ts` já existente.

Problemas priorizados
- O polling no `GET /orders` deixa a integração dos clientes lenta, porque não há aviso de mudança e o cliente só descobre no próximo ciclo de consulta. Prioridade alta, com impacto direto em retenção de contrato, dado o risco declarado de perda da Atlas Comercial
- O polling é caro para o cliente, que precisa emitir requisições repetidas sem garantia de que algo mudou. Prioridade alta, pelo mesmo motivo comercial
- Hipótese de priorização. A reunião não classificou formalmente os problemas em alta, média e baixa. A classificação acima é hipótese derivada do fato de as duas dores serem o motivador direto do pedido formal dos três clientes
- Não existe número medido de custo, volume de requisições de polling ou tempo perdido. A fonte só traz o impacto qualitativo e o risco comercial

---

### Objetivos e métricas

| Objetivo                                                               | Métrica                                                         | Meta                      |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------- |
| Notificar o cliente da mudança de status dentro do limiar que ele considera tempo real | Tempo entre o commit da mudança de status e a chegada da requisição HTTP no endpoint do cliente | Menor que 10 segundos |
| Sustentar indisponibilidade temporária do endpoint do cliente sem descartar o evento | Número de tentativas de entrega e janela total de backoff | 5 tentativas cobrindo cerca de 15 horas |
| Garantir que nenhuma mudança de status elegível ocorra sem evento correspondente gravado | Proporção entre eventos gravados na outbox e mudanças de status com webhook inscrito | 100 por cento, garantido por transação |
| Entregar a feature sem provisionar infraestrutura nova | Número de novos componentes de infraestrutura introduzidos | Zero |

Observação sobre as métricas
- Não existe métrica de produto definida pela fonte. A reunião não estabeleceu taxa de adoção pelos clientes, redução percentual de chamadas de polling, taxa de entrega bem sucedida ou qualquer indicador de acompanhamento após o lançamento. As metas acima são parâmetros de projeto usados como critério verificável, e não substituem um plano de mensuração de sucesso, que continua em aberto.

---

### Escopo

Incluso
- Tabela `webhook_outbox` em MySQL, populada dentro da mesma transação SQL do `changeStatus`
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada pelo `order.service.ts` recebendo o client da transação corrente
- Filtro de status de interesse por webhook, aplicado no momento da inserção na outbox
- Worker em processo separado, com PrismaClient próprio, em polling a cada 2 segundos sobre os eventos pendentes mais antigos
- Montagem e envio do payload JSON com os headers padronizados, com timeout de 10 segundos por requisição
- Retry com backoff exponencial de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, totalizando 5 tentativas
- Tabela `webhook_dead_letter` separada, com payload, motivo da falha e timestamp
- Endpoint administrativo de replay de item da dead letter queue, restrito ao papel ADMIN e com log de auditoria de quem executou
- Assinatura HMAC-SHA256 do corpo da requisição, com secret única por endpoint gerada pela plataforma
- Rotação de secret pela API, com a secret anterior permanecendo válida por 24 horas
- Validação de URL obrigatoriamente HTTPS via schema Zod
- Limite de payload de 64 KB, com erro quando ultrapassado
- Entrega at-least-once com `X-Event-Id` em UUID para deduplicação pelo cliente
- CRUD de configuração de webhook, autenticado com o JWT já existente do sistema
- Consulta dos últimos 100 registros de entrega de um webhook, com status, payload, resposta e tempo de resposta
- Módulo `src/modules/webhooks` no padrão da codebase, reusando `AppError`, logger Pino, error middleware e prefixo de erro `WEBHOOK_`

Fora de escopo
- Notificação por e-mail ao cliente quando o webhook dele falha repetidamente. Adiado para fase seguinte, após medir o impacto desta entrega
- Dashboard visual para o cliente acompanhar seus webhooks. Adiado por ser projeto separado do time de frontend
- Arquivamento das linhas já entregues da outbox após cerca de 30 dias. Adiado, reconhecido como necessário mas fora desta entrega
- Escala para múltiplos workers em paralelo, com particionamento por `order_id` ou lock pessimista. Adiado e registrado como limitação conhecida de ordenação
- Endurecimento das permissões do CRUD de configuração de webhook, hoje aberto a qualquer papel autenticado. Adiado explicitamente
- Webhook disparado de forma síncrona dentro da transação de mudança de status. Descartado por travar a mudança de status de outros pedidos quando o cliente estiver lento e por não haver critério sensato de rollback
- Fila dedicada do tipo Redis Streams no lugar da outbox. Descartada por exigir infraestrutura nova, considerada overengineering para o tamanho do time
- Trigger de banco para acionar o worker de forma reativa. Descartada porque o MySQL não oferece mecanismo nativo de notificação de processo externo
- Retry com 3 tentativas em janela curta. Descartado por matar o evento durante indisponibilidade planejada do cliente
- Retry indefinido. Descartado por deixar evento pendurado para sempre caso o cliente desapareça
- Truncamento do payload acima de 64 KB. Descartado em favor de erro explícito
- Dead letter representada como campo de estado na própria outbox. Descartada em favor de tabela separada

Ponto em aberto, sem decisão de dentro ou fora
- Rate limiting do envio de webhooks ao cliente quando muitos eventos do mesmo customer disparam em pouco tempo. A reunião registrou apenas observar e decidir depois, sem classificar como incluso, adiado ou descartado
- Qual secret o worker usa para assinar durante as 24 horas de grace period da rotação, ou se envia mais de uma assinatura. A fonte define apenas que a secret anterior continua válida nesse período
- Nome definitivo da tabela de configuração de webhook e estrutura de persistência do histórico de entregas
- Código e mensagem de erro devolvidos ao chamador do `changeStatus` quando a inserção na outbox falha e a transação sofre rollback
- Se os cinco intervalos de backoff correspondem a cinco tentativas com quatro esperas entre elas, ou a uma tentativa inicial seguida de cinco reagendamentos. A fala de origem cita 5 tentativas e 5 intervalos, e a ambiguidade não foi resolvida na reunião

---

### Requisitos funcionais

Observação sobre prioridade
- A fonte não classificou formalmente a prioridade de nenhum requisito funcional. As prioridades declaradas abaixo são hipótese, derivadas do papel de cada requisito no fluxo principal de entrega e da ordem de execução discutida na reunião.

#### RF-01 Cadastro e gestão de configuração de webhook
Permite que um usuário autenticado cadastre, edite, liste e remova os endpoints de webhook de um customer, incluindo os status de pedido que deseja receber.

**Fluxo principal**
- O usuário autentica com o JWT já existente do sistema
- Envia `POST /webhooks` com a URL do endpoint, a lista de status de interesse e o `customer_id` no corpo da requisição
- O serviço valida a URL como HTTPS por schema Zod
- O serviço gera a secret do endpoint, que não é enviada pelo cliente, e a devolve na resposta da criação
- `GET /webhooks` lista os webhooks cadastrados de um customer
- `PATCH /webhooks/:id` altera URL, lista de status filtrados e estado ativo
- `DELETE /webhooks/:id` remove o webhook

**Fluxos alternativos e exceções**
- URL cadastrada com esquema `http` é recusada na validação do schema
- Operação sobre webhook inexistente retorna erro de recurso não encontrado, seguindo o padrão `NotFoundError` já existente
- O CRUD aceita qualquer papel autenticado nesta entrega, sem restrição por papel

**Erros previstos**
- `WEBHOOK_NOT_FOUND` quando o webhook informado não existe
- `WEBHOOK_INVALID_URL` quando a URL não é HTTPS ou não passa na validação
- `WEBHOOK_SECRET_REQUIRED` nas operações que dependem de secret

**Prioridade:** alta

---

#### RF-02 Rotação de secret do webhook
Permite que o cliente solicite pela API uma nova secret para um endpoint cadastrado, mantendo a secret anterior válida durante um período de transição.

**Fluxo principal**
- O cliente chama o endpoint de rotação de secret do webhook
- A plataforma gera uma nova secret para aquele endpoint
- A secret anterior permanece válida em paralelo por 24 horas, para o cliente migrar seus sistemas
- A nova secret é devolvida ao cliente na resposta
- Após 24 horas, a secret anterior deixa de ser válida

**Fluxos alternativos e exceções**
- Rotação sobre webhook inexistente retorna erro de recurso não encontrado
- Qual das duas secrets é usada para assinar durante a janela de 24 horas não foi definida pela fonte e permanece em aberto

**Erros previstos**
- `WEBHOOK_NOT_FOUND` quando o webhook informado não existe
- `WEBHOOK_SECRET_REQUIRED` quando a operação depende de uma secret ativa inexistente

**Prioridade:** alta

---

#### RF-03 Publicação transacional do evento na outbox
Ao mudar o status de um pedido, o sistema insere o evento correspondente na `webhook_outbox` dentro da mesma transação SQL da mudança de status.

**Fluxo principal**
- O `changeStatus` do `order.service.ts` abre a transação, como já faz hoje
- Dentro da transação são executados o update em `orders`, o insert em `order_status_history` e o ajuste de `stock_quantity` já existentes
- O service chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` passando o mesmo client de transação
- A função verifica se algum webhook ativo do customer está inscrito no status de destino
- Havendo inscrição, o payload é renderizado no momento da inserção, como snapshot, e gravado na outbox com id em UUID
- A transação comita, persistindo mudança de status e evento em conjunto

**Fluxos alternativos e exceções**
- Se nenhum webhook do customer estiver inscrito no status de destino, nada é inserido na outbox, e isso é comportamento esperado, não erro
- Se a inserção na outbox falhar, toda a transação sofre rollback, incluindo a mudança de status, para que não exista status alterado sem evento gravado
- O payload não é recalculado no momento do envio, para que o evento reflita o estado do pedido no instante da mudança

**Erros previstos**
- Falha de inserção propaga como falha da transação e é tratada pelo error middleware já existente. O código de erro específico devolvido ao chamador não foi definido pela fonte

**Prioridade:** alta

---

#### RF-04 Entrega do webhook pelo worker
Processo separado que consulta periodicamente a outbox, monta a requisição e entrega o evento ao endpoint cadastrado pelo cliente.

**Fluxo principal**
- O worker roda em loop e, a cada 2 segundos, consulta os eventos pendentes ordenados por `created_at`, em lotes pequenos
- Para cada evento, usa o payload já renderizado na inserção, contendo `event_id`, `event_type` com o valor `order.status_changed`, `timestamp` em ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, sem a lista de itens
- Monta os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`
- Envia a requisição POST para a URL cadastrada, com timeout de 10 segundos
- Em caso de resposta de sucesso, marca o evento como entregue na outbox
- Registra a entrega para consulta posterior pelo cliente

**Fluxos alternativos e exceções**
- Timeout de 10 segundos sem resposta é tratado como falha e aciona o fluxo de retry
- Resposta de erro do endpoint do cliente também é tratada como falha e aciona o fluxo de retry
- Payload acima de 64 KB não é enviado e resulta em erro registrado, sem truncamento
- A entrega é at-least-once, então o mesmo evento pode chegar mais de uma vez ao cliente, que deduplica pelo `X-Event-Id`

**Erros previstos**
- Falha de rede ou timeout na chamada ao endpoint do cliente, tratada internamente pelo worker, que não responde a requisição externa
- Excedente de tamanho de payload acima do limite de 64 KB

**Prioridade:** alta

---

#### RF-05 Retry com backoff exponencial e dead letter queue
Em caso de falha de entrega, o worker reagenda o envio com intervalos crescentes e, ao esgotar as tentativas, move o evento para a dead letter queue.

**Fluxo principal**
- Na falha de entrega, o worker registra a tentativa
- O worker calcula o horário da próxima tentativa seguindo a progressão de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas
- O ciclo se repete até o total de 5 tentativas
- Falhando a quinta tentativa, o evento é movido para a tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp

**Fluxos alternativos e exceções**
- Se o endpoint do cliente voltar a responder com sucesso antes de esgotar as tentativas, o evento é marcado como entregue e o ciclo de retry é encerrado

**Erros previstos**
- Falha permanente de entrega após a quinta tentativa, registrada como estado interno na dead letter queue e não como resposta HTTP a um chamador

**Prioridade:** alta

---

#### RF-06 Assinatura HMAC-SHA256 do payload
Toda requisição enviada ao cliente é assinada com HMAC-SHA256 usando a secret exclusiva daquele endpoint, permitindo que o cliente verifique a origem e a integridade do evento.

**Fluxo principal**
- No cadastro do webhook, a plataforma gera uma secret única para aquele endpoint, e não uma secret global compartilhada
- No momento do envio, o worker calcula o HMAC-SHA256 sobre o corpo da requisição usando a secret ativa
- O resultado é enviado no header `X-Signature`, acompanhado de `X-Timestamp` e `X-Webhook-Id`
- O cliente recalcula a assinatura do seu lado e compara antes de processar o evento

**Fluxos alternativos e exceções**
- Durante as 24 horas de grace period após uma rotação, a secret anterior continua válida. Qual secret é usada para assinar nesse intervalo não foi definida pela fonte

**Erros previstos**
- `WEBHOOK_SECRET_REQUIRED` quando não há secret ativa disponível para o endpoint

**Prioridade:** alta

---

#### RF-07 Replay administrativo de item da dead letter queue
Endpoint administrativo que recoloca como pendente na outbox um evento que falhou de forma definitiva.

**Fluxo principal**
- O operador chama `POST /admin/webhooks/dead-letter/:id/replay` autenticado
- O middleware `requireRole` já existente restringe o acesso ao papel ADMIN
- O serviço busca o item na `webhook_dead_letter` pelo id
- O evento é recolocado como pendente na `webhook_outbox`
- A execução é registrada em log, identificando quem fez o replay

**Fluxos alternativos e exceções**
- Item de dead letter inexistente retorna erro de recurso não encontrado
- Usuário autenticado sem o papel ADMIN recebe erro de autorização, pelo mecanismo já existente

**Erros previstos**
- `WEBHOOK_NOT_FOUND` aplicado ao contexto da dead letter queue
- Erro de autorização pelo middleware existente quando o papel não é ADMIN

**Prioridade:** media

---

#### RF-08 Consulta do histórico de entregas
Endpoint que devolve ao cliente o histórico recente de envios de um webhook, para que ele audite a própria integração.

**Fluxo principal**
- O cliente chama `GET /webhooks/:id/deliveries` autenticado
- O serviço retorna os últimos 100 registros de entrega daquele webhook
- Cada registro traz o resultado de sucesso ou falha, o payload enviado, a resposta recebida e o tempo de resposta

**Fluxos alternativos e exceções**
- Consulta sobre webhook inexistente retorna erro de recurso não encontrado

**Erros previstos**
- `WEBHOOK_NOT_FOUND` quando o webhook informado não existe

**Prioridade:** media

---

### Requisitos não funcionais

Performance
- Latência de entrega menor que 10 segundos entre o commit da mudança de status e a chegada da requisição no endpoint do cliente, limiar definido pelos próprios clientes
- Intervalo de polling do worker de 2 segundos, o que estabelece 2 segundos como pior caso de latência de disparo
- Timeout de 10 segundos por requisição HTTP ao endpoint do cliente
- Limite de payload de 64 KB por evento
- Worker processa os eventos pendentes em lotes pequenos. O tamanho exato do lote não foi definido pela fonte
- Hipótese. Não há volume esperado de eventos, throughput alvo nem tempo de resposta definido para os endpoints REST da feature. Como default para os endpoints síncronos, adota-se p95 menor que 150 ms, marcado como hipótese por não ter sido definido pela fonte

Disponibilidade
- Hipótese. A fonte não define uptime alvo para a API nem para o worker. Como default para sistema voltado ao cliente externo, adota-se 99.9 por cento de disponibilidade mensal em produção, marcado como hipótese
- O worker roda em processo separado da API justamente para não ser derrubado quando a API reinicia
- Hipótese. O comportamento esperado caso o processo do worker pare por completo, incluindo restart, health check e alerta, não foi definido pela fonte e permanece em aberto

Segurança e autorização
- Autenticação por JWT do próprio sistema, no mesmo padrão do restante da API
- O CRUD de configuração de webhook aceita qualquer papel autenticado nesta entrega, condição registrada como provisória
- O replay de item da dead letter queue exige o papel ADMIN, aplicado pelo middleware `requireRole` já existente
- Secret única por endpoint, gerada pela plataforma, rotacionável, com a secret anterior válida por 24 horas
- Assinatura HMAC-SHA256 sobre o corpo da requisição, transmitida no header `X-Signature`
- URL do webhook obrigatoriamente HTTPS, validada por schema Zod, com recusa de cadastro em HTTP
- Log de auditoria identificando quem executou o replay administrativo
- Revisão de segurança do código antes do deploy, com pelo menos 2 dias úteis reservados, focada em HMAC e geração de secret

Observabilidade
- Logging pelo Pino já presente em todo o projeto, sem introduzir ferramenta nova
- Histórico de entregas exposto ao cliente com resultado, payload, resposta e tempo de resposta dos últimos 100 envios
- Hipótese. Métricas internas de operação, alertas de taxa de falha e tracing distribuído não foram definidos pela fonte. Como default, adota-se observabilidade mínima de logs estruturados, métricas de erro por endpoint e tracing distribuído ponta a ponta, marcado como hipótese

Confiabilidade e integridade de dados
- Inserção do evento na outbox dentro da mesma transação SQL da mudança de status, com rollback conjunto
- Garantia de entrega at-least-once, e não exactly-once
- Deduplicação de responsabilidade do cliente, com base no `X-Event-Id` em UUID único por evento
- Ordenação garantida por `order_id` apenas enquanto houver um único worker. Não há garantia de ordenação global, e essa é uma limitação conhecida
- Payload gravado como snapshot no momento da inserção na outbox, e não recalculado no envio
- Identificadores em UUID, seguindo o padrão do restante do projeto

Compatibilidade e portabilidade
- Payload em JSON com campos fixos, `timestamp` em ISO 8601 e `Content-Type: application/json`
- Worker executado como entry point Node separado do processo da API, na mesma stack e com instância própria de PrismaClient, apontando para o mesmo banco
- Nenhuma comunicação direta entre os processos da API e do worker, que se sincronizam apenas pelo banco de dados
- Versionamento do payload e estratégia de evolução de schema do evento não foram definidos pela fonte e permanecem em aberto

Compliance
- Trilha de auditoria do replay administrativo, registrando quem executou a ação
- Arquivamento das linhas entregues da outbox após cerca de 30 dias é reconhecido como necessário, mas está fora desta entrega
- Exigências regulatórias de retenção e tratamento de dados não foram tratadas pela fonte

---

### Arquitetura e abordagem

Abordagem
- Transactional outbox no MySQL já existente, com worker em processo separado fazendo polling e entregando os eventos por HTTP, com retry, backoff, dead letter queue e assinatura HMAC-SHA256 por endpoint. A abordagem foi escolhida para evitar tanto o disparo síncrono dentro da transação de pedidos quanto a introdução de infraestrutura nova de mensageria

Componentes
- Tabela `webhook_outbox`, que registra os eventos a entregar, com o payload em snapshot, estado do evento, `created_at` e `order_id`, e índices sobre estado e data de criação
- Função `publishWebhookEvent`, que recebe o client da transação corrente, aplica o filtro de status de interesse e insere o evento
- `OrderService.changeStatus` em `src/modules/orders/order.service.ts`, ponto de integração existente que passa a chamar a publicação do evento dentro da própria transação
- Módulo `src/modules/webhooks`, com controller, service, repository, routes e schemas, responsável pelo CRUD de configuração, rotação de secret, assinatura e consulta de entregas
- Worker em processo Node separado, com entry point próprio e script dedicado, responsável pelo polling, montagem do payload, envio, retry e movimentação para a dead letter queue
- Tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp
- Endpoint administrativo de replay, restrito ao papel ADMIN
- Componentes reaproveitados sem alteração: `AppError` e classes de erro específicas com prefixo `WEBHOOK_`, logger Pino, error middleware centralizado e middleware `requireRole`

Integrações
- Mudança de status para outbox, por transação de banco de dados, sem chamada HTTP intermediária
- Worker para banco de dados, por consultas SQL via Prisma a cada 2 segundos, no mesmo banco da API e com instância própria de PrismaClient
- Worker para endpoint do cliente, por HTTP POST com timeout de 10 segundos e corpo assinado por HMAC-SHA256. Esta é a única integração externa da feature, e o fluxo é exclusivamente outbound
- Cliente para plataforma, por endpoints REST autenticados com JWT, para configuração de webhook, rotação de secret e consulta de entregas

Pontos de arquitetura ainda não fechados
- Estrutura completa de colunas das tabelas novas, incluindo o nome definitivo da tabela de configuração e a persistência do histórico de entregas
- Estratégia de assinatura durante o grace period de rotação de secret

---

### Decisões e trade-offs

#### Decisão: adotar transactional outbox no MySQL existente, em vez de disparo síncrono ou fila dedicada
- **Justificativa:** o disparo síncrono travaria a mudança de status de outros pedidos quando o endpoint do cliente estivesse lento, e não haveria critério sensato para decidir sobre rollback da mudança de status por causa de uma falha de entrega. A fila dedicada resolveria o problema, mas exigiria provisionar e operar infraestrutura nova, considerada desproporcional ao tamanho do time.
- **Trade-off:** acopla a gravação do evento a uma transação de pedidos que já é pesada, acrescentando mais uma escrita a ela, em troca de garantia forte de consistência entre status e evento.

#### Decisão: worker executado como processo separado da API
- **Justificativa:** se o worker rodasse dentro da mesma instância da API, um reinício da API interromperia o processamento dos eventos pendentes.
- **Trade-off:** exige um entry point e um ciclo de deploy adicionais, além de gerenciamento de um segundo processo em produção.

#### Decisão: worker em polling a cada 2 segundos, sem mecanismo reativo
- **Justificativa:** o MySQL não oferece mecanismo nativo de notificação de processo externo equivalente ao do PostgreSQL, e improvisar reatividade por trigger seria frágil. O polling de 2 segundos já atende com folga o limiar de 10 segundos exigido pelos clientes.
- **Trade-off:** a latência mínima de disparo passa a ser de 2 segundos no pior caso, custo aceito explicitamente na reunião.

#### Decisão: assinatura HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24 horas
- **Justificativa:** garante ao cliente a autenticidade e a integridade do evento recebido, e a secret por endpoint limita o raio de impacto de um vazamento a um único cliente. A motivação foi um precedente concreto de cliente que vazou a secret em log de aplicação.
- **Trade-off:** aumenta a complexidade de gestão de secrets, com mais de uma secret válida simultaneamente durante a rotação, e torna o esquema de assinatura um contrato difícil de alterar depois que os clientes estiverem integrados.

#### Decisão: garantia de entrega at-least-once, com deduplicação a cargo do cliente
- **Justificativa:** exactly-once exigiria coordenação cara entre plataforma e cliente. A entrega at-least-once com identificador único por evento é o padrão adotado por plataformas de mercado equivalentes.
- **Trade-off:** transfere ao cliente a responsabilidade de deduplicar pelo `X-Event-Id`, e um cliente que não implemente isso processará a mesma mudança de status mais de uma vez.

#### Decisão: ordenação garantida apenas por `order_id` e apenas em regime de worker único
- **Justificativa:** os clientes nunca pediram ordenação global entre pedidos distintos, e a ordenação por pedido é suficiente para o caso de uso relatado.
- **Trade-off:** a garantia se perde no momento em que o sistema escalar para múltiplos workers, o que exigirá particionamento por `order_id` ou lock pessimista, trabalho deliberadamente adiado.

#### Decisão: 5 tentativas de entrega com backoff exponencial de 1 minuto a 12 horas
- **Justificativa:** a janela total de cerca de 15 horas cobre indisponibilidades planejadas do lado do cliente, com base em um precedente real de 2 horas de indisponibilidade. Um número menor de tentativas em janela curta descartaria o evento cedo demais, e o retry indefinido deixaria eventos pendurados para sempre caso o cliente desaparecesse.
- **Trade-off:** um evento pode levar até cerca de 15 horas para ser considerado definitivamente perdido, e o número de tentativas já foi contestado internamente, podendo ser revisto com dados de produção.

#### Decisão: CRUD de configuração de webhook aberto a qualquer papel autenticado, com ADMIN exigido apenas no replay
- **Justificativa:** reduz a fricção operacional nesta primeira entrega, enquanto a operação sensível de reprocessamento fica protegida pelo papel ADMIN e auditada.
- **Trade-off:** qualquer usuário autenticado pode alterar o destino dos webhooks de um customer nesta fase. A própria área de segurança registrou a permissão como provisória, a ser endurecida depois.

---

### Dependências

#### Organizacional: revisão de segurança antes do deploy
A engenheira de segurança precisa revisar o código antes da implantação, com pelo menos 2 dias úteis reservados na janela de entrega, com foco na implementação do HMAC e na geração das secrets. Sem essa revisão concluída, o deploy não deve acontecer.

#### Organizacional: revisão do documento de design com o time de engenharia
A tech lead precisa abrir o documento de design e conduzir uma sessão de revisão com os engenheiros envolvidos antes do início da implementação, para que as decisões fechadas na reunião sejam validadas contra o desenho detalhado.

#### Organizacional: comunicação e documentação para os clientes
O product manager precisa atualizar Atlas Comercial, MaxDistribuição e Nova Cargo sobre o prazo de entrega, e documentar no portal do desenvolvedor o comportamento at-least-once e a necessidade de deduplicação pelo `X-Event-Id`. Sem essa documentação, os clientes integram contra uma expectativa errada de garantia de entrega.

#### Técnica: nenhuma infraestrutura nova necessária
A feature depende apenas do que já existe no sistema: o banco MySQL acessado via Prisma, o `changeStatus` em `src/modules/orders/order.service.ts`, as classes de erro em `AppError`, o error middleware, o logger Pino e o middleware `requireRole`. A escolha pelo outbox no MySQL foi tomada justamente para não criar dependência de provisionamento de infraestrutura por outra equipe.

#### Externa: implementação do lado de cada cliente integrador
Cada cliente precisa expor um endpoint HTTPS capaz de receber requisições POST, verificar a assinatura HMAC-SHA256 com a secret fornecida para aquele endpoint, deduplicar eventos pelo `X-Event-Id` e migrar para a nova secret dentro da janela de 24 horas sempre que fizer uma rotação. Sem isso a integração não funciona ponta a ponta, mesmo com a plataforma entregando corretamente.

---

### Riscos e mitigação

Observação sobre probabilidade
- A fonte não atribuiu probabilidade a nenhum risco. Os valores de probabilidade declarados abaixo são hipótese do redator do PRD, e não consenso registrado na reunião. Os riscos em si, bem como suas mitigações, vieram de preocupações verbalizadas.

#### Endpoint do cliente indisponível ou lento faz o evento esgotar as tentativas e não ser entregue
- **Probabilidade:** media
- **Impacto:** o cliente deixa de ser notificado da mudança de status e volta a depender de consulta manual, com desgaste em uma relação comercial já sensível
- **Mitigação:**
  - Retry com backoff exponencial de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, cobrindo cerca de 15 horas de indisponibilidade
  - Timeout de 10 segundos por tentativa, para não bloquear o processamento dos demais eventos
- **Plano de contingência:** o evento é preservado na `webhook_dead_letter` com payload e motivo da falha, e pode ser reprocessado manualmente pelo endpoint administrativo de replay assim que o cliente se restabelecer.

#### Vazamento da secret de um cliente permite que um terceiro forje eventos assinados
- **Probabilidade:** media
- **Impacto:** um terceiro consegue forjar notificações válidas para o endpoint daquele cliente, comprometendo a confiança na integração
- **Mitigação:**
  - Secret única por endpoint, e não secret global, limitando o raio de impacto a um único cliente
  - Rotação de secret disponível pela API, sem depender de intervenção manual da plataforma
  - Revisão de segurança do código de geração de secret e de assinatura antes do deploy
- **Plano de contingência:** o cliente solicita a rotação da secret pela API, e a secret comprometida deixa de ser válida ao fim das 24 horas de transição.

#### Escalar para múltiplos workers quebra a ordenação de eventos do mesmo pedido
- **Probabilidade:** baixa
- **Impacto:** o cliente pode receber mudanças de status do mesmo pedido fora de ordem, e interpretar um estado antigo como atual
- **Mitigação:**
  - Manter regime de worker único nesta entrega, no qual a ordenação por `order_id` é garantida
  - Documentar a ausência de ordenação global como limitação conhecida da feature
- **Plano de contingência:** implementar particionamento por `order_id` ou lock pessimista antes de introduzir o segundo worker, trabalho já esboçado e deliberadamente adiado.

#### Volume alto de mudanças de status em curto intervalo sobrecarrega o endpoint do cliente
- **Probabilidade:** media
- **Impacto:** o endpoint do cliente pode passar a recusar ou demorar a responder, gerando falhas de entrega e consumo desnecessário de tentativas de retry
- **Mitigação:**
  - Nenhuma mitigação implementada nesta entrega. A decisão registrada foi observar o comportamento em produção e agir apenas se o problema se materializar
- **Plano de contingência:** hipótese. A fonte não define plano de contingência para este risco, nem gatilho ou limite a partir do qual agir. Na ausência de definição, o comportamento previsto é que a sobrecarga se manifeste como falha de entrega e seja absorvida pelo retry e pela dead letter queue, com a decisão sobre rate limiting reaberta a partir dos dados observados.

#### Cliente que não implementa deduplicação processa o mesmo evento mais de uma vez
- **Probabilidade:** media
- **Impacto:** inconsistência no sistema do cliente, com uma mesma mudança de status contabilizada em duplicidade
- **Mitigação:**
  - Envio do `X-Event-Id` em UUID único por evento em todas as requisições
  - Documentação explícita da garantia at-least-once e da necessidade de deduplicação no portal do desenvolvedor
- **Plano de contingência:** não há ação corretiva do lado da plataforma além da documentação e do suporte ao cliente na correção da integração. O risco residual foi aceito de forma consciente ao escolher at-least-once.

#### Falha na inserção do evento na outbox impede a mudança de status do pedido
- **Probabilidade:** baixa
- **Impacto:** a operação de mudança de status falha para o usuário, já que evento e mudança compartilham a mesma transação
- **Mitigação:**
  - Inserção do evento dentro da mesma transação SQL da mudança de status, com rollback conjunto, garantindo que nunca exista status alterado sem evento correspondente
  - Payload renderizado como snapshot no momento da inserção, evitando dependência de estado recalculado
- **Plano de contingência:** hipótese. A fonte não define o comportamento devolvido ao chamador nesse cenário. A hipótese é que o erro propague pelo error middleware já existente, como ocorre hoje com qualquer exceção dentro da transação, cabendo ao chamador repetir a operação de mudança de status.

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Mudar o status de um pedido para um status inscrito por ao menos um webhook ativo do customer insere uma linha em `webhook_outbox` dentro da mesma transação, e um rollback da transação não deixa nenhuma linha na outbox
- Mudar o status de um pedido para um status ao qual nenhum webhook do customer está inscrito não insere linha na outbox
- Um evento pendente é entregue ao endpoint do cliente em menos de 10 segundos contados do commit da transação, no cenário sem falha de entrega
- A requisição enviada ao cliente contém os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`
- O corpo da requisição contém os campos `event_id`, `event_type` com valor `order.status_changed`, `timestamp` em ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, e não contém a lista de itens do pedido
- O valor de `X-Signature` corresponde ao HMAC-SHA256 do corpo da requisição calculado com a secret do endpoint, verificável recalculando a assinatura no teste
- Cadastro de webhook com URL em HTTP é recusado com erro de validação, e cadastro com URL em HTTPS é aceito
- Evento cujo payload ultrapassa 64 KB não é enviado e resulta em erro registrado, sem truncamento do conteúdo
- Falhas sucessivas de entrega reagendam o evento respeitando a progressão de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, e a falha da última tentativa move o evento para `webhook_dead_letter` com payload, motivo e timestamp
- `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento como pendente na outbox quando chamado por usuário com papel ADMIN, e retorna erro de autorização quando chamado por usuário sem esse papel
- A execução do replay administrativo gera registro de log identificando quem executou a ação
- A rotação de secret gera uma nova secret, mantém a anterior válida por 24 horas e a invalida após esse período
- `GET /webhooks/:id/deliveries` retorna até 100 registros mais recentes, cada um com resultado de sucesso ou falha, payload, resposta e tempo de resposta
- Dois eventos consecutivos do mesmo pedido chegam ao endpoint do cliente na ordem de `created_at` da outbox, em regime de worker único
- O worker continua processando eventos após um reinício do processo da API, comprovando que roda em processo separado
- Todos os erros do módulo usam código com prefixo `WEBHOOK_`, herdam de `AppError` e são tratados pelo error middleware existente sem alteração nele

---

### Testes e validação

Tipos de teste obrigatórios
- Testes de integração ponta a ponta no padrão já vigente no projeto, com Vitest como runner e Supertest fazendo requisições contra a aplicação real construída pelos helpers de teste, sem mock de banco
- Teste de integração cobrindo a publicação do evento na outbox a partir do `changeStatus`, incluindo o caso de rollback conjunto e o caso de nenhum webhook inscrito no status
- Teste de entrega do evento verificando headers, corpo do payload e recálculo da assinatura HMAC-SHA256
- Teste do ciclo de retry, do backoff e da movimentação para a dead letter queue
- Teste de autorização do endpoint de replay administrativo, cobrindo o acesso com papel ADMIN e a recusa sem ele
- Hipótese. A reunião não definiu uma matriz formal de tipos de teste obrigatórios. A lista acima deriva do padrão vigente na codebase e do escopo de esforço estimado para a integração no `order.service` e para os testes ponta a ponta

Estratégia de validação
- Seguir a convenção já estabelecida no projeto: cada teste sobe a aplicação, popula dados pelos helpers de factory e valida status e corpo da resposta, com limpeza das tabelas antes de cada execução. Isso exige acrescentar helpers de factory para webhook cadastrado, evento de outbox e item de dead letter, e incluir as novas tabelas na rotina de limpeza
- Hipótese. O worker roda em loop contínuo de polling, o que não se encaixa no padrão atual de teste de endpoint HTTP síncrono. A abordagem prevista é expor uma função de processamento de um lote, testável isoladamente, em vez de exercitar o loop completo. Isso decorre do desenho da arquitetura e não de uma decisão explícita da fonte
- Portão de aprovação obrigatório antes do deploy: revisão de segurança do código, com pelo menos 2 dias úteis reservados, focada na implementação do HMAC e na geração das secrets
- Hipótese. Não há definição de pipeline de integração contínua, ambiente de homologação ou teste automatizado como bloqueio de deploy. O único portão formal registrado pela fonte é a revisão de segurança
