# Da Reunião ao Documento: processo de produção

Documentação do processo usado para transformar a transcrição de uma reunião técnica em um pacote de design docs. O enunciado original do desafio está preservado em [`DESAFIO.md`](DESAFIO.md).

---

## Sobre o desafio

Uma empresa opera um Order Management System em produção e decidiu construir um sistema de webhooks para notificar clientes B2B quando o status de um pedido muda. A decisão técnica já foi tomada numa reunião de 55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança, e nada sobrou dela além da transcrição literal da call. A tarefa é produzir, a partir dessa transcrição e do código da aplicação existente, a documentação técnica que permita ao time começar a implementar: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

A restrição que define o desafio não é a quantidade de documentos, é a proibição de inventar. Toda informação registrada precisa ser rastreável a um timestamp da transcrição ou a um arquivo do código. Isso muda a natureza do trabalho: o difícil não é gerar texto, é impedir que o texto gerado escorregue para o plausível. A reunião contém decisões fechadas, mas também alternativas recusadas, pontos adiados para fases futuras e temas que ninguém tocou, e essas quatro categorias precisam terminar em lugares diferentes do documento. Separar o que foi descartado do que foi adiado, e ambos do que simplesmente não foi discutido, é a parte que exige critério.

---

## Ferramentas de IA utilizadas

- **Claude Code (Opus 5)** como ambiente principal. Sessão interativa no terminal, com acesso de leitura e escrita ao repositório, usada tanto para produzir os documentos quanto para orquestrar os subagentes.
- **Subagentes do Claude Code** para separar papéis que não podem ser exercidos pelo mesmo contexto. Três foram criados neste projeto, descritos na seção de workflow.
- **Skill customizada do Claude Code** (`.claude/skills/`) para empacotar a entrevista de PRD como comando invocável, com modo manual e modo automatizado.

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
6. RFC, FDD e ADRs, derivados do PRD e da transcrição, seguindo o mesmo ciclo.

### Os quatro papéis

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

---

## Como navegar a entrega

### Ordem sugerida de leitura

1. **[`README.md`](README.md)** este arquivo, para entender o processo antes do produto.
2. **[`docs/prompts/entrevista-prd.md`](docs/prompts/entrevista-prd.md)** o prompt que gerou o PRD. Ler antes do PRD deixa visível o que é estrutura imposta e o que é conteúdo extraído da reunião.
3. **[`docs/PRD.md`](docs/PRD.md)** o problema, o público, o escopo e as métricas. Responde por que e o quê.
4. **[`docs/RFC.md`](docs/RFC.md)** a proposta técnica e as questões em aberto. Responde como pretendemos resolver.
5. **[`docs/adrs/`](docs/adrs/)** cada decisão isolada, com contexto e consequências.
6. **[`docs/FDD.md`](docs/FDD.md)** a especificação de implementação.
7. **[`docs/TRACKER.md`](docs/TRACKER.md)** a rastreabilidade de cada item. Serve como conferência: qualquer afirmação dos documentos anteriores deve ter linha aqui, com timestamp e falante ou caminho de arquivo.

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
└── prompts/
    └── entrevista-prd.md              prompt de entrevista de PRD

.claude/
├── skills/
│   └── entrevista-prd/SKILL.md        empacota a entrevista como comando
└── agents/
    ├── po-entrevistado.md             responde como o time da reunião
    ├── revisor-prd.md                 audita consistência e rastreabilidade
    └── tracker-rastreabilidade.md     escritor único do tracker

src/                                   aplicação existente (OMS)
```

### Como reproduzir o processo

Com o repositório aberto no Claude Code:

```
/entrevista-prd auto
```

Roda a entrevista completa com o subagente respondendo a partir da transcrição, gera o PRD, chama o revisor, aplica os achados e atualiza o tracker. Sem o argumento `auto`, a mesma entrevista roda com você respondendo às perguntas.
