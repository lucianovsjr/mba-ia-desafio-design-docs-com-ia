# Objetivo

Este documento é a fonte única do fluxo que produz a RFC do Sistema de Webhooks de
Notificação de Pedidos do OMS e as ADRs derivadas dela. As skills `rfc-draft`,
`rfc-debate`, `rfc-adr` e `rfc-close` e os agentes `rfc-arquiteto`, `rfc-dev`,
`rfc-seguranca` e `rfc-revisor` seguem as regras daqui. Nenhum deles reproduz este
conteúdo nem improvisa uma versão resumida.

O PRD da feature já existe em `docs/PRD.md` e já foi rastreado em `docs/TRACKER.md`.
A RFC não repete o PRD: ela responde "como pretendemos resolver, e o que ainda está
em aberto".

## Fontes da verdade

Quatro fontes, nesta ordem de precedência:

1. `docs/PRD.md`, escopo e requisitos já acordados no nível de produto
2. `docs/TRACKER.md`, o vocabulário de IDs e a origem já verificada de cada item
3. `TRANSCRICAO.md`, a reunião completa
4. `src/`, `prisma/` e `tests/`, o código existente

Se a RFC contradisser o PRD, o PRD vence e o ponto vira divergência a ser tratada,
não um fato novo.

## Regra de âncora

Toda afirmação técnica registrada em qualquer artefato deste fluxo carrega origem.
Só existem três formas de fechar uma afirmação:

- **ID do tracker**, quando o item já está rastreado. Exemplo: `PRD-ESC-06`.
  Preferível sempre que existir, porque a origem já foi verificada de forma
  independente.
- **Citação direta**, quando o item não tem linha no tracker: `[TRANSCRICAO 09:17]`
  ou `src/modules/orders/order.service.ts`. Nesse caso o ponto é marcado como
  `NOVO`, e o fechamento do fluxo garante que ele entre no tracker.
- **HIPÓTESE**, quando não há origem nenhuma. Precisa estar escrito com essa
  palavra, no próprio ponto. Uma hipótese pode aparecer como questão em aberto na
  RFC. Uma hipótese nunca vira ADR e nunca vira requisito.

Citação de timestamp não carrega o nome de quem falou e vários timestamps têm mais
de um falante. Quem escreve nome de falante é o `tracker-rastreabilidade`, depois
de ler a linha inteira da transcrição. Nos artefatos deste fluxo, cite o timestamp
sem inventar atribuição.

É proibido criar requisito, número, limite, endpoint ou integração que não venha de
uma das quatro fontes.

## Taxonomia dos pontos

Todo ponto levantado no debate termina classificado em um destes quatro estados:

| Estado | Significado | Destino |
| --- | --- | --- |
| `ACORDADO` | o autor aceitou o ponto, ou o ponto confirmou a proposta com evidência nova | vira ADR em `rfc-adr`, ou ajuste direto no texto da RFC |
| `DIVERGENTE` | debatedor e autor não convergiram dentro das rodadas | vai para "Questões em aberto" da RFC, nomeando as duas posições |
| `EM ABERTO` | a reunião não decidiu, ou adiou explicitamente | vai para "Questões em aberto" da RFC |
| `FORA DE ESCOPO` | descartado ou adiado na reunião, ou fora da altura da RFC | não entra na RFC, fica registrado só na ata |

Um ponto sem classificação impede o encerramento da rodada.

## Formato da análise de cada debatedor

Cada agente escreve um único arquivo em
`docs/debates/RFC-001/analise-<papel>.md`, com este formato:

```
# Análise da RFC: <papel>

Agente: rfc-<papel>
Documento analisado: docs/RFC.md (versão Rascunho)
Data: <AAAA-MM-DD>

## Pontos

| ID | Ponto | Evidência | Severidade | Proposta |
| --- | --- | --- | --- | --- |

## Leitura geral

<3 a 6 linhas: o que a proposta acerta e onde ela é mais frágil>
```

Regras da tabela:

- `ID` no prefixo do papel: `ARQ-01`, `DEV-01`, `SEC-01`
- `Ponto` em uma frase, afirmativa, dizendo qual é o problema
- `Evidência` é um ID do tracker, uma citação direta ou a palavra `HIPÓTESE`
- `Severidade` é `bloqueador`, `ajuste` ou `nota`
- `Proposta` é a correção concreta na RFC, não um conselho genérico
- Entre 4 e 10 pontos. Menos que isso é análise superficial, mais que isso é ruído
- Pelo menos um ponto de severidade `bloqueador` ou `ajuste` que **objete** à
  proposta escolhida. Análise que só concorda não cumpre o papel
- Não escreva elogio. O autor da RFC não precisa de validação, precisa de defeito

Na fase de análise, nenhum agente lê a análise de outro agente. Ler o arquivo de um
colega antes de entregar o seu contamina o resultado: os três passam a repetir o
primeiro que escreveu.

## Formato da ata do debate

`docs/debates/RFC-001/ata.md`:

```
# Ata do debate da RFC-001

Documento em debate: docs/RFC.md
Participantes: rfc-arquiteto, rfc-dev, rfc-seguranca; conduzido pelo Tech Lead
Rodadas: <n>
Data: <AAAA-MM-DD>

## Pontos consolidados

| ID | Ponto | Evidência | Resposta do autor | Classificação | Destino |
| --- | --- | --- | --- | --- | --- |

## Decisões a registrar

| Provisório | Decisão | Pontos de origem |
| --- | --- | --- |
| ADR-TBD-01 | <decisão em uma frase> | ARQ-03, DEV-01 |

## Registro das rodadas

### Rodada 1
<o que cada agente sustentou e o que o autor respondeu, em prosa curta>
```

A coluna `Destino` diz onde o ponto foi parar: `RFC seção X`, `ADR-TBD-NN`,
`Questões em aberto` ou `nenhum`.

## Esqueleto da RFC

Seções obrigatórias, nesta ordem:

```
# RFC: <título da proposta>

Status: Rascunho | Em revisão | Aceita
Autor: <nome> (Tech Lead)
Data: <AAAA-MM-DD>
Revisores: <participantes da reunião>
PRD relacionado: docs/PRD.md

## TL;DR
## Contexto e problema
## Proposta técnica
## Alternativas consideradas
## Questões em aberto
## Impacto e riscos
## Decisões relacionadas
```

Regras de conteúdo:

- `Alternativas consideradas`: no mínimo duas alternativas reais discutidas e
  descartadas na reunião, cada uma com o trade-off que motivou o descarte
- `Questões em aberto`: no mínimo dois pontos levantados na reunião e não decididos
  ou adiados
- `Decisões relacionadas`: links para os ADRs. Enquanto os ADRs não existem, a
  seção lista os provisórios `ADR-TBD-NN` vindos da ata
- `Impacto e riscos`: impacto no código existente e nos clientes já integrados

## Regra de altura

A RFC opera em nível de arquitetura. O detalhamento de implementação é do FDD, que
será produzido depois deste fluxo. É proibido na RFC:

- payload de exemplo de request ou response
- matriz de erros ou lista de códigos `WEBHOOK_*`
- assinatura de função, DDL de tabela, nome de coluna, schema de validação
- lista de endpoints com status codes

Citar que existe uma tabela de outbox, um worker e um endpoint de administração é
nível de RFC. Descrever as colunas dessa tabela não é.

Alvo de tamanho: 2 a 4 páginas. Na prática, entre 120 e 250 linhas de Markdown.

## Esqueleto de ADR

Um arquivo por decisão, em `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`:

```
# ADR-NNN: <título da decisão>

Data: <AAAA-MM-DD>
RFC de origem: docs/RFC.md
Decisores: <participantes da reunião que sustentaram a decisão>

## Status

Proposto

## Contexto
## Decisão
## Alternativas Consideradas
## Consequências

### Positivas
### Negativas

## Trade-off aceito
```

Regras:

- `Alternativas Consideradas` com pelo menos uma alternativa real, discutida na
  reunião ou tecnicamente plausível, e o motivo do descarte
- `Consequências` sempre com os dois lados. ADR sem consequência negativa é ADR
  que não pensou no problema
- Pelo menos um ADR do conjunto referencia arquivos, módulos ou classes reais do
  código base, com caminho verificado
- `Status` é seção de primeiro nível, não linha de metadado. Valores: `Proposto`,
  `Aceito`, `Substituído por ADR-NNN` ou `Descontinuado`. Um ADR só chega a
  `Aceito` no fechamento da RFC

## Ciclo de vida de um ADR

Um ADR tem dois momentos formais: a abertura, que registra a decisão, e o aceite,
que a torna vigente. Entre os dois existe uma janela de revisão. Nenhum ADR nasce
`Aceito`.

### Abertura

1. **Gatilho.** A decisão está classificada como `ACORDADO` na ata do debate e
   aparece na tabela "Decisões a registrar". Decisão que não passou pelo debate não
   abre ADR. Ponto `DIVERGENTE` ou `EM ABERTO` vira questão em aberto na RFC, não
   ADR.
2. **Deduplicação.** Uma ADR por decisão, não uma por ponto de debate. Dois pontos
   de eixos diferentes que descrevem a mesma escolha viram um único arquivo.
3. **Numeração.** O número é alocado pela skill `rfc-adr`, em sequência, olhando o
   que já existe em `docs/adrs/`. Agentes nunca escolhem o próprio número: dois
   agentes em paralelo colidem no mesmo `ADR-003`.
4. **Escrita.** O arquivo nasce com `Status: Proposto`, com todas as seções do
   esqueleto preenchidas e toda afirmação ancorada pela regra de âncora.
5. **Rastreabilidade.** O `tracker-rastreabilidade` acrescenta as linhas do ADR ao
   `docs/TRACKER.md`. Um ADR fora do tracker não avança para o aceite.

### Aceite

O aceite acontece no `rfc-close`, para o conjunto inteiro de uma vez, nunca ADR a
ADR isoladamente. São cinco condições, e todas precisam valer:

1. O `rfc-revisor` rodou sobre `docs/adrs/` e nenhum achado `bloqueador` está aberto.
2. Nenhum item do ADR está marcado como `INVENTADO` no relatório do revisor.
3. O link é de mão dupla: o ADR aponta para `docs/RFC.md` e a seção "Decisões
   relacionadas" da RFC aponta para o arquivo do ADR, sem nenhum `ADR-TBD` restante.
4. Todo caminho de código citado no ADR existe no repositório.
5. As linhas do ADR já estão em `docs/TRACKER.md`.

Cumpridas as cinco, a skill troca `Proposto` por `Aceito` em cada arquivo e só então
marca a RFC como `Aceita`. RFC aceita com ADR em `Proposto` é estado inválido.

### Depois do aceite

Um ADR aceito não é editado para mudar a decisão. Ele é substituído: abre-se um novo
ADR, e o antigo passa a `Substituído por ADR-NNN`, mantendo o texto original como
registro histórico. Correção de erro factual, caminho de arquivo ou redação é edição
normal e não muda o status. Decisão que deixou de valer sem sucessora vira
`Descontinuado`.

## Estilo

- Português do Brasil, direto, sem adjetivo de venda
- Sem travessão longo. Use vírgula, dois pontos ou parênteses
- Sem placeholder entre colchetes no documento final
- Sem seção de cronograma, prazo, marco de entrega ou próximos passos
- Sem repetir no artefato o que já está no PRD. Referencie o ID e siga

## Estados do fluxo

| Etapa | Skill | Status da RFC ao final |
| --- | --- | --- |
| Rascunho pelo Tech Lead | `rfc-draft` | `Rascunho` |
| Análises e debate | `rfc-debate` | `Em revisão` |
| Registro das decisões | `rfc-adr` | `Em revisão` |
| Validação e fechamento | `rfc-close` | `Aceita` |

Toda skill deste fluxo é idempotente: rodar de novo corrige o que existe, não
duplica arquivo, linha de tabela nem ADR.
