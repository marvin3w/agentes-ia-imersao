# [06-03] Padrões Avançados Multiagente — Debate, Reflexão e Auto-Melhoria

## Por que isso importa?

Os padrões básicos (supervisor, pipeline) funcionam para tarefas com objetivos claros. Mas alguns problemas requerem raciocínio mais sofisticado: sistemas que se questionam, que usam múltiplas perspectivas para melhorar qualidade, ou que aprendem com seus próprios erros. Esses padrões representam o estado da arte em sistemas de IA de 2025.

---

## Conceito Central

**Os padrões avançados giram em torno de dois princípios: (1) perspectivas múltiplas produzem outputs de maior qualidade do que um único agente, e (2) reflexão iterativa (o agente avaliando e melhorando seu próprio trabalho) é mais eficaz do que geração em uma única passagem.**

---

## Aprofundamento

### Padrão 1 — Reflexão (Self-Critique)

Um agente gera um output, depois outro (ou o mesmo em novo contexto) critica:

```python
# Agente Gerador
generator_prompt = """Você é um escritor técnico especializado.
Escreva um artigo sobre: {topic}
Requisitos: preciso, 500 palavras, com exemplos concretos.
"""

# Agente Crítico
critic_prompt = """Você é um editor técnico rigoroso.
Revise o artigo abaixo e identifique:
1. Imprecisões técnicas
2. Argumentos sem evidência
3. Partes pouco claras
4. O que está faltando

Artigo para revisar:
{draft}

Forneça feedback específico e acionável."""

# Agente Refinador
refine_prompt = """Você recebeu um rascunho e feedback de um editor.
Reescreva o artigo incorporando TODAS as sugestões do editor.

Rascunho original:
{draft}

Feedback do editor:
{feedback}

Artigo revisado:"""

# Pipeline de reflexão
def reflection_pipeline(topic: str, max_rounds: int = 3) -> str:
    draft = (generator_prompt | llm | StrOutputParser()).invoke({"topic": topic})
    
    for round_num in range(max_rounds):
        feedback = (critic_prompt | llm | StrOutputParser()).invoke({"draft": draft})
        
        # Verificar se há algo a melhorar
        if "APROVADO" in feedback or round_num == max_rounds - 1:
            break
            
        draft = (refine_prompt | llm | StrOutputParser()).invoke({
            "draft": draft, "feedback": feedback
        })
    
    return draft
```

**Quando usar:** geração de código, redação técnica, análises financeiras — qualquer output onde qualidade importa mais que velocidade.

### Padrão 2 — Society of Mind (Debate entre Agentes)

Múltiplos agentes com perspectivas diferentes debatem e um árbitro sintetiza:

```python
agents = {
    "optimist": "Você vê o potencial positivo e as oportunidades em qualquer situação.",
    "pessimist": "Você identifica riscos, problemas e o que pode dar errado.",
    "pragmatist": "Você foca no que é realizável com os recursos atuais.",
    "innovator": "Você propõe abordagens não convencionais e criativas."
}

def debate_round(question: str, position_history: list) -> dict:
    """Uma rodada de debate entre todos os agentes."""
    responses = {}
    for agent_name, agent_persona in agents.items():
        prompt = f"""Você é um especialista com a seguinte perspectiva: {agent_persona}
        
Questão em debate: {question}
Posições anteriores: {position_history}

Apresente sua análise e posição. Você pode concordar ou discordar das posições anteriores, mas justifique."""
        
        responses[agent_name] = llm.invoke(prompt).content
    
    return responses

def synthesize_debate(question: str, all_responses: list) -> str:
    """Árbitro sintetiza as posições em uma decisão balanceada."""
    synthesis_prompt = f"""Você é um árbitro imparcial.
    
Questão: {question}

Posições de múltiplos especialistas:
{json.dumps(all_responses, indent=2, ensure_ascii=False)}

Sua tarefa: sintetize as perspectivas em uma conclusão balanceada que:
1. Reconhece os pontos válidos de cada perspectiva
2. Resolve as contradições com raciocínio explícito
3. Entrega uma recomendação clara e acionável"""
    
    return llm.invoke(synthesis_prompt).content
```

**Quando usar:** decisões estratégicas de alto impacto, avaliação de riscos, brainstorming estruturado.

### Padrão 3 — STORM (Synthesis of Topic Outlines through Retrieval and Multi-perspective Question Asking)

Originado em Stanford (2024), o STORM usa múltiplos "perspectivistas" para gerar perguntas diversas antes de pesquisar:

```
Tópico → [Agente 1 como jornalista: gera perguntas da perspectiva de um jornalista]
        → [Agente 2 como especialista técnico: gera perguntas técnicas]
        → [Agente 3 como usuário: gera perguntas práticas]
        ↓
Todas as perguntas → [Pesquisador: busca respostas para cada]
        ↓
[Sintetizador: organiza em artigo estruturado com cobertura múltipla]
```

```python
# Implementação simplificada do STORM
perspectives = ["jornalista investigativo", "engenheiro de software sênior", "executivo de negócios"]

def storm_pipeline(topic: str) -> str:
    # Fase 1: gerar perguntas de múltiplas perspectivas
    all_questions = []
    for perspective in perspectives:
        questions_prompt = f"""Como {perspective}, gere 5 perguntas profundas sobre: {topic}
        Foque no que seria mais importante para sua perspectiva."""
        
        questions = llm.invoke(questions_prompt).content
        all_questions.extend(questions.split("\n"))
    
    # Fase 2: pesquisar respostas para cada pergunta
    qa_pairs = []
    for question in all_questions:
        if question.strip():
            answer = rag_chain.invoke(question)
            qa_pairs.append({"question": question, "answer": answer})
    
    # Fase 3: sintetizar em artigo estruturado
    synthesis_prompt = f"""Com base nas pesquisas abaixo sobre "{topic}", 
    escreva um artigo abrangente que cubra todas as perspectivas:
    
    {json.dumps(qa_pairs, ensure_ascii=False)}"""
    
    return llm.invoke(synthesis_prompt).content
```

### Padrão 4 — MCTS (Monte Carlo Tree Search) para Agentes

Inspirado em IA para jogos (AlphaGo), MCTS explora múltiplos caminhos de raciocínio e seleciona o melhor:

```
                    [Estado Inicial]
                   /       |        \
            [Path A]   [Path B]   [Path C]
           /    \          |        /    \
       [A1]    [A2]      [B1]   [C1]   [C2]
                           ↓
               Score cada leaf com o resultado
                           ↓
               Propagar scores para cima
                           ↓
               Selecionar path com maior score esperado
```

```python
def mcts_reasoning(problem: str, branching_factor: int = 3, depth: int = 3) -> str:
    """Reasoning com busca em árvore — melhor para problemas com múltiplas soluções."""
    
    def expand_node(context: str, step: int) -> list[str]:
        """Gera N abordagens alternativas para o próximo passo."""
        prompt = f"""Dado o contexto: {context}
        Gere {branching_factor} abordagens DIFERENTES e DISTINTAS para o próximo passo.
        Retorne como JSON: [{{"approach": "...", "rationale": "..."}}]"""
        return json.loads(llm.invoke(prompt).content)
    
    def evaluate_path(path: list[str]) -> float:
        """Avalia a qualidade de um caminho de raciocínio."""
        eval_prompt = f"""Avalie este caminho de raciocínio para resolver: {problem}
        Caminho: {path}
        
        Score de 0 a 1 (1 = resolve completamente, 0 = irrelevante/incorreto).
        Retorne: {{"score": X.X, "reason": "..."}}"""
        return json.loads(llm.invoke(eval_prompt).content)["score"]
    
    # Construir e explorar árvore
    best_path, best_score = [], 0
    # [implementação completa usaria recursão com backtracking]
    
    return generate_final_answer(problem, best_path)
```

### Padrão 5 — Self-Play e Evolução

Agentes que melhoram gerando exemplos e aprendendo com eles (inspirado em AlphaZero):

```
[Agente v1] → gera outputs → [Avaliador] → filtra os melhores
→ exemplos de qualidade → [Fine-tuning] → [Agente v2] → melhor
→ [Agente v2] → gera outputs → ... → [Agente vN]
```

Este padrão requer fine-tuning (custo maior) mas produz melhorias drásticas de qualidade em domínios específicos.

---

## Aplicação Prática

**Escolhendo o padrão avançado:**

| Objetivo | Padrão |
|---|---|
| Melhorar qualidade de texto/código iterativamente | Reflexão |
| Decisão que beneficia de perspectivas opostas | Debate/Society of Mind |
| Pesquisa abrangente e multi-perspectiva | STORM |
| Problema com espaço de soluções grande | MCTS |
| Especialização em domínio específico (longo prazo) | Self-Play + fine-tuning |

---

## Conexões

- [06-01 — Arquitetura Multiagente](06-01-Arquitetura-Multiagente.md): fundamentos que esses padrões estendem
- [05-03 — Padrões de Orquestração](../Modulo-05-Orquestracao-Fluxos/05-03-Padroes-de-Orquestracao.md): Evaluator-Optimizer é a versão simples de Reflexão
- [06-04 — Debugging LangSmith](06-04-Debugging-LangSmith.md): rastrear loops de reflexão e debate

---

## Recursos Complementares

- **STORM (Stanford, 2024)**: [github.com/stanford-oval/storm](https://github.com/stanford-oval/storm) — implementação completa
- **"Reflexion: Language Agents with Verbal Reinforcement Learning"** (Shinn et al., 2023): [arxiv.org/abs/2303.11366](https://arxiv.org/abs/2303.11366) — o paper original de reflexão
- **"Society of Mind"** — Marvin Minsky (1986): o livro seminal que inspirou esses padrões
- **Tree of Thoughts** (Yao et al., 2023): [arxiv.org/abs/2305.10601](https://arxiv.org/abs/2305.10601) — MCTS para LLMs

---

## Auto-reflexão

1. Por que reflexão iterativa (gerar → criticar → refinar) produz outputs de maior qualidade do que geração única com um prompt muito detalhado?
2. O padrão de Debate tem um agente "otimista" e um "pessimista". Como o árbitro decide qual perspectiva tem mais peso? O que influencia essa decisão?
3. STORM usa perspectivistas para gerar perguntas diversas antes de pesquisar. Por que isso produz cobertura mais abrangente do que um único pesquisador?
4. Qual é a relação entre MCTS (Monte Carlo Tree Search) em jogos como xadrez e MCTS para raciocínio de LLMs? O que é diferente?

---

## Próximo Passo

### [→ 06-04 — Debugging com LangSmith](06-04-Debugging-LangSmith.md)


---

*[← 06-02 — Comunicação A2A](06-02-Comunicacao-A2A.md) · [↑ M06 — Sistemas Multiagente](06-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
