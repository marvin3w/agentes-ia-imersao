# [04-01] O que é um Agente — Da Inferência à Ação

## Por que isso importa?

"Agente de IA" virou buzzword em 2024-2025. Mas a definição vaga ("uma IA que faz coisas") não ajuda a construir um. Este documento estabelece uma definição precisa, distingue agentes de chatbots e LLMs simples, e apresenta a arquitetura conceitual que sustenta tudo o mais no módulo.

---

## Conceito Central

**Um agente é um sistema que percebe seu ambiente, decide que ação tomar (possivelmente usando um LLM para raciocinar), executa a ação no mundo, e observa o resultado — em um loop contínuo até atingir um objetivo. O que o diferencia de um LLM puro é o loop de percepção-decisão-ação-observação.**

---

## Aprofundamento

### O espectro: LLM → Chatbot → Agente

**LLM puro:** recebe texto, gera texto. Um shot. Sem contexto, sem ferramentas, sem loop.

**Chatbot:** LLM com memória de conversa. Múltiplas turns. Mas ainda reativo — só responde, não age.

**Agente:** LLM + ferramentas + loop de execução. Proativo — pode tomar iniciativa, usar ferramentas, observar resultados e decidir o próximo passo.

```
LLM puro:   [Input] → [LLM] → [Output]

Chatbot:    [Histórico] + [Input] → [LLM] → [Output]
             ↑__________________________________|

Agente:     [Objetivo] → [LLM raciocina] → [Decide ação]
                                                   ↓
                              [Executa ferramenta no ambiente]
                                                   ↓
                              [Observa resultado]
                                                   ↓
                              [LLM avalia: objetivo atingido?]
                              [Se não: próxima ação] → loop
```

### Os componentes de um agente

**1. O Cérebro (LLM)**
Responsável por raciocinar, planejar e decidir. Recebe observações do ambiente e decide qual ação tomar. Em agentes modernos, o LLM recebe um "system prompt de agente" que define seu papel, as ferramentas disponíveis e o formato de saída esperado.

**2. As Ferramentas (Tools)**
Funções que o agente pode invocar para agir no ambiente. Exemplos:
- `search_web(query)` — busca na internet
- `read_file(path)` — lê um arquivo
- `execute_sql(query)` — roda uma query no banco
- `send_email(to, subject, body)` — envia email
- `search_knowledge_base(query)` — RAG (M03-04)
- `run_python(code)` — executa código

**3. A Memória**
O que o agente "lembra" ao longo do loop:
- **Working memory:** o contexto atual da conversa e dos passos executados
- **External memory:** RAG, banco de dados (M03-04)
- **Episódica:** histórico de conversas anteriores

**4. O Loop de Execução (Agent Runtime)**
O motor que conecta tudo: "pega o estado atual → pede ao LLM o próximo passo → executa a ferramenta → adiciona o resultado ao estado → repete".

### O modelo PEAS (Perceive-Execute-Act-Sense)

Da inteligência artificial clássica (Russell & Norvig, 1995), mas diretamente aplicável:

| Componente | O que é | Em um agente LLM |
|---|---|---|
| **Performance** | Critério de sucesso | Objetivo definido no prompt |
| **Environment** | Onde o agente opera | APIs, arquivos, web, banco de dados |
| **Actuators** | Como o agente age | Ferramentas (tools) |
| **Sensors** | Como percebe | Outputs das ferramentas, mensagens do usuário |

### Tipos de agentes por autonomia

**Nível 1 — Assistente simples (tool use)**
O usuário faz uma pergunta → o agente usa uma ferramenta → responde. Exemplo: "qual é o tempo em São Paulo?" → `get_weather("São Paulo")` → responde.

**Nível 2 — Agente de tarefa**
O usuário define um objetivo → o agente planeja e executa múltiplos passos. Exemplo: "pesquise os 5 maiores concorrentes da Gran Cursos e me dê um relatório" → busca web, agrega, formata, responde.

**Nível 3 — Agente autônomo de longo prazo**
O agente opera sozinho por horas ou dias, tomando decisões sem intervenção humana. Exemplo: um agente de monitoramento que verifica dashboards, cria tickets quando encontra anomalias, e notifica o time via Slack.

**Nível 4 — Agente multiagente**
Múltiplos agentes colaborando, cada um com papel especializado. M06 detalha isso.

### Por que agentes falham

**Erro de planejamento:** o agente erra na sequência de passos necessários para atingir o objetivo.

**Alucinação de ferramentas:** o agente "inventa" argumentos para ferramentas (ex.: chama `get_file("relatorio_q3.pdf")` quando o arquivo se chama `Q3-Report.pdf`).

**Loop infinito:** sem condição de parada, o agente fica refinando indefinidamente.

**Escalada de escopo:** o agente toma ações além do objetivo (ex.: "organize minha pasta" → deleta arquivos que "pareciam" duplicados).

**Por isso existem:** guardrails, timeouts, revisão humana em ações críticas, e o padrão Human-in-the-Loop (HITL).

---

## Aplicação Prática

**Anatomia de um agente mínimo em Python:**

```python
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain.tools import tool
from langchain import hub

# 1. Definir ferramentas
@tool
def calculate(expression: str) -> str:
    """Calcula expressões matemáticas. Input: expressão como '2 + 2' ou '15% de 340'."""
    try:
        result = eval(expression)  # em produção, use um parser seguro
        return str(result)
    except Exception as e:
        return f"Erro: {e}"

@tool  
def get_current_date() -> str:
    """Retorna a data atual."""
    from datetime import date
    return date.today().isoformat()

tools = [calculate, get_current_date]

# 2. Criar o agente
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
prompt = hub.pull("hwchase17/openai-tools-agent")

agent = create_openai_tools_agent(llm, tools, prompt)
executor = AgentExecutor(
    agent=agent, 
    tools=tools, 
    verbose=True,  # mostra o raciocínio passo a passo
    max_iterations=10,  # guardrail contra loop infinito
    early_stopping_method="generate"
)

# 3. Executar
result = executor.invoke({
    "input": "Quantos dias faltam para o dia 25 de dezembro deste ano?"
})
print(result["output"])
```

---

## Conexões

- [04-02 — ReAct e Toolcalling](04-02-ReAct-e-Toolcalling.md): como o LLM decide qual ferramenta usar
- [04-03 — LangChain na Prática](04-03-LangChain-na-Pratica.md): framework para construir agentes
- [03-04 — RAG como Ferramenta](../Modulo-03-RAG-Memoria-Contextual/03-04-RAG-como-Ferramenta-de-Agente.md): memória como ferramenta do agente
- [05-01 — Grafos de Decisão](../Modulo-05-Orquestracao-Fluxos/05-01-Grafos-de-Decisao-LangGraph.md): controlar o loop com mais sofisticação

---

## Recursos Complementares

- **"Agents" — Anthropic documentation**: [docs.anthropic.com/agents](https://docs.anthropic.com/en/docs/build-with-claude/agents) — visão do criador do Claude
- **"What are AI Agents?" — LangChain blog**: série introdutória com exemplos práticos
- **Russell & Norvig — "Artificial Intelligence: A Modern Approach"** cap. 2 (agentes racionais): a base teórica clássica
- **Alura — Formação Agentes IA, semana 1**: "Introdução a Agentes de IA com LangChain" (5h, pt-BR)

---

## Auto-reflexão

1. Qual é a diferença fundamental entre um chatbot com memória de conversa e um agente? O que o agente tem que o chatbot não tem?
2. Um agente com acesso a `send_email` e `delete_file` é pedido para "limpar minha caixa de entrada e deletar tudo que for spam". Quais riscos surgem? Como mitigar?
3. Por que `max_iterations` é um guardrail importante? O que acontece sem ele?
4. Você precisa construir um sistema que monitora métricas de um blog a cada hora e envia alert no Slack quando algo está anormal. É isso um agente de Nível 1, 2 ou 3? Justifique.

---

## Próximo Passo

### [→ 04-02 — ReAct e Toolcalling](04-02-ReAct-e-Toolcalling.md)


---

*[← M04 — Agentes de IA](04-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
