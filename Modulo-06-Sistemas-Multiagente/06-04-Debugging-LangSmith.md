# [06-04] Debugging com LangSmith — Observabilidade para Agentes

## Por que isso importa?

Debugar um agente que alucinatou um tool call, um sistema multiagente onde um subagente falhou silenciosamente, ou um pipeline RAG que recuperou documentos errados — sem observabilidade, você está debugando às cegas. LangSmith é a plataforma de observabilidade nativa do ecossistema LangChain/LangGraph.

---

## Conceito Central

**LangSmith registra cada passo de execução de qualquer chain, agente ou grafo LangChain/LangGraph: inputs, outputs, latência, custo de tokens, traces hierárquicos. Você consegue ver exatamente o que o modelo pensou, qual ferramenta chamou, o que retornou, e onde errou — sem instrumentar código manualmente.**

---

## Aprofundamento

### Setup mínimo

```bash
pip install langsmith
```

```python
import os

# Variáveis de ambiente (adicionar ao .env)
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "lsv2_..."  # de smith.langchain.com
os.environ["LANGCHAIN_PROJECT"] = "meu-agente-projeto"

# Pronto — qualquer chain LangChain agora é rastreada automaticamente
# SEM modificar nenhum código adicional
```

### O que o LangSmith captura automaticamente

Para qualquer run de chain/agent/graph:

```
Run trace:
├── Input: {"question": "O que é RAG?"}
├── Output: "RAG (Retrieval-Augmented Generation) é..."
├── Latency: 2.4s
├── Tokens: 1.2K prompt + 0.3K completion = 1.5K total
├── Cost: $0.0023
└── Nested traces:
    ├── [ChatOpenAI] prompt → completion (0.8s)
    ├── [VectorStoreRetriever] query → 4 docs (0.3s)
    │   └── Results: ["doc1.pdf p.4", "doc2.pdf p.12", ...]
    └── [ChatOpenAI] context + question → answer (1.3s)
```

### Traces de agentes — ver o loop de raciocínio

Para agentes com múltiplas ferramentas, o LangSmith mostra cada iteração:

```
AgentExecutor Run:
├── Iteration 1:
│   ├── [ChatOpenAI] → Thought: "Preciso buscar na web"
│   │                  Action: search_web("LangGraph updates 2025")
│   ├── [search_web] → "LangGraph v0.2.0 lançado com..."
│   └── Observation added to context
├── Iteration 2:
│   ├── [ChatOpenAI] → Thought: "Agora preciso analisar os dados"
│   │                  Action: run_analysis(data=...)
│   ├── [run_analysis] → ERROR: TypeError: expected DataFrame
│   └── ERROR logged — modelo vai tentar corriger
├── Iteration 3:
│   ├── [ChatOpenAI] → Thought: "Houve erro. Vou formatar os dados primeiro"
│   │                  Action: run_analysis(data=formatted_data)
│   ├── [run_analysis] → "Análise: 3 updates principais..."
│   └── OK
└── Final Answer: "..."
```

Sem LangSmith: você vê apenas o erro final. Com LangSmith: você vê onde exatamente a iteração 2 falhou.

### Avaliação (Evals) — medir qualidade sistematicamente

```python
from langsmith import Client
from langsmith.evaluation import evaluate, LangChainStringEvaluator

client = Client()

# Dataset de exemplos de referência
dataset = client.create_dataset("rag_qa_test")
client.create_examples(
    inputs=[
        {"question": "Quais são os direitos do titular na LGPD?"},
        {"question": "Qual é o prazo de resposta para solicitações LGPD?"},
    ],
    outputs=[
        {"answer": "O titular tem direito a: confirmação, acesso, correção, anonimização..."},
        {"answer": "O prazo é de 15 dias."},
    ],
    dataset_id=dataset.id
)

# Avaliadores automáticos
evaluators = [
    LangChainStringEvaluator("qa"),           # resposta correta?
    LangChainStringEvaluator("faithfulness"), # fundamentada nos documentos?
    LangChainStringEvaluator("conciseness"),  # concisa?
]

# Rodar avaliação
results = evaluate(
    lambda x: rag_chain.invoke(x["question"]),
    data=dataset.name,
    evaluators=evaluators,
    experiment_prefix="rag_v2",
)

print(f"Correct: {results['qa']:.2%}")
print(f"Faithful: {results['faithfulness']:.2%}")
```

### Datasets e Regression Testing

```python
# Adicionar exemplos de falhas anteriores ao dataset
# Garante que bugs corrigidos não reaparecem

# Exemplo: agente que errou em um caso edge
client.create_examples(
    inputs=[{"question": "Qual é a DIFERENÇA entre RAG e fine-tuning?"}],  # maiúscula proposital
    outputs=[{"expected_topics": ["fine-tuning", "RAG", "diferença", "custo"]}],
    dataset_id=dataset.id,
    metadata={"category": "edge_case", "reported_bug": "BUG-123"}
)
```

### Anotações humanas e feedback

```python
# Salvar feedback do usuário final de volta ao LangSmith
from langsmith import Client

client = Client()

# Registrar feedback positivo/negativo
client.create_feedback(
    run_id=run.id,  # run_id disponível via callbacks
    key="user_rating",
    score=0.9,  # 0 a 1
    comment="Resposta correta mas poderia ser mais concisa"
)

# Filtrar runs com feedback negativo para análise
bad_runs = client.list_runs(
    project_name="meu-agente",
    filter="and(eq(feedback_key, 'user_rating'), lt(feedback_score, 0.5))"
)
```

### Debugging de sistemas multiagente

Em sistemas multiagente, LangSmith mostra a hierarquia completa de runs:

```
[Supervisor] Run (4.2s, $0.012)
├── [Researcher Agent] Run (1.8s, $0.004)
│   ├── [search_web] (0.9s) → 3 results
│   └── [ChatOpenAI] → summary
├── [Analyst Agent] Run (1.2s, $0.005)
│   ├── [run_python] (0.3s) → DataFrame
│   └── [ChatOpenAI] → insights
└── [Writer Agent] Run (1.2s, $0.003)
    └── [ChatOpenAI] → final report
```

Para identificar gargalos de latência e custo em sistemas multiagente, o dashboard mostra:
- Qual agente está consumindo mais tokens
- Qual ferramenta está demorando mais
- Onde a cadeia de erros começa

### Alertas e monitoramento de produção

```python
# LangSmith tem monitoramento contínuo para produção
# Regras configuráveis via dashboard:

# Exemplos de alertas:
# - Latência média > 5s nas últimas 100 runs
# - Taxa de erro de tools > 10%
# - Score de avaliação automática < 0.7
# - Custo diário > $50

# Via API, você pode puxar métricas
stats = client.get_run_stats(
    project_name="meu-agente-producao",
    start_time=datetime.now() - timedelta(hours=24)
)
print(f"Runs: {stats.total_runs}")
print(f"Errors: {stats.error_rate:.2%}")
print(f"Avg latency: {stats.latency_p50}ms")
print(f"Total cost: ${stats.total_tokens * 0.000001:.4f}")
```

---

## Aplicação Prática

**Workflow de debugging estruturado:**

```
1. Deploy com LANGCHAIN_TRACING_V2=true
2. Reproduzir o bug ou comportamento inesperado
3. Abrir LangSmith → encontrar o run pelo timestamp
4. Navegar na árvore de traces → identificar onde divergiu
5. Isolar o passo problemático → abrir no "Playground" (rerun com modificações)
6. Corrigir e adicionar ao dataset de testes
7. Rodar avaliação para garantir que não quebrou outros casos
```

---

## Conexões

- [06-01 — Arquitetura Multiagente](06-01-Arquitetura-Multiagente.md): sistemas que precisam de observabilidade
- [04-03 — LangChain na Prática](../Modulo-04-Agentes-de-IA/04-03-LangChain-na-Pratica.md): LangSmith integra com qualquer cadeia LangChain
- [07-03 — Servidores MCP](../Modulo-07-MCP-Protocolos-Modernos/07-03-Criando-Servidores-MCP.md): instrumentar MCP servers com observabilidade

---

## Recursos Complementares

- **LangSmith docs**: [docs.smith.langchain.com](https://docs.smith.langchain.com) — documentação oficial
- **LangSmith evaluation guide**: [docs.smith.langchain.com/evaluation](https://docs.smith.langchain.com/evaluation)
- **"Evaluating LLM Applications" — Hamel Husain**: melhores práticas de avaliação
- **RAGAS** (framework de avaliação de RAG): [docs.ragas.io](https://docs.ragas.io) — complementar ao LangSmith para RAG específico

---

## Auto-reflexão

1. Sem LangSmith, como você debugaria um agente que chama 5 ferramentas em sequência e retorna um resultado errado? Com LangSmith?
2. Por que "evals" (avaliação sistemática) são mais valiosas do que testar manualmente um caso por vez?
3. Um sistema de produção tem 1000 runs/dia. Como você configuraria um sistema de monitoramento que detecta degradação de qualidade antes que os usuários reclamem?
4. Qual é a diferença entre rastrear para debugging (desenvolvimento) e monitorar para produção? O que muda na configuração?

---

*[06-04] | Anterior: [06-03](06-03-Padroes-Avancados-Multiagente.md) | Próximo: [M07-00](../Modulo-07-MCP-Protocolos-Modernos/07-00-Visao-Geral.md)*
*Referências: LangSmith docs; RAGAS framework; "Evaluating LLM Applications" (Hamel Husain)*
