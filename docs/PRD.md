### PRD: Order Management System (OMS) Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-08-31
Responsável: Luciano Vieira Junior

---

### Resumo

Clientes B2B integrados ao OMS hoje descobrem mudanças de status de pedido fazendo polling no `GET /orders`, o que torna a integração deles lenta e cara. Esta feature entrega notificação push outbound: quando o status de um pedido muda, o OMS envia um HTTP POST assinado para o endpoint cadastrado pelo cliente, em menos de 10 segundos. A solução usa transactional outbox no MySQL existente, com um worker em processo separado que faz polling a cada 2 segundos, retry com backoff, dead letter queue e assinatura HMAC-SHA256. Nenhuma infraestrutura nova é introduzida.

---

### Contexto e problema

Público-alvo
- Clientes B2B integrados à plataforma, nominalmente Atlas Comercial, MaxDistribuição e Nova Cargo
- Usuários autenticados do OMS que operam o cadastro de webhooks em nome do cliente
- Usuários de papel ADMIN, únicos autorizados a reprocessar a dead letter queue

Cenários de uso chave
- Cliente cadastra um endpoint https e a lista de status que deseja receber, e passa a ser notificado a cada transição assinada
- Cliente fica indisponível por algumas horas e recebe os eventos quando volta, sem perda
- Cliente consulta o histórico das últimas entregas para diagnosticar a própria integração
- ADMIN reprocessa manualmente um evento que esgotou as tentativas e caiu na dead letter queue
- Cliente rotaciona a secret de assinatura e migra seus sistemas dentro da janela de tolerância

Onde essa feature será implantada
- Sistema existente. Entra no `order-management-api` como o módulo `src/modules/webhooks`, seguindo o padrão de controller, service, repository, routes e schemas dos demais domínios, mais três tabelas novas no MySQL já em uso e um processo worker independente com entry-point próprio em `src/worker.ts`. O ponto de acoplamento com o código atual é o `changeStatus` do `OrderService`, que já opera dentro de uma transação Prisma.

Problemas priorizados
A classificação de impacto e prioridade abaixo é hipótese do autor. A reunião levantou os problemas mas não os classificou.
- Clientes fazem polling contínuo no `GET /orders` para detectar mudanças de status, o que deixa a integração deles lenta e cara e exige atualização quase manual. Impacto alto, prioridade alta.
- Risco comercial concreto de perda de cliente. A Atlas Comercial sinalizou que pode migrar para o concorrente se a capacidade não for entregue no prazo acordado. Impacto alto, prioridade alta.
- A plataforma não possui hoje nenhum mecanismo de notificação externa, eventos ou filas, o que impede oferecer integração push como capacidade de produto. Impacto médio, prioridade média.

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Substituir o polling do cliente por notificação push em tempo real | Latência entre o commit da mudança de status e a saída da chamada HTTP | Abaixo de 10 segundos, sendo 2 segundos o pior caso do desenho adotado |
| Entregar a notificação dentro da janela que o cliente considera tempo real | Latência ponta a ponta até a recepção pelo cliente, em caminho feliz | Abaixo de 10 segundos quando o endpoint do cliente responde normalmente. Entregas que entram em ciclo de retry ficam fora desta meta |
| Não degradar o caminho crítico de mudança de status | Latência de rede acrescentada à transação do `changeStatus` | Zero. Apenas um INSERT local entra na transação. Meta derivada do desenho, não acordada como indicador |
| Tolerar indisponibilidade temporária do cliente sem perder eventos | Janela de recuperação coberta pelo ciclo de retry | Aproximadamente 15 horas, com 5 tentativas em backoff 1m, 5m, 30m, 2h e 12h |
| Garantir que nenhum evento seja descartado silenciosamente | Destino de eventos que esgotaram as tentativas | 100 por cento persistidos na dead letter queue e reprocessáveis. Meta derivada do desenho, não acordada como indicador |
| Permitir rotação de secret sem downtime do cliente | Janela de validade simultânea da secret anterior | 24 horas |


---

### Escopo

Incluso
- Tabela de configuração de webhook com url, secret, customer_id, estado ativo e lista de status assinados
- Tabela `webhook_outbox` com estados pendente, processando, falhou e entregue, id UUID, payload em snapshot e índices em status e `created_at`
- Tabela `webhook_dead_letter` com payload, motivo da falha e timestamp
- Gravação do evento na outbox dentro da transação do `changeStatus`, via função que recebe o transaction client
- Filtro por status assinados aplicado no momento da inserção na outbox
- Worker como processo separado, com entry-point `src/worker.ts`, script `npm run worker` e polling de 2 segundos
- Envio HTTP POST com timeout de 10 segundos e marcação do resultado
- Retry com backoff de 1m, 5m, 30m, 2h e 12h, totalizando 5 tentativas
- Dead letter queue e endpoint de replay restrito a ADMIN, com log de auditoria
- Assinatura HMAC-SHA256 do corpo, secret única por endpoint, rotação com grace period de 24 horas
- Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e Content-Type application/json
- Validação de URL https obrigatória e teto de payload de 64KB com erro
- CRUD de configuração de webhook autenticado por JWT
- Endpoint de histórico de entregas com sucesso, falha, payload, response e tempo de resposta
- Reuso dos padrões existentes: AppError, prefixo `WEBHOOK_` nos códigos de erro, schemas Zod, error middleware centralizado, logger Pino, `requireRole` e ids em UUID

Fora de escopo
- Adiado para fase futura: email de alerta ao cliente quando o webhook dele falha repetidamente, a ser avaliado depois de medir o impacto
- Adiado para fase futura: rate limiting de saída, a ser implementado apenas se virar problema observado
- Adiado para fase futura: múltiplos workers em paralelo e ordenação global de eventos, que exigiriam particionamento por `order_id` ou lock pessimista
- Adiado para fase futura: arquivamento das linhas entregues da outbox após aproximadamente 30 dias
- Adiado para fase futura: endurecimento da autorização do CRUD de webhook, hoje aberto a qualquer papel autenticado
- Adiado para fase futura: dashboard visual para o cliente, transferido para o time de frontend como projeto separado

- Descartado na reunião: disparo síncrono de HTTP dentro do `changeStatus`, recusado porque um cliente lento travaria a mudança de status de outros pedidos e não há rollback possível para uma entrega já feita
- Descartado na reunião: Redis Streams ou qualquer fila externa, recusado como overengineering para um time pequeno e por exigir infraestrutura nova
- Descartado na reunião: trigger de banco em vez de polling, recusado porque o MySQL não possui NOTIFY e LISTEN e um trigger não notifica processo externo
- Descartado na reunião: worker rodando dentro da instância da API, recusado porque um reinício da API derrubaria a entrega
- Descartado na reunião: retry indefinido, recusado por deixar eventos pendurados quando o cliente desaparece
- Descartado na reunião: dead letter queue como flag na própria outbox, recusada em favor de tabela separada
- Descartado na reunião: entrega exactly-once, recusada por exigir coordenação bilateral. Adotado at-least-once com deduplicação pelo cliente
- Descartado na reunião: secret global de plataforma, recusada porque o vazamento de uma comprometeria todas
- Descartado na reunião: truncamento de payload acima do limite, recusado em favor de erro explícito
- Descartado na reunião: envio dos itens do pedido no payload, recusado por inflar a mensagem
- Descartado na reunião: guardar apenas o `order_id` e renderizar o payload no envio, recusado porque o evento deixaria de refletir o estado do momento da transição
- Descartado na reunião: webhooks inbound, fora de escopo por definição. A feature é outbound-only

---

### Requisitos funcionais

#### RF-01 Cadastro de configuração de webhook
O sistema deve permitir cadastrar um endpoint de webhook para um customer, gerando a secret de assinatura e devolvendo-a na criação.

**Fluxo principal**
- Usuário autenticado por JWT chama o endpoint de criação de webhook
- O body traz a url, o customer_id e a lista de status que o endpoint deseja receber
- O schema Zod valida o body, exigindo que a url use https
- O sistema gera a secret e persiste url, secret, customer_id, estado ativo e lista de status
- O sistema retorna o webhook criado com a secret em claro

**Fluxos alternativos e exceções**
- Um mesmo customer pode ter mais de um webhook cadastrado, cada um com sua própria secret e sua própria lista de status assinados

**Erros previstos**
- URL fora de https bloqueia o cadastro com `WEBHOOK_INVALID_URL`
- Body inválido retorna `VALIDATION_ERROR` com status 400, tratado pelo error middleware existente
- Requisição sem JWT válido retorna `UNAUTHORIZED` com status 401
- Em aberto: limite de webhooks por customer, unicidade de url por customer e validação de existência do customer_id

**Prioridade:** alta

---

#### RF-02 Edição, remoção e listagem de webhooks
O sistema deve oferecer edição, remoção e listagem por customer das configurações de webhook.

**Fluxo principal**
- Usuário autenticado chama PATCH para editar, DELETE para remover ou GET para listar por customer
- O sistema valida params e body com Zod, aplica a alteração e retorna o resultado

**Fluxos alternativos e exceções**
- O webhook possui estado ativo, o que permite desativar sem remover
- Em aberto: se a desativação é feita via PATCH ou por endpoint dedicado

**Erros previstos**
- Webhook inexistente retorna `WEBHOOK_NOT_FOUND` com status 404
- Body ou params inválidos retornam `VALIDATION_ERROR` com status 400
- Requisição sem JWT válido retorna `UNAUTHORIZED` com status 401
- Em aberto: remoção lógica ou física, destino dos eventos pendentes na outbox quando o webhook é removido, e isolamento entre customers

**Prioridade:** alta

---

#### RF-03 Filtro de eventos por endpoint
O sistema deve gerar evento apenas para os status que o endpoint assinou.

**Fluxo principal**
- Cada webhook guarda a lista de status de interesse, dentro do enum já existente de status de pedido
- Na mudança de status, o filtro é aplicado no momento da inserção na outbox, não no envio
- Se nenhum webhook do customer assina aquele status, nenhuma linha é inserida na outbox

**Fluxos alternativos e exceções**
- Customer com múltiplos webhooks e filtros distintos é suportado, e o header X-Webhook-Id identifica qual cadastro originou cada envio
- Em aberto: se o desenho gera uma linha de outbox por webhook ou uma linha por evento com fan-out no envio

**Erros previstos**
- Ausência de assinante é caminho normal e não gera erro

**Prioridade:** alta

---

#### RF-04 Gravação na outbox dentro da transação de mudança de status
O sistema deve inserir o evento na `webhook_outbox` dentro da mesma transação SQL que altera o status do pedido.

**Fluxo principal**
- O `changeStatus` abre a transação Prisma já existente
- Dentro dela ocorrem a validação da transição pela máquina de estados, o ajuste de estoque, o update da order e o insert em `order_status_history`
- É chamada a função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o transaction client em vez de injetar o repository no `OrderService`
- A função aplica o filtro do RF-03 e, havendo assinante, insere a linha na outbox com o payload já renderizado e um event_id UUID
- A transação commita, e o evento fica registrado se e somente se a mudança de status commitou

**Fluxos alternativos e exceções**
- Nenhum status assinado significa nenhuma inserção, e a transação segue normalmente

**Erros previstos**
- Falha ao inserir na outbox provoca rollback de toda a transação, mantendo a order no status anterior
- Erros anteriores do fluxo continuam válidos: `INVALID_STATUS_TRANSITION` com 409, `INSUFFICIENT_STOCK` com 422 e `NOT_FOUND` com 404
- Em aberto: se a criação inicial do pedido também gera evento ou apenas as transições posteriores

**Prioridade:** alta

---

#### RF-05 Worker de polling e envio HTTP
Um processo separado deve varrer a outbox a cada 2 segundos, enviar os eventos pendentes por HTTP POST assinado e registrar o resultado.

**Fluxo principal**
- O processo sobe por `src/worker.ts` e `npm run worker`, com PrismaClient próprio apontando para o mesmo banco
- A cada 2 segundos, busca os eventos pendentes mais antigos em batch pequeno, usando os índices de status e `created_at`
- Ordena por `created_at`, preservando a ordem de processamento dos eventos de um mesmo pedido enquanto houver um único worker. Eventos que entram em ciclo de retry podem chegar ao cliente fora de ordem
- Marca o evento como processando, monta os headers do RF-08 e envia o POST com Content-Type application/json
- Em caso de sucesso, marca como entregue e registra a entrega conforme o RF-10

**Fluxos alternativos e exceções**
- Ausência de resposta em 10 segundos é tratada como falha e o evento entra no ciclo de retry
- Erro de conexão ou resposta de erro do cliente leva ao mesmo ciclo de retry
- Payload acima de 64KB não é enviado e gera erro
- A garantia é at-least-once, portanto o cliente pode receber o mesmo evento mais de uma vez e deve deduplicar pelo X-Event-Id
- A ordenação é garantida por pedido e apenas enquanto houver um único worker, o que constitui limitação conhecida e documentada

**Erros previstos**
- Em aberto: quais códigos HTTP contam como sucesso, se respostas 4xx são falha permanente em vez de retentável, tamanho do batch, tratamento de evento preso no estado processando quando o worker morre no meio, e comportamento diante de webhook removido ou desativado após a inserção do evento

**Prioridade:** alta

---

#### RF-06 Retry com backoff exponencial
Uma entrega que falha deve ser retentada até 5 vezes com backoff exponencial antes de virar falha permanente.

**Fluxo principal**
- O envio falha por erro de conexão, erro HTTP ou timeout de 10 segundos
- O evento é reagendado com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas
- São 5 tentativas no total, cobrindo aproximadamente 15 horas
- Esgotadas as tentativas, o evento é tratado como falha permanente e segue para o RF-07

**Fluxos alternativos e exceções**
- Sucesso em qualquer tentativa intermediária encerra o ciclo e marca o evento como entregue

**Erros previstos**
- O estado falhou já faz parte do conjunto de estados da outbox
- Em aberto: se as 5 tentativas incluem a tentativa original, se há jitter no backoff e se o contador é por evento ou por par de evento e webhook

**Prioridade:** alta

---

#### RF-07 Dead letter queue e reprocessamento manual
Eventos que esgotaram as tentativas devem ser persistidos em tabela separada e poder ser reprocessados manualmente por um ADMIN.

**Fluxo principal**
- Após a última falha, o evento é movido para a `webhook_dead_letter`, que guarda payload, motivo da falha e timestamp
- Um ADMIN chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
- O middleware de autenticação valida o JWT e o `requireRole('ADMIN')` valida o papel
- O evento é recolocado na outbox com estado pendente
- O sistema registra em log quem executou o replay, para auditoria

**Fluxos alternativos e exceções**
- Em aberto: replay em lote, listagem da dead letter queue por endpoint e se a linha sai da tabela ou é marcada como reprocessada

**Erros previstos**
- Papel diferente de ADMIN retorna `FORBIDDEN` com status 403
- Requisição sem JWT válido retorna `UNAUTHORIZED` com status 401
- Identificador inexistente retorna 404 no padrão de erro do módulo
- Em aberto: replay de evento cujo webhook foi removido ou cuja secret foi rotacionada desde a falha

**Prioridade:** alta

---

#### RF-08 Assinatura HMAC e headers do envio
Todo envio deve ser assinado com HMAC-SHA256 usando a secret do endpoint e carregar os headers de identificação e verificação.

**Fluxo principal**
- O worker monta o corpo JSON do evento com event_id, event_type, timestamp em ISO 8601, order_id, order_number, from_status, to_status, customer_id e total_cents, sem os itens do pedido
- Calcula o HMAC-SHA256 sobre o corpo do request usando a secret daquele endpoint
- Envia os headers X-Signature, X-Event-Id, X-Timestamp, X-Webhook-Id e Content-Type application/json
- O cliente verifica a assinatura do lado dele

**Fluxos alternativos e exceções**
- Cliente que precise dos itens do pedido consulta o `GET /orders/:id` em uma chamada separada

**Erros previstos**
- A verificação da assinatura é responsabilidade do cliente e não gera erro do lado do OMS
- Em aberto: encoding da assinatura entre hexadecimal e base64, formato exato do header e se o X-Timestamp entra no cálculo do HMAC

**Prioridade:** alta

---

#### RF-09 Secret por endpoint com rotação e grace period
Cada endpoint deve ter secret única, gerada pela plataforma, rotacionável via API, com a secret anterior válida por 24 horas em paralelo.

**Fluxo principal**
- A secret é gerada pela plataforma na criação do webhook e devolvida na resposta
- A secret é única por endpoint, nunca compartilhada entre cadastros
- O cliente chama o endpoint de rotação para obter uma nova secret
- A nova secret passa a ser usada e a anterior permanece válida por 24 horas, permitindo a migração dos sistemas do cliente
- Após 24 horas a secret anterior deixa de ser aceita

**Fluxos alternativos e exceções**
- Em aberto: se durante o grace period o envio carrega duas assinaturas ou apenas a da secret vigente. Esta é a lacuna mais relevante do requisito e deve ser fechada na revisão de segurança

**Erros previstos**
- Evento com payload acima de 64KB não é enviado e gera erro, sem truncamento
- A transcrição cita `WEBHOOK_SECRET_REQUIRED` entre os códigos do módulo, sem definir a condição que o dispara
- Em aberto: método e caminho do endpoint de rotação, forma de armazenamento da secret em repouso, rotação forçada pela plataforma e rotação disparada por incidente

**Prioridade:** alta

---

#### RF-10 Histórico de entregas
O cliente deve conseguir consultar as entregas recentes de um webhook, com o resultado e os detalhes de cada tentativa.

**Fluxo principal**
- Usuário autenticado chama `GET /api/v1/webhooks/:id/deliveries`
- O sistema retorna as entregas com sucesso ou falha, payload enviado, response recebida e tempo de resposta
- O recorte pedido é o das últimas 100 entregas

**Fluxos alternativos e exceções**
- Hipótese: adotar o helper de paginação já existente na codebase em vez de um limite fixo, mantendo coerência com os demais endpoints de listagem

**Erros previstos**
- Webhook inexistente retorna `WEBHOOK_NOT_FOUND` com status 404
- Requisição sem JWT válido retorna `UNAUTHORIZED` com status 401
- Em aberto: retenção do histórico, truncamento da response armazenada e exposição de dados sensíveis no payload persistido

**Prioridade:** media

---

#### RF-11 Padronização de erros e observabilidade do módulo
O módulo deve usar os padrões de erro, log e tratamento centralizado já existentes, sem introduzir dependência nova.

**Fluxo principal**
- Os erros do módulo são classes derivadas de AppError, no mesmo estilo dos erros de domínio já existentes
- Todos os códigos de erro levam o prefixo `WEBHOOK_`
- O error middleware centralizado, que já trata AppError, Zod e Prisma, responde os erros novos sem alteração
- O log usa o Pino já presente no projeto, sem acrescentar biblioteca
- O replay de dead letter queue gera log de auditoria com o autor da operação

**Fluxos alternativos e exceções**
- O error middleware é um handler do ciclo HTTP do Express e não cobre o processo worker, que precisa do próprio tratamento de erro de topo. Este ponto é inferido do código e não foi decidido na reunião

**Erros previstos**
- Não aplicável. Este requisito define a política de erros e não um fluxo de negócio

**Prioridade:** alta

---

#### RF-12 Registro do módulo e entry-point do worker
O módulo deve ser criado no padrão de módulos da codebase e o worker deve existir como entry-point independente.

**Fluxo principal**
- Criar `src/modules/webhooks` com controller, service, repository, routes e schemas
- Registrar o router no agregador de rotas da API
- Criar `src/worker.ts` ao lado de `src/server.ts` e o script `npm run worker`
- Manter a lógica de processamento em arquivo próprio dentro do módulo de webhooks
- Usar UUID como identificador, seguindo o padrão do projeto

**Fluxos alternativos e exceções**
- Nenhum

**Erros previstos**
- Em aberto: como o processo worker é implantado e supervisionado, incluindo política de reinício e health check

**Prioridade:** alta

---

### Requisitos não funcionais

Performance
- Latência entre o commit da mudança de status e a saída do POST abaixo de 10 segundos, com 2 segundos no pior caso pelo intervalo de polling
- Timeout de 10 segundos por tentativa de envio
- Nenhuma chamada de rede dentro da transação do `changeStatus`, apenas um INSERT local
- Leitura da outbox restrita a eventos pendentes, em batch pequeno, apoiada por índices em status e `created_at`
- Nenhuma linha é inserida quando o status não é assinado por nenhum webhook do customer
- Hipótese: envio sequencial dentro do batch, para preservar a ordenação por pedido que foi assumida como decisão

Disponibilidade
- Hipótese: disponibilidade alvo de 99.9 por cento mensal para a API de webhooks e para o processo worker, por se tratar de capacidade voltada ao cliente externo. Valor a confirmar antes do go live
- Nenhum SLA, SLO ou uptime alvo foi definido na reunião, e nenhuma métrica de sucesso de produto foi formulada. O alvo acima é default assumido, não decisão registrada
- A tolerância efetivamente dimensionada é a de indisponibilidade do cliente: aproximadamente 15 horas de janela de recuperação
- Hipótese: o desenho tolera indisponibilidade do worker sem perda de dados, porque os eventos permanecem pendentes na outbox até serem processados

Segurança e autorização
- Assinatura HMAC-SHA256 sobre o corpo do request, enviada no header X-Signature
- Secret única por endpoint, gerada pela plataforma e devolvida na resposta de criação. Se a secret deixa de ser recuperável depois disso é ponto a fechar na revisão de segurança, junto com a forma de armazenamento em repouso
- Rotação de secret com grace period de 24 horas
- TLS obrigatório. Cadastro com url http é recusado na validação
- Teto de 64KB por payload, com erro em vez de truncamento
- Header X-Timestamp disponível para o cliente detectar tentativa de replay
- Replay de dead letter queue restrito ao papel ADMIN, com log de auditoria do autor
- CRUD de configuração aberto a qualquer papel autenticado nesta fase, com endurecimento explicitamente adiado
- Escopo outbound-only. O sistema não recebe webhooks
- Em aberto: armazenamento da secret em repouso e isolamento entre customers no CRUD
- Hipótese do autor, não levantada na reunião: proteção contra requisições para endereços internos a partir da url cadastrada, a levar para a revisão de segurança

Observabilidade
- Log estruturado com o Pino já existente, sem dependência nova
- Error middleware centralizado responde os erros do módulo sem alteração
- Log de auditoria do replay de dead letter queue
- Histórico de entregas com sucesso, falha, payload, response e tempo de resposta, exposto ao cliente
- Dead letter queue com payload, motivo da falha e timestamp como evidência de diagnóstico
- Estados observáveis da outbox: pendente, processando, falhou e entregue
- Hipótese: log por evento processado no worker, com event_id, webhook_id, número da tentativa, status HTTP e duração
- Em aberto: métricas, alertas operacionais, health check do worker e correlação de identificador de requisição fora do ciclo HTTP

Confiabilidade e integridade de dados
- O insert na outbox ocorre na mesma transação da mudança de status. Se a transação falha, o evento não existe
- Falha ao gravar o evento provoca rollback da mudança de status, por decisão explícita
- Garantia de entrega at-least-once, com deduplicação pelo cliente através do X-Event-Id
- Payload gravado como snapshot no momento da inserção, refletindo o estado do pedido na transição
- Ordenação de processamento garantida por pedido apenas em cenário de worker único, e não garantida para eventos que entraram em ciclo de retry. Registrada como limitação conhecida
- Nenhum evento é descartado silenciosamente. O que falha em definitivo é persistido na dead letter queue
- Em aberto: recuperação de evento preso no estado processando e ausência de lock entre instâncias do worker

Compatibilidade e portabilidade
- Restrição dura de não introduzir infraestrutura nova. A solução usa o MySQL já existente
- Worker usa a mesma stack, o mesmo banco e a mesma string de conexão, apenas em processo e client separados
- Contrato de saída escolhido para consumo universal: JSON, HMAC-SHA256 e timestamp em ISO 8601
- A escolha por polling decorre também de portabilidade, já que o MySQL não oferece o mecanismo de notificação disponível em outros bancos

Compliance
- Não discutido na reunião. Não há menção a legislação de proteção de dados, classificação de dados pessoais ou retenção regulatória, apesar de o payload conter identificador de cliente, número e valor do pedido, e de esse payload ser persistido em outbox, histórico de entregas e dead letter queue
- Ponto a levar para a revisão de segurança: definição de prazo de expurgo dos payloads persistidos

Acessibilidade no frontend consumidor
- Não aplicável nesta fase. A entrega é exclusivamente de API e o dashboard visual foi retirado do escopo

---

### Arquitetura e abordagem

Abordagem
- Transactional outbox com worker em polling. A produção do evento é desacoplada da entrega usando o banco existente como buffer durável, sem infraestrutura nova e sem acrescentar latência de rede ao caminho crítico de mudança de status
- Do lado do código, reuso máximo dos padrões da codebase, sem dependência nova

Componentes
- Módulo `src/modules/webhooks` com controller, service, repository, routes e schemas, responsável pelo CRUD, rotação de secret, histórico de entregas e replay
- Tabela de configuração de webhook com url, secret, customer_id, estado ativo e lista de status assinados
- Tabela `webhook_outbox` como buffer durável, com estados, índices, id UUID e payload em snapshot
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o transaction client, aplica o filtro e insere o evento
- Entry-point `src/worker.ts` com script `npm run worker` e PrismaClient próprio
- Processador do worker, responsável pelo loop de polling, montagem do payload e headers, assinatura, envio, marcação do resultado e agendamento do retry
- Tabela `webhook_dead_letter` com payload, motivo e timestamp
- Registro de entregas, base do endpoint de histórico
- Endpoint administrativo de replay, restrito a ADMIN
- `OrderService.changeStatus`, componente existente que passa a chamar a publicação do evento dentro da transação

Integrações
- Endpoints HTTPS dos clientes B2B, única integração externa da feature, em modelo outbound-only, com POST JSON assinado e timeout de 10 segundos
- Não há message broker, mensageria, streaming ou cache. O mecanismo de fila é uma tabela MySQL lida por polling
- A comunicação é síncrona nos endpoints REST e no POST do worker para o cliente, e assíncrona entre a mudança de status e a entrega do evento

### Decisões e trade-offs

#### Decisão: transactional outbox no MySQL existente
- **Justificativa:** garante atomicidade absoluta entre a mudança de estado e o registro do evento, sem introduzir infraestrutura nova, o que é adequado ao tamanho do time
- **Trade-off:** a transação do `changeStatus`, que já atualiza pedido, histórico e estoque, ganha mais um insert e mais uma tabela na sua fronteira transacional. A entrega deixa de ser imediata e passa a depender de um segundo processo

#### Decisão: worker em polling de 2 segundos em vez de mecanismo reativo
- **Justificativa:** o MySQL não oferece notificação de processo externo, e 2 segundos atendem com folga o limite de 10 segundos aceito pelo cliente
- **Trade-off:** latência mínima de 2 segundos assumida explicitamente, e carga contínua de consultas ao banco mesmo sem eventos, mitigada pelos índices

#### Decisão: worker como processo separado da API
- **Justificativa:** um reinício da API não pode interromper a entrega de eventos
- **Trade-off:** passam a existir dois artefatos de deploy onde havia um, com um segundo pool de conexões contra o mesmo banco. O custo operacional dessa segunda unidade não foi definido

#### Decisão: worker único, com ordenação por pedido e não global
- **Justificativa:** um único processo preserva a ordem que importa para o cliente, e escalar horizontalmente exigiria particionamento ou lock pessimista
- **Trade-off:** o throughput fica limitado a um processo e não há escala horizontal sem redesenho. A garantia de ordem é condicional e se quebra silenciosamente se uma segunda instância for iniciada

#### Decisão: retry com 5 tentativas em backoff de 1m, 5m, 30m, 2h e 12h
- **Justificativa:** três tentativas encerrariam o ciclo em cerca de 30 minutos e não cobririam manutenção planejada de duas horas, cenário já observado com cliente real. Retry indefinido deixaria eventos pendurados
- **Trade-off:** um evento pode levar até aproximadamente 15 horas para ser entregue, e eventos de cliente cronicamente indisponível ocupam a outbox durante esse período

#### Decisão: dead letter queue em tabela separada com replay manual restrito a ADMIN
- **Justificativa:** mantém a leitura da outbox limpa e preserva evidência para diagnóstico. Operar fila de entrega não é atribuição de papel operacional
- **Trade-off:** a recuperação é inteiramente manual e individual, sem reprocessamento automático, em lote ou alerta. Na prática, um evento só é recuperado se alguém inspecionar a tabela

#### Decisão: entrega at-least-once com idempotência delegada ao cliente
- **Justificativa:** exactly-once exigiria coordenação entre as duas pontas. At-least-once com identificador de evento é o padrão adotado por plataformas de referência e resolve a grande maioria dos casos
- **Trade-off:** transfere responsabilidade de correção para o cliente, objeção que foi levantada e assumida conscientemente. A mitigação é documental e não técnica

#### Decisão: integração no OrderService por função que recebe o transaction client
- **Justificativa:** mantém o acoplamento mínimo e explícito e garante que o insert participe da transação em curso, sem injetar um repository inteiro no serviço de pedidos
- **Trade-off:** o módulo de pedidos passa a depender do módulo de webhooks, dependência entre domínios que não existia, e o tipo do transaction client atravessa a fronteira do módulo

#### Decisão: filtro de assinatura aplicado na inserção
- **Justificativa:** evita gravar linhas que nunca seriam entregues
- **Trade-off:** o evento fica congelado contra a configuração vigente no instante da transição. Assinar um status novo depois não recupera eventos que nunca foram gravados. Esta consequência é inferida e não foi enunciada na reunião

#### Decisão: payload em snapshot renderizado na inserção
- **Justificativa:** o evento precisa refletir o estado do pedido no momento da transição, mesmo que o pedido mude depois
- **Trade-off:** duplicação de dados e crescimento das tabelas, já que o payload é armazenado na outbox, no histórico de entregas e na dead letter queue, sem rotina de arquivamento nesta fase

#### Decisão: payload enxuto com teto de 64KB e erro em vez de truncamento
- **Justificativa:** mantém a mensagem pequena, e um payload acima desse limite indica anomalia que deve falhar visivelmente
- **Trade-off:** o cliente que precisar de detalhe do pedido faz uma chamada adicional, de modo que o tráfego de leitura não é eliminado por completo, apenas o polling cego

#### Decisão: HMAC-SHA256 com secret por endpoint e rotação com grace period de 24 horas
- **Justificativa:** o cliente precisa validar origem e integridade. Secret compartilhada significaria que um vazamento comprometeria todos os clientes, e já houve caso de cliente que expôs secret em log
- **Trade-off:** durante as 24 horas de grace period duas secrets são simultaneamente válidas, ou seja, existe uma janela deliberada em que uma secret possivelmente comprometida continua sendo aceita

#### Decisão: reuso integral dos padrões da codebase, sem dependência nova
- **Justificativa:** o error middleware existente já trata os tipos de erro usados, e um time pequeno não comporta divergência de padrão
- **Trade-off:** o error middleware cobre apenas o ciclo HTTP, de modo que o worker fica sem esse tratamento e precisa do próprio guarda de topo, ponto não decidido na reunião

#### Decisão: CRUD de webhook aberto a qualquer papel autenticado nesta fase
- **Justificativa:** aceito como suficiente para a primeira entrega, com endurecimento previsto para depois
- **Trade-off:** um usuário de papel operacional pode cadastrar ou alterar o endpoint de destino de dados de pedidos, e o token não carrega vínculo com customer, o que deixa o isolamento entre clientes sem controle definido

---

### Dependências

#### Organizacional: revisão de segurança antes do deploy
Revisão de HMAC e geração de secret pela engenharia de segurança, portão obrigatório antes de subir a feature. A revisão precisa fechar também o mecanismo de assinatura durante o grace period e o isolamento entre customers no CRUD.

#### Organizacional: sessão de revisão do documento de design
O documento de design da feature precisa ser revisado com os engenheiros de plataforma e de pedidos antes do início da implementação. É nessa sessão que devem ser fechadas as lacunas de desenho ainda abertas.

#### Técnica: migrations das três tabelas
As tabelas de configuração de webhook, outbox e dead letter queue precisam existir antes de qualquer código de worker. É o primeiro bloco da ordem de construção sugerida, derivada da estimativa de esforço por sprint feita na reunião e não de uma priorização acordada.

#### Técnica: entry-point e script do worker
O projeto hoje possui apenas o entry-point da API. É necessário criar o entry-point do worker e o script de execução correspondente.

#### Técnica: configuração de ambiente do worker
O schema de variáveis de ambiente é validado de forma estrita e derruba o processo quando falta variável. Parametrizar intervalo de polling, timeout e tamanho de batch exige estender esse schema. Registrado como hipótese, já que não foi decidido.

#### Técnica: nenhuma infraestrutura nova
Pré-condição negativa e restritiva. Toda a solução precisa caber no banco e na stack já existentes.

#### Externa: endpoints dos clientes B2B
Os clientes precisam expor endpoints https, receber a secret e implementar a verificação da assinatura. Sem isso não há validação ponta a ponta com tráfego real.

#### Externa: documentação no portal de desenvolvedor
A documentação de integração precisa destacar a semântica at-least-once e a necessidade de deduplicação pelo identificador de evento. Não bloqueia o código, mas é a única mitigação acordada para o risco de processamento duplicado no cliente.

---

### Riscos e mitigação

#### Worker único é ponto único de falha e teto de throughput
- **Probabilidade:** media
- **Impacto:** alto. Worker parado significa nenhum cliente notificado, e o limite de 10 segundos passa a ser violado sem que ninguém perceba
- **Mitigação:**
  - A outbox é durável, portanto eventos não se perdem, apenas atrasam
  - O worker roda em processo separado, de modo que reinícios da API não o derrubam
- **Plano de contingência:** reinício manual do processo, seguido de reprocessamento natural do backlog, já que a outbox preserva os eventos pendentes e o worker retoma de onde parou. Definir health check, alerta de fila parada e política de reinício automático é ação de fechamento sob a engenharia de plataforma, antes do deploy

#### Mecanismo de assinatura durante o grace period de rotação não está definido
- **Probabilidade:** alta
- **Impacto:** alto. Uma implementação equivocada quebra a entrega para o cliente ou mantém válida uma secret comprometida
- **Mitigação:**
  - É exatamente o escopo da revisão de segurança já prevista
  - Fechar a definição na sessão de revisão do design, antes de codificar
- **Plano de contingência:** entregar a primeira versão com secret fixa por endpoint e sem rotação por API, mantendo a geração pela plataforma, e liberar a rotação em incremento posterior assim que o mecanismo for aprovado pela engenharia de segurança

#### Ausência de isolamento entre customers no CRUD de webhook
- **Probabilidade:** media
- **Impacto:** alto. Dados de pedidos de um cliente podem ser redirecionados para o endpoint de outro
- **Mitigação:**
  - Endurecimento de autorização já sinalizado como evolução prevista
  - Levar o ponto explicitamente para a revisão de segurança
- **Plano de contingência:** restringir todo o CRUD de webhook ao papel ADMIN até que exista controle de isolamento por customer, reduzindo a superfície ao custo de exigir intervenção administrativa em cada cadastro. Este é o ponto aberto de segurança mais relevante do documento

#### Evento preso no estado processando quando o worker morre no meio do envio
- **Probabilidade:** media
- **Impacto:** médio. O evento nunca é entregue e nunca chega à dead letter queue, permanecendo invisível
- **Mitigação:**
  - Hipótese de rotina de recuperação por tempo, devolvendo ao estado pendente os eventos travados além do timeout de envio
  - O risco de reenvio decorrente já é absorvido pela semântica at-least-once e pelo identificador de evento
- **Plano de contingência:** replay manual, que hoje cobre apenas a dead letter queue e não a outbox travada

#### Falha definitiva de entrega passa despercebida
- **Probabilidade:** alta
- **Impacto:** médio. O cliente fica sem eventos e ninguém percebe até ele reclamar
- **Mitigação:**
  - A dead letter queue persiste payload, motivo e timestamp para diagnóstico
  - O histórico de entregas expõe as falhas ao próprio cliente
- **Plano de contingência:** replay manual pelo endpoint administrativo. O alerta automático ao cliente está previsto para a fase seguinte

#### Prazo comercial com risco de perda de cliente
- **Probabilidade:** media
- **Impacto:** alto. O cliente que motivou a demanda sinalizou possibilidade de migrar para o concorrente
- **Mitigação:**
  - Ordem de construção sugerida, derivada da estimativa de esforço, priorizando outbox, worker e retry antes do CRUD e do histórico
  - Confirmação do prazo diretamente com o cliente
- **Plano de contingência:** corte de escopo priorizado, adiando o histórico de entregas do RF-10, de prioridade media, e preservando outbox, worker, retry, dead letter queue e assinatura, que são o núcleo do valor pedido pelo cliente

#### Worker sem tratamento de erro de topo
- **Probabilidade:** media
- **Impacto:** alto. Uma exceção não tratada encerra o processo e recai no risco de worker parado
- **Mitigação:**
  - Definir o tratamento na sessão de revisão do design, já que o middleware de erro existente cobre apenas o ciclo HTTP
- **Plano de contingência:** supervisor externo reinicia o processo em caso de encerramento inesperado, e a durabilidade da outbox garante a retomada sem perda de eventos

#### Cliente não implementa deduplicação por identificador de evento
- **Probabilidade:** media
- **Impacto:** médio. Processamento duplicado no lado do cliente
- **Mitigação:**
  - Documentação destacada no portal de desenvolvedor
  - Aderência a um padrão de mercado já familiar para integradores
- **Plano de contingência:** nenhum plano técnico. Entrega exatamente uma vez foi descartada

#### Crescimento descontrolado das tabelas de eventos
- **Probabilidade:** media
- **Impacto:** médio. Degradação progressiva do polling do worker
- **Mitigação:**
  - Índices em status e data de criação, com leitura restrita a pendentes em batch pequeno
  - Payload enxuto, sem os itens do pedido, e teto de 64KB
- **Plano de contingência:** rotina de arquivamento das linhas entregues, já mapeada e explicitamente fora do escopo desta entrega

#### Insert na outbox amplia a fronteira de uma transação já pesada
- **Probabilidade:** baixa
- **Impacto:** alto. Uma falha nesse ponto derruba a mudança de status, que é operação central do negócio
- **Mitigação:**
  - O insert é local e não envolve rede
  - O filtro na inserção evita gravar linhas desnecessárias
  - A função recebe o transaction client, mantendo o acoplamento mínimo
- **Plano de contingência:** o rollback é o comportamento desejado e decidido, não um incidente a ser remediado

#### Ausência de limitação de taxa na saída
- **Probabilidade:** media
- **Impacto:** médio. Uma rajada de mudanças de status vira uma rajada equivalente de chamadas ao cliente
- **Mitigação:**
  - Decisão consciente de observar antes de implementar
- **Plano de contingência:** implementar na fase seguinte. O backoff do retry absorve parte do efeito caso o cliente passe a recusar as chamadas

#### Ordenação se quebra silenciosamente com uma segunda instância do worker
- **Probabilidade:** baixa
- **Impacto:** médio. O cliente pode receber transições fora de ordem
- **Mitigação:**
  - Registrar em documentação como limitação conhecida, e não como garantia
  - Confirmado que os clientes não pediram ordenação global
- **Plano de contingência:** particionamento por pedido ou lock pessimista, adiado para quando houver necessidade de escala

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Mudança de status bem-sucedida com webhook assinante grava exatamente uma linha na outbox, na mesma transação
- Falha forçada no insert da outbox provoca rollback: a order permanece no status anterior e nenhuma linha nova aparece no histórico de status
- Mudança para status que nenhum webhook do customer assina não gera linha na outbox
- A linha da outbox tem identificador UUID e payload renderizado na inserção, e alterar o pedido depois não altera o payload gravado
- O worker sobe por `npm run worker`, em processo distinto da API e com client de banco próprio
- Reiniciar a API não interrompe o processamento de eventos
- Evento pendente é enviado em no máximo 2 segundos após o commit, no pior caso
- Endpoint que não responde em 10 segundos é tratado como falha e marcado para nova tentativa
- Múltiplos eventos do mesmo pedido são processados na ordem de `created_at` enquanto houver um único worker, ressalvado que entregas em ciclo de retry podem chegar fora de ordem
- Falha de entrega gera nova tentativa após 1 minuto, depois 5 minutos, 30 minutos, 2 horas e 12 horas, totalizando 5 tentativas
- Esgotadas as tentativas, o evento aparece na dead letter queue com payload, motivo e timestamp e deixa de ser reentregue pelo worker
- O endpoint de replay com token de papel ADMIN recoloca o evento na outbox como pendente
- O mesmo endpoint com papel operacional retorna 403
- O replay gera log identificando quem o executou
- Todo envio carrega os headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e Content-Type application/json
- O header de assinatura é um HMAC-SHA256 do corpo exato do request, verificável com a secret do endpoint
- Dois webhooks distintos possuem secrets distintas
- Cadastro com url http é rejeitado com erro de validação
- Após a rotação, o cliente consegue validar os envios usando a secret anterior durante 24 horas, conforme o mecanismo definido na revisão de segurança, e deixa de conseguir após esse prazo
- Payload acima de 64KB não é enviado e produz erro, sem truncamento
- A secret é gerada pela plataforma e devolvida na resposta de criação
- O CRUD completo está disponível e autenticado por JWT, com criação, edição, remoção e listagem por customer
- O identificador do customer é aceito no corpo ou no caminho da requisição e não é lido do token
- O endpoint de histórico retorna, por entrega, sucesso ou falha, payload, response e tempo de resposta
- O payload contém identificador do evento, tipo do evento, timestamp em ISO 8601, identificador e número do pedido, status de origem e destino, identificador do cliente e valor total, e não contém os itens do pedido
- Todo erro específico do domínio de webhooks usa o prefixo `WEBHOOK_`, incluindo ao menos os de recurso não encontrado, url inválida e secret obrigatória, sem alterar os códigos transversais já existentes como `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN` e `NOT_FOUND`
- Os erros do módulo são respondidos pelo error middleware existente, no formato de erro já padronizado, sem alteração no middleware
- O módulo vive em `src/modules/webhooks` com controller, service, repository, routes e schemas
- Nenhuma dependência de log nova é adicionada
- Nenhum serviço novo de infraestrutura é adicionado ao ambiente
- A revisão de segurança sobre HMAC e geração de secret está concluída antes do deploy

---

### Testes e validação

Tipos de teste obrigatórios
- Testes de integração de API com Supertest contra o app e o banco reais, no padrão já consolidado no repositório, cobrindo CRUD, rejeição de url http, recurso não encontrado, ausência de token, replay negado para papel operacional e histórico de entregas
- Teste transacional do `changeStatus`, verificando a gravação na outbox no caminho feliz e o rollback completo quando o insert da outbox falha
- Testes unitários de assinatura, verificando reprodutibilidade do HMAC, unicidade da secret por endpoint e comportamento durante o grace period de rotação
- Testes do worker cobrindo a progressão do backoff, o encerramento após a quinta tentativa, a transição para a dead letter queue e o timeout de envio tratado como falha, com controle de tempo simulado
- Testes de filtro de assinatura e de snapshot do payload, verificando que status não assinado não gera linha e que alterações posteriores no pedido não modificam o payload gravado
- Teste ponta a ponta da feature, da mudança de status até o recebimento do POST assinado por um servidor de teste e o registro da entrega

Estratégia de validação
- Validação em quatro portões, na ordem definida: testes automatizados no padrão do repositório, executados de forma serial contra banco real; sessão de revisão do documento de design com os engenheiros de plataforma e de pedidos antes de codificar, onde as lacunas em aberto precisam ser fechadas; revisão de segurança sobre HMAC e geração de secret, portão obrigatório antes do deploy; e validação com o cliente real que motivou a demanda
- Hipótese não decidida: habilitar a feature primeiro para um único cliente antes dos três, reduzindo a exposição das lacunas operacionais ainda abertas
- Não definidos na reunião: teste de carga, ambiente de homologação, rollout gradual ou feature flag, e endpoint de envio de evento de teste para o cliente validar a própria integração
