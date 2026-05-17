# [02-04] Limitações e Alucinações

## Por que isso importa?

Um praticante que não entende as limitações reais dos LLMs vai construir sistemas frágeis e vai culpar o modelo quando na verdade é design. Um que entende as limitações constrói mitigações antes que o problema apareça em produção.

---

## Conceito Central

**As limitações dos LLMs não são bugs — são consequências arquiteturais previsíveis. Cada limitação tem causa conhecida e mitigação específica.**

---

## Aprofundamento

### As 6 limitações fundamentais

**1. Corte de conhecimento (knowledge cutoff)**
O modelo só sabe o que estava nos dados de treinamento, até a data de corte. Para GPT-4o: início de 2024. Para Claude 3.5: início de 2024.

*Mitigação:* RAG com fontes atualizadas, busca web via ferramentas, function calling para dados em tempo real.

**2. Alucinação factual**
O modelo gera informações plausíveis mas incorretas — datas, nomes, citações, números. A causa é a otimização para coerência linguística, não para veracidade factual.

*Tipolgia prática:*
- Confabulação de citações (o clássico "inventa referência bibliográfica")
- Extrapolação de dados (extrapola números reais para outros contextos)
- Falsa memória contextual (mistura informações de documentos diferentes no contexto)

*Mitigação:* RAG com fontes verificáveis, temperatura baixa para tarefas factuais, pedir ao modelo que cite a fonte de cada afirmação, verificação externa para informações críticas.

**3. Raciocínio matemático e lógico frágil**
LLMs são modelos de linguagem, não calculadoras. Erros em aritmética de múltiplos passos, lógica formal e problemas que requerem precisão são comuns.

*Por que:* o modelo "lembra" que 7×8=56 de treinamento, mas cálculos novos são probabilísticos.

*Mitigação:* function calling para calculadora, Python interpreter, Code Interpreter (ChatGPT). Nunca confie em matemática do LLM sem verificação.

**4. Sensibilidade ao framing do prompt**
Reformular a mesma pergunta pode gerar respostas muito diferentes — ou até opostas. O modelo é sensível a pistas linguísticas que ativam diferentes "padrões" do treinamento.

*Exemplos:*
- "Quais são os problemas desta abordagem?" → resposta crítica
- "Quais são os benefícios e limitações desta abordagem?" → resposta mais balanceada
- "Como um expert criticaria esta abordagem?" → resposta com vocabulário mais técnico

*Mitigação:* teste prompts com múltiplas formulações; use few-shot para ancorar o comportamento; self-consistency para respostas críticas.

**5. Propagação de erro em cadeia**
Em geração autoregressiva (M01-04), um erro inicial se propaga e amplifica. Um erro na linha 3 de um raciocínio compromete tudo que vem depois.

*Mitigação:* chain-of-thought explícito para verificação passo a passo; ReAct com observações externas que "âncora" o raciocínio em realidade; dividir problemas complexos em sub-problemas menores.

**6. Inconsistência entre sessões**
O modelo não tem memória entre sessões (por padrão). A mesma pergunta pode ter respostas levemente diferentes em sessões diferentes, mesmo com temperatura 0, devido a variações de infraestrutura.

*Mitigação:* memória explícita (RAG com histórico de interações), estado persistente via banco de dados, identificadores de sessão em sistemas de produção.

### Limitações de contexto e o "lost in the middle"

Experimentos documentados mostram que LLMs têm pior desempenho em recuperar informações do meio de documentos longos — tendem a focar mais no início e no final.

*Implicação prática:* se você tem 10 documentos relevantes e injeta todos no contexto, os do meio são menos "lembrados". Estratégias:
- RAG com k menor e threshold de similaridade alto (qualidade > quantidade)
- Ordenar do mais relevante para o menos relevante
- Usar reranking (um segundo modelo avalia a relevância antes de injetar)

### O que LLMs fazem bem (para contrabalançar)

| Forte | Fraco |
|---|---|
| Compreensão e resumo de texto | Aritmética precisa |
| Geração de texto fluente e coerente | Raciocínio lógico formal |
| Tradução e adaptação de tom/estilo | Informações em tempo real |
| Classificação e extração estruturada | Consistência entre sessões |
| Código em linguagens populares | Contagem de caracteres/tokens |
| Brainstorming e variações criativas | Verificação de fatos |

---

## Aplicação Prática

**Framework de decisão — quando confiar vs. verificar:**

```
Alta confiança (usar sem verificação externa):
  ✓ Texto bem formado e coerente
  ✓ Código em linguagens populares (Python, JS) — revisar, não confiar cegamente
  ✓ Resumos de documentos fornecidos no contexto
  ✓ Classificação com exemplos few-shot

Verificação necessária:
  ! Dados numéricos específicos (datas, estatísticas, valores)
  ! Referências bibliográficas
  ! Afirmações factuais sobre eventos recentes
  ! Cálculos matemáticos de múltiplos passos
  ! Informações jurídicas ou médicas críticas

Nunca sem RAG/ferramenta externa:
  ✗ Dados em tempo real (preços, temperatura, notícias)
  ✗ Informações pós-cutoff do modelo
  ✗ Consultas a bancos de dados específicos
```

---

## Conexões

- [01-04 — LLMs do Token ao Output](../Modulo-01-Fundamentos-Conceituais/01-04-LLMs-do-Token-ao-Output.md): por que alucinações acontecem mecanicamente
- [M03 — RAG](../Modulo-03-RAG-Memoria-Contextual/03-00-Visao-Geral.md): a principal mitigação para limitações factuais
- [M04 — Agentes](../Modulo-04-Agentes-de-IA/04-00-Visao-Geral.md): ferramentas que compensam limitações matemáticas e de tempo real

---

## Recursos Complementares

- **"Lost in the Middle: How Language Models Use Long Contexts"** (paper, 2023) — evidência do problema de atenção em contextos longos
- **Anthropic's Claude model card** — documentação honesta de limitações de um modelo de produção
- **"Hallucination Is Inevitable"** (paper, 2024) — por que não é possível eliminar alucinações completamente

---

## Auto-reflexão

1. Você tem um sistema que usa LLM para verificar dados financeiros. Que mitigações você implementaria antes de ir para produção?
2. Por que um modelo com temperatura 0 ainda pode alucinar?
3. "Lost in the middle" afeta mais um documento de 500 tokens ou de 50.000 tokens? Por quê?
4. Se você precisasse escolher: sistema com RAG que às vezes recupera chunks irrelevantes, ou sistema sem RAG que alucina com mais frequência — qual você preferiria e por quê?

---

*[02-04] | Anterior: [02-03](02-03-Padroes-Avancados-de-Prompt.md) | Próximo: [M03 — RAG](../Modulo-03-RAG-Memoria-Contextual/03-00-Visao-Geral.md)*
*Referências: "Lost in the Middle" paper (2023); Anthropic model card*
