# [04-03] LangChain na Prática — O Framework que Cola Tudo

## Por que isso importa?

LangChain é o framework mais usado para construir sistemas com LLMs e agentes. Ele não é perfeito — tem críticos legítimos — mas dominar seu modelo mental (chains, runnable, LCEL) elimina 80% do boilerplate de projetos de IA.

---

## Conceito Central

**LangChain é um framework de composição: você encadeia componentes (LLMs, prompts, retrievers, parsers, tools) em pipelines chamados "chains". O padrão moderno é LCEL (LangChain Expression Language), que usa o operador `|` para compor componentes como pipes Unix.**

---

## Aprofundamento

### A evolução do LangChain: chains → LCEL

**Versão antiga (ainda documentada, mas legada):**
```python
from langchain.chains import LLMChain
chain = LLMChain(llm=llm, prompt=prompt)
result = chain.run("minha pergunta")
```

**Versão moderna (LCEL):**
```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

prompt = ChatPromptTemplate.from_template("Responda em pt-BR: {question}")
llm = ChatOpenAI(model="gpt-4o-mini")

# O operador | cria uma chain
chain = prompt | llm

result = chain.invoke({"question": "O que é RAG?"})
```

LCEL tem streaming nativo, batch nativo, async nativo, e visualização de grafo. Use LCEL para qualquer coisa nova.

### Os componentes principais

**Prompts**
```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages([
    ("system", "Você é um assistente especializado em {domain}."),
    MessagesPlaceholder("chat_history"),  # histórico de conversa
    ("human", "{input}"),
])
```

**Models (LLMs e Chat Models)**
```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_community.llms import Ollama  # modelos locais

llm = ChatOpenAI(model="gpt-4o", temperature=0.2, max_tokens=2000)
# ou
llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
# ou
llm = Ollama(model="llama3.2")  # local, sem custo de API
```

**Output Parsers**
```python
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.pydantic_v1 import BaseModel, Field

class AnalysisResult(BaseModel):
    sentiment: str = Field(description="positivo, negativo ou neutro")
    score: float = Field(description="score de 0 a 1")
    justification: str = Field(description="justificativa da análise")

parser = JsonOutputParser(pydantic_object=AnalysisResult)

chain = prompt | llm | parser
result = chain.invoke({"text": "O produto chegou rápido e funcionou perfeitamente!"})
# result = {"sentiment": "positivo", "score": 0.95, "justification": "..."}
```

**Retrievers (para RAG)**
```python
# Qualquer vectorstore pode virar retriever
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# RAG chain completa em LCEL
from langchain_core.runnables import RunnablePassthrough

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("O que é titular de dados?")
```

### Memória de conversa

```python
from langchain_community.chat_message_histories import ChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# Armazenar histórico por session_id
store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = ChatMessageHistory()
    return store[session_id]

# Adicionar memória à chain
chain_with_memory = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="chat_history",
)

# Conversa com contexto
response1 = chain_with_memory.invoke(
    {"input": "Meu nome é Marcus."},
    config={"configurable": {"session_id": "user_123"}}
)

response2 = chain_with_memory.invoke(
    {"input": "Qual é o meu nome?"},  # o modelo lembra
    config={"configurable": {"session_id": "user_123"}}
)
# Output: "Seu nome é Marcus."
```

### Agentes com LCEL

```python
from langchain.agents import create_openai_tools_agent, AgentExecutor
from langchain import hub

# Tools definidas com @tool decorator
@tool
def search_web(query: str) -> str:
    """Busca informações atuais na internet."""
    # implementação com DuckDuckGo ou Tavily
    ...

@tool
def run_python(code: str) -> str:
    """Executa código Python e retorna o resultado. Apenas para cálculos e análise de dados."""
    ...

tools = [search_web, run_python, rag_tool]  # RAG como uma das tools

# Prompt de agente (baixar do hub ou customizar)
agent_prompt = hub.pull("hwchase17/openai-tools-agent")

# Criar agente
agent = create_openai_tools_agent(llm, tools, agent_prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True, max_iterations=15)
```

### LCEL: streaming, batch e async

Uma das grandes vantagens do LCEL é que qualquer chain tem essas capacidades automaticamente:

```python
# Streaming (token por token, útil para UX responsiva)
for chunk in chain.stream({"question": "Explique RAG em 3 parágrafos"}):
    print(chunk.content, end="", flush=True)

# Batch (múltiplos inputs em paralelo)
questions = ["O que é RAG?", "O que é ReAct?", "O que é LangGraph?"]
results = chain.batch([{"question": q} for q in questions])

# Async (útil em aplicações web — FastAPI, etc.)
import asyncio
result = await chain.ainvoke({"question": "O que é embeddings?"})
```

### Limitações do LangChain (críticas legítimas)

**Abstração excessiva:** às vezes é mais simples chamar a API diretamente do que entender 5 camadas de abstração do LangChain para fazer a mesma coisa.

**Documentação fragmentada:** a transição de chains antigas para LCEL criou inconsistência na documentação.

**Versionamento agressivo:** LangChain quebra compatibilidade com frequência. Fixe versões em `requirements.txt`.

**Quando ir além do LangChain:**
- Para fluxos com estado complexo: LangGraph (M05) — feito pelos mesmos criadores
- Para máxima performance/controle: chamar APIs diretamente (anthropic, openai packages)
- Para deployment: LangServe ou FastAPI com endpoints próprios

---

## Aplicação Prática

**Template de projeto LangChain mínimo e funcional:**

```
my_agent/
├── requirements.txt
├── .env
├── config.py          # constantes, model names
├── tools/
│   ├── __init__.py
│   ├── search.py      # web search tool
│   └── knowledge.py   # RAG tool
├── chains/
│   ├── __init__.py
│   └── main_chain.py  # chain principal LCEL
├── agent.py           # montagem do agente + executor
└── main.py            # entry point
```

```
# requirements.txt (fixe versões!)
langchain==0.3.7
langchain-openai==0.2.9
langchain-community==0.3.7
langchain-core==0.3.19
openai==1.55.0
python-dotenv==1.0.0
qdrant-client==1.12.0
```

---

## Conexões

- [04-02 — ReAct e Toolcalling](04-02-ReAct-e-Toolcalling.md): o padrão que LangChain implementa
- [03-03 — Pipelines RAG](../Modulo-03-RAG-Memoria-Contextual/03-03-Pipelines-RAG-do-Zero.md): RAG com LangChain LCEL
- [05-01 — LangGraph](../Modulo-05-Orquestracao-Fluxos/05-01-Grafos-de-Decisao-LangGraph.md): extensão do LangChain para fluxos complexos

---

## Recursos Complementares

- **LangChain Python docs**: [python.langchain.com](https://python.langchain.com) — documentação oficial com exemplos LCEL
- **LangChain Expression Language (LCEL) guide**: [python.langchain.com/docs/expression_language](https://python.langchain.com/docs/expression_language)
- **LangChain templates**: [github.com/langchain-ai/langchain/tree/master/templates](https://github.com/langchain-ai/langchain/tree/master/templates) — projetos prontos
- **Alura — "LangChain: construindo aplicações com LLMs"** (8h, pt-BR)

---

## Auto-reflexão

1. Qual é a vantagem do operador `|` em LCEL sobre a instanciação de `LLMChain` no estilo antigo?
2. Por que fixar versões em `requirements.txt` é especialmente importante em projetos LangChain?
3. Você está construindo um chatbot com histórico de conversa para 10.000 usuários simultâneos. O que muda na implementação de memória quando você sai de "store em memória local" para produção?
4. Para que tipo de problema LangChain sozinho não é suficiente, e LangGraph seria necessário?

---

*[04-03] | Anterior: [04-02](04-02-ReAct-e-Toolcalling.md) | Próximo: [04-04](04-04-Memoria-e-Persistencia.md)*
*Referências: LangChain docs v0.3; LCEL guide; Alura LangChain course*
