# [02-02] Engenharia de Prompt

## Por que isso importa?

Prompt engineering não é "arte" — é engenharia com princípios verificáveis. Praticantes avançados que formalizam o que sabem intuitivamente ganham consistência, conseguem ensinar outros e param de "testar no escuro" quando algo não funciona.

---

## Conceito Central

**Um prompt é o contexto completo que o modelo recebe antes de gerar o output. Engenharia de prompt é estruturar esse contexto para maximizar a probabilidade de o output ser o desejado.**

O modelo não lê intenções — processa tokens. A qualidade do output é uma função direta da qualidade do input.

---

## Aprofundamento

### Os quatro componentes de um prompt bem estruturado

**1. Role / Persona** — quem o modelo deve "ser"
Estabelece o tom, o vocabulário e a perspectiva. Mais eficaz do que parece porque molda toda a distribuição de probabilidade das respostas.

```
Você é um especialista em direito tributário brasileiro com 20 anos de experiência
assessorando pequenas empresas. Sua comunicação é direta, sem jargão desnecessário,
e você sempre aponta riscos práticos além da teoria.
```

**2. Contexto / Background** — o que o modelo precisa saber
Informações que o modelo não tem (dados atuais, detalhes específicos do problema, documentos relevantes). Quanto mais preciso, menos o modelo precisa "inventar".

```
Contexto: Empresa X, optante pelo Simples Nacional, faturamento 2024 de R$800.000,
setor de serviços de TI. Consulta sobre enquadramento para 2025.
```

**3. Instrução / Tarefa** — o que fazer
A instrução deve ser:
- Específica (evitar "analise" quando "liste os 3 principais riscos" é mais preciso)
- No imperativo ("liste", "calcule", "resuma" — não "você poderia listar?")
- Com critérios de qualidade quando possível ("em até 200 palavras", "em formato JSON", "com exemplos para cada ponto")

**4. Output Format** — como deve ser o output
Especificar o formato explicitamente elimina ambiguidade. JSON, markdown, lista numerada, tabela — diga qual.

```
Output: JSON com campos: {"risco": string, "severidade": "alta|média|baixa", "recomendacao": string}
Máximo 3 riscos. Sem texto adicional além do JSON.
```

### Zero-shot vs. Few-shot vs. Chain-of-Thought

**Zero-shot:** só instrução, sem exemplos. Funciona bem para tarefas comuns. Frágil para tarefas específicas ou outputs estruturados.

**Few-shot:** instrução + 2-5 exemplos de input→output. Dramaticamente mais eficaz para outputs estruturados, classificação, e tarefas específicas do domínio.

```
Classifique o sentimento:
Input: "O produto chegou rápido e funciona perfeitamente" → Positivo
Input: "Demorou 3 semanas e veio errado" → Negativo
Input: "É ok, mas esperava mais pelo preço" → [a classificar]
```

**Chain-of-Thought (CoT):** força o modelo a raciocinar em etapas antes de responder. Aumenta qualidade em tarefas que requerem raciocínio. Mais tokens = mais custo.

```
Antes de responder, pense passo a passo:
1. Quais são os dados disponíveis?
2. Qual é o objetivo principal?
3. Quais são os riscos?
4. Qual é a recomendação final?

Mostre seu raciocínio para cada etapa.
```

### System prompt vs. User prompt vs. Assistant prompt

A maioria das APIs de LLM tem três roles:

| Role | Propósito | Persistência |
|---|---|---|
| **System** | Instruções globais, persona, restrições | Toda a conversa |
| **User** | Input do usuário | Turno atual |
| **Assistant** | Output anterior do modelo | Histórico de conversa |

**Regra de ouro:** coloque instruções permanentes no system prompt. Dados e perguntas no user prompt. Nunca coloque credenciais ou informações sensíveis em nenhum dos dois.

### Structured outputs — eliminando ambiguidade de formato

Modelos como GPT-4o suportam "structured outputs" nativamente — você define um schema JSON e o modelo garante que o output obedece ao schema. Zero parsing errors, zero formatação incorreta.

```python
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Extraia os dados da nota fiscal: ..."}],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "nota_fiscal",
            "schema": {
                "type": "object",
                "properties": {
                    "valor": {"type": "number"},
                    "data": {"type": "string"},
                    "fornecedor": {"type": "string"}
                },
                "required": ["valor", "data", "fornecedor"]
            }
        }
    }
)
```

---

## Aplicação Prática

**Template de prompt para tarefas de extração (alta precisão necessária):**

```
System: Você é um extrator de dados preciso. Extraia apenas o que está explicitamente no texto.
Nunca infira ou complete informações ausentes. Se um campo não estiver presente, retorne null.

User: Texto: [TEXTO]

Extraia em JSON:
{
  "nome": string | null,
  "cpf": string | null,
  "valor": number | null,
  "data": string (YYYY-MM-DD) | null
}
```

**Debugging de prompts — checklist quando o output é ruim:**

- [ ] O modelo tem todas as informações necessárias no contexto?
- [ ] A instrução é específica o suficiente (evite "analise", prefira "liste")
- [ ] O formato de output está explicitamente definido?
- [ ] Está usando few-shot para tarefas específicas de domínio?
- [ ] A temperatura está correta para a tarefa?
- [ ] A instrução está no system prompt (não enterrada no user)?

---

## Conexões

- [02-03 — Padrões Avançados de Prompt](02-03-Padroes-Avancados-de-Prompt.md): ReAct, Self-Consistency, ToT
- [01-04 — LLMs do Token ao Output](../Modulo-01-Fundamentos-Conceituais/01-04-LLMs-do-Token-ao-Output.md): por que temperatura afeta o output
- [M04-01 — O que é um Agente](../Modulo-04-Agentes-de-IA/04-01-O-que-e-um-Agente.md): agentes são sistemas de prompts encadeados

---

## Recursos Complementares

- **Anthropic Prompt Engineering Guide**: [docs.anthropic.com](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — melhor guia para Claude
- **OpenAI Prompt Engineering**: [platform.openai.com/docs/guides/prompt-engineering](https://platform.openai.com/docs/guides/prompt-engineering)
- **DAIR.AI Prompt Engineering Guide**: [promptingguide.ai](https://www.promptingguide.ai) — abrangente, gratuito

---

## Auto-reflexão

1. Qual é a diferença entre zero-shot e few-shot? Quando você escolheria um sobre o outro?
2. Por que especificar o formato de output explicitamente ("responda em JSON com campos X, Y, Z") é mais eficaz do que "responda de forma estruturada"?
3. Você tem um prompt que funciona bem às vezes e mal outras. O que você checaria primeiro?
4. Por que chain-of-thought custa mais tokens? Quando o custo extra vale a pena?

---

*[02-02] | Anterior: [02-01](02-01-Embeddings-e-Espaco-Vetorial.md) | Próximo: [02-03](02-03-Padroes-Avancados-de-Prompt.md)*
*Referências: Anthropic Prompt Guide; DAIR.AI Prompt Engineering*
