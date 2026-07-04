# Reavaliação Cruzada — Respostas do Assistente NovaTech
**Fase:** Governança e Validação (Cenário 3)
**Data:** 2026-07-04
**Autores:** Product Specialist + QA
**Insumos:** Avaliação QA (tabela), Avaliação PS (sessão anterior), Anexo A, guardrails v1.1

---

## Metodologia de cruzamento

Cada resposta foi avaliada por dois revisores independentes (QA e PS). Este documento
registra: onde as avaliações **convergem**, onde **divergem com justificativa**, a
**classificação canônica de tipo de erro**, e o **ajuste de produto** proposto.

### Taxonomia de tipos de erro usada neste documento

| Tipo | Definição operacional |
|---|---|
| **Alucinação** | O modelo gerou informação factual não presente em nenhum chunk recuperado nem em nenhum documento da base — fabricação ativa. |
| **Fonte não confiável** | O modelo citou ou se baseou em documento informal (FAQ não validado) como se fosse normativo. |
| **Informação incompleta** | A informação presente está correta, mas omite distinções, exceções ou contexto operacionalmente relevante para o atendente. |
| **Citação incorreta** | A fonte existe e é normativa, mas a seção/versão/localização citada está errada ou aponta para conteúdo diferente do informado. |
| **Encaminhamento incorreto** | A regra substantiva está certa, mas o destino de escalada ou o próximo passo indicado está errado. |

---

## Resposta 1 — "Qual o prazo de devolução para produtos standard?"

### Cruzamento das avaliações

| Dimensão | Avaliação QA | Avaliação PS | Convergência |
|---|---|---|---|
| Classificação geral | Parcialmente correta | Parcialmente correta | ✅ Convergente |
| Problema principal | Processo incompleto; falta seção 3.3 | Seção citada errada (3.2 em vez de 3.1) | ⚠️ Divergente |
| Conteúdo do prazo (7 dias úteis) | Correto | Correto | ✅ Convergente |

### Análise da divergência

As duas avaliações identificaram problemas reais mas em dimensões diferentes. A avaliação QA focou na **incompletude do processo** (faltam os passos do 3.3, prazos de triagem, coleta reversa, reembolso). A avaliação PS focou na **citação de seção incorreta** (3.2 é a seção de exceções — cargas inelegíveis — que é o oposto da informação transmitida). Ambos os problemas coexistem na mesma resposta.

A citação errada de seção é o mais grave dos dois: um atendente que consulte POL-001 seção 3.2 para verificar a resposta vai encontrar a lista de cargas que **não** podem ser devolvidas, não o prazo geral. Num contexto de auditoria ou treinamento, isso invalida a rastreabilidade da resposta.

### Classificação canônica de erro

**Citação incorreta** (primário) + **Informação incompleta** (secundário)

- Seção 3.2 citada; seção correta é 3.1 (prazo geral)
- Procedimento parcial: falta CT-e obrigatório, prazo de triagem de 4h úteis, prazo de coleta reversa (2 dias úteis), prazo de reembolso (5 dias úteis)
- Ausência do critério de contagem: prazo começa na data de recebimento **confirmada no tracking**, não na data de entrega estimada

### Ajuste de produto proposto

**Camada:** `[CÓDIGO]` — injeção de metadado de seção via pipeline

**Problema raiz:** o modelo gerou a citação de seção por inferência semântica ("seção 3.2" ficou semanticamente próxima de "exceções ao prazo", que estava no contexto), não a extraiu do metadado do chunk. O guardrail G-DEVE-02 (versão injetada pelo código) cobre o campo `document_version`, mas não o campo `section`. A correção é estender o metadado de chunk para incluir `section_id` e `section_title`, injetados programaticamente no template de resposta — exatamente como já é feito com `document_version`.

**Ajuste secundário:** `[PROMPT]` — instrução de completude para procedimentos

Adicionar ao system prompt: *"Quando a resposta envolver um procedimento com múltiplas etapas (devolução, coleta reversa, reembolso), cite TODAS as etapas com seus respectivos prazos. Não resuma procedimentos com etapas omitidas."*

**Guardrail afetado:** extensão de G-DEVE-02 para cobrir `section_id` além de `document_version`.

---

## Resposta 2 — "Meu cliente é Silver. Qual o prazo de resolução?"

### Cruzamento das avaliações

| Dimensão | Avaliação QA | Avaliação PS | Convergência |
|---|---|---|---|
| Classificação geral | Parcialmente correta | Parcialmente correta | ✅ Convergente |
| Problema principal | Falta "horas úteis" | Omite distinção chamados gerais vs. incidentes críticos | ⚠️ Divergente em gravidade |
| Valor "48h" | Correto para chamados gerais | Correto para chamados gerais | ✅ Convergente |

### Análise da divergência

A avaliação QA identificou que "48h" sem qualificação pode ser lido como horas corridas — a omissão de "úteis" é relevante porque o relógio de SLA pausa fora do horário comercial para chamados gerais (SLA-2024, seção 5). Correto.

A avaliação PS identificou a omissão de maior impacto operacional: para **incidentes críticos**, o SLA de resolução de Silver é **8 horas** (não 48h) — e essas 8 horas **não pausam** fora do horário comercial. A diferença entre 8h contínuas e 48h úteis é a diferença entre uma violação de SLA em horas e em dias. Este é o erro de maior risco do par, porque o atendente não tem sinal de que precisa checar se o chamado se qualifica como incidente crítico antes de informar o prazo ao cliente.

Ambos os problemas são reais. A hierarquia de gravidade é: omissão de incidente crítico > omissão de "úteis".

### Classificação canônica de erro

**Informação incompleta** (dois níveis de incompletude sobrepostos)

- Nível 1 (QA): "48h" sem qualificar como "úteis" — omissão do regime de contagem
- Nível 2 (PS): ausência da distinção chamados gerais (48h úteis) vs. incidentes críticos (8h corridas) — omissão de categoria que muda o SLA em uma ordem de magnitude

### Ajuste de produto proposto

**Camada:** `[PROMPT]` — instrução de completude para respostas sobre SLA

Adicionar ao system prompt cláusula específica para queries de SLA:

*"Quando responder sobre SLA de qualquer tier, SEMPRE inclua: (1) a qualificação 'horas úteis' ou 'horas corridas' conforme o tipo de chamado; (2) a distinção entre chamados gerais e incidentes críticos com os respectivos prazos; (3) a nota de que o relógio de SLA não pausa para incidentes críticos de clientes Gold."*

**Camada complementar:** `[PIPELINE]` — detecção de query de SLA + injeção forçada de chunk

Assim como G-DEVE-05 injeta forçadamente o chunk de exceção de carga perigosa quando o padrão "carga perigosa + devolução" é detectado, o pipeline deve injetar forçadamente o chunk da seção 3 do SLA-2024 (definição de incidente crítico) sempre que uma query de SLA for detectada — independentemente do score. A distinção chamados gerais vs. críticos é operacionalmente obrigatória e não pode depender do retrieval acertar os dois chunks na mesma chamada.

**Guardrail novo proposto:** G-DEVE-07

> Quando a query envolver SLA de qualquer tier, o pipeline DEVE injetar o chunk da definição de incidente crítico (SLA-2024, seção 3) junto aos chunks de SLA por tier. `[CÓDIGO]`

---

## Resposta 3 — "Posso devolver carga perigosa classe 3?"

### Cruzamento das avaliações

| Dimensão | Avaliação QA | Avaliação PS | Convergência |
|---|---|---|---|
| Classificação geral | Incorreta | Correta com ressalva | ⚠️ Divergente em classificação |
| Regra substantiva (inelegibilidade) | Correta | Correta | ✅ Convergente |
| Encaminhamento | Incorreto (supervisor em vez de ramal 4500) | Incorreto (ramal 4500 em vez de supervisor) | ✅ Convergente no problema, convergente na solução |

### Análise da divergência de classificação

A divergência é terminológica, não factual. A avaliação QA classificou como "Incorreta" porque o encaminhamento errado é parte da resposta. A avaliação PS classificou como "Correta com ressalva" porque a regra principal (inelegibilidade de carga perigosa) está certa e a fonte está correta — o erro é no passo seguinte.

**Classificação canônica adotada: Parcialmente correta.** A regra substantiva e a citação estão corretas. O encaminhamento está errado. Classificar como "Incorreta" seria equiparar ao nível das respostas 4 e 6, que são erros de conteúdo factual — aqui o conteúdo está certo e o destino está errado. A distinção importa para priorização de correção.

### Classificação canônica de erro

**Encaminhamento incorreto**

- POL-001 seção 3.2 é explícita: o cliente deve contatar **Gestão de Riscos (ramal 4500)**, não o supervisor
- "Escalar para o supervisor" é um destino de hierarquia interna de atendimento — a Gestão de Riscos é a área técnica com autoridade para tratamento individual de cargas perigosas
- Consequência operacional: o chamado chega ao supervisor sem capacidade técnica para resolver, gerando reescalada e perda de tempo

### Ajuste de produto proposto

**Camada:** `[PROMPT]` — extensão da cláusula de carga perigosa no system prompt

A cláusula atual instrui a declarar inelegibilidade e orientar o ramal 4500. Adicionar texto que torna o destino inequívoco e proíbe a alternativa incorreta:

*"Para cargas perigosas inelegíveis para devolução padrão, o encaminhamento OBRIGATÓRIO é: 'O cliente deve contatar a Gestão de Riscos pelo ramal 4500 para tratamento individual.' NUNCA orientar escalada para supervisor — a Gestão de Riscos é a área responsável, não a hierarquia de atendimento."*

**Camada complementar:** `[CÓDIGO]` — lista de encaminhamentos canônicos por tema

A tabela de roteamento em `src/services/routing-table.ts` deve incluir o encaminhamento canônico para cada tema, e o pipeline deve injetar o encaminhamento correto no template de resposta quando o tema for detectado — não deixar o modelo formular o destino livremente.

```typescript
// src/services/routing-table.ts
const ROUTING_TABLE = {
  "devolucao_carga_perigosa": {
    destination: "Gestão de Riscos",
    contact: "ramal 4500",
    instruction: "O cliente deve contatar a Gestão de Riscos pelo ramal 4500 para tratamento individual."
  },
  // ... outros temas
} as const;
```

**Guardrail afetado:** G-NAO-05 — extensão para incluir encaminhamentos canônicos proibidos/obrigatórios por tema, não apenas a proibição genérica de autorizar.

---

## Resposta 4 — "Qual a política para carga danificada durante transporte?"

### Cruzamento das avaliações

| Dimensão | Avaliação QA | Avaliação PS | Convergência |
|---|---|---|---|
| Classificação geral | Incorreta | Incorreta | ✅ Convergente |
| Ausência de fonte | Identificada | Identificada | ✅ Convergente |
| Natureza do erro | "Faltam dados do processo completo" | Alucinação sobre gap documental | ⚠️ Divergente em diagnóstico |

### Análise da divergência de diagnóstico

Esta é a divergência mais importante do conjunto — e tem implicação direta no ajuste de produto.

A avaliação QA referenciou o "Item 41" do FAQ como base para o que faltou. O Item 41 do FAQ trata da **diferença entre SLA de resposta e SLA de resolução** — não de carga danificada. O item relevante para carga danificada é o **Item 38**, que descreve um processo via `sinistros@novatech.com.br` e via Jurídico. Esta referência ao Item 41 parece ser um erro de indexação na avaliação QA.

Mais importante: mesmo o Item 38 é **documento informal não validado**. Carga danificada em trânsito é GAP-02 — não existe normativo formal (POL ou PROC) cobrindo este tema. O assistente não apenas "faltou com dados do processo completo" — ele **fabricou** uma política ("reembolso integral quando comprovada negligência, mediante laudo técnico e fotos") que não existe em nenhum documento normativo da base, e atribuiu confiança "Alta" sem citar nenhuma fonte.

O diagnóstico correto não é incompletude — é alucinação sobre gap documental.

### Classificação canônica de erro

**Alucinação** (primário) + **Ausência de fonte** (consequência)

- Não existe normativo formal sobre carga danificada em trânsito na base (GAP-02)
- O conteúdo gerado ("reembolso integral", "negligência da transportadora", "laudo técnico") não está em nenhum dos 5 documentos do Anexo A — nem no normativo nem no FAQ
- A confiança "Alta" sem fonte citada é a combinação mais perigosa: o atendente não tem sinal de risco
- Violação dos guardrails G-NAO-04 (gap documental), G-NAO-01 (valor sem chunk), G-DEVE-03 (deveria ter declarado fallback)

### Ajuste de produto proposto

**Camada:** `[CÓDIGO]` — bloqueio pré-geração para gaps documentais conhecidos (G-NAO-04, já especificado)

Este é o ajuste mais crítico e estrutural. O pipeline deve classificar a intenção da query **antes** de chamar o retriever, e verificar se o tema mapeado é um dos gaps documentais configurados. Para GAP-02 (carga danificada), o pipeline retorna o fallback determinístico sem acionar o LLM:

```typescript
// src/services/search.ts
if (DOCUMENT_GAPS.includes(detectedTopic)) {
  return buildFallbackResponse({
    reason: "gap_documental",
    topic: detectedTopic,
    routing: ROUTING_TABLE[detectedTopic]
  });
  // LLM não é chamado
}
```

**Camada complementar:** `[HITL]` — ponto de revisão humana obrigatória para confiança "Alta" sem fonte

Enquanto o bloqueio via código não estiver implementado, o harness deve implementar um ponto HITL: respostas com `confidence_level: "high"` e `source_document: null` (ou ausente) devem ser retidas para revisão humana antes de chegar ao atendente. Esta combinação — alta confiança + sem fonte — é estruturalmente impossível num sistema com guardrails corretos e deve ser tratada como sinal de falha de pipeline, não como resposta válida.

**Nota sobre a avaliação QA:** a referência ao "Item 41" parece ser erro de indexação (Item 41 trata de SLA de resposta vs. resolução). Recomenda-se verificar se a intenção era referenciar o Item 38 (carga danificada). De qualquer forma, mesmo o Item 38 não é normativo — o diagnóstico correto permanece alucinação, não incompletude.

---

## Resposta 5 — "Qual o SLA do cliente Enterprise?"

### Cruzamento das avaliações

| Dimensão | Avaliação QA | Avaliação PS | Convergência |
|---|---|---|---|
| Classificação geral | Correta | Correta | ✅ Convergente |
| Ponto de melhoria | Não mencionado | Poderia citar SLA-2024 seção 1 | ✅ Convergente (melhoria, não erro) |

### Análise

Convergência total. Nenhum erro factual, comportamental ou de guardrail. O assistente aplicou corretamente G-NAO-03 (negação de tier inexistente), citou os tiers válidos e orientou verificação — sem fabricar um SLA para "Enterprise".

O único ponto de melhoria identificado por ambos: citar SLA-2024 v2024.1, seção 1 como base normativa para a afirmação de que apenas três tiers existem fortaleceria a rastreabilidade da resposta, mesmo sendo uma negação.

### Classificação canônica de erro

**Nenhum erro.** Resposta correta.

### Ajuste de produto proposto

**Camada:** `[PROMPT]` — instrução de citação para respostas de negação

Adicionar ao system prompt: *"Mesmo quando a resposta for uma negação (ex: 'este tier não existe'), cite o documento normativo que sustenta o conjunto de opções válidas. Exemplo: 'O tier Enterprise não existe na NovaTech. Os tiers vigentes são Gold, Silver e Standard, conforme SLA-2024 v2024.1, seção 1.'"*

Este ajuste é de baixa prioridade — a resposta está correta. Mas padroniza o comportamento para que negações também sejam auditáveis.

---

## Resposta 6 — "Posso enviar carga perigosa com frete expresso?"

### Cruzamento das avaliações

| Dimensão | Avaliação QA | Avaliação PS | Convergência |
|---|---|---|---|
| Classificação geral | Incorreta | Incorreta | ✅ Convergente |
| Problema identificado | Citação de FAQ informal | Citação de FAQ informal + gap documental | ✅ Convergente (PS mais completo) |
| Conteúdo da resposta ("Sim, com autorização") | Não avaliado separadamente | Identificado como sem cobertura normativa | ✅ Convergente |

### Análise

Convergência total na classificação. A avaliação PS foi mais completa ao identificar que o problema não é apenas a qualidade da fonte (FAQ informal) mas a ausência de qualquer normativo que ampare a informação — o Anexo A documenta explicitamente este caso como contradição/gap: *"FAQ Item 32 vs documentação formal: não existe documento formal (PROC ou POL) que defina esse processo."*

O assistente respondeu "Sim" com confiança "Alta" a uma pergunta sobre um processo que existe apenas como prática informal não documentada. A diferença em relação à resposta 4 é que aqui o modelo citou uma fonte (FAQ Item 32) — o que cria uma aparência de rastreabilidade que na verdade aponta para um documento não validado. Isso é potencialmente mais perigoso que ausência de fonte: o atendente que verificar o FAQ vai encontrar a informação lá, sem perceber que o FAQ não tem valor normativo.

### Classificação canônica de erro

**Fonte não confiável** (primário) + **Gap documental tratado como coberto** (secundário)

- FAQ-Atendimento é documento informal, não validado por Compliance ou Operações
- GAP-04 (frete expresso para carga perigosa) está na lista de gaps documentais — sem normativo formal
- A citação cria falsa aparência de rastreabilidade normativa
- Violação de G-NAO-04 (gap documental), G-DEVE-01 (FAQ informal não é fonte válida para resposta normativa)

### Ajuste de produto proposto

**Camada:** `[PIPELINE]` — tag de fonte no metadado de chunk + bloqueio de FAQ não validado

O ajuste raiz é garantir que o FAQ-Atendimento, enquanto não curado, **não seja indexado como fonte válida** para respostas normativas. Conforme REQ-01 do requirements.md, o FAQ só entra na base após curadoria formal por Compliance e Operações.

Se o FAQ estiver indexado (sem curadoria completa), o pipeline deve marcar chunks de FAQ com `source_type: "informal"` e o validador de resposta deve bloquear respostas que citem exclusivamente fontes com `source_type: "informal"` sobre temas com gap documental:

```typescript
// src/services/response-validator.ts
if (
  response.source_document.source_type === "informal" &&
  DOCUMENT_GAPS.includes(detectedTopic)
) {
  return buildFallbackResponse({
    reason: "fonte_nao_confiavel_sobre_gap",
    topic: detectedTopic,
    routing: ROUTING_TABLE[detectedTopic]
  });
}
```

**Camada complementar:** `[HITL]` — revisão obrigatória para respostas que citam FAQ como única fonte

Qualquer resposta cujo `source_document.document_id` começa com `"FAQ-"` deve passar por revisão humana antes de ser entregue ao atendente — até que a curadoria formal do FAQ esteja completa. Este é um ponto HITL temporário, não permanente.

---

## Consolidado final

| # | Classif. QA | Classif. PS | **Classif. canônica** | **Tipo(s) de erro** | **Ajuste proposto** | **Camada** | **Prioridade** |
|---|---|---|---|---|---|---|---|
| 1 | Parcial | Parcial | **Parcialmente correta** | Citação incorreta + Informação incompleta | Estender G-DEVE-02 para incluir `section_id`; instrução de completude de procedimento no prompt | `[CÓDIGO]` + `[PROMPT]` | Média |
| 2 | Parcial | Parcial | **Parcialmente correta** | Informação incompleta (dois níveis) | Instrução de completude para SLA no prompt; injeção forçada do chunk de incidente crítico | `[PROMPT]` + `[PIPELINE]` | Alta |
| 3 | Incorreta | Correta c/ ressalva | **Parcialmente correta** | Encaminhamento incorreto | Encaminhamentos canônicos na routing-table injetados pelo código; extensão da cláusula de carga perigosa no prompt | `[CÓDIGO]` + `[PROMPT]` | Alta |
| 4 | Incorreta | Incorreta | **Incorreta** | Alucinação + Ausência de fonte | Bloqueio pré-geração para GAP-02; HITL para `confidence: high` + `source: null` | `[CÓDIGO]` + `[HITL]` | **Crítica** |
| 5 | Correta | Correta | **Correta** | — (nenhum) | Instrução de citação para negações (baixa prioridade) | `[PROMPT]` | Baixa |
| 6 | Incorreta | Incorreta | **Incorreta** | Fonte não confiável + Gap documental tratado como coberto | Tag `source_type` no metadado de chunk; bloqueio de FAQ informal sobre gaps; HITL para respostas de FAQ não curado | `[PIPELINE]` + `[HITL]` | **Crítica** |

---

## Mapa de ajustes por camada

```
CÓDIGO (determinístico)
├── Estender G-DEVE-02: injetar section_id no template além de document_version (Resp. 1)
├── Injetar chunk de incidente crítico em toda query de SLA (Resp. 2) → novo G-DEVE-07
├── Encaminhamentos canônicos na routing-table, injetados pelo pipeline (Resp. 3)
└── Bloqueio pré-geração para gaps documentais GAP-01 a GAP-05 (Resp. 4 e 6) → G-NAO-04

PIPELINE (dados / indexação)
├── Tag source_type: "informal" | "normativo" em todos os chunks no momento da ingestão (Resp. 6)
├── Bloqueio de chunks com source_type: "informal" em respostas sobre gaps documentais (Resp. 6)
└── Classificação de intenção pré-retrieval para detecção de gaps e tiers inválidos (Resp. 4, 6)

PROMPT (probabilístico)
├── Instrução de completude para procedimentos com múltiplas etapas (Resp. 1)
├── Instrução de completude para respostas de SLA (chamados gerais vs. incidentes críticos) (Resp. 2)
├── Encaminhamento canônico explícito para carga perigosa inelegível: ramal 4500, não supervisor (Resp. 3)
└── Instrução de citação para respostas de negação (Resp. 5) — baixa prioridade

HITL (revisão humana)
├── Reter respostas com confidence: "high" + source_document: null antes de entregar ao atendente (Resp. 4)
└── Reter respostas que citam exclusivamente FAQ como fonte até curadoria formal completa (Resp. 6)
```

---

## Nota sobre a referência ao "Item 41" na avaliação QA (Resposta 4)

O FAQ Item 41 trata da diferença entre SLA de resposta e SLA de resolução — não de carga
danificada. O item relevante para carga danificada é o FAQ Item 38. Recomenda-se verificar
se houve erro de indexação na avaliação. De qualquer forma, o diagnóstico não muda: nem o
Item 38 nem qualquer outro documento da base cobre carga danificada com normativo formal —
o ajuste correto é o bloqueio por gap documental, independentemente do item do FAQ
referenciado.

---

*Documento gerado para registro da fase de governança. Ajustes propostos devem ser
incorporados ao backlog técnico e ao guardrails v1.2 antes do go-live.*
