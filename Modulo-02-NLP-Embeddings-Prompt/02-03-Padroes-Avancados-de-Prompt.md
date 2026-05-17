# [02-03] Padrões Avançados de Prompt

## Por que isso importa?

CoT básico é o começo. Os padrões avançados — ReAct, Self-Consistency, Tree of Thought — são o que separa sistemas de agentes robustos de demos frágeis. Esses padrões são a base teórica dos agentes que você vai construir em M04.

---

## Conceito Central

**Padrões avançados de prompt estruturam o raciocínio do modelo em etapas verificáveis, permitindo que o modelo "pense antes de agir", "verifique seu próprio trabalho" e "explore múltiplos caminhos".**

---

## Aprofundamento

### ReAct — Reason + Act

ReAct é o padrão de raciocínio usado pela maioria dos agentes modernos. O modelo alterna entre:
- **Thought:** raciocina sobre o que fazer
- **Action:** executa uma ação (chamar uma ferramenta, buscar informação)
- **Observation:** observa o resultado da ação
- **Thought:** raciocina sobre o próximo passo

```
Pergunta: Qual é a temperatura atual em São Paulo e como isso compara à média histórica de maio?

Thought: Preciso da temperatura atual e da média histórica. Vou buscar cada uma.
Action: search("temperatura atual São Paulo")
Observation: 18°C, céu nublado
Thought: Agora preciso da média histórica de maio em São Paulo.
Action: search("média histórica temperatura São Paulo maio")
Observation: Média histórica de maio: 19°C
Thought: Tenho os dois dados. A temperatura atual (18°C) está 1°C abaixo da média histórica (19°C) — dentro do normal.
Resposta: A temperatura atual em São Paulo é 18°C, 1°C abaixo da média histórica de maio (19°C). Variação normal.
```

Este padrão é exatamente o que acontece "por baixo dos panos" quando agentes com ferramentas (M04) executam tarefas. O modelo não executa as ações diretamente — o framework (LangChain, LangGraph) intercepta os "Action" e executa a ferramenta real.

### Self-Consistency — verificação por múltiplos caminhos

Em vez de gerar uma resposta e aceitar, gera N respostas independentes e faz votação majoritária (ou averaging). Mais caro, mas dramaticamente mais preciso para tarefas de raciocínio.

```python
# Pseudocódigo de Self-Consistency
responses = []
for i in range(5):  # 5 amostras independentes
    response = llm.generate(prompt, temperature=0.7)
    responses.append(extract_answer(response))

# Voto majoritário
from collections import Counter
final_answer = Counter(responses).most_common(1)[0][0]
```

**Quando usar:** tarefas matemáticas, raciocínio lógico, classificação onde a confiança importa. Custo: 5x mais tokens.

### Tree of Thought (ToT)

Extensão de CoT onde o modelo explora múltiplos "galhos" de raciocínio e avalia qual parece mais promissor, como um humano que considera várias abordagens antes de se comprometer com uma.

```
Problema: Como maximizar engajamento em post de LinkedIn?

Galho A: Abordagem emocional → storytelling pessoal → exemplo → CTA
  Avaliação: Alta ressonância, médio alcance orgânico

Galho B: Abordagem de valor → insight contraintuitivo → dados → CTA
  Avaliação: Médio ressonância, alto alcance por compartilhamentos

Galho C: Abordagem de lista → "X formas de..." → exemplos → CTA
  Avaliação: Baixa profundidade, alto alcance por formato

Decisão: Combinar B + A (insight contraintuitivo + storytelling) para balancear alcance e ressonância.
```

ToT é computacionalmente caro e raramente implementado puro — mas o **conceito** de "explorar antes de comprometer" é fundamental para agentes com planejamento.

### Padrão de verificação — Criticize and Revise

O modelo gera uma resposta, depois age como crítico da própria resposta, e finalmente revisa com base na crítica:

```
[Geração] Rascunho inicial: ...

[Crítica] Problemas identificados:
- O argumento X não está suportado por evidência
- O tom está muito técnico para o público-alvo
- Faltou mencionar a limitação Y

[Revisão] Versão revisada: ...
```

Simples de implementar, surpreendentemente eficaz para documentos que precisam de qualidade consistente.

### Meta-prompting — prompts que geram prompts

Para tarefas repetitivas com variações, o modelo pode gerar um prompt especializado para cada variação:

```
Dado o seguinte tipo de tarefa: [TIPO]
Gere um prompt otimizado para executar esta tarefa com máxima precisão.
O prompt deve incluir: role, contexto necessário, instrução específica, formato de output.
```

**Quando usar:** quando você tem muitas variações de uma mesma categoria de tarefa e quer sistematizar.

---

## Aplicação Prática

**Qual padrão usar em cada situação:**

| Situação | Padrão Recomendado |
|---|---|
| Tarefa simples, baixo custo | Zero-shot ou Few-shot |
| Raciocínio multi-etapa | Chain-of-Thought |
| Agente com ferramentas | ReAct (implementado pelo framework) |
| Matemática ou lógica com alta precisão | Self-Consistency |
| Planejamento complexo com alternativas | Tree of Thought (simplificado) |
| Documentos que precisam de qualidade consistente | Criticize and Revise |

**Template de ReAct para uso direto:**

```
Você tem acesso às seguintes ferramentas:
- search(query): busca informações na web
- calculate(expression): calcula expressões matemáticas
- read_file(path): lê o conteúdo de um arquivo

Para cada passo, use este formato exato:
Thought: [seu raciocínio]
Action: [nome_da_ferramenta]([argumento])
Observation: [resultado — será preenchido pelo sistema]

Quando tiver a resposta final:
Thought: Tenho todas as informações necessárias.
Final Answer: [resposta]

Pergunta: [PERGUNTA DO USUÁRIO]
```

---

## Conexões

- [02-02 — Engenharia de Prompt](02-02-Engenharia-de-Prompt.md): fundamentos que este módulo estende
- [M04-02 — ReAct na Prática](../Modulo-04-Agentes-de-IA/04-02-ReAct-e-Toolcalling.md): como ReAct vira agente real
- [M05-01 — LangGraph](../Modulo-05-Orquestracao-Fluxos/05-01-Grafos-de-Decisao-LangGraph.md): como ToT e ramificação viram fluxos

---

## Recursos Complementares

- **"ReAct: Synergizing Reasoning and Acting in Language Models"** (paper, 2022) — paper original
- **"Tree of Thoughts: Deliberate Problem Solving"** (paper, 2023) — paper original ToT
- **LangChain Agents docs**: [python.langchain.com/docs/modules/agents](https://python.langchain.com/docs/modules/agents)

---

## Auto-reflexão

1. ReAct alterna entre "Thought" e "Action". Por que essa alternância é mais eficaz do que gerar a resposta diretamente?
2. Se Self-Consistency gera 5 respostas e faz voto majoritário, quando essa abordagem pode falhar?
3. Você tem uma tarefa que o modelo consistentemente erra. Como você aplicaria Criticize and Revise para melhorar a consistência?
4. Tree of Thought é caro computacionalmente. Para qual tipo de problema o custo extra seria justificável?

---

## Próximo Passo

### [→ 02-04 — Limitações e Alucinações](02-04-Limitacoes-e-Alucinacoes.md)


---

*[← 02-02 — Engenharia de Prompt](02-02-Engenharia-de-Prompt.md) · [↑ M02 — NLP, Embeddings e Prompt](02-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
