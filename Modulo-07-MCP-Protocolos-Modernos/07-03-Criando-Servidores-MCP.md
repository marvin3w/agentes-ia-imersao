# [07-03] Criando Servidores MCP — Expor Qualquer Sistema para Agentes

## Por que isso importa?

Quando o servidor MCP que você precisa não existe, você cria. E criar um servidor MCP é surpreendentemente simples com o SDK Python da Anthropic. Qualquer API interna, banco de dados proprietário, ou serviço customizado pode ser exposto via MCP em poucas horas.

---

## Conceito Central

**Um servidor MCP em Python é uma classe com métodos decorados com `@mcp.tool()`, `@mcp.resource()` ou `@mcp.prompt()`. O SDK cuida de todo o protocolo JSON-RPC — você só escreve a lógica de negócio. É tão simples quanto escrever um script Python com funções.**

---

## Aprofundamento

### Setup mínimo

```bash
pip install mcp
```

```python
# server.py — servidor MCP mínimo funcional
from mcp.server.fastmcp import FastMCP

# Criar o servidor
mcp = FastMCP("meu-servidor")

# Definir uma ferramenta
@mcp.tool()
def saudacao(nome: str) -> str:
    """Retorna uma saudação personalizada."""
    return f"Olá, {nome}! Bem-vindo ao MCP."

# Executar
if __name__ == "__main__":
    mcp.run()
```

```bash
python server.py
# Servidor rodando via stdio, pronto para ser chamado por um MCP client
```

### Servidor MCP completo com Tools, Resources e Prompts

```python
from mcp.server.fastmcp import FastMCP
from typing import Optional
import httpx

mcp = FastMCP("empresa-interna", description="Servidor MCP para sistemas internos")

# ============================================================
# TOOLS — funções que o agente pode executar
# ============================================================

@mcp.tool()
def buscar_pedido(pedido_id: str) -> dict:
    """
    Busca informações de um pedido pelo ID.
    
    Use quando o usuário perguntar sobre status, detalhes ou histórico de um pedido específico.
    Returns: dict com campos: id, status, cliente, valor, data_criacao, itens
    """
    # Chamar sua API interna
    response = httpx.get(
        f"https://api.empresa.com/pedidos/{pedido_id}",
        headers={"Authorization": f"Bearer {API_TOKEN}"}
    )
    
    if response.status_code == 404:
        return {"error": f"Pedido {pedido_id} não encontrado"}
    
    return response.json()

@mcp.tool()
def listar_pedidos_cliente(
    cliente_id: str, 
    status: Optional[str] = None,
    limit: int = 10
) -> list:
    """
    Lista os pedidos de um cliente.
    
    Args:
        cliente_id: ID único do cliente
        status: filtrar por status ('pendente', 'processando', 'entregue', 'cancelado')
        limit: máximo de resultados (1-50)
    
    Returns: lista de pedidos resumidos
    """
    params = {"cliente_id": cliente_id, "limit": min(limit, 50)}
    if status:
        params["status"] = status
    
    response = httpx.get("https://api.empresa.com/pedidos", params=params)
    return response.json()["pedidos"]

@mcp.tool()
def atualizar_status_pedido(pedido_id: str, novo_status: str, motivo: str) -> dict:
    """
    Atualiza o status de um pedido.
    
    ATENÇÃO: Esta é uma ação com efeito colateral. Confirme com o usuário antes de executar.
    
    Args:
        pedido_id: ID do pedido
        novo_status: novo status ('processando', 'entregue', 'cancelado')
        motivo: justificativa para a mudança (obrigatório)
    
    Returns: pedido atualizado com novo status
    """
    valid_statuses = ["processando", "entregue", "cancelado"]
    if novo_status not in valid_statuses:
        return {"error": f"Status inválido. Use: {valid_statuses}"}
    
    response = httpx.patch(
        f"https://api.empresa.com/pedidos/{pedido_id}",
        json={"status": novo_status, "motivo": motivo}
    )
    return response.json()

# ============================================================
# RESOURCES — dados que o agente pode ler
# ============================================================

@mcp.resource("empresa://catalogo/produtos")
def catalogo_produtos() -> str:
    """Catálogo completo de produtos da empresa em formato markdown."""
    products = httpx.get("https://api.empresa.com/produtos").json()
    
    md = "# Catálogo de Produtos\n\n"
    for product in products:
        md += f"## {product['nome']}\n"
        md += f"**SKU:** {product['sku']}\n"
        md += f"**Preço:** R$ {product['preco']:.2f}\n"
        md += f"**Estoque:** {product['estoque']} unidades\n\n"
    
    return md

@mcp.resource("empresa://relatorio/{data}")
def relatorio_diario(data: str) -> str:
    """
    Relatório diário de operações.
    
    Args:
        data: data no formato YYYY-MM-DD
    """
    report = httpx.get(f"https://api.empresa.com/relatorios/{data}").json()
    return f"""# Relatório de {data}
    
**Pedidos processados:** {report['pedidos_total']}
**Receita:** R$ {report['receita']:.2f}
**Cancelamentos:** {report['cancelamentos']}
**NPS:** {report['nps']:.1f}
    """

# ============================================================
# PROMPTS — templates reutilizáveis
# ============================================================

@mcp.prompt()
def analise_cliente(cliente_id: str, pedidos: str) -> str:
    """Template para análise do perfil de um cliente."""
    return f"""Analise o perfil do cliente {cliente_id} com base nos pedidos abaixo.

Identifique:
1. Frequência de compra
2. Ticket médio
3. Categorias preferidas
4. Tendências e padrões
5. Risco de churn
6. Oportunidades de upsell

Pedidos:
{pedidos}

Forneça análise em formato de relatório executivo com recomendações acionáveis."""

# ============================================================
# EXECUTAR
# ============================================================

if __name__ == "__main__":
    # stdio: para uso local (Claude Desktop, Cursor)
    mcp.run(transport="stdio")
    
    # SSE: para uso remoto (serviço HTTP)
    # mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

### Servidor MCP para banco de dados interno

```python
from mcp.server.fastmcp import FastMCP
import sqlite3  # ou psycopg2 para PostgreSQL
import pandas as pd

mcp = FastMCP("analytics-db")

def get_db():
    return sqlite3.connect("analytics.db")

@mcp.resource("db://schema")
def database_schema() -> str:
    """Schema do banco de dados de analytics."""
    conn = get_db()
    tables = conn.execute("SELECT name FROM sqlite_master WHERE type='table'").fetchall()
    
    schema = "# Schema do Banco Analytics\n\n"
    for (table_name,) in tables:
        cols = conn.execute(f"PRAGMA table_info({table_name})").fetchall()
        schema += f"## {table_name}\n"
        for col in cols:
            schema += f"- `{col[1]}` ({col[2]})\n"
        schema += "\n"
    
    return schema

@mcp.tool()
def query_analytics(sql: str) -> str:
    """
    Executa query SQL no banco de analytics.
    
    IMPORTANTE: Apenas SELECT é permitido. Queries destrutivas são bloqueadas.
    Limite: 1000 linhas máximo.
    
    Returns: resultado em formato tabela markdown
    """
    # Validação de segurança
    sql_upper = sql.upper().strip()
    forbidden = ["INSERT", "UPDATE", "DELETE", "DROP", "CREATE", "ALTER", "TRUNCATE", "EXEC"]
    
    if not sql_upper.startswith("SELECT"):
        return "Erro: apenas queries SELECT são permitidas."
    
    if any(word in sql_upper for word in forbidden):
        return f"Erro: operações proibidas detectadas: {forbidden}"
    
    try:
        conn = get_db()
        df = pd.read_sql_query(sql, conn)
        
        if len(df) > 1000:
            df = df.head(1000)
            note = f"\n\n*Resultado truncado: mostrando 1000 de {len(df)} linhas.*"
        else:
            note = ""
        
        return df.to_markdown(index=False) + note
    
    except Exception as e:
        return f"Erro na query: {str(e)}"
```

### Servidor MCP com autenticação

```python
import os
from mcp.server.fastmcp import FastMCP
from functools import wraps

mcp = FastMCP("secure-server")

# Verificação de token (para servidores SSE/HTTP)
def require_auth(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        # Em servidores stdio, a auth é gerenciada pelo processo pai
        # Em SSE, verificar header Authorization
        return func(*args, **kwargs)
    return wrapper

# Carregar credenciais do ambiente (nunca hardcoded)
API_TOKEN = os.environ.get("INTERNAL_API_TOKEN")
if not API_TOKEN:
    raise ValueError("INTERNAL_API_TOKEN não configurado")

@mcp.tool()
def operacao_segura(dados: str) -> str:
    """Operação que requer autenticação."""
    # API_TOKEN disponível no closure, sem hardcoding
    response = httpx.post(
        "https://api.interna.com/operacao",
        headers={"Authorization": f"Bearer {API_TOKEN}"},
        json={"dados": dados}
    )
    return response.text
```

### Publicar e distribuir

**Para uso local (Claude Desktop / Cursor):**
```bash
# O servidor roda como processo filho — sem publicação necessária
# Configurar no claude_desktop_config.json ou cursor settings
```

**Para uso como serviço (SSE):**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY server.py .
EXPOSE 8000
CMD ["python", "server.py", "--transport", "sse", "--port", "8000"]
```

**Publicar no PyPI (para distribuição):**
```bash
# Após empacotar com pyproject.toml
pip install build twine
python -m build
twine upload dist/*

# Outros instalam com:
pip install meu-servidor-mcp
```

---

## Aplicação Prática

**Tempo estimado para criar um servidor MCP básico:**
- Setup + primeira tool: 15 minutos
- Servidor completo com 5-10 tools: 2-4 horas
- Com autenticação e deploy em Docker: +2 horas

**Quando vale criar vs. adaptar:**
- Adaptar servidor existente (fork + modificação): API similar ao que já existe
- Criar do zero: API proprietária, sistema legado, requisitos muito específicos

---

## Conexões

- [07-01 — O que é MCP](07-01-O-que-e-MCP.md): protocolo que você está implementando
- [07-02 — Servidores Prontos](07-02-Servidores-MCP-Prontos.md): verificar se já existe antes de criar
- [07-04 — MCP Backend para Agentes](07-04-MCP-Backend-para-Agentes.md): usar servidores MCP em agentes customizados
- [08-02 — Rules, Skills e Hooks](../Modulo-08-Cursor-Plataforma-Agentes/08-02-Rules-Skills-e-Hooks.md): como o Cursor usa seus servidores MCP

---

## Recursos Complementares

- **FastMCP (Python SDK)**: [github.com/jlowin/fastmcp](https://github.com/jlowin/fastmcp) — abstração mais simples sobre o SDK oficial
- **MCP Python SDK**: [github.com/modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) — SDK oficial
- **"Build Your First MCP Server"** — Anthropic tutorial oficial
- **MCP Inspector**: [github.com/modelcontextprotocol/inspector](https://github.com/modelcontextprotocol/inspector) — ferramenta para testar servidores MCP

---

## Auto-reflexão

1. Qual é a diferença entre `@mcp.tool()` e `@mcp.resource()`? Dê um exemplo de quando usar cada um.
2. Por que validar e bloquear queries SQL destrutivas é responsabilidade do servidor MCP, não do agente que chama?
3. Um servidor MCP stdio vs. SSE: quais são as implicações de segurança de cada um?
4. Você tem uma API REST interna com 50 endpoints. Você vai criar uma tool MCP para cada endpoint? Como você decidiria quais endpoints expor como tools?

---

## Próximo Passo

### [→ 07-04 — MCP Backend para Agentes](07-04-MCP-Backend-para-Agentes.md)


---

*[← 07-02 — Servidores MCP Prontos](07-02-Servidores-MCP-Prontos.md) · [↑ M07 — MCP e Protocolos Modernos](07-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
