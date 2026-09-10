### FDD: Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-09-09
Responsável: Larissa (Tech Lead)

---

### 1. Contexto e motivação técnica

O problema técnico é a ausência de um canal outbound de notificação de mudança de status de pedido: hoje os clientes B2B só descobrem a mudança fazendo polling em `GET /orders`, sem nenhum aviso de evento [PRD seção Problemas priorizados]. A feature publica o evento de forma transacionalmente consistente com a mudança de status: se a transação do `changeStatus` commitou, o evento existe; se sofreu rollback, o evento não existe [RFC seção "TL;DR"] [ADR-001], usando outbox no MySQL já existente, sem inventar infraestrutura nova de mensageria.

A integração entra num único ponto de código já existente, o `changeStatus` em `src/modules/orders/order.service.ts`, que já roda dentro de `this.prisma.$transaction` fazendo update de `orders`, insert em `order_status_history` e ajuste de `stock_quantity`; a feature passa a chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro dessa mesma transação, depois do `tx.order.update` [ADR-001]. O restante é um módulo novo, `src/modules/webhooks`, seguindo o padrão de domínio já usado em `src/modules/orders/` (controller, service, repository, routes, schemas) [ADR-006], mais um worker em processo Node separado, com entry point próprio ao lado de `src/server.ts` e instância própria de `PrismaClient`, sincronizando com a API apenas pelo banco [ADR-002].

Atores: o cliente B2B integrador, que consome os endpoints REST de configuração de webhook, rotação de secret e consulta de entregas, autenticado pelo JWT já existente [PRD seção Público-alvo] [PRD seção RF-01]; o operador interno com papel ADMIN, que reprocessa manualmente eventos da dead letter queue, restrito por `requireRole` em `src/middlewares/auth.middleware.ts` [PRD seção RF-07] [ADR-003]; e o endpoint HTTPS do próprio cliente, único destinatário externo do fluxo, exclusivamente outbound [RFC seção "Contexto e problema"]. Limite técnico do escopo: nenhuma chamada inbound a autenticar, nenhuma infraestrutura nova provisionada, e ordenação garantida apenas por `order_id` em regime de worker único, sem garantia técnica de instância única [RFC seção "Contexto e problema"] [ADR-002].

---

### 2. Objetivos técnicos

- Latência de entrega menor que 10 segundos entre o commit da mudança de status e a chegada da requisição no endpoint do cliente, no cenário sem falha [PRD seção Objetivos e métricas]
- Polling do worker a cada 2 segundos, estabelecendo esse valor como pior caso de latência de disparo [PRD seção Requisitos não funcionais - Performance] [ADR-002]
- Timeout de 10 segundos por tentativa de entrega HTTP [PRD seção RF-04] [ADR-003]
- Retry cobrindo cerca de 15 horas em 5 tentativas (1 min, 5 min, 30 min, 2h, 12h) antes de mover para dead letter [PRD seção Objetivos e métricas] [ADR-003]
- 100% de proporção entre mudanças de status elegíveis (com webhook inscrito) e eventos gravados na outbox, garantido por transação [PRD seção Objetivos e métricas] [ADR-001]
- Zero novos componentes de infraestrutura [PRD seção Objetivos e métricas] [ADR-006]
- Consistência transacional forte entre status e evento, com rollback conjunto em caso de falha [ADR-001]
- Entrega at-least-once, não exactly-once, com deduplicação de responsabilidade do cliente via `X-Event-Id` em UUID único por evento [ADR-005]
- Ordenação garantida por `order_id` (entrega sequencial entre eventos do mesmo pedido, concorrente entre pedidos distintos) apenas em regime de worker único, sem mecanismo técnico de exclusão mútua que garanta instância única [ADR-002]
- Payload gravado como snapshot no momento da inserção na outbox, nunca recalculado no envio [ADR-001]
- Assinatura HMAC-SHA256 determinística sobre o corpo da requisição, verificável pelo cliente [ADR-004]

---

### 3. Escopo e exclusões

**Incluído**
- Tabela `webhook_outbox`, inserida dentro da transação do `changeStatus` via `publishWebhookEvent(tx, order, fromStatus, toStatus)` [PRD seção Escopo] [ADR-001]
- Módulo `src/modules/webhooks` no padrão de `src/modules/orders/` (controller, service, repository, routes, schemas) [ADR-006]
- Worker em processo Node separado, entry point próprio, `PrismaClient` próprio, polling a cada 2s, com função de processamento de lote isolável do loop [ADR-002]
- Entrega HTTP POST com timeout de 10s, headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` [PRD seção RF-04]
- Retry com backoff exponencial (5 tentativas, 1min a 12h) e tabela `webhook_dead_letter` separada, com replay administrativo restrito a ADMIN via `requireRole` [ADR-003]
- Assinatura HMAC-SHA256 com secret única por endpoint, rotacionável com grace period de 24h, secret nunca exposta fora do momento de emissão [ADR-004]
- Entrega at-least-once com `X-Event-Id`, replay preservando o identificador original [ADR-005]
- Trilha de auditoria própria e consultável do replay administrativo (peça nova, sem model existente a reaproveitar) [ADR-006]
- Parametrização de `createLogger()` em `src/shared/logger/index.ts` para dar identidade de serviço própria ao worker, e extensão de `redactPaths` para cobrir secret e assinatura [ADR-006] [ADR-004]
- CRUD de configuração de webhook aberto a qualquer papel autenticado, JWT existente [PRD seção RF-01]

**Excluído**

*Adiado*
- Notificação por e-mail em falha repetida, próxima fase, após medir impacto [PRD seção Fora de escopo]
- Dashboard visual do cliente, projeto separado do time de frontend [PRD seção Fora de escopo]
- Arquivamento das linhas entregues da outbox (~30 dias), reconhecido como necessário, fora desta entrega [PRD seção Fora de escopo]
- Escala para múltiplos workers com particionamento por `order_id` ou lock pessimista, limitação de ordenação conhecida e registrada [PRD seção Fora de escopo] [ADR-002]
- Endurecimento de permissões do CRUD de webhook, hoje aberto a qualquer papel, registrado como provisório [PRD seção Fora de escopo] [RFC seção "Questões em aberto"]
- Rate limiting de envio de webhooks, a reunião registrou apenas "observar e decidir depois" [PRD seção Ponto em aberto] [RFC seção "Questões em aberto"]
- Definição de qual secret assina durante o grace period, e distinção entre rotação de rotina e rotação por vazamento (hipótese: fora desta entrega) [ADR-004] [RFC seção "Questões em aberto"]
- Restart, health check e alerta para o worker, não definidos pela fonte [PRD seção Disponibilidade]
- Versionamento/evolução de schema do payload, não definido [RFC seção "Questões em aberto"]
- Validação da URL de destino contra rede privada/metadados de nuvem, divergente entre segurança e arquitetura, não resolvida nesta RFC [RFC seção "Questões em aberto"]
- Mecanismo técnico de garantia de worker único, permanece como questão em aberto, não decidido no ADR-002 [ADR-002]

*Descartado*
- Disparo síncrono de HTTP dentro da transação de mudança de status, travaria a transação com cliente lento e não há critério de rollback sensato [RFC seção "Alternativas consideradas"] [ADR-001]
- Fila dedicada tipo Redis Streams, exigiria infraestrutura nova, desproporcional ao tamanho do time [RFC seção "Alternativas consideradas"] [ADR-001]
- Trigger de banco para acionar o worker reativamente, MySQL não tem notificação nativa de processo externo [ADR-002]
- Worker embutido no processo da API, reinício da API interromperia o processamento [ADR-002]
- Retry com 3 tentativas em janela curta, mataria evento em indisponibilidade planejada (precedente de 2h) [ADR-003]
- Retry indefinido, deixaria evento pendurado para sempre [ADR-003]
- Truncamento de payload acima de 64 KB, descartado em favor de erro explícito [PRD seção Fora de escopo]
- Dead letter como campo de estado na própria outbox, descartada em favor de tabela separada, para manter leitura limpa [ADR-003]
- Secret global compartilhada entre endpoints, vazamento comprometeria todos os clientes ao mesmo tempo [ADR-004]
- Entrega exactly-once, exigiria coordenação cara entre plataforma e cliente [ADR-005]
- Auditoria do replay apoiada apenas no log compartilhado Pino, esvaziaria o único controle reforçado da feature [ADR-006]

---

### 4. Fluxos detalhados e diagramas

**Fluxo principal**
- No `changeStatus` de `src/modules/orders/order.service.ts`, dentro de `this.prisma.$transaction`: leitura do pedido, ajuste de `stockQuantity` item a item, `tx.order.update` gravando o novo status, insert em `order_status_history` [ADR-001]
- Depois do `tx.order.update` e antes do fim da transação: `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que consulta os webhooks ativos do customer, aplica o filtro de status de interesse e, havendo inscrição, insere o evento na `webhook_outbox` com payload em snapshot e id em UUID [ADR-001]
- A transação comita, persistindo mudança de status e evento em conjunto [PRD seção RF-03]
- O worker, em processo separado com `PrismaClient` próprio, roda em loop consultando a cada 2 segundos os eventos pendentes mais antigos em lotes pequenos, com o processamento de um lote isolado numa função própria separada do loop de polling [ADR-002]
- Para cada evento: monta os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`, envia POST HTTP para a URL cadastrada com timeout de 10 segundos [PRD seção RF-04]
- Em caso de resposta de sucesso, marca o evento como entregue na outbox e registra a entrega para consulta posterior [PRD seção RF-04]

**Fluxos alternativos e exceções**
- Retry: timeout de 10s ou resposta de erro do cliente conta como falha; o worker registra a tentativa e calcula o horário da próxima tentativa pela progressão 1min, 5min, 30min, 2h, 12h [ADR-003]
- Se o endpoint voltar a responder com sucesso antes de esgotar as tentativas, o evento é marcado como entregue e o ciclo é encerrado [PRD seção RF-05]
- Escalonamento para dead letter: esgotada a quinta tentativa sem sucesso, o evento é movido da `webhook_outbox` para a tabela `webhook_dead_letter`, separada, com payload, motivo da falha e timestamp [ADR-003]
- Replay administrativo: `POST /admin/webhooks/dead-letter/:id/replay`, autenticado, com `requireRole` restringindo a ADMIN; o serviço busca o item na `webhook_dead_letter` pelo id, recoloca o evento como pendente na `webhook_outbox` preservando o identificador de evento original (não gera UUID novo), e registra a execução em log de auditoria identificando quem fez o replay [PRD seção RF-07] [ADR-003] [ADR-005] [ADR-006]

**Diagramas**

Sequência (a): publicação transacional
```
tx.order.update
  -> publishWebhookEvent(tx, order, fromStatus, toStatus)
    -> consulta webhooks ativos do customer + filtro de status de interesse
    -> insert em webhook_outbox (payload snapshot, event_id UUID)
  -> commit da transação
```
[ADR-001]

Sequência (b): worker / polling
```
loop a cada 2s
  -> seleciona lote de pendentes por created_at
  -> processa lote (sequencial por order_id, concorrente entre pedidos)
    -> monta headers (X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id)
    -> POST com timeout 10s
      -> sucesso: marca entregue
      -> falha: aciona retry
```
[ADR-002]

Sequência (c): retry / dead letter
```
falha
  -> registra tentativa
  -> calcula próximo horário (1min / 5min / 30min / 2h / 12h)
  -> repete até 5 tentativas
  -> esgotada a 5ª tentativa: move para webhook_dead_letter
```
[ADR-003]

Não há diagrama gráfico nos documentos-fonte, apenas descrição em prosa nas ADRs citadas; as três sequências acima reproduzem essa descrição de forma esquemática [ADR-001] [ADR-002] [ADR-003].

---

### 5. Contratos públicos (assinaturas, endpoints, headers, exemplos)

Os documentos-fonte (PRD, RFC, ADRs) descrevem os campos e o comportamento de cada endpoint, mas não fixam o schema JSON literal nem os status codes numéricos: isso é NÃO DISCUTIDO na fonte [PRD seção RF-01]. Os status codes abaixo são inferência do padrão já em uso no código (`GET`→200, criação→201, `PATCH`→200, `DELETE`→204, ausência→404, erro de validação→400, papel insuficiente→403), verificado em `src/modules/orders/order.controller.ts` e `src/shared/errors/http-errors.ts`, e marcados como tal, não como decisão do PRD/RFC/ADR. Campos de exemplo não citados literalmente pela fonte estão marcados `(hipótese)`.

**Contrato 1: Cadastro de webhook**
- Tipo: http_endpoint
- Assinatura/Rota: `POST /webhooks`
- Método: POST
- Semântica de status/headers:
  - `201 Created`: webhook criado (inferência do padrão de código, `src/modules/orders/order.controller.ts`)
  - `400 Bad Request`: `WEBHOOK_INVALID_URL`, URL não é HTTPS ou não passa na validação [PRD seção RF-01]
  - `401 Unauthorized`: sem JWT válido (reuso do middleware `authenticate` existente) [PRD seção RF-01]

**Exemplo de requisição**
```json
{
  "customer_id": "cus_456",
  "url": "https://cliente.example.com/webhooks/oms",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

**Exemplo de resposta**
```json
{
  "id": "b2e1c9e4-...",
  "customer_id": "cus_456",
  "url": "https://cliente.example.com/webhooks/oms",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_...",
  "createdAt": "2026-09-09T12:00:00.000Z"
}
```
Nota: `customer_id`, `url` e `statuses` no request, e a devolução da `secret` gerada na criação, vêm do PRD/ADR-004 [PRD seção RF-01] [ADR-004]. `customer_id` vem explicitamente do corpo da requisição, não do JWT: o JWT identifica o usuário operador que faz a chamada, não o customer dono do webhook [PRD seção RF-01]. Os campos `id`, `active` e `createdAt` são hipótese, compostos a partir do padrão de resposta dos demais módulos, não citados literalmente pela fonte.

---

**Contrato 2: Listagem de webhooks**
- Tipo: http_endpoint
- Assinatura/Rota: `GET /webhooks?customer_id={id}` (hipótese quanto ao mecanismo, ver nota)
- Método: GET
- Semântica de status/headers:
  - `200 OK`: lista os webhooks cadastrados do customer [PRD seção RF-01]

**Exemplo de requisição**
```json
{}
```

**Exemplo de resposta**
```json
{
  "data": [
    {
      "id": "b2e1c9e4-...",
      "customer_id": "cus_456",
      "url": "https://cliente.example.com/webhooks/oms",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ]
}
```
Nota: a lista de webhooks do customer é comportamento citado na fonte [PRD seção RF-01]; o envelope `{ "data": [...] }` reaproveita parcialmente o padrão de listagem paginada existente em `src/shared/http/response.ts` (`PaginatedResponse<T>`), mas sem o objeto `pagination` completo do arquivo, hipótese por não haver indicação na fonte de que a lista de webhooks de um customer precise de paginação (volume tipicamente pequeno). A ausência da `secret` na listagem é hipótese consistente com o invariante de nunca expor a secret fora do momento de emissão [ADR-004]. NÃO DISCUTIDO pela fonte: como a listagem é escopada por customer, já que o `customer_id` vem do corpo da requisição no cadastro e não do JWT do usuário operador [PRD seção RF-01]. O filtro `customer_id` como query param é hipótese de composição, não uma decisão registrada.

---

**Contrato 3: Atualização de webhook**
- Tipo: http_endpoint
- Assinatura/Rota: `PATCH /webhooks/:id`
- Método: PATCH
- Semântica de status/headers:
  - `200 OK`: URL, lista de status filtrados ou estado ativo atualizados [PRD seção RF-01]
  - `404 Not Found`: `WEBHOOK_NOT_FOUND`, webhook informado não existe [PRD seção RF-01]
  - `400 Bad Request`: `WEBHOOK_INVALID_URL`, nova URL inválida [PRD seção RF-01]

**Exemplo de requisição**
```json
{
  "statuses": ["DELIVERED"],
  "active": false
}
```

**Exemplo de resposta**
```json
{
  "id": "b2e1c9e4-...",
  "url": "https://cliente.example.com/webhooks/oms",
  "statuses": ["DELIVERED"],
  "active": false
}
```
Nota: a alteração de URL, status filtrados e estado ativo vem do PRD [PRD seção RF-01]; o corpo de resposta espelhando o recurso atualizado é hipótese, pelo padrão de `PATCH` já usado em `src/modules/orders/order.controller.ts` (`changeStatus`). NÃO DISCUTIDO pela fonte: se a operação confere que o `:id` pertence ao `customer_id` de quem chama, já que o CRUD está aberto a qualquer papel autenticado e não a um login do próprio customer [PRD seção RF-01] [PRD seção Fora de escopo].

---

**Contrato 4: Remoção de webhook**
- Tipo: http_endpoint
- Assinatura/Rota: `DELETE /webhooks/:id`
- Método: DELETE
- Semântica de status/headers:
  - `204 No Content`: remoção do webhook, sem corpo de resposta (inferência do padrão de código, `OrderController.delete`) [PRD seção RF-01]
  - `404 Not Found`: `WEBHOOK_NOT_FOUND` [PRD seção RF-01]

**Exemplo de requisição**
```json
{}
```

**Exemplo de resposta**
```json
{}
```
Nota: a remoção do webhook é comportamento citado na fonte [PRD seção RF-01]; o corpo vazio com `204` é inferência do padrão já usado em `src/modules/orders/order.controller.ts`. NÃO DISCUTIDO pela fonte: verificação de que o `:id` pertence ao `customer_id` de quem chama, pela mesma razão do Contrato 3 [PRD seção RF-01].

---

**Contrato 5: Histórico de entregas**
- Tipo: http_endpoint
- Assinatura/Rota: `GET /webhooks/:id/deliveries`
- Método: GET
- Semântica de status/headers:
  - `200 OK`: até 100 registros mais recentes, cada um com resultado, payload enviado, resposta recebida e tempo de resposta [PRD seção RF-08]
  - `404 Not Found`: `WEBHOOK_NOT_FOUND` [PRD seção RF-08]

**Exemplo de requisição**
```json
{}
```

**Exemplo de resposta**
```json
{
  "data": [
    {
      "eventId": "5f2a...",
      "success": true,
      "requestPayload": { "event_type": "order.status_changed" },
      "responseStatus": 200,
      "responseTimeMs": 340
    }
  ]
}
```
Nota: quantidade (até 100), resultado, payload, resposta e tempo de resposta vêm da fonte [PRD seção RF-08]; os nomes literais dos campos do JSON (`eventId`, `success`, `requestPayload`, `responseStatus`, `responseTimeMs`) são hipótese, compostos a partir do que a fonte descreve. NÃO DISCUTIDO pela fonte: verificação de que o `:id` pertence ao `customer_id` de quem chama, pela mesma razão do Contrato 3 [PRD seção RF-08].

---

**Contrato 6: Replay administrativo de dead letter**
- Tipo: http_endpoint
- Assinatura/Rota: `POST /admin/webhooks/dead-letter/:id/replay`
- Método: POST
- Semântica de status/headers:
  - `200 OK`: evento recolocado como pendente na outbox [PRD seção RF-07] [ADR-003]
  - `403 Forbidden`: `WEBHOOK_REPLAY_FORBIDDEN` (hipótese), usuário autenticado sem papel ADMIN, reuso de `requireRole` [PRD seção RF-07] [ADR-006]
  - `404 Not Found`: `WEBHOOK_NOT_FOUND` aplicado ao contexto da dead letter [PRD seção RF-07]

**Exemplo de requisição**
```json
{}
```

**Exemplo de resposta**
```json
{
  "eventId": "5f2a...",
  "status": "pending"
}
```
Nota: restrição a ADMIN, recolocação como pendente preservando o `X-Event-Id` original e registro de auditoria vêm da fonte [PRD seção RF-07] [ADR-003] [ADR-005] [ADR-006]; o formato literal do corpo de resposta é hipótese. NÃO DISCUTIDO pela fonte: se o replay é irrestrito entre customers ou verifica o `customer_id` do evento antes de reprocessar; o ADMIN opera pelo id do item na dead letter, sem menção a filtro por customer, na mesma linha do CRUD de webhook, que também é irrestrito [PRD seção RF-01] [PRD seção RF-07].

---

**Contrato 7: Rotação de secret**
- Tipo: http_endpoint
- Assinatura/Rota: `POST /webhooks/:id/secret/rotate` (hipótese quanto ao path exato; a fonte descreve o fluxo, "o cliente chama o endpoint de rotação de secret do webhook", sem fixar a rota literal) [PRD seção RF-02]
- Método: POST
- Semântica de status/headers:
  - `200 OK`: nova secret gerada; a anterior permanece válida por 24h (grace period) e é invalidada ao fim desse período [PRD seção RF-02] [ADR-004]
  - `404 Not Found`: `WEBHOOK_NOT_FOUND` [PRD seção RF-01]
  - `422 Unprocessable Entity`: `WEBHOOK_SECRET_REQUIRED` quando não há secret ativa a rotacionar [PRD seção RF-02] [PRD seção RF-06]

**Exemplo de requisição**
```json
{}
```

**Exemplo de resposta**
```json
{
  "id": "b2e1c9e4-...",
  "secret": "whsec_novo...",
  "previousSecretValidUntil": "2026-09-10T12:00:00.000Z"
}
```
Nota: a geração de nova secret e o grace period de 24h para a secret anterior vêm da fonte [PRD seção RF-02] [PRD seção Critérios de aceitação] [ADR-004]. O nome exato do campo `previousSecretValidUntil` é hipótese; o invariante de nunca expor a secret fora do momento de emissão se aplica também aqui, então a secret anterior nunca é devolvida neste corpo, só a nova [ADR-004]. Qual secret assina durante o grace period, e a distinção entre rotação de rotina e rotação por vazamento, permanecem NÃO DISCUTIDO [ADR-004] [RFC seção "Questões em aberto"].

---

**Contrato 8: Entrega do evento ao endpoint do cliente**
- Tipo: webhook (chamada HTTP feita pelo worker ao endpoint HTTPS cadastrado pelo cliente, não um endpoint da plataforma)
- Assinatura/Rota: URL cadastrada pelo cliente no Contrato 1
- Método: POST
- Semântica de status/headers:
  - `X-Event-Id`: UUID único gerado quando o evento entra na outbox, usado pelo cliente para deduplicação [ADR-005]
  - `X-Signature`: HMAC-SHA256 do corpo da requisição, calculado com a secret ativa do endpoint [ADR-004]
  - `X-Timestamp` e `X-Webhook-Id`: enviados junto à assinatura; se `X-Timestamp` deveria entrar no cálculo do HMAC é questão em aberto, não resolvida [PRD seção RF-06] [ADR-004]
  - `Content-Type: application/json` [PRD seção RF-04]
  - Resposta de sucesso do cliente (classe 2xx, faixa exata não fixada pela fonte) marca o evento como entregue; qualquer outra resposta ou timeout de 10s conta como falha [PRD seção RF-04]

**Exemplo de requisição**
```json
{
  "event_id": "5f2a1c8e-...",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-09T12:00:05.123Z",
  "order_id": "ord_123",
  "order_number": "OMS-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "cus_456",
  "total_cents": 15990
}
```

**Exemplo de resposta**
```json
{}
```
Nota: todos os campos do corpo enviado vêm da fonte [PRD seção RF-04] [Critérios de aceitação]; a resposta é do endpoint do cliente, fora do controle da plataforma, por isso não há corpo definido.

Limites: payload até 64 KB, acima disso não é enviado, gera erro registrado sem truncamento [PRD seção Escopo] [PRD seção RF-04]. Timeout de 10 segundos por requisição [PRD seção RF-04] [ADR-003]. Rate limiting de saída: ADIADO, "observar e decidir depois" [PRD seção Ponto em aberto] [RFC seção "Questões em aberto"].

---

### 6. Erros, exceções e fallback

**Matriz de erros**

| Código | Condição | Tratamento |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | webhook ou item de dead letter informado não existe | `404`, tratado pelo error middleware sem alteração [PRD seção RF-01] [PRD seção RF-07] [PRD seção RF-08] [ADR-006]. O PRD pede explicitamente que se siga "o padrão `NotFoundError` já existente" [PRD seção RF-01]; isso é seguido em semântica (`404`, mensagem de recurso não encontrado), mas não em herança de classe: `NotFoundError` (`src/shared/errors/http-errors.ts`) fixa `errorCode = 'NOT_FOUND'` no próprio construtor, sem parâmetro, diferente de `ConflictError`/`UnprocessableEntityError`/`BadRequestError`, que aceitam código customizado. Observação (hipótese): para produzir `WEBHOOK_NOT_FOUND` sem alterar `NotFoundError`, a classe do módulo de webhooks constrói `AppError` diretamente com `404` em vez de estender `NotFoundError` |
| `WEBHOOK_INVALID_URL` | URL não é HTTPS ou não passa na validação de schema | herda de `BadRequestError`, `400`; não de `ValidationError`, que também fixa `errorCode = 'VALIDATION_ERROR'` sem parâmetro no construtor, mesma limitação de `NotFoundError` acima (inferência de `src/shared/errors/http-errors.ts`) [PRD seção RF-01] [ADR-006] |
| `WEBHOOK_SECRET_REQUIRED` | operação depende de secret ativa e ela não existe | herda de `UnprocessableEntityError`, `422` (hipótese por analogia ao padrão de `InsufficientStockError` em `src/shared/errors/http-errors.ts`, não definido pela fonte) [PRD seção RF-01] [PRD seção RF-02] [PRD seção RF-06] |
| `WEBHOOK_REPLAY_FORBIDDEN` (hipótese) | usuário autenticado sem papel ADMIN no replay | `403`; a checagem de papel reusa `requireRole` como está, mas o PRD exige que todo erro do módulo carregue prefixo `WEBHOOK_` [PRD seção Critérios de aceitação], então o código é composição para cumprir esse critério. Mesma limitação de `NotFoundError`: `ForbiddenError` (`src/shared/errors/http-errors.ts`) fixa `errorCode = 'FORBIDDEN'` no construtor, sem parâmetro; a classe do módulo constrói `AppError` diretamente com `403` em vez de estender `ForbiddenError` [PRD seção RF-07] [ADR-006] |
| `WEBHOOK_OUTBOX_INSERT_FAILED` (hipótese) | falha de inserção na outbox dentro da transação do `changeStatus` | propaga como falha de transação, tratada pelo error middleware existente; o código específico não é definido pela fonte, composto aqui só para manter a tabela 100% `WEBHOOK_*`, conforme exigido [PRD seção Critérios de aceitação] [PRD seção RF-03] [RFC seção "Impacto e riscos"] |

Todos os códigos de erro do módulo usam prefixo `WEBHOOK_`, herdam de `AppError` (via subclasses HTTP intermediárias quando há equivalente) e são tratados pelo `error.middleware.ts` sem alteração nele, conforme exigido pelo PRD [PRD seção Critérios de aceitação] [ADR-006]. `WEBHOOK_REPLAY_FORBIDDEN` e `WEBHOOK_OUTBOX_INSERT_FAILED` são os dois únicos códigos da tabela compostos por hipótese, marcados como tal; os demais são citação direta do PRD.

**Estratégias de resiliência**
- Timeout: 10 segundos por tentativa de entrega HTTP [PRD seção RF-04] [ADR-003]
- Retry: backoff exponencial em 5 tentativas, progressão 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas, totalizando cerca de 15 horas [PRD seção RF-05] [ADR-003]. Ambiguidade não resolvida entre "5 tentativas com 4 esperas" e "1 tentativa inicial mais 5 reagendamentos (6 envios)", registrada como questão em aberto [ADR-003] [PRD seção Ponto em aberto]
- Backoff: ver progressão acima; não há circuit breaker por endpoint definido pela fonte (NÃO DISCUTIDO)
- Isolamento: retry de um evento não bloqueia outros; entrega sequencial por `order_id`, concorrente entre pedidos distintos, para que um cliente lento não atrase eventos de outros customers [ADR-002]

**Política de fallback**
Esgotada a 5ª tentativa, o evento é movido para `webhook_dead_letter` (tabela separada), com payload, motivo da falha e timestamp, e só é reprocessado por ação manual administrativa via replay [ADR-003].

**Invariantes**
- Nunca existe status de pedido alterado sem evento correspondente gravado, garantido por transação única entre `changeStatus` e `publishWebhookEvent`, com rollback conjunto em falha de inserção [ADR-001] [PRD seção Critérios de aceitação]
- Payload gravado como snapshot no momento da inserção, nunca recalculado no envio [ADR-001]
- Secret nunca exposta fora do momento de emissão (criação e rotação), nunca em respostas de leitura [ADR-004]
- Replay preserva o `X-Event-Id` original, nunca gera identificador novo [ADR-005]
- Entrega é at-least-once, nunca exactly-once; duplicidade é esperada e de responsabilidade de deduplicação do cliente [ADR-005]
- Payload acima de 64 KB nunca é enviado nem truncado, apenas gera erro registrado [PRD seção RF-04]
- A ordenação por `order_id` é invariante apenas sob a premissa de worker único, não garantida por mecanismo técnico: risco assumido e documentado, não invariante forte [ADR-002] [RFC seção "Impacto e riscos"]

---

### 7. Observabilidade

**Métricas**
- Proporção entre eventos gravados na outbox e mudanças de status com webhook inscrito, meta de 100% [PRD seção Objetivos e métricas]
- Número de tentativas de entrega e janela total de backoff (meta: 5 tentativas, ~15h) [PRD seção Objetivos e métricas]
- Tempo entre o commit da mudança de status e a chegada da requisição no endpoint do cliente (meta: <10s) [PRD seção Objetivos e métricas]
- Histórico de entregas com resultado, payload, resposta e tempo de resposta dos últimos 100 envios, exposto ao cliente via Contrato 5 [PRD seção Requisitos não funcionais - Observabilidade]
- (hipótese, não definida pela fonte) contagem de eventos publicados, taxa de entrega bem-sucedida, tamanho/crescimento da dead letter, tentativas por evento como métricas internas de operação: o PRD registra essa lacuna e adota como hipótese "observabilidade mínima de logs estruturados, métricas de erro por endpoint e tracing distribuído ponta a ponta" [PRD seção Requisitos não funcionais - Observabilidade]

**Logs**
- Pino, já presente em todo o projeto, sem introduzir ferramenta nova [PRD seção Requisitos não funcionais - Observabilidade]
- `redactPaths` em `src/shared/logger/index.ts` hoje cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`; passa a ser estendido para cobrir também a secret do webhook e a assinatura calculada [ADR-004]
- `createLogger()` hoje não aceita parâmetro e fixa `service: 'order-management-api'`; precisa ser parametrizada para receber o nome do serviço, dando identidade própria ao worker na saída de log, para que os registros dos dois processos não se misturem [ADR-006]
- Formato exato de campos estruturados além do que já existe no Pino: NÃO DISCUTIDO

**Tracing**
- NÃO DISCUTIDO como decisão fechada. Único registro é a hipótese do PRD de adotar "tracing distribuído ponta a ponta" como default, marcada como hipótese por não ter sido definida pela fonte [PRD seção Requisitos não funcionais - Observabilidade]. Nenhuma RFC ou ADR menciona spans, correlation-id ou ferramenta de tracing.

**Dashboards e alertas**
- Não há definição de alerta ou painel na fonte
- Comportamento do worker parado por completo (restart, health check, alerta) não definido, permanece em aberto [PRD seção Requisitos não funcionais - Disponibilidade]
- Risco correlato: sem definição de restart e alerta do worker, "a falha é silenciosa, porque nenhum evento vai para a dead letter, todos ficam pendentes" [RFC seção "Questões em aberto"]
- Crescimento da dead letter como gatilho de alerta: NÃO DISCUTIDO; a RFC registra apenas o risco de crescimento indefinido da outbox pela ausência de arquivamento, sem propor alerta associado [RFC seção "Impacto e riscos"]

---

### 8. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| --- | --- | --- |
| MySQL via Prisma | não definida pela fonte | banco e ORM já existentes, sem infraestrutura nova [PRD seção Dependências - Técnica] |
| `src/modules/orders/order.service.ts` (`changeStatus`) | não se aplica (código existente) | ponto de integração transacional já existente [PRD seção Dependências - Técnica] [ADR-001] |
| `AppError` / `src/shared/errors/http-errors.ts` | não se aplica | base das classes de erro `WEBHOOK_*` [PRD seção Dependências - Técnica] [ADR-006] |
| `src/middlewares/error.middleware.ts` | não se aplica | trata os erros do módulo sem alteração [PRD seção Dependências - Técnica] [ADR-006] |
| `src/shared/logger/index.ts` | não se aplica | logger Pino, com `redactPaths` estendido e `createLogger()` parametrizada [PRD seção Dependências - Técnica] [ADR-006] [ADR-004] |
| `src/middlewares/auth.middleware.ts` (`requireRole`) | não se aplica | protege o replay administrativo [PRD seção Dependências - Técnica] [ADR-006] |
| `PrismaClient` (instância própria do worker) | não definida pela fonte | mesma `DATABASE_URL` da API [ADR-002] |

Versão mínima de Node, Prisma ou qualquer biblioteca: NÃO DISCUTIDO, a fonte não cita números de versão.

**Garantias de compatibilidade**
- `src/middlewares/error.middleware.ts` trata os erros do módulo sem qualquer alteração, porque já despacha por `instanceof AppError` [ADR-006]
- O padrão de módulo (controller, service, repository, routes, schemas) é preservado, seguindo `src/modules/orders/` como referência [ADR-006]
- `requireRole` e `authenticate` são consumidos como estão, sem alteração [ADR-006]
- A feature é aditiva para clientes já integrados: nenhuma quebra, e o `GET /orders` segue funcionando [RFC seção "Impacto e riscos"]
- Alterações necessárias ao código compartilhado são limitadas e explícitas: parametrização de `createLogger()` e extensão de `redactPaths`; fora isso, nenhuma infraestrutura ou biblioteca nova é introduzida [ADR-006] [ADR-004]

---

### 9. Integração com o sistema existente

**`src/modules/orders/order.service.ts`**
- O `changeStatus`, que já roda dentro de `this.prisma.$transaction` fazendo leitura do pedido, ajuste de `stockQuantity` item a item, update do pedido, insert de histórico e releitura final com relações, passa a chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` com o client `tx` da transação corrente, depois do `tx.order.update` e antes do fim da transação [ADR-001]

**`src/shared/errors/app-error.ts`**
- Define a classe base `AppError`, com `statusCode`, `errorCode` e `details`, da qual as subclasses HTTP intermediárias herdam; as classes de erro `WEBHOOK_*` do novo módulo seguem essa mesma base [ADR-006]

**`src/shared/errors/http-errors.ts`**
- O padrão real do projeto não herda de `AppError` diretamente quando existe equivalente HTTP intermediário com suporte a código customizado (`InvalidStatusTransitionError` estende `ConflictError`, `InsufficientStockError` estende `UnprocessableEntityError`); as classes de erro do módulo de webhooks herdam dessas subclasses quando o equivalente aceita `code` customizado no construtor, com prefixo próprio `WEBHOOK_` [ADR-006]. Exceção: `NotFoundError` e `ForbiddenError` fixam o `errorCode` no próprio construtor, sem parâmetro; para `WEBHOOK_NOT_FOUND` e `WEBHOOK_REPLAY_FORBIDDEN`, a classe do módulo constrói `AppError` diretamente com o `statusCode` correspondente, em vez de estender essas duas (ver seção 6)

**`src/middlewares/error.middleware.ts`**
- Trata `AppError` pelo `instanceof`, depois `ZodError`, depois `Prisma.PrismaClientKnownRequestError`, sem menção a domínio específico; trata os erros do módulo de webhooks sem qualquer alteração nele [ADR-006]

**`src/middlewares/auth.middleware.ts`**
- Contém `authenticate` e `requireRole`, operando sobre o `AuthUser` extraído do JWT; o endpoint de replay administrativo reusa ambos, restringindo o acesso ao papel `ADMIN` [ADR-003] [ADR-006]

**`src/shared/logger/index.ts`**
- Exporta a factory `createLogger()` e uma instância singleton `logger`; `redactPaths` hoje cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, sem alcançar secret ou assinatura, passa a ser estendido para cobrir ambos; `createLogger()` precisa ser parametrizada para aceitar nome de serviço, hoje fixo em `service: 'order-management-api'`, para que o worker tenha identidade própria na saída de log [ADR-004] [ADR-006]

**`src/app.ts`**
- `buildControllers` monta `OrderService` por injeção manual e precisa ser alterado para fiar a nova dependência; o módulo de webhooks segue o mesmo padrão de injeção manual de construtor (repository, service, controller) usado para `orders`, `products`, `customers` e `users`, montado antes de registrar rotas em `buildApiRouter` [ADR-001] [ADR-006]

**`src/server.ts`**
- Citado como precedente de entry point único do processo da API, ao lado do qual o worker ganha entry point próprio; o caminho do entry point do worker em si é artefato a criar, ainda não existente no repositório [ADR-002]

---

### 10. Critérios de aceite técnicos

- Mudar o status para um status inscrito insere linha em `webhook_outbox` dentro da mesma transação; rollback não deixa linha na outbox. Mudar para status não inscrito não insere linha [PRD seção Critérios de aceitação]
- A requisição de entrega contém os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`, e o corpo contém `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` [PRD seção Critérios de aceitação]
- `X-Signature` corresponde ao HMAC-SHA256 do corpo calculado com a secret do endpoint, verificável recalculando no teste [PRD seção Critérios de aceitação]
- Cadastro com URL HTTP é recusado, com HTTPS é aceito [PRD seção Critérios de aceitação]
- Payload acima de 64 KB não é enviado, resulta em erro registrado, sem truncamento [PRD seção Critérios de aceitação]
- Rotação de secret gera nova secret, mantém a anterior válida por 24h e invalida após esse período [PRD seção Critérios de aceitação]
- `GET /webhooks/:id/deliveries` retorna até 100 registros com resultado, payload, resposta, tempo de resposta [PRD seção Critérios de aceitação]
- Todos os erros do módulo usam prefixo `WEBHOOK_`, herdam de `AppError`, tratados pelo error middleware sem alteração nele [PRD seção Critérios de aceitação]
- Evento pendente entregue em menos de 10 segundos do commit, no cenário sem falha [PRD seção Critérios de aceitação]
- Falhas sucessivas reagendam respeitando 1min/5min/30min/2h/12h; a falha da última tentativa move para `webhook_dead_letter` com payload, motivo e timestamp [PRD seção Critérios de aceitação]
- `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento como pendente quando chamado por ADMIN, retorna erro de autorização sem esse papel, e gera registro de log identificando quem executou [PRD seção Critérios de aceitação]
- Dois eventos consecutivos do mesmo pedido chegam na ordem de `created_at` da outbox, em regime de worker único [PRD seção Critérios de aceitação]
- Worker continua processando após reinício da API, comprovando processo separado [PRD seção Critérios de aceitação]

---

### 11. Riscos e mitigação

### Endpoint do cliente indisponível ou lento esgota as tentativas sem entrega

- **Probabilidade:** média
- **Impacto:** cliente deixa de ser notificado e volta a depender de consulta manual
- **Mitigação:**
    - Retry com backoff cobrindo ~15h
    - Timeout de 10s por tentativa
- **Plano de contingência:** evento preservado em `webhook_dead_letter`, reprocessável via replay administrativo [PRD seção Riscos e mitigação]

### Vazamento da secret de um cliente permite forjar eventos assinados

- **Probabilidade:** média
- **Impacto:** terceiro forja notificações válidas para o cliente afetado
- **Mitigação:**
    - Secret única por endpoint
    - Rotação disponível pela API
    - Revisão de segurança do código antes do deploy
- **Plano de contingência:** cliente solicita rotação; secret comprometida deixa de ser válida ao fim das 24h de grace period. Risco correlato não resolvido: uma secret sabidamente comprometida segue válida por até 24h depois do pedido de troca, porque a regra de grace period não distingue rotação de rotina de rotação por incidente [PRD seção Riscos e mitigação] [ADR-004]

### Escalar para múltiplos workers quebra a ordenação de eventos do mesmo pedido

- **Probabilidade:** baixa
- **Impacto:** cliente pode receber mudanças de status fora de ordem
- **Mitigação:**
    - Manter worker único nesta entrega
    - Documentar ausência de ordenação global
- **Plano de contingência:** particionamento por `order_id` ou lock pessimista antes de introduzir um segundo worker, adiado para fase futura. Nenhum mecanismo técnico impede duas instâncias coexistindo por acidente de deploy hoje; um claim atômico exigiria SQL cru, sem precedente em `src/`, permanece questão em aberto [PRD seção Riscos e mitigação] [ADR-002]

### Volume alto de mudanças de status sobrecarrega o endpoint do cliente

- **Probabilidade:** média
- **Impacto:** endpoint do cliente recusa ou demora, gerando falhas e consumo de retries
- **Mitigação:**
    - Nenhuma implementada nesta entrega; decisão de observar e agir depois
- **Plano de contingência:** hipótese: sobrecarga absorvida por retry e dead letter; decisão de rate limiting reaberta com dados observados em produção [PRD seção Riscos e mitigação]

### Cliente que não implementa deduplicação processa o mesmo evento mais de uma vez

- **Probabilidade:** média
- **Impacto:** inconsistência no sistema do cliente
- **Mitigação:**
    - `X-Event-Id` único por evento
    - Documentação da garantia at-least-once
- **Plano de contingência:** nenhuma ação corretiva além de documentação e suporte; risco residual aceito conscientemente [PRD seção Riscos e mitigação] [ADR-005]

### Falha na inserção do evento na outbox impede a mudança de status do pedido

- **Probabilidade:** baixa
- **Impacto:** operação de mudança de status falha para o usuário, por motivo alheio ao domínio de pedidos
- **Mitigação:**
    - Inserção do evento na mesma transação do `changeStatus`, com rollback conjunto
    - Payload como snapshot, sem dependência externa no momento da inserção
- **Plano de contingência:** hipótese: erro propaga pelo error middleware existente, cabendo ao chamador repetir a operação; código de erro específico devolvido ao chamador não foi definido pela fonte [PRD seção Riscos e mitigação] [ADR-001] [RFC seção "Impacto e riscos"]

### Vazamento de secret ou assinatura pelo log estruturado

- **Probabilidade:** baixa
- **Impacto:** exposição de segredo criptográfico em log, equivalente a vazamento da secret
- **Mitigação:**
    - Extensão de `redactPaths` em `src/shared/logger/index.ts` para cobrir secret e assinatura
- **Plano de contingência:** risco levantado no debate da RFC (`NOVO`) e só se fecha quando a extensão de `redactPaths` existir de fato no código [RFC seção "Impacto e riscos"] [ADR-004]

### Crescimento indefinido da tabela de outbox degrada o polling do worker

- **Probabilidade:** baixa
- **Impacto:** consulta de pendentes a cada 2s compete com tabela cada vez maior, sem arquivamento
- **Mitigação:**
    - Nenhuma implementada nesta entrega
- **Plano de contingência:** arquivamento das linhas entregues (~30 dias) fica como próxima fase [RFC seção "Impacto e riscos"] [PRD seção Fora de escopo]

### Duas instâncias do worker coexistindo tornam entregas legítimas indistinguíveis de reentrega maliciosa

- **Probabilidade:** baixa
- **Impacto:** duas entregas legítimas do mesmo evento chegam com o mesmo `X-Event-Id`, indistinguíveis de uma reentrega capturada por terceiro
- **Mitigação:**
    - Nenhuma implementada nesta entrega
- **Plano de contingência:** levantado no debate como agravante de segurança sobre a garantia at-least-once, permanece questão em aberto, não resolvido [ADR-005]
