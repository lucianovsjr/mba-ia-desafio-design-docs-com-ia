# Da Reunião ao Documento: processo de produção

Documentação do processo usado para transformar a transcrição de uma reunião técnica em um pacote de design docs. O enunciado original do desafio está preservado em [`DESAFIO.md`](DESAFIO.md).

---

## Sobre o desafio

Uma empresa opera um Order Management System em produção e decidiu construir um sistema de webhooks para notificar clientes B2B quando o status de um pedido muda. A decisão técnica já foi tomada numa reunião de 55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança, e nada sobrou dela além da transcrição literal da call. A tarefa é produzir, a partir dessa transcrição e do código da aplicação existente, a documentação técnica que permita ao time começar a implementar: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

A restrição que define o desafio não é a quantidade de documentos, é a proibição de inventar. Toda informação registrada precisa ser rastreável a um timestamp da transcrição ou a um arquivo do código. Isso muda a natureza do trabalho: o difícil não é gerar texto, é impedir que o texto gerado escorregue para o plausível. A reunião contém decisões fechadas, mas também alternativas recusadas, pontos adiados para fases futuras e temas que ninguém tocou, e essas quatro categorias precisam terminar em lugares diferentes do documento. Separar o que foi descartado do que foi adiado, e ambos do que simplesmente não foi discutido, é a parte que exige critério.

---

## Ferramentas de IA utilizadas

- **Claude Code (Opus 5)** como ambiente principal. Sessão interativa no terminal, com acesso de leitura e escrita ao repositório, usada tanto para produzir os documentos quanto para orquestrar os subagentes.
- **Subagentes do Claude Code** para separar papéis que não podem ser exercidos pelo mesmo contexto. Sete foram criados neste projeto, três no fluxo de PRD e quatro no fluxo de RFC, descritos na seção de workflow.
- **Skills customizadas do Claude Code** (`.claude/skills/`) para empacotar cada fluxo como comando invocável: uma para a entrevista de PRD, com modo manual e automatizado, e quatro para as etapas da RFC.

Nenhuma outra ferramenta de IA foi usada.

---

## Workflow adotado

O trabalho foi organizado em torno de uma ideia: **nenhum agente audita o próprio trabalho.** Cada papel que envolve julgamento sobre a qualidade do que foi produzido roda em um contexto separado, que não participou da produção.

### Ordem de produção

1. **Prompt de entrevista de PRD**, escrito antes de qualquer documento. É o artefato de processo mais importante da entrega, e foi ele que definiu a estrutura de tudo que veio depois.
2. **Estrutura de agentes**, montada antes de rodar a entrevista.
3. **PRD**, produzido por entrevista automatizada em 12 etapas.
4. **Auditoria do PRD** e aplicação das correções.
5. **Tracker de rastreabilidade** do PRD.
6. **Fluxo da RFC**, em quatro etapas encadeadas: rascunho pelo Tech Lead, análise independente e debate com três especialistas, registro das decisões como ADRs, fechamento com auditoria e tracker.
7. FDD, derivado da RFC e das ADRs já fechadas, seguindo o mesmo ciclo.

### Os quatro papéis do fluxo de PRD

| Papel | O que faz | Por que é separado |
| --- | --- | --- |
| **Entrevistador** | Conduz as 12 etapas e redige o documento | É quem produz |
| **Entrevistado** (`po-entrevistado`) | Responde no papel dos cinco participantes da reunião, ancorado apenas na transcrição e no código | Responder e perguntar no mesmo contexto produz um documento limpo demais, que não revela onde o prompt falha |
| **Revisor** (`revisor-prd`) | Audita o documento contra as checagens de consistência e classifica cada item como rastreado, derivado ou inventado | Quem redigiu revisa a lembrança do que quis dizer, não a fonte |
| **Tracker** (`tracker-rastreabilidade`) | Escritor único do tracker. Vai à transcrição por conta própria para determinar a origem de cada item | Se receber a atribuição pronta de quem redigiu, deixa de ser uma segunda leitura e vira uma cópia |

### Como o loop de entrevista roda

A sessão principal conduz a entrevista, envia cada bloco de etapas ao entrevistado, recebe as respostas, resume para revisão humana e segue. Ao final, chama o revisor e o tracker, e cruza os dois relatórios. Quatro regras de execução saíram de erros observados na prática:

- **Um único subagente, do começo ao fim.** A primeira etapa abre o entrevistado, e todas as seguintes continuam a mesma conversa. Abrir um subagente novo a cada etapa perde o histórico, e o entrevistado passa a se contradizer entre etapas.
- **Perguntas agrupadas por etapa, não uma a uma.** São 12 mensagens em vez de cerca de 40 idas e voltas. O princípio de uma pergunta por vez existe para não sobrecarregar uma pessoa, e não se aplica a um subagente que lê a fonte inteira antes de responder.
- **O cabeçalho do documento é perguntado ao humano.** Responsável e versão são metadados do documento, não da feature: não existem na transcrição nem no código. Perguntados ao entrevistado, voltariam como um nome qualquer da reunião. Nesta execução o responsável e a versão foram confirmados diretamente com quem rodou o comando, antes de a entrevista começar.
- **O resumo de cada etapa é escrito para o humano, e a entrevista segue sem esperar confirmação.** O prompt base pede confirmação ao final de cada etapa, mas a confirmação do próprio entrevistado não tem valor de revisão. Quem revisa é quem lê o resumo no terminal.

### O fluxo da RFC

O PRD nasceu de uma entrevista, com um entrevistado respondendo. A RFC não pode nascer assim: ela é uma proposta técnica submetida a revisão, e o que dá valor a ela é o ataque que ela sobrevive. Por isso o fluxo da RFC troca a entrevista por um debate, em quatro etapas, cada uma com uma skill própria.

| Etapa | Comando | O que produz | Status da RFC | Status dos ADRs |
| --- | --- | --- | --- | --- |
| Rascunho | `/rfc-draft` | `docs/RFC.md` escrita pelo Tech Lead a partir do PRD, do tracker, da transcrição e do código | `Rascunho` | não existem |
| Análise e debate | `/rfc-debate` | três análises independentes e a ata em `docs/debates/RFC-001/` | `Em revisão` | provisórios `ADR-TBD-NN` na ata |
| Registro das decisões | `/rfc-adr` | as ADRs em `docs/adrs/`, uma por decisão acordada | `Em revisão` | `Proposto` |
| Fechamento | `/rfc-close` | links bidirecionais, auditoria, linhas no tracker | `Aceita` | `Aceito` |

Os três debatedores têm eixos de ataque que não se sobrepõem, e critérios de aceite próprios para a análise que entregam:

| Agente | Ataca | Precisa entregar |
| --- | --- | --- |
| `rfc-arquiteto` | alternativas descartadas, acoplamento, custo operacional, escala | ao menos uma alternativa da reunião com o trade-off explícito |
| `rfc-dev` | viabilidade no código real, transação do `changeStatus`, reuso dos padrões do projeto, testabilidade | toda objeção com caminho de arquivo real, verificado |
| `rfc-seguranca` | assinatura, secret, replay, vazamento em log, destino do webhook | ao menos um ponto sobre secret e um sobre idempotência da entrega |
| `rfc-revisor` | audita a RFC e as ADRs prontas contra o esqueleto, a rastreabilidade e os critérios de aceite | veredito PRONTO ou NÃO PRONTO, com achados acionáveis |

Seis decisões de desenho sustentam esse fluxo:

- **As três análises são escritas às cegas.** Na primeira fase nenhum agente lê a análise dos outros. Três instâncias do mesmo modelo debatendo ao vivo convergem rápido demais, e a convergência não é acordo, é ancoragem no primeiro que falou. Escrever isolado e só depois confrontar é o que preserva três leituras de verdade.
- **Todo ponto termina classificado.** `ACORDADO` vira ADR ou ajuste no texto, `DIVERGENTE` e `EM ABERTO` vão para as questões em aberto da RFC, `FORA DE ESCOPO` fica só na ata. O debate tem no máximo três rodadas, e divergência que sobrevive às três é questão em aberto, não motivo para uma quarta. O documento exige no mínimo duas questões em aberto, e é o debate que as produz, em vez de o autor ter que inventá-las.
- **O Tech Lead é a sessão principal, não um subagente.** Ele é o único papel presente nas quatro etapas, e é quem aciona os outros agentes, edita a RFC e chama o tracker. Um subagente não orquestra outros subagentes, então transformá-lo em agente partiria o contexto do debate em dois sem ganho nenhum.
- **Segurança registra risco, não veta.** Risco que o Tech Lead assume de forma explícita vira `DIVERGENTE` na ata, com o risco nomeado, e sobe para as questões em aberto. Dar poder de veto a um agente transformaria a auditoria em bloqueio, e o autor deixaria de ser o dono da proposta.
- **A numeração das ADRs é alocada pela skill, nunca pelos agentes.** Dois agentes escolhendo em paralelo colidem no mesmo `ADR-003`. As ADRs também são escritas em sequência, e não em paralelo, porque só assim o autor enxerga o que já escreveu e evita duas ADRs dizendo a mesma coisa com palavras diferentes.
- **Nenhum ADR nasce aceito, e o aceite é do conjunto inteiro.** A abertura escreve o arquivo em `Proposto`; só `/rfc-close` promove para `Aceito`, e para todos de uma vez. As seis decisões da reunião são interdependentes, e aceitar uma isolada congelaria uma escolha cuja premissa o ADR seguinte ainda pode contradizer.

A ata e as análises não entram no tracker. São artefatos de processo, não documentos de design: incluí-las infla a cobertura com linhas que não são entregáveis. O tracker só é chamado quando o documento para de mudar, uma vez por ADR em `/rfc-adr` e uma vez pela RFC em `/rfc-close`.

### A dependência circular entre RFC e ADRs

A RFC precisa linkar as ADRs, e as ADRs nascem do debate da RFC. Fechar a RFC antes das ADRs deixaria os links vazios; escrever as ADRs antes do debate anularia o debate. A saída foi um estado intermediário: o debate encerra com a RFC em `Em revisão` e uma lista de decisões provisórias, `ADR-TBD-01` e seguintes, na seção de decisões relacionadas. A skill `/rfc-adr` consome essa lista, e só `/rfc-close` troca os provisórios pelos links reais, confere o link nos dois sentidos e marca a RFC como `Aceita`. Nenhum `TBD` pode sobrar no documento final, e isso é uma das checagens do auditor.

### O ciclo de vida de um ADR

Um ADR tem dois momentos formais, a abertura e o aceite, e uma janela de revisão entre os dois. O ciclo está escrito por extenso em [`docs/prompts/rfc-fluxo.md`](docs/prompts/rfc-fluxo.md), seção "Ciclo de vida de um ADR", e é seguido pelas skills `/rfc-adr` e `/rfc-close`.

A **abertura** exige um gatilho: a decisão está classificada como `ACORDADO` na ata. Ponto `DIVERGENTE` ou `EM ABERTO` não abre ADR, vira questão em aberto na RFC. Em seguida vem a deduplicação, porque dois pontos de eixos diferentes podem descrever a mesma escolha e o critério é um arquivo por decisão, não por ponto de debate. O número é alocado pela skill, o arquivo nasce em `Proposto`, e o tracker recebe as linhas antes de o ADR avançar.

O **aceite** roda no fechamento e é coletivo. São cinco condições, todas obrigatórias: nenhum achado `bloqueador` aberto no relatório do revisor, nenhum item marcado como `INVENTADO`, link de mão dupla entre RFC e ADR sem nenhum `TBD` restante, todo caminho de código citado existente no repositório, e as linhas já presentes no tracker. Cumpridas as cinco, `Proposto` vira `Aceito` em cada arquivo, e só então a RFC é marcada como `Aceita`. RFC aceita com ADR ainda em `Proposto` é estado inválido, e o revisor trata isso como bloqueador.

**Depois do aceite, um ADR não é editado para mudar a decisão.** Ele é substituído: abre-se um novo ADR e o antigo passa a `Substituído por ADR-NNN`, mantendo o texto original como registro histórico. Corrigir um erro factual, um caminho de arquivo ou a redação é edição normal e não mexe no status. Decisão que deixou de valer sem sucessora vira `Descontinuado`. Reescrever a decisão dentro do arquivo aceito apagaria justamente o que o ADR existe para preservar, que é o motivo pelo qual se decidiu daquele jeito naquele momento.

### O tracker está preso a uma versão do documento

Cada linha do tracker descreve um item de uma versão específica do documento. Regerar o documento invalida essas linhas em bloco, e elas não deixam de existir sozinhas. Por isso a regeração remove as linhas do documento antigo antes de chamar o tracker, que reconstrói a partir do texto atual. Um tracker desatualizado é pior que um tracker ausente, porque aparenta rastreabilidade sobre conteúdo que já não existe.

### A decisão de fundo sobre o entrevistado

O entrevistado poderia ter recebido um briefing fictício sobre um produto inventado. Não recebeu. Como o repositório já contém a fonte da verdade real, ele responde exclusivamente a partir de `TRANSCRICAO.md` e do código, e classifica toda lacuna em uma de três categorias: `NÃO DISCUTIDO`, `ADIADO` ou `DESCARTADO`. Essa classificação é o que alimenta corretamente as seções de escopo e trade-offs, e é o que impede que uma pergunta sem resposta vire uma resposta inventada.

---

## Prompts customizados

### 1. Prompt de entrevista de PRD

O artefato central da entrega, em [`docs/prompts/entrevista-prd.md`](docs/prompts/entrevista-prd.md). São 572 linhas que definem papel, princípios de entrevista, 12 etapas, estrutura JSON, checagens de consistência e o esqueleto exato de saída. Os trechos abaixo são os que foram acrescentados **depois** de rodar a entrevista, em resposta a falhas observadas na prática:

```markdown
3. Objetivos e métricas de sucesso

    Transformar objetivos em metas quantitativas. Ligar objetivo → métrica → meta alvo.

    Distinga meta de produto de parâmetro de engenharia. Um timeout, um limite de
    tamanho ou um número de tentativas é parâmetro de projeto, não meta de sucesso.
    Quando a fonte só tiver parâmetros, use-os como metas verificáveis, mas registre
    explicitamente que não existe métrica de produto definida, em vez de promover
    parâmetro a KPI sem aviso.
```

```markdown
## Checagens de Consistência antes de finalizar

- Nenhum critério de aceitação afirma um comportamento que algum requisito declara
  em aberto. Critério que depende de decisão não tomada é critério não verificável.
- Cada afirmação do PRD veio do que o usuário respondeu. Ao redigir, você não
  acrescentou restrição, exclusividade ou limite que ninguém enunciou. Palavras como
  "apenas", "único", "sempre" e "nunca" precisam ter sido ditas, não inferidas.
- O que está marcado como hipótese durante a coleta continua marcado como hipótese
  no PRD, e não foi promovido a fato na redação.
```

### 2. Prompt do entrevistado

Em [`.claude/agents/po-entrevistado.md`](.claude/agents/po-entrevistado.md). O núcleo é a classificação de lacunas:

```markdown
# Quando a informação não existe

Não preencha o vazio. Classifique explicitamente:

- **NÃO DISCUTIDO**: o tema não aparece na transcrição nem no código.
  Diga "Isso não foi discutido na reunião" e, se o entrevistador oferecer opções,
  escolha a mais coerente com o que já foi decidido, marcando como hipótese.
- **ADIADO**: foi mencionado e empurrado para uma fase futura. Diga qual fase e
  cite o timestamp. Isso é insumo de "fora de escopo", não de requisito.
- **DESCARTADO**: foi levantado e recusado. Diga quem recusou e o motivo.
  Isso é insumo de "fora de escopo" e de trade-off.

Distinguir esses três casos é a parte mais valiosa da sua resposta. Não colapse
"adiado" e "descartado" em "fora de escopo" sem dizer qual dos dois é.
```

### 3. Prompt do agente de rastreabilidade

Em [`.claude/agents/tracker-rastreabilidade.md`](.claude/agents/tracker-rastreabilidade.md). A regra que impede o erro de atribuição:

```markdown
# Regra central: a atribuição vem da transcrição, nunca do documento

O documento pode conter citações de origem, no formato `[TRANSCRICAO 09:02]` ou
um caminho de arquivo. **Trate essas citações como ponteiro para conferir, nunca
como atribuição pronta.** Elas não carregam o nome do falante, e vários
timestamps da reunião têm dois ou três falantes diferentes. Copiar a citação
produz atribuição errada.
```

E a regra que protege a honestidade da cobertura:

```markdown
# Honestidade da cobertura

Um item do documento que você não conseguir rastrear não vira linha inventada e
não é omitido em silêncio. Ele entra no seu relatório como item sem origem
identificável. Inflar a taxa de cobertura escondendo o que não foi rastreado é a
pior falha possível deste agente, porque destrói exatamente a garantia que o
tracker existe para dar.
```

### 4. Prompt do fluxo de RFC

Em [`docs/prompts/rfc-fluxo.md`](docs/prompts/rfc-fluxo.md). É a fonte única das quatro skills e dos quatro agentes da RFC: fontes da verdade, formatos de análise e de ata, esqueletos de RFC e de ADR, estilo. O núcleo é a regra que decide o que pode virar decisão:

```markdown
# Regra de âncora

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
```

E a regra que impede a RFC de virar FDD, o erro mais provável quando o mesmo modelo escreve os dois documentos:

```markdown
# Regra de altura

A RFC opera em nível de arquitetura. É proibido na RFC:

- payload de exemplo de request ou response
- matriz de erros ou lista de códigos `WEBHOOK_*`
- assinatura de função, DDL de tabela, nome de coluna, schema de validação
- lista de endpoints com status codes

Citar que existe uma tabela de outbox, um worker e um endpoint de administração é
nível de RFC. Descrever as colunas dessa tabela não é.
```

---

## Iterações e ajustes

Dez iterações principais até o PRD e o tracker ficarem prontos, ao longo de duas execuções completas do ciclo. As mais instrutivas:

### 1. A IA inventou um requisito, e não foi o agente que eu esperava

O erro mais relevante do processo inteiro não veio do entrevistado, veio da redação. A transcrição diz, em `[09:31] Marcos`, que a secret "é gerada pela gente e devolvida na criação". No PRD isso virou "devolvida na criação, **único momento em que ela é devolvida**". A restrição de exclusividade nunca foi dita por ninguém, e ainda contradizia outro item do próprio documento, que registrava o armazenamento da secret como ponto em aberto.

Isso mudou o desenho do processo. O risco de invenção não estava na coleta, estava na escrita, e por isso o revisor deixou de ser opcional. Também gerou a checagem sobre as palavras "apenas", "único", "sempre" e "nunca", que são o rastro linguístico típico desse tipo de invenção.

### 2. Critérios de aceitação que afirmavam o que os requisitos declaravam em aberto

Dois dos cinco bloqueadores da auditoria eram a mesma falha estrutural. O RF-07 registrava como pergunta em aberto se a linha sai da outbox ao ir para a dead letter queue, e o critério de aceitação correspondente afirmava com todas as letras que ela sai. O RF-09 dizia que o mecanismo de assinatura durante o grace period não estava definido, e o critério testava esse mecanismo.

É uma inconsistência que passa despercebida em leitura linear, porque as duas seções estão a trezentas linhas de distância e cada uma está correta isoladamente. Só um cruzamento explícito pega, e por isso virou checagem no prompt.

### 3. O prompt exigia uma métrica que a fonte não tinha

A etapa de objetivos manda ligar objetivo, métrica e meta alvo. A reunião não produziu nenhum KPI: produziu parâmetros de engenharia como timeout de 10 segundos, teto de 64KB e cinco tentativas. O entrevistado se recusou a promover parâmetro a métrica de sucesso e marcou como não discutido, o que estava certo, mas deixou o PRD sem meta de disponibilidade e em conflito com a checagem que exige performance e disponibilidade sempre presentes.

A resolução foi assumir o default de 99,9%, marcado como hipótese e com a ausência de SLO registrada logo abaixo. A lição virou regra no prompt: um default silencioso é indistinguível de uma decisão real para quem lê o documento seis meses depois.

### 4. O tracker não foi gerado, e a causa não era esquecimento

Terminado o ciclo, o tracker seguia vazio. Investigando, a causa foi de design: a skill mencionava o tracker uma única vez, numa frase passiva ("elas alimentam o `docs/TRACKER.md`"), e o checklist de fechamento tinha quatro passos, nenhum deles gerando o arquivo. Uma menção descritiva não vira ação.

Havia uma segunda causa, mais séria. O formato de citação do entrevistado é `[TRANSCRICAO 09:02]`, sem o nome do falante, e o tracker exige `[09:17] Diego`. Vários timestamps da reunião têm dois ou três falantes: às 09:34 falam Diego, Bruno e Marcos. Copiar as citações teria produzido atribuições erradas em série. A correção foi criar um agente que vai à transcrição por conta própria, e proibir explicitamente que a atribuição de quem redigiu seja repassada a ele.

### 5. Quatro correções propostas, duas descartadas

Diagnosticada a falha do tracker, a proposta inicial tinha quatro correções. Duas foram descartadas depois de decidir criar o agente: ambas existiam para transportar a atribuição do entrevistado até o tracker, e com o agente lendo a fonte por conta própria, esse transporte deixou de ser desnecessário e passou a ser nocivo. Criaria uma segunda fonte de atribuição, mais fraca que a primeira, disponível para alguém confiar nela num momento de pressa.

A quarta correção mudou de natureza e melhorou: em vez de o revisor apenas verificar se o tracker existe, ele cruza classificações. Nenhum item pode estar marcado como inventado no relatório do revisor e como rastreado no tracker. É um erro que nenhum dos dois pega sozinho.

### 6. Vinte achados aplicados no PRD

A auditoria devolveu 5 bloqueadores, 7 ajustes e 8 notas. Todos foram aplicados. Além dos já descritos, os mais substantivos: cinco riscos tinham "plano de contingência não definido", que é um campo vazio disfarçado de conteúdo, e receberam plano B real; a garantia de ordenação afirmava mais do que o desenho sustenta, já que retry com backoff quebra a ordem de chegada; e o critério de prefixo `WEBHOOK_` em todos os códigos de erro contradizia quatro requisitos que usam códigos transversais já existentes no código.

### 7. A marcação de hipótese foi aplicada de forma desigual dentro do mesmo documento

Na segunda execução, o revisor apontou dois bloqueadores da mesma família. O PRD dizia com todas as letras, na seção de problemas, que a reunião não classificou prioridade e que a classificação era hipótese. Mas atribuía prioridade alta ou média a cada um dos oito requisitos funcionais, e probabilidade baixa ou média a cada um dos seis riscos, sem nenhuma marcação, como se fossem consenso da reunião. A transcrição não atribui prioridade nem probabilidade a nada.

O que torna esse caso instrutivo é que a regra já estava no prompt e já tinha sido obedecida uma vez, poucas seções antes. O redator marcou a hipótese onde ela era visível, na seção que fala explicitamente sobre priorização, e deixou passar onde ela estava embutida em um campo de formulário. Um campo obrigatório do esqueleto é um convite a preencher, e preencher parece menos uma afirmação do que escrever uma frase. A correção foi registrar a hipótese uma vez por seção, no mesmo formato já usado na tabela de métricas.

### 8. Um bloqueador de rastreabilidade cujo diagnóstico estava certo pela razão errada

O revisor abriu o relatório acusando o tracker de fabricar cobertura: nove linhas descreviam riscos, dependências e requisitos não funcionais que não existiam em lugar nenhum do PRD. A acusação era verificável e correta. A causa não era.

O tracker não tinha inventado nada. Ele estava íntegro para o PRD da execução anterior, que acabara de ser sobrescrito. Eram 156 linhas descrevendo um documento que deixara de existir minutos antes. O revisor não tinha como saber disso, porque só enxerga os arquivos no estado atual, e a leitura mais natural de uma linha sem correspondência é que ela foi inventada. Duas falhas muito diferentes, um sintoma idêntico. Foi o que motivou a regra de que a regeração de um documento remove as linhas dele do tracker antes de reconstruir.

### 9. Um critério de aceitação que perdeu um número no meio do caminho

O backoff é 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. O critério de aceitação listou os quatro primeiros e omitiu o último. Isoladamente parece um deslize de digitação, mas o efeito é maior: sem as 12 horas a soma dá cerca de 2,6 horas, e o documento afirma em outros três lugares que a janela cobre cerca de 15 horas. Um critério de aceitação é o que alguém vai transformar em teste, então a versão truncada teria virado um teste que valida o comportamento errado e passa.

O revisor pegou pela aritmética, não pela leitura. Vale registrar como a classe de erro que só um cruzamento numérico entre seções encontra, irmã da inconsistência descrita na iteração 2.

### 10. Um achado do revisor que não era um achado

O relatório questionou o campo Responsável, por não haver na transcrição nenhuma linha designando alguém como dono do PRD. Está correto quanto ao fato e errado quanto à conclusão: responsável e versão são os dois campos que o processo deliberadamente pergunta ao humano, justamente porque não estão na fonte. O revisor lê arquivos, não a sessão, e por isso enxerga como lacuna de rastreabilidade aquilo que foi confirmado fora do arquivo.

O achado foi registrado e não aplicado. É o exemplo mais limpo de que o relatório de um auditor é entrada para julgamento, não veredito: aplicar os quatro achados sem discriminar teria degradado o documento em vez de corrigi-lo. Também mostra o limite do papel, que audita o produto e não consegue auditar o processo de coleta.

E uma iteração que não é sobre documento nenhum, e sim sobre o processo que os produz:

### 11. O fluxo da RFC foi redesenhado três vezes antes de virar arquivo

Iteração de processo, não de documento: aconteceu antes de existir uma linha da RFC. O desenho inicial tinha três etapas, uma skill para criar a RFC, uma para debater e uma para fechar, com as ADRs saindo do debate. A revisão crítica derrubou três coisas. O elenco do debate não tinha segurança, e três das seis decisões principais da reunião são de segurança e de garantia de entrega, então elas sairiam fracas ou nem apareceriam. O debate não tinha critério de parada nem contrato de saída, o que produz rodadas que não convergem e pontos que somem sem registro. E fechar a RFC exigia links para ADRs que ainda não existiam, a dependência circular descrita acima.

O desenho final tem quatro etapas, quatro agentes e um estado intermediário. Vale registrar que o ganho maior não veio de acrescentar um agente, veio de separar a análise do debate: a fase cega é o que impede que os três especialistas virem uma voz só.

---

## Como navegar a entrega

### Ordem sugerida de leitura

1. **[`README.md`](README.md)** este arquivo, para entender o processo antes do produto.
2. **[`docs/prompts/entrevista-prd.md`](docs/prompts/entrevista-prd.md)** o prompt que gerou o PRD. Ler antes do PRD deixa visível o que é estrutura imposta e o que é conteúdo extraído da reunião.
3. **[`docs/PRD.md`](docs/PRD.md)** o problema, o público, o escopo e as métricas. Responde por que e o quê.
4. **[`docs/prompts/rfc-fluxo.md`](docs/prompts/rfc-fluxo.md)** as regras do fluxo que gerou a RFC e as ADRs, incluindo a regra de âncora e a regra de altura entre documentos.
5. **[`docs/RFC.md`](docs/RFC.md)** a proposta técnica e as questões em aberto. Responde como pretendemos resolver.
6. **[`docs/debates/`](docs/debates/)** as três análises e a ata do debate. Não é entregável do desafio, é a evidência de onde saíram as questões em aberto e as decisões registradas.
7. **[`docs/adrs/`](docs/adrs/)** cada decisão isolada, com contexto e consequências.
8. **[`docs/FDD.md`](docs/FDD.md)** a especificação de implementação.
9. **[`docs/TRACKER.md`](docs/TRACKER.md)** a rastreabilidade de cada item. Serve como conferência: qualquer afirmação dos documentos anteriores deve ter linha aqui, com timestamp e falante ou caminho de arquivo.

### Mapa de arquivos

```
README.md                              processo de produção (este arquivo)
DESAFIO.md                             enunciado original do desafio
TRANSCRICAO.md                         fonte primária: a reunião de 55 minutos

docs/
├── PRD.md                             produto e negócio
├── RFC.md                             proposta técnica para revisão
├── FDD.md                             especificação de implementação
├── TRACKER.md                         rastreabilidade item a item
├── adrs/                              decisões arquiteturais isoladas
├── debates/
│   └── RFC-001/                       análises dos três especialistas e ata do debate
└── prompts/
    ├── entrevista-prd.md              prompt de entrevista de PRD
    └── rfc-fluxo.md                   prompt do fluxo de RFC e ADRs

.claude/
├── skills/
│   ├── entrevista-prd/SKILL.md        empacota a entrevista como comando
│   ├── rfc-draft/SKILL.md             rascunho da RFC pelo Tech Lead
│   ├── rfc-debate/SKILL.md            análises independentes e debate
│   ├── rfc-adr/SKILL.md               registro das decisões como ADRs
│   └── rfc-close/SKILL.md             auditoria, links e fechamento
└── agents/
    ├── po-entrevistado.md             responde como o time da reunião
    ├── revisor-prd.md                 audita consistência e rastreabilidade do PRD
    ├── tracker-rastreabilidade.md     escritor único do tracker
    ├── rfc-arquiteto.md               debate arquitetura e escreve as ADRs
    ├── rfc-dev.md                     debate viabilidade no código existente
    ├── rfc-seguranca.md               debate assinatura, secret e replay
    └── rfc-revisor.md                 audita a RFC e as ADRs prontas

src/                                   aplicação existente (OMS)
```

### Como reproduzir o processo

Com o repositório aberto no Claude Code:

```
/entrevista-prd auto
```

Roda a entrevista completa com o subagente respondendo a partir da transcrição, gera o PRD, chama o revisor, aplica os achados e atualiza o tracker. Sem o argumento `auto`, a mesma entrevista roda com você respondendo às perguntas.

Com o PRD e o tracker prontos, o fluxo da RFC roda em quatro comandos, na ordem:

```
/rfc-draft
/rfc-debate
/rfc-adr
/rfc-close
```

Cada um para ao final e pede aprovação antes do seguinte. As pausas são de propósito: debater em cima de um rascunho ruim desperdiça o debate inteiro, e registrar como ADR uma decisão que não foi conferida propaga o erro para todos os documentos que citarem aquela ADR. As quatro skills são idempotentes, rodar de novo corrige o que existe em vez de duplicar arquivo, ADR ou linha de tracker.
