# [01-04] LLMs — Do Token ao Output

## Por que isso importa?

Quando você digita um prompt e o modelo responde, o que exatamente acontece? Entender o pipeline completo — tokenização → embeddings → Transformer → decodificação → output — explica por que temperatura importa, por que o modelo é probabilístico, e por que às vezes respostas iguais geram outputs diferentes.

---

## Conceito Central

**Um LLM é um modelo que, dado um contexto de tokens, prediz qual token deve vir a seguir — e repete isso até terminar a resposta.**

Isso é tudo. A aparente "inteligência" emerge de fazer essa predição muito bem, em escala massiva. Entender isso é libertador: o modelo não "pensa" — ele executa uma função de probabilidade sofisticada em alta velocidade.

---

## Aprofundamento

### Passo 1 — Tokenização

Texto não entra no modelo como caracteres ou palavras — entra como **tokens**. Um token é uma unidade de texto que pode ser uma palavra inteira, parte de uma palavra, ou um caractere especial.

```
"Hello, world!"  →  ["Hello", ",", " world", "!"]    (4 tokens)
"Antidisestablishmentarianism"  →  ["Anti", "dis", "est", "ab", "lish", "ment", "ari", "anism"]  (8 tokens)
```

**Por que isso importa:**
- Palavras em inglês tendem a ser 1-2 tokens; em português, 2-4 tokens (modelos treinados majoritariamente em inglês)
- Tokens custam dinheiro em APIs — "gpt-4o-mini a $0,15 por milhão de tokens de input"
- Limites de contexto são em tokens, não em palavras
- O modelo vê tokens, não palavras — "chat" e "Chat" podem ser tokens diferentes

**Regra prática:** 1 token ≈ 0,75 palavras em inglês ≈ 0,6 palavras em português.

### Passo 2 — Embeddings de input

Cada token é convertido em um vetor de números (embedding) — uma representação matemática que captura o significado do token no espaço vetorial. Tokens com significados semelhantes ficam próximos nesse espaço.

O embedding de input é a "língua" que o Transformer fala. (Aprofundamento em [M02-01](../Modulo-02-NLP-Embeddings-Prompt/02-01-Embeddings-e-Espaco-Vetorial.md).)

### Passo 3 — Transformer processa

O mecanismo de atenção (M01-03) processa todos os tokens em paralelo, criando representações contextualizadas — o mesmo token "banco" terá representações diferentes dependendo do contexto ao redor.

### Passo 4 — Decodificação e Temperatura

Após o processamento, o modelo produz uma distribuição de probabilidade sobre todos os tokens possíveis. O token seguinte é amostrado dessa distribuição.

**Temperatura** controla quão "sharp" (concentrada) ou "flat" (dispersa) é essa distribuição:

| Temperatura | Comportamento | Uso recomendado |
|---|---|---|
| 0.0 | Sempre o token mais provável (determinístico) | Extração de dados, classificação |
| 0.3-0.5 | Conservador, coerente | Análise, sumarização |
| 0.7-0.9 | Balanceado | Uso geral, conversação |
| 1.0+ | Criativo, imprevisível | Brainstorming, escrita criativa |

**Top-p (nucleus sampling):** alternativa à temperatura — só considera os tokens que somam p% da probabilidade. `top_p=0.9` significa "só considere os tokens mais prováveis até chegar em 90% da probabilidade total."

### Passo 5 — Geração autoregressiva

O modelo gera um token, adiciona ao contexto, e repete o processo. É por isso que modelos são **autoregressivos** — cada token gerado se torna parte do contexto para o próximo token.

Isso tem implicações:
- **Velocidade:** quanto mais longa a resposta, mais tempo leva (linear)
- **Erros se propagam:** se o modelo "entra no caminho errado", cada token seguinte reforça o erro
- **Por que chain-of-thought funciona:** forçar o modelo a raciocinar passo a passo cria tokens intermediários que guiam os próximos tokens na direção correta

### O que são "alucinações" mecanicamente

O modelo alucina quando a distribuição de probabilidade do próximo token leva a um token que parece plausível no contexto linguístico mas é factualmente incorreto. O modelo não "sabe" que está errando — ele está otimizando para coerência linguística, não para verdade factual.

Tipos comuns:
- **Confabulação:** inventa fatos plausíveis (datas, nomes, referências)
- **Raciocínio falho:** encadeamento de premissas corretas em conclusão errada
- **Extrapolação:** generaliza além dos dados de treinamento com confiança excessiva

**Mitigações:** RAG (ancora em fontes verificáveis), grounding (forçar citações), temperatura baixa em tarefas factuais, verificação externa.

---

## Aplicação Prática

**Configurações de parâmetros por caso de uso:**

```python
# Extração de dados estruturados
{"temperature": 0, "top_p": 1, "max_tokens": 500}

# Sumarização equilibrada
{"temperature": 0.3, "top_p": 0.9, "max_tokens": 1000}

# Conversa geral / assistente
{"temperature": 0.7, "top_p": 0.9, "max_tokens": 2000}

# Brainstorming / criatividade
{"temperature": 1.0, "top_p": 0.95, "max_tokens": 2000}
```

**Diagnóstico de problemas comuns:**

- Resposta sempre igual? → Temperatura muito baixa (ou 0)
- Resposta inconsistente demais? → Temperatura muito alta
- Modelo "esquece" instruções? → Instrução enterrada no meio do contexto (ver M01-03)
- Modelo inventa informações? → Temperatura baixa + RAG (M03)
- Modelo em loop? → Adicionar penalidade de repetição (`presence_penalty`, `frequency_penalty`)

---

## Conexões

- [01-03 — Transformers e Atenção](01-03-Transformers-e-Atencao.md): o processamento interno
- [02-02 — Engenharia de Prompt](../Modulo-02-NLP-Embeddings-Prompt/02-02-Engenharia-de-Prompt.md): como formatar input para melhores outputs
- [M03 — RAG](../Modulo-03-RAG-Memoria-Contextual/03-00-Visao-Geral.md): como injetar informação factual verificável
- [M04 — Agentes](../Modulo-04-Agentes-de-IA/04-00-Visao-Geral.md): como LLMs se tornam agentes com ferramentas

---

## Recursos Complementares

- **Andrej Karpathy — "Intro to Large Language Models"** (YouTube, 1h) — melhor overview técnico acessível disponível
- **Simon Willison — "What I think about when I think about LLMs"** (blog) — perspectiva de praticante experiente
- **CS50 AI — Semana de Language Models** — contexto da progressão do campo

---

## Auto-reflexão

1. Por que o mesmo prompt com temperatura 0 sempre gera a mesma resposta, mas com temperatura 0.7 pode gerar respostas diferentes?
2. Se o modelo alucina mecanicamente "otimizando coerência linguística", o que isso implica sobre como você deve verificar informações críticas?
3. Por que chain-of-thought ("pense passo a passo") melhora o raciocínio do modelo? Use o que aprendeu sobre geração autoregressiva para explicar.
4. Você paga por tokens de input E output. Qual estratégia de prompt minimiza custo sem sacrificar qualidade?

---

*[01-04] | Anterior: [01-03](01-03-Transformers-e-Atencao.md) | Próximo: [M02 — NLP e Embeddings](../Modulo-02-NLP-Embeddings-Prompt/02-00-Visao-Geral.md)*
*Referência externa: Andrej Karpathy; CS50 AI (Harvard)*

---

## Próximo Passo

### [→ M02 — NLP, Embeddings e Prompt](../Modulo-02-NLP-Embeddings-Prompt/02-00-Visao-Geral.md)

*Módulo concluído — clique acima para iniciar o próximo.*


---

*[← 01-03 — Transformers e Atenção](01-03-Transformers-e-Atencao.md) · [↑ M01 — Fundamentos Conceituais](01-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
