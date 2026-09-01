# Tracker de Rastreabilidade

Referência cruzada entre os itens registrados nos documentos de design e sua origem na transcrição da reunião ou no código da aplicação.

Convenções:

- **Fonte** `TRANSCRICAO` refere-se a `TRANSCRICAO.md`, com timestamp e nome do falante.
- **Fonte** `CODIGO` refere-se ao código da aplicação existente, com caminho do arquivo.
- Itens marcados no PRD como hipótese ou inferência aparecem aqui com o sufixo `(hipótese)` no resumo, e sua Localização aponta para a origem parcial que os motivou.

Cobertura atual: `docs/PRD.md`. As linhas de RFC, FDD e ADRs serão acrescentadas conforme esses documentos forem produzidos.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B pediram formalmente notificação em tempo real de mudança de status | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling no GET /orders, tornando a integração lenta e cara | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-03 | docs/PRD.md | Restrição | Risco de a Atlas migrar para o concorrente se não houver entrega no prazo | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-04 | docs/PRD.md | Contexto | Público-alvo inclui ADMIN interno, único autorizado a reprocessar a DLQ | TRANSCRICAO | `[09:36] Sofia` |
| PRD-CTX-05 | docs/PRD.md | Contexto | Feature entra em sistema existente, como módulo no padrão dos demais domínios | CODIGO | `src/modules/orders/` |
| PRD-CTX-06 | docs/PRD.md | Contexto | A aplicação não possui hoje mecanismo de notificação externa, eventos ou filas | CODIGO | `src/modules/` |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência aceita pelo cliente como tempo real é abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Latência de enfileiramento de 2 segundos no pior caso, aceita como decisão | TRANSCRICAO | `[09:10] Larissa` |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Cliente aprovou explicitamente a latência de 2 segundos | TRANSCRICAO | `[09:10] Marcos` |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Janela de recuperação de aproximadamente 15 horas aceita como suficiente | TRANSCRICAO | `[09:17] Marcos` |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Grace period de 24 horas na rotação de secret | TRANSCRICAO | `[09:21] Sofia` |
| PRD-OBJ-06 | docs/PRD.md | Objetivo | Nenhum evento descartado silenciosamente, com DLQ como destino final (meta derivada do desenho) | TRANSCRICAO | `[09:18] Diego` |
| PRD-SCP-01 | docs/PRD.md | Escopo | Escopo outbound-only: o sistema envia webhooks e não recebe | TRANSCRICAO | `[09:02] Marcos` |
| PRD-SCP-02 | docs/PRD.md | Escopo | Confirmação de que outbound simplifica o desenho de segurança | TRANSCRICAO | `[09:03] Sofia` |
| PRD-SCP-03 | docs/PRD.md | Fora de escopo | Adiado: email de alerta ao cliente quando o webhook falha repetidamente | TRANSCRICAO | `[09:37] Larissa` |
| PRD-SCP-04 | docs/PRD.md | Fora de escopo | Adiado: rate limiting de saída, a observar antes de implementar | TRANSCRICAO | `[09:39] Diego` |
| PRD-SCP-05 | docs/PRD.md | Fora de escopo | Adiado: múltiplos workers em paralelo e ordenação global | TRANSCRICAO | `[09:13] Diego` |
| PRD-SCP-06 | docs/PRD.md | Fora de escopo | Adiado: arquivamento das linhas entregues da outbox | TRANSCRICAO | `[09:08] Diego` |
| PRD-SCP-07 | docs/PRD.md | Fora de escopo | Adiado: endurecimento da autorização do CRUD de webhook | TRANSCRICAO | `[09:37] Sofia` |
| PRD-SCP-08 | docs/PRD.md | Fora de escopo | Adiado: dashboard visual, transferido para o time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-SCP-09 | docs/PRD.md | Trade-off | Descartado: disparo síncrono de HTTP dentro do changeStatus | TRANSCRICAO | `[09:04] Bruno` |
| PRD-SCP-10 | docs/PRD.md | Trade-off | Descartado: síncrono reafirmado como fora de questão | TRANSCRICAO | `[09:06] Diego` |
| PRD-SCP-11 | docs/PRD.md | Trade-off | Descartado: Redis Streams ou fila externa, por overengineering e infra nova | TRANSCRICAO | `[09:07] Diego` |
| PRD-SCP-12 | docs/PRD.md | Trade-off | Descartado: trigger de banco, pois MySQL não notifica processo externo | TRANSCRICAO | `[09:09] Diego` |
| PRD-SCP-13 | docs/PRD.md | Trade-off | Descartado: worker dentro da instância da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-SCP-14 | docs/PRD.md | Trade-off | Descartado: retry indefinido com backoff | TRANSCRICAO | `[09:15] Diego` |
| PRD-SCP-15 | docs/PRD.md | Trade-off | Descartado: 3 tentativas, insuficientes para manutenção planejada de 2 horas | TRANSCRICAO | `[09:16] Diego` |
| PRD-SCP-16 | docs/PRD.md | Trade-off | Descartado: DLQ como flag na própria outbox | TRANSCRICAO | `[09:18] Diego` |
| PRD-SCP-17 | docs/PRD.md | Trade-off | Descartado: entrega exactly-once, por exigir coordenação bilateral | TRANSCRICAO | `[09:25] Diego` |
| PRD-SCP-18 | docs/PRD.md | Trade-off | Descartado: secret global de plataforma | TRANSCRICAO | `[09:21] Sofia` |
| PRD-SCP-19 | docs/PRD.md | Trade-off | Descartado: truncamento de payload acima do limite, em favor de erro | TRANSCRICAO | `[09:23] Sofia` |
| PRD-SCP-20 | docs/PRD.md | Trade-off | Descartado: envio dos itens do pedido no payload | TRANSCRICAO | `[09:43] Diego` |
| PRD-SCP-21 | docs/PRD.md | Trade-off | Descartado: guardar apenas order_id e renderizar o payload no envio | TRANSCRICAO | `[09:52] Larissa` |
| PRD-SCP-22 | docs/PRD.md | Trade-off | Descartado: id auto incremental na outbox, em favor de UUID | TRANSCRICAO | `[09:51] Larissa` |
| PRD-SCP-23 | docs/PRD.md | Trade-off | Descartado: customer_id implícito no JWT | TRANSCRICAO | `[09:32] Larissa` |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook com url, lista de status e secret gerada pela plataforma | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-01a | docs/PRD.md | Requisito Funcional | Tabela de configuração armazena url, secret, customer_id e estado ativo | TRANSCRICAO | `[09:21] Bruno` |
| PRD-FR-01b | docs/PRD.md | Requisito Funcional | customer_id vai no body ou no path, não vem do JWT | TRANSCRICAO | `[09:32] Larissa` |
| PRD-FR-01c | docs/PRD.md | Restrição | JWT atual é do usuário operador e não carrega vínculo com customer | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-FR-01d | docs/PRD.md | Requisito Funcional | Validação de body por schema Zod, no padrão do middleware existente | CODIGO | `src/middlewares/validate.middleware.ts` |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | PATCH para editar, DELETE para remover e GET para listar webhooks do customer | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status que o webhook quer ouvir | TRANSCRICAO | `[09:33] Marcos` |
| PRD-FR-03a | docs/PRD.md | Decisão | Filtro aplicado na inserção da outbox, não no envio | TRANSCRICAO | `[09:34] Bruno` |
| PRD-FR-03b | docs/PRD.md | Decisão | Concordância com o filtro na inserção | TRANSCRICAO | `[09:34] Diego` |
| PRD-FR-03c | docs/PRD.md | Restrição | Status válidos são os do enum de status de pedido existente | CODIGO | `prisma/schema.prisma` |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Insert na outbox dentro da mesma transação que atualiza orders e history | TRANSCRICAO | `[09:06] Diego` |
| PRD-FR-04a | docs/PRD.md | Requisito Funcional | Alteração crítica ocorre no método changeStatus do service de orders | TRANSCRICAO | `[09:40] Bruno` |
| PRD-FR-04b | docs/PRD.md | Restrição | Fora da transação a garantia de atomicidade se perde | TRANSCRICAO | `[09:41] Diego` |
| PRD-FR-04c | docs/PRD.md | Requisito Funcional | Transação existente já atualiza order, histórico de status e estoque | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-FR-04d | docs/PRD.md | Restrição | Transição de status é validada por máquina de estados existente | CODIGO | `src/modules/orders/order.status.ts` |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Worker em polling a cada 2 segundos, buscando os pendentes mais antigos | TRANSCRICAO | `[09:09] Diego` |
| PRD-FR-05a | docs/PRD.md | Requisito Funcional | Leitura só de pendentes em batch pequeno, com índices em status e created_at | TRANSCRICAO | `[09:08] Diego` |
| PRD-FR-05b | docs/PRD.md | Requisito Funcional | Timeout de 10 segundos no envio, tratado como falha | TRANSCRICAO | `[09:42] Diego` |
| PRD-FR-05c | docs/PRD.md | Restrição | Estados da outbox: pendente, processando, falhou e entregue | TRANSCRICAO | `[09:08] Diego` |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Backoff exponencial com teto de tentativas antes de falha permanente | TRANSCRICAO | `[09:15] Diego` |
| PRD-FR-06a | docs/PRD.md | Decisão | Progressão 1m, 5m, 30m, 2h e 12h, totalizando quase 15 horas | TRANSCRICAO | `[09:17] Diego` |
| PRD-FR-06b | docs/PRD.md | Decisão | Cinco tentativas confirmadas como decisão fechada | TRANSCRICAO | `[09:17] Larissa` |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | DLQ em tabela separada com payload, motivo da falha e timestamp | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-07a | docs/PRD.md | Requisito Funcional | Replay manual por endpoint admin, recolocando o evento na outbox | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-07b | docs/PRD.md | Requisito Funcional | Caminho POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | `[09:35] Diego` |
| PRD-FR-07c | docs/PRD.md | Requisito Funcional | Replay exige role ADMIN e log de auditoria de quem executou | TRANSCRICAO | `[09:36] Sofia` |
| PRD-FR-07d | docs/PRD.md | Decisão | Reuso do requireRole existente para o controle de papel | TRANSCRICAO | `[09:36] Larissa` |
| PRD-FR-07e | docs/PRD.md | Restrição | requireRole e o retorno 403 já existem no middleware de autenticação | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-FR-07f | docs/PRD.md | Restrição | Caminhos ficam sob o prefixo versionado da API | CODIGO | `src/app.ts` |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Assinatura HMAC do payload enviada no header X-Signature | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-08a | docs/PRD.md | Decisão | Algoritmo SHA-256, por ser padrão de mercado com biblioteca disponível | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-08b | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp e Content-Type application/json | TRANSCRICAO | `[09:44] Diego` |
| PRD-FR-08c | docs/PRD.md | Requisito Funcional | Header X-Webhook-Id para cliente com múltiplos cadastros | TRANSCRICAO | `[09:44] Sofia` |
| PRD-FR-08d | docs/PRD.md | Requisito Funcional | Formato do payload em JSON com event_id, event_type, timestamp ISO 8601 e campos da order | TRANSCRICAO | `[09:43] Diego` |
| PRD-FR-08e | docs/PRD.md | Restrição | Campos do payload existem no model de pedido | CODIGO | `prisma/schema.prisma` |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Secret única por endpoint, nunca global | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-09a | docs/PRD.md | Requisito Funcional | Rotação de secret por API com grace period de 24 horas | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-09b | docs/PRD.md | Contexto | Motivação: cliente já vazou secret em log de aplicação | TRANSCRICAO | `[09:22] Diego` |
| PRD-FR-09c | docs/PRD.md | Requisito Não Funcional | Teto de 64KB por payload, com erro em vez de truncamento | TRANSCRICAO | `[09:24] Diego` |
| PRD-FR-09d | docs/PRD.md | Requisito Funcional | Código de erro WEBHOOK_SECRET_REQUIRED citado, sem condição definida | TRANSCRICAO | `[09:28] Bruno` |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Histórico de entregas com sucesso, falha, payload, response e tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-10a | docs/PRD.md | Requisito Funcional | Helper de paginação já existente como padrão de listagem (hipótese) | CODIGO | `src/shared/http/response.ts` |
| PRD-FR-11 | docs/PRD.md | Decisão | Prefixo WEBHOOK_ em todos os códigos de erro do módulo | TRANSCRICAO | `[09:29] Larissa` |
| PRD-FR-11a | docs/PRD.md | Requisito Funcional | Erros do módulo derivam de AppError, no padrão dos erros de domínio existentes | TRANSCRICAO | `[09:28] Bruno` |
| PRD-FR-11b | docs/PRD.md | Restrição | Error middleware centralizado já trata AppError, Zod e Prisma sem alteração | TRANSCRICAO | `[09:29] Bruno` |
| PRD-FR-11c | docs/PRD.md | Restrição | AppError e classes derivadas existentes | CODIGO | `src/shared/errors/http-errors.ts` |
| PRD-FR-11d | docs/PRD.md | Restrição | Error middleware é handler do ciclo HTTP e não cobre o worker (inferência) | CODIGO | `src/middlewares/error.middleware.ts` |
| PRD-FR-11e | docs/PRD.md | Restrição | Logger Pino já presente, sem dependência nova | TRANSCRICAO | `[09:29] Bruno` |
| PRD-FR-11f | docs/PRD.md | Restrição | Configuração do logger existente | CODIGO | `src/shared/logger/index.ts` |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Módulo em src/modules/webhooks com controller, service, repository, routes e schemas | TRANSCRICAO | `[09:27] Bruno` |
| PRD-FR-12a | docs/PRD.md | Requisito Funcional | Entry-point src/worker.ts e script npm run worker | TRANSCRICAO | `[09:11] Larissa` |
| PRD-FR-12b | docs/PRD.md | Requisito Funcional | Lógica de processamento em arquivo dentro do módulo de webhooks | TRANSCRICAO | `[09:28] Bruno` |
| PRD-FR-12c | docs/PRD.md | Decisão | Ids em UUID, seguindo o padrão do projeto | TRANSCRICAO | `[09:51] Larissa` |
| PRD-FR-12d | docs/PRD.md | Restrição | Padrão de UUID confirmado no schema do banco | CODIGO | `prisma/schema.prisma` |
| PRD-FR-12e | docs/PRD.md | Restrição | Projeto hoje possui apenas o entry-point da API | CODIGO | `package.json` |
| PRD-FR-12f | docs/PRD.md | Restrição | Router do módulo é registrado no agregador de rotas existente | CODIGO | `src/routes/index.ts` |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência abaixo de 10 segundos, com 2 segundos no pior caso | TRANSCRICAO | `[09:10] Larissa` |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Nenhuma chamada de rede dentro da transação de mudança de status | TRANSCRICAO | `[09:04] Bruno` |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Nenhuma linha inserida quando o status não é assinado | TRANSCRICAO | `[09:34] Bruno` |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Envio sequencial dentro do batch para preservar a ordenação (hipótese) | TRANSCRICAO | `[09:12] Diego` |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Disponibilidade alvo de 99.9 por cento (hipótese, não definida na reunião) | TRANSCRICAO | `[09:11] Diego` |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório, com recusa de url http na validação | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Header X-Timestamp permite ao cliente detectar tentativa de replay | TRANSCRICAO | `[09:44] Diego` |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | CRUD aberto a qualquer papel autenticado nesta fase | TRANSCRICAO | `[09:37] Sofia` |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Redação de campos sensíveis já configurada no logger | CODIGO | `src/shared/logger/index.ts` |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Papéis ADMIN e OPERATOR existentes como base de autorização | CODIGO | `prisma/schema.prisma` |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Log de auditoria do replay de DLQ | TRANSCRICAO | `[09:36] Sofia` |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once, com possibilidade de evento duplicado | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Deduplicação pelo cliente através do X-Event-Id com UUID por evento | TRANSCRICAO | `[09:25] Diego` |
| PRD-NFR-14 | docs/PRD.md | Requisito Não Funcional | Payload gravado como snapshot na inserção | TRANSCRICAO | `[09:52] Larissa` |
| PRD-NFR-15 | docs/PRD.md | Requisito Não Funcional | Ordenação garantida por pedido apenas com worker único, como limitação conhecida | TRANSCRICAO | `[09:13] Larissa` |
| PRD-NFR-16 | docs/PRD.md | Restrição | Nenhuma infraestrutura nova, solução restrita ao MySQL existente | TRANSCRICAO | `[09:07] Diego` |
| PRD-NFR-17 | docs/PRD.md | Restrição | Stack e serviços atuais do ambiente | CODIGO | `docker-compose.yml` |
| PRD-NFR-18 | docs/PRD.md | Restrição | Portabilidade: MySQL não possui o mecanismo de notificação do Postgres | TRANSCRICAO | `[09:09] Diego` |
| PRD-NFR-19 | docs/PRD.md | Requisito Não Funcional | Proteção contra requisições para endereços internos (hipótese do autor, não levantada na reunião) | TRANSCRICAO | `[09:23] Sofia` |
| PRD-ARQ-01 | docs/PRD.md | Decisão | Padrão outbox transacional, com atomicidade entre status e evento | TRANSCRICAO | `[09:06] Diego` |
| PRD-ARQ-02 | docs/PRD.md | Decisão | Outbox em MySQL registrada como decisão fechada | TRANSCRICAO | `[09:08] Larissa` |
| PRD-ARQ-03 | docs/PRD.md | Decisão | Worker em processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-ARQ-04 | docs/PRD.md | Decisão | PrismaClient próprio no worker, mesmo banco e mesma string de conexão | TRANSCRICAO | `[09:30] Bruno` |
| PRD-ARQ-05 | docs/PRD.md | Decisão | Função publishWebhookEvent recebendo o transaction client | TRANSCRICAO | `[09:41] Bruno` |
| PRD-ARQ-06 | docs/PRD.md | Decisão | Função pura recebendo o tx, sem injetar repository inteiro | TRANSCRICAO | `[09:41] Diego` |
| PRD-ARQ-07 | docs/PRD.md | Restrição | Tipo do transaction client atravessa a fronteira do módulo | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-ARQ-08 | docs/PRD.md | Decisão | Reuso máximo dos padrões existentes, sem dependência nova | TRANSCRICAO | `[09:30] Larissa` |
| PRD-ARQ-09 | docs/PRD.md | Restrição | Integração externa única: endpoints HTTPS dos clientes B2B | TRANSCRICAO | `[09:02] Marcos` |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Outbox amplia a fronteira de uma transação já pesada | TRANSCRICAO | `[09:04] Bruno` |
| PRD-DEC-02 | docs/PRD.md | Trade-off | Latência mínima de 2 segundos assumida explicitamente | TRANSCRICAO | `[09:10] Larissa` |
| PRD-DEC-03 | docs/PRD.md | Trade-off | Escala horizontal exigiria particionamento ou lock pessimista | TRANSCRICAO | `[09:13] Diego` |
| PRD-DEC-04 | docs/PRD.md | Trade-off | Evento pode levar até cerca de 15 horas para ser entregue | TRANSCRICAO | `[09:17] Diego` |
| PRD-DEC-05 | docs/PRD.md | Trade-off | At-least-once transfere responsabilidade de deduplicação ao cliente | TRANSCRICAO | `[09:25] Sofia` |
| PRD-DEC-06 | docs/PRD.md | Trade-off | Mitigação do at-least-once é documental, no portal de desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| PRD-DEC-07 | docs/PRD.md | Trade-off | Snapshot duplica dados entre outbox, histórico e DLQ | TRANSCRICAO | `[09:52] Diego` |
| PRD-DEC-08 | docs/PRD.md | Trade-off | Payload enxuto obriga o cliente a uma chamada extra para obter detalhes | TRANSCRICAO | `[09:43] Diego` |
| PRD-DEC-09 | docs/PRD.md | Trade-off | Grace period mantém duas secrets simultaneamente válidas por 24 horas | TRANSCRICAO | `[09:21] Sofia` |
| PRD-DEC-10 | docs/PRD.md | Trade-off | Clientes não pediram ordenação global, o que sustenta a limitação aceita | TRANSCRICAO | `[09:14] Marcos` |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança sobre HMAC e geração de secret antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| PRD-DEP-02 | docs/PRD.md | Dependência | Reforço da revisão de segurança como condição de subida | TRANSCRICAO | `[09:49] Sofia` |
| PRD-DEP-03 | docs/PRD.md | Dependência | Sessão de revisão do documento de design antes de iniciar a implementação | TRANSCRICAO | `[09:50] Larissa` |
| PRD-DEP-04 | docs/PRD.md | Dependência | Ordem de construção derivada da estimativa de esforço por sprint | TRANSCRICAO | `[09:46] Larissa` |
| PRD-DEP-05 | docs/PRD.md | Dependência | Documentação da integração no portal de desenvolvedor | TRANSCRICAO | `[09:40] Marcos` |
| PRD-DEP-06 | docs/PRD.md | Dependência | Confirmação de prazo e atualização dos clientes | TRANSCRICAO | `[09:47] Marcos` |
| PRD-DEP-07 | docs/PRD.md | Dependência | Migrations das três tabelas antes do código do worker | TRANSCRICAO | `[09:46] Larissa` |
| PRD-DEP-08 | docs/PRD.md | Dependência | Schema de variáveis de ambiente precisa ser estendido para o worker (hipótese) | CODIGO | `src/config/env.ts` |
| PRD-RSK-01 | docs/PRD.md | Risco | Worker único é ponto de falha e teto de throughput | TRANSCRICAO | `[09:12] Diego` |
| PRD-RSK-02 | docs/PRD.md | Risco | Mecanismo de assinatura durante o grace period não foi definido | TRANSCRICAO | `[09:21] Sofia` |
| PRD-RSK-03 | docs/PRD.md | Risco | Ausência de isolamento entre customers no CRUD de webhook | TRANSCRICAO | `[09:37] Sofia` |
| PRD-RSK-03a | docs/PRD.md | Risco | Token não carrega vínculo com customer, o que amplia o risco de isolamento | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-RSK-04 | docs/PRD.md | Risco | Evento preso em processando se o worker morre no meio do envio (lacuna) | TRANSCRICAO | `[09:08] Diego` |
| PRD-RSK-05 | docs/PRD.md | Risco | Falha definitiva passa despercebida, pois o alerta ao cliente foi adiado | TRANSCRICAO | `[09:37] Larissa` |
| PRD-RSK-06 | docs/PRD.md | Risco | Prazo comercial com risco de perda de cliente | TRANSCRICAO | `[09:45] Marcos` |
| PRD-RSK-07 | docs/PRD.md | Risco | Worker sem tratamento de erro de topo (inferência do código) | CODIGO | `src/middlewares/error.middleware.ts` |
| PRD-RSK-08 | docs/PRD.md | Risco | Cliente pode não implementar deduplicação por identificador de evento | TRANSCRICAO | `[09:25] Sofia` |
| PRD-RSK-09 | docs/PRD.md | Risco | Crescimento das tabelas sem rotina de arquivamento nesta fase | TRANSCRICAO | `[09:08] Diego` |
| PRD-RSK-10 | docs/PRD.md | Risco | Insert na outbox pode derrubar a mudança de status | TRANSCRICAO | `[09:41] Diego` |
| PRD-RSK-11 | docs/PRD.md | Risco | Ausência de limitação de taxa na saída, registrada como ponto em aberto | TRANSCRICAO | `[09:38] Diego` |
| PRD-RSK-12 | docs/PRD.md | Risco | Ordenação se quebra se uma segunda instância do worker for iniciada | TRANSCRICAO | `[09:13] Larissa` |
| PRD-TST-01 | docs/PRD.md | Restrição | Runner de teste, execução serial e includes já configurados | CODIGO | `vitest.config.ts` |
| PRD-TST-02 | docs/PRD.md | Restrição | Padrão vigente de teste de integração de API contra banco real | CODIGO | `tests/orders.test.ts` |
| PRD-TST-03 | docs/PRD.md | Restrição | Fábricas de dados usadas pelos testes existentes | CODIGO | `tests/helpers` |
| PRD-TST-04 | docs/PRD.md | Requisito Não Funcional | Teste do rollback da transação quando o insert na outbox falha | TRANSCRICAO | `[09:40] Bruno` |
| PRD-TST-05 | docs/PRD.md | Requisito Não Funcional | Testes ponta a ponta previstos na estimativa de esforço | TRANSCRICAO | `[09:46] Larissa` |
| PRD-TST-06 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança como portão de validação obrigatório | TRANSCRICAO | `[09:46] Sofia` |
