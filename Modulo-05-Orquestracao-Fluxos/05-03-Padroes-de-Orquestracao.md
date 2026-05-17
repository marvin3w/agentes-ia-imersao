# [05-03] Padrões de Orquestração — Os Blueprints Que Todo Agente Usa

## Por que isso importa?

Assim como design patterns em software (Strategy, Observer, Factory) são soluções reutilizáveis para problemas recorrentes, existem padrões consolidados de orquestração de agentes. Conhecê-los evita reinventar a roda e permite comunicar arquiteturas de forma precisa.

---

## Conceito Central

**Orquestração é o meta-nível de agentes: não o que cada agente faz, mas como múltiplos passos, agentes e ferramentas são coordenados para atingir um objetivo. Seis padrões cobrem a maioria dos sistemas de IA em produção.**

---

## Aprofundamento

### Padrão 1 — Chain (Sequência Linear)

O mais simples: A → B → C. Cada passo recebe o output do anterior.

```
Query → [Traduzir] → [Pesquisar] → [Sumarizar] → Resposta
```

```python
# Em LCEL
chain = translate_prompt | llm | search_tool | summarize_prompt | llm | StrOutputParser()
```

**Quando usar:** pipeline de processamento onde a ordem importa e não há condicionais.
**Risco:** erro em qualquer etapa propaga para as seguintes.

### Padrão 2 — Router (Roteador)

Um classificador decide qual caminho seguir:

```
Query → [Classificador] → Se "técnico": → [Agente Técnico]
                        → Se "financeiro": → [Agente Financeiro]
                        → Se "outro": → [Agente Geral]
```

```python
def route_query(state: State) -> str:
    classification = llm.invoke(f"Classifique a query: {state['query']}")
    if "técnico" in classification.content:
        return "technical_agent"
    elif "financeiro" in classification.content:
        return "financial_agent"
    return "general_agent"

workflow.add_conditional_edges("classifier", route_query, {
    "technical_agent": "technical_agent",
    "financial_agent": "financial_agent",
    "general_agent": "general_agent",
})
```

**Quando usar:** quando diferentes tipos de input requerem processamento especializado.
**Risco:** classificador incorreto envia para o caminho errado.

### Padrão 3 — Parallelization (Execução Paralela)

Múltiplos agentes trabalham simultaneamente, outputs são agregados:

```
Query → [Pesquisa Web] ─┐
      → [Busca RAG]   ─→ [Aggregator] → Resposta final
      → [Análise SQL] ─┘
```

```python
from langgraph.graph import StateGraph, END
import asyncio

async def parallel_research(state: State) -> dict:
    # Executar em paralelo com asyncio
    web, rag, sql = await asyncio.gather(
        web_search_async(state["query"]),
        rag_search_async(state["query"]),
        sql_query_async(state["query"])
    )
    return {
        "web_results": web,
        "rag_results": rag,
        "sql_results": sql
    }
```

**Quando usar:** quando sub-tarefas são independentes e latência importa.
**Risco:** tasks que dependem umas das outras não podem ser paralelizadas sem cuidado.

### Padrão 4 — Evaluator-Optimizer (Loop de Qualidade)

Um avaliador critica o output e um refinador melhora, até atingir qualidade suficiente:

```
Query → [Generator] → [Evaluator] → Se qualidade < threshold → [Generator]
                                  → Se qualidade >= threshold → Output final
```

```python
def evaluator_node(state: State) -> dict:
    score_prompt = f"""
    Avalie esta resposta de 0 a 1 com base em: completude, precisão, clareza.
    Resposta: {state['draft']}
    Retorne apenas: {{"score": 0.X, "feedback": "..."}}
    """
    result = llm.invoke(score_prompt)
    evaluation = json.loads(result.content)
    return {
        "quality_score": evaluation["score"],
        "quality_feedback": evaluation["feedback"],
        "iteration": state["iteration"] + 1
    }

def route_after_evaluation(state: State) -> str:
    if state["quality_score"] >= 0.85:
        return "finalize"
    elif state["iteration"] >= 3:  # guardrail
        return "finalize"  # aceitar mesmo imperfeito
    else:
        return "generate"  # tentar novamente
```

**Quando usar:** quando qualidade é mais importante que latência; tradução, geração de código, redação.
**Risco:** loop infinito sem guardrail de max_iterations.

### Padrão 5 — Orchestrator-Subagents (Delegação Hierárquica)

Um orquestrador decompõe o problema e delega para subagentes especializados:

```
Objetivo → [Orquestrador]
              ↓
         Decompõe em subtarefas
              ↓
    [Agente A] [Agente B] [Agente C]
              ↓
         Agrega resultados
              ↓
         Resposta final
```

Este padrão é o coração do M06 (Multiagente). Aqui, o foco é a lógica de decomposição:

```python
def orchestrator_node(state: State) -> dict:
    """Orquestrador que decide quais subagentes usar e com quais tasks."""
    decomposition_prompt = f"""
    Dado o objetivo: {state['objective']}
    
    Decomponha em 2-4 subtarefas independentes.
    Para cada subtarefa, especifique qual agente deve executar:
    - web_search_agent: para pesquisa online
    - data_analysis_agent: para análise de dados  
    - code_agent: para geração/execução de código
    - writing_agent: para síntese e redação
    
    Retorne JSON: [{{"task": "...", "agent": "...", "input": "..."}}]
    """
    plan = json.loads(llm.invoke(decomposition_prompt).content)
    return {"execution_plan": plan, "current_phase": "planning_done"}
```

### Padrão 6 — Plan and Execute

Um planejador cria um plano completo antes de executar qualquer passo:

```
Query → [Planner: cria plano completo] → [Executor: executa passo 1]
                                       → [Executor: executa passo 2]
                                       → [Replanejador (se necessário)]
                                       → Output final
```

Diferente do ReAct (que planeja um passo por vez), Plan-and-Execute planeja tudo primeiro — melhor para tarefas longas onde coerência entre passos importa.

```python
# LangChain tem implementação nativa
from langchain_experimental.plan_and_execute import PlanAndExecute, load_agent_executor, load_chat_planner

planner = load_chat_planner(llm)
executor = load_agent_executor(llm, tools, verbose=True)
agent = PlanAndExecute(planner=planner, executor=executor, verbose=True)
```

### Combinando padrões

Os padrões mais poderosos surgem da combinação:

```
Router → identifica o tipo de tarefa
         ↓ (se complexo)
Orchestrator → decompõe em subtarefas
               ↓
Parallel → executa subtarefas simultaneamente
            ↓
Evaluator-Optimizer → refina o resultado
                       ↓
Chain → formata e entrega
```

---

## Aplicação Prática

**Escolhendo o padrão certo:**

| Pergunta | Padrão indicado |
|---|---|
| Passos sequenciais sem condicionais? | Chain |
| Diferentes inputs requerem tratamentos diferentes? | Router |
| Sub-tarefas independentes que podem ser paralelas? | Parallelization |
| Precisa de qualidade garantida por iteração? | Evaluator-Optimizer |
| Objetivo complexo que precisa de decomposição? | Orchestrator-Subagents |
| Tarefa longa onde coerência do plano importa? | Plan-and-Execute |

---

## Conexões

- [05-01 — Grafos LangGraph](05-01-Grafos-de-Decisao-LangGraph.md): infraestrutura onde esses padrões são implementados
- [06-01 — Arquitetura Multiagente](../Modulo-06-Sistemas-Multiagente/06-01-Arquitetura-Multiagente.md): Orchestrator-Subagents em escala
- [05-04 — Ferramentas e Integrações](05-04-Tools-Externas-e-Integracoes.md): as ferramentas que os agentes usam

---

## Recursos Complementares

- **"Building Effective Agents" — Anthropic Engineering** (2024): [anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents) — os 6 padrões apresentados com exemplos visuais
- **LangGraph tutorials por padrão**: [langchain-ai.github.io/langgraph/tutorials](https://langchain-ai.github.io/langgraph/tutorials)
- **"Agentic Design Patterns" — Andrew Ng (DeepLearning.AI)**: série de artigos sobre cada padrão

---

## Auto-reflexão

1. Por que Plan-and-Execute é preferível ao ReAct para tarefas longas onde coerência entre passos importa?
2. Um sistema de atendimento ao cliente recebe perguntas de produto, financeiras e técnicas. Qual padrão de orquestração você usaria e por quê?
3. O Evaluator-Optimizer pode entrar em loop infinito. Quais são os dois guardrails que você sempre implementa?
4. Combine padrões para projetar um sistema que: recebe um objetivo de pesquisa, decide se é simples ou complexo, decompõe tarefas se complexo, pesquisa em paralelo, e garante qualidade do resultado final antes de entregar.

---

*[05-03] | Anterior: [05-02](05-02-Controle-de-Estado-LangGraph.md) | Próximo: [05-04](05-04-Tools-Externas-e-Integracoes.md)*
*Referências: Anthropic "Building Effective Agents" (2024); LangGraph patterns; Andrew Ng DeepLearning.AI*
