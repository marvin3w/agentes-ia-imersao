# [07-01] O que é MCP — O Protocolo que Conecta Agentes ao Mundo

## Por que isso importa?

Antes do MCP, cada integração de agente com ferramenta externa era um esforço custom: você escrevia código específico para conectar ao Jira, ao Google Drive, ao banco de dados — e refazia para cada novo agente. O MCP padroniza essa conexão com um protocolo universal, assim como HTTP padronizou a web.

---

## Conceito Central

**MCP (Model Context Protocol) é um protocolo aberto criado pela Anthropic que define uma linguagem padrão para agentes (clientes) se conectarem a fontes de dados e ferramentas (servidores). Com MCP, um agente pode usar qualquer servidor MCP existente sem código de integração específico — assim como um browser acessa qualquer site HTTP.**

---

## Aprofundamento

### O problema que o MCP resolve

Imagine o cenário anterior:

```
[Agente] → precisa acessar Google Drive
           → precisa escrever código específico para Google Drive API
           → precisa de auth, rate limiting, error handling
           → refazer tudo isso para Jira, Slack, GitHub, banco de dados...
```

Cada integração é um projeto separado. Com 10 ferramentas × 5 agentes = 50 integrações.

Com MCP:

```
[Servidor MCP Google Drive] → expõe ferramentas via protocolo padrão
[Servidor MCP Jira]          → idem
[Servidor MCP GitHub]        → idem

[Agente] → "lista os servidores MCP disponíveis"
         → "usa qualquer ferramenta de qualquer servidor"
         → 1 integração universal ao invés de N específicas
```

### Arquitetura MCP

```
┌─────────────────────────────────────────────────────┐
│                    HOST                              │
│   (Claude Desktop, Cursor, sua aplicação)            │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │              MCP CLIENT                      │   │
│  │  (gerencia conexões com servidores MCP)       │   │
│  └──────────────────────────────────────────────┘   │
│         │              │              │              │
└─────────┼──────────────┼──────────────┼─────────────┘
          │              │              │
   ┌──────┴──────┐ ┌─────┴────┐ ┌─────┴────────┐
   │  MCP SERVER │ │MCP SERVER│ │  MCP SERVER  │
   │ Google Drive│ │   Jira   │ │   GitHub     │
   └─────────────┘ └──────────┘ └──────────────┘
```

**Host:** a aplicação que usa agentes (Claude Desktop, Cursor IDE, sua app personalizada).

**MCP Client:** componente interno do host que gerencia conexões com servidores.

**MCP Server:** serviço que expõe capabilities via protocolo MCP. Pode ser:
- Processo local (stdio)
- Servidor HTTP remoto (SSE)

### As três primitivas do MCP

**1. Tools (Ferramentas)**
Funções que o agente pode executar para causar efeitos:
```python
# Exemplo de tool exposta via MCP
@mcp.tool()
def create_jira_issue(title: str, description: str, project_key: str) -> dict:
    """Cria uma issue no Jira."""
    return jira_client.create_issue({
        "summary": title,
        "description": description,
        "project": {"key": project_key}
    })
```

**2. Resources (Recursos)**
Dados que o agente pode ler:
```python
# Recurso: arquivo ou dado estático/dinâmico
@mcp.resource("drive://files/{file_id}")
def get_drive_file(file_id: str) -> str:
    """Retorna o conteúdo de um arquivo do Google Drive."""
    return drive_client.get_file_content(file_id)
```

**3. Prompts (Templates)**
Prompts reutilizáveis que o servidor expõe:
```python
# Template de prompt que o servidor oferece
@mcp.prompt()
def code_review_prompt(code: str, language: str) -> str:
    """Template para revisão de código."""
    return f"""Você é um expert em {language}.
    Revise o seguinte código e identifique problemas:
    
    ```{language}
    {code}
    ```"""
```

### O protocolo de comunicação

MCP usa JSON-RPC 2.0. Comunicação básica:

**1. Handshake (inicialização)**
```json
// Cliente → Servidor
{"jsonrpc": "2.0", "id": 1, "method": "initialize", 
 "params": {"protocolVersion": "2024-11-05", "clientInfo": {"name": "cursor"}}}

// Servidor → Cliente
{"jsonrpc": "2.0", "id": 1, "result": {
  "protocolVersion": "2024-11-05",
  "capabilities": {"tools": {}, "resources": {}},
  "serverInfo": {"name": "google-drive-mcp", "version": "1.0.0"}
}}
```

**2. Listar ferramentas disponíveis**
```json
// Cliente → Servidor
{"jsonrpc": "2.0", "id": 2, "method": "tools/list"}

// Servidor → Cliente
{"jsonrpc": "2.0", "id": 2, "result": {"tools": [
  {"name": "create_jira_issue", "description": "...", "inputSchema": {...}},
  {"name": "get_drive_file", "description": "...", "inputSchema": {...}}
]}}
```

**3. Chamar uma ferramenta**
```json
// Cliente → Servidor
{"jsonrpc": "2.0", "id": 3, "method": "tools/call",
 "params": {"name": "create_jira_issue", 
            "arguments": {"title": "Bug crítico", "project_key": "PROJ"}}}

// Servidor → Cliente
{"jsonrpc": "2.0", "id": 3, "result": {
  "content": [{"type": "text", "text": "Issue criada: PROJ-1234"}]
}}
```

### Transportes suportados

**stdio (local)**
Servidor roda como processo filho. Comunicação via stdin/stdout. Mais simples, mais seguro (sem porta exposta).

```json
// Claude Desktop / Cursor config
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["/path/to/server.py"],
      "env": {"API_KEY": "..."}
    }
  }
}
```

**SSE (HTTP, remoto)**
Servidor roda como serviço HTTP. Permite servidores em máquinas remotas, compartilhados por múltiplos clientes.

```python
# Servidor MCP via HTTP/SSE
mcp_server.run(transport="sse", host="0.0.0.0", port=8000)
```

### Ecossistema atual (2025)

**Servidores MCP oficiais (Anthropic):**
- Filesystem (ler/escrever arquivos locais)
- GitHub (repos, issues, PRs)
- Slack (mensagens, canais)
- Google Drive (arquivos, documentos)
- PostgreSQL (queries SQL)
- Puppeteer (automação de browser)
- Brave Search (busca web)

**Servidores MCP da comunidade:**
- Jira (Atlassian)
- Linear (issues e projetos)
- Notion (documentação)
- Spotify (controle de música)
- VS Code (integração com editor)
- Cursor (integração nativa — M08)

**Registry:** [modelcontextprotocol.io/servers](https://modelcontextprotocol.io/servers)

---

## Aplicação Prática

**MCP em uso no seu vault:**

Você já usa MCP diariamente sem perceber explicitamente:
- `user-github-gran` — servidor MCP que permite ao agente criar issues, ler PRs
- `user-atlassian` — servidor MCP para Jira e Confluence
- `user-google-workspace` — servidor MCP para Gmail e Drive
- `cursor-ide-browser` — servidor MCP para automação de browser

Cada vez que o agente "lê um arquivo do Drive" ou "cria um ticket no Jira", está usando o protocolo MCP para chamar o servidor correspondente.

---

## Conexões

- [07-02 — Servidores MCP Prontos](07-02-Servidores-MCP-Prontos.md): ecossistema de servidores disponíveis
- [07-03 — Criando Servidores MCP](07-03-Criando-Servidores-MCP.md): construir seus próprios servidores
- [06-02 — Comunicação A2A](../Modulo-06-Sistemas-Multiagente/06-02-Comunicacao-A2A.md): MCP para ferramentas, A2A para agentes
- [08-01 — Cursor como Ambiente](../Modulo-08-Cursor-Plataforma-Agentes/08-01-Cursor-como-Ambiente.md): MCP nativo no Cursor

---

## Recursos Complementares

- **MCP Specification**: [spec.modelcontextprotocol.io](https://spec.modelcontextprotocol.io) — spec completa do protocolo
- **MCP Introduction (Anthropic)**: [modelcontextprotocol.io/introduction](https://modelcontextprotocol.io/introduction)
- **"MCP: The USB-C of AI" — Simon Willison blog**: analogia que ficou famosa para explicar o MCP
- **Awesome MCP Servers (GitHub)**: [github.com/punkpeye/awesome-mcp-servers](https://github.com/punkpeye/awesome-mcp-servers) — lista curada da comunidade

---

## Auto-reflexão

1. Por que a analogia "MCP é o USB-C dos agentes de IA" é útil? Onde a analogia quebra?
2. Qual é a diferença entre uma "tool" MCP e um "resource" MCP? Dê um exemplo de cada.
3. Por que transporte stdio (local) é mais seguro que SSE (HTTP remoto)? Quando você escolheria cada um?
4. Antes do MCP, você tinha 5 agentes diferentes e 8 ferramentas externas. Quantas integrações precisaria escrever? Com MCP, quantas?

---

*[07-01] | Anterior: [06-04](../Modulo-06-Sistemas-Multiagente/06-04-Debugging-LangSmith.md) | Próximo: [07-02](07-02-Servidores-MCP-Prontos.md)*
*Referências: MCP Specification; Anthropic MCP docs; "Model Context Protocol Explained" (Simon Willison)*
