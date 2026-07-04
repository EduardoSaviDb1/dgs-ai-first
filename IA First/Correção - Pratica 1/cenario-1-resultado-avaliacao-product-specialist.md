# Avaliação Consolidada — Product Specialist (Cenário 1)

## Escopo validado
- Exercício 1.1: mapeamento de intent com engenharia de contexto
- Exercício 1.2: design de jornada com componente de IA
- Exercício 1.3: especificação de requisitos de RAG

Arquivos avaliados:
- `Prática 1/exercicios/exercicio-1-1-product-specialist.md`
- `Prática 1/exercicios/exercicio-1-2-product-specialist.md`
- `Prática 1/exercicios/exercicio-1-2-product-specialist-fluxo.puml`
- `Prática 1/exercicios/exercicio-1-3-product-specialist.md`

Rubricas usadas:
- `Correção - Pratica 1/cenario-1-avaliacao-foundation.md`
- `Correção - Pratica 1/cenario-1-avaliacao-product-specialist.md`

---

## Avaliação do Exercício 1.1

### Resumo
Entregável forte em engenharia de contexto, com progressive disclosure real em 3 etapas e boa leitura de risco de contradição documental. A reflexão sobre orçamento de atenção e context rot está consistente com o objetivo do exercício.

### Scores por Dimensão

| Dimensão | Score | Justificativa |
|----------|-------|---------------|
| D1 — Domínio Conceitual | 3 | Demonstra entendimento sólido de context engineering: sequenciamento em camadas, priorização, e análise comparativa progressivo vs tudo de uma vez. |
| D2 — Uso de Ferramentas | 2 | Há uso estruturado do Claude e outputs por etapa, porém os prompts da etapa 2/3 poderiam ter constraints mais explícitos de formato e critérios para elevar robustez. |
| D3 — Qualidade do Entregável | 3 | Entregável completo, claro e acionável; riscos e tratamento estão bem documentados. |
| D4 — Pensamento Crítico | 3 | Não houve aceitação acrítica: há análise própria da qualidade por etapa e implicações para discovery humano. |
| D5 — Aplicabilidade ao Projeto | 3 | Usa evidências específicas da NovaTech (PROC-042 vs v2, FAQ, contexto operacional) com boa aderência ao domínio. |

**Score do exercício: 2.8**

### Verificação de Armadilhas
- Armadilha do exercício: fornecer os 5 documentos completos no primeiro prompt.
- Resultado: não caiu na armadilha; a estratégia adotada foi progressiva e alinhada ao objetivo.

### Pontos Fortes
- Estratégia de 3 etapas bem justificada e executada.
- Mapeamento de riscos concreto e orientado a ação no discovery.
- Reflexão madura sobre degradação de qualidade com contexto excessivo.

### Pontos de Melhoria
- Tornar os prompts de etapa 2 e 3 mais prescritivos (formato de saída, critérios de severidade, estrutura de comparação).
- Acrescentar evidência explícita de iteração no chat (ex.: ajustes de prompt entre rodadas na mesma etapa).

### Classificação
Aprovado com distinção

### Tópicos da Trilha para Reforço
Não aplicável para score desta faixa.

---

## Avaliação do Exercício 1.2

### Resumo
Jornada completa com os três caminhos exigidos (principal, fallback e feedback), incluindo guardrails e fechamento de loop de curadoria. O diagrama está coerente com o fluxo textual e legível para públicos não técnicos.

### Scores por Dimensão

| Dimensão | Score | Justificativa |
|----------|-------|---------------|
| D1 — Domínio Conceitual | 3 | Evidencia entendimento de limitações de IA no atendimento e necessidade de feedback loop contínuo em RAG. |
| D2 — Uso de Ferramentas | 2 | O artefato final está bom, mas há pouca evidência explícita de iteração entre versões no uso de Claude/Claude Design. |
| D3 — Qualidade do Entregável | 3 | Fluxos completos, critérios de saída e definição operacional de feedback bem estruturados. |
| D4 — Pensamento Crítico | 3 | Decisões de produto justificadas por dados do discovery e por risco operacional. |
| D5 — Aplicabilidade ao Projeto | 3 | Forte contextualização no domínio NovaTech (atendimento, escala, supervisão, curadoria). |

**Score do exercício: 2.8**

### Verificação de Armadilhas
Nenhuma armadilha específica definida na rubrica para este exercício.

### Pontos Fortes
- Cobertura integral de caminho feliz e exceções.
- Guardrails úteis para reduzir alucinação e ambiguidade.
- Fluxo de feedback com rastreabilidade e encaminhamento para curadoria.

### Pontos de Melhoria
- Incluir evidência de iteração entre versão inicial e refinada da jornada/diagrama.
- Tornar ao menos um guardrail ainda mais específico do domínio (ex.: regra explícita para carga perigosa).

### Classificação
Aprovado com distinção

### Tópicos da Trilha para Reforço
Não aplicável para score desta faixa.

---

## Avaliação do Exercício 1.3

### Resumo
Entregável excelente, cobrindo integralmente os cinco blocos requeridos e expandindo com requisito adicional de confiança mínima. A iteração v1→v2 está clara, com rastreabilidade direta entre gaps apontados e ajustes realizados.

### Scores por Dimensão

| Dimensão | Score | Justificativa |
|----------|-------|---------------|
| D1 — Domínio Conceitual | 3 | Compreensão robusta de RAG, curadoria, contradição documental e limites de confiança. |
| D2 — Uso de Ferramentas | 3 | Uso efetivo da IA como revisor: v1, feedback estruturado e refinamento verificável em v2. |
| D3 — Qualidade do Entregável | 3 | Requisitos completos, claros e testáveis por QA, com critérios objetivos. |
| D4 — Pensamento Crítico | 3 | Gaps relevantes identificados e tratados com decisões de produto maduras, sem delegação cega ao modelo. |
| D5 — Aplicabilidade ao Projeto | 3 | Totalmente aderente ao contexto NovaTech, incluindo processos, áreas e cadência operacional. |

**Score do exercício: 3.0**

### Verificação de Armadilhas
Nenhuma armadilha específica definida na rubrica para este exercício.

### Pontos Fortes
- Cobertura completa das áreas exigidas com testabilidade real.
- Tratamento maduro de contradições e ausência de resposta.
- Iteração explícita e bem governada entre versões.

### Pontos de Melhoria
- Separar com mais clareza o que é requisito de produto versus decisão de arquitetura operacional em alguns trechos.
- Definir, quando possível, faixas-alvo iniciais para métricas de aceitação por requisito.

### Classificação
Aprovado com distinção

### Tópicos da Trilha para Reforço
Não aplicável para score desta faixa.

---

## Resultado Final Consolidado

- Score E1.1: 2.8
- Score E1.2: 2.8
- Score E1.3: 3.0

**Média final do cenário: 2.9**

**Classificação final: Aprovado com distinção**

## Observação sobre evidências
Os conteúdos avaliados demonstram alta qualidade técnica e de produto. Para avaliação formal de certificação, recomenda-se anexar evidências explícitas de uso das ferramentas (histórico de prompts, versões intermediárias e, quando aplicável, captura/export do Claude Design), pois isso impacta diretamente D2.