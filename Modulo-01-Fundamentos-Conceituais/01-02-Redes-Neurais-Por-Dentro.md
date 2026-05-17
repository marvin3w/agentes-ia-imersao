# [01-02] Redes Neurais por Dentro

## Por que isso importa?

Toda vez que alguém menciona "o modelo tem 70 bilhões de parâmetros" ou "foi treinado com backpropagation", você precisa saber o que isso significa para tomar decisões inteligentes — sobre custo de inferência, sobre o que pode e não pode ser ajustado, sobre por que modelos menores às vezes surpreendem.

---

## Conceito Central

**Uma rede neural é uma função matemática com muitas variáveis ajustáveis, organizada em camadas.**

A analogia do neurônio biológico é famosa mas enganosa — o que importa é a matemática: cada "neurônio" artificial recebe números como entrada, aplica uma operação matemática simples, e passa um número adiante. A magia emerge da combinação de milhões desses neurônios em camadas.

```
Input → [Camada 1] → [Camada 2] → ... → [Camada N] → Output
         (pesos)       (pesos)              (pesos)
```

Os "pesos" são os parâmetros — os números ajustáveis. Treinar a rede é encontrar os valores de pesos que fazem o output ser o mais correto possível.

---

## Aprofundamento

### A intuição de "camadas"

Por que usar camadas em vez de uma operação direta?

Camadas permitem **hierarquia de abstrações**. Numa rede que reconhece rostos:
- Camada 1: detecta bordas e gradientes de cor
- Camada 2: combina bordas em formas (olhos, nariz)
- Camada 3: combina formas em estruturas (rosto, fundo)
- Camada final: "é ou não é um rosto"

Cada camada aprende uma abstração mais sofisticada. Em LLMs, as primeiras camadas detectam padrões sintáticos simples; as últimas, conceitos semânticos complexos como raciocínio causal.

### Backpropagation — como a rede aprende

O algoritmo de backpropagation é o motor de aprendizado de todas as redes neurais modernas. A intuição:

1. **Forward pass:** a rede processa um input e produz um output
2. **Calcular erro:** compara o output com o correto (função de perda)
3. **Backward pass:** propaga o erro de volta pela rede, calculando a contribuição de cada peso para o erro
4. **Atualização:** ajusta cada peso um pouco na direção que reduz o erro (gradiente descendente)

Repita bilhões de vezes com bilhões de exemplos. É isso que custa meses de computação e US$ milhões para treinar um LLM grande.

**Por que você precisa saber disso:** entender backprop explica por que fine-tuning é caro (precisa recalcular gradientes para toda a rede), por que LoRA/PEFT existem (ajustam só uma fração dos pesos), e por que modelos maiores precisam de mais dados para não fazer overfitting.

### Funções de ativação — a não-linearidade que torna tudo possível

Sem funções de ativação não-lineares, uma rede de 100 camadas seria matematicamente equivalente a uma rede de 1 camada. As funções de ativação (ReLU, GELU, SiLU) são o que permite às redes aprenderem padrões complexos.

Nos LLMs modernos, a função GELU é comum. Você raramente precisará ajustá-la, mas saber que existe explica por que há limites no que a arquitetura pode aprender.

### Parâmetros — o que "70 bilhões" significa na prática

| Modelo | Parâmetros | RAM necessária (FP16) | Velocidade |
|---|---|---|---|
| GPT-2 | 1.5B | ~3 GB | Muito rápido |
| LLaMA 3 8B | 8B | ~16 GB | Rápido |
| LLaMA 3 70B | 70B | ~140 GB | Lento (requer múltiplas GPUs) |
| GPT-4 (estimado) | ~1 trilhão | Infraestrutura de data center | API only |

Isso explica por que LLMs grandes só rodam na nuvem e por que modelos locais (Ollama, LM Studio) usam versões menores ou quantizadas.

**Quantização:** técnica que reduz a precisão dos pesos (de 32-bit para 4-bit) para economizar memória, com pequena perda de qualidade. Quando você vê "GGUF Q4_K_M", está vendo quantização de 4 bits no formato GGUF.

---

## Aplicação Prática

**Experimento mental — escolha de modelo:**

Você precisa de um modelo para processar 10.000 documentos por hora. O modelo "grande" tem qualidade superior mas custa 10x mais por token. O modelo "pequeno" tem qualidade boa suficiente para a tarefa. Qual escolhe?

A resposta depende de entender que:
- Qualidade não é binária — existe uma curva de rendimento decrescente
- Custo de inferência escala linearmente com parâmetros
- Para tarefas estruturadas (extração, classificação), modelos menores frequentemente competem com maiores

**Checklist antes de escolher um modelo:**

- [ ] Preciso de raciocínio complexo e criatividade? → Modelo grande (GPT-4, Claude 3.5+)
- [ ] Preciso de velocidade e baixo custo em tarefas estruturadas? → Modelo médio (Claude Haiku, GPT-4o mini)
- [ ] Preciso de privacidade ou offline? → Modelo local quantizado (LLaMA 3 8B Q4)
- [ ] Preciso de fine-tuning específico? → Modelo pequeno base (LLaMA 3 8B não-instruct)

---

## Conexões

- [01-01 — Como Máquinas Aprendem](01-01-Como-Maquinas-Aprendem.md): o contexto de onde redes neurais se encaixam
- [01-03 — Transformers e Atenção](01-03-Transformers-e-Atencao.md): arquitetura específica usada em LLMs
- [M07 — MCP](../Modulo-07-MCP-Protocolos-Modernos/07-00-Visao-Geral.md): como modelos locais vs. API afeta a arquitetura do sistema

---

## Recursos Complementares

- **3Blue1Brown — "Gradient descent, how neural networks learn"** (YouTube, 21 min)
- **Harvard "Introduction to Neural Networks and Deep Learning with Python"** ($299) — aprofundamento técnico com implementação
- **Andrej Karpathy — "Neural Networks: Zero to Hero"** (YouTube, série gratuita) — implementa backprop do zero em Python

---

## Auto-reflexão

1. Por que um modelo de 70B parâmetros é mais lento que um de 7B, mesmo com hardware equivalente?
2. O que é quantização e qual o trade-off entre usar Q4 vs. Q8?
3. Por que fine-tuning é caro mas inferência (usar o modelo) é relativamente barato?
4. Se backpropagation requer ver o "erro" da rede, como um LLM sabe qual é o erro correto durante o treinamento?

---

*[01-02] | Anterior: [01-01](01-01-Como-Maquinas-Aprendem.md) | Próximo: [01-03](01-03-Transformers-e-Atencao.md)*
*Referência externa: Harvard Neural Networks; 3Blue1Brown*
