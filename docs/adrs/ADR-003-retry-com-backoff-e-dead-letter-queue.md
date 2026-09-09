# ADR-003: Retry com backoff exponencial e dead letter queue

Data: 2026-09-01
RFC de origem: docs/RFC.md
Decisores: Larissa (Tech Lead), Diego (Engenheiro Sênior, time de Plataforma), Bruno (Engenheiro Pleno, time de Pedidos)

## Status

Aceito

## Contexto

A ADR-002 fixa que o worker consulta a outbox em polling e entrega os eventos por HTTP POST
com timeout de 10 segundos por tentativa (PRD-FR-04d, `[TRANSCRICAO 09:42]`). Falta decidir
o que acontece quando a entrega falha: o endpoint do cliente pode estar indisponível ou
lento, com precedente concreto de indisponibilidade planejada de 2 horas (PRD-RISK-01).

`[TRANSCRICAO 09:15]` registra a escolha do mecanismo: "Backoff exponencial. Tenta de novo
depois de algum tempo, vai aumentando o intervalo, e depois de um teto de tentativas
considera falha permanente e move pra DLQ", rejeitando ali mesmo o retry indefinido: "Eu
sugiro 5. Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de
evento ficar pendurado pra sempre se o cliente sumiu." `[TRANSCRICAO 09:16]` registra a
rejeição de um teto menor: "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente
retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade
de duas horas em manutenção planejada." `[TRANSCRICAO 09:17]` fixa a progressão: "1 minuto,
5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e
última tentativa", com o próprio Marcos ponderando ali: "Se um cliente meu cair por 15
horas, ele já tá com problema sério dele. Acho aceitável" (PRD-DEC-07, PRD-MET-02).

`[TRANSCRICAO 09:18]` fixa a dead letter em tabela separada e o reprocessamento manual: "Eu
fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp.
Mais limpa a leitura da outbox principal (...) Manual via endpoint admin. Tipo um POST
/admin/webhooks/dead-letter/:id/replay. Recoloca na outbox como pendente" (PRD-ESC-07,
PRD-ESC-08a). O acesso ao replay é restrito ao papel ADMIN reusando o `requireRole` já
existente em `src/middlewares/auth.middleware.ts` (PRD-ESC-08b, PRD-FR-07b).

Esta decisão não nasceu de um ponto do debate da RFC-001: nenhum dos três debatedores levantou
ponto sobre ela, e por isso ela não tem linha na tabela "Pontos consolidados" da ata nem recebe
a classificação `ACORDADO`. Ela é aberta pela via estrutural descrita em
`docs/prompts/rfc-fluxo.md`, aparece na tabela "Decisões a registrar" da ata como
`ADR-TBD-06`, com a origem declarada como estrutural, e é ancorada direto no PRD e na
transcrição.

## Decisão

Em caso de falha de entrega, o worker reagenda o evento seguindo a progressão de backoff de
1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, totalizando 5 tentativas e uma janela de
cerca de 15 horas entre a primeira falha e a última tentativa (PRD-DEC-07, PRD-ESC-06,
PRD-FR-05a, `[TRANSCRICAO 09:17]`). Timeout de resposta ou resposta de erro do endpoint do
cliente contam como falha e disparam o reagendamento (PRD-FR-04d).

Esgotada a quinta tentativa sem sucesso, o evento é movido para a tabela
`webhook_dead_letter`, separada da `webhook_outbox`, com payload, motivo da falha e
timestamp (PRD-ESC-07, PRD-FR-05b). O reprocessamento de um item da dead letter é manual,
por `POST /admin/webhooks/dead-letter/:id/replay`, restrito ao papel ADMIN pelo middleware
`requireRole` já existente em `src/middlewares/auth.middleware.ts` (PRD-ESC-08a, PRD-ESC-08b,
PRD-FR-07a, PRD-FR-07b).

Este ADR não decide o que acontece com o identificador do evento original no momento do
replay: essa decisão é da ADR-005. Também não decide a forma da trilha de auditoria de quem
executou o replay: essa decisão é da ADR-006.

Este ADR não resolve uma ambiguidade da fonte: não ficou definido se as 5 tentativas
correspondem a 5 tentativas com 4 esperas entre elas, ou a 1 tentativa inicial seguida de 5
reagendamentos, totalizando 6 envios. `[TRANSCRICAO 09:17]` cita "5 tentativas" e também "5
intervalos" sem reconciliar os dois números, e a reunião não voltou ao ponto (PRD-OPEN-05).
Essa ambiguidade permanece como questão em aberto da RFC, e não é resolvida por este ADR.

## Alternativas Consideradas

**Retry com 3 tentativas em janela curta.** Descartado porque mataria o evento durante uma
indisponibilidade planejada do cliente, com precedente real de 2 horas fora do ar
(PRD-FESC-09). `[TRANSCRICAO 09:16]`: "3 é pouco. Se o cliente teve indisponibilidade de
manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com
indisponibilidade de duas horas em manutenção planejada."

**Retry indefinido, sem teto de tentativas.** Descartado por deixar evento pendurado para
sempre caso o cliente desapareça definitivamente (PRD-FESC-10). `[TRANSCRICAO 09:15]`:
"Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento
ficar pendurado pra sempre se o cliente sumiu."

**Dead letter como campo de estado na própria outbox, em vez de tabela separada.** Descartada
para manter a leitura da outbox principal limpa e isolar o material de debug e
reprocessamento (PRD-FESC-12). `[TRANSCRICAO 09:18]`: "Eu fazia uma tabela
webhook_dead_letter separada, com a payload, motivo da falha e timestamp. Mais limpa a
leitura da outbox principal, e fica como evidence pra debug e reprocessamento."

## Consequências

### Positivas

- Indisponibilidades planejadas do lado do cliente, incluindo o precedente real de 2 horas,
  são absorvidas sem perder o evento (PRD-RISK-01, `[TRANSCRICAO 09:16]`)
- A outbox principal permanece com leitura limpa para o polling do worker, porque falhas
  definitivas saem dela e vão para uma tabela dedicada (PRD-ESC-07)
- O evento falho não é perdido de forma silenciosa: fica preservado na dead letter com
  motivo e timestamp, reprocessável manualmente quando o cliente se restabelecer
  (PRD-ESC-08a)

### Negativas

- Um evento pode levar cerca de 15 horas para ser declarado definitivamente perdido e movido
  para a dead letter, deixando o cliente sem notificação por esse período no pior caso
  (PRD-DEC-07-TO). O próprio número de tentativas foi contestado dentro da reunião antes de
  ser fechado em 5, com Bruno defendendo um teto menor (`[TRANSCRICAO 09:16]`), o que indica
  que a janela pode precisar de revisão com dados reais de produção
- Enquanto um evento está em ciclo de retry, o cliente fica sem a notificação e sem
  visibilidade de que ela está pendente, a não ser que consulte o histórico de entregas; não
  há aviso proativo ao cliente durante o ciclo de retry, apenas ao esgotar (PRD-FESC-01 já
  registra que notificação de falha repetida por e-mail ficou fora desta entrega)
- A ambiguidade sobre o número exato de tentativas e reagendamentos (PRD-OPEN-05) permanece
  sem solução, o que impede fixar com precisão o critério de aceite de teste do backoff até
  que a questão em aberto da RFC seja resolvida

## Trade-off aceito

Troca-se velocidade em declarar um evento como definitivamente perdido por tolerância a
indisponibilidades longas e planejadas do lado do cliente: aceita-se uma janela de cerca de
15 horas antes de mover o evento para a dead letter, em vez de um teto mais curto que
descartaria eventos durante manutenções programadas legítimas.
