# [01-01] Como Máquinas Aprendem

## Por que isso importa?

Toda vez que você ajusta um prompt e o modelo "melhora", você está observando aprendizado de máquina em ação — mas ao contrário: você está ajustando o input, não o modelo. Entender como o modelo foi treinado muda completamente como você pensa em engenharia de prompt, fine-tuning e quando usar cada abordagem.

---

## Conceito Central

**Machine Learning é otimização de parâmetros por minimização de erro.**

Isso soa técnico, mas a ideia é simples: imagine que você quer ensinar uma criança a identificar gatos. Você mostra mil fotos com etiqueta ("gato" / "não gato"). A criança erra, você corrige, ela ajusta. Após mil correções, ela generaliza — consegue identificar gatos que nunca viu.

Um modelo de ML faz exatamente isso, mas com números:

1. **Dados de treinamento:** exemplos rotulados (input → output esperado)
2. **Parâmetros:** milhões de números ajustáveis (os "pesos" da rede)
3. **Função de perda:** métrica que mede o quanto o modelo errou
4. **Otimização:** algoritmo que ajusta os pesos para minimizar o erro

O que chamamos de "treinar um modelo" é esse processo de ajuste iterativo. Um LLM com 70 bilhões de parâmetros tem 70 bilhões de números que foram ajustados — cada um influenciando levemente como o modelo responde.

---

## Aprofundamento

### Os três tipos de aprendizado (o que importa para você)

**Aprendizado supervisionado:** você tem pares (input, output correto). O modelo aprende a mapear um no outro. Exemplo: classificar emails como spam/não-spam. **Relevante para:** fine-tuning de LLMs com exemplos de como você quer que ele responda.

**Aprendizado não-supervisionado:** você tem só inputs, sem etiquetas. O modelo encontra padrões sozinho. Exemplo: agrupar clientes por comportamento de compra. **Relevante para:** como os embeddings são gerados — o modelo aprende estrutura da linguagem sem ser explicitamente dito o que procurar.

**Aprendizado por reforço com feedback humano (RLHF):** humanos avaliam respostas do modelo, e ele aprende a produzir respostas mais bem avaliadas. **Relevante para:** como ChatGPT, Claude e outros LLMs foram "alinhados" — RLHF é o que faz o modelo preferir respostas úteis e seguras.

### A intuição de "generalização"

O perigo no ML é o **overfitting**: o modelo decora os dados de treinamento em vez de aprender padrões gerais. É como um estudante que memoriza as respostas de provas antigas sem entender o conteúdo — vai bem na prova antiga, mal na nova.

LLMs gigantes são treinados em proporções tão massivas de dados que generalizam de forma impressionante — mas ainda alucinam quando extrapolam além do que viram. Isso não é um bug aleatório: é uma consequência previsível de como ML funciona.

### O que "aprender" significa de verdade

O modelo não "entende" no sentido humano. Ele é um compressor de padrões estatísticos. Quando o GPT-4 responde sobre direito tributário, não é porque "sabe" direito — é porque viu tantos textos jurídicos que aprendeu padrões de como advogados escrevem. Isso explica:

- Por que ele inventa referências bibliográficas (padrão de citação sem verificar existência)
- Por que ele é melhor em inglês (mais dados de treinamento)
- Por que ele erra em matemática de nível médio mas acerta problemas difíceis (padrão vs. raciocínio)

---

## Aplicação Prática

**Exercício de calibração:** nas próximas 48h, quando o modelo alucinar ou cometer erro, tente classificar: foi falta de dados de treinamento no tema? Foi extrapolação além do aprendido? Foi um padrão errado que foi reforçado? Nomear o tipo de erro muda como você lida com ele.

**Checklist de decisão — quando usar o quê:**

- [ ] Preciso que o modelo responda de uma forma muito específica e consistente → **Fine-tuning**
- [ ] Preciso que o modelo acesse informações que não estavam no treinamento → **RAG** (M03)
- [ ] Preciso que o modelo execute ações no mundo real → **Agentes** (M04)
- [ ] Quero melhorar respostas sem mudar o modelo → **Engenharia de Prompt** (M02)

---

## Conexões

- [01-02 — Redes Neurais por Dentro](01-02-Redes-Neurais-Por-Dentro.md): como os parâmetros são organizados e ajustados
- [01-04 — LLMs do Token ao Output](01-04-LLMs-do-Token-ao-Output.md): como tudo isso se aplica a modelos de linguagem
- [M03 — RAG](../Modulo-03-RAG-Memoria-Contextual/03-00-Visao-Geral.md): alternativa ao fine-tuning para injetar conhecimento

---

## Recursos Complementares

- **3Blue1Brown — "But what is a neural network?"** (YouTube, 19 min) — melhor visualização disponível gratuitamente
- **CS50 AI — Semana de Machine Learning** — implementação prática em Python
- **"The Bitter Lesson" — Rich Sutton** (2019, ensaio gratuito) — por que escala bate algoritmos inteligentes

---

## Auto-reflexão

1. Qual é a diferença entre um modelo que "aprendeu" e um que "memorizou"? Por que isso importa ao usar LLMs?
2. Por que LLMs em inglês são melhores do que em português? O que isso implica para como você usa?
3. Se você pudesse fazer fine-tuning no seu LLM preferido com 100 exemplos de como quer que ele responda, o que você escolheria como exemplos?
4. Quando um modelo alucina uma referência bibliográfica, isso é um bug ou um comportamento previsível? Explique.

---

*[01-01] | Próximo: [01-02 — Redes Neurais por Dentro](01-02-Redes-Neurais-Por-Dentro.md)*
*Referência externa: CS50 AI (Harvard) — Semana ML*

---

## Próximo Passo

### [→ 01-02 — Redes Neurais por Dentro](01-02-Redes-Neurais-Por-Dentro.md)


---

*[← M01 — Fundamentos Conceituais](01-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
