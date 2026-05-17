# [03-04] RAG como Ferramenta de Agente — Memória de Longo Prazo

## Por que isso importa?

RAG pode ser um pipeline estático ("pergunta → recupera → gera") ou uma ferramenta dinâmica que um agente decide usar ou não. A diferença determina o nível de sofisticação dos sistemas que você vai construir.

---

## Conceito Central

**Quando um agente pode chamar RAG como ferramenta, ele não busca sempre — ele decide quando buscar, o que buscar, e pode refinar a busca com base nos resultados. Isso transforma recuperação passiva em raciocínio ativo.**

---

## Aprofundamento

### RAG estático vs. RAG agêntico

**RAG estático (pipeline fixo):**
```
Query → Sempre busca → Sempre gera com contexto
```
Funciona bem para chatbots de documentação simples. Problema: busca mesmo quando não precisa, busca sempre com a query original (sem refinamento), não adapta a estratégia.

**RAG agêntico:**
```
Query → Agente decide se precisa buscar
       → Agente formula query de busca (pode ser diferente da query original)
       → Agente avalia resultados
       → Agente decide se precisa buscar mais, reformular, ou gerar
       → Geração
```

### Memória em sistemas de agentes

Agentes têm, na prática, quatro tipos de memória:

| Tipo | Onde vive | Escopo | Exemplo |
|---|---|---|---|
| **In-context (working)** | Janela de contexto atual | Conversa atual | Últimas 10 mensagens |
| **External (RAG)** | Banco vetorial | Longo prazo, sem limite | Base de conhecimento inteira |
| **Episódica** | Banco de dados + embeddings | Histórico de conversas passadas | "Na última semana, o usuário perguntou sobre X" |
| **Semântica (pesos do modelo)** | Parâmetros do LLM | Estático, global | O que o modelo aprendeu no treinamento |

RAG é a principal implementação de memória externa (longo prazo) para agentes.

### Implementando RAG como ferramenta

No LangChain, tools são funções que o agente pode decidir chamar:

```python
from langchain.tools import Tool
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_openai import ChatOpenAI
from langchain import hub

# 1. Criar a ferramenta de busca
def search_knowledge_base(query: str) -> str:
    """Busca informações relevantes na base de conhecimento interna.
    Use quando precisar de informações sobre documentos, contratos, políticas ou procedimentos internos.
    Retorna os trechos mais relevantes encontrados.
    """
    docs = vectorstore.similarity_search(query, k=4)
    
    if not docs:
        return "Nenhuma informação encontrada sobre este tema."
    
    results = []
    for doc in docs:
        source = doc.metadata.get("source", "desconhecido")
        results.append(f"[Fonte: {source}]\n{doc.page_content}")
    
    return "\n\n---\n\n".join(results)

rag_tool = Tool(
    name="search_knowledge_base",
    func=search_knowledge_base,
    description="""Busca informações relevantes na base de conhecimento interna.
    Use quando precisar de informações sobre documentos, contratos, políticas ou procedimentos internos.
    Input: uma query de busca em linguagem natural.
    Output: trechos relevantes encontrados."""
)

# 2. Criar o agente com a ferramenta
llm = ChatOpenAI(model="gpt-4o", temperature=0)
prompt = hub.pull("hwchase17/openai-tools-agent")

tools = [rag_tool]  # pode ter mais tools: calculator, web_search, etc.
agent = create_openai_tools_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 3. O agente decide quando usar a ferramenta
result = agent_executor.invoke({
    "input": "Qual é o prazo de resposta para solicitações LGPD segundo nossa política interna?"
})
```

### Adaptive RAG — refinamento iterativo

O agente pode avaliar se os resultados são suficientes e refinar:

```python
# Prompt de avaliação de resultados
eval_prompt = """Avalie se os documentos recuperados são suficientes para responder à pergunta.

Pergunta original: {question}
Documentos recuperados: {context}

Responda:
- SUFICIENTE: se os documentos respondem diretamente à pergunta
- INSUFICIENTE: se os documentos são irrelevantes ou não cobrem o tema
- PARCIAL: se cobrem parcialmente — especifique o que está faltando
"""

# O agente pode então reformular a query e buscar novamente
# Exemplo simplificado do loop:
def adaptive_rag(question: str, max_iterations: int = 3):
    for i in range(max_iterations):
        context = search_knowledge_base(question)
        evaluation = llm.invoke(eval_prompt.format(question=question, context=context))
        
        if "SUFICIENTE" in evaluation.content:
            return generate_answer(question, context)
        elif "PARCIAL" in evaluation.content:
            # Extrai o que está faltando e reformula a query
            missing = extract_missing(evaluation.content)
            question = f"{question} Especificamente sobre: {missing}"
        else:  # INSUFICIENTE
            # Tenta uma reformulação diferente
            question = reformulate_query(question, llm)
    
    return "Não encontrei informações suficientes após múltiplas tentativas."
```

### CRAG (Corrective RAG) — com avaliação de relevância

Variação onde o agente avalia os documentos recuperados antes de gerar:

```
Query → Recupera K chunks
     → Avalia relevância de cada chunk (score 0-1)
     → Se score alto: usa para geração
     → Se score médio: complementa com web search
     → Se score baixo: descarta e busca na web
     → Gera resposta com contexto filtrado/enriquecido
```

### Graph RAG — para dados relacionados

Quando documentos têm relações entre si (ex.: normas legais que se referenciam), RAG simples recupera chunks isolados. Graph RAG combina banco vetorial com grafo de conhecimento:

```
Chunks indexados como nós no grafo
Referências entre documentos = arestas
Query → busca vetorial → expande pelos vizinhos do grafo → contexto mais rico
```

Implementações: Microsoft GraphRAG ([github.com/microsoft/graphrag](https://github.com/microsoft/graphrag)), LlamaIndex Property Graph.

---

## Aplicação Prática

**Padrões de uso no vault (Marcus-Docs context):**

O vault que você usa tem exatamente a estrutura que se beneficia de RAG agêntico:
- Centenas de markdowns auto-referenciados
- Estrutura semântica (demandas, conduções, cockpit)
- Necessidade de recuperação por contexto, não por path exato

Um agente com RAG sobre o vault poderia:
1. Receber "qual é o status da demanda GPLG-468?"
2. Decidir buscar em `Meta-Marcus-Docs/` com query específica
3. Avaliar se encontrou o documento certo
4. Reformular para "GPLG-468 OR GMKTC-468 OR regressão INP"
5. Montar resposta com citações de arquivos específicos

---

## Conexões

- [03-03 — Pipelines RAG](03-03-Pipelines-RAG-do-Zero.md): base técnica que este módulo expande
- [04-01 — O que é um Agente](../Modulo-04-Agentes-de-IA/04-01-O-que-e-um-Agente.md): agentes que usam RAG como ferramenta
- [04-02 — ReAct e Toolcalling](../Modulo-04-Agentes-de-IA/04-02-ReAct-e-Toolcalling.md): como agentes decidem quando usar ferramentas
- [04-04 — Memória e Persistência](../Modulo-04-Agentes-de-IA/04-04-Memoria-e-Persistencia.md): tipos de memória de agentes em profundidade

---

## Recursos Complementares

- **Adaptive RAG paper**: "Adaptive-RAG: Learning to Adapt Retrieval-Augmented Large Language Models through Question Complexity" (Jeong et al., 2024)
- **CRAG paper**: "Corrective Retrieval Augmented Generation" (Yan et al., 2024)
- **Microsoft GraphRAG**: [microsoft.github.io/graphrag](https://microsoft.github.io/graphrag/) — documentação e exemplos
- **LangGraph + RAG tutorial**: [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph) — agentes com memória de longo prazo

---

## Auto-reflexão

1. Qual é a diferença fundamental entre RAG estático e RAG agêntico? Em que situação você escolheria cada um?
2. Um agente busca K chunks mas todos são irrelevantes. Sem lógica de avaliação de relevância, o que acontece? Com Adaptive RAG, o que acontece?
3. Por que Graph RAG é mais poderoso que RAG simples para bases de conhecimento com muitas referências cruzadas?
4. Projete a estrutura de memória de um agente de suporte ao cliente que precisa: lembrar do histórico de conversas do usuário, ter acesso à base de documentação, e conhecer o estado atual dos pedidos.

---

## Próximo Passo

### [→ M04 — Agentes de IA](../Modulo-04-Agentes-de-IA/04-00-Visao-Geral.md)

*Módulo concluído — clique acima para iniciar o próximo.*


---

*[← 03-03 — Pipelines RAG do Zero](03-03-Pipelines-RAG-do-Zero.md) · [↑ M03 — RAG e Memória Contextual](03-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
