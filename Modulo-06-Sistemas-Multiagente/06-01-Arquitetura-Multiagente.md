# [06-01] Arquitetura Multiagente — Quando Um Agente Não Basta

## Por que isso importa?

Sistemas multiagente são a fronteira atual de aplicações de IA em produção. Entender quando um único agente é suficiente (e quando não é), como dividir responsabilidades, e como agentes se comunicam é o que separa projetos de demonstração de sistemas que escalam.

---

## Conceito Central

**Sistemas multiagente dividem um problema complexo entre agentes especializados que colaboram. A vantagem não é apenas paralelismo — é especialização: cada agente tem seu próprio contexto, ferramentas, e system prompt otimizado para seu papel, sem a "confusão" de um agente tentando fazer tudo.**

---

## Aprofundamento

### Por que múltiplos agentes?

**Problema 1: Janela de contexto**
Um único agente acumulando contexto de uma tarefa longa fica "distraído" e perde rastreamento de instruções antigas. Múltiplos agentes especialistas com contextos menores são mais precisos.

**Problema 2: Especialização**
Um único agente com 20 tools tende a confundir quando usar cada uma. Um agente de busca com 3 tools de busca e um agente de análise com 3 tools de análise são mais precisos cada um no seu domínio.

**Problema 3: Paralelismo**
Tarefas independentes bloqueiam umas às outras em um único agente sequencial. Múltiplos agentes paralelos (M05-03) resolvem isso.

**Problema 4: Verificação e checks-and-balances**
Um agente gera, outro critica. Esse padrão de "revisor" é mais robusto que auto-revisão.

### As topologias fundamentais

**1. Supervisor (Hub-and-Spoke)**
Um agente orquestrador central delega para workers especializados:

```
              [Supervisor LLM]
             /        |         \
    [Pesquisador] [Analista] [Redator]
```

O supervisor recebe a tarefa, decide qual worker chamar, coleta o resultado, decide o próximo passo. Workers não se comunicam diretamente.

```python
from langgraph.graph import StateGraph
from langchain_openai import ChatOpenAI

# Supervisor decide qual agente chamar
supervisor_prompt = """Você é um supervisor que coordena uma equipe de agentes.
Agentes disponíveis:
- researcher: busca informações na internet e documentos internos
- analyst: analisa dados e gera insights quantitativos
- writer: redige textos, relatórios e comunicações

Baseado na tarefa atual, decida qual agente chamar a seguir.
Se a tarefa estiver completa, responda FINISH.
"""

def supervisor_node(state):
    messages = [SystemMessage(content=supervisor_prompt)] + state["messages"]
    response = llm.invoke(messages)
    return {"next_agent": response.content, "messages": [response]}

def router(state):
    next_agent = state.get("next_agent", "").strip()
    if "FINISH" in next_agent or not next_agent:
        return END
    return next_agent

workflow.add_conditional_edges("supervisor", router, {
    "researcher": "researcher",
    "analyst": "analyst", 
    "writer": "writer",
    END: END
})
# Todos os workers retornam ao supervisor
for worker in ["researcher", "analyst", "writer"]:
    workflow.add_edge(worker, "supervisor")
```

**2. Hierárquica (Multi-nível)**
Supervisores gerenciam supervisores:

```
[Orquestrador Principal]
         ↓
[Sub-orquestrador Pesquisa] [Sub-orquestrador Produção]
         ↓                           ↓
[Web Agent] [DB Agent]      [Writer] [Reviewer]
```

Útil para organizações complexas onde cada "departamento" tem seu próprio orquestrador.

**3. Peer-to-Peer (Handoff direto)**
Agentes passam o controle diretamente entre si:

```
[Agente A] → handoff → [Agente B] → handoff → [Agente C]
```

```python
from langgraph_sdk import get_client

# Agent A passa para Agent B via handoff
def agent_a_node(state):
    result = agent_a.invoke(state)
    if result.requires_specialized_help:
        # Handoff explícito para Agent B
        return {"next": "agent_b", "context_for_b": result.partial_work}
    return {"final_answer": result.answer}
```

**4. Debate (Multi-perspectiva)**
Múltiplos agentes propõem soluções, um juiz seleciona ou sintetiza:

```
[Agente Conservador] ─┐
[Agente Inovador]    ─→ [Juiz/Sintetizador] → Decisão
[Agente Crítico]     ─┘
```

Útil para decisões de alta importância onde perspectivas múltiplas são valiosas.

### Comunicação entre agentes

**Mensagens no estado compartilhado (LangGraph)**
O mais simples: agentes leem e escrevem no mesmo estado.

```python
# Agente A escreve
def researcher_node(state: State) -> dict:
    findings = do_research(state["query"])
    return {"research_findings": findings}  # disponível para todos os agentes

# Agente B lê
def analyst_node(state: State) -> dict:
    findings = state["research_findings"]  # lê o que A escreveu
    analysis = analyze(findings)
    return {"analysis": analysis}
```

**Via API/mensagens (sistemas distribuídos)**
Para agentes em processos/máquinas diferentes:
```python
# LangGraph Cloud / LangGraph Platform
from langgraph_sdk import get_sync_client

client = get_sync_client(url="http://localhost:2024")

# Agente A cria um thread e passa para B
thread = client.threads.create()
run = client.runs.create(thread["thread_id"], "agent_a", input={"query": "..."})
# Agent B continua no mesmo thread
run_b = client.runs.create(thread["thread_id"], "agent_b", input={})
```

### Handoffs — transferência de controle

```python
from langgraph_supervisor import create_handoff_tool

# Tool de handoff: permite ao supervisor passar controle explicitamente
transfer_to_researcher = create_handoff_tool(
    agent_name="researcher",
    description="Transferir para o pesquisador quando precisar de informações externas"
)

transfer_to_analyst = create_handoff_tool(
    agent_name="analyst", 
    description="Transferir para o analista quando precisar de análise de dados"
)

supervisor_tools = [transfer_to_researcher, transfer_to_analyst]
```

---

## Aplicação Prática

**Quando usar sistema multiagente:**

| Sinal | Indicação |
|---|---|
| Agente único está "confuso" com contexto longo | Multiagente — dividir contexto |
| Tarefas claramente separáveis (pesquisa vs. análise) | Multiagente — especialização |
| Um agente tem 10+ ferramentas | Multiagente — dividir ferramentas |
| Precisamos de revisão independente | Agente revisor separado |
| Latência importa (tarefas paralelizáveis) | Multiagente paralelo |
| Tarefa simples de ponta a ponta | Agente único — não adicionar complexidade |

---

## Conexões

- [05-03 — Padrões de Orquestração](../Modulo-05-Orquestracao-Fluxos/05-03-Padroes-de-Orquestracao.md): padrões que sistemas multiagente implementam
- [06-02 — Comunicação A2A](06-02-Comunicacao-A2A.md): protocolos de comunicação entre agentes
- [06-04 — Debugging com LangSmith](06-04-Debugging-LangSmith.md): como rastrear sistemas multiagente

---

## Recursos Complementares

- **LangGraph multi-agent tutorial**: [langchain-ai.github.io/langgraph/tutorials/multi_agent](https://langchain-ai.github.io/langgraph/tutorials/multi_agent)
- **LangGraph Supervisor**: [github.com/langchain-ai/langgraph-supervisor-py](https://github.com/langchain-ai/langgraph-supervisor-py) — framework para padrão supervisor
- **Google A2A Protocol**: [github.com/google-deepmind/a2a](https://github.com/google-deepmind/a2a) — protocolo open para comunicação entre agentes
- **Alura — "Sistemas Multi-Agente com LangGraph"**: módulo 5 da Formação Agentes IA (pt-BR)

---

## Auto-reflexão

1. Por que um único agente com 15 ferramentas é inferior a 3 agentes especializados com 5 ferramentas cada, em termos de qualidade de decisão?
2. No padrão Supervisor, o que acontece se o supervisor entra em loop (fica chamando o mesmo worker indefinidamente)? Como prevenir?
3. Quais são as vantagens e desvantagens do padrão Peer-to-Peer vs. Supervisor?
4. Projete um sistema multiagente para o seguinte objetivo: dado um conjunto de dados de vendas, produza um relatório executivo com análise estatística, gráficos, e recomendações estratégicas. Quais agentes você criaria? Qual topologia usaria?

---

*[06-01] | Anterior: [05-04](../Modulo-05-Orquestracao-Fluxos/05-04-Tools-Externas-e-Integracoes.md) | Próximo: [06-02](06-02-Comunicacao-A2A.md)*
*Referências: LangGraph multi-agent docs; Google A2A protocol; Anthropic "Building Effective Agents"*
