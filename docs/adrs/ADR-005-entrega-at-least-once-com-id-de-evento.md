# ADR-005: Entrega at-least-once com identificador único de evento

Data: 2026-09-01
RFC de origem: docs/RFC.md
Decisores: Larissa (Tech Lead), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança)

## Status

Proposto

## Contexto

A ADR-002 fixa que o worker entrega os eventos da outbox por HTTP POST, e a ADR-003 fixa
que uma falha de entrega reagenda o evento por até 5 tentativas antes de movê-lo para a
dead letter. Falta decidir qual garantia de entrega a plataforma assume perante o cliente, e
o que fazer quando o mesmo evento pode chegar mais de uma vez.

`[TRANSCRICAO 09:24]` fixa a garantia assumida: "a gente vai garantir at-least-once. Pode
acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado"
(PRD-DEC-05, PRD-FR-04f). `[TRANSCRICAO 09:25]` fixa o mecanismo de diferenciação: "A gente
manda um event_id no header, X-Event-Id, com um UUID gerado quando o evento entra na outbox.
É único por evento. Se o cliente recebeu duas vezes, ele dedupica pelo event_id do lado
dele" (PRD-ESC-13). O mesmo minuto tem mais de um falante e registra também a objeção
levantada na reunião, "Isso joga responsabilidade pro cliente", e a resposta a ela, de outro
participante: "Joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim. Garantir
exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com
event_id resolve 99% dos casos."
`[TRANSCRICAO 09:26]` fixa o compromisso de documentação: "Eu posso documentar isso bem
destacado no portal de desenvolvedor pros clientes" (PRD-DEP-03).

A garantia at-least-once em si é decisão estrutural da reunião, não contestada por nenhum
debatedor no debate da RFC-001, e é ancorada aqui direto na transcrição e no PRD. O que o
debate trouxe de novo foi o ponto SEC-03: com a outbox e a dead letter em tabelas separadas
(ADR-003) e identificadores em UUID (PRD-NFR-CONF-06), o caminho mais provável de
implementação do replay administrativo é inserir uma linha nova na outbox com um id novo, o
que faria o cliente, ao deduplicar pelo `X-Event-Id`, tratar a reentrega como um evento
inédito e potencialmente reaplicar uma mudança de status já superada por eventos mais
recentes. No debate, o dev confirmou que não há obstáculo estrutural a preservar o
identificador original: é questão de copiar a coluna do identificador na movimentação entre
tabelas, não de contornar o modelo de dados.

## Decisão

A entrega de eventos é at-least-once, e não exactly-once: o mesmo evento pode chegar mais de
uma vez ao endpoint do cliente (PRD-DEC-05, `[TRANSCRICAO 09:24]`). Cada evento carrega um
identificador único, gerado como UUID no momento em que o evento entra na outbox, enviado no
header `X-Event-Id`, e a deduplicação do lado do cliente é feita com base nesse identificador
(PRD-ESC-13, `[TRANSCRICAO 09:25]`).

Como consequência direta dessa garantia, o replay administrativo de um item da dead letter
preserva o identificador de evento original ao recolocar o evento como pendente na outbox
(PRD-FR-07a), em vez de gerar um identificador novo. Preservar o identificador é o que torna
o replay coerente com a promessa de deduplicação feita ao cliente: um reprocessamento
administrativo de um evento que já havia sido tentado antes precisa continuar sendo
reconhecível pelo cliente como o mesmo evento, não como um evento novo.

## Alternativas Consideradas

**Entrega exactly-once.** Descartada por exigir coordenação entre plataforma e cliente para
garantir que cada evento fosse processado uma única vez, o que aumentaria
significativamente a complexidade da integração, enquanto at-least-once com identificador de
evento é o padrão adotado por plataformas de mercado equivalentes (PRD-DEC-05).
`[TRANSCRICAO 09:25]`: "Garantir exactly-once exigiria coordenação dos dois lados e fica
muito mais complexo. At-least-once com event_id resolve 99% dos casos."

## Consequências

### Positivas

- A plataforma não depende de coordenação de duas fases nem de confirmação de entrega para
  considerar um evento processado, o que mantém o desenho simples e alinhado ao padrão de
  mercado citado na reunião (`[TRANSCRICAO 09:25]`)
- O `X-Event-Id` dá ao cliente um mecanismo direto de deduplicação, sem exigir que ele
  interprete o conteúdo do payload para identificar reentregas (PRD-ESC-13)
- Preservar o identificador original no replay evita que um reprocessamento administrativo
  seja tratado pelo cliente como um evento novo e desconectado da tentativa original
  (consequência do ponto SEC-03 do debate)

### Negativas

- Um cliente que não implemente a deduplicação processa a mesma mudança de status mais de
  uma vez, e não há ação corretiva do lado da plataforma além de documentação e suporte à
  correção da integração (PRD-RISK-05)
- A responsabilidade de garantir idempotência é transferida ao cliente, o que exige que cada
  um dos três clientes implemente corretamente a deduplicação pelo `X-Event-Id` antes de
  confiar na integração (PRD-DEP-05)
- Se duas instâncias do worker chegarem a coexistir, como discutido na ADR-002, duas
  entregas legítimas do mesmo evento ficam indistinguíveis, do lado do cliente, de uma
  reentrega maliciosa capturada e reproduzida por um terceiro, já que ambas chegam com o
  mesmo `X-Event-Id`. Esse agravante foi levantado pela segurança no debate e permanece como
  questão em aberto da RFC, não resolvida por este ADR

## Trade-off aceito

Troca-se a garantia mais forte de exactly-once, que exigiria coordenação cara entre
plataforma e cliente, por uma garantia at-least-once mais simples de operar: aceita-se que o
cliente possa receber o mesmo evento mais de uma vez e que a responsabilidade de deduplicar
fique do lado dele, em troca de um desenho que não depende de confirmação de processamento
nem de estado compartilhado entre os dois lados da integração.
