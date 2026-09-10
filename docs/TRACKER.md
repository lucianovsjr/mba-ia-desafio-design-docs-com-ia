# Tracker de Rastreabilidade

Referência cruzada entre os itens registrados nos documentos de design e sua origem na transcrição da reunião ou no código da aplicação.

Convenções:

- **Fonte** `TRANSCRICAO` refere-se a `TRANSCRICAO.md`, com timestamp e nome do falante.
- **Fonte** `CODIGO` refere-se ao código da aplicação existente, com caminho do arquivo.
- Itens marcados no PRD como hipótese ou inferência aparecem aqui com o sufixo `(hipótese)` no resumo, e sua Localização aponta para a origem parcial que os motivou.

Cobertura atual: `docs/PRD.md`, `docs/adrs/ADR-001-outbox-transacional-no-mysql.md`, `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` (decisão central de processo separado/polling em `PRD-DEC-02`/`PRD-DEC-03`; linha própria `ADR-002-DEC-01` para a regra de entrega sequencial por `order_id` e concorrente entre pedidos distintos, resolução do ponto ARQ-03 do debate; risco em `ADR-002-OPEN-01`/`ADR-002-OPEN-02`), `docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md` (auditado: sem linha nova — a progressão de backoff, a tabela de dead letter e o endpoint de replay não acrescentam decisão além do que já está em `PRD-DEC-07`, `PRD-ESC-06`, `PRD-ESC-07`, `PRD-ESC-08a/b`, `PRD-FR-05a/b`, `PRD-FR-07a/b`; o próprio ADR delega a preservação do `X-Event-Id` no replay à ADR-005 e a trilha de auditoria à ADR-006), `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md`, `docs/adrs/ADR-005-entrega-at-least-once-com-id-de-evento.md` (decisão central de at-least-once em `PRD-DEC-05`; linha própria `ADR-005-DEC-01` para a preservação do `X-Event-Id` original no replay administrativo, resolução do ponto SEC-03 do debate; o agravante de duas instâncias de worker referencia `ADR-002-OPEN-01`), `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md`, `docs/RFC.md`, `docs/FDD.md`.

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
| ADR-001-DEC-01 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Posicionamento exato da publicação dentro da transação de `changeStatus`: depois do `tx.order.update` que grava o novo status, antes do fim da transação (após o insert de histórico e a releitura final com relações) | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-001-CONS-01 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Risco | `src/app.ts` precisa ser tocado para fiar a nova dependência na construção do `OrderService` (`buildControllers` monta `OrderService` por injeção manual), contrariando a leitura inicial de que a alteração ficaria isolada em `order.service.ts` | CODIGO | `src/app.ts` |
| ADR-002-OPEN-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Risco | Nenhum mecanismo técnico impede duas instâncias do worker rodando ao mesmo tempo; um claim atômico exigiria SQL cru (`UPDATE ... LIMIT` ou `SELECT ... FOR UPDATE SKIP LOCKED`), sem precedente em `src/`, que hoje só usa o Prisma Client; questão em aberto, não decidida neste ADR | CODIGO | `src/config/database.ts` |
| ADR-002-OPEN-02 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Risco | Mecanismo de teste do worker e do ciclo de retry (ferramenta de transporte ou simulação de tempo) não decidido; padrão atual do projeto é supertest contra a app real, sem nock/msw/fake timers; questão em aberto, não decidida neste ADR | CODIGO | `package.json` |
| ADR-002-DEC-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Dentro de um lote, entrega sequencial entre eventos do mesmo `order_id` e concorrente entre eventos de `order_id` distintos; resolução do ponto ARQ-03 do debate da RFC-001, derivada de PRD-DEC-06 | TRANSCRICAO | `[09:12] Diego` |
| ADR-004-DEC-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Redação do logger (`redactPaths`) passa a cobrir também a secret do webhook e a assinatura calculada; hoje a lista cobre só `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, sem alcançar secret ou assinatura | CODIGO | `src/shared/logger/index.ts` |
| ADR-004-OPEN-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Risco | Divergência não resolvida do debate (SEC-01, classificado DIVERGENTE) sobre incluir o timestamp no material assinado: hoje o HMAC cobre só o corpo (PRD-ESC-09a) e o `X-Timestamp` vai apenas como header, fora da assinatura, o que deixa uma entrega capturada potencialmente reproduzível indefinidamente; este ADR não resolve, mantém como questão em aberto da RFC | TRANSCRICAO | `[09:44] Diego` |
| ADR-004-DEC-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret do webhook devolvida apenas na resposta da criação e na da rotação, nunca nas respostas de leitura do CRUD; decisão do ponto SEC-07 do debate, derivada do propósito da secret única por endpoint, que existe para limitar o raio de um vazamento | TRANSCRICAO | `[09:21] Sofia` |
| ADR-006-DEC-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Classes de erro do módulo herdam das subclasses HTTP intermediárias quando existe equivalente (padrão real do projeto), e não de `AppError` diretamente: `InvalidStatusTransitionError` estende `ConflictError`, `InsufficientStockError` estende `UnprocessableEntityError`, ambas subclasses de `AppError` | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-DEC-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Dar identidade própria de serviço ao worker na saída de log exige parametrizar a factory `createLogger()` de `src/shared/logger/index.ts` para receber o nome do serviço; hoje ela não aceita parâmetro e fixa `service: 'order-management-api'` no campo `base`, então apenas trocar o singleton `logger` pela factory, sem parametrizá-la, não separa os dois processos | CODIGO | `src/shared/logger/index.ts` |
| ADR-006-DEC-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Trilha de auditoria do replay administrativo é peça nova do módulo de webhooks, não reuso: confirmada a ausência de qualquer model ou utilitário de auditoria reaproveitável em `prisma/schema.prisma` e em `src/` | CODIGO | `prisma/schema.prisma` |
| ADR-005-DEC-01 | docs/adrs/ADR-005-entrega-at-least-once-com-id-de-evento.md | Decisão | Replay administrativo de item da dead letter preserva o `X-Event-Id` original ao recolocar o evento como pendente na outbox, em vez de gerar identificador novo; resolução do ponto SEC-03 do debate da RFC-001, consequência direta de PRD-ESC-13 | TRANSCRICAO | `[09:25] Diego` |
| RFC-NOVO-01 | docs/RFC.md | Problema | Marcado `NOVO`: `create` insere o pedido em `PENDING` sem passar pelo `changeStatus`, então um cliente inscrito em `PENDING` não recebe evento; a RFC trata como fora de escopo desta entrega, mas registra a decisão explicitamente em vez de por omissão | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-DIV-01 | docs/RFC.md | Risco | Divergência não resolvida do debate (SEC-04, classificado DIVERGENTE): a validação da URL do webhook cobre só o esquema HTTPS, não o destino, o que pode expor o worker a SSRF contra rede privada ou endereço de metadados de nuvem; cenário marcado `HIPÓTESE` pelo próprio proponente, não discutido na reunião (hipótese) | TRANSCRICAO | `[09:23] Sofia` |
| FDD-CTX-01 | docs/FDD.md | Contexto | Ausência de canal outbound: clientes só descobrem mudança de status fazendo polling em `GET /orders` | TRANSCRICAO | `[09:00] Marcos` |
| FDD-CTX-02 | docs/FDD.md | Contexto | Evento publicado de forma transacionalmente consistente: commit da transação implica evento gravado, rollback implica ausência do evento | TRANSCRICAO | `[09:06] Diego` |
| FDD-CTX-03 | docs/FDD.md | Contexto | Uso do outbox no MySQL já existente, sem inventar infraestrutura nova de mensageria | TRANSCRICAO | `[09:07] Diego` |
| FDD-CTX-04 | docs/FDD.md | Contexto | Ponto único de integração: `changeStatus` em `src/modules/orders/order.service.ts`, já dentro de `this.prisma.$transaction` com update de `orders`, insert em `order_status_history` e ajuste de `stock_quantity` | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-CTX-05 | docs/FDD.md | Contexto | Chamada de `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da mesma transação, depois do `tx.order.update` | TRANSCRICAO | `[09:41] Bruno` |
| FDD-CTX-06 | docs/FDD.md | Contexto | Módulo novo `src/modules/webhooks`, seguindo o padrão de domínio já usado em `src/modules/orders/` | TRANSCRICAO | `[09:27] Bruno` |
| FDD-CTX-07 | docs/FDD.md | Contexto | Worker em processo Node separado, entry point próprio ao lado de `src/server.ts`, sincronizando com a API apenas pelo banco | TRANSCRICAO | `[09:11] Larissa` |
| FDD-CTX-08 | docs/FDD.md | Contexto | Ator: cliente B2B integrador, consumindo endpoints REST de configuração de webhook, rotação de secret e consulta de entregas, autenticado pelo JWT já existente | TRANSCRICAO | `[09:32] Marcos` |
| FDD-CTX-09 | docs/FDD.md | Contexto | Ator: operador interno com papel ADMIN, que reprocessa manualmente eventos da dead letter queue, restrito por `requireRole` | TRANSCRICAO | `[09:36] Larissa` |
| FDD-CTX-09-cod | docs/FDD.md | Contexto | `requireRole` existente em `src/middlewares/auth.middleware.ts`, reusado para restringir o replay ao papel ADMIN | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-CTX-10 | docs/FDD.md | Contexto | Ator: endpoint HTTPS do próprio cliente, único destinatário externo do fluxo, exclusivamente outbound | TRANSCRICAO | `[09:02] Sofia` |
| FDD-CTX-11 | docs/FDD.md | Contexto | Limite técnico do escopo: nenhuma chamada inbound a autenticar, nenhuma infraestrutura nova provisionada | TRANSCRICAO | `[09:07] Diego` |
| FDD-CTX-12 | docs/FDD.md | Contexto | Ordenação garantida apenas por `order_id` em regime de worker único, sem garantia técnica de instância única | TRANSCRICAO | `[09:12] Diego` |
| FDD-OBJ-01 | docs/FDD.md | Objetivo | Latência de entrega menor que 10s entre commit da mudança de status e chegada da requisição no endpoint do cliente | TRANSCRICAO | `[09:02] Marcos` |
| FDD-OBJ-02 | docs/FDD.md | Objetivo | Polling do worker a cada 2 segundos, estabelecido como pior caso de latência de disparo | TRANSCRICAO | `[09:10] Larissa` |
| FDD-OBJ-03 | docs/FDD.md | Objetivo | Timeout de 10 segundos por tentativa de entrega HTTP | TRANSCRICAO | `[09:42] Diego` |
| FDD-OBJ-04 | docs/FDD.md | Objetivo | Retry cobrindo cerca de 15 horas em 5 tentativas antes de mover para dead letter | TRANSCRICAO | `[09:17] Diego` |
| FDD-OBJ-05 | docs/FDD.md | Objetivo | 100% de proporção entre mudanças de status elegíveis e eventos gravados na outbox, garantido por transação | TRANSCRICAO | `[09:06] Diego` |
| FDD-OBJ-06 | docs/FDD.md | Objetivo | Zero novos componentes de infraestrutura | TRANSCRICAO | `[09:07] Diego` |
| FDD-OBJ-07 | docs/FDD.md | Objetivo | Consistência transacional forte entre status e evento, com rollback conjunto em caso de falha | TRANSCRICAO | `[09:04] Bruno` |
| FDD-OBJ-08 | docs/FDD.md | Objetivo | Entrega at-least-once, não exactly-once, com deduplicação de responsabilidade do cliente via `X-Event-Id` em UUID | TRANSCRICAO | `[09:25] Diego` |
| FDD-OBJ-09 | docs/FDD.md | Objetivo | Ordenação garantida por `order_id`, sequencial entre eventos do mesmo pedido, apenas em regime de worker único, sem mecanismo técnico de exclusão mútua | TRANSCRICAO | `[09:12] Diego` |
| FDD-OBJ-10 | docs/FDD.md | Objetivo | Payload gravado como snapshot no momento da inserção na outbox, nunca recalculado no envio | TRANSCRICAO | `[09:52] Larissa` |
| FDD-OBJ-11 | docs/FDD.md | Objetivo | Assinatura HMAC-SHA256 determinística sobre o corpo da requisição, verificável pelo cliente | TRANSCRICAO | `[09:20] Sofia` |
| FDD-ESC-01 | docs/FDD.md | Escopo | Tabela `webhook_outbox`, inserida dentro da transação do `changeStatus` via `publishWebhookEvent` | TRANSCRICAO | `[09:06] Diego` |
| FDD-ESC-02 | docs/FDD.md | Escopo | Módulo `src/modules/webhooks` no padrão de `src/modules/orders/` (controller, service, repository, routes, schemas) | TRANSCRICAO | `[09:27] Bruno` |
| FDD-ESC-03 | docs/FDD.md | Escopo | Worker em processo Node separado, entry point próprio, `PrismaClient` próprio, polling a cada 2s, com função de processamento de lote isolável do loop | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ESC-04 | docs/FDD.md | Escopo | Entrega HTTP POST com timeout de 10s e headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` | TRANSCRICAO | `[09:44] Diego` |
| FDD-ESC-05a | docs/FDD.md | Escopo | Retry com backoff exponencial (5 tentativas, 1min a 12h) | TRANSCRICAO | `[09:17] Diego` |
| FDD-ESC-05b | docs/FDD.md | Escopo | Tabela `webhook_dead_letter` separada para eventos esgotados | TRANSCRICAO | `[09:18] Diego` |
| FDD-ESC-05c | docs/FDD.md | Escopo | Replay administrativo restrito a ADMIN via `requireRole` | TRANSCRICAO | `[09:36] Larissa` |
| FDD-ESC-06 | docs/FDD.md | Escopo | Assinatura HMAC-SHA256 com secret única por endpoint, rotacionável com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| FDD-ESC-07 | docs/FDD.md | Escopo | Entrega at-least-once com `X-Event-Id`; replay preserva o identificador original | TRANSCRICAO | `[09:25] Diego` |
| FDD-ESC-08a | docs/FDD.md | Escopo | Trilha de auditoria própria do replay administrativo, requisito de logar quem executou | TRANSCRICAO | `[09:36] Sofia` |
| FDD-ESC-08b | docs/FDD.md | Escopo | Confirmação de que não existe model ou utilitário de auditoria reaproveitável, tornando a trilha uma peça nova | CODIGO | `prisma/schema.prisma` |
| FDD-ESC-09 | docs/FDD.md | Escopo | `createLogger()` precisa ser parametrizada e `redactPaths` estendido para cobrir secret e assinatura, hoje sem esse alcance | CODIGO | `src/shared/logger/index.ts` |
| FDD-ESC-10 | docs/FDD.md | Escopo | CRUD de configuração de webhook aberto a qualquer papel autenticado, JWT existente | TRANSCRICAO | `[09:37] Sofia` |
| FDD-FESC-01 | docs/FDD.md | Fora de escopo | Notificação por e-mail em falha repetida, adiada para próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| FDD-FESC-02 | docs/FDD.md | Fora de escopo | Dashboard visual do cliente, projeto separado do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| FDD-FESC-03 | docs/FDD.md | Fora de escopo | Arquivamento das linhas entregues da outbox (~30 dias), reconhecido como necessário, fora desta entrega | TRANSCRICAO | `[09:08] Diego` |
| FDD-FESC-04 | docs/FDD.md | Fora de escopo | Escala para múltiplos workers com particionamento por `order_id` ou lock pessimista, adiada | TRANSCRICAO | `[09:13] Diego` |
| FDD-FESC-05 | docs/FDD.md | Fora de escopo | Endurecimento de permissões do CRUD de webhook, registrado como provisório | TRANSCRICAO | `[09:37] Sofia` |
| FDD-FESC-06 | docs/FDD.md | Fora de escopo | Rate limiting de envio de webhooks, "observar e decidir depois" | TRANSCRICAO | `[09:39] Larissa` |
| FDD-FESC-07 | docs/FDD.md | Fora de escopo | Definição de qual secret assina durante o grace period, e distinção entre rotação de rotina e por vazamento (hipótese: fora desta entrega) | TRANSCRICAO | `[09:21] Sofia` |
| FDD-FESC-08 | docs/FDD.md | Fora de escopo | Restart, health check e alerta para o worker, não definidos pela fonte | TRANSCRICAO | `[09:11] Diego` |
| FDD-FESC-10 | docs/FDD.md | Risco | Validação da URL de destino contra rede privada/metadados de nuvem, divergente entre segurança e arquitetura, não resolvida (hipótese) | TRANSCRICAO | `[09:23] Sofia` |
| FDD-FESC-11 | docs/FDD.md | Risco | Mecanismo técnico de garantia de worker único permanece questão em aberto | CODIGO | `src/config/database.ts` |
| FDD-DESC-01 | docs/FDD.md | Trade-off | Disparo síncrono de HTTP dentro da transação, descartado por travar a transação e não haver critério de rollback sensato | TRANSCRICAO | `[09:04] Bruno` |
| FDD-DESC-02 | docs/FDD.md | Trade-off | Fila dedicada tipo Redis Streams, descartada por exigir infraestrutura nova desproporcional ao time | TRANSCRICAO | `[09:07] Diego` |
| FDD-DESC-03 | docs/FDD.md | Trade-off | Trigger de banco para acionar o worker, descartado por MySQL não ter notificação nativa de processo externo | TRANSCRICAO | `[09:09] Diego` |
| FDD-DESC-04 | docs/FDD.md | Trade-off | Worker embutido no processo da API, descartado porque reinício da API interromperia o processamento | TRANSCRICAO | `[09:11] Diego` |
| FDD-DESC-05 | docs/FDD.md | Trade-off | Retry com 3 tentativas em janela curta, descartado por matar evento em indisponibilidade planejada | TRANSCRICAO | `[09:16] Diego` |
| FDD-DESC-06 | docs/FDD.md | Trade-off | Retry indefinido, descartado por deixar evento pendurado para sempre | TRANSCRICAO | `[09:15] Diego` |
| FDD-DESC-07 | docs/FDD.md | Trade-off | Truncamento de payload acima de 64 KB, descartado em favor de erro explícito | TRANSCRICAO | `[09:23] Sofia` |
| FDD-DESC-08 | docs/FDD.md | Trade-off | Dead letter como campo de estado na própria outbox, descartada em favor de tabela separada | TRANSCRICAO | `[09:18] Diego` |
| FDD-DESC-09 | docs/FDD.md | Trade-off | Secret global compartilhada entre endpoints, descartada porque vazamento comprometeria todos os clientes | TRANSCRICAO | `[09:21] Sofia` |
| FDD-DESC-10 | docs/FDD.md | Trade-off | Entrega exactly-once, descartada por exigir coordenação cara entre plataforma e cliente | TRANSCRICAO | `[09:25] Diego` |
| FDD-DESC-11 | docs/FDD.md | Trade-off | Auditoria do replay apoiada apenas no log compartilhado Pino, descartada por esvaziar o único controle reforçado da feature | TRANSCRICAO | `[09:36] Sofia` |
| FDD-FLU-01 | docs/FDD.md | Requisito Funcional | Fluxo principal: `publishWebhookEvent` consulta webhooks ativos do customer, aplica filtro de status e insere na outbox com payload snapshot e id UUID | TRANSCRICAO | `[09:41] Bruno` |
| FDD-FLU-02 | docs/FDD.md | Requisito Funcional | Fluxo principal: worker roda em loop a cada 2s consultando pendentes mais antigos em lotes pequenos | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLU-03 | docs/FDD.md | Requisito Funcional | Fluxo alternativo: se o endpoint responder com sucesso antes de esgotar tentativas, evento marcado entregue e ciclo encerrado | TRANSCRICAO | `[09:15] Diego` |
| FDD-FLU-04 | docs/FDD.md | Requisito Funcional | Escalonamento para dead letter: esgotada a 5ª tentativa, evento movido para `webhook_dead_letter` com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLU-05 | docs/FDD.md | Requisito Funcional | Replay administrativo: busca item na dead letter pelo id, recoloca como pendente preservando o identificador original, registra execução em log de auditoria | TRANSCRICAO | `[09:18] Diego` |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /webhooks`: cadastro com `customer_id`, `url`, `statuses` no corpo; secret gerada e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-01a | docs/FDD.md | Restrição | `WEBHOOK_INVALID_URL`: URL não HTTPS recusada pela validação de schema | TRANSCRICAO | `[09:23] Sofia` |
| FDD-CONTRATO-01b | docs/FDD.md | Contrato | Status codes numéricos (201/400/401 etc.) são inferência do padrão já em uso no código, não fixados pela fonte (hipótese) | CODIGO | `src/modules/orders/order.controller.ts` |
| FDD-CONTRATO-01c | docs/FDD.md | Contrato | Campos de resposta `id`, `active`, `createdAt` são hipótese composta a partir do padrão de resposta dos demais módulos (hipótese) | CODIGO | `src/modules/orders/order.controller.ts` |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /webhooks`: listagem dos webhooks cadastrados do customer | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-02a | docs/FDD.md | Contrato | Ausência da secret na listagem é hipótese consistente com o invariante de nunca expor a secret fora do momento de emissão (hipótese) | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /webhooks/:id`: atualização de URL, lista de status filtrados ou estado ativo | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /webhooks/:id`: remoção do webhook | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04a | docs/FDD.md | Contrato | `204 No Content` sem corpo de resposta é inferência do padrão já usado no `OrderController.delete` (hipótese) | CODIGO | `src/modules/orders/order.controller.ts` |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `GET /webhooks/:id/deliveries`: até 100 registros mais recentes, com resultado, payload, resposta e tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-05a | docs/FDD.md | Contrato | Nomes literais dos campos JSON de deliveries (`eventId`, `success`, `requestPayload`, `responseStatus`, `responseTimeMs`) são hipótese composta a partir do que a fonte descreve (hipótese) | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay`: evento recolocado como pendente na outbox | TRANSCRICAO | `[09:18] Diego` |
| FDD-CONTRATO-06a | docs/FDD.md | Contrato | `WEBHOOK_REPLAY_FORBIDDEN` (hipótese): código específico composto para cumprir a exigência de prefixo `WEBHOOK_` em todo erro do módulo | TRANSCRICAO | `[09:29] Larissa` |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | Rotação de secret: nova secret gerada, anterior válida por 24h (grace period) | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-07a | docs/FDD.md | Contrato | Path exato `POST /webhooks/:id/secret/rotate` é hipótese; a fonte só descreve o fluxo, sem fixar a rota literal (hipótese) | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Entrega ao endpoint do cliente: headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`, corpo com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONTRATO-08a | docs/FDD.md | Restrição | Payload até 64 KB, acima disso não é enviado, gera erro registrado sem truncamento | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-01 | docs/FDD.md | Requisito Funcional | `WEBHOOK_NOT_FOUND`: webhook ou item de dead letter informado não existe, tratado como 404 pelo error middleware | TRANSCRICAO | `[09:18] Diego` |
| FDD-ERRO-01-cod | docs/FDD.md | Restrição | `NotFoundError` fixa `errorCode = 'NOT_FOUND'` no próprio construtor sem parâmetro, diferente de `ConflictError`/`UnprocessableEntityError`/`BadRequestError`, motivando a hipótese de construir `AppError` diretamente para `WEBHOOK_NOT_FOUND` | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-02 | docs/FDD.md | Requisito Funcional | `WEBHOOK_INVALID_URL`: URL não HTTPS ou inválida no schema, herda de `BadRequestError`, 400 | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-02-cod | docs/FDD.md | Contexto | `BadRequestError` aceita `code` customizado no construtor, ao contrário de `ValidationError`, que fixa `errorCode = 'VALIDATION_ERROR'` | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-03 | docs/FDD.md | Requisito Funcional | `WEBHOOK_SECRET_REQUIRED`: operação depende de secret ativa inexistente | TRANSCRICAO | `[09:21] Sofia` |
| FDD-ERRO-03-cod | docs/FDD.md | Contexto | Herança de `UnprocessableEntityError` (422) por analogia ao padrão de `InsufficientStockError`, não definida pela fonte (hipótese) | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-04 | docs/FDD.md | Requisito Funcional | `WEBHOOK_REPLAY_FORBIDDEN` (hipótese): usuário autenticado sem papel ADMIN no replay, composto para cumprir a exigência de prefixo `WEBHOOK_` em todo erro do módulo | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERRO-04-cod | docs/FDD.md | Contexto | `ForbiddenError` fixa `errorCode = 'FORBIDDEN'` no construtor sem parâmetro, motivando a hipótese de construir `AppError` diretamente | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-05 | docs/FDD.md | Requisito Funcional | `WEBHOOK_OUTBOX_INSERT_FAILED` (hipótese): falha de inserção na outbox dentro da transação do `changeStatus`, composto para manter a tabela 100% `WEBHOOK_*` | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERRO-06 | docs/FDD.md | Requisito Não Funcional | Todos os códigos de erro do módulo usam prefixo `WEBHOOK_`, herdam de `AppError`, tratados pelo error middleware sem alteração | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERRO-06-cod | docs/FDD.md | Contexto | Confirmação de que `error.middleware.ts` despacha por `instanceof AppError`, sem menção a domínio específico | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Métrica: proporção entre eventos gravados na outbox e mudanças de status com webhook inscrito, meta de 100% | TRANSCRICAO | `[09:06] Diego` |
| FDD-OBS-02 | docs/FDD.md | Requisito Não Funcional | Métrica: número de tentativas de entrega e janela total de backoff (meta: 5 tentativas, ~15h) | TRANSCRICAO | `[09:17] Diego` |
| FDD-OBS-03 | docs/FDD.md | Requisito Não Funcional | Métrica: tempo entre commit da mudança de status e chegada da requisição no endpoint do cliente (meta: <10s) | TRANSCRICAO | `[09:02] Marcos` |
| FDD-OBS-04 | docs/FDD.md | Requisito Funcional | Histórico de entregas com resultado, payload, resposta e tempo de resposta dos últimos 100 envios, exposto ao cliente | TRANSCRICAO | `[09:34] Marcos` |
| FDD-OBS-05 | docs/FDD.md | Requisito Não Funcional | Logs via Pino já presente no projeto, sem introduzir ferramenta nova | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-06 | docs/FDD.md | Requisito Não Funcional | `redactPaths` hoje não cobre secret nem assinatura; passa a ser estendido | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-07 | docs/FDD.md | Requisito Não Funcional | `createLogger()` hoje não aceita parâmetro e fixa `service: 'order-management-api'`; precisa ser parametrizada para dar identidade própria ao worker | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-08 | docs/FDD.md | Risco | Sem definição de restart e alerta do worker, a falha é silenciosa porque nenhum evento vai para a dead letter, todos ficam pendentes | TRANSCRICAO | `[09:11] Diego` |
| FDD-DEP-01 | docs/FDD.md | Dependência | `changeStatus` em `src/modules/orders/order.service.ts` como ponto de integração transacional já existente | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-DEP-02 | docs/FDD.md | Dependência | `AppError` (`src/shared/errors/http-errors.ts`) como base das classes de erro `WEBHOOK_*` | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-DEP-03 | docs/FDD.md | Dependência | `src/middlewares/error.middleware.ts` trata os erros do módulo sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-DEP-04 | docs/FDD.md | Dependência | `src/shared/logger/index.ts`, logger Pino, com `redactPaths` estendido e `createLogger()` parametrizada | CODIGO | `src/shared/logger/index.ts` |
| FDD-DEP-05 | docs/FDD.md | Dependência | `src/middlewares/auth.middleware.ts` (`requireRole`) protege o replay administrativo | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-DEP-06 | docs/FDD.md | Dependência | `PrismaClient` própria do worker, mesma `DATABASE_URL` da API | TRANSCRICAO | `[09:30] Bruno` |
| FDD-INTEG-01 | docs/FDD.md | Contexto | `changeStatus` já roda dentro de `this.prisma.$transaction` fazendo leitura do pedido, ajuste de `stockQuantity`, update, insert de histórico e releitura final com relações | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INTEG-02 | docs/FDD.md | Contexto | `AppError` define `statusCode`, `errorCode` e `details`, base de que as subclasses HTTP herdam | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INTEG-03 | docs/FDD.md | Contexto | `InvalidStatusTransitionError` estende `ConflictError`, `InsufficientStockError` estende `UnprocessableEntityError`, sem herdar de `AppError` diretamente quando há equivalente HTTP com suporte a código customizado | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INTEG-04 | docs/FDD.md | Contexto | `error.middleware.ts` trata `AppError` por `instanceof`, depois `ZodError`, depois `Prisma.PrismaClientKnownRequestError`, sem menção a domínio específico | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INTEG-05 | docs/FDD.md | Contexto | `auth.middleware.ts` contém `authenticate` e `requireRole`, operando sobre `AuthUser` extraído do JWT | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INTEG-06 | docs/FDD.md | Contexto | `src/shared/logger/index.ts` exporta `createLogger()` e instância singleton `logger`, com `redactPaths` e `service` fixos atualmente | CODIGO | `src/shared/logger/index.ts` |
| FDD-INTEG-07 | docs/FDD.md | Contexto | `buildControllers` em `src/app.ts` monta `OrderService` por injeção manual e precisa ser alterado para fiar a nova dependência do módulo de webhooks | CODIGO | `src/app.ts` |
| FDD-INTEG-08 | docs/FDD.md | Contexto | `src/server.ts` como precedente de entry point único do processo da API, ao lado do qual o worker ganha entry point próprio | TRANSCRICAO | `[09:11] Larissa` |
| FDD-CRIT-01 | docs/FDD.md | Requisito Funcional | Mudar status para status inscrito insere linha na outbox na mesma transação; rollback não deixa linha; status não inscrito não insere | TRANSCRICAO | `[09:34] Bruno` |
| FDD-CRIT-02 | docs/FDD.md | Requisito Funcional | Requisição de entrega contém os headers e o corpo definidos no Contrato 8 | TRANSCRICAO | `[09:43] Diego` |
| FDD-CRIT-03 | docs/FDD.md | Requisito Funcional | `X-Signature` corresponde ao HMAC-SHA256 do corpo calculado com a secret do endpoint, verificável recalculando no teste | TRANSCRICAO | `[09:20] Sofia` |
| FDD-CRIT-04 | docs/FDD.md | Requisito Funcional | Cadastro com URL HTTP é recusado, com HTTPS é aceito | TRANSCRICAO | `[09:23] Sofia` |
| FDD-CRIT-05 | docs/FDD.md | Requisito Funcional | Payload acima de 64 KB não é enviado, resulta em erro registrado, sem truncamento | TRANSCRICAO | `[09:23] Sofia` |
| FDD-CRIT-06 | docs/FDD.md | Requisito Funcional | Rotação de secret gera nova secret, mantém a anterior válida por 24h e invalida após esse período | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CRIT-07 | docs/FDD.md | Requisito Funcional | `GET /webhooks/:id/deliveries` retorna até 100 registros com resultado, payload, resposta, tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CRIT-08 | docs/FDD.md | Requisito Funcional | Todos os erros do módulo usam prefixo `WEBHOOK_`, herdam de `AppError`, tratados pelo error middleware sem alteração nele | TRANSCRICAO | `[09:28] Bruno` |
| FDD-CRIT-09 | docs/FDD.md | Requisito Não Funcional | Evento pendente entregue em menos de 10 segundos do commit, no cenário sem falha | TRANSCRICAO | `[09:02] Marcos` |
| FDD-CRIT-10 | docs/FDD.md | Requisito Funcional | Falhas sucessivas reagendam respeitando 1min/5min/30min/2h/12h; a falha da última tentativa move para `webhook_dead_letter` com payload, motivo e timestamp | TRANSCRICAO | `[09:17] Diego` |
| FDD-CRIT-11 | docs/FDD.md | Requisito Funcional | `POST /admin/webhooks/dead-letter/:id/replay` recoloca evento como pendente quando chamado por ADMIN, retorna erro de autorização sem esse papel, gera registro de log | TRANSCRICAO | `[09:36] Larissa` |
| FDD-CRIT-12 | docs/FDD.md | Requisito Funcional | Dois eventos consecutivos do mesmo pedido chegam na ordem de `created_at` da outbox, em regime de worker único | TRANSCRICAO | `[09:12] Diego` |
| FDD-CRIT-13 | docs/FDD.md | Requisito Não Funcional | Worker continua processando após reinício da API, comprovando processo separado | TRANSCRICAO | `[09:11] Diego` |
| FDD-RISCO-01 | docs/FDD.md | Risco | Endpoint do cliente indisponível ou lento esgota as tentativas sem entrega; mitigação por retry/timeout, contingência via dead letter e replay | TRANSCRICAO | `[09:15] Diego` |
| FDD-RISCO-02 | docs/FDD.md | Risco | Vazamento da secret de um cliente permite forjar eventos assinados; mitigação por secret única, rotação e revisão de segurança | TRANSCRICAO | `[09:22] Diego` |
| FDD-RISCO-03 | docs/FDD.md | Risco | Escalar para múltiplos workers quebra a ordenação de eventos do mesmo pedido; mitigação manter worker único e documentar ausência de ordenação global | TRANSCRICAO | `[09:12] Diego` |
| FDD-RISCO-04 | docs/FDD.md | Risco | Volume alto de mudanças de status sobrecarrega o endpoint do cliente; nenhuma mitigação implementada, decisão de observar e agir depois | TRANSCRICAO | `[09:38] Diego` |
| FDD-RISCO-05 | docs/FDD.md | Risco | Cliente sem deduplicação processa o mesmo evento mais de uma vez; mitigação por `X-Event-Id` único e documentação da garantia at-least-once | TRANSCRICAO | `[09:25] Sofia` |
| FDD-RISCO-06 | docs/FDD.md | Risco | Falha na inserção do evento na outbox impede a mudança de status do pedido; mitigação pela inserção na mesma transação com rollback conjunto | TRANSCRICAO | `[09:40] Bruno` |
| FDD-RISCO-07 | docs/FDD.md | Risco | Vazamento de secret ou assinatura pelo log estruturado; mitigação pela extensão de `redactPaths` em `src/shared/logger/index.ts` | CODIGO | `src/shared/logger/index.ts` |
| FDD-RISCO-08 | docs/FDD.md | Risco | Crescimento indefinido da tabela de outbox degrada o polling do worker; sem mitigação implementada, arquivamento adiado | TRANSCRICAO | `[09:08] Diego` |
| FDD-RISCO-09 | docs/FDD.md | Risco | Duas instâncias do worker coexistindo tornam entregas legítimas indistinguíveis de reentrega maliciosa, agravante sobre a garantia at-least-once (hipótese, composição entre a ausência de mecanismo técnico de instância única e a garantia at-least-once por `X-Event-Id`) | CODIGO | `src/config/database.ts` |
