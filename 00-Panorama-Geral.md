# [IA-00] Panorama Geral — Imersão em Agentes de IA

## Por que este curso existe?

Milhões de pessoas já usam LLMs, Cursor, ChatGPT e ferramentas de IA no dia a dia. Mas a grande maioria opera no modo "caixa preta": sabe que funciona, não sabe por quê. Isso cria um teto invisível — na hora de depurar, personalizar, arquitetar ou explicar o que está fazendo, o praticante sem fundamentos improvisa onde deveria raciocinar.

Este curso é para quem já ultrapassou o nível "usuário avançado" e quer o vocabulário, os modelos mentais e o stack técnico para operar como construtor — não apenas como usuário.

---

## O que você vai conseguir fazer ao final

- Explicar por que um LLM alucina e o que fazer a respeito
- Construir um pipeline RAG funcional do zero
- Criar um agente com ferramentas, memória e controle de estado
- Orquestrar múltiplos agentes com LangGraph
- Implementar e consumir servidores MCP
- Usar o Cursor IDE como plataforma de automação inteligente
- Ter vocabulário para ler papers, documentação e artigos técnicos sem barreira

---

## A Metáfora do Curso

Pense numa cozinha profissional. Um cliente de restaurante sabe o que é um prato bom. Um cozinheiro amador sabe seguir uma receita. Um chef sabe por que cada técnica funciona, consegue improvisar quando o ingrediente falta e ensina outros.

Este curso te move da posição de "cozinheiro avançado que segue receitas" para "chef que entende a química do que está fazendo."

---

## Mapa de Dependências entre Módulos

```
M01 ──→ M02 ──→ M03 ──→ M04 ──→ M05 ──→ M06
                              ↓              ↓
                             M07 ←─────── M08
```

- **M01 → M02:** precisa entender embeddings e Transformers antes de entender NLP aplicado
- **M02 → M03:** RAG usa embeddings — sem M02, RAG é mágica
- **M03 → M04:** agentes usam RAG como ferramenta — integração natural
- **M04 → M05:** fluxos complexos pressupõem agentes simples funcionando
- **M05 → M06:** multiagente é orquestração de múltiplos M04-M05
- **M04 → M07:** MCP é o protocolo que conecta agentes a ferramentas — extensão de M04
- **M07 → M08:** Cursor usa MCP nativamente — M08 aplica tudo no contexto real

---

## Glossário Inicial — Termos que vão aparecer muito

| Termo | O que é | Onde aprofundar |
|---|---|---|
| LLM | Modelo de linguagem grande — prediz o próximo token | M01-02 |
| Token | Unidade de texto que o modelo processa | M01-03 |
| Embedding | Representação numérica de texto em espaço vetorial | M02-01 |
| RAG | Retrieval-Augmented Generation — memória externa para LLMs | M03 |
| Agente | LLM com acesso a ferramentas e capacidade de agir | M04 |
| ReAct | Padrão Reason + Act para agentes | M04-02 |
| LangChain | Framework para construção de pipelines LLM | M04-03 |
| LangGraph | Framework para fluxos de agentes com estado | M05 |
| MCP | Model Context Protocol — protocolo de integração de ferramentas | M07 |
| Transformer | Arquitetura de rede neural base dos LLMs modernos | M01-03 |
| Backpropagation | Algoritmo de aprendizagem de redes neurais | M01-02 |

---

## Estimativa de Tempo

| Módulo | Leitura | Prática recomendada | Total |
|---|---|---|---|
| M01 | 2-3h | 1-2h (CS50 AI semanas NLP/NN) | ~4h |
| M02 | 2h | 1h | ~3h |
| M03 | 2h | 2h (montar pipeline RAG) | ~4h |
| M04 | 2-3h | 3h (agente com ferramentas) | ~5h |
| M05 | 2-3h | 3h (fluxo LangGraph) | ~5h |
| M06 | 2h | 2h | ~4h |
| M07 | 2h | 2h (servidor MCP) | ~4h |
| M08 | 1-2h | ongoing | ~3h |
| **Total** | | | **~32h** |

---

## Antes de Começar — Auto-diagnóstico

Responda honestamente para calibrar onde você está:

1. Você consegue explicar o que é um token sem consultar nada?
2. Você sabe a diferença entre fine-tuning e RAG?
3. Você já criou um agente com ferramentas (não apenas usou um)?
4. Você sabe o que acontece dentro do Cursor quando você usa um MCP?
5. Você consegue depurar um agente quando ele fica em loop?

**0-1 sim:** comece pelo M01 sem pular.
**2-3 sim:** leia M01-02 em modo rápido (foque nas seções "Conceito Central"), comece devagar no M03.
**4-5 sim:** vá direto ao M05 e use os módulos anteriores como referência quando precisar.

---

## Próximo Passo

### [→ M01 — Fundamentos Conceituais](Modulo-01-Fundamentos-Conceituais/01-00-Visao-Geral.md)


---

*[← Índice do Curso](README.md)*
