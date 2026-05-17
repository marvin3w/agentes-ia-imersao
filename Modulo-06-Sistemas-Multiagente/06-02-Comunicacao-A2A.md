# [06-02] Comunicação A2A — Como Agentes Falam Entre Si

## Por que isso importa?

Em sistemas multiagente simples (LangGraph com estado compartilhado), comunicação é implícita: agentes leem e escrevem no mesmo objeto de estado. Mas à medida que sistemas crescem — agentes em microserviços diferentes, desenvolvidos por times diferentes, usando frameworks diferentes — é preciso um protocolo de comunicação explícito e padronizado.

---

## Conceito Central

**A2A (Agent-to-Agent) é o problema de como agentes descobrem uns aos outros, entendem o que cada um faz, e trocam tarefas e resultados de forma interoperável. Google, OpenAI e Anthropic lançaram propostas diferentes em 2025 — e o espaço está convergindo para padrões abertos.**

---

## Aprofundamento

### Os três problemas de comunicação entre agentes

**1. Descoberta (Discovery)**
Como o agente A sabe que o agente B existe e o que ele faz?
- Registro central (como um "diretório" de agentes)
- Descrições em formato padronizado (`agent_card`)
- Discovery por arquivo conhecido (`/.well-known/agent.json`)

**2. Invocação (Invocation)**
Como A chama B e passa uma tarefa?
- HTTP REST (síncrono)
- Mensagens assíncronas (filas: Kafka, RabbitMQ)
- Streaming (SSE, WebSockets) para tarefas longas

**3. Contexto (Context Transfer)**
Como B entende o contexto que A passou?
- Histórico de mensagens estruturado
- Artefatos (dados, arquivos, resultados parciais)
- Estado de sessão

### Google A2A Protocol (2025)

O Google DeepMind lançou o protocolo A2A como standard aberto:

**Agent Card** — descrição padronizada de um agente:
```json
{
  "name": "Research Agent",
  "description": "Agente especializado em pesquisa web e análise de documentos",
  "version": "1.0.0",
  "url": "https://my-agent.company.com",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false,
    "stateTransitionHistory": true
  },
  "skills": [
    {
      "id": "web_research",
      "name": "Pesquisa Web",
      "description": "Busca e sumariza informações da internet",
      "inputModes": ["text"],
      "outputModes": ["text", "data"]
    }
  ],
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text"]
}
```

**Task Object** — unidade de trabalho trocada entre agentes:
```json
{
  "id": "task_abc123",
  "sessionId": "session_xyz",
  "message": {
    "role": "user",
    "parts": [
      {
        "type": "text",
        "text": "Pesquise as últimas notícias sobre LangGraph e resuma os 3 principais updates"
      }
    ]
  }
}
```

**Estados de uma task:**
```
submitted → working → input-required → working → completed
                                               → failed
                                               → canceled
```

**Implementação básica com Python SDK:**
```python
from a2a.client import A2AClient
from a2a.types import TaskSendParams, Message, TextPart

# Cliente para chamar outro agente
client = A2AClient(url="http://research-agent:8000")

# Enviar task
task = await client.send_task(
    TaskSendParams(
        message=Message(
            role="user",
            parts=[TextPart(text="Pesquise LangGraph updates 2025")]
        )
    )
)

# Aguardar resultado
result = await client.get_task(task.id)
print(result.result.parts[0].text)
```

### OpenAI Swarm (2024) — padrão leve

OpenAI lançou o Swarm como referência educacional para handoffs entre agentes:

```python
from swarm import Swarm, Agent

client = Swarm()

# Definir agentes
research_agent = Agent(
    name="Research Agent",
    instructions="Você pesquisa informações. Quando tiver os dados, passe para o analyst_agent.",
    functions=[search_web, search_documents]
)

analyst_agent = Agent(
    name="Analyst Agent",
    instructions="Você analisa dados e gera insights. Quando finalizar, passe para o writer_agent.",
    functions=[run_analysis, create_chart]
)

def transfer_to_analyst():
    """Transfere para o Analyst Agent quando a pesquisa estiver completa."""
    return analyst_agent

# Adicionar a função de transferência ao research_agent
research_agent.functions.append(transfer_to_analyst)

# Executar
response = client.run(
    agent=research_agent,
    messages=[{"role": "user", "content": "Analise as vendas do Q3 e recomende ações"}]
)
```

### Anthropic — Model Context Protocol (MCP)

O MCP (M07 detalhará) resolve o outro lado: como agentes se conectam a **ferramentas e recursos**, não a outros agentes. É complementar ao A2A:

```
A2A: Agente ↔ Agente (comunicação entre sistemas de IA)
MCP: Agente ↔ Ferramenta/Recurso (acesso a sistemas externos)
```

### Padrões de mensagem entre agentes

**Formato de mensagem interoperável:**
```python
class AgentMessage(BaseModel):
    id: str                      # UUID único da mensagem
    sender_id: str               # ID do agente remetente
    recipient_id: str            # ID do agente destinatário
    conversation_id: str         # para rastrear a conversa
    message_type: str            # "task" | "result" | "clarification" | "error"
    content: dict                # payload da mensagem
    metadata: dict               # contexto adicional
    timestamp: datetime
    parent_message_id: Optional[str]  # para threading
```

### Desafios práticos de sistemas A2A

**Confiança e autenticação**
Um agente que recebe uma task de outro agente precisa verificar a identidade. JWT, mTLS, ou assinaturas criptográficas são necessários em produção.

**Gestão de estado distribuído**
Quando o estado de uma conversa está distribuído entre múltiplos agentes, "quem sabe o que" se torna complexo. LangGraph Platform resolve isso centralizando o estado.

**Falhas e resiliência**
O que acontece quando um subagente falha no meio de uma task longa? O orquestrador precisa de estratégias de retry, fallback, e circuit breaker.

**Observabilidade**
Rastrear uma conversa que passa por 5 agentes diferentes requer correlação de logs por `conversation_id` e `task_id`.

---

## Aplicação Prática

**Minimal A2A setup com LangGraph:**

```python
# O LangGraph Platform tem suporte nativo a múltiplos agentes via API
# Cada grafo compilado expõe endpoints que outros agentes podem chamar

# Agent A chamando Agent B via HTTP
import httpx

async def call_specialist_agent(task: str, agent_url: str) -> str:
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{agent_url}/invoke",
            json={"input": {"task": task}},
            headers={"Authorization": f"Bearer {API_KEY}"},
            timeout=120.0
        )
        return response.json()["output"]
```

---

## Conexões

- [06-01 — Arquitetura Multiagente](06-01-Arquitetura-Multiagente.md): topologias que usam A2A
- [07-01 — O que é MCP](../Modulo-07-MCP-Protocolos-Modernos/07-01-O-que-e-MCP.md): protocolo complementar para agente-ferramenta
- [06-04 — Debugging com LangSmith](06-04-Debugging-LangSmith.md): rastrear conversas A2A

---

## Recursos Complementares

- **Google A2A Protocol**: [github.com/google-deepmind/a2a](https://github.com/google-deepmind/a2a) — spec + SDK Python
- **OpenAI Swarm**: [github.com/openai/swarm](https://github.com/openai/swarm) — referência educacional leve
- **LangGraph Platform (multi-agent)**: [langchain-ai.github.io/langgraph/cloud](https://langchain-ai.github.io/langgraph/cloud)
- **"The Rise of Multi-Agent Systems" — Simon Willison**: análise crítica dos protocolos emergentes

---

## Auto-reflexão

1. Qual é a diferença entre A2A (Agent-to-Agent) e MCP (Model Context Protocol)? O que cada um resolve?
2. Por que comunicação assíncrona (filas) é preferível a HTTP síncrono para tarefas de agente que podem levar minutos?
3. Um sistema tem 4 agentes: Pesquisador, Analista, Redator e Revisor. Proponha um Agent Card para o Analista, incluindo skills, capacidades e modos de input/output.
4. Quais são os desafios de segurança em um sistema onde qualquer agente pode chamar qualquer outro via HTTP?

---

*[06-02] | Anterior: [06-01](06-01-Arquitetura-Multiagente.md) | Próximo: [06-03](06-03-Padroes-Avancados-Multiagente.md)*
*Referências: Google A2A spec (2025); OpenAI Swarm; Anthropic MCP docs*
