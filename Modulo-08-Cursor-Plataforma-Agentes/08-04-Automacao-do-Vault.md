# [08-04] Automação do Vault — O Sistema que Se Autogerencia

## Por que isso importa?

Este documento fecha o ciclo da imersão com o caso mais concreto possível: o próprio vault Marcus-Docs como exemplo vivo de um sistema de agentes em produção. Ao invés de um exemplo genérico, você vê como todos os conceitos — RAG, agentes, MCP, Rules, Skills, orquestração — funcionam integrados em um sistema real.

---

## Conceito Central

**Um "vault inteligente" é um segundo cérebro que não apenas armazena informação, mas processa, conecta e age sobre ela. O Marcus-Docs é um exemplo de vault com: agentes especializados (personas via Rules), ferramentas externas (MCP), pipelines automatizados (scripts Python), e um subsistema de memória (NDJSON logs, transcripts).**

---

## Aprofundamento

### Arquitetura geral do vault como sistema de agentes

```
┌─────────────────────────────────────────────────────────────────┐
│                    MARCUS-DOCS VAULT                            │
│                                                                 │
│  CAMADA DE ORQUESTRAÇÃO                                         │
│  ├── Wesley (V3) — demandas formais com artefatos rastreáveis   │
│  ├── Edy (V4) — orquestração rápida com routing                 │
│  ├── Arth — sessão diária, cockpit, encaminhamentos             │
│  └── AI Master — composição multi-agente                        │
│                                                                 │
│  CAMADA DE AGENTES ESPECIALIZADOS (Rules)                       │
│  ├── Tech Lead — revisão PHP/WordPress                          │
│  ├── Business Analyst — EF/ET para demandas                     │
│  ├── Chief of Staff — triagem proativa                          │
│  └── [12+ outros agentes especializados]                        │
│                                                                 │
│  CAMADA DE SKILLS (Capacidades)                                 │
│  ├── jira-demand-doc — criar docs de demanda                    │
│  ├── daily-standup-prep — montar daily                          │
│  ├── panorama-de-acervo — inventário semântico                  │
│  └── [10+ skills]                                               │
│                                                                 │
│  CAMADA DE INTEGRAÇÕES (MCP)                                    │
│  ├── GitHub — commits, PRs, issues                              │
│  ├── Jira/Confluence — demandas, documentação                   │
│  ├── Slack — comunicação com o time                             │
│  └── Google Workspace — Drive, Gmail, Calendar                  │
│                                                                 │
│  CAMADA DE MEMÓRIA (NDJSON Logs)                                │
│  ├── operacoes.ndjson — histórico de operações O1-O5            │
│  ├── briefing-audits.ndjson — rastreamento de navegação         │
│  ├── transcripts/ — histórico de sessões                        │
│  └── routing-decisions.ndjson — decisões de roteamento          │
│                                                                 │
│  CAMADA DE AUTOMAÇÃO (Scripts Python)                           │
│  ├── transcript-miner-run.py — minerar entidades de sessões     │
│  ├── harvest_git.py — coletar evidências de commits             │
│  ├── worktree-guardian.py — monitorar saúde de worktrees        │
│  └── glossario-candidatos-imersao.py — detectar gaps de aprendizado│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Fluxo de uma demanda no vault

**Ponto de entrada: mensagem no Slack**
```
Marcus recebe mensagem de um stakeholder no Slack
  ↓
[Skill: process-slack-pool] — captura a conversa em _Pool/
  ↓
[Script: auto_process_pool.py] — classifica e roteia automaticamente
  ↓
[Skill: review-routing] — Marcus revisa as decisões de roteamento
  ↓
Se é demanda técnica → [Skill: jira-demand-doc] → doc de demanda criado
  ↓
Se é demanda de produto → [Orquestrador Edy] → fase de análise
  ↓
[Agente: Business Analyst] → gera EF + ET
  ↓
[Agente: Tech Lead] → review técnico da solução
  ↓
[Desenvolvedor Gran] → implementação + stage-only
  ↓
[Git commit → Jira MCP] → registro e rastreabilidade
```

### O subsistema de memória (NDJSON)

Cada operação do vault gera uma linha NDJSON em `Meta-Marcus-Docs/Logs/`:

```json
// operacoes.ndjson — cada linha é uma operação
{"iso":"2025-05-17T10:30:00-03:00","session_id":"sess_abc","tipo_operacao":"O2","descricao":"Criação de imersão em IA Agentes — M03 completo","arm":"baseline","nps":null}
{"iso":"2025-05-17T11:15:00-03:00","session_id":"sess_abc","tipo_operacao":"O3","descricao":"Commit e push dos módulos M01-M03","arm":"baseline","nps":null}
```

**Tipos de operação (O1-O5):**
- **O1:** Leitura e análise (sem escrita)
- **O2:** Criação de conteúdo no vault
- **O3:** Operação git (commit, push, merge)
- **O4:** Comunicação externa (Slack, Jira, email)
- **O5:** Execução de script/automação

Este log permite auditoria retrospectiva, análise de produtividade, e identificação de padrões.

### Pipeline de transcript mining (memória episódica do sistema)

Os transcripts das conversas com o agente são processados para extrair conhecimento:

```bash
# Rodar o pipeline de mineração
python3 Meta-Marcus-Docs/Scripts/transcript-miner-run.py

# O pipeline:
# 1. Identifica transcripts não processados
# 2. Extrai entidades (termos técnicos, siglas, projetos)
# 3. Grava em transcripts-entities.ndjson
# 4. Alimenta o Glossário Vivo
# 5. Identifica candidatos a novos módulos de imersão
```

```python
# glossario-candidatos-imersao.py — como este módulo foi gerado
# Lê transcripts-entities.ndjson
# Agrupa entidades por domínio semântico
# Pontua por frequência e recência
# Retorna: "IA Aplicada" com score 25.730 — 166 sessões
# → Trigger para criar este curso de 8 módulos
```

### Guardrails como código — enforcement automático

```python
# .cursor/hooks/jira-write-gate.py
# Roda ANTES de qualquer escrita no Jira
# Impede chamadas não confirmadas

# Resultado: zero incidentes de escrita acidental no Jira
# após implementação (antes: incidente Comm-2 em 2026-05-08)
```

Este é um exemplo de padrão **Mecânico** vs. **Probabilístico**:
- **Probabilístico:** Rule que diz "peça confirmação antes de escrever" → o agente pode esquecer
- **Mecânico:** Hook que intercepta a chamada MCP antes de executar → impossível de "esquecer"

### Evoluindo o vault — o meta-loop

O vault melhora a si mesmo via:

1. **Feedback das operações** → log em `operacoes.ndjson`
2. **Análise de logs** → agent data-analyst identifica padrões
3. **Decisões de melhoria** → FED (Framework de Estruturação de Decisão)
4. **Novos guardrails** → Rules ou Hooks criados
5. **Novas Skills** → procedimentos que antes eram ad-hoc
6. **Commit no vault** → a melhoria persiste

```
[Sessão operacional]
      ↓
[Log NDJSON]
      ↓
[Transcript mining]
      ↓
[Agent: Data Analyst] → identifica oportunidade
      ↓
[Agent: Chief of Staff] → propõe melhoria
      ↓
[Marcus aprova]
      ↓
[Agent: Platform Engineer] → implementa hook/script
      ↓
[Commit → Push] → melhoria persiste
      ↓
[Próximas sessões se beneficiam]
```

### Próximos passos do vault (aplicação prática deste curso)

Com os conceitos deste curso, as melhorias naturais do vault são:

**RAG sobre o vault:**
Indexar os markdowns do vault em um banco vetorial (Qdrant local) para permitir "busca semântica no vault" como ferramenta de agente.

**Pipeline de avaliação automática:**
Usar RAGAS/LangSmith para medir qualidade das respostas do agente ao longo do tempo. Detectar degradação antes do usuário reclamar.

**Sistema multiagente explícito:**
Implementar o AI Master como grafo LangGraph formal, com Vigilante, Socrático e outros como subgrafos especializados com handoffs tipados.

**MCP server do vault:**
Criar um servidor MCP que expõe o vault como recurso: `vault://demands/Q-037`, `vault://cockpit`, `vault://glossary`. Qualquer agente externo poderia consumir o vault via MCP.

---

## Conexões

Todos os módulos desta imersão:
- [M01 — Fundamentos](../Modulo-01-Fundamentos-Conceituais/) → a base conceitual que permite entender por que o vault funciona assim
- [M02 — NLP e Embeddings](../Modulo-02-NLP-Embeddings-Prompt/) → embeddings para RAG sobre o vault
- [M03 — RAG](../Modulo-03-RAG-Memoria-Contextual/) → memória semântica do vault
- [M04 — Agentes](../Modulo-04-Agentes-de-IA/) → as personas como agentes com Tools
- [M05 — Orquestração](../Modulo-05-Orquestracao-Fluxos/) → Wesley/Edy como grafos LangGraph
- [M06 — Multiagente](../Modulo-06-Sistemas-Multiagente/) → AI Master como supervisor
- [M07 — MCP](../Modulo-07-MCP-Protocolos-Modernos/) → integrações GitHub, Jira, Slack
- [M08 — Cursor](../) → o ambiente onde tudo isso roda

---

## Recursos para continuar

**Próximos passos de aprendizado além desta imersão:**

- **LangGraph Platform (Cloud)**: deploy de agentes em produção com escalabilidade
- **Anthropic's "Prompt Engineering Guide"**: aprofundar em prompts para agentes avançados
- **"Building LLM-Powered Apps"** — Valentina Alto (Packt, 2024): livro técnico completo
- **AI Engineer Summit** (YouTube): conferência com os principais practitioners
- **DeepLearning.AI Short Courses**: [deeplearning.ai/courses](https://www.deeplearning.ai/courses) — cursos curtos de alta qualidade, atualizados

**Comunidades:**
- LangChain Discord: [discord.gg/langchain](https://discord.gg/langchain)
- Cursor Forum: [forum.cursor.com](https://forum.cursor.com)
- r/LocalLLaMA (Reddit): comunidade sobre modelos locais
- Latent Space (podcast + newsletter): [latent.space](https://www.latent.space)

---

## Auto-reflexão — Síntese da Imersão

1. Antes desta imersão, qual era o seu modelo mental de "como o agente no Cursor funciona"? Como ele mudou?
2. Identifique um sistema ou processo no seu trabalho atual que se beneficiaria de um agente com RAG. Descreva a arquitetura que você usaria.
3. O vault Marcus-Docs usa guardrails mecânicos (hooks) e probabilísticos (rules). Por que as duas abordagens são necessárias e complementares?
4. Qual é o próximo experimento que você vai fazer com os conceitos desta imersão? Seja específico: qual problema, qual tecnologia, qual resultado esperado.

---

## 🏁 Imersão Concluída

Você percorreu os 8 módulos da Imersão em Agentes de IA. O próximo passo é construir algo real.

**[→ Voltar ao Índice do Curso](../README.md)**


---

*[← 08-03 — Cursor CLI e SDK](08-03-Cursor-CLI-e-SDK.md) · [↑ M08 — Cursor como Plataforma](08-00-Visao-Geral.md) · [🏠 Índice do Curso](../README.md)*
