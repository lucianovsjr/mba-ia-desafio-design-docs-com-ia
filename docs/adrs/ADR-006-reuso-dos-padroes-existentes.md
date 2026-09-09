# ADR-006: Reuso dos padrões existentes do projeto

Data: 2026-09-01
RFC de origem: docs/RFC.md
Decisores: Larissa (Tech Lead), Bruno (Engenheiro Pleno, time de Pedidos), Sofia (Engenheira de Segurança)

## Status

Aceito

## Contexto

O time não quer provisionar infraestrutura nova para esta entrega (PRD-DEC-01, PRD-MET-04).
`[TRANSCRICAO 09:27]` fixa o padrão de módulo a seguir: "A gente tem um padrão claro na
codebase. Cada domínio é um módulo em src/modules com controller, service, repository,
routes e schemas. Webhook vai seguir igual", com `src/modules/orders/` como exemplo já
existente do padrão, composto por `order.controller.ts`, `order.repository.ts`,
`order.routes.ts`, `order.schemas.ts`, `order.service.ts` e `order.status.ts` (PRD-CTX-07,
PRD-ARQ-05).

`[TRANSCRICAO 09:28]` fixa o reuso das classes de erro: "Tem classe AppError, classes
específicas tipo InsufficientStockError, InvalidStatusTransitionError. Todas usam código
tipo INSUFFICIENT_STOCK, INVALID_STATUS_TRANSITION. Quero seguir igual pra webhook. Códigos
tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc" (PRD-ESC-16a).
Conferindo o código: `src/shared/errors/app-error.ts` define a classe base `AppError`, com
`statusCode`, `errorCode` e `details`. `src/shared/errors/http-errors.ts` mostra que o
padrão real do projeto não herda de `AppError` diretamente quando existe um equivalente
HTTP intermediário: `InvalidStatusTransitionError` estende `ConflictError`, e
`InsufficientStockError` estende `UnprocessableEntityError`, ambos por sua vez subclasses de
`AppError`. Herança direta de `AppError` fica reservada às subclasses HTTP intermediárias em
si, como `ConflictError`, `NotFoundError` e `UnprocessableEntityError`.

`[TRANSCRICAO 09:29]` fixa o reuso do logger e do error middleware: "o logger, que é Pino, já
tá no projeto inteiro. Não vamos botar nada novo. O middleware de erro centralizado já trata
AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada" (PRD-ESC-16b).
Conferindo: `src/middlewares/error.middleware.ts` trata `AppError` pelo `instanceof`, depois
`ZodError`, depois `Prisma.PrismaClientKnownRequestError` para os códigos `P2002` e `P2025`,
sem nenhuma menção a um domínio específico, o que confirma que nenhuma alteração é
necessária para o módulo de webhooks ser tratado por ele.

`[TRANSCRICAO 09:30]` fixa que o worker usa uma instância própria de `PrismaClient`, e
`[TRANSCRICAO 09:36]` fixa que o replay administrativo precisa logar quem executou, para
auditoria: "E o endpoint de admin tem que logar quem fez o replay, pra auditoria." O acesso
ao replay é restrito ao papel ADMIN reusando `requireRole`, que existe em
`src/middlewares/auth.middleware.ts` junto com `authenticate`, ambos operando sobre o
`AuthUser` extraído do JWT.

O debate da RFC-001 trouxe dois ajustes de precisão sobre esse reuso, e uma decisão nova.
DEV-03 corrigiu a leitura inicial da RFC de que o módulo herdaria de `AppError` direto: o
padrão real, verificado em `src/shared/errors/http-errors.ts`, é herdar das subclasses HTTP
intermediárias quando existe equivalente. DEV-05 apontou que `src/shared/logger/index.ts`
exporta tanto a factory `createLogger()` quanto uma instância singleton `logger`; se o worker
importar o singleton, seus logs ficam marcados como se fossem da API, tornando os dois
processos indistinguíveis na saída de log. Verificando o arquivo, porém, a factory hoje não
recebe parâmetro nenhum, e o valor `service: 'order-management-api'` está fixo dentro dela, no
campo `base`. Trocar o singleton pela factory, sozinho, não separa os dois processos.

DEV-04 trouxe a decisão nova: não existe, em `prisma/schema.prisma` nem em `src/`, nenhum
model ou utilitário de auditoria reaproveitável (confirmado por inspeção direta dos dois). O
único registro de "quem fez o quê" hoje é o log de aplicação via Pino, que não é consultável
como trilha de auditoria estruturada. A segurança deu peso a esse ponto na rodada 2 do
debate: o replay administrativo é a única operação da feature protegida por autorização
reforçada, já que o CRUD de configuração de webhook é aberto a qualquer papel autenticado
(PRD-DEC-08); se a auditoria desse replay ficasse diluída no volume geral de log
compartilhado, o único controle reforçado da feature perderia rastreabilidade própria. A
exigência de um prazo de retenção para essa trilha foi levantada e depois retirada pela
própria segurança na rodada 2, por falta de âncora nas quatro fontes, e não entra nesta
decisão.

## Decisão

O módulo `src/modules/webhooks` segue o padrão já estabelecido em `src/modules/orders/`, com
controller, service, repository, routes e schemas (PRD-CTX-07, PRD-ARQ-05,
`[TRANSCRICAO 09:27]`).

As classes de erro do módulo herdam das subclasses HTTP intermediárias de
`src/shared/errors/http-errors.ts` quando existe equivalente para o caso, seguindo o mesmo
padrão de `InvalidStatusTransitionError` e `InsufficientStockError`, e usam um prefixo
próprio de código no padrão dos já existentes, sem listar aqui o conjunto de códigos, que é
detalhe de FDD (PRD-ESC-16a, `[TRANSCRICAO 09:28]`). O `src/middlewares/error.middleware.ts`
trata esses erros sem qualquer alteração, porque já despacha por `instanceof AppError`
(PRD-ESC-16b, `[TRANSCRICAO 09:29]`).

O worker tem identidade de serviço própria na saída de log, para que os registros dos dois
processos não fiquem misturados (PRD-NFR-OBS-01). Como `createLogger()` em
`src/shared/logger/index.ts` hoje não aceita parâmetro e fixa `service:
'order-management-api'` no campo `base`, alcançar isso exige parametrizar a factory para
receber o nome do serviço, e não apenas trocar o singleton pela factory no worker. Esta é a
única alteração que o reuso do logger impõe ao código compartilhado; o restante do módulo
consome o logger como ele já é.

O endpoint de replay administrativo reusa `authenticate` e `requireRole` de
`src/middlewares/auth.middleware.ts`, restringindo o acesso ao papel `ADMIN`
(PRD-ESC-08b, PRD-FR-07b). A montagem de dependências do módulo em `src/app.ts` segue o
mesmo padrão de `buildControllers`, que hoje instancia repository, service e controller de
cada domínio por injeção manual de construtor e os monta antes de registrar as rotas em
`buildApiRouter`, como já ocorre para `orders`, `products`, `customers` e `users`.

Como limite explícito desse reuso, a trilha de auditoria do replay administrativo é uma peça
nova do módulo de webhooks, consultável, e não uma linha solta no log compartilhado da
aplicação. Não existe hoje, nem em `prisma/schema.prisma` nem em `src/`, nenhum model ou
utilitário de auditoria a reaproveitar, e diluir o único controle reforçado da feature no
volume geral de log esvaziaria a rastreabilidade que a segurança exigiu para essa operação.
Esta decisão não fixa prazo de retenção para essa trilha, por falta de fonte para esse
número.

## Alternativas Consideradas

**Introduzir biblioteca ou infraestrutura nova para dar suporte ao módulo.** Descartada pela
restrição organizacional de não provisionar nada novo, dado o tamanho do time (PRD-MET-04).
`[TRANSCRICAO 09:07]`: "a gente é um time pequeno. Subir Redis Cluster pra isso é
overengineering." `[TRANSCRICAO 09:30]`: "reuso máximo do que já existe. AppError, Pino,
error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro."

**Registrar a auditoria do replay apoiada apenas no log compartilhado via Pino, sem peça
nova.** Foi a leitura inicial derrubada no debate: `[TRANSCRICAO 09:36]` só fixa que o
replay precisa "logar quem fez o replay, pra auditoria", sem dizer se isso é uma linha de
log ou uma trilha própria. O debate (DEV-04, reforçado pela segurança) mostrou que essa
leitura esvaziaria o único controle reforçado da feature, já que o CRUD de configuração é
aberto a qualquer papel autenticado (PRD-DEC-08) e o replay é a única operação com
autorização adicional a auditar de forma distinguível.

## Consequências

### Positivas

- A feature não introduz biblioteca, ferramenta ou infraestrutura nova, cumprindo a
  restrição organizacional do time (PRD-DEC-01, PRD-DEP-04)
- Erros, autenticação, autorização e tratamento de erro HTTP seguem exatamente o padrão já
  em produção, sem exigir mudança no `error.middleware.ts` nem em `auth.middleware.ts`
- A trilha de auditoria do replay fica consultável como parte do módulo, e não perdida no
  volume geral de log, preservando a rastreabilidade do único controle reforçado da feature

### Negativas

- O módulo fica amarrado às escolhas já feitas no projeto, incluindo o padrão de injeção
  manual de construtor em `src/app.ts`, o que significa que qualquer limitação desse padrão,
  como o acoplamento explícito entre `buildControllers` e cada módulo, se propaga também
  para o webhooks
- O limite do reuso custa trabalho novo justamente na peça de auditoria: ela foi tratada no
  rascunho inicial como consequência do reuso do logger existente, mas na prática exige
  model, persistência e consulta próprios, porque não há nada equivalente para reaproveitar
  no projeto hoje
- Dar identidade de serviço própria ao worker obriga a mexer em `src/shared/logger/index.ts`,
  que hoje serve a aplicação inteira, para parametrizar o nome do serviço. É alteração pequena,
  mas é a única que esta decisão de reuso impõe a código compartilhado, e quem implementar o
  worker precisa saber disso, ou os logs dos dois processos ficam indistinguíveis

## Trade-off aceito

Troca-se a velocidade de reusar cegamente tudo o que já existe por uma trilha de auditoria
construída do zero: aceita-se o custo de modelar e persistir um registro próprio para o
replay administrativo, em vez de apoiar essa auditoria no log compartilhado, porque o replay
é a única operação da feature com controle de autorização reforçado e precisa continuar
rastreável mesmo quando o restante do módulo reusa tudo o que o projeto já oferece.
