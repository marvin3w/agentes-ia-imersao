# [05-04] Ferramentas Externas e Integrações — Conectar Agentes ao Mundo Real

## Por que isso importa?

Um agente sem ferramentas é um LLM glorificado. As ferramentas são o ponto onde a IA encontra o mundo real: bancos de dados, APIs, sistemas de arquivos, email, Slack, e qualquer serviço externo. A qualidade e o design das ferramentas determinam o que o agente consegue fazer.

---

## Conceito Central

**Uma ferramenta (tool) é qualquer função que o agente pode invocar. Boas ferramentas têm: nome autodescritivo, docstring precisa (é o que o LLM lê para decidir se e como usar), argumentos tipados, e tratamento de erros robusto. Tools mal escritas são a principal causa de agentes que "alucinam" chamadas incorretas.**

---

## Aprofundamento

### Anatomia de uma ferramenta bem projetada

```python
from langchain.tools import tool
from pydantic import BaseModel, Field
from typing import Optional

class SearchInput(BaseModel):
    query: str = Field(description="A query de busca em linguagem natural")
    max_results: Optional[int] = Field(
        default=5, 
        description="Número máximo de resultados (1-10)",
        ge=1, 
        le=10
    )
    date_filter: Optional[str] = Field(
        default=None,
        description="Filtrar por data: 'last_week', 'last_month', 'last_year'"
    )

@tool(args_schema=SearchInput)
def search_company_docs(query: str, max_results: int = 5, date_filter: Optional[str] = None) -> str:
    """
    Busca documentos internos da empresa.
    
    Use esta ferramenta quando precisar de:
    - Informações sobre políticas e procedimentos internos
    - Contratos e documentação jurídica
    - Histórico de decisões e registros
    
    NÃO use para:
    - Informações públicas disponíveis na internet (use search_web)
    - Dados em tempo real de sistemas (use query_database)
    
    Returns:
        String com os trechos mais relevantes encontrados e suas fontes.
        Retorna "Nenhum documento encontrado" se não há resultados.
    """
    try:
        results = internal_vectorstore.similarity_search(
            query, k=max_results,
            filter={"date_filter": date_filter} if date_filter else None
        )
        
        if not results:
            return "Nenhum documento encontrado para esta consulta."
        
        formatted = []
        for doc in results:
            formatted.append(f"[{doc.metadata.get('source', 'desconhecido')}]\n{doc.page_content}")
        
        return "\n\n---\n\n".join(formatted)
    
    except Exception as e:
        return f"Erro ao buscar documentos: {str(e)}"
```

**Por que a docstring importa tanto?** O LLM lê a docstring para decidir se deve usar esta ferramenta. Uma docstring ruim = ferramenta usada no contexto errado.

### Categorias de ferramentas comuns

**Busca e informação**
```python
@tool
def search_web(query: str) -> str:
    """Busca informações atuais na internet. Use para fatos recentes, notícias, preços."""
    from tavily import TavilyClient
    client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])
    results = client.search(query, max_results=5)
    return "\n".join([r["content"] for r in results["results"]])

@tool
def get_wikipedia(topic: str) -> str:
    """Busca artigo da Wikipedia sobre um tópico. Use para conceitos e definições gerais."""
    import wikipedia
    try:
        return wikipedia.summary(topic, sentences=10)
    except Exception as e:
        return f"Não encontrado: {e}"
```

**Execução de código e dados**
```python
@tool
def run_python(code: str) -> str:
    """
    Executa código Python e retorna o resultado.
    Use para: cálculos, análise de dados, transformações.
    NÃO use para: operações de arquivo, networking, instalação de pacotes.
    """
    import subprocess
    result = subprocess.run(
        ["python3", "-c", code],
        capture_output=True, text=True,
        timeout=10  # timeout de segurança
    )
    if result.returncode != 0:
        return f"Erro: {result.stderr}"
    return result.stdout

@tool
def query_database(sql_query: str) -> str:
    """
    Executa queries SQL READ-ONLY no banco de dados.
    Use para: relatórios, contagens, análises de dados.
    PROIBIDO: INSERT, UPDATE, DELETE, DROP — serão rejeitados.
    """
    # Validação de segurança
    forbidden = ["INSERT", "UPDATE", "DELETE", "DROP", "CREATE", "ALTER", "TRUNCATE"]
    if any(word in sql_query.upper() for word in forbidden):
        return "Erro: apenas queries SELECT são permitidas."
    
    with db_connection() as conn:
        result = pd.read_sql(sql_query, conn)
        return result.to_markdown(index=False)
```

**Comunicação e notificação**
```python
@tool
def send_slack_message(channel: str, message: str) -> str:
    """
    Envia uma mensagem no Slack.
    Use apenas quando o usuário explicitamente pedir para notificar alguém.
    canal: nome do canal sem '#', ex: 'geral', 'alertas'
    """
    # Esta tool requer aprovação humana em alguns workflows
    slack_client.chat_postMessage(channel=f"#{channel}", text=message)
    return f"Mensagem enviada para #{channel}"

@tool
def create_jira_task(title: str, description: str, assignee: Optional[str] = None) -> str:
    """
    Cria uma tarefa no Jira.
    Use apenas quando o usuário pedir explicitamente para criar um ticket.
    Sempre confirme com o usuário antes de usar em workflows autônomos.
    """
    issue = jira_client.create_issue({
        "project": {"key": "PROJ"},
        "summary": title,
        "description": description,
        "issuetype": {"name": "Task"},
        "assignee": {"name": assignee} if assignee else None
    })
    return f"Tarefa criada: {issue.key} — {issue.permalink()}"
```

### Ferramentas perigosas — guardrails obrigatórios

Ferramentas com side effects irreversíveis (delete, send, pay) precisam de proteção:

```python
from langgraph.graph import interrupt_after

# Padrão 1: Confirmar antes de executar
@tool
def delete_files(file_pattern: str) -> str:
    """Deleta arquivos. SEMPRE pedir confirmação antes de usar."""
    # O agente NUNCA deve chamar isso diretamente
    # Apenas após confirmação explícita do usuário
    ...

# Padrão 2: Dry run mode
@tool
def send_emails(recipients: List[str], subject: str, body: str, dry_run: bool = True) -> str:
    """
    Envia emails. 
    dry_run=True (padrão): apenas simula e mostra o que seria enviado.
    dry_run=False: envia de verdade. Requer confirmação explícita.
    """
    if dry_run:
        return f"[SIMULAÇÃO] Enviaria para {recipients}:\nAssunto: {subject}\n{body}"
    
    for recipient in recipients:
        email_client.send(recipient, subject, body)
    return f"Emails enviados para {len(recipients)} destinatários."
```

### Integrações com ecosistema de ferramentas

**LangChain Community Tools (prontas para usar):**
```python
from langchain_community.tools import (
    DuckDuckGoSearchRun,      # busca web gratuita
    TavilySearchResults,       # busca web avançada (pago)
    WikipediaQueryRun,         # Wikipedia
    ArxivQueryRun,             # artigos científicos
    PythonREPLTool,            # executar Python
    ShellTool,                 # shell Unix
    E2BDataAnalysisTool,       # sandbox seguro para código
)

# Toolkits (conjunto de tools relacionadas)
from langchain_community.agent_toolkits import (
    GmailToolkit,      # ler/enviar emails
    GitHubToolkit,     # operações git
    JiraToolkit,       # gestão de tickets
    SQLDatabaseToolkit # queries SQL
)
```

---

## Aplicação Prática

**Princípios para design de ferramentas:**

1. **Uma tool = uma responsabilidade** — não combine busca + análise em uma tool
2. **Docstring é contrato** — se o LLM pode interpretá-la errado, reescreva
3. **Sempre trate exceções** — nunca deixe stack trace vazar para o contexto do agente
4. **Dry run para side effects** — qualquer ação destrutiva ou irreversível tem modo de simulação
5. **Timeout explícito** — nunca deixe uma tool bloquear indefinidamente

---

## Conexões

- [04-02 — ReAct e Toolcalling](../Modulo-04-Agentes-de-IA/04-02-ReAct-e-Toolcalling.md): como o agente decide qual tool usar
- [07-01 — O que é MCP](../Modulo-07-MCP-Protocolos-Modernos/07-01-O-que-e-MCP.md): protocolo padrão para expor ferramentas a agentes
- [08-03 — Cursor CLI e SDK](../Modulo-08-Cursor-Plataforma-Agentes/08-03-Cursor-CLI-e-SDK.md): ferramentas no contexto do Cursor IDE

---

## Recursos Complementares

- **LangChain Community tools list**: [python.langchain.com/docs/integrations/tools](https://python.langchain.com/docs/integrations/tools) — 100+ tools prontas
- **Tavily AI (web search for agents)**: [tavily.com](https://tavily.com) — melhor web search API para agentes
- **E2B (sandboxed code execution)**: [e2b.dev](https://e2b.dev) — executar código de forma segura em sandbox
- **"Tool Design for LLM Agents" — Hamel Husain blog**: melhores práticas de design de docstrings

---

## Auto-reflexão

1. Por que a docstring de uma ferramenta é mais crítica do que a docstring de uma função Python comum?
2. Uma ferramenta retorna um stack trace de Python no output. O que isso causa no agente? Como prevenir?
3. Você tem uma ferramenta `send_payment(amount, recipient)`. Quais guardrails obrigatórios você implementa antes de colocá-la em produção em um agente autônomo?
4. Quando você escolheria `ShellTool` (executa qualquer comando shell) vs. `PythonREPLTool` vs. `E2BDataAnalysisTool`? Quais são os trade-offs de segurança?

---

## Próximo Passo

### [→ M06 — Sistemas Multiagente](../Modulo-06-Sistemas-Multiagente/06-00-Visao-Geral.md)

*Módulo concluído — clique acima para iniciar o próximo.*


---

*[← 05-03 — Padrões de Orquestração](05-03-Padroes-de-Orquestracao.md) · [↑ M05 — Orquestração e Fluxos](05-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
