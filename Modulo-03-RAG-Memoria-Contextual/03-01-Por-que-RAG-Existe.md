# [03-01] Por que RAG Existe — O Problema de Contexto

## Por que isso importa?

Antes de implementar RAG, entender por que ele foi inventado. O problema que RAG resolve é fundamental — e entendê-lo determina quando usar RAG, quando usar fine-tuning, e quando nenhum dos dois é a resposta.

---

## Conceito Central

**RAG (Retrieval-Augmented Generation) existe porque LLMs têm dois problemas estruturais: conhecimento estático com data de corte, e janela de contexto finita. RAG resolve ambos injetando informação relevante e atualizada no momento da query.**

---

## Aprofundamento

### O problema 1 — Conhecimento Estático

Um LLM é treinado uma vez e seu conhecimento "congela" na data de corte. O modelo não aprende com conversas, não atualiza automaticamente e não tem acesso a informações posteriores ao treinamento.

Consequências práticas:
- Pergunta sobre legislação aprovada em 2025 → o modelo de 2024 não sabe
- Pergunta sobre o estado atual de um projeto interno → o modelo nunca viu
- Pergunta sobre dados de um cliente específico → o modelo não tem acesso

A solução ingênua seria re-treinar o modelo com frequência. Impossível: treinar um LLM grande custa milhões de dólares e semanas de computação.

**RAG resolve:** ao invés de re-treinar, você recupera a informação relevante em tempo real e injeta no contexto antes de gerar a resposta.

### O problema 2 — Janela de Contexto Limitada

Mesmo com janelas de contexto grandes (200K tokens no Claude), você não pode simplesmente colocar toda a sua base de conhecimento no prompt. Uma empresa pode ter terabytes de documentação, contratos, e-mails e dados históricos. Não cabe no contexto.

Além disso, mesmo quando cabe, "lost in the middle" (M01-03) faz o modelo ignorar informação no meio de contextos longos.

**RAG resolve:** ao invés de injetar tudo, você recupera apenas os trechos mais relevantes para a query específica — tipicamente 3-10 chunks de 500 tokens cada.

### O problema 3 — Rastreabilidade

Um LLM sem grounding não pode citar fontes verificáveis. Ele "sabe" coisas mas não sabe de onde vem esse conhecimento.

**RAG resolve:** cada trecho recuperado tem origem conhecida (documento, URL, timestamp). A resposta pode citar fontes verificáveis.

### Por que não fine-tuning?

Fine-tuning e RAG resolvem problemas diferentes:

| Aspecto | Fine-tuning | RAG |
|---|---|---|
| **Custo** | Alto (GPU, tempo) | Baixo (inference only) |
| **Atualização** | Re-treinar para atualizar | Atualizar o banco de documentos |
| **Rastreabilidade** | Não — o modelo "absorveu" | Sim — cita a fonte |
| **Dados necessários** | Centenas de exemplos curados | Documentos brutos |
| **Ideal para** | Estilo/formato/persona | Conhecimento atualizado e verificável |

**Regra:** use fine-tuning quando quer mudar *como* o modelo responde. Use RAG quando quer mudar *o que* o modelo sabe.

### O pipeline conceitual do RAG

```
[Pergunta do usuário]
       ↓
[Embedding da query]  ←── Mesmo modelo de embedding do banco
       ↓
[Busca por similaridade no banco vetorial]
       ↓
[Top-K chunks recuperados]
       ↓
[Prompt enriquecido: "Usando os trechos abaixo, responda: [query]\n\nTrechos: [chunks]"]
       ↓
[LLM gera resposta fundamentada nos chunks]
       ↓
[Resposta com citações verificáveis]
```

A beleza do RAG: o LLM não precisa "saber" a resposta. Ele só precisa sintetizar o que foi recuperado.

---

## Aplicação Prática

**Quando RAG é a resposta certa:**

- [ ] Sua base de conhecimento muda frequentemente (docs atualizados, logs, emails)
- [ ] Você precisa de rastreabilidade (citar fontes)
- [ ] Você tem muito conteúdo (mais do que cabe no contexto)
- [ ] Você quer evitar alucinações factuais
- [ ] Privacidade requer que os dados não entrem no treinamento do modelo

**Quando RAG não é a solução:**

- [ ] O problema é o modelo não saber um *estilo* de resposta → fine-tuning
- [ ] A informação cabe inteiramente no contexto e é estática → context stuffing simples
- [ ] O modelo precisa raciocinar sobre a informação, não apenas recuperá-la → agentes (M04)

---

## Conexões

- [02-01 — Embeddings](../Modulo-02-NLP-Embeddings-Prompt/02-01-Embeddings-e-Espaco-Vetorial.md): fundação técnica do RAG
- [03-02 — Bancos Vetoriais](03-02-Bancos-Vetoriais-na-Pratica.md): onde os embeddings são armazenados
- [03-03 — Pipelines RAG](03-03-Pipelines-RAG-do-Zero.md): implementação completa
- [04-04 — Memória de Agentes](../Modulo-04-Agentes-de-IA/04-04-Memoria-e-Persistencia.md): RAG como memória de longo prazo para agentes

---

## Recursos Complementares

- **Lewis et al. — "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"** (paper original, 2020)
- **LangChain RAG tutorial**: [python.langchain.com/docs/use_cases/question_answering](https://python.langchain.com/docs/use_cases/question_answering)
- **Alura — Flash Skills: RAG e Agentes de IA** (2h) — implementação prática guiada

---

## Auto-reflexão

1. Por que re-treinar o modelo não é a solução para dados atualizados?
2. Uma empresa tem 500.000 contratos jurídicos em PDF. Qual solução — fine-tuning ou RAG — você escolheria para um chatbot interno que responde perguntas sobre os contratos? Por quê?
3. Se RAG injeta apenas 5-10 chunks no contexto, o que acontece quando a resposta correta requer sintetizar informações de 50 documentos diferentes?
4. Qual é a diferença entre "o modelo não sabe" e "o modelo alucina"? Como RAG mitiga cada caso?

---

*[03-01] | Anterior: [03-00](03-00-Visao-Geral.md) | Próximo: [03-02](03-02-Bancos-Vetoriais-na-Pratica.md)*
*Referência: RAG paper (Lewis et al., 2020); LangChain docs*

---

## Próximo Passo

### [→ 03-02 — Bancos Vetoriais na Prática](03-02-Bancos-Vetoriais-na-Pratica.md)


---

*[← M03 — RAG e Memória Contextual](03-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
