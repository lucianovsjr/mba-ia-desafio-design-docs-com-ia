# Ata do debate da RFC-001

Documento em debate: docs/RFC.md
Participantes: rfc-arquiteto, rfc-dev, rfc-seguranca; conduzido pelo Tech Lead
Rodadas: 2
Data: 2026-09-01

Encerrado na segunda rodada porque nenhum ponto novo com evidência apareceu: as
posições restantes ou recuaram, ou foram sustentadas sem fonte nova.

## Pontos consolidados

| ID | Ponto | Evidência | Resposta do autor | Classificação | Destino |
| --- | --- | --- | --- | --- | --- |
| ARQ-01 | Worker embutido na API foi descartado na reunião mas não consta em "Alternativas consideradas" | PRD-DEC-02, `[TRANSCRICAO 09:11]` | Aceito sem ressalva | ACORDADO | RFC Alternativas consideradas |
| ARQ-02 | Nada no desenho impede duas instâncias do worker: a unicidade é disciplina operacional, não mecanismo, e um rolling restart já produz duas | PRD-DEC-06, PRD-DEC-02; mecanismo proposto sem âncora | Diagnóstico aceito, mecanismo rejeitado por falta de origem. O debatedor recuou da solução e manteve o diagnóstico | ACORDADO | RFC Impacto e riscos, e Questões em aberto para o mecanismo |
| ARQ-03 | A RFC não diz se o lote é entregue sequencial ou concorrentemente, o que afeta o limiar de 10s | PRD-ESC-04a, PRD-FR-04d, PRD-DEC-06, `[TRANSCRICAO 09:12]` | Aceito na forma refinada na rodada 2, sequencial por `order_id` e concorrente entre pedidos distintos, derivável de PRD-DEC-06 | ACORDADO | ADR-TBD-02 |
| ARQ-04 | Crescimento da outbox sem rotina de purga não é discutido, e o polling compete com tabela cada vez maior | PRD-FESC-03 | Aceito sem ressalva | ACORDADO | RFC Impacto e riscos |
| ARQ-05 | A proposta central é apresentada como fechada, mas a saúde da transação fica inteiramente em aberto, sem posição mínima | Seção "Questões em aberto" da própria RFC | Aceito. Tomo posição de manter a publicação na transação e aceitar a contenção, como aceite qualitativo sem número, conforme enquadramento do dev | ACORDADO | ADR-TBD-01 |
| ARQ-06 | Monitoramento mínimo do worker seria decisão de arquitetura, não questão em aberto | PRD-NFR-DISP-03 | Rejeitado por criar requisito sem origem. O debatedor recuou reconhecendo a falta de âncora | ACORDADO | RFC Questões em aberto, marcado hipótese |
| DEV-01 | A RFC afirma que `src/app.ts` não é tocado, mas `buildControllers` monta `OrderService` com DI manual | `src/app.ts`, `src/modules/orders/order.service.ts` | Aceito. Verificado no código pelo autor | ACORDADO | RFC Impacto e riscos |
| DEV-02 | Não está dito em que ponto da transação a publicação entra, o que muda o status capturado no snapshot | `src/modules/orders/order.service.ts` | Aceito, no nível de altura da RFC: depois do update de status, antes do fim da transação | ACORDADO | ADR-TBD-01 |
| DEV-03 | O padrão real do projeto é herdar das subclasses HTTP intermediárias, não de `AppError` direto | `src/shared/errors/http-errors.ts` | Aceito. Verificado no código pelo autor | ACORDADO | RFC Proposta técnica, parágrafo de reuso |
| DEV-04 | A trilha de auditoria do replay é peça nova, não reuso: não há model nem util de auditoria no projeto | `prisma/schema.prisma`, ausência confirmada em `src/` | Aceito, e elevado a decisão pelo argumento da segurança: o replay é a única operação com autorização reforçada. Retenção cortada por falta de fonte, e o proponente retirou essa parte | ACORDADO | ADR-TBD-05 |
| DEV-05 | O worker importando o singleton do logger sai marcado com o serviço da API | `src/shared/logger/index.ts` | Aceito sem ressalva | ACORDADO | RFC Proposta técnica, parágrafo de reuso |
| DEV-06 | Não há infraestrutura de teste capaz de exercitar o worker nem o retry; o padrão é supertest contra a app real | `tests/setup.ts`, `package.json` | Aceito. Verificado no código pelo autor. A fronteira loop/lote tem âncora em PRD-TEST-07 e vira decisão; a escolha do mecanismo de teste fica aberta | ACORDADO | ADR-TBD-02, e Questões em aberto para o mecanismo |
| SEC-01 | O HMAC cobre só o corpo, com o timestamp fora do material assinado, então uma entrega capturada é reproduzível para sempre | PRD-ESC-09a, PRD-FR-06b, `[TRANSCRICAO 09:22]`, `[TRANSCRICAO 09:44]` | Concordo tecnicamente, mas o PRD fixou o material assinado e a RFC não sobrepõe o PRD. O debatedor sustentou o risco e aceitou o destino | DIVERGENTE | RFC Questões em aberto, com as duas posições |
| SEC-02 | Secret comprometida segue válida por 24h porque o grace period não distingue rotina de incidente | PRD-ESC-10, PRD-RISK-02 | Aceito parcialmente: não crio caminho de revogação como requisito, mas amplio a questão em aberto que já existia | EM ABERTO | RFC Questões em aberto, ampliada |
| SEC-03 | Não está definido se o replay preserva o `X-Event-Id`, e gerar id novo quebra a deduplicação do cliente | PRD-ESC-13, PRD-FR-07a | Aceito como decisão, por ser consequência de PRD-ESC-13 e não requisito novo | ACORDADO | ADR-TBD-03 |
| SEC-04 | A validação cobre só o esquema HTTPS, não o destino, o que expõe o worker a SSRF para rede privada e metadados de nuvem | PRD-ESC-11, `[TRANSCRICAO 09:23]`; cenário marcado HIPÓTESE pelo próprio proponente | A mitigação não tem âncora e não entra na proposta técnica. A segurança recuou para questão em aberto; a arquitetura sustentou como bloqueador pela composição com PRD-DEC-08 | DIVERGENTE | RFC Questões em aberto, marcado hipótese, com as duas posições |
| SEC-05 | O CRUD aberto a qualquer papel autenticado não aparece em lugar nenhum da RFC, nem como decisão nem como aberto | PRD-DEC-08, `[TRANSCRICAO 09:37]` | Aceito sem ressalva. Omitir uma decisão não é o mesmo que aceitá-la | ACORDADO | RFC Proposta técnica, e Questões em aberto para a provisoriedade |
| SEC-06 | O risco de vazamento de secret no log fica só diagnosticado, sem proposta: o redact do Pino não cobre secret nem assinatura | `src/shared/logger/index.ts`, `[TRANSCRICAO 09:22]` | Aceito como decisão | ACORDADO | ADR-TBD-04 |
| SEC-07 | Não está definido se a secret reaparece nas respostas de leitura do CRUD | PRD-ESC-09b, PRD-FR-01a, PRD-FR-01b | Aceito como decisão, por derivar do propósito da secret por endpoint | ACORDADO | ADR-TBD-04 |

Totais: 16 `ACORDADO`, 2 `DIVERGENTE`, 1 `EM ABERTO`, 0 `FORA DE ESCOPO`.

## Decisões a registrar

| Provisório | Decisão | Pontos de origem |
| --- | --- | --- |
| ADR-TBD-01 | Manter a publicação do evento dentro da transação do `changeStatus`, posicionada depois do update de status, aceitando de forma declarada e não quantificada a contenção adicional | ARQ-05, DEV-02 |
| ADR-TBD-02 | Processar a outbox por uma função de lote isolável do loop de polling, entregando sequencialmente por `order_id` e concorrentemente entre pedidos distintos | ARQ-03, DEV-06 |
| ADR-TBD-03 | O replay de item da dead letter preserva o identificador de evento original, para não quebrar a deduplicação do cliente | SEC-03 |
| ADR-TBD-04 | Não expor a secret do webhook fora do momento de emissão: redação no log e ausência nas respostas de leitura | SEC-06, SEC-07 |
| ADR-TBD-05 | Trilha de auditoria do replay como peça própria do módulo de webhooks, consultável, e não linha solta no log compartilhado, sem exigência de retenção | DEV-04 |

## Registro das rodadas

### Rodada 1

O arquiteto atacou por dois lados. O primeiro foi de forma: a alternativa do worker embutido
na API sumiu da seção de alternativas (ARQ-01), e o crescimento da outbox sem purga não
aparece nos riscos (ARQ-04). Aceitei ambos sem discussão. O segundo foi de substância:
sustentou que a proposta central é apresentada como fechada enquanto a saúde da transação
fica inteiramente em aberto (ARQ-05), e que a garantia de worker único não existe como
mecanismo (ARQ-02). Aceitei ARQ-05 e me comprometi a tomar posição. Em ARQ-02 aceitei o
diagnóstico e rejeitei o mecanismo proposto, lock otimista, por não vir de nenhuma das quatro
fontes. Rejeitei ARQ-06 pelo mesmo motivo: transformar monitoramento do worker em requisito
do v1 seria inventar escopo sobre uma ausência que o PRD já registra como ausência.

O dev trouxe o material mais verificável do debate, e conferi cada alegação no código antes
de responder. `buildControllers` em `src/app.ts` monta `OrderService` por injeção manual, o
que derruba minha afirmação de que o arquivo não seria tocado (DEV-01). O padrão de erro do
projeto passa pelas subclasses HTTP intermediárias, não por `AppError` direto (DEV-03). Não
existe auditoria reaproveitável no projeto (DEV-04). O logger é singleton com serviço fixo
(DEV-05). E `tests/setup.ts` limpa uma lista fixa de tabelas com supertest contra a app real,
sem nock, msw ou fake timers (DEV-06). Aceitei os cinco.

A segurança levantou sete pontos, quatro deles como bloqueador. Rejeitei SEC-01 como ajuste
de texto por uma razão de fluxo e não de mérito: o PRD fixou o HMAC sobre o corpo, e a regra
diz que a RFC não sobrepõe o PRD. Recusei transformar SEC-04 em requisito, porque o próprio
proponente marcou o cenário de SSRF como hipótese não discutida na reunião, e hipótese não
vira requisito. Aceitei SEC-03, SEC-05, SEC-06 e SEC-07, e ampliei a questão em aberto do
grace period para cobrir SEC-02. SEC-05 foi o melhor ponto do conjunto: a RFC tinha
simplesmente omitido do texto que o CRUD é aberto a qualquer papel autenticado.

### Rodada 2

O arquiteto recuou em ARQ-02 e ARQ-06, reconhecendo em ambos que suas propostas eram conselho
sem âncora. Em ARQ-03 não recuou: refinou. Mostrou que fechar em "sequencial" resolveria a
ordenação mas faria um cliente lento atrasar pedidos de outros clientes sem ganho nenhum, já
que a garantia acordada é por `order_id` e não global, e propôs sequencial por `order_id` com
concorrência entre pedidos distintos. Aceitei: é melhor que a minha formulação e continua
derivável de PRD-DEC-06. Em SEC-04 escolheu sustentar mesmo depois de eu apontar que o
proponente do ponto já havia encaminhado, argumentando que a composição de PRD-DEC-08 com
entrega outbound automática torna o risco ativo e não teórico. Registrado como divergência,
com a severidade que ele atribui.

O dev respondeu à única pergunta que sobrou de forma exemplar: perguntei de que lado da regra
de âncora caía a exigência de uma interface de transporte substituível, e ele respondeu que
cai do lado sem âncora, demonstrando que PRD-TEST-07 é satisfeito sem abstração nenhuma,
apontando a função de lote para um servidor HTTP local, no mesmo espírito do padrão de teste
que o projeto já usa. Fechamos a fronteira loop/lote como decisão e a interface de transporte
desceu para o FDD como recomendação registrada, sem virar requisito.

A segurança confirmou que a decisão de auditoria sem retenção resolve o ponto dela, e retirou
a exigência de retenção reconhecendo que era overreach sem fonte. Também trouxe o único
argumento cruzado que mudou a formulação de uma questão em aberto: dois workers coexistindo
produzem duas entregas autenticadas com timestamps diferentes, indistinguíveis do lado do
cliente de uma reentrega capturada, o que liga ARQ-02 a SEC-01 e dá peso à posição dela na
divergência.

## Registrado para o FDD

Recomendação do rfc-dev, sem âncora nas quatro fontes e portanto sem status de requisito: a
entrega HTTP do worker atrás de uma interface substituível, para não travar a escolha futura
entre servidor de teste local e fake timers.
