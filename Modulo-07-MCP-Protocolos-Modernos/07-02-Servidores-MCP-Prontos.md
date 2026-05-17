# [07-02] Servidores MCP Prontos — O Ecossistema que Já Existe

## Por que isso importa?

Antes de construir um servidor MCP do zero, vale saber o que já existe. O ecossistema em 2025 tem centenas de servidores prontos para ferramentas comuns — e conectar um ao seu agente é questão de minutos.

---

## Conceito Central

**O ecossistema MCP tem três camadas: servidores oficiais da Anthropic (reference implementations), servidores de parceiros (integrações empresariais), e servidores da comunidade (open-source). A maioria dos casos de uso empresariais é coberta pela combinação das três camadas.**

---

## Aprofundamento

### Servidores Oficiais da Anthropic

Instalados diretamente via npx (Node.js) ou pip:

**Filesystem — acesso a arquivos locais**
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"]
    }
  }
}
```
Tools: `read_file`, `write_file`, `list_directory`, `search_files`, `move_file`, `get_file_info`

**GitHub — repositórios, issues, PRs**
```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..."}
    }
  }
}
```
Tools: `create_repository`, `get_file_contents`, `create_or_update_file`, `search_repositories`, `create_issue`, `create_pull_request`, `list_commits`

**PostgreSQL — queries SQL read-only**
```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```
Tools: `query`, `list_tables`, `describe_table`
Resources: cada tabela exposta como recurso navegável

**Brave Search — busca web**
```json
{
  "mcpServers": {
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {"BRAVE_API_KEY": "BSA..."}
    }
  }
}
```
Tools: `brave_web_search`, `brave_local_search`

**Puppeteer — automação de browser**
```json
{
  "mcpServers": {
    "puppeteer": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-puppeteer"]
    }
  }
}
```
Tools: `puppeteer_navigate`, `puppeteer_screenshot`, `puppeteer_click`, `puppeteer_fill`, `puppeteer_evaluate`

**Slack — mensagens e canais**
```json
{
  "mcpServers": {
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {"SLACK_BOT_TOKEN": "xoxb-...", "SLACK_TEAM_ID": "T..."}
    }
  }
}
```
Tools: `slack_post_message`, `slack_reply_to_thread`, `slack_list_channels`, `slack_get_channel_history`

### Servidores de Terceiros Relevantes

**Atlassian (Jira + Confluence)**
```bash
pip install mcp-atlassian
```
```json
{
  "mcpServers": {
    "atlassian": {
      "command": "python",
      "args": ["-m", "mcp_atlassian"],
      "env": {
        "JIRA_URL": "https://company.atlassian.net",
        "JIRA_EMAIL": "user@company.com",
        "JIRA_TOKEN": "..."
      }
    }
  }
}
```

**Google Workspace (Gmail + Drive + Calendar)**
```json
{
  "mcpServers": {
    "google-workspace": {
      "command": "node",
      "args": ["/path/to/google-workspace-mcp/index.js"],
      "env": {"GOOGLE_CREDENTIALS": "..."}
    }
  }
}
```

**Linear (gestão de projetos)**
```bash
npx @linear/mcp-server
```

**Notion (documentação e base de conhecimento)**
```bash
npx @notionhq/notion-mcp-server
```

**Sentry (error tracking)**
```bash
uvx mcp-server-sentry --auth-token SENTRY_TOKEN
```

### Descoberta via Registry

O repositório oficial mantém a lista curada:
```
https://github.com/modelcontextprotocol/servers — servidores oficiais
https://github.com/punkpeye/awesome-mcp-servers — comunidade (500+ servidores)
https://mcp.run — marketplace com discovery e instalação
```

### Configuração multi-servidor (cenário real)

Uma configuração realista para um agente de engenharia de produto:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..."}
    },
    "jira": {
      "command": "python",
      "args": ["-m", "mcp_atlassian"],
      "env": {"JIRA_URL": "...", "JIRA_TOKEN": "..."}
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {"SLACK_BOT_TOKEN": "xoxb-..."}
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://..."]
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {"BRAVE_API_KEY": "BSA..."}
    }
  }
}
```

Com essa config, o agente tem acesso a: arquivos locais, repositórios git, tickets Jira, mensagens Slack, banco de dados de produção (read-only) e busca web — sem escrever uma linha de integração.

### Segurança em servidores MCP

**Princípio do menor privilégio:**
- Filesystem: só expor o path necessário, nunca `/` ou `/home`
- GitHub: token com escopo mínimo (read-only se não precisa escrever)
- PostgreSQL: usuário com apenas SELECT em tabelas específicas
- Slack: bot com apenas canais necessários

**Variáveis de ambiente vs. configs hardcoded:**
```json
// CORRETO: tokens em env vars
{"env": {"API_KEY": "${MY_API_KEY}"}}

// ERRADO: tokens hardcoded na config
{"env": {"API_KEY": "sk-proj-abc123..."}}
```

**Auditoria:** logs de chamadas MCP ficam acessíveis no Cursor IDE (Dev Tools) e no LangSmith quando integrado.

---

## Aplicação Prática

**Checklist para adicionar um servidor MCP ao seu ambiente:**

- [ ] Verificar se existe servidor oficial ou da comunidade para a integração desejada
- [ ] Avaliar credenciais necessárias (quais escopos/permissões)
- [ ] Testar em ambiente isolado antes de adicionar ao vault/produção
- [ ] Configurar com princípio do menor privilégio
- [ ] Documentar no Cockpit Operacional do vault (G-13)
- [ ] Verificar se o servidor tem política de rate limiting configurada

---

## Conexões

- [07-01 — O que é MCP](07-01-O-que-e-MCP.md): protocolo base que todos esses servidores implementam
- [07-03 — Criando Servidores MCP](07-03-Criando-Servidores-MCP.md): quando o servidor que você precisa não existe
- [08-01 — Cursor como Ambiente](../Modulo-08-Cursor-Plataforma-Agentes/08-01-Cursor-como-Ambiente.md): como configurar MCP no Cursor IDE

---

## Recursos Complementares

- **MCP Servers (oficial)**: [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)
- **Awesome MCP Servers**: [github.com/punkpeye/awesome-mcp-servers](https://github.com/punkpeye/awesome-mcp-servers) — lista curada da comunidade com 500+ servidores
- **mcp.run**: [mcp.run](https://mcp.run) — marketplace com discovery visual
- **Smithery**: [smithery.ai](https://smithery.ai) — deploy fácil de servidores MCP

---

## Auto-reflexão

1. Dado o ecossistema atual de servidores MCP prontos, em que situações você realmente precisaria criar um servidor do zero?
2. Por que usar um usuário PostgreSQL read-only dedicado para o servidor MCP, em vez de usar as credenciais de admin?
3. Um agente tem acesso a um servidor MCP com `slack_post_message`. Quais guardrails você implementaria para evitar que o agente poste mensagens não autorizadas?
4. Você precisa integrar com um sistema legado interno da empresa que tem uma API REST proprietária. Não existe servidor MCP para ela. Qual é o caminho mais rápido para ter isso funcionando?

---

## Próximo Passo

### [→ 07-03 — Criando Servidores MCP](07-03-Criando-Servidores-MCP.md)


---

*[← 07-01 — O que é MCP](07-01-O-que-e-MCP.md) · [↑ M07 — MCP e Protocolos Modernos](07-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
