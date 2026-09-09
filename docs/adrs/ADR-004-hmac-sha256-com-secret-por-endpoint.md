# ADR-004: HMAC-SHA256 com secret por endpoint

Data: 2026-09-01
RFC de origem: docs/RFC.md
Decisores: Larissa (Tech Lead), Sofia (Engenheira de Segurança), Diego (Engenheiro Sênior, time de Plataforma)

## Status

Aceito

## Contexto

A ADR-002 fixa que o worker entrega o evento por HTTP POST ao endpoint cadastrado pelo
cliente. Falta decidir como o cliente verifica que a requisição veio realmente da
plataforma e que o payload não foi adulterado no caminho, já que o fluxo é outbound para uma
infraestrutura que não é da plataforma (PRD-ARQ-12).

`[TRANSCRICAO 09:20]` fixa o padrão e o algoritmo: "Padrão é HMAC. A gente assina o payload
com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo
X-Signature. Cliente verifica do lado dele", com a escolha do algoritmo logo em seguida:
"SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso"
(PRD-ESC-09a, PRD-FR-06b).

`[TRANSCRICAO 09:21]` fixa que a secret é por endpoint, não global, e a rotação com grace
period: "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret
global da nossa plataforma. Senão se vaza uma, vaza tudo (...) a secret tem que ser
rotacionável. Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele
rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os
sistemas dele. Depois disso, a antiga morre" (PRD-ESC-09b, PRD-ESC-10, PRD-FR-02a). A
motivação concreta veio de `[TRANSCRICAO 09:22]`: "A gente já teve cliente que vazou secret
em log de aplicação dele uma vez" (PRD-RISK-02). `[TRANSCRICAO 09:23]` fixa HTTPS
obrigatório para o transporte: "TLS obrigatório. URL do webhook tem que ser https"
(PRD-ESC-11), tratado como validação de schema e não como parte desta decisão.

O debate da RFC-001 (SEC-06, SEC-07) mostrou que o precedente de vazamento citado em
`[TRANSCRICAO 09:22]` não estava refletido na configuração do logger: `redactPaths` em
`src/shared/logger/index.ts` cobre `req.headers.authorization`, `req.headers.cookie`,
`*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, e nenhum desses caminhos
alcança a secret do webhook nem a assinatura calculada (SEC-06). O debate também mostrou que
não estava definido se a secret reaparece nas respostas de leitura do CRUD, o que o proponente
argumentou derivar do próprio propósito de uma secret por endpoint: se ela reaparecesse em
qualquer leitura, o raio de exposição deixaria de ser limitado ao momento da emissão
(PRD-ESC-09b, SEC-07). Aceitei os dois como decisão.

## Decisão

Toda requisição de entrega enviada pelo worker é assinada com HMAC-SHA256 sobre o corpo da
requisição, usando a secret ativa daquele endpoint, transmitida no header `X-Signature`
(PRD-ESC-09a, PRD-FR-06b, `[TRANSCRICAO 09:20]`).

Cada endpoint de webhook tem sua própria secret, gerada pela plataforma no cadastro,
nunca uma secret global compartilhada entre clientes (PRD-ESC-09b, `[TRANSCRICAO 09:21]`). A
secret é rotacionável pela API; ao rotacionar, a secret anterior permanece válida em
paralelo por 24 horas, e depois desse prazo deixa de ser aceita (PRD-ESC-10, PRD-FR-02a).

A secret não é exposta fora do momento de emissão: ela é devolvida ao cliente apenas na
resposta da criação do webhook e na resposta da rotação, e nunca nas respostas de leitura do
CRUD, seja `GET /webhooks` ou a consulta de um único registro. A configuração de redação do
logger em `src/shared/logger/index.ts` passa a cobrir também a secret do webhook e a
assinatura calculada, estendendo `redactPaths`, que hoje cobre apenas
`req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`
e `*.accessToken`, sem alcançar nenhum dado desta feature.

Este ADR não decide qual secret assina durante as 24 horas de grace period da rotação, nem
o que acontece com uma secret sabidamente comprometida, que sob a regra descrita aqui
continua válida por até 24 horas depois de o cliente pedir a troca. Essa lacuna está
registrada como PRD-OPEN-02 e PRD-ARQ-15, ampliada no debate pelo ponto SEC-02 da ata,
classificado `EM ABERTO`, e segue como questão em aberto da RFC.

Este ADR também não decide se o material assinado deveria incluir o timestamp de envio além
do corpo da requisição. No debate, a segurança sustentou (ponto SEC-01 da ata) que assinar
apenas o corpo torna uma entrega capturada por um atacante reproduzível indefinidamente,
porque nada na assinatura amarra a requisição a um instante de tempo. O autor da RFC recusou
mudar a decisão porque o PRD fixa o HMAC sobre o corpo da requisição (PRD-ESC-09a) e a regra
do fluxo de RFC não permite sobrepor o PRD. O ponto ficou `DIVERGENTE` na ata, com as duas
posições registradas, e este ADR não resolve a divergência nem toma partido: assina-se o
corpo, como fixado pela fonte, com a divergência preservada como questão em aberto da RFC.

## Alternativas Consideradas

**Secret global da plataforma, compartilhada entre todos os endpoints de webhook.**
Descartada porque o vazamento de uma única secret comprometeria a autenticidade de todos os
clientes ao mesmo tempo, em vez de limitar o raio de impacto a um único endpoint
(PRD-ESC-09b). `[TRANSCRICAO 09:21]`: "cada endpoint de webhook do cliente tem que ter uma
secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo."

## Consequências

### Positivas

- O cliente consegue verificar que a requisição veio da plataforma e que o corpo não foi
  adulterado, recalculando o HMAC-SHA256 do lado dele (PRD-ESC-09a, `[TRANSCRICAO 09:20]`)
- O raio de impacto de uma secret vazada fica limitado a um único endpoint, e não a todos os
  clientes integrados, respondendo diretamente ao precedente relatado em
  `[TRANSCRICAO 09:22]`
- A secret nunca aparece fora do momento de emissão: não é devolvida em leitura, e passa a
  ser redigida no log junto com a assinatura, fechando o caminho de vazamento que motivou a
  decisão

### Negativas

- Durante a janela de rotação, mais de uma secret fica válida ao mesmo tempo para o mesmo
  endpoint, o que aumenta a superfície de verificação do lado do cliente e deixa em aberto,
  sem decisão desta ADR, qual delas o worker usa para assinar nesse intervalo (PRD-OPEN-02)
- O esquema de assinatura se torna um contrato difícil de alterar depois que os três clientes
  estiverem integrados: mudar o material assinado, por exemplo para incluir o timestamp,
  exigiria coordenar uma migração com cada cliente já em produção, o que é exatamente o
  motivo pelo qual a divergência sobre incluir o timestamp não foi resolvida agora
- Uma secret sabidamente comprometida continua válida por até 24 horas depois de o cliente
  pedir a rotação, porque a regra de grace period não distingue rotação de rotina de rotação
  por incidente (PRD-RISK-02, ponto SEC-02 da ata)

## Trade-off aceito

Troca-se simplicidade de uma única secret por cliente por contenção de dano: aceita-se a
complexidade operacional de gerenciar mais de uma secret válida por endpoint durante a
rotação, e um contrato de assinatura que fica caro de mudar depois que os clientes
integrarem, em troca de limitar o impacto de um vazamento a um único endpoint em vez de
comprometer todos os clientes de uma vez.
