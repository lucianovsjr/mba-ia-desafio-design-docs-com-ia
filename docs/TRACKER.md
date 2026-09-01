# Tracker de Rastreabilidade

Referência cruzada entre os itens registrados nos documentos de design e sua origem na transcrição da reunião ou no código da aplicação.

Convenções:

- **Fonte** `TRANSCRICAO` refere-se a `TRANSCRICAO.md`, com timestamp e nome do falante.
- **Fonte** `CODIGO` refere-se ao código da aplicação existente, com caminho do arquivo.
- Itens marcados no PRD como hipótese ou inferência aparecem aqui com o sufixo `(hipótese)` no resumo, e sua Localização aponta para a origem parcial que os motivou.

Cobertura atual: `docs/PRD.md`. As linhas de RFC, FDD e ADRs serão acrescentadas conforme esses documentos forem produzidos.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Eliminar dependência de polling caro/lento no `GET /orders`, entregando notificação em até 10s (limiar definido pelos clientes) | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | docs/PRD.md | Contexto | Motivação comercial: Atlas Comercial sinalizou risco de migrar para concorrente se a feature não for entregue | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-01 | docs/PRD.md | Contexto | Público-alvo: clientes B2B integradores (Atlas Comercial, MaxDistribuição, Nova Cargo) | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | docs/PRD.md | Contexto | Operador interno com papel ADMIN responsável por reprocessar manualmente eventos da dead letter queue | TRANSCRICAO | `[09:36] Larissa` |
| PRD-CTX-03 | docs/PRD.md | Contexto | Cenário: cliente cadastra endpoint de webhook e recebe notificação em vez de fazer polling | TRANSCRICAO | `[09:31] Marcos` |
| PRD-CTX-04 | docs/PRD.md | Contexto | Cenário: cliente restringe notificação aos status de interesse, ex. SHIPPED e DELIVERED | TRANSCRICAO | `[09:33] Marcos` |
| PRD-CTX-05 | docs/PRD.md | Contexto | Cenário: cliente consulta histórico de entregas dos webhooks para auditar sua integração | TRANSCRICAO | `[09:34] Marcos` |
| PRD-CTX-06 | docs/PRD.md | Contexto | Cenário: operador ADMIN reprocessa manualmente evento que caiu na dead letter queue | TRANSCRICAO | `[09:18] Diego` |
| PRD-CTX-07 | docs/PRD.md | Contexto | Feature entra como novo módulo `src/modules/webhooks`, seguindo padrão da codebase | TRANSCRICAO | `[09:27] Bruno` |
| PRD-CTX-08 | docs/PRD.md | Contexto | Alteração no `changeStatus` (update `orders`, insert `order_status_history`, ajuste `stock_quantity`) | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-CTX-08a | docs/PRD.md | Contexto | Alteração no `changeStatus` de `order.service.ts` como ponto de integração, confirmada em reunião | TRANSCRICAO | `[09:40] Bruno` |
| PRD-CTX-09 | docs/PRD.md | Contexto | Worker roda como processo separado, com entry point próprio ao lado do `src/server.ts` | TRANSCRICAO | `[09:11] Larissa` |
| PRD-PROB-01 | docs/PRD.md | Problema | Polling no `GET /orders` deixa integração lenta, sem aviso de mudança; prioridade alta (hipótese) | TRANSCRICAO | `[09:00] Marcos` |
| PRD-PROB-02 | docs/PRD.md | Problema | Polling é caro para o cliente, requisições repetidas sem garantia de mudança; prioridade alta (hipótese) | TRANSCRICAO | `[09:00] Marcos` |
| PRD-PROB-03 | docs/PRD.md | Contexto | Não existe número medido de custo, volume de polling ou tempo perdido; fonte só traz impacto qualitativo e risco comercial | TRANSCRICAO | `[09:00] Marcos` |
| PRD-MET-01 | docs/PRD.md | Requisito Não Funcional | Meta: notificação em menos de 10s entre commit da mudança de status e chegada no endpoint do cliente | TRANSCRICAO | `[09:02] Marcos` |
| PRD-MET-02 | docs/PRD.md | Requisito Não Funcional | Meta: 5 tentativas de entrega cobrindo cerca de 15h de backoff | TRANSCRICAO | `[09:17] Diego` |
| PRD-MET-03 | docs/PRD.md | Requisito Não Funcional | Meta: 100% de eventos gravados na outbox garantido por transação | TRANSCRICAO | `[09:06] Diego` |
| PRD-MET-04 | docs/PRD.md | Restrição | Meta: entregar a feature sem provisionar infraestrutura nova (zero novos componentes) | TRANSCRICAO | `[09:07] Diego` |
| PRD-ESC-01 | docs/PRD.md | Escopo | Tabela `webhook_outbox` no MySQL, populada na mesma transação do `changeStatus` | TRANSCRICAO | `[09:06] Diego` |
| PRD-ESC-02 | docs/PRD.md | Escopo | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada pelo `order.service.ts` com client da transação | TRANSCRICAO | `[09:41] Bruno` |
| PRD-ESC-03 | docs/PRD.md | Escopo | Filtro de status de interesse por webhook, aplicado na inserção na outbox | TRANSCRICAO | `[09:34] Bruno` |
| PRD-ESC-04a | docs/PRD.md | Escopo | Worker em processo separado, em polling a cada 2s sobre eventos pendentes mais antigos | TRANSCRICAO | `[09:09] Diego` |
| PRD-ESC-04b | docs/PRD.md | Escopo | Worker com PrismaClient próprio, separado do processo da API | TRANSCRICAO | `[09:30] Bruno` |
| PRD-ESC-05 | docs/PRD.md | Escopo | Montagem/envio do payload JSON com headers padronizados, timeout de 10s por requisição | TRANSCRICAO | `[09:42] Diego` |
| PRD-ESC-06 | docs/PRD.md | Escopo | Retry com backoff exponencial 1min/5min/30min/2h/12h, totalizando 5 tentativas | TRANSCRICAO | `[09:17] Diego` |
| PRD-ESC-07 | docs/PRD.md | Escopo | Tabela `webhook_dead_letter` separada, com payload, motivo da falha e timestamp | TRANSCRICAO | `[09:18] Diego` |
| PRD-ESC-08a | docs/PRD.md | Escopo | Endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | `[09:18] Diego` |
| PRD-ESC-08b | docs/PRD.md | Escopo | Endpoint de replay restrito ao papel ADMIN | TRANSCRICAO | `[09:36] Larissa` |
| PRD-ESC-08c | docs/PRD.md | Escopo | Replay administrativo com log de auditoria de quem executou | TRANSCRICAO | `[09:36] Sofia` |
| PRD-ESC-09a | docs/PRD.md | Escopo | Assinatura HMAC-SHA256 do corpo da requisição | TRANSCRICAO | `[09:20] Sofia` |
| PRD-ESC-09b | docs/PRD.md | Escopo | Secret única por endpoint, gerada pela plataforma (não secret global) | TRANSCRICAO | `[09:21] Sofia` |
| PRD-ESC-10 | docs/PRD.md | Escopo | Rotação de secret pela API, com secret anterior válida por 24h | TRANSCRICAO | `[09:21] Sofia` |
| PRD-ESC-11 | docs/PRD.md | Escopo | Validação de URL obrigatoriamente HTTPS via schema Zod | TRANSCRICAO | `[09:23] Sofia` |
| PRD-ESC-12 | docs/PRD.md | Escopo | Limite de payload de 64 KB, com erro quando ultrapassado | TRANSCRICAO | `[09:24] Diego` |
| PRD-ESC-13 | docs/PRD.md | Escopo | Entrega at-least-once com `X-Event-Id` em UUID para deduplicação pelo cliente | TRANSCRICAO | `[09:25] Diego` |
| PRD-ESC-14 | docs/PRD.md | Escopo | CRUD de configuração de webhook, autenticado com o JWT já existente | TRANSCRICAO | `[09:32] Marcos` |
| PRD-ESC-15 | docs/PRD.md | Escopo | Consulta dos últimos 100 registros de entrega, com status, payload, resposta e tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| PRD-ESC-16a | docs/PRD.md | Escopo | Módulo `src/modules/webhooks` reusando `AppError` e prefixo de erro `WEBHOOK_` | TRANSCRICAO | `[09:28] Bruno` |
| PRD-ESC-16b | docs/PRD.md | Escopo | Módulo reusando logger Pino e error middleware já existentes | TRANSCRICAO | `[09:29] Bruno` |
| PRD-FESC-01 | docs/PRD.md | Fora de escopo | Notificação por e-mail quando webhook falha repetidamente, adiado para fase seguinte | TRANSCRICAO | `[09:37] Larissa` |
| PRD-FESC-02 | docs/PRD.md | Fora de escopo | Dashboard visual para o cliente, adiado por ser projeto separado do frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-FESC-03 | docs/PRD.md | Fora de escopo | Arquivamento das linhas entregues da outbox após cerca de 30 dias, reconhecido como necessário mas fora da entrega | TRANSCRICAO | `[09:08] Diego` |
| PRD-FESC-04 | docs/PRD.md | Fora de escopo | Escala para múltiplos workers, com particionamento por `order_id` ou lock pessimista, adiado | TRANSCRICAO | `[09:13] Diego` |
| PRD-FESC-05 | docs/PRD.md | Fora de escopo | Endurecimento das permissões do CRUD de webhook, hoje aberto a qualquer papel autenticado, adiado | TRANSCRICAO | `[09:37] Sofia` |
| PRD-FESC-06 | docs/PRD.md | Fora de escopo | Webhook síncrono dentro da transação, descartado por travar mudança de status e não haver critério de rollback | TRANSCRICAO | `[09:04] Bruno` |
| PRD-FESC-07 | docs/PRD.md | Fora de escopo | Fila Redis Streams no lugar da outbox, descartada por exigir infraestrutura nova/overengineering | TRANSCRICAO | `[09:07] Diego` |
| PRD-FESC-08 | docs/PRD.md | Fora de escopo | Trigger de banco para acionar worker, descartada porque MySQL não notifica processo externo | TRANSCRICAO | `[09:09] Diego` |
| PRD-FESC-09 | docs/PRD.md | Fora de escopo | Retry com 3 tentativas em janela curta, descartado por matar evento durante indisponibilidade planejada | TRANSCRICAO | `[09:16] Diego` |
| PRD-FESC-10 | docs/PRD.md | Fora de escopo | Retry indefinido, descartado por deixar evento pendurado para sempre | TRANSCRICAO | `[09:15] Diego` |
| PRD-FESC-11 | docs/PRD.md | Fora de escopo | Truncamento do payload acima de 64 KB, descartado em favor de erro explícito | TRANSCRICAO | `[09:23] Sofia` |
| PRD-FESC-12 | docs/PRD.md | Fora de escopo | Dead letter como campo de estado na outbox, descartada em favor de tabela separada | TRANSCRICAO | `[09:18] Diego` |
| PRD-OPEN-01 | docs/PRD.md | Risco | Rate limiting de envio de webhooks quando muitos eventos disparam em pouco tempo, sem decisão ("observar e decidir depois") | TRANSCRICAO | `[09:39] Larissa` |
| PRD-OPEN-02 | docs/PRD.md | Dependência | Qual secret o worker usa para assinar durante o grace period de 24h não foi definida | TRANSCRICAO | `[09:21] Sofia` |
| PRD-OPEN-05 | docs/PRD.md | Contexto | Ambiguidade se os 5 intervalos de backoff correspondem a 5 tentativas com 4 esperas ou a 1 tentativa + 5 reagendamentos, não resolvida na reunião | TRANSCRICAO | `[09:17] Diego` |
| PRD-FR-01a | docs/PRD.md | Requisito Funcional | RF-01: `POST /webhooks` com URL, status de interesse e `customer_id` no corpo; secret gerada e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-01b | docs/PRD.md | Requisito Funcional | RF-01: `GET` lista, `PATCH` edita URL/status/estado ativo, `DELETE` remove | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-01c | docs/PRD.md | Requisito Funcional | RF-01: URL em HTTP recusada na validação do schema, só HTTPS aceito | TRANSCRICAO | `[09:23] Sofia` |
| PRD-FR-01d | docs/PRD.md | Requisito Funcional | RF-01: CRUD aceita qualquer papel autenticado nesta entrega, sem restrição por papel | TRANSCRICAO | `[09:37] Sofia` |
| PRD-FR-01e | docs/PRD.md | Requisito Funcional | RF-01: `customer_id` vem do corpo/path, não do JWT (JWT é do usuário operador) | TRANSCRICAO | `[09:32] Larissa` |
| PRD-FR-01f | docs/PRD.md | Decisão | RF-01: prioridade alta (hipótese, prioridade não classificada formalmente pela fonte) | TRANSCRICAO | `[09:31] Larissa` |
| PRD-FR-02a | docs/PRD.md | Requisito Funcional | RF-02: cliente solicita rotação, nova secret gerada, anterior válida 24h em paralelo, nova devolvida na resposta | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-02b | docs/PRD.md | Contexto | RF-02: qual secret é usada para assinar durante o grace period não foi definida pela fonte | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-02c | docs/PRD.md | Decisão | RF-02: prioridade alta (hipótese) | TRANSCRICAO | `[09:31] Larissa` |
| PRD-FR-03a | docs/PRD.md | Requisito Funcional | RF-03: `changeStatus` abre transação, executa update `orders`/insert `order_status_history`/ajuste estoque já existentes | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-FR-03b | docs/PRD.md | Requisito Funcional | RF-03: service chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` com client da transação | TRANSCRICAO | `[09:41] Bruno` |
| PRD-FR-03c | docs/PRD.md | Requisito Funcional | RF-03: payload renderizado como snapshot no momento da inserção, não recalculado no envio | TRANSCRICAO | `[09:52] Larissa` |
| PRD-FR-03d | docs/PRD.md | Requisito Funcional | RF-03: se nenhum webhook inscrito no status, nada é inserido, comportamento esperado | TRANSCRICAO | `[09:34] Bruno` |
| PRD-FR-03e | docs/PRD.md | Requisito Funcional | RF-03: falha na inserção causa rollback de toda a transação, incluindo a mudança de status | TRANSCRICAO | `[09:40] Bruno` |
| PRD-FR-03f | docs/PRD.md | Decisão | RF-03: prioridade alta (hipótese) | TRANSCRICAO | `[09:31] Larissa` |
| PRD-FR-04a | docs/PRD.md | Requisito Funcional | RF-04: worker consulta a cada 2s eventos pendentes ordenados por `created_at`, em lotes pequenos | TRANSCRICAO | `[09:09] Diego` |
| PRD-FR-04b | docs/PRD.md | Requisito Funcional | RF-04: payload com `event_id`, `event_type` `order.status_changed`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`, sem itens | TRANSCRICAO | `[09:43] Diego` |
| PRD-FR-04c1 | docs/PRD.md | Requisito Funcional | RF-04: headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type: application/json` | TRANSCRICAO | `[09:44] Diego` |
| PRD-FR-04c2 | docs/PRD.md | Requisito Funcional | RF-04: header `X-Webhook-Id` com o id do endpoint webhook | TRANSCRICAO | `[09:44] Sofia` |
| PRD-FR-04d | docs/PRD.md | Requisito Não Funcional | RF-04: timeout de 10s por requisição | TRANSCRICAO | `[09:42] Diego` |
| PRD-FR-04e | docs/PRD.md | Requisito Funcional | RF-04: payload acima de 64 KB não é enviado, erro registrado sem truncamento | TRANSCRICAO | `[09:23] Sofia` |
| PRD-FR-04f | docs/PRD.md | Requisito Funcional | RF-04: entrega at-least-once, cliente deduplica pelo `X-Event-Id` | TRANSCRICAO | `[09:25] Diego` |
| PRD-FR-04g | docs/PRD.md | Decisão | RF-04: prioridade alta (hipótese) | TRANSCRICAO | `[09:31] Larissa` |
| PRD-FR-05a | docs/PRD.md | Requisito Funcional | RF-05: backoff 1min/5min/30min/2h/12h, 5 tentativas | TRANSCRICAO | `[09:17] Diego` |
| PRD-FR-05b | docs/PRD.md | Requisito Funcional | RF-05: falhando a 5ª tentativa, evento movido para `webhook_dead_letter` com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-05d | docs/PRD.md | Decisão | RF-05: prioridade alta (hipótese) | TRANSCRICAO | `[09:31] Larissa` |
| PRD-FR-06a | docs/PRD.md | Requisito Funcional | RF-06: secret única por endpoint (não global) gerada no cadastro | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-06b | docs/PRD.md | Requisito Funcional | RF-06: worker calcula HMAC-SHA256 sobre o corpo com a secret ativa, envia em `X-Signature` | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-06c | docs/PRD.md | Decisão | RF-06: prioridade alta (hipótese) | TRANSCRICAO | `[09:31] Larissa` |
| PRD-FR-07a | docs/PRD.md | Requisito Funcional | RF-07: `POST /admin/webhooks/dead-letter/:id/replay` recoloca evento como pendente na outbox | TRANSCRICAO | `[09:35] Diego` |
| PRD-FR-07b | docs/PRD.md | Requisito Funcional | RF-07: middleware `requireRole` restringe o acesso ao papel ADMIN | TRANSCRICAO | `[09:36] Larissa` |
| PRD-FR-07b-cod | docs/PRD.md | Requisito Funcional | RF-07: `requireRole` já existe no código | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-FR-07c | docs/PRD.md | Requisito Funcional | RF-07: execução do replay registrada em log identificando quem executou | TRANSCRICAO | `[09:36] Sofia` |
| PRD-FR-07d | docs/PRD.md | Decisão | RF-07: prioridade média (hipótese) | TRANSCRICAO | `[09:31] Larissa` |
| PRD-FR-08a | docs/PRD.md | Requisito Funcional | RF-08: `GET /webhooks/:id/deliveries` retorna últimos 100 registros com sucesso/falha, payload, resposta, tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-08b | docs/PRD.md | Decisão | RF-08: prioridade média (hipótese) | TRANSCRICAO | `[09:31] Larissa` |
| PRD-NFR-PERF-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega menor que 10s, limiar definido pelos clientes | TRANSCRICAO | `[09:02] Marcos` |
| PRD-NFR-PERF-02 | docs/PRD.md | Requisito Não Funcional | Intervalo de polling do worker de 2s, pior caso de latência de disparo | TRANSCRICAO | `[09:09] Diego` |
| PRD-NFR-PERF-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por requisição HTTP ao endpoint do cliente | TRANSCRICAO | `[09:42] Diego` |
| PRD-NFR-PERF-04 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64 KB por evento | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-PERF-05 | docs/PRD.md | Requisito Não Funcional | Worker processa eventos pendentes em lotes pequenos; tamanho exato não definido pela fonte | TRANSCRICAO | `[09:08] Diego` |
| PRD-NFR-DISP-02 | docs/PRD.md | Requisito Não Funcional | Worker roda em processo separado da API para não ser derrubado quando a API reinicia | TRANSCRICAO | `[09:11] Diego` |
| PRD-NFR-DISP-03 | docs/PRD.md | Contexto | Comportamento esperado se o worker parar por completo (restart, health check, alerta) não definido (hipótese) | TRANSCRICAO | `[09:11] Diego` |
| PRD-NFR-SEG-01 | docs/PRD.md | Requisito Não Funcional | Autenticação por JWT do próprio sistema, mesmo padrão da API | TRANSCRICAO | `[09:32] Marcos` |
| PRD-NFR-SEG-02 | docs/PRD.md | Requisito Funcional | CRUD de configuração aceita qualquer papel autenticado, condição provisória | TRANSCRICAO | `[09:37] Sofia` |
| PRD-NFR-SEG-03 | docs/PRD.md | Requisito Não Funcional | Replay exige o papel ADMIN, aplicado pelo middleware `requireRole` | TRANSCRICAO | `[09:36] Larissa` |
| PRD-NFR-SEG-04 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint, gerada pela plataforma, rotacionável, anterior válida 24h | TRANSCRICAO | `[09:21] Sofia` |
| PRD-NFR-SEG-05 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 sobre o corpo da requisição, transmitida em `X-Signature` | TRANSCRICAO | `[09:20] Sofia` |
| PRD-NFR-SEG-06 | docs/PRD.md | Requisito Não Funcional | URL do webhook obrigatoriamente HTTPS, validada por schema Zod, recusa em HTTP | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-SEG-07 | docs/PRD.md | Requisito Não Funcional | Log de auditoria identificando quem executou o replay administrativo | TRANSCRICAO | `[09:36] Sofia` |
| PRD-NFR-SEG-08 | docs/PRD.md | Dependência | Revisão de segurança do código antes do deploy, pelo menos 2 dias úteis, focada em HMAC e geração de secret | TRANSCRICAO | `[09:46] Sofia` |
| PRD-NFR-OBS-01 | docs/PRD.md | Requisito Não Funcional | Logging pelo Pino já presente no projeto, sem introduzir ferramenta nova | TRANSCRICAO | `[09:29] Bruno` |
| PRD-NFR-OBS-02 | docs/PRD.md | Requisito Funcional | Histórico de entregas exposto ao cliente (resultado, payload, resposta, tempo de resposta) dos últimos 100 envios | TRANSCRICAO | `[09:34] Marcos` |
| PRD-NFR-CONF-01 | docs/PRD.md | Requisito Não Funcional | Inserção do evento na outbox na mesma transação SQL da mudança de status, com rollback conjunto | TRANSCRICAO | `[09:06] Diego` |
| PRD-NFR-CONF-02 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once, e não exactly-once | TRANSCRICAO | `[09:25] Diego` |
| PRD-NFR-CONF-03 | docs/PRD.md | Requisito Não Funcional | Deduplicação de responsabilidade do cliente, com base no `X-Event-Id` em UUID | TRANSCRICAO | `[09:25] Diego` |
| PRD-NFR-CONF-04 | docs/PRD.md | Restrição | Ordenação garantida por `order_id` apenas com worker único; sem garantia de ordenação global | TRANSCRICAO | `[09:12] Diego` |
| PRD-NFR-CONF-05 | docs/PRD.md | Requisito Não Funcional | Payload gravado como snapshot no momento da inserção na outbox, não recalculado no envio | TRANSCRICAO | `[09:52] Larissa` |
| PRD-NFR-CONF-06 | docs/PRD.md | Requisito Não Funcional | Identificadores em UUID, seguindo o padrão do restante do projeto | TRANSCRICAO | `[09:51] Larissa` |
| PRD-NFR-COMPAT-01 | docs/PRD.md | Requisito Não Funcional | Payload em JSON com campos fixos, `timestamp` em ISO 8601 e `Content-Type: application/json` | TRANSCRICAO | `[09:43] Diego` |
| PRD-NFR-COMPAT-02 | docs/PRD.md | Requisito Não Funcional | Worker como entry point Node separado, mesma stack, com instância própria de PrismaClient, mesmo banco | TRANSCRICAO | `[09:30] Bruno` |
| PRD-NFR-COMPAT-03 | docs/PRD.md | Requisito Não Funcional | Nenhuma comunicação direta entre API e worker, que se sincronizam apenas pelo banco | TRANSCRICAO | `[09:11] Diego` |
| PRD-NFR-COMPL-01 | docs/PRD.md | Requisito Não Funcional | Trilha de auditoria do replay administrativo, registrando quem executou a ação | TRANSCRICAO | `[09:36] Sofia` |
| PRD-NFR-COMPL-02 | docs/PRD.md | Fora de escopo | Arquivamento das linhas entregues da outbox após cerca de 30 dias, reconhecido como necessário mas fora desta entrega | TRANSCRICAO | `[09:08] Diego` |
| PRD-ARQ-01 | docs/PRD.md | Decisão | Transactional outbox + worker separado + HTTP com retry/backoff/DLQ/HMAC, evitando disparo síncrono e infra nova | TRANSCRICAO | `[09:06] Diego` |
| PRD-ARQ-02 | docs/PRD.md | Contexto | Tabela `webhook_outbox` com payload snapshot, estado, `created_at`, `order_id`, índices sobre estado e data | TRANSCRICAO | `[09:08] Diego` |
| PRD-ARQ-03 | docs/PRD.md | Contexto | Função `publishWebhookEvent` recebe client da transação, aplica filtro de status, insere o evento | TRANSCRICAO | `[09:41] Bruno` |
| PRD-ARQ-04 | docs/PRD.md | Contexto | `OrderService.changeStatus` passa a chamar a publicação do evento dentro da própria transação | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-ARQ-05 | docs/PRD.md | Contexto | Módulo `src/modules/webhooks` com controller, service, repository, routes e schemas | TRANSCRICAO | `[09:27] Bruno` |
| PRD-ARQ-06 | docs/PRD.md | Contexto | Worker em processo Node separado, com entry point próprio e script dedicado | TRANSCRICAO | `[09:11] Larissa` |
| PRD-ARQ-07 | docs/PRD.md | Contexto | Tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp | TRANSCRICAO | `[09:18] Diego` |
| PRD-ARQ-08 | docs/PRD.md | Contexto | Endpoint administrativo de replay, restrito ao papel ADMIN | TRANSCRICAO | `[09:36] Larissa` |
| PRD-ARQ-09 | docs/PRD.md | Contexto | Componentes reaproveitados sem alteração: `AppError`, prefixo `WEBHOOK_`, Pino, error middleware, `requireRole` | TRANSCRICAO | `[09:30] Larissa` |
| PRD-ARQ-09-cod | docs/PRD.md | Contexto | `AppError` e `requireRole` existentes na codebase | CODIGO | `src/shared/errors/app-error.ts` |
| PRD-ARQ-10 | docs/PRD.md | Contexto | Mudança de status para outbox, por transação de banco, sem chamada HTTP intermediária | TRANSCRICAO | `[09:06] Diego` |
| PRD-ARQ-11 | docs/PRD.md | Contexto | Worker para banco de dados, por consultas SQL via Prisma a cada 2s, com instância própria de PrismaClient | TRANSCRICAO | `[09:30] Bruno` |
| PRD-ARQ-12 | docs/PRD.md | Contexto | Worker para endpoint do cliente por HTTP POST com timeout de 10s; única integração externa, exclusivamente outbound | TRANSCRICAO | `[09:03] Sofia` |
| PRD-ARQ-13 | docs/PRD.md | Contexto | Cliente para plataforma por endpoints REST autenticados com JWT, para configuração, rotação de secret e consulta de entregas | TRANSCRICAO | `[09:32] Larissa` |
| PRD-ARQ-15 | docs/PRD.md | Contexto | Estratégia de assinatura durante o grace period de rotação de secret ainda não fechada | TRANSCRICAO | `[09:21] Sofia` |
| PRD-DEC-01 | docs/PRD.md | Decisão | Adotar transactional outbox no MySQL, em vez de disparo síncrono ou fila dedicada | TRANSCRICAO | `[09:06] Diego` |
| PRD-DEC-01-TO | docs/PRD.md | Trade-off | Acopla gravação do evento a uma transação já pesada, em troca de consistência forte entre status e evento | TRANSCRICAO | `[09:04] Bruno` |
| PRD-DEC-02 | docs/PRD.md | Decisão | Worker executado como processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-DEC-02-TO | docs/PRD.md | Trade-off | Exige entry point e ciclo de deploy adicionais, gerenciamento de um segundo processo | TRANSCRICAO | `[09:11] Larissa` |
| PRD-DEC-03 | docs/PRD.md | Decisão | Worker em polling a cada 2s, sem mecanismo reativo | TRANSCRICAO | `[09:09] Diego` |
| PRD-DEC-03-TO | docs/PRD.md | Trade-off | Latência mínima de disparo passa a ser 2s no pior caso, custo aceito explicitamente | TRANSCRICAO | `[09:10] Larissa` |
| PRD-DEC-04 | docs/PRD.md | Decisão | HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h | TRANSCRICAO | `[09:22] Sofia` |
| PRD-DEC-04-TO | docs/PRD.md | Trade-off | Aumenta a complexidade de gestão de secrets e torna a assinatura um contrato difícil de alterar depois | TRANSCRICAO | `[09:21] Sofia` |
| PRD-DEC-05 | docs/PRD.md | Decisão | Garantia de entrega at-least-once, com deduplicação a cargo do cliente | TRANSCRICAO | `[09:25] Diego` |
| PRD-DEC-05-TO | docs/PRD.md | Trade-off | Transfere ao cliente a responsabilidade de deduplicar; cliente sem isso processa evento em duplicidade | TRANSCRICAO | `[09:25] Sofia` |
| PRD-DEC-06 | docs/PRD.md | Decisão | Ordenação garantida apenas por `order_id` e apenas em regime de worker único | TRANSCRICAO | `[09:12] Diego` |
| PRD-DEC-06-TO | docs/PRD.md | Trade-off | Garantia se perde ao escalar para múltiplos workers, exigindo particionamento ou lock pessimista | TRANSCRICAO | `[09:13] Diego` |
| PRD-DEC-07 | docs/PRD.md | Decisão | 5 tentativas de entrega com backoff exponencial de 1 minuto a 12 horas | TRANSCRICAO | `[09:17] Diego` |
| PRD-DEC-07-TO | docs/PRD.md | Trade-off | Evento pode levar até cerca de 15h para ser considerado perdido; número de tentativas já contestado internamente | TRANSCRICAO | `[09:16] Bruno` |
| PRD-DEC-08 | docs/PRD.md | Decisão | CRUD de configuração de webhook aberto a qualquer papel autenticado, com ADMIN exigido apenas no replay | TRANSCRICAO | `[09:36] Larissa` |
| PRD-DEC-08-TO | docs/PRD.md | Trade-off | Qualquer usuário autenticado pode alterar o destino dos webhooks de um customer; permissão registrada como provisória | TRANSCRICAO | `[09:37] Sofia` |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança antes do deploy, ao menos 2 dias úteis, foco em HMAC e geração de secrets | TRANSCRICAO | `[09:46] Sofia` |
| PRD-DEP-02 | docs/PRD.md | Dependência | Revisão do documento de design com o time de engenharia antes da implementação | TRANSCRICAO | `[09:50] Larissa` |
| PRD-DEP-03 | docs/PRD.md | Dependência | Comunicação e documentação aos clientes sobre prazo e comportamento at-least-once/deduplicação | TRANSCRICAO | `[09:26] Marcos` |
| PRD-DEP-04 | docs/PRD.md | Dependência | Nenhuma infraestrutura nova necessária; reaproveita MySQL/Prisma, `AppError`, error middleware, Pino, `requireRole` | TRANSCRICAO | `[09:07] Diego` |
| PRD-DEP-05 | docs/PRD.md | Dependência | Cliente precisa expor endpoint HTTPS, verificar HMAC, deduplicar por `X-Event-Id` e migrar secret em até 24h | TRANSCRICAO | `[09:25] Diego` |
| PRD-RISK-01 | docs/PRD.md | Risco | Endpoint do cliente indisponível/lento esgota as tentativas e o evento não é entregue; probabilidade média (hipótese) | TRANSCRICAO | `[09:15] Diego` |
| PRD-RISK-01-MIT | docs/PRD.md | Risco | Mitigação: retry com backoff cobrindo ~15h e timeout de 10s por tentativa | TRANSCRICAO | `[09:17] Diego` |
| PRD-RISK-01-PC | docs/PRD.md | Risco | Contingência: evento preservado na DLQ, reprocessável via replay administrativo | TRANSCRICAO | `[09:18] Diego` |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento da secret de um cliente permite forjar eventos assinados; probabilidade média (hipótese) | TRANSCRICAO | `[09:22] Diego` |
| PRD-RISK-02-MIT | docs/PRD.md | Risco | Mitigação: secret única por endpoint, rotação pela API, revisão de segurança antes do deploy | TRANSCRICAO | `[09:21] Sofia` |
| PRD-RISK-02-PC | docs/PRD.md | Risco | Contingência: cliente solicita rotação; secret comprometida invalida ao fim das 24h de transição | TRANSCRICAO | `[09:21] Sofia` |
| PRD-RISK-03 | docs/PRD.md | Risco | Escalar para múltiplos workers quebra a ordenação de eventos do mesmo pedido; probabilidade baixa (hipótese) | TRANSCRICAO | `[09:12] Diego` |
| PRD-RISK-03-MIT | docs/PRD.md | Risco | Mitigação: manter regime de worker único nesta entrega, documentar ausência de ordenação global | TRANSCRICAO | `[09:13] Larissa` |
| PRD-RISK-03-PC | docs/PRD.md | Risco | Contingência: implementar particionamento por `order_id` ou lock pessimista antes de segundo worker | TRANSCRICAO | `[09:13] Diego` |
| PRD-RISK-04 | docs/PRD.md | Risco | Volume alto de mudanças de status sobrecarrega o endpoint do cliente; probabilidade média (hipótese) | TRANSCRICAO | `[09:38] Diego` |
| PRD-RISK-04-MIT | docs/PRD.md | Risco | Mitigação: nenhuma implementada nesta entrega; decisão de observar o comportamento em produção | TRANSCRICAO | `[09:39] Larissa` |
| PRD-RISK-05 | docs/PRD.md | Risco | Cliente que não implementa deduplicação processa o mesmo evento mais de uma vez; probabilidade média (hipótese) | TRANSCRICAO | `[09:25] Sofia` |
| PRD-RISK-05-MIT | docs/PRD.md | Risco | Mitigação: `X-Event-Id` em UUID único por evento e documentação da garantia at-least-once | TRANSCRICAO | `[09:26] Marcos` |
| PRD-RISK-05-PC | docs/PRD.md | Risco | Contingência: sem ação corretiva da plataforma além de documentação e suporte; risco residual aceito conscientemente | TRANSCRICAO | `[09:25] Diego` |
| PRD-RISK-06 | docs/PRD.md | Risco | Falha na inserção do evento na outbox impede a mudança de status do pedido; probabilidade baixa (hipótese) | TRANSCRICAO | `[09:40] Bruno` |
| PRD-RISK-06-MIT | docs/PRD.md | Risco | Mitigação: inserção do evento na mesma transação SQL, com rollback conjunto; payload como snapshot | TRANSCRICAO | `[09:41] Diego` |
| PRD-TEST-01 | docs/PRD.md | Requisito Não Funcional | Testes de integração ponta a ponta com Vitest e Supertest, sem mock de banco (hipótese, matriz não definida pela fonte) | CODIGO | `tests/orders.test.ts` |
| PRD-TEST-02 | docs/PRD.md | Requisito Não Funcional | Teste de publicação do evento na outbox via `changeStatus`, incluindo rollback e caso sem webhook inscrito (hipótese) | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-TEST-03 | docs/PRD.md | Requisito Não Funcional | Teste de entrega verificando headers, corpo do payload e recálculo da assinatura HMAC-SHA256 (hipótese) | TRANSCRICAO | `[09:20] Sofia` |
| PRD-TEST-04 | docs/PRD.md | Requisito Não Funcional | Teste do ciclo de retry, do backoff e da movimentação para a dead letter queue (hipótese) | TRANSCRICAO | `[09:17] Diego` |
| PRD-TEST-05 | docs/PRD.md | Requisito Não Funcional | Teste de autorização do endpoint de replay administrativo (ADMIN e recusa sem esse papel) (hipótese) | TRANSCRICAO | `[09:36] Larissa` |
| PRD-TEST-06 | docs/PRD.md | Requisito Não Funcional | Convenção de teste do projeto: sobe a aplicação, popula por factories, valida status/corpo, limpeza de tabelas antes de cada execução | CODIGO | `tests/helpers/factories.ts` |
| PRD-TEST-07 | docs/PRD.md | Requisito Não Funcional | Teste do worker via função de processamento de lote testável isoladamente, em vez do loop completo (inferência de arquitetura, não decisão explícita da fonte) | TRANSCRICAO | `[09:09] Diego` |
| PRD-TEST-08 | docs/PRD.md | Dependência | Portão de aprovação obrigatório antes do deploy: revisão de segurança, 2 dias úteis, foco em HMAC e geração de secrets | TRANSCRICAO | `[09:46] Sofia` |
