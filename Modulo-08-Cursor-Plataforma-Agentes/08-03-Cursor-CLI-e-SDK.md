# [08-03] Cursor CLI e SDK — Agentes Fora da Interface Gráfica

## Por que isso importa?

O Cursor tem uma interface gráfica poderosa, mas às vezes você precisa executar agentes em pipelines automatizados, scripts de CI/CD, ou aplicações sem interface. O Cursor CLI e o SDK TypeScript permitem isso — invocar o agente do Cursor programaticamente, sem abrir o IDE.

---

## Conceito Central

**O Cursor SDK (`@cursor/sdk`) expõe a capacidade de criar e executar sessões de agente do Cursor programaticamente. Você pode escrever um script que: inicializa um agente, passa instruções, recebe outputs via streaming, e encadeia múltiplas sessões — tudo sem o IDE aberto.**

---

## Aprofundamento

### Cursor CLI — Execução pela Linha de Comando

```bash
# Instalar a CLI do Cursor (incluída no Cursor Desktop)
cursor --version

# Modo headless: executar um prompt sem abrir GUI
cursor chat "Revise o arquivo auth.py e crie testes unitários"

# Com arquivo de contexto específico
cursor chat --file src/auth.py "Revise este arquivo e crie testes"

# Agent mode (pode editar arquivos)
cursor agent "Implemente o CRUD completo para o modelo User seguindo o padrão dos outros modelos"

# Com contexto de pasta
cursor agent --folder src/models "Adicione type hints faltantes em todos os modelos"
```

### Cursor SDK (`@cursor/sdk`) — TypeScript

```bash
npm install @cursor/sdk
```

```typescript
import { Agent } from "@cursor/sdk";

// Criar e executar um agente
const agent = Agent.create({
  model: "claude-3-5-sonnet-20241022",
  mcpServers: [
    {
      name: "filesystem",
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    }
  ]
});

// Executar com streaming
const run = await agent.send(
  "Analise o projeto em /workspace e liste as principais áreas de melhoria"
);

// Stream de eventos
for await (const event of run.stream()) {
  if (event.type === "message") {
    process.stdout.write(event.content);
  } else if (event.type === "tool_call") {
    console.log(`\n[Tool: ${event.tool}] ${JSON.stringify(event.args)}`);
  }
}

// Aguardar conclusão
const result = await run.wait();
console.log("\nConcluído:", result.finalMessage);
```

### Sessões com múltiplas mensagens

```typescript
// Conversa multi-turn com um agente
const agent = Agent.create({ model: "claude-3-5-sonnet-20241022" });

// Primeira mensagem
let run = await agent.send("Liste os arquivos Python no diretório atual");
await run.wait();

// Continuar a conversa (mesmo agente, mesmo contexto)
run = await agent.send("Agora analise o maior arquivo e sugira refatorações");
await run.wait();

// Resumir a sessão
run = await agent.send("Resuma o que discutimos e crie um plano de ação priorizado");
const summary = await run.wait();
console.log(summary.finalMessage);
```

### Integração com CI/CD (GitHub Actions)

```yaml
# .github/workflows/ai-code-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install Cursor SDK
        run: npm install @cursor/sdk
      
      - name: Run AI Code Review
        env:
          CURSOR_API_KEY: ${{ secrets.CURSOR_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          node scripts/ai-review.mjs \
            --pr-number ${{ github.event.number }} \
            --repo ${{ github.repository }}
```

```javascript
// scripts/ai-review.mjs
import { Agent } from "@cursor/sdk";
import { Octokit } from "@octokit/rest";

const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });
const [owner, repo] = process.env.GITHUB_REPO.split("/");

// Pegar diff do PR
const { data: files } = await octokit.pulls.listFiles({
  owner, repo,
  pull_number: parseInt(process.env.PR_NUMBER)
});

const diff = files.map(f => `## ${f.filename}\n\`\`\`diff\n${f.patch}\n\`\`\``).join("\n\n");

// Criar agente para revisão
const agent = Agent.create({
  model: "claude-3-5-sonnet-20241022",
  systemPrompt: `Você é um tech lead sênior fazendo code review.
  Identifique: bugs, problemas de segurança, performance, e violações de padrão.
  Seja direto e construtivo. Formate como lista de comentários por arquivo.`
});

const run = await agent.send(`Revise este PR:\n\n${diff}`);
const result = await run.wait();

// Postar revisão no PR
await octokit.pulls.createReview({
  owner, repo,
  pull_number: parseInt(process.env.PR_NUMBER),
  body: result.finalMessage,
  event: "COMMENT"
});

console.log("Review postado com sucesso!");
```

### Agentes em scripts de automação

```python
# agent_automation.py — Python com subprocess para Cursor CLI
import subprocess
import json

def run_cursor_agent(prompt: str, folder: str = None) -> str:
    """Executar um agente do Cursor e retornar o resultado."""
    cmd = ["cursor", "agent", "--json"]
    if folder:
        cmd.extend(["--folder", folder])
    cmd.append(prompt)
    
    result = subprocess.run(
        cmd,
        capture_output=True,
        text=True,
        timeout=300  # 5 minutos
    )
    
    if result.returncode != 0:
        raise RuntimeError(f"Cursor agent failed: {result.stderr}")
    
    return json.loads(result.stdout)["output"]

# Uso em pipeline
def code_review_pipeline(changed_files: list[str]) -> str:
    """Pipeline de revisão usando Cursor agent."""
    
    # Passo 1: análise de segurança
    security_review = run_cursor_agent(
        f"Faça revisão de segurança dos arquivos: {', '.join(changed_files)}. "
        f"Foque em: SQL injection, XSS, autenticação, autorização."
    )
    
    # Passo 2: análise de performance
    perf_review = run_cursor_agent(
        f"Analise performance dos mesmos arquivos. "
        f"Contexto de revisão de segurança: {security_review[:500]}"
    )
    
    # Passo 3: síntese
    synthesis = run_cursor_agent(
        f"Sintetize estas revisões em um relatório executivo:\n"
        f"Segurança: {security_review}\n"
        f"Performance: {perf_review}"
    )
    
    return synthesis
```

### Cursor Agents API (REST)

Para integração máxima flexibilidade:

```python
import httpx

CURSOR_API_BASE = "https://api.cursor.com/v1"
CURSOR_API_KEY = os.environ["CURSOR_API_KEY"]

headers = {
    "Authorization": f"Bearer {CURSOR_API_KEY}",
    "Content-Type": "application/json"
}

# Criar sessão de agente
response = httpx.post(
    f"{CURSOR_API_BASE}/agents",
    headers=headers,
    json={
        "model": "claude-3-5-sonnet-20241022",
        "system": "Você é um especialista em análise de código.",
        "mcpServers": [{"name": "filesystem", "command": "npx", "args": [...]}]
    }
)
agent_id = response.json()["agent_id"]

# Enviar mensagem
response = httpx.post(
    f"{CURSOR_API_BASE}/agents/{agent_id}/messages",
    headers=headers,
    json={"content": "Analise o arquivo main.py"},
    stream=True
)

# Processar streaming
for line in response.iter_lines():
    if line.startswith("data: "):
        event = json.loads(line[6:])
        if event["type"] == "content":
            print(event["text"], end="")
```

---

## Aplicação Prática

**Casos de uso do SDK/CLI em produção:**

| Cenário | Ferramenta |
|---|---|
| Code review automático em PRs | SDK TypeScript + GitHub Actions |
| Geração de documentação em batch | CLI com glob de arquivos |
| Análise semanal de codebase | Script Python + Cursor API |
| Pair programming em terminal | CLI interativo |
| Pipeline de migração de código | SDK TypeScript encadeado |

---

## Conexões

- [08-01 — Cursor como Ambiente](08-01-Cursor-como-Ambiente.md): o IDE que o SDK controla programaticamente
- [08-02 — Rules, Skills e Hooks](08-02-Rules-Skills-e-Hooks.md): Rules aplicam mesmo em modo CLI/SDK
- [07-04 — MCP Backend](../Modulo-07-MCP-Protocolos-Modernos/07-04-MCP-Backend-para-Agentes.md): MCP servers funcionam tanto via IDE quanto via SDK

---

## Recursos Complementares

- **Cursor SDK docs**: [docs.cursor.com/sdk](https://docs.cursor.com/sdk) — referência TypeScript
- **Cursor CLI docs**: [docs.cursor.com/cli](https://docs.cursor.com/cli) — comandos disponíveis
- **`@cursor/sdk` npm**: [npmjs.com/package/@cursor/sdk](https://www.npmjs.com/package/@cursor/sdk)
- **`.cursor/skills-cursor/sdk/SKILL.md`**: skill do vault para orientar uso do SDK

---

## Auto-reflexão

1. Qual é a principal vantagem de usar o Cursor SDK em um pipeline CI/CD comparado com chamar a API OpenAI/Anthropic diretamente?
2. Em que situações você usaria o Cursor CLI (linha de comando) vs. o SDK programático?
3. Um agent executado via SDK tem acesso às mesmas Rules do `.cursor/rules/` que um agent executado via interface gráfica?
4. Projete um pipeline de automação que: toda semana, roda o Cursor agent sobre todos os arquivos Python do projeto, gera um relatório de qualidade de código, e posta no Slack. Quais componentes você usaria?

---

## Próximo Passo

### [→ 08-04 — Automação do Vault](08-04-Automacao-do-Vault.md)


---

*[← 08-02 — Rules, Skills e Hooks](08-02-Rules-Skills-e-Hooks.md) · [↑ M08 — Cursor como Plataforma](08-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
