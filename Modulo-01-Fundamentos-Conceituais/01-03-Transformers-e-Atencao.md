# [01-03] Transformers e Atenção

## Por que isso importa?

O Transformer é a arquitetura que está por trás de todo LLM moderno — GPT, Claude, Gemini, LLaMA. Entender sua lógica central (atenção) muda completamente como você pensa em limites de contexto, por que o modelo "esquece" o início de conversas longas, e como o RAG compensa essas limitações.

---

## Conceito Central

**O mecanismo de atenção permite que cada token "olhe" para todos os outros tokens e decida o quanto cada um é relevante para entender o seu próprio significado.**

Antes dos Transformers (2017), modelos de linguagem processavam texto sequencialmente — palavra por palavra, sem olhar para frente e com memória limitada do passado. O Transformer quebrou isso: processa todos os tokens em paralelo, e cada token pode "atender" a qualquer outro token na sequência.

---

## Aprofundamento

### O problema que o Transformer resolveu

Considere a frase: *"O banco onde eu sentei estava perto do banco onde eu guardei meu dinheiro."*

A palavra "banco" aparece duas vezes com significados diferentes. Como o modelo sabe qual é qual?

Antes dos Transformers: com dificuldade. Modelos RNN processavam sequencialmente e frequentemente "esqueciam" contexto distante.

Com atenção: o segundo "banco" pode "olhar" para o contexto inteiro — "guardei meu dinheiro" — e decidir que este é o banco financeiro. O primeiro pode olhar para "sentei" e decidir que é o banco de praça.

### Atenção — a mecânica simplificada

Para cada token, o mecanismo de atenção calcula:
1. **Query:** "O que eu estou procurando?"
2. **Key:** "O que cada outro token tem para oferecer?"
3. **Value:** "Qual informação extraio de cada token relevante?"

O resultado é uma média ponderada: tokens muito relevantes contribuem mais para o entendimento do token atual. Isso acontece em paralelo para todos os tokens simultaneamente.

**Multi-head attention:** na prática, esse processo ocorre em paralelo em múltiplas "cabeças" de atenção, cada uma aprendendo a atender a aspectos diferentes (sintaxe, semântica, relações de longo alcance, etc.).

### Janela de contexto — o limite que você já sentiu

**Context window** é quantos tokens o modelo processa de uma vez. Como a atenção calcula relações entre todos os pares de tokens, o custo computacional cresce quadraticamente com o tamanho do contexto.

| Modelo | Contexto | Equivalente aproximado |
|---|---|---|
| GPT-3 | 4.096 tokens | ~3.000 palavras |
| GPT-4 Turbo | 128.000 tokens | ~100.000 palavras (~300 páginas) |
| Claude 3.5 Sonnet | 200.000 tokens | ~150.000 palavras (~450 páginas) |
| Gemini 1.5 Pro | 1.000.000 tokens | ~750.000 palavras (~2.000 páginas) |

**Por que o modelo "esquece" o início:** não é que esquece — é que em contextos muito longos, a atenção se dilui. O modelo tende a ponderar mais o início e o fim da conversa do que o meio ("lost in the middle"). Isso é documentado empiricamente e explica por que instruções importantes devem estar no início ou no final do prompt, não no meio de um documento longo.

### Positional encoding — a ordem importa

Como a atenção não tem noção inerente de sequência (processa tudo em paralelo), o Transformer injeta informação posicional em cada token — uma espécie de "endereço" na sequência. Isso é o que permite ao modelo distinguir "gato mordeu cachorro" de "cachorro mordeu gato", mesmo que processe os tokens em paralelo.

### Por que Transformers dominaram

Antes: RNNs (LSTM, GRU) processavam sequencialmente — boas mas lentas para treinar em paralelo.
Depois: Transformers processam tudo em paralelo → aproveitam GPUs modernas ao máximo → escalação dramática do treinamento.

O paper original "Attention Is All You Need" (2017, Google) é um dos mais citados da história da IA — não pela ideia radicalmente nova, mas pela elegância da implementação que permitiu escalar.

---

## Aplicação Prática

**Como usar o contexto de forma inteligente:**

- [ ] Instruções críticas: sempre no início do prompt (system prompt), nunca enterradas no meio
- [ ] Documentos longos: se o conteúdo importante está no meio, divida em chunks menores (RAG resolve isso — M03)
- [ ] Contexto de conversa: se a conversa está muito longa, resuma as partes antigas explicitamente
- [ ] Múltiplos documentos: ordene do menos para o mais importante — o modelo tende a ponderar mais o final

**Regra prática:** se você está colocando mais do que 50% da janela de contexto com documentos, considere RAG.

---

## Conexões

- [01-02 — Redes Neurais por Dentro](01-02-Redes-Neurais-Por-Dentro.md): os Transformers são redes neurais com arquitetura específica
- [01-04 — LLMs do Token ao Output](01-04-LLMs-do-Token-ao-Output.md): como o Transformer vira um modelo de linguagem
- [02-01 — Embeddings](../Modulo-02-NLP-Embeddings-Prompt/02-01-Embeddings-e-Espaco-Vetorial.md): Transformers geram embeddings
- [M03 — RAG](../Modulo-03-RAG-Memoria-Contextual/03-00-Visao-Geral.md): solução para a limitação da janela de contexto

---

## Recursos Complementares

- **"Attention Is All You Need"** — paper original, 2017 (leitura opcional mas histórica)
- **3Blue1Brown — "Attention in transformers, visually explained"** (YouTube) — melhor visualização disponível
- **CS50 AI — Semana de NLP** — contexto de como Transformers se encaixam em NLP aplicado
- **Andrej Karpathy — "Let's build GPT from scratch"** (YouTube, 2h) — implementa um Transformer do zero

---

## Auto-reflexão

1. Por que o custo de processar um contexto de 128.000 tokens é muito maior do que processar 4.000 tokens? (dica: pense em "todos os pares de tokens")
2. Se o modelo tende a prestar mais atenção ao início e fim do contexto, como você reorganizaria um prompt que hoje enterra as instruções no meio?
3. Por que Transformers treinam mais rápido que RNNs, mesmo sendo tecnicamente mais complexos?
4. O que acontece quando você passa mais tokens do que a janela de contexto suporta?

---

*[01-03] | Anterior: [01-02](01-02-Redes-Neurais-Por-Dentro.md) | Próximo: [01-04](01-04-LLMs-do-Token-ao-Output.md)*
*Referência externa: CS50 AI (Harvard) — Semana NLP; "Attention Is All You Need" (2017)*
