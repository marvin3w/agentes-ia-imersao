# [03-03] Pipelines RAG do Zero — Da Pergunta à Resposta Fundamentada

## Por que isso importa?

Saber que RAG existe e que bancos vetoriais funcionam é diferente de construir um pipeline que funciona em produção. Este documento conecta os conceitos em código real, com as armadilhas que só aparecem quando você implementa.

---

## Conceito Central

**Um pipeline RAG tem duas fases separadas: indexação (offline, roda uma vez) e recuperação (online, roda em cada query). Mantê-las mentalmente separadas evita a confusão de quem "quebra as coisas" quando o sistema responde mal.**

---

## Aprofundamento

### Fase 1 — Indexação (offline)

```
[Documentos brutos]
       ↓
[Pré-processamento: limpar, extrair texto de PDFs, HTML, etc.]
       ↓
[Chunking: dividir em pedaços menores]
       ↓
[Embedding: transformar cada chunk em vetor]
       ↓
[Indexar no banco vetorial com metadados]
```

Esta fase roda uma vez (ou quando documentos mudam). Pode ser um script Python, um job cron, ou um pipeline de dados.

### Fase 2 — Recuperação (online, por query)

```
[Query do usuário]
       ↓
[Embedding da query] (mesmo modelo usado na indexação)
       ↓
[Busca top-K chunks similares]
       ↓
[Opcional: reranking dos chunks recuperados]
       ↓
[Montar prompt: contexto + query]
       ↓
[LLM gera resposta]
       ↓
[Opcional: extrair e formatar citações]
```

### Implementação completa com LangChain

```python
# FASE 1 — Indexação
from langchain_community.document_loaders import PyPDFLoader, DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Qdrant

def build_index(docs_path: str, collection_name: str):
    # Carregar documentos
    loader = DirectoryLoader(docs_path, glob="**/*.pdf", loader_cls=PyPDFLoader)
    documents = loader.load()
    
    # Chunking
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=512,
        chunk_overlap=64,
        length_function=len,
    )
    chunks = splitter.split_documents(documents)
    
    # Adicionar metadados úteis
    for i, chunk in enumerate(chunks):
        chunk.metadata["chunk_id"] = i
        chunk.metadata["indexed_at"] = "2025-01-15"
    
    # Indexar
    vectorstore = Qdrant.from_documents(
        documents=chunks,
        embedding=OpenAIEmbeddings(),
        url="http://localhost:6333",
        collection_name=collection_name,
    )
    
    print(f"Indexed {len(chunks)} chunks from {len(documents)} documents")
    return vectorstore
```

```python
# FASE 2 — Pipeline de recuperação e geração
from langchain_openai import ChatOpenAI
from langchain.prompts import PromptTemplate
from langchain.chains import RetrievalQA

def create_rag_chain(vectorstore):
    # Template de prompt com instruções explícitas de grounding
    prompt_template = """Você é um assistente que responde perguntas com base nos documentos fornecidos.
    
Use APENAS as informações dos documentos abaixo para responder.
Se a resposta não estiver nos documentos, diga "Não encontrei essa informação nos documentos disponíveis."
Não invente informações.

Documentos recuperados:
{context}

Pergunta: {question}

Resposta fundamentada:"""
    
    PROMPT = PromptTemplate(
        template=prompt_template,
        input_variables=["context", "question"]
    )
    
    retriever = vectorstore.as_retriever(
        search_type="similarity",
        search_kwargs={
            "k": 5,  # recuperar 5 chunks
        }
    )
    
    chain = RetrievalQA.from_chain_type(
        llm=ChatOpenAI(model="gpt-4o-mini", temperature=0),
        chain_type="stuff",  # coloca todos os chunks juntos no contexto
        retriever=retriever,
        return_source_documents=True,
        chain_type_kwargs={"prompt": PROMPT}
    )
    
    return chain

# Uso
chain = create_rag_chain(vectorstore)
result = chain.invoke({"query": "Quais são os direitos do titular sob a LGPD?"})

print("Resposta:", result["result"])
print("\nFontes:")
for doc in result["source_documents"]:
    print(f"  - {doc.metadata['source']}, pág. {doc.metadata.get('page', 'N/A')}")
```

### Armadilhas comuns e como evitar

**Armadilha 1: Usar modelo de embedding diferente na indexação e na query**
```python
# ERRADO: indexou com ada-002, busca com text-embedding-3-large
vectorstore = Chroma.from_documents(docs, OpenAIEmbeddings(model="text-embedding-ada-002"))
results = vectorstore.similarity_search(query, embedding=OpenAIEmbeddings(model="text-embedding-3-large"))
# Os espaços vetoriais são incompatíveis — resultados serão lixo

# CORRETO: mesmo modelo nos dois lugares, sempre
EMBEDDING_MODEL = OpenAIEmbeddings(model="text-embedding-3-small")
```

**Armadilha 2: Chunks muito grandes — o modelo ignora o meio**
Janela de contexto grande não é suficiente. Chunks de 2000+ tokens frequentemente levam ao modelo não usar o conteúdo no meio.

**Armadilha 3: Prompt que não força grounding**
Sem instrução explícita "use apenas os documentos", o modelo mistura conhecimento interno com os documentos recuperados — e você perde rastreabilidade.

**Armadilha 4: Não avaliar a qualidade de recuperação**
O LLM pode gerar ótimas respostas com base em documentos errados (irrelevantes). Avaliar retrieval separado da geração é essencial.

### Variações do chain_type

O LangChain oferece 4 chain types para combinar múltiplos chunks:

| Chain type | Como funciona | Quando usar |
|---|---|---|
| **stuff** | Coloca todos os chunks no prompt de uma vez | Quando os chunks cabem no contexto |
| **map_reduce** | Processa cada chunk separado, depois reduz | Muitos chunks, contexto limitado |
| **refine** | Itera sobre chunks, refinando a resposta progressivamente | Quando a síntese progressiva importa |
| **map_rerank** | Gera resposta para cada chunk, ranqueia por confiança | Quando quer a melhor resposta individual |

### Reranking — melhorar recuperação antes da geração

A busca vetorial retorna os K chunks mais próximos matematicamente. Mas similaridade semântica nem sempre é a mesma coisa que relevância para a query específica. Reranking resolve isso:

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

# Modelo de reranking (compara query + chunk diretamente, mais preciso)
model = HuggingFaceCrossEncoder(model_name="BAAI/bge-reranker-base")
compressor = CrossEncoderReranker(model=model, top_n=3)

# Pipeline: busca 10, reranqueia, mantém 3 melhores
retriever = vectorstore.as_retriever(search_kwargs={"k": 10})
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=retriever
)
```

---

## Aplicação Prática

**Decisão de arquitetura para produção:**

```
Volume de docs < 10K páginas  → Chroma local ou pgvector
Volume de docs 10K-1M páginas → Qdrant self-hosted ou Weaviate
Volume de docs > 1M páginas   → Pinecone ou solução gerenciada

Frequência de updates:
Estático (meses)              → IVF index (FAISS)
Dinâmico (dias/horas)         → HNSW (Qdrant, Weaviate)
Tempo real                    → Streaming ingestion com HNSW

Tipo de busca necessária:
Semântica pura                → Vetorial simples
Semântica + keywords exatas   → Híbrida (vetorial + BM25)
Filtros por metadados         → Vetorial + filtered search
```

---

## Conexões

- [03-02 — Bancos Vetoriais](03-02-Bancos-Vetoriais-na-Pratica.md): onde ficam os chunks indexados
- [03-04 — RAG como Ferramenta](03-04-RAG-como-Ferramenta-de-Agente.md): integrar RAG em um agente que decide quando buscar
- [04-01 — O que é um Agente](../Modulo-04-Agentes-de-IA/04-01-O-que-e-um-Agente.md): RAG é uma das ferramentas do agente

---

## Recursos Complementares

- **LangChain RAG from scratch (YouTube, LangChain)**: série de 10 vídeos curtos, cada um com implementação
- **RAGAS — framework de avaliação de RAG**: [github.com/explodinggradients/ragas](https://github.com/explodinggradients/ragas)
- **"Building RAG-based LLM Applications" (Anyscale blog)**: produção-grade, com benchmarks reais
- **Alura — Formação Agentes IA, semana 2-3**: RAG com LangChain e Qdrant em pt-BR

---

## Auto-reflexão

1. Por que é crítico usar o mesmo modelo de embedding na indexação e na query?
2. Se um sistema RAG está dando respostas incorretas, qual é o primeiro passo de debug — o retrieval ou a geração? Por quê?
3. Um pipeline RAG usando "stuff" chain com 10 chunks de 500 tokens cada = 5000 tokens no contexto. Por que isso pode ser problemático mesmo com janela de 100K tokens?
4. Você tem uma base de 50.000 documentos que é atualizada diariamente. Proponha uma arquitetura de indexação incremental (que não re-indexa tudo todo dia).

---

*[03-03] | Anterior: [03-02](03-02-Bancos-Vetoriais-na-Pratica.md) | Próximo: [03-04](03-04-RAG-como-Ferramenta-de-Agente.md)*
*Referências: LangChain docs; RAGAS framework; Qdrant examples*
