# [03-02] Bancos Vetoriais na Prática — Onde Moram os Embeddings

## Por que isso importa?

Bancos vetoriais são a infraestrutura invisível por trás de todo sistema RAG. Entender como funcionam — e por que são diferentes de bancos SQL — determina qual escolher, como escalar e o que vai travar em produção.

---

## Conceito Central

**Um banco vetorial armazena embeddings (vetores de alta dimensão) e é otimizado para uma única operação que bancos tradicionais não fazem bem: busca por similaridade. Você pergunta "quem é parecido com X?" e o banco responde em milissegundos, mesmo com milhões de vetores.**

---

## Aprofundamento

### Por que não usar SQL para embeddings?

Um embedding típico tem 1.536 dimensões (OpenAI ada-002). Buscar "os 10 vetores mais próximos de [v1, v2, ..., v1536]" em SQL requer calcular a distância euclidiana ou cosine similarity contra cada linha. Com 1 milhão de documentos: 1 bilhão de operações de ponto flutuante por query. Impraticável.

Bancos vetoriais resolvem isso com índices especializados (ANN — Approximate Nearest Neighbors) que trocam precisão perfeita por velocidade. Na prática, 95% dos casos de uso não precisam do "mais próximo exato", precisam do "suficientemente próximo, rapidamente".

### Algoritmos de indexação (o que está por baixo)

Os principais algoritmos usados internamente pelos bancos vetoriais:

**HNSW (Hierarchical Navigable Small World)**
- Cria um grafo de múltiplas camadas entre vetores similares
- Busca começa nas camadas superiores (granularidade grossa) e afina
- Ótimo para updates frequentes e alta velocidade
- Usado por: Qdrant, Weaviate, pgvector

**IVF (Inverted File Index)**
- Divide o espaço vetorial em "clusters" (Voronoi cells)
- Busca ocorre só nos clusters mais próximos da query
- Mais eficiente em memória, mas pior para updates
- Usado por: FAISS (Meta), Pinecone

**ANNOY (Approximate Nearest Neighbors Oh Yeah)**
- Constrói árvores binárias aleatórias no espaço vetorial
- Mais lento que HNSW mas usa menos RAM
- Usado por: Spotify (para recomendação musical)

### Métricas de distância

A "similaridade" entre dois vetores é calculada com:

**Cosine similarity** (mais comum em NLP)
```
sim(A, B) = (A · B) / (|A| × |B|)
```
Mede o ângulo entre os vetores, ignorando magnitude. Um texto longo e um curto sobre o mesmo tema terão cosine similarity alta.

**Distância euclidiana**
```
dist(A, B) = √(Σ(Ai - Bi)²)
```
Mede a distância "real" no espaço. Sensível à magnitude, menos comum em NLP.

**Dot product (produto interno)**
Usado quando os embeddings foram treinados especificamente para isso. Mais rápido mas requer embeddings normalizados.

**Regra prática:** use cosine similarity para texto quase sempre.

### Os principais bancos vetoriais em 2024-2025

| Banco | Tipo | Ideal para | Trade-off |
|---|---|---|---|
| **Chroma** | Local/open-source | Prototipagem, desenvolvimento | Não escala para produção |
| **Qdrant** | Self-hosted ou cloud | Produção, controle de infra | Requer DevOps |
| **Weaviate** | Self-hosted ou cloud | Busca híbrida (vetorial + keyword) | Configuração complexa |
| **Pinecone** | SaaS gerenciado | Produção sem DevOps | Custo por query, vendor lock-in |
| **pgvector** | Extensão PostgreSQL | Times com infra Postgres já | Performance inferior a VDBs especializados |
| **FAISS** | Biblioteca (Meta) | Offline, altíssima performance | Sem persistência nativa, API low-level |

**Para começar:** Chroma (local) → Qdrant (produção).

### Chunking — a arte de dividir documentos

Antes de indexar, você precisa dividir os documentos em chunks. A estratégia importa mais do que a maioria pensa:

**Fixed-size chunking**
```python
text_splitter = CharacterTextSplitter(chunk_size=500, chunk_overlap=50)
```
Simples, previsível. Problema: pode cortar frases no meio.

**Recursive character splitting** (padrão LangChain)
```python
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", " ", ""]
)
```
Tenta respeitar parágrafos, depois frases, depois palavras.

**Semantic chunking**
Usa embeddings para encontrar fronteiras semânticas naturais. Mais inteligente, mais caro.

**Regra prática:**
- Chunk size 200-300 tokens → alta precisão, mais chunks, recuperação mais granular
- Chunk size 500-1000 tokens → mais contexto por chunk, menos precisão
- Overlap de 10-15% → evita perder contexto nas bordas

### Metadados — o poder ignorado

Cada chunk pode ter metadados associados:

```python
{
    "text": "A LGPD define titular como...",
    "source": "lgpd-lei-13709.pdf",
    "page": 12,
    "section": "Artigo 5",
    "date_indexed": "2025-01-15",
    "doc_type": "legislation"
}
```

Com metadados, você pode fazer **filtered search**: "buscar chunks similares à query, mas apenas dos documentos do último trimestre do tipo 'contrato'". Isso é fundamental para produção.

---

## Aplicação Prática

**Fluxo completo de indexação:**

```python
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 1. Chunking
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(docs)

# 2. Embedding + indexação
embedding_model = OpenAIEmbeddings()  # ou modelo local via Ollama
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embedding_model,
    persist_directory="./chroma_db"
)

# 3. Busca
results = vectorstore.similarity_search(
    query="O que é titular de dados na LGPD?",
    k=4,  # top-4 chunks
    filter={"doc_type": "legislation"}  # filtro por metadado
)
```

**Avaliando qualidade de recuperação:**
- **Precision@K:** dos K chunks recuperados, quantos são realmente relevantes?
- **Recall:** dos chunks relevantes totais, quantos foram recuperados?
- **MRR (Mean Reciprocal Rank):** o chunk mais relevante está em qual posição?

---

## Conexões

- [02-01 — Embeddings](../Modulo-02-NLP-Embeddings-Prompt/02-01-Embeddings-e-Espaco-Vetorial.md): o que é gerado antes de indexar
- [03-01 — Por que RAG](03-01-Por-que-RAG-Existe.md): o problema que os bancos vetoriais resolvem
- [03-03 — Pipelines RAG](03-03-Pipelines-RAG-do-Zero.md): usando o banco em um pipeline completo

---

## Recursos Complementares

- **Chroma docs**: [docs.trychroma.com](https://docs.trychroma.com)
- **Qdrant docs**: [qdrant.tech/documentation](https://qdrant.tech/documentation)
- **"The Illustrated RAG" — Jay Alammar**: explicação visual de excelente qualidade
- **Alura — Formação Agentes IA, semana 2**: implementação com Qdrant + LangChain

---

## Auto-reflexão

1. Por que "busca por similaridade" é um problema que SQL resolve mal mas bancos vetoriais resolvem bem?
2. Se você tem 10 milhões de documentos, por que ANN (Approximate) é preferível a busca exata?
3. Um chunk de 200 tokens vs. um chunk de 1000 tokens — quais são as vantagens de cada um no contexto de recuperação para RAG?
4. Para um sistema interno de uma empresa com 500GB de contratos PDF, quais critérios você usaria para escolher entre Chroma, Qdrant e Pinecone?

---

*[03-02] | Anterior: [03-01](03-01-Por-que-RAG-Existe.md) | Próximo: [03-03](03-03-Pipelines-RAG-do-Zero.md)*
*Referências: Qdrant docs; LangChain splitters; "The Illustrated Word2Vec" (Jay Alammar)*
