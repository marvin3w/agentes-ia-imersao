# [07-04] MCP como Backend para Agentes — Integração Programática

## Por que isso importa?

Você sabe criar servidores MCP e usar os prontos do ecossistema. O próximo passo é integrar MCP programaticamente em agentes LangChain/LangGraph customizados — não apenas via Claude Desktop ou Cursor, mas em qualquer aplicação Python.

---

## Conceito Central

**O SDK Python do MCP permite conectar agentes programáticos a servidores MCP, transformando as tools e resources expostos pelo servidor em ferramentas LangChain padrão. O resultado: um agente LangGraph pode usar qualquer servidor MCP como se fossem funções Python normais.**

---

## Aprofundamento

### Conectar um agente LangChain a um servidor MCP

```python
from langchain_mcp_adapters.tools import load_mcp_tools
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def create_agent_with_mcp():
    # Parâmetros do servidor MCP (stdio)
    server_params = StdioServerParameters(
        command="python",
        args=["server.py"],
        env={"API_TOKEN": "...", "DATABASE_URL": "..."}
    )
    
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # Inicializar conexão
            await session.initialize()
            
            # Carregar tools do servidor como LangChain tools
            tools = await load_mcp_tools(session)
            
            # tools agora são objetos Tool do LangChain
            for tool in tools:
                print(f"Tool disponível: {tool.name} — {tool.description[:50]}...")
            
            # Criar agente com as tools MCP
            llm = ChatOpenAI(model="gpt-4o", temperature=0)
            agent = create_react_agent(llm, tools)
            
            # Executar
            result = await agent.ainvoke({
                "messages": [{"role": "user", "content": "Liste os últimos 5 pedidos do cliente 123"}]
            })
            
            return result
```

### Múltiplos servidores MCP simultaneamente

```python
from langchain_mcp_adapters.tools import load_mcp_tools
from contextlib import AsyncExitStack

async def create_multi_mcp_agent():
    servers = {
        "orders": StdioServerParameters(command="python", args=["orders_server.py"]),
        "analytics": StdioServerParameters(command="python", args=["analytics_server.py"]),
        "github": StdioServerParameters(
            command="npx", 
            args=["-y", "@modelcontextprotocol/server-github"],
            env={"GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..."}
        )
    }
    
    all_tools = []
    
    async with AsyncExitStack() as stack:
        for server_name, params in servers.items():
            read, write = await stack.enter_async_context(stdio_client(params))
            session = await stack.enter_async_context(ClientSession(read, write))
            await session.initialize()
            
            tools = await load_mcp_tools(session)
            print(f"Servidor '{server_name}': {len(tools)} tools carregadas")
            all_tools.extend(tools)
        
        agent = create_react_agent(ChatOpenAI(model="gpt-4o"), all_tools)
        
        result = await agent.ainvoke({
            "messages": [{
                "role": "user",
                "content": "Crie uma issue no GitHub com base no relatório de hoje"
            }]
        })
        
        return result
```

### Servidor MCP via SSE (HTTP remoto)

```python
from mcp.client.sse import sse_client

async def connect_remote_mcp(server_url: str, api_key: str):
    """Conecta a um servidor MCP remoto via HTTP/SSE."""
    async with sse_client(
        url=f"{server_url}/sse",
        headers={"Authorization": f"Bearer {api_key}"}
    ) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await load_mcp_tools(session)
            return tools
```

### Chamar tools e resources diretamente (sem LangChain)

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def direct_mcp_usage():
    """Usar MCP diretamente, sem LangChain — máximo controle."""
    server_params = StdioServerParameters(
        command="python", args=["server.py"]
    )
    
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            # Listar tools disponíveis
            tools_result = await session.list_tools()
            for tool in tools_result.tools:
                print(f"{tool.name}: {tool.description}")
            
            # Chamar uma tool diretamente
            result = await session.call_tool(
                "buscar_pedido",
                arguments={"pedido_id": "PED-456"}
            )
            print(result.content[0].text)
            
            # Ler um resource
            resource_result = await session.read_resource("empresa://catalogo/produtos")
            print(resource_result.contents[0].text)
            
            # Obter um prompt
            prompt_result = await session.get_prompt(
                "analise_cliente",
                arguments={"cliente_id": "CLI-789", "pedidos": "..."}
            )
            print(prompt_result.messages[0].content.text)
```

### Integrar MCP em um grafo LangGraph

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]
    mcp_tools: list  # tools carregadas do MCP

def create_mcp_langgraph_agent(mcp_tools: list):
    """Criar um grafo LangGraph com tools MCP."""
    
    llm = ChatOpenAI(model="gpt-4o")
    llm_with_tools = llm.bind_tools(mcp_tools)
    
    def agent_node(state: State):
        response = llm_with_tools.invoke(state["messages"])
        return {"messages": [response]}
    
    def tools_node(state: State):
        """Executar as tool calls do agente."""
        from langchain_core.messages import ToolMessage
        
        tool_map = {tool.name: tool for tool in mcp_tools}
        tool_calls = state["messages"][-1].tool_calls
        
        results = []
        for tc in tool_calls:
            tool = tool_map.get(tc["name"])
            if tool:
                output = tool.invoke(tc["args"])
                results.append(ToolMessage(content=str(output), tool_call_id=tc["id"]))
        
        return {"messages": results}
    
    def should_continue(state: State) -> str:
        last_msg = state["messages"][-1]
        if hasattr(last_msg, "tool_calls") and last_msg.tool_calls:
            return "tools"
        return END
    
    graph = StateGraph(State)
    graph.add_node("agent", agent_node)
    graph.add_node("tools", tools_node)
    graph.set_entry_point("agent")
    graph.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
    graph.add_edge("tools", "agent")
    
    return graph.compile()
```

### Padrões de produção com MCP

**Connection pooling** — reutilizar sessões:
```python
class MCPConnectionPool:
    """Pool de conexões MCP para alta performance."""
    
    def __init__(self, server_params: StdioServerParameters, pool_size: int = 5):
        self.server_params = server_params
        self.pool_size = pool_size
        self._sessions = []
    
    async def initialize(self):
        for _ in range(self.pool_size):
            read, write = await stdio_client(self.server_params).__aenter__()
            session = await ClientSession(read, write).__aenter__()
            await session.initialize()
            self._sessions.append(session)
    
    async def get_session(self) -> ClientSession:
        # Round-robin simples
        session = self._sessions.pop(0)
        self._sessions.append(session)
        return session
```

---

## Aplicação Prática

**Quando usar MCP programático vs. via cliente (Cursor/Claude Desktop):**

| Cenário | Abordagem |
|---|---|
| Desenvolvimento e exploração | Via Cursor/Claude Desktop |
| Aplicação web com agentes | MCP programático (SDK Python) |
| Pipeline batch automatizado | MCP programático |
| Múltiplos agentes em paralelo | Connection pool + MCP programático |
| Protótipo rápido | Via Cursor/Claude Desktop |

---

## Conexões

- [07-03 — Criando Servidores MCP](07-03-Criando-Servidores-MCP.md): os servidores que você vai conectar programaticamente
- [04-03 — LangChain na Prática](../Modulo-04-Agentes-de-IA/04-03-LangChain-na-Pratica.md): framework que recebe as tools MCP
- [05-01 — LangGraph](../Modulo-05-Orquestracao-Fluxos/05-01-Grafos-de-Decisao-LangGraph.md): grafo que orquestra agents com tools MCP
- [08-03 — Cursor CLI e SDK](../Modulo-08-Cursor-Plataforma-Agentes/08-03-Cursor-CLI-e-SDK.md): uso de MCP via Cursor SDK

---

## Recursos Complementares

- **langchain-mcp-adapters**: [github.com/langchain-ai/langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters) — integração LangChain + MCP
- **MCP Python SDK**: [github.com/modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk)
- **"Using MCP with LangGraph"** — tutorial oficial LangChain
- **MCP Inspector**: ferramenta de debug para testar chamadas MCP

---

## Auto-reflexão

1. Por que é necessário um adapter (`langchain-mcp-adapters`) para usar tools MCP em um agente LangChain? O que o adapter faz?
2. Qual é a diferença entre chamar MCP via Claude Desktop e via SDK programático? Quando cada um é preferível?
3. Em um sistema com 100 requests/segundo que precisam usar um servidor MCP, por que um connection pool é necessário?
4. Você tem um servidor MCP com 30 tools. Ao carregar todas para um agente LangChain, o que acontece com a qualidade das decisões do agente? Como mitigar?

---

*[07-04] | Anterior: [07-03](07-03-Criando-Servidores-MCP.md) | Próximo: [M08-00](../Modulo-08-Cursor-Plataforma-Agentes/08-00-Visao-Geral.md)*
*Referências: langchain-mcp-adapters; MCP Python SDK; LangChain MCP tutorial*
