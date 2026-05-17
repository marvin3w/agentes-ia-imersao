# [04-02] ReAct e Toolcalling — Como o Agente Pensa e Age

## Por que isso importa?

ReAct é o padrão que domina a implementação de agentes modernos. Entender o mecanismo interno — por que o LLM consegue decidir "preciso usar uma ferramenta agora" — é essencial para debugar agentes, melhorar performance e ir além dos exemplos básicos.

---

## Conceito Central

**ReAct (Reasoning + Acting) é um padrão de prompt que intercala pensamento explícito do LLM com ações. Ao escrever seu raciocínio antes de agir, o modelo comete menos erros de planejamento. Tool calling é o mecanismo técnico (API-level) que permite ao LLM invocar funções de forma estruturada.**

---

## Aprofundamento

### O paper original do ReAct (2022)

Yao et al. publicaram "ReAct: Synergizing Reasoning and Acting in Language Models" demonstrando que modelos que pensam em voz alta antes de agir são significativamente mais capazes em tarefas de raciocínio complexo.

O padrão fundamental:

```
Thought: [O que o modelo está pensando]
Action: [Qual ferramenta invocar] com [quais argumentos]
Observation: [O que a ferramenta retornou]
Thought: [Avaliação do resultado]
Action: [Próxima ferramenta ou resposta final]
...
```

**Exemplo real (simplificado):**
```
Question: Quantos habitantes tem a capital do Brasil?

Thought: Preciso saber qual é a capital do Brasil primeiro. Já sei que é Brasília, mas vou confirmar a população atual buscando na web.

Action: search_web("população Brasília 2024")
Observation: "Brasília tem aproximadamente 3,1 milhões de habitantes (2024, IBGE)"

Thought: Tenho a resposta. A capital é Brasília e tem ~3,1 milhões de habitantes.

Final Answer: A capital do Brasil é Brasília, com aproximadamente 3,1 milhões de habitantes.
```

O poder: ao externalizar o raciocínio, o modelo pode ser corrigido, auditado, e produz menos alucinações (o "Thought" age como freio antes de agir).

### Tool Calling — o mecanismo técnico

Tool calling (também chamado "function calling") é uma capacidade específica de APIs como OpenAI e Anthropic: o modelo pode retornar, ao invés de texto livre, um JSON estruturado indicando qual função chamar e com quais argumentos.

**Como funciona na API OpenAI:**

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "Busca informações na internet",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "A query de busca"
                    }
                },
                "required": ["query"]
            }
        }
    }
]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Qual é a população de Brasília?"}],
    tools=tools,
    tool_choice="auto"  # LLM decide se usa ou não a ferramenta
)

# O modelo retorna:
# response.choices[0].message.tool_calls = [
#   ToolCall(function=Function(name='search_web', arguments='{"query": "população Brasília 2024"}'))
# ]
```

O executador (AgentExecutor no LangChain) então:
1. Detecta que o modelo quer chamar uma ferramenta
2. Executa `search_web("população Brasília 2024")`
3. Adiciona o resultado ao histórico de mensagens
4. Chama o modelo novamente com o resultado
5. Modelo gera a resposta final

**Por que JSON estruturado é melhor que texto livre?**

Antes de function calling, o modelo precisava gerar texto como:
```
Action: search_web
Action Input: população Brasília 2024
```

E o código analisava esse texto com regex — quebrando facilmente com variações de formatação. JSON estruturado é robusto, tipado e sem ambiguidade.

### Parallel Tool Calling

Modelos modernos (GPT-4o, Claude 3.5) podem invocar múltiplas ferramentas em paralelo:

```
User: Compare o tempo em São Paulo e Rio de Janeiro agora.

Model: [chamadas paralelas]
  tool_call_1: get_weather("São Paulo")
  tool_call_2: get_weather("Rio de Janeiro")

[executar ambas em paralelo]

Observation 1: São Paulo: 22°C, parcialmente nublado
Observation 2: Rio de Janeiro: 28°C, ensolarado

Final Answer: Em São Paulo está 22°C e nublado, enquanto no Rio está 28°C e ensolarado.
```

Sem paralelo: 2 chamadas seriais. Com paralelo: ambas simultâneas, resposta 2x mais rápida.

### O system prompt de um agente — anatomia

O system prompt define o comportamento do agente:

```
Você é um assistente de análise de dados.

Ferramentas disponíveis:
- execute_sql: executa queries SQL no banco de dados da empresa
- create_chart: gera gráficos a partir de dados
- search_knowledge_base: busca na documentação interna

Instruções:
1. Sempre verifique os dados antes de gerar análises
2. Cite a fonte dos dados (tabela, query utilizada)
3. Se não encontrar os dados, informe explicitamente
4. Para análises complexas, explique o raciocínio passo a passo
5. Nunca execute queries destrutivas (DELETE, DROP, UPDATE)

Formato de resposta:
- Comece com um resumo de 1-2 frases
- Mostre os dados relevantes
- Conclua com insight acionável
```

A qualidade do system prompt é crítica — é o "DNA" do agente.

### Debugging de agentes ReAct

`verbose=True` no LangChain expõe o loop interno. Mas para produção, use LangSmith (M06):

```
> Entering new AgentExecutor chain...
Thought: O usuário quer saber os 5 produtos mais vendidos. Vou buscar no banco de dados.
Action: execute_sql
Action Input: SELECT product_name, SUM(quantity) as total FROM orders GROUP BY product_name ORDER BY total DESC LIMIT 5
Observation: [("Curso PLG", 1240), ("Curso Concurso", 987), ...]
Thought: Tenho os dados. Vou formatar a resposta.
Final Answer: Os 5 produtos mais vendidos são...
```

**Sinais de problema:**
- Loop sem progresso (mesma ação repetida)
- Tool calls com argumentos incorretos (modelo alucinando APIs)
- Observation retorna erro mas modelo ignora e prossegue
- Final Answer muito cedo (modelo desiste antes de usar as ferramentas)

---

## Aplicação Prática

**Melhorando a qualidade do raciocínio:**

```python
# Forçar CoT antes de tool calls (útil para tarefas complexas)
system_prompt = """Antes de usar qualquer ferramenta, pense em voz alta:
1. O que preciso descobrir para responder isso?
2. Qual ferramenta é mais adequada?
3. Qual é o formato correto dos argumentos?

Só então chame a ferramenta."""

# Isso aumenta a qualidade em tarefas complexas
# mas adiciona latência (mais tokens de "thinking")
```

---

## Conexões

- [04-01 — O que é um Agente](04-01-O-que-e-um-Agente.md): contexto geral de agentes
- [02-03 — Padrões Avançados de Prompt](../Modulo-02-NLP-Embeddings-Prompt/02-03-Padroes-Avancados-de-Prompt.md): ReAct como padrão de prompting avançado
- [04-03 — LangChain na Prática](04-03-LangChain-na-Pratica.md): implementação de agentes ReAct com LangChain
- [05-01 — LangGraph](../Modulo-05-Orquestracao-Fluxos/05-01-Grafos-de-Decisao-LangGraph.md): quando ReAct não é suficiente para fluxos complexos

---

## Recursos Complementares

- **"ReAct: Synergizing Reasoning and Acting in Language Models"** (Yao et al., 2022): [arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629) — paper original
- **OpenAI Function Calling guide**: [platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling)
- **Anthropic Tool Use guide**: [docs.anthropic.com/en/docs/build-with-claude/tool-use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- **LangSmith Tracing**: [smith.langchain.com](https://smith.langchain.com) — rastrear e debugar loops de agentes em produção

---

## Auto-reflexão

1. Por que o ReAct (intercalar raciocínio e ação) produz menos erros que um modelo que age diretamente sem "pensar em voz alta"?
2. Por que JSON estruturado (function calling) é preferível a parsear texto livre para invocar ferramentas?
3. Um agente tem acesso a 20 ferramentas. O que acontece com a qualidade das decisões do agente conforme o número de ferramentas cresce? Como mitigar?
4. Se `tool_choice="auto"` permite o modelo decidir se usa ou não ferramentas, quando você usaria `tool_choice="required"` ou `tool_choice={"name": "specific_tool"}`?

---

## Próximo Passo

### [→ 04-03 — LangChain na Prática](04-03-LangChain-na-Pratica.md)


---

*[← 04-01 — O que é um Agente](04-01-O-que-e-um-Agente.md) · [↑ M04 — Agentes de IA](04-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
