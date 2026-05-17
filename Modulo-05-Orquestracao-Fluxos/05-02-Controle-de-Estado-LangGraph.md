# [05-02] Controle de Estado — O Coração do LangGraph

## Por que isso importa?

Em LangGraph, o estado é a única fonte de verdade do workflow. Entender como o estado é definido, atualizado, compartilhado e persistido é o que diferencia sistemas frágeis de sistemas robustos.

---

## Conceito Central

**O estado no LangGraph é um TypedDict que flui entre todos os nós do grafo. Cada nó recebe o estado atual, processa, e retorna apenas as chaves que modificou. A "reducers" determina como chaves com múltiplos valores são combinadas — especialmente crítico para listas de mensagens.**

---

## Aprofundamento

### Definindo estado com Reducers

O reducer define o que acontece quando dois nós retornam o mesmo campo:

```python
from typing import TypedDict, Annotated, Sequence
from langgraph.graph.message import add_messages
from langchain_core.messages import BaseMessage
import operator

class SimpleState(TypedDict):
    # Sem reducer: o novo valor sobrescreve o anterior
    counter: int
    current_node: str
    
    # Com reducer add_messages: novas mensagens são ADICIONADAS à lista
    messages: Annotated[list[BaseMessage], add_messages]
    
    # Com reducer operator.add: novos itens são concatenados
    processed_items: Annotated[list[str], operator.add]
```

**Por que isso importa?**

Sem reducer em `messages`, cada nó que retorna `messages` sobrescreveria o histórico inteiro. Com `add_messages`, cada nó pode adicionar sua mensagem ao histórico existente.

### Estado complexo para workflows avançados

```python
from typing import TypedDict, Annotated, Optional, List
from datetime import datetime

class ResearchWorkflowState(TypedDict):
    # Input
    user_query: str
    user_id: str
    session_id: str
    
    # Contexto coletado
    messages: Annotated[list, add_messages]
    web_results: List[str]
    rag_results: List[str]
    
    # Controle de fluxo
    current_phase: str  # "research" | "synthesis" | "review" | "done"
    iteration: int
    max_iterations: int
    
    # Qualidade
    quality_score: float
    quality_feedback: str
    human_approved: Optional[bool]
    
    # Output
    draft_answer: str
    final_answer: str
    sources_cited: List[str]
    
    # Metadata
    started_at: str
    completed_at: Optional[str]
    total_tokens_used: int
```

### Nós que modificam apenas parte do estado

```python
def web_search_node(state: ResearchWorkflowState) -> dict:
    """Busca na web — retorna apenas o que mudou."""
    query = state["user_query"]
    results = search_web(query)
    
    # Retorna APENAS os campos modificados
    # O resto do estado é preservado automaticamente
    return {
        "web_results": results,
        "current_phase": "researched",
        "messages": [AIMessage(content=f"Encontrei {len(results)} resultados na web.")]
    }
    # NÃO precisa retornar user_query, user_id, etc.

def rag_search_node(state: ResearchWorkflowState) -> dict:
    """Busca na base de conhecimento interna."""
    query = state["user_query"]
    results = knowledge_base.search(query)
    
    return {
        "rag_results": results,
        "total_tokens_used": state["total_tokens_used"] + estimate_tokens(results)
    }
```

### Checkpointing — persistência entre execuções

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# Desenvolvimento: memória (não persiste ao reiniciar)
memory_checkpointer = MemorySaver()

# Produção simples: SQLite
sqlite_checkpointer = SqliteSaver.from_conn_string("./checkpoints.db")

# Produção escalável: PostgreSQL
postgres_checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/langgraph_db"
)

# Compilar com checkpoint
app = workflow.compile(checkpointer=postgres_checkpointer)

# Executar com thread_id — cada thread é uma sessão independente
config = {"configurable": {"thread_id": f"user_{user_id}_session_{session_id}"}}
result = app.invoke(initial_state, config=config)

# Recuperar estado de uma execução anterior
past_state = app.get_state(config)
print(past_state.values)  # o estado salvo

# Ver histórico de todos os checkpoints
for checkpoint in app.get_state_history(config):
    print(checkpoint.metadata, checkpoint.values["current_phase"])
```

### Subgrafos — componentes modulares

Você pode encapsular subworkflows em subgrafos:

```python
# Subgrafo de pesquisa
research_graph = StateGraph(ResearchSubState)
research_graph.add_node("web_search", web_search_node)
research_graph.add_node("rag_search", rag_search_node)
research_graph.set_entry_point("web_search")
research_graph.add_edge("web_search", "rag_search")
research_graph.add_edge("rag_search", END)
research_app = research_graph.compile()

# Grafo principal que usa o subgrafo como um nó
main_graph = StateGraph(ResearchWorkflowState)
main_graph.add_node("research", research_app)  # subgrafo como nó
main_graph.add_node("synthesize", synthesize_node)
main_graph.add_node("review", review_node)

main_graph.set_entry_point("research")
main_graph.add_edge("research", "synthesize")
main_graph.add_conditional_edges("review", route_after_review, {...})
```

### Streaming de estado (observabilidade)

```python
# Executar com streaming de cada etapa
async for event in app.astream_events(initial_state, config=config, version="v1"):
    if event["event"] == "on_chain_start":
        print(f"Iniciando: {event['name']}")
    elif event["event"] == "on_chain_end":
        print(f"Concluído: {event['name']}")
        print(f"Output: {event['data']['output']}")
    elif event["event"] == "on_tool_start":
        print(f"Tool: {event['name']}({event['data']['input']})")
```

---

## Aplicação Prática

**Padrão de estado para agentes de produção:**

```python
class ProductionAgentState(TypedDict):
    # — SEMPRE incluir estes campos —
    messages: Annotated[list, add_messages]  # histórico
    session_id: str                          # rastreamento
    
    # — Controle de execução —
    step: int                                # contador de passos
    max_steps: int                           # guardrail
    
    # — Resultado —
    output: str
    error: Optional[str]                     # para tratamento de falhas
    
    # — Observabilidade —
    execution_log: Annotated[list, operator.add]
```

---

## Conexões

- [05-01 — Grafos de Decisão](05-01-Grafos-de-Decisao-LangGraph.md): estrutura de grafos que usa este estado
- [05-03 — Padrões de Orquestração](05-03-Padroes-de-Orquestracao.md): padrões que dependem de estado bem definido
- [04-04 — Memória e Persistência](../Modulo-04-Agentes-de-IA/04-04-Memoria-e-Persistencia.md): como estado se relaciona com memória de longo prazo

---

## Recursos Complementares

- **LangGraph State guide**: [langchain-ai.github.io/langgraph/concepts/low_level](https://langchain-ai.github.io/langgraph/concepts/low_level) — referência técnica completa
- **LangGraph Persistence tutorial**: [langchain-ai.github.io/langgraph/tutorials/persistence](https://langchain-ai.github.io/langgraph/tutorials/persistence)
- **LangGraph Subgraphs**: [langchain-ai.github.io/langgraph/how-tos/subgraph](https://langchain-ai.github.io/langgraph/how-tos/subgraph)

---

## Auto-reflexão

1. O que é um "reducer" no contexto do estado LangGraph? Por que `add_messages` é o reducer correto para o histórico de mensagens (vs. substituição simples)?
2. Por que um nó deve retornar apenas os campos que modificou (e não o estado inteiro)?
3. Quais são as implicações de usar `MemorySaver` vs. `PostgresSaver` para uma aplicação com 1000 usuários simultâneos?
4. Como checkpointing permite implementar "pausar e aguardar aprovação humana" em um workflow?

---

*[05-02] | Anterior: [05-01](05-01-Grafos-de-Decisao-LangGraph.md) | Próximo: [05-03](05-03-Padroes-de-Orquestracao.md)*
*Referências: LangGraph docs; "State Management in Agentic Systems" (Anthropic engineering blog)*
