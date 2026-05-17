# [08-01] Cursor como Ambiente — O IDE como Plataforma de Agentes

## Por que isso importa?

Cursor não é um editor de texto com autocompletar. É uma plataforma de desenvolvimento onde agentes de IA têm acesso ao contexto completo do projeto, ferramentas do sistema operacional, e qualquer serviço externo via MCP. Entender essa arquitetura muda como você pensa no que é possível construir.

---

## Conceito Central

**Cursor é um IDE construído sobre VS Code que expõe o contexto do projeto (arquivos, terminal, histórico git, linter errors) a modelos de IA de última geração. Com MCP integrado nativo, o agente do Cursor pode acessar qualquer sistema externo. Com Rules e Skills, o comportamento do agente é customizável e persistente.**

---

## Aprofundamento

### A arquitetura do Cursor como plataforma de agentes

```
┌─────────────────────────────────────────────────────────────┐
│                    CURSOR IDE                                │
│                                                             │
│  ┌──────────────┐    ┌────────────────────────────────┐    │
│  │  VS Code     │    │        AGENT ENGINE             │    │
│  │  (Editor)    │    │  LLM: Claude 3.5 / GPT-4o /    │    │
│  │              │    │       Gemini Pro                │    │
│  └──────────────┘    └────────────────────────────────┘    │
│         │                        │                          │
│         │            ┌───────────┴──────────────────┐      │
│         │            │       CONTEXT ENGINE          │      │
│         │            │  - Arquivos abertos           │      │
│         │            │  - Codebase indexado          │      │
│         │            │  - Histórico de edições       │      │
│         │            │  - Linter errors              │      │
│         │            │  - Terminal output            │      │
│         │            │  - @mentions                  │      │
│         │            └───────────────────────────────┘      │
│         │                        │                          │
│         │            ┌───────────┴──────────────────┐      │
│         │            │       MCP CLIENT              │      │
│         │            │  - GitHub MCP                 │      │
│         │            │  - Jira MCP                   │      │
│         │            │  - Custom servers             │      │
│         │            └───────────────────────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Modos de interação com o agente

**Chat (Ctrl+L)**
Conversa com o agente sobre qualquer coisa. O agente tem acesso ao contexto do projeto mas não edita automaticamente.

**Composer/Agent Mode (Ctrl+I)**
O agente pode editar arquivos, criar novos, executar comandos no terminal, e encadear múltiplas ações. É o modo "agente autônomo" — execute e observe.

**Inline Edit (Ctrl+K)**
Edição em linha de um trecho específico de código ou texto. Mais rápido que o Agent Mode para mudanças localizadas.

**Tab Completion**
O agente prediz o próximo trecho de código enquanto você digita. Pode completar linhas inteiras ou blocos.

### Contexto que o agente tem acesso

**Arquivos e código:**
- Arquivo atual aberto (sempre no contexto)
- Arquivos abertos recentemente
- `@file` — incluir arquivo específico
- `@folder` — incluir pasta
- `@codebase` — busca semântica no código inteiro

**Ambiente:**
- `@terminal` — output do terminal atual
- Linter errors do arquivo atual
- Diff do git (mudanças não commitadas)

**Documentação e web:**
- `@docs` — documentação indexada de bibliotecas
- `@web` — busca na web em tempo real

**Contexto do vault (Marcus-Docs específico):**
- `.cursor/rules/*.mdc` — regras que injetam contexto automaticamente
- `AGENTS.md` — mapa de navegação para agentes
- Skills em `.cursor/skills/` — capacidades especializadas

### Configuração do Cursor para desenvolvimento de agentes

**settings.json (relevante para IA):**
```json
{
  // Modelo padrão para diferentes modos
  "cursor.chat.model": "claude-3-5-sonnet-20241022",
  "cursor.agent.model": "claude-3-5-sonnet-20241022",
  
  // Indexação do codebase
  "cursor.indexCodebase": true,
  "cursor.indexingEnabled": true,
  
  // Contexto automático
  "cursor.includeCurrentFile": true,
  "cursor.includeDiffContext": true,
  
  // Privacidade (para código sensível)
  "cursor.privacyMode": false  // true = não envia código para treino
}
```

**Configuração de MCP (`.cursor/mcp.json`):**
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/root/Marcus-Docs"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"}
    },
    "atlassian": {
      "command": "python",
      "args": ["-m", "mcp_atlassian"],
      "env": {
        "JIRA_URL": "${JIRA_URL}",
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    }
  }
}
```

### Cursor como ambiente de desenvolvimento de agentes

O Cursor se torna um meta-ambiente: você usa um agente (Cursor) para **construir** outros agentes. As capacidades se combinam:

1. O agente do Cursor escreve o código LangGraph do seu agente
2. Executa testes no terminal integrado
3. Lê os erros e corrige automaticamente
4. Commita via GitHub MCP
5. Cria ticket de follow-up no Jira MCP
6. Notifica o time no Slack MCP

Tudo isso em uma única sessão de chat, sem sair do IDE.

### Privacidade e segurança no Cursor

**Modo Privacy (para código sensível):**
```json
{"cursor.privacyMode": true}
// Código não é enviado para treino, processamento local quando possível
```

**Context exclusions (.cursorignore):**
```
# Similar ao .gitignore — excluir da indexação
.env
*.pem
secrets/
node_modules/
```

**Audit logs:** Cursor Business tem logs de auditoria completos de todas as interações com o agente.

---

## Aplicação Prática

**Workflow típico de desenvolvimento de agentes no Cursor:**

```
1. Descrever o agente em linguagem natural no Chat
   → "Quero um agente que monitora métricas do blog e alerta no Slack"

2. Agente do Cursor gera a estrutura inicial
   → Cria arquivos, instala dependências, escreve código base

3. Iterar no Agent Mode
   → Editar, testar no terminal, corrigir erros

4. Usar Skills especializadas do vault
   → @desenvolvedor-gran para padrões específicos

5. Commit e PR via MCP
   → Agente commita, cria PR no GitHub, notifica no Slack
```

---

## Conexões

- [08-02 — Rules, Skills e Hooks](08-02-Rules-Skills-e-Hooks.md): customizar o comportamento do agente
- [07-01 — O que é MCP](../Modulo-07-MCP-Protocolos-Modernos/07-01-O-que-e-MCP.md): protocolo usado pelos MCP servers do Cursor
- [08-03 — Cursor CLI e SDK](08-03-Cursor-CLI-e-SDK.md): ir além da interface gráfica

---

## Recursos Complementares

- **Cursor documentation**: [docs.cursor.com](https://docs.cursor.com) — documentação oficial
- **Cursor changelog**: [cursor.com/changelog](https://cursor.com/changelog) — updates frequentes
- **Cursor forum**: [forum.cursor.com](https://forum.cursor.com) — comunidade ativa
- **"Cursor is More Than a Copilot"** — Simon Willison blog: análise da arquitetura

---

## Auto-reflexão

1. Qual é a diferença fundamental entre um "GitHub Copilot" (inline suggestions) e o Agent Mode do Cursor?
2. Por que ter o contexto do terminal, linter errors, e diff do git disponíveis para o agente muda qualitativamente o tipo de tarefa que ele pode executar?
3. Quando você usaria Privacy Mode no Cursor? Quais são as implicações para a qualidade das sugestões?
4. Em que sentido o Cursor é um "meta-ambiente" para desenvolvimento de agentes de IA?

---

*[08-01] | Anterior: [07-04](../Modulo-07-MCP-Protocolos-Modernos/07-04-MCP-Backend-para-Agentes.md) | Próximo: [08-02](08-02-Rules-Skills-e-Hooks.md)*
*Referências: Cursor docs; docs.cursor.com/context; Cursor changelog*
