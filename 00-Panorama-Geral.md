# Panorama Geral — Imersão em Agentes de IA

## Por que esta imersão existe

Existe um gap entre *usar* ferramentas de IA (ChatGPT, Cursor, APIs) e *entender* o que está acontecendo por baixo. Esta imersão fecha esse gap: do conceito de token ao agente operacional conectado via MCP.

**Não é um curso de programação.** É uma imersão conceitual com código. Você sairá sabendo explicar — e construir — cada camada.

---

## O que você vai saber ao final

- Por que LLMs "alucinam" e quando isso é previsível
- Como embeddings e RAG resolvem o problema de contexto
- Como um agente decide o que fazer (ReAct, toolcalling)
- Como orquestrar fluxos complexos com LangGraph
- Como sistemas multiagente se comunicam
- O que é MCP e por que ele está mudando o ecossistema
- Como usar o Cursor como plataforma de automação real

---

## A metáfora que guia tudo

```
Humano  →  Percebe  →  Raciocina  →  Age  →  Observa resultado
Agente  →  Recebe   →  Planeja    →  Chama tools → Observa → Repete
```

Cada módulo aprofunda uma parte desse ciclo.

---

## Mapa de dependências entre módulos

```
M01 (Fundamentos)
└── M02 (Embeddings/Prompt)
    └── M03 (RAG)
        └── M04 (Agentes)
            ├── M05 (LangGraph)
            │   └── M06 (Multi-agente)
            └── M07 (MCP)
                └── M08 (Cursor)
```

---

## Glossário inicial

| Termo | Definição rápida |
|-------|------------------|
| Token | Unidade mínima de texto que o modelo processa |
| Embedding | Representação vetorial de significado |
| RAG | Retrieval-Augmented Generation: buscar antes de gerar |
| Agent | Sistema que decide e age em ciclos |
| Tool | Função que o agente pode chamar |
| LangGraph | Framework para grafos de controle de agentes |
| MCP | Model Context Protocol — padrão de conexão agente↔ferramenta |

---

## Tempo estimado por módulo

| Módulo | Leitura | Prática | Total |
|--------|---------|---------|-------|
| M01 | 2h | 2h | ~4h |
| M02 | 2h | 2h | ~4h |
| M03 | 2h | 2h | ~4h |
| M04 | 2h | 2h | ~4h |
| M05 | 2h | 2h | ~4h |
| M06 | 2h | 2h | ~4h |
| M07 | 2h | 2h | ~4h |
| M08 | 2h | 2h | ~4h |
| **Total** | **16h** | **16h** | **~32h** |

---

## Auto-diagnóstico de entrada

Responda honestamente antes de começar:

1. Consigo explicar como um transformer calcula atenção? → se não, M01 é essencial
2. Sei o que é um embedding e por que é vetorial? → se não, M02 primeiro
3. Já implementei RAG com chunking + retrieval? → se não, M03 antes de M04
4. Entendo por que LangGraph existe em vez de chains simples? → se não, M04+M05

---

## Próximo Passo

### [→ M01 — Fundamentos Conceituais](Modulo-01-Fundamentos-Conceituais/01-00-Visao-Geral.md)


---

*[← Índice do Curso](README.md) · [🏠 Índice do Curso](README.md)*
