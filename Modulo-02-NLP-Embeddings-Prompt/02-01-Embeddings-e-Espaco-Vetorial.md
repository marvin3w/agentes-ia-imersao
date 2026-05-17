# [02-01] Embeddings e Espaço Vetorial

## Por que isso importa?

Embeddings são a tecnologia central do RAG, da busca semântica e de qualquer sistema que precise "entender" o significado de texto — não apenas comparar palavras iguais. Sem entender embeddings, RAG é magia. Com embeddings, RAG é engenharia.

---

## Conceito Central

**Um embedding é a representação numérica de um texto em um espaço de alta dimensão, onde textos semanticamente similares ficam próximos geometricamente.**

Imagine um mapa onde cada texto ocupa uma posição. "Cachorro" e "cão" ficam perto um do outro. "Cachorro" e "banco financeiro" ficam distantes. "Banco de praça" e "assento de parque" ficam próximos.

Essa geometria captura significado de uma forma que pesquisa por palavras-chave não consegue. Você pode perguntar "onde está o documento sobre felinos domésticos?" e encontrar o documento sobre gatos — mesmo que a palavra "gatos" não apareça.

---

## Aprofundamento

### Como embeddings são gerados

Um modelo de embedding é uma rede neural treinada para converter texto em vetores. O treinamento usa pares de textos similares e diferentes para que o modelo aprenda que textos semelhantes devem produzir vetores próximos.

Modelos de embedding populares:
- **text-embedding-3-small / large** (OpenAI) — 1.536 ou 3.072 dimensões, API paga
- **all-MiniLM-L6-v2** (Sentence Transformers) — 384 dimensões, gratuito, local
- **nomic-embed-text** (Nomic AI) — 768 dimensões, gratuito, roda via Ollama
- **voyage-3** (Voyage AI) — alta qualidade, API paga

**Dimensões:** um vetor de 1.536 dimensões é uma lista de 1.536 números. Mais dimensões geralmente significam mais expressividade, mas mais custo de armazenamento e processamento.

### Similaridade de cosseno — a métrica que importa

A proximidade entre dois vetores é medida pelo **cosseno do ângulo** entre eles (não pela distância euclidiana). Varia de -1 (opostos) a 1 (idênticos).

```
similaridade("gato", "felino") ≈ 0.89   (muito similar)
similaridade("gato", "cachorro") ≈ 0.72  (relacionados)
similaridade("gato", "banco") ≈ 0.12    (pouco relacionados)
similaridade("gato", "economia") ≈ 0.05 (não relacionados)
```

Quando você faz busca semântica ("encontre documentos sobre IA generativa"), o sistema converte sua query em embedding e encontra os documentos cujos embeddings têm maior cosseno com o da query.

### Bancos vetoriais — onde os embeddings vivem

Um banco vetorial é um banco de dados especializado em armazenar e buscar vetores por similaridade de forma eficiente:

| Banco Vetorial | Tipo | Ideal para |
|---|---|---|
| Chroma | Local, em memória/disco | Desenvolvimento, protótipos |
| Pinecone | Cloud, gerenciado | Produção, alta escala |
| Weaviate | Self-hosted ou cloud | Controle total, open-source |
| pgvector | Extensão PostgreSQL | Se já usa Postgres |
| FAISS | Biblioteca local (Meta) | Pesquisa, alto desempenho local |

Para começar: Chroma. Para produção: Pinecone ou pgvector se já tem Postgres.

### Chunking — o problema que ninguém te conta

Você não pode colocar um documento inteiro num embedding. Modelos de embedding têm limite de tokens (geralmente 512-8.192 tokens). Documentos maiores precisam ser divididos em **chunks** (pedaços).

A qualidade do RAG depende muito da estratégia de chunking:

**Chunking fixo:** divide por número fixo de tokens. Simples mas pode cortar no meio de frases.

**Chunking semântico:** divide em parágrafos ou seções semânticas. Mais inteligente, melhor para recuperação.

**Chunking com overlap:** cada chunk compartilha tokens com o anterior. Evita perder contexto nas bordas.

```python
# Exemplo de chunking com overlap
chunk_size = 500    # tokens por chunk
chunk_overlap = 50  # tokens compartilhados com chunk anterior
```

**Regra prática:** chunks menores = mais precisão na recuperação, mas mais "barulho" na geração. Chunks maiores = mais contexto, mas recuperação menos precisa. Teste com 300-500 tokens + 10% de overlap como ponto de partida.

---

## Aplicação Prática

**Pipeline mínimo de embedding + busca:**

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer('all-MiniLM-L6-v2')

# Indexar documentos
docs = ["Gatos são felinos domésticos.", "Bitcoin é uma criptomoeda.", "Python é uma linguagem de programação."]
embeddings = model.encode(docs)

# Buscar
query = "linguagens de programação populares"
query_embedding = model.encode([query])

# Calcular similaridade
similarities = np.dot(embeddings, query_embedding.T).flatten()
best_match_idx = similarities.argmax()
print(f"Resultado: {docs[best_match_idx]} (score: {similarities[best_match_idx]:.2f})")
# → "Python é uma linguagem de programação." (score: 0.74)
```

**Checklist antes de implementar RAG:**

- [ ] Qual modelo de embedding usarei? (custo vs. qualidade vs. privacidade)
- [ ] Qual banco vetorial? (local/cloud, escala esperada)
- [ ] Qual estratégia de chunking? (tamanho e overlap)
- [ ] Como atualizo os embeddings quando documentos mudam?
- [ ] Como evito buscar chunks irrelevantes? (threshold de similaridade)

---

## Conexões

- [01-03 — Transformers e Atenção](../Modulo-01-Fundamentos-Conceituais/01-03-Transformers-e-Atencao.md): Transformers geram os embeddings
- [M03-02 — Bancos Vetoriais na Prática](../Modulo-03-RAG-Memoria-Contextual/03-02-Bancos-Vetoriais-na-Pratica.md): implementação completa
- [M03-03 — Pipelines RAG do Zero](../Modulo-03-RAG-Memoria-Contextual/03-03-Pipelines-RAG-do-Zero.md): onde embeddings se tornam RAG

---

## Recursos Complementares

- **Sentence Transformers** (biblioteca Python, gratuita): [sbert.net](https://www.sbert.net)
- **"What are embeddings?"** — Simon Willison (blog post acessível)
- **Chroma** (banco vetorial local, gratuito): [docs.trychroma.com](https://docs.trychroma.com)

---

## Auto-reflexão

1. Por que busca por palavras-chave falha ao encontrar "felino doméstico" quando a query é "gato"? Como embeddings resolvem isso?
2. Se você dividir um documento de 10.000 tokens em chunks de 500 com overlap de 50, quantos chunks terá? Por que o overlap existe?
3. Qual banco vetorial você escolheria para um protótipo que precisa rodar offline? E para produção com 1 milhão de documentos?
4. Por que é problemático usar o mesmo modelo LLM para gerar embeddings e para responder queries? (dica: especialização vs. generalização)

---

*[02-01] | Anterior: [02-00](02-00-Visao-Geral.md) | Próximo: [02-02](02-02-Engenharia-de-Prompt.md)*
*Referência: CS50 AI (Harvard) — NLP; Sentence Transformers*
