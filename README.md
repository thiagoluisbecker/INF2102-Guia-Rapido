# Guia Rápido RAG

Assistente de consulta, em linguagem natural, aos **Guias Rápidos de saúde da
Secretaria Municipal de Saúde do Rio de Janeiro** (Diabetes Mellitus e
Pré-Natal). O sistema usa RAG (Retrieval-Augmented Generation) multimodal:
indexa cada página dos PDFs como um vetor, recupera as páginas mais relevantes
para a pergunta e gera uma resposta que **cita a página de origem** de cada
afirmação (`pág. N`), com link direto para a página do PDF.

O relatório completo do projeto (descrição, cenários, documentação técnica e
manual de utilização) está em [RELATORIO.md](RELATORIO.md).

## Estrutura

| Arquivo | Papel |
|---|---|
| `main.ipynb` | Pipeline completo: split dos PDFs, vetorização, indexação no Chroma e exemplos de consulta |
| `agent.py` | Função `answer_question`: busca semântica + geração com citação obrigatória |
| `guias/` | Os dois PDFs oficiais usados como base de conhecimento |
| `pyproject.toml` | Dependências do projeto (gerenciadas com `uv`) |
| `.env.example` | Modelo do arquivo de credenciais |

A pasta `chroma_db/` (índice vetorial) é gerada localmente na primeira
execução e não é versionada.

## Como rodar

Pré-requisitos: Python ≥ 3.10, [uv](https://docs.astral.sh/uv/) e uma chave da
API do Google AI Studio (<https://aistudio.google.com/apikey>).

```bash
# 1. Instale as dependências (cria .venv automaticamente)
uv sync

# 2. Configure a credencial
cp .env.example .env   # e edite, colocando sua GOOGLE_API_KEY

uv run jupyter lab main.ipynb  
```

Execute as células do `main.ipynb` em ordem:

1. **Split e embeddings** — define as funções de corte do PDF e a classe
   `GeminiEmbeddings`.
2. **Vector store** — abre/cria a coleção `guias_saude` em um Chroma local.
3. **Ingestão** — na primeira execução, vetoriza os dois guias em `/guias`
   (~276 páginas, uma chamada de API por página leva alguns minutos). Nas execuções seguintes a célula detecta a coleção
   populada e pula a ingestão.
4. **Consultas** — exemplos de busca semântica pura (`query`) e de resposta
   gerada com citações (`answer_question`).

Para fazer sua própria pergunta:

```python
from agent import answer_question

resposta = answer_question(
    vector_store, client,
    "Quais são os sintomas de hipoglicemia?",
    k=5,                # nº de páginas recuperadas 
    # source="Livro_GuiaRapido-DiabetesMellitus_PDFDigital_20231113.pdf",  # filtro opcional por guia
)
print(resposta.as_markdown())
```

## Dependências e por que cada uma foi usada

| Dependência | Para quê | Por que ela (e não outra) |
|---|---|---|
| `google-genai` | Embeddings (`gemini-embedding-2-preview`) e geração (`gemini-2.5-flash`) | Único provedor de embeddings que aceita **PDF nativo** (`Part.from_bytes`), preservando tabelas, fluxogramas e layout sem OCR; |
| `pypdf` | Corte do PDF em páginas individuais e extração de texto | sem dependências nativas e sem restrição de licença |
| `langchain-chroma` | Vector store persistente local  | Chroma roda embutido, sem servidor nem custo , e suporta filtro por metadado (vs. FAISS) |
| `python-dotenv` | Carrega `GOOGLE_API_KEY` do `.env` | Mantém a credencial fora do código e do versionamento |
| `ipykernel` (dev) | Kernel Jupyter | Necessário apenas para executar o `main.ipynb` |


## Ressalvas

- Ferramenta de **apoio à consulta**, não de decisão clínica: a resposta deve
  sempre ser conferida no guia oficial.
- Cobre apenas os dois guias incluídos; perguntas fora deles recebem indicação
  explícita de que a informação não está no material .
- Requer conexão com a internet e chave de API válida (dados das perguntas
  trafegam para a API do Google).
