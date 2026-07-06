# Relatório Final do Projeto

# Guia Rápido RAG: Assistente de Consulta a Guias Rápidos de Saúde

## Breve Descrição

O **Guia Rápido RAG** é um assistente de recuperação de informação clínica que
responde, em linguagem natural e em português, perguntas sobre os **Guias
Rápidos oficiais da Secretaria Municipal de Saúde do Rio de Janeiro (SMS-Rio)**,
citando obrigatoriamente a página de origem de cada afirmação.

**Principal função:** recuperação rápida e citável de condutas clínicas
recomendadas em guias de referência (protocolos) em PDF.

**Funções específicas relevantes:**

- Indexação de guias em PDF, página a página, com embeddings multimodais
  (o vetor representa a página inteira: texto, tabelas e fluxogramas);
- Busca semântica vetorial: dada uma pergunta, recupera as `k` páginas mais
  relevantes, com filtro opcional por guia;
- Geração de resposta fundamentada: o modelo lê as páginas originais do PDF e
  responde citando `(pág. N)`, com link direto `arquivo.pdf#page=N`;
- Indicação explícita quando a informação solicitada **não** está no material
  indexado;
- Suporte à inclusão incremental de novos guias, sem reprocessar os já
  indexados.

**Usuários primordiais:** profissionais de saúde da Atenção Primária (médicos
e enfermeiros) que precisam confirmar condutas no ponto de atendimento, e,
secundariamente, gestores de saúde e estudantes das áreas de saúde que queiram
consultar os protocolos vigentes.

**Natureza do programa:** prova de conceito (PoC) parcial, desenvolvida na
disciplina INF2921 (PUC-Rio) pelo Grupo E. Nesta versão estão indexados dois
guias: *Diabetes Mellitus* (2023) e *Pré-Natal* (2025). A integração com dados
estatísticos do Data Zoom Saúde / DataSUS (via MCP) foi demonstrada em
separado e não faz parte deste pacote.

**Ressalvas:**

- O sistema é um **apoio à consulta de protocolos, não um decisor clínico**.
  Toda resposta cita a página de origem justamente para que o profissional
  confira o guia oficial antes de qualquer conduta.
- As respostas são geradas por um modelo de linguagem (Gemini 2.5 Flash) e,
  como todo LLM, ele está sujeito a **alucinações** (afirmações que parecem
  plausíveis mas não constam do material). O projeto adota mitigações em várias
  camadas: (i) a própria arquitetura RAG restringe o modelo às páginas
  recuperadas dos guias, em vez de deixá-lo responder de memória; (ii) as
  páginas são enviadas ao modelo como o PDF original, e não como texto
  extraído, reduzindo erros de interpretação de tabelas e fluxogramas;
  (iii) a instrução de sistema obriga citar `(pág. N)` em cada afirmação
  importante e declarar explicitamente quando a informação não está nas
  páginas fornecidas; (iv) a citação obrigatória torna cada afirmação
  verificável na fonte. Outras mitigações são possíveis (reduzir a temperatura
  da geração, reranking das páginas recuperadas, validação automática das
  citações), mas **nenhuma combinação de técnicas elimina o risco**: o modelo
  ainda pode omitir uma condição relevante, misturar condutas de páginas
  diferentes ou citar uma página que não sustenta a afirmação. Por isso, a
  resposta deve ser tratada como um apontador para o guia; a conduta só deve
  ser adotada após conferência na página citada do documento oficial.


## Visão de Projeto

### Cenário Positivo 1 (cenário que dá certo)

Marina é médica de família em uma Clínica da Família no Rio de Janeiro.
Durante um atendimento, recebe um homem de 47 anos com quadro sugestivo de
estado hiperosmolar hiperglicêmico. Entre um paciente e outro, ela não tem
tempo de folhear as mais de cem páginas do Guia Rápido de Diabetes Mellitus.
Ela pergunta ao assistente: *"Estou com um paciente homem de 47 anos com
estado hiperosmolar hiperglicêmico, como posso proceder?"*. Em segundos, o
assistente responde com os passos de tratamento imediato (correção da
desidratação, dos distúrbios eletrolíticos e da hiperglicemia) e a orientação
de solicitar Vaga Zero em caso de suspeita de SHH, cada item acompanhado da
citação `(pág. 101)`. Marina clica no link da fonte, o PDF abre exatamente na
página 101, ela confirma a conduta no documento oficial e segue o atendimento.

### Cenário Positivo 2

Carlos é enfermeiro e acompanha o pré-natal de uma gestante de 26 anos, na 12ª
semana, que acaba de receber diagnóstico de infecção por Zika vírus. Ele quer
ter certeza de que não está esquecendo nenhuma providência obrigatória.
Pergunta ao assistente quais procedimentos realizar e recebe uma lista
objetiva: solicitar ultrassonografia mensal a partir do primeiro trimestre,
registrar o exantema no cartão de pré-natal, notificar o caso em até 24 horas
no SINAN-Rio e cadastrar o exame no GAL com a idade gestacional, tudo com a
citação `(pág. 108)` do Guia Rápido de Pré-Natal. Carlos confere a página
citada, executa as notificações no prazo e anota no prontuário que seguiu o
protocolo municipal vigente.

### Cenário Negativo 1 (cenário que expõe uma limitação conhecida e esperada do programa)

Marina atende um paciente com diabetes tipo 2 em uso de metformina e pergunta
ao assistente qual a conduta indicada diante de uma taxa de filtração
glomerular reduzida. O assistente recupera páginas do Guia Rápido de Diabetes
(o guia correto, efetivamente indexado e disponível na coleção) e responde
com uma recomendação de ajuste de dose que soa plausível, citando
`(pág. N)`. Ao abrir a página citada para conferir, Marina percebe que a
afirmação **não consta ali**: o valor de corte de TFG mencionado pelo modelo
não aparece na página indicada nem em nenhuma outra do documento. É uma
alucinação do modelo de linguagem: ele preencheu uma lacuna com uma
informação plausível, mas inexistente no material, mesmo estando restrito às
páginas recuperadas (ver Ressalvas). A falha aqui é do processo de geração da
resposta (do RAG), não de cobertura documental: diferentemente de uma
pergunta sobre um guia que simplesmente não está indexado (caso em que o
sistema declara explicitamente a ausência de informação), aqui o tema está
coberto e a página certa foi recuperada, mas o modelo erra ao gerar a
resposta. Marina só percebe o problema porque confere a citação obrigatória
na página oficial; sem essa conferência, teria seguido uma orientação sem
respaldo no guia.

### Cenário Negativo 2

Ana é enfermeira e pergunta ao assistente qual o esquema de tratamento para
bacteriúria assintomática em uma gestante no segundo trimestre. O assistente
recupera as páginas mais próximas do tema no Guia Rápido de Pré-Natal e
responde com um esquema terapêutico aparentemente completo (antibiótico,
dose e duração), citando `(pág. N)`. Ao abrir a página citada para conferir,
Ana percebe que a resposta **misturou informações**: parte da posologia veio
de uma tabela de outra condição presente na mesma página, e a duração indicada
não corresponde à faixa recomendada pelo guia para o caso dela. A falha é uma
limitação conhecida do programa: a resposta é gerada por um modelo de
linguagem que, mesmo restrito às páginas recuperadas, pode combinar trechos
indevidamente, ler uma tabela de forma errada ou atribuir uma afirmação à
página incorreta (ver Ressalvas). É exatamente para esse caso que a citação
obrigatória existe: como toda afirmação aponta para a página de origem, a
divergência é detectável na conferência. Ana descarta a parte incorreta,
segue o esquema descrito na página oficial do guia e trata a resposta do
assistente como o que ele é: um localizador de condutas, não a conduta em si.

## Documentação Técnica do Projeto

### Especificação de requisitos funcionais e não-funcionais

Requisitos levantados na concepção do projeto para a disciplina `INF2921`, com prioridade e situação 
prova de conceito:

| ID | Requisito funcional | Prioridade | Situação |
|---|---|---|---|
| RF01 | Perguntas em português em linguagem natural | Alta | Atendido |
| RF02 | Busca semântica vetorial em PDFs indexados | Alta | Atendido |
| RF03 | Respostas fundamentadas e com citação | Alta | Atendido |
| RF04 | Suporte a múltiplos PDFs e atualização incremental | Alta | Parcial (2 guias; upsert incremental funciona) |
| RF05 | Manutenção de contexto conversacional na sessão | Alta | Não atendido |
| RF06 | Integração com Data Zoom Saúde (indicadores) | Média | Demonstrado à parte (MCP), não integrado |
| RF07 | Exibir o trecho original do documento na resposta | Média | Atendido (link para a página do PDF) |
| RF08 | Indicar nível de confiabilidade da resposta | Média | Parcial (scores de similaridade exibidos) |
| RF09 | Filtragem de consultas por categoria de protocolo | Baixa | Atendido (filtro por guia via `source`) |
| RF10 | Suporte a consultas por voz | Baixa | Não atendido |

| ID | Requisito não-funcional | Situação |
|---|---|---|
| RNF01 | Usabilidade: interface via browser, sem instalação | Não atendido (uso via notebook) |
| RNF02 | Rastreabilidade: toda resposta cita documento e página | Atendido |
| RNF03 | Segurança: dados clínicos identificáveis não trafegam ao LLM | Não garantido pelo sistema (depende do usuário) |
| RNF04 | Confiabilidade: indicação explícita quando informação é insuficiente | Atendido |
| RNF05 | Escalabilidade: suporte à inclusão de novos documentos | Parcial |
| RNF06 | Conformidade: aderência à LGPD | Não avaliado formalmente |
| RNF07 | Manutenibilidade: suporte à atualização de documentos | Parcial (upsert idempotente por página) |

### Descrição da arquitetura

O sistema tem duas fases:

**Fase 1: Ingestão (uma vez por guia):**

```
PDF do guia
  → split página a página (pypdf; cada página vira um PDF em memória)
  → embedding multimodal da página (gemini-embedding-2-preview,
    task_type=RETRIEVAL_DOCUMENT; o binário do PDF é enviado, sem OCR)
  → upsert no Chroma persistente (./chroma_db, distância cosseno) com
    id determinístico "arquivo::page_N" e metadados {source, path, page}
```

**Fase 2: Consulta e resposta (a cada pergunta):**

```
Pergunta em linguagem natural
  → embedding da pergunta (mesmo modelo, task_type=RETRIEVAL_QUERY)
  → busca vetorial no Chroma (top-k páginas; filtro opcional por guia)
  → releitura das páginas originais no PDF em disco (o índice apenas
    localiza; o conteúdo enviado ao modelo é a página fiel)
  → geração com gemini-2.5-flash, com instrução de sistema que obriga
    citação "(pág. N)" e declaração explícita quando a informação não
    está nas páginas fornecidas
  → resposta final + lista de fontes com link arquivo.pdf#page=N e score
```

Decisões estruturantes:

- **1 página = 1 vetor, sem chunking textual.** Os guias são organizados em
  páginas autocontidas (fluxogramas, tabelas de conduta)
- **Embedding do PDF binário, não do texto extraído.** Captura layout, tabelas
  e imagens; o texto extraído pelo pypdf ainda é guardado como `page_content`
  para eventual reranking textual futuro.

### Sobre o código

- **Linguagem:** Python ≥ 3.10, com type hints e dataclasses.
- **Organização:** `main.ipynb` concentra o pipeline de ingestão e os
  exemplos de consulta (formato notebook facilita a inspeção passo a passo da
  PoC); `agent.py` isola a lógica de resposta (`answer_question`) para reúso
  fora do notebook.
- **Dependências:** `google-genai` (embeddings multimodais e geração),
  `pypdf` (split e extração de texto), `langchain-chroma` (vector store
  persistente + abstrações `Embeddings`/`VectorStore`, que permitem trocar de
  backend), `python-dotenv` (credenciais fora do código). A justificativa
  detalhada de cada escolha está no [README.md](README.md).
- **Pontos de atenção para quem for reutilizar:**
  - `GeminiEmbeddings` implementa a interface `Embeddings` do LangChain, mas a
    ingestão usa `vector_store._collection.upsert` diretamente: a API pública
    do `langchain-chroma` re-embeddaria os documentos como texto, descartando
    os vetores multimodais pré-computados.
  - O metadado `path` é relativo à raiz do projeto; execute o código a partir
    da pasta `pfp/` (ou ajuste os paths) para que a releitura dos PDFs
    funcione.
  - `task_type` é assimétrico por design (`RETRIEVAL_QUERY` na pergunta,
    `RETRIEVAL_DOCUMENT` na ingestão); usar o mesmo dos dois lados degrada o
    recall.
  - Limitações conhecidas: sem reranking, `k` fixo por chamada, ingestão sem
    batching (uma chamada de API por página) e geração síncrona (sem
    streaming).

## Manual de Utilização para Usuários Contemplados

O manual cobre as tarefas dos dois perfis: **quem opera o sistema**
(instalação e indexação) e **quem consulta** (profissionais de saúde).

### Guia de Instruções: PREPARAR O AMBIENTE

Para preparar o ambiente, faça:

- Passo 1: instale o [uv](https://docs.astral.sh/uv/) e garanta Python ≥ 3.10.
- Passo 2: rode `uv sync` para criar a `.venv` e instalar as
  dependências.
- Passo 3: copie `.env.example` para `.env` e preencha `GOOGLE_API_KEY` com
  uma chave do Google AI Studio (<https://aistudio.google.com/apikey>).
- Passo 4: abra `main.ipynb` no editor (VS Code, Jupyter Lab) selecionando o
  kernel da `.venv` criada e rode o notebook.

Exceções ou potenciais problemas:

- Se `uv` não estiver disponível: instale as dependências listadas no
  `pyproject.toml` com `pip install` em um ambiente virtual comum.
- Se aparecer erro de autenticação ao rodar o notebook (`API key not valid` ou
  similar): é porque o `.env` não foi criado, está fora da pasta `pfp/` ou a
  chave é inválida; confira o passo 3 e reinicie o kernel.

### Guia de Instruções: INDEXAR OS GUIAS (primeira execução)

Para indexar os guias, faça:

- Passo 1: execute as células do `main.ipynb` em ordem até a célula de
  ingestão (a que define `paths` e chama `pipeline`).
- Passo 2: aguarde a vetorização (~276 páginas, uma chamada de API por
  página; alguns minutos). O progresso é impresso página a página.
- Passo 3: confirme na saída da célula do Chroma o total de itens na coleção
  (deve ser > 0).

Para adicionar um guia novo, faça:

- Passo 1: coloque o PDF na pasta `guias/`.
- Passo 2: acrescente o caminho do arquivo à lista `paths` e execute
  `pipeline("guias/NomeDoArquivo.pdf")`.
- Passo 3: os guias já indexados não são reprocessados (ids determinísticos
  por página garantem upsert sem duplicação).

Exceções ou potenciais problemas:

- Se a ingestão for interrompida no meio: rode novamente
- Se ocorrer erro de cota/limite da API (`429`): é porque a chave gratuita tem
  limite de chamadas por minuto; aguarde e reexecute, ou use uma chave com
  cota maior.
- Se a célula de ingestão indicar "Coleção já populada": é o comportamento
  esperado nas execuções seguintes; para reindexar do zero, apague a pasta
  `chroma_db/` e reexecute.

### Guia de Instruções: CONSULTAR UMA CONDUTA CLÍNICA (fazer uma pergunta)

Para obter uma resposta com citações, faça:

- Passo 1: com o notebook carregado (células de setup executadas), chame:
  ```python
  from agent import answer_question
  resposta = answer_question(vector_store, client, "sua pergunta aqui", k=5)
  ```
- Passo 2: exiba `resposta.as_markdown()`: a resposta vem com citações
  `(pág. N)` no corpo e a lista de fontes ao final, com links que abrem o PDF
  na página exata.
- Passo 3: **confira a página citada no guia oficial** antes de adotar
  qualquer conduta.

Maneiras alternativas de realizar a tarefa:

- Para restringir a busca a um único guia (melhor quando a pergunta é
  claramente de um domínio, p.ex. só pré-natal), passe
  `source="Livro_GuiaRapido-PreNatal_PDFDigital_20250331.pdf"`.
- Para apenas localizar páginas relevantes, sem gerar resposta (mais rápido e
  sem custo de geração), use `query("sua pergunta", k=3)` e abra as páginas
  indicadas.
- Ajuste `k` (nº de páginas recuperadas): valores maiores (5–10) dão mais
  contexto para perguntas amplas; valores menores (3) são mais rápidos e
  baratos para perguntas pontuais.

Exceções ou potenciais problemas:

- Se a resposta disser que a informação não está nas páginas fornecidas: é
  porque o assunto não consta dos guias indexados (ver Cenário Negativo 1) ou
  porque as páginas recuperadas não foram as certas; reformule a pergunta com
  termos do domínio (p.ex. "cetoacidose diabética" em vez de "açúcar alto") ou
  aumente `k`.
- Se a resposta parecer incompleta ou genérica: verifique os scores das
  fontes; valores altos (> ~0,6, distância cosseno) indicam recuperação fraca;
  reformule a pergunta ou filtre por `source`.
- Se o link da fonte não abrir na página correta: é porque o visualizador de
  PDF não suporta a âncora `#page=N` — abra o PDF manualmente na página
  indicada na citação.
- Se aparecer "Nenhuma página relevante encontrada": a coleção está vazia —
  execute a tarefa INDEXAR OS GUIAS.
- **Nunca inclua dados que identifiquem o paciente na pergunta** (nome, CPF,
  prontuário): a pergunta trafega pela API externa.
