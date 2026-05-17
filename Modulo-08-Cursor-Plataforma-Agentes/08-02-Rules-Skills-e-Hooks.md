# [08-02] Rules, Skills e Hooks — Programar o Comportamento do Agente

## Por que isso importa?

Rules, Skills e Hooks são o "sistema operacional" do agente no Cursor. Eles determinam o que o agente sabe, como ele age e o que acontece automaticamente. Dominar os três transforma o Cursor de uma ferramenta genérica em uma plataforma customizada para o seu contexto específico.

---

## Conceito Central

**Rules definem o comportamento persistente do agente (como ele "pensa"). Skills definem capacidades especializadas (o que ele "sabe fazer"). Hooks definem gatilhos automáticos (o que acontece sem o agente precisar ser chamado explicitamente). Os três juntos formam a camada de orquestração do agente no Cursor.**

---

## Aprofundamento

### Rules — Instruções Persistentes

Rules são documentos markdown armazenados em `.cursor/rules/` que são injetados automaticamente no contexto do agente quando uma condição é atendida.

**Tipos de rules:**

**Always Applied (`alwaysApply: true`)**
```yaml
---
description: "Guardrails operacionais globais"
alwaysApply: true
---
# Guardrails Operacionais

## Regras não-negociáveis
- [G-01] Nunca fazer push para gran-legacy diretamente
- [G-04] Não declarar "resolvido" sem evidência objetiva
- [G-14] Proibido usar em dash (—) em comunicação externa
```

**Agent Requestable (sob demanda)**
```yaml
---
description: "Tech Lead — revisão técnica de código PHP/WordPress"
---
# Tech Lead

Você é um tech lead especializado em PHP, WordPress e WooCommerce.
Ao revisar código, verifique: segurança (SQL injection, XSS), performance (N+1 queries), 
padrões WordPress (sanitization, nonces), e documentação inline.
```

**Glob Pattern (baseado em arquivo)**
```yaml
---
globs: "**/*.py"
---
# Python Code Standards

Para arquivos Python:
- Type hints obrigatórios em funções públicas
- Docstrings no formato Google Style
- Testes em pytest
- Sem magic numbers — use constantes nomeadas
```

**Estrutura recomendada de uma rule:**
```markdown
---
description: "Descrição concisa do que a rule faz"
alwaysApply: false  # ou true para sempre injetar
globs: "pattern/*.ext"  # ou vazio para sem glob
---

# Nome da Rule

## Contexto (quando se aplica)
[Quando este agent/rule deve ser usado]

## Comportamento esperado
[O que o agente deve fazer]

## Anti-padrões (o que evitar)
[O que o agente não deve fazer]

## Referências
[Links para documentação relacionada]
```

### Skills — Capacidades Especializadas

Skills são SKILL.md files em `.cursor/skills/<nome>/SKILL.md` que o agente lê quando invocado, contendo instruções detalhadas para executar uma tarefa específica.

**Diferença entre Rule e Skill:**
- **Rule:** comportamento geral, sempre presente (ou ativado por contexto)
- **Skill:** procedimento para tarefa específica, lido quando necessário

**Estrutura de uma Skill:**
```markdown
# Nome da Skill — Descrição do que faz

## Quando usar esta skill
[Sinais que indicam que esta skill é necessária]

## Pré-condições
- [ ] Verificar que X está disponível
- [ ] Ter acesso a Y

## Procedimento
### Fase 1 — Preparação
1. Ler [arquivo de contexto]
2. Verificar estado atual

### Fase 2 — Execução
1. Executar passo A
2. Verificar resultado
3. Se OK → passo B; se erro → [tratamento]

### Fase 3 — Finalização
1. Registrar resultado em [local]
2. Notificar [quem]

## Critérios de aceite
- [ ] Output X gerado em [local]
- [ ] Log registrado

## Referências
- [doc relacionado]
```

**Exemplos do vault (skills reais):**
- `jira-demand-doc/SKILL.md` — criar documentação de demanda a partir de ticket Jira
- `daily-standup-prep/SKILL.md` — montar texto de daily a partir do vault
- `panorama-de-acervo/SKILL.md` — gerar panorama semântico de um corpus
- `worktree-open/SKILL.md` — abrir worktree git com protocolo correto

### Hooks — Automação Sem Intervenção

Hooks são scripts executados automaticamente em eventos do Cursor:

**Configuração (`hooks.json`):**
```json
{
  "hooks": [
    {
      "name": "jira-write-gate",
      "event": "beforeMCPExecution",
      "matcher": {
        "serverName": "user-atlassian",
        "toolNames": ["create_jira_issue", "update_jira_issue", "add_jira_comment"]
      },
      "command": "python3 .cursor/hooks/jira-write-gate.py",
      "description": "Gate de confirmação antes de escrever no Jira"
    },
    {
      "name": "worktree-gate",
      "event": "beforeToolUse",
      "matcher": {
        "toolName": "run_terminal_command",
        "pattern": "git worktree"
      },
      "command": "bash .cursor/hooks/worktree-gate.sh",
      "description": "Verificar protocolo antes de operações de worktree"
    }
  ]
}
```

**Hook como gate de segurança:**
```python
# .cursor/hooks/jira-write-gate.py
import sys
import json

def main():
    # Receber contexto da operação
    context = json.loads(sys.stdin.read())
    tool_name = context.get("toolName")
    arguments = context.get("arguments", {})
    
    # Log da operação
    with open("Meta-Marcus-Docs/Logs/mcp-writes.ndjson", "a") as f:
        import json, datetime
        f.write(json.dumps({
            "iso": datetime.datetime.now().isoformat(),
            "tool": tool_name,
            "args_keys": list(arguments.keys())  # não logar valores sensíveis
        }) + "\n")
    
    # Sempre permitir (gate de confirmação é feito pelo agente via AskQuestion)
    print(json.dumps({"allow": True}))

main()
```

### Arquitetura do sistema de rules do vault

```
.cursor/rules/
├── guardrails-operacionais.mdc    # always applied — regras globais
├── agent-nav-protocol.mdc        # always applied — navegação
├── gate-escrita-externa.mdc      # always applied — gate de escrita
├── agent-ai-master.mdc           # requestable — orquestrador
├── agent-tech-lead.mdc           # requestable — revisão técnica
├── Wesley.mdc                    # requestable — orquestrador V3
└── [...]

.cursor/skills/
├── jira-demand-doc/SKILL.md
├── daily-standup-prep/SKILL.md
├── worktree-open/SKILL.md
└── [...]

.cursor/hooks/
├── jira-write-gate.py
├── worktree-gate.sh
└── [...]
```

### Criar uma Rule nova (padrão do vault)

```markdown
# skill: create-rule (SKILL.md de exemplo)

## Procedimento

1. Identificar o escopo: 
   - Global (always applied) → tem guardrails universais
   - Contextual (requestable) → agente especialista
   - Por arquivo (glob) → padrão de código

2. Criar o arquivo:
   `.cursor/rules/nome-da-rule.mdc`

3. Estrutura obrigatória:
   - Frontmatter com description e alwaysApply
   - Seção de contexto (quando usar)
   - Comportamento esperado (o que fazer)
   - Anti-padrões (o que não fazer)
   - Referências

4. Adicionar entrada em `AGENTS.md` se for uma persona nova

5. Testar: invocar o agente com um caso que deveria ativar a rule
```

---

## Aplicação Prática

**Hierarquia de instrução no Cursor:**
```
System Prompt do Cursor (fixo, Anthropic define)
    ↓
Always Applied Rules (seu .cursor/rules/ alwaysApply: true)
    ↓
Glob-matched Rules (ativas para o tipo de arquivo atual)
    ↓
Agent Requestable Rules (ativas quando o agente lê o SKILL.md)
    ↓
Skill (quando invocada, lida pelo agente)
    ↓
Contexto do usuário (@mentions, arquivos abertos, chat history)
```

---

## Conexões

- [08-01 — Cursor como Ambiente](08-01-Cursor-como-Ambiente.md): infraestrutura onde Rules/Skills/Hooks rodam
- [08-03 — Cursor CLI e SDK](08-03-Cursor-CLI-e-SDK.md): automatizar criação de rules e skills programaticamente
- [07-01 — O que é MCP](../Modulo-07-MCP-Protocolos-Modernos/07-01-O-que-e-MCP.md): hooks podem ser disparados por chamadas MCP

---

## Recursos Complementares

- **Cursor Rules docs**: [docs.cursor.com/context/rules](https://docs.cursor.com/context/rules)
- **Cursor Hooks docs**: [docs.cursor.com/context/hooks](https://docs.cursor.com/context/hooks)
- **"Awesome Cursor Rules"** — [github.com/PatrickJS/awesome-cursorrules](https://github.com/PatrickJS/awesome-cursorrules): coleção de rules da comunidade
- **`.cursor/skills/create-rule/SKILL.md`** do vault: skill especializada para criar rules no padrão do vault

---

## Auto-reflexão

1. Qual é a diferença entre uma Rule "always applied" e uma Rule "agent requestable"? Quando você usaria cada uma?
2. Por que Skills são documentos separados das Rules, ao invés de serem incorporadas diretamente como Rules?
3. Um Hook é executado antes de uma operação MCP de escrita no Jira. O que acontece se o hook retornar `{"allow": false}`?
4. Projete uma Rule para um agente que vai revisar documentação técnica: o que ela deveria conter em "comportamento esperado" e "anti-padrões"?

---

*[08-02] | Anterior: [08-01](08-01-Cursor-como-Ambiente.md) | Próximo: [08-03](08-03-Cursor-CLI-e-SDK.md)*
*Referências: Cursor Rules docs; Cursor Hooks docs; vault AGENTS.md*
