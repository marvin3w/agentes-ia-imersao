# [04-04] Memória e Persistência — Como Agentes Lembram

## Por que isso importa?

Um agente sem memória é um robô que começa do zero em cada conversa. Memória é o que transforma interações isoladas em relacionamentos duradouros — e é também um dos pontos mais complexos de implementar corretamente em produção.

---

## Conceito Central

**Memória de agente não é um único sistema — é uma taxonomia com quatro tipos (working, episódica, semântica, procedural) que operam em escalas de tempo diferentes. Implementar só o tipo certo para o problema certo economiza custo e complexidade.**

---

## Aprofundamento

### Os quatro tipos de memória (inspirados na psicologia cognitiva)

**1. Memória de Trabalho (Working Memory)**
- **O que é:** a janela de contexto atual — o que o agente "sabe" agora
- **Onde vive:** os tokens na janela de contexto do LLM
- **Escopo:** a conversa atual
- **Limite:** tamanho da janela de contexto (32K, 200K tokens dependendo do modelo)
- **Implementação:** simplesmente o histórico de mensagens passado ao LLM

```python
messages = [
    SystemMessage(content="Você é um assistente..."),
    HumanMessage(content="Qual é a capital do Brasil?"),
    AIMessage(content="A capital do Brasil é Brasília."),
    HumanMessage(content="E a do Chile?"),  # contexto preservado
]
```

**2. Memória Episódica**
- **O que é:** histórico de conversas anteriores, armazenado e recuperável
- **Onde vive:** banco de dados (SQL, Redis, banco vetorial)
- **Escopo:** histórico do usuário específico ao longo do tempo
- **Caso de uso:** chatbots que "lembram" de conversas de semanas atrás

```python
# Salvar episódio
memory_db.save({
    "user_id": "marcus_123",
    "session_id": "sess_2025-01-15",
    "summary": "Usuário perguntou sobre LGPD, está implementando conformidade",
    "timestamp": "2025-01-15T14:30:00Z",
    "key_facts": ["empresa com 50 funcionários", "setor financeiro", "prazo: Q1 2025"]
})

# Recuperar contexto relevante
past_context = memory_db.search(
    user_id="marcus_123",
    query="conformidade LGPD",
    k=3  # 3 episódios mais relevantes
)
```

**3. Memória Semântica**
- **O que é:** conhecimento geral, fatos, conceitos — o que o modelo "sabe"
- **Onde vive:** nos pesos do LLM (treinamento) + bancos vetoriais (RAG)
- **Escopo:** global, compartilhado entre todos os usuários
- **Implementação:** RAG (M03) é a extensão da memória semântica para conhecimento atualizado

**4. Memória Procedural**
- **O que é:** como realizar tarefas — skills, workflows, comportamentos
- **Onde vive:** no system prompt + fine-tuning + exemplos few-shot
- **Implementação:** rules e skills (M08) são memória procedural externalizada

### Estratégias de compressão de memória

O problema: conversas longas excedem a janela de contexto. Soluções:

**Buffer Window Memory**
Mantém apenas as últimas N mensagens:
```python
from langchain.memory import ConversationBufferWindowMemory
memory = ConversationBufferWindowMemory(k=10)  # últimas 10 mensagens
```
Simples, barato. Perde contexto antigo completamente.

**Summary Memory**
Usa um LLM para comprimir o histórico em um resumo:
```python
from langchain.memory import ConversationSummaryMemory
memory = ConversationSummaryMemory(llm=llm)
# Automaticamente resume: "Usuário perguntou X, eu respondi Y, então..."
```
Preserva o essencial, mas adiciona latência e custo de LLM para comprimir.

**Summary Buffer Memory (híbrida)**
Mantém as últimas N mensagens completas + resumo do que veio antes:
```python
from langchain.memory import ConversationSummaryBufferMemory
memory = ConversationSummaryBufferMemory(
    llm=llm,
    max_token_limit=2000,  # acima desse limite, comprime
)
```
Melhor trade-off na maioria dos casos.

**Entity Memory**
Extrai e armazena entidades relevantes mencionadas na conversa:
```python
from langchain.memory import ConversationEntityMemory
memory = ConversationEntityMemory(llm=llm)
# Automaticamente rastreia: "Marcus → Product Manager, usa Cursor, trabalha na Gran"
```

### Persistência — sobreviver a reinicializações

Memória em variável Python se perde quando o processo reinicia. Para produção:

```python
# Redis (in-memory, rápido, com TTL)
from langchain_community.chat_message_histories import RedisChatMessageHistory

history = RedisChatMessageHistory(
    session_id="user_123_session_456",
    url="redis://localhost:6379",
    ttl=86400  # expirar após 24h
)

# PostgreSQL (persistente, transacional)
from langchain_community.chat_message_histories import PostgresChatMessageHistory

history = PostgresChatMessageHistory(
    session_id="user_123",
    connection_string="postgresql://user:pass@localhost/mydb"
)

# MongoDB (flexível, bom para histórico de documentos longos)
from langchain_community.chat_message_histories import MongoDBChatMessageHistory

history = MongoDBChatMessageHistory(
    session_id="user_123",
    connection_string="mongodb://localhost:27017",
    database_name="agent_memory",
    collection_name="chat_history"
)
```

### Memória episódica com recuperação semântica

O mais sofisticado: armazenar episódios como embeddings para recuperar por relevância:

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma

class EpisodicMemory:
    def __init__(self):
        self.vectorstore = Chroma(embedding_function=OpenAIEmbeddings())
    
    def save_episode(self, content: str, metadata: dict):
        """Salva episódio com timestamp e metadados."""
        self.vectorstore.add_texts(
            texts=[content],
            metadatas=[{**metadata, "timestamp": datetime.now().isoformat()}]
        )
    
    def recall(self, query: str, k: int = 3, user_id: str = None):
        """Recupera episódios mais relevantes para o contexto atual."""
        filter_dict = {"user_id": user_id} if user_id else None
        return self.vectorstore.similarity_search(
            query, k=k, filter=filter_dict
        )

# Uso em um agente
memory = EpisodicMemory()

# Após cada conversa, salvar um resumo
memory.save_episode(
    content="Usuário implementou conformidade LGPD, principal dúvida foi sobre prazo para resposta a titulares (15 dias).",
    metadata={"user_id": "marcus_123", "topic": "LGPD", "session": "sess_001"}
)

# Na próxima conversa sobre LGPD, o agente recupera este episódio
past_context = memory.recall("prazos LGPD", user_id="marcus_123")
```

### O problema do "long tail" e Mem0

Gerenciar memória de longo prazo é um problema não trivial. A biblioteca **Mem0** (ex-EmbedChain) é especializada nisso:

```python
from mem0 import Memory

m = Memory()

# Salvar memórias explicitamente ou deixar o sistema extrair
m.add("O usuário prefere respostas em português e diretas ao ponto", user_id="marcus")
m.add("Contexto: trabalha em uma EdTech chamada Gran Cursos Online", user_id="marcus")

# Recuperar contexto relevante automaticamente
memories = m.search("preferências de comunicação", user_id="marcus")
```

---

## Aplicação Prática

**Decisão de arquitetura de memória:**

```
Tipo de aplicação       → Tipo de memória recomendado

Chatbot simples         → Buffer Window (k=10-20 mensagens)
Assistente pessoal      → Summary Buffer + Episódica em Redis
Sistema de suporte      → Episódica em Postgres + Semântica com RAG
Agente de longo prazo   → Todos os 4 tipos combinados
Pipeline batch          → Sem memória (cada execução independente)
```

---

## Conexões

- [03-04 — RAG como Ferramenta](../Modulo-03-RAG-Memoria-Contextual/03-04-RAG-como-Ferramenta-de-Agente.md): memória semântica via RAG
- [04-03 — LangChain na Prática](04-03-LangChain-na-Pratica.md): implementação de memória com LangChain
- [05-02 — Controle de Estado](../Modulo-05-Orquestracao-Fluxos/05-02-Controle-de-Estado-LangGraph.md): estado em grafos LangGraph (memória estruturada para workflows)

---

## Recursos Complementares

- **Mem0 — biblioteca de memória para LLMs**: [mem0.ai](https://mem0.ai) e [github.com/mem0ai/mem0](https://github.com/mem0ai/mem0)
- **LangChain Memory guide**: [python.langchain.com/docs/modules/memory](https://python.langchain.com/docs/modules/memory)
- **"Building Production-Ready Conversational AI"** — blog série no Towards Data Science
- **Alura — "Agentes com Memória e Persistência"** — módulo específico na formação

---

## Auto-reflexão

1. Por que memória em variável Python não serve para produção? Quais são as implicações quando o processo reinicia?
2. Uma conversa de suporte tem 80 mensagens. Qual estratégia de compressão você usaria e por quê?
3. Você quer que um assistente "lembre" de todas as conversas de um usuário nos últimos 6 meses (potencialmente mil conversas). Como você implementa recuperação eficiente?
4. Qual é o risco de privacidade em sistemas de memória episódica compartilhados? Como mitigar em um produto B2C?

---

*[04-04] | Anterior: [04-03](04-03-LangChain-na-Pratica.md) | Próximo: [M05-00](../Modulo-05-Orquestracao-Fluxos/05-00-Visao-Geral.md)*
*Referências: LangChain memory docs; Mem0 library; "Cognitive Architectures for LLM Agents" (survey)*
