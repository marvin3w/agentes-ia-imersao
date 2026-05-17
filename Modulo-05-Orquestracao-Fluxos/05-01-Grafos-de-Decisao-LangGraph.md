# [05-01] Grafos de Decisão — Por que LangGraph Existe

## Por que isso importa?

AgentExecutor do LangChain funciona para agentes simples (pergunta → tools → resposta). Mas quando você precisa de fluxos condicionais ("se o resultado for X, faça Y, senão faça Z"), loops com condição de parada, e múltiplos agentes com handoffs — o executor simples quebra. LangGraph resolve isso com uma abstração diferente: grafos de estado.

---

## Conceito Central

**LangGraph modela seu sistema de IA como um grafo dirigido: cada nó é um passo de processamento (LLM, ferramenta, função), cada aresta é uma transição entre passos (podendo ser condicional). O estado é passado entre nós e persiste ao longo do fluxo. Isso permite criar workflows sofisticados que um simples loop não suporta.**

---

## Aprofundamento

### O problema que LangGraph resolve

AgentExecutor: "repita: LLM decide tool → executa → observe → até terminar".

Problemas com esse modelo simples:
- Não suporta decisões condicionais ("se qualidade ok → prossegue, senão → refaz")
- Não suporta múltiplos subagentes com handoff explícito
- Estado não é tipado — difícil rastrear o que foi feito
- Difícil de pausar, inspecionar e retomar (Human-in-the-Loop)
- Difícil de criar backtracking (voltar a um passo anterior)

LangGraph resolve tudo isso com um modelo de grafo de estado.

### Conceitos fundamentais

**State — o estado compartilhado**
Um TypedDict que flui entre todos os nós:

```python
from typing import TypedDict, Annotated, List
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # histórico de mensagens
    current_step: str                         # qual passo está sendo executado
    quality_score: float                      # resultado de avaliação
    retry_count: int                          # controle de loops
    final_answer: str                         # resposta final
```

**Nodes — funções que transformam o estado**
```python
def research_node(state: AgentState) -> AgentState:
    """Pesquisa informações sobre o tópico."""
    query = state["messages"][-1].content
    results = web_search(query)
    return {"messages": [AIMessage(content=results)], "current_step": "researched"}

def quality_check_node(state: AgentState) -> AgentState:
    """Avalia a qualidade da pesquisa."""
    content = state["messages"][-1].content
    score = evaluate_quality(content)  # LLM ou heurística
    return {"quality_score": score, "current_step": "quality_checked"}

def answer_node(state: AgentState) -> AgentState:
    """Gera a resposta final."""
    context = state["messages"]
    answer = llm.invoke(context)
    return {"final_answer": answer.content, "current_step": "answered"}
```

**Edges — transições entre nós (podem ser condicionais)**
```python
def route_after_quality(state: AgentState) -> str:
    """Decide o próximo passo baseado na qualidade."""
    if state["quality_score"] >= 0.8:
        return "answer"  # qualidade ok, ir para resposta
    elif state["retry_count"] < 3:
        return "research"  # refazer pesquisa
    else:
        return "answer"  # desistir, responder com o que tem
```

**Graph — montagem e compilação**
```python
from langgraph.graph import StateGraph, END

workflow = StateGraph(AgentState)

# Adicionar nós
workflow.add_node("research", research_node)
workflow.add_node("quality_check", quality_check_node)
workflow.add_node("answer", answer_node)

# Definir ponto de entrada
workflow.set_entry_point("research")

# Arestas fixas
workflow.add_edge("research", "quality_check")

# Aresta condicional (o nó decide para onde ir)
workflow.add_conditional_edges(
    "quality_check",      # nó de origem
    route_after_quality,  # função que retorna o próximo nó
    {
        "answer": "answer",
        "research": "research",
    }
)

workflow.add_edge("answer", END)

# Compilar
app = workflow.compile()

# Executar
result = app.invoke({
    "messages": [HumanMessage(content="Quais são as melhores práticas de RAG em 2025?")],
    "current_step": "",
    "quality_score": 0.0,
    "retry_count": 0,
    "final_answer": ""
})
```

### Visualizando o grafo

```python
# Gerar diagrama ASCII do grafo
print(app.get_graph().draw_ascii())

# Ou em PNG (requer graphviz)
from IPython.display import Image
Image(app.get_graph().draw_mermaid_png())
```

### Human-in-the-Loop com Checkpoints

LangGraph tem suporte nativo para pausar e aguardar input humano:

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import interrupt_after

# Adicionar persistência de estado
memory = MemorySaver()
app = workflow.compile(
    checkpointer=memory,
    interrupt_after=["quality_check"]  # pausa após quality_check para revisão humana
)

# Thread de execução
config = {"configurable": {"thread_id": "task_001"}}

# Iniciar (vai pausar em quality_check)
state = app.invoke(initial_state, config=config)

# Agente pausa, humano revisa state["quality_score"]
# O operador pode modificar o estado antes de retomar

# Retomar de onde parou
state = app.invoke(None, config=config)  # None = retomar do checkpoint
```

### Quando LangGraph vs. AgentExecutor

| Situação | Use AgentExecutor | Use LangGraph |
|---|---|---|
| Agente simples com tools | ✓ | Overkill |
| Fluxo com condicionais | | ✓ |
| Múltiplos agentes/subagentes | | ✓ |
| Human-in-the-Loop obrigatório | | ✓ |
| Estado tipado e auditável | | ✓ |
| Loops com condição de parada | | ✓ |
| Prototipagem rápida | ✓ | Verboso demais |

---

## Aplicação Prática

**Padrão ReAct implementado no LangGraph:**

```python
# O AgentExecutor é, em essência, um grafo LangGraph simples:
# agent_node → tools_node → agent_node → ... → END

# LangGraph permite fazer o mesmo com mais controle:
from langgraph.prebuilt import create_react_agent

agent_app = create_react_agent(
    model=llm,
    tools=tools,
    state_modifier="Você é um assistente útil."
)

# Idêntico ao AgentExecutor, mas sobre LangGraph — extensível
```

---

## Conexões

- [04-01 — O que é um Agente](../Modulo-04-Agentes-de-IA/04-01-O-que-e-um-Agente.md): agentes simples com AgentExecutor
- [04-02 — ReAct e Toolcalling](../Modulo-04-Agentes-de-IA/04-02-ReAct-e-Toolcalling.md): o padrão que LangGraph implementa com mais controle
- [05-02 — Controle de Estado](05-02-Controle-de-Estado-LangGraph.md): gerenciar estado complexo em LangGraph
- [06-01 — Arquitetura Multiagente](../Modulo-06-Sistemas-Multiagente/06-01-Arquitetura-Multiagente.md): LangGraph como base para sistemas multiagente

---

## Recursos Complementares

- **LangGraph documentation**: [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph) — documentação oficial
- **LangGraph quickstart**: [langchain-ai.github.io/langgraph/tutorials/introduction](https://langchain-ai.github.io/langgraph/tutorials/introduction)
- **"LangGraph: A Gentle Introduction"** — blog LangChain, com animações do fluxo
- **Alura — "Orquestração com LangGraph"**: módulo 4 da Formação Agentes IA (pt-BR)

---

## Auto-reflexão

1. Por que um loop simples (ReAct/AgentExecutor) não é suficiente para implementar "se a qualidade não for boa o suficiente, refaça a pesquisa"?
2. O que diferencia uma "aresta condicional" de uma "aresta fixa" em LangGraph?
3. Qual é a vantagem de tipagem forte no estado (TypedDict) vs. dicionário genérico?
4. Em que contexto Human-in-the-Loop é essencial? Dê um exemplo de processo onde você não deixaria o agente prosseguir sem aprovação humana.

---

## Próximo Passo

### [→ 05-02 — Controle de Estado LangGraph](05-02-Controle-de-Estado-LangGraph.md)


---

*[← M05 — Orquestração e Fluxos](05-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
