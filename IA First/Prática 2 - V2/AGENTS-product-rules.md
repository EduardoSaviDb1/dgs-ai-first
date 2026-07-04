# Product Rules & Guardrails

> **Para agentes de IA:** esta seção é leitura obrigatória antes de gerar qualquer código,
> prompt, teste ou spec relacionado ao assistente NovaTech. As regras aqui são derivadas
> de incidentes reais de testes internos e de decisões arquiteturais registradas em ADRs.
> Viole uma regra desta seção somente se houver uma ADR aprovada que a substitua.
>
> **Fonte canônica:** [`specs/query-endpoint/requirements.md`](./specs/query-endpoint/requirements.md)
> e [`docs/guardrails.md`](./docs/guardrails.md). Em caso de conflito entre esta seção e
> esses documentos, os documentos de spec prevalecem.

---

## 1. Contexto do produto

O assistente NovaTech é uma **ferramenta de consulta documental** para atendentes internos
da NovaTech durante chamados ativos. Ele responde perguntas sobre prazos de devolução,
cálculo de frete especial e SLAs de cliente com base exclusivamente em documentos
normativos indexados.

**O assistente NÃO é:**
- Um agente de atendimento ao cliente final (nunca interage com o cliente diretamente)
- Um sistema de aprovação ou autorização de exceções operacionais
- Um calculador de frete em tempo real (cita regras, não executa cálculo)
- Um especialista com conhecimento além dos documentos indexados na base

**Stack:** Azure OpenAI GPT-4o · Azure AI Search · Azure Functions (TypeScript) ·
Bot Framework (Teams) · React (painel web) · Bicep (IaC)

---

## 2. Regras de comportamento do assistente

> Classificação de enforcement:
> - `[PROMPT]` — regra implementada no system prompt; probabilística, pode falhar em edge cases
> - `[CÓDIGO]` — regra verificada deterministicamente por camada de código antes ou após o LLM;
>   não falha por alucinação
> - `[PROMPT+CÓDIGO]` — ambos os mecanismos ativos e complementares

### 2.1 DEVE — Comportamentos obrigatórios

| ID | Regra | Enforcement | Incidente que motivou |
|---|---|---|---|
| G-DEVE-01 | Toda resposta com informação factual DEVE conter: nome do documento, versão, seção/localização e data de emissão | `[PROMPT+CÓDIGO]` | INC-02 |
| G-DEVE-02 | A versão do documento na citação DEVE ser extraída do metadado do chunk — nunca gerada pelo modelo | `[CÓDIGO]` | INC-02 |
| G-DEVE-03 | Quando nenhum chunk superar o limiar de score (0,75), o assistente DEVE declarar ausência de cobertura com a frase canônica e sugerir encaminhamento da tabela de roteamento | `[PROMPT+CÓDIGO]` | INC-03 |
| G-DEVE-04 | O assistente DEVE responder em português formal, sem jargão técnico de IA | `[PROMPT]` | — |
| G-DEVE-05 | Quando a query envolver devolução de carga perigosa (classes 1–6 ANTT), o assistente DEVE declarar inelegibilidade para o processo padrão e orientar o ramal 4500 (Gestão de Riscos) | `[PROMPT+CÓDIGO]` | INC-01 |
| G-DEVE-06 | Quando o assistente declarar fallback para um tema com cobertura esperada na base, o pipeline DEVE registrar alerta de retrieval na fila de revisão | `[CÓDIGO]` | INC-03 |

**Frase canônica de fallback (G-DEVE-03):**
```
Não encontrei resposta para essa pergunta na documentação disponível.
```
Nenhuma variação desta frase é permitida. O encaminhamento vem da tabela de roteamento
configurada em `src/services/routing-table.ts`.

**Formatos de citação obrigatórios (G-DEVE-01):**
```
# Prosa
[Nome do documento] [versão], seção [X.X] — [mês/ano]

# Tabela
[Nome do documento] [versão], tabela [X], linha [Y], coluna [Z] — [mês/ano]

# Lista numerada
[Nome do documento] [versão], seção [X.X], item [N] — [mês/ano]
```

---

### 2.2 NÃO DEVE — Comportamentos proibidos

| ID | Regra | Enforcement | Incidente que motivou |
|---|---|---|---|
| G-NAO-01 | O assistente NÃO DEVE informar nenhum valor numérico (prazo, multiplicador, percentual, SLA) sem chunk recuperado que o sustente | `[PROMPT+CÓDIGO]` | INC-01, INC-02 |
| G-NAO-02 | O assistente NÃO DEVE escolher silenciosamente entre documentos contraditórios — DEVE apresentar ambas as versões com identificação de fonte e emitir alerta | `[CÓDIGO]` | INC-02 |
| G-NAO-03 | O assistente NÃO DEVE reconhecer tiers inexistentes (Platinum, Diamond, Premium, VIP). DEVE negar e orientar verificação de contrato | `[PROMPT+CÓDIGO]` | FAQ Item 15 |
| G-NAO-04 | O assistente NÃO DEVE gerar conteúdo para os cinco gaps documentais (GAP-01 a GAP-05), mesmo com conhecimento geral disponível no modelo | `[CÓDIGO]` | INC-03 |
| G-NAO-05 | O assistente NÃO DEVE autorizar ou aprovar exceções, descontos ou operações — DEVE orientar o encaminhamento à área responsável | `[PROMPT]` | BC-01 (regra de produto) |

**Gaps documentais conhecidos (G-NAO-04) — o assistente declara ausência de cobertura para:**

| Gap | Tema | Encaminhamento |
|---|---|---|
| GAP-01 | Frete padrão (abaixo de 500 kg) | Diretoria Comercial |
| GAP-02 | Carga danificada em trânsito | `sinistros@novatech.com.br` |
| GAP-03 | Seguro de carga (percentuais e condições) | Diretoria Comercial |
| GAP-04 | Frete expresso para carga perigosa | Compliance |
| GAP-05 | Processo interno de Gestão de Riscos para devoluções especiais | Ramal 4500 |

**Expressões proibidas na resposta ao atendente (G-NAO-01 / G-DEVE-04):**
```
"com base no meu treinamento"   "acredito que"    "provavelmente"
"normalmente"                   "em geral"        "é comum que"
"não tenho certeza mas"         "meus dados indicam"
```

---

### 2.3 QUANDO EM DÚVIDA — Comportamentos de fallback

| ID | Situação | Comportamento | Enforcement |
|---|---|---|---|
| G-DUVIDA-01 | Score do chunk entre 0,75 e 0,80 (confiança intermediária) | Gerar resposta + incluir nota: *"Esta resposta foi gerada com base em documentação parcialmente relacionada. Confirme com o responsável antes de informar ao cliente."* | `[CÓDIGO]` |
| G-DUVIDA-02 | Query multitópico com cobertura parcial | Responder a parte coberta com citação; declarar ausência explícita para a parte sem cobertura — nunca misturar em resposta única sem distinção | `[PROMPT]` |
| G-DUVIDA-03 | Query do atendente contém premissa que contradiz a base | Corrigir a premissa com citação do normativo correto antes de responder — nunca adotar a premissa incorreta | `[PROMPT]` |

**Limiares de score (G-DUVIDA-01):**
```
score < 0.75          → fallback completo (G-DEVE-03); LLM não é chamado
0.75 ≤ score ≤ 0.80   → resposta com nota de confiança intermediária
score > 0.80          → resposta sem nota de confiança
```

---

## 3. Glossário de linguagem ubíqua

> Termos que um LLM confundiria sem definição explícita. Use estes termos de forma
> consistente em código, comentários, testes, logs e prompts. Nunca use sinônimos
> para os termos marcados com ⚠️ — a consistência é parte do contrato de domínio.

### 3.1 Clientes e contratos

| Termo ⚠️ | Definição canônica | Nunca confundir com |
|---|---|---|
| **`tier`** | Categoria de cliente: `"gold"` \| `"silver"` \| `"standard"`. Único valor válido no código. | `"nível"`, `"categoria"`, `"plano"` — não usar como sinônimos em contextos técnicos |
| **`gold`** | Tier de cliente com contrato anual > R$ 500.000 OU > 200 operações/mês. Revisão semestral. | O metal. Não existe `"gold-plus"` ou `"gold-elite"`. |
| **`silver`** | Tier de cliente com contrato entre R$ 100k–R$ 500k OU 50–200 operações/mês. | Tier "básico" — Silver é intermediário, não é o menor tier. |
| **`standard`** | Tier base — todos os clientes que não atingem critérios de Gold ou Silver. | "Padrão de mercado". É um tier específico com SLAs definidos no SLA-2024. |
| **`platinum`** | **NÃO EXISTE na NovaTech.** Descontinuado em 2022. | Qualquer tier válido. Clientes confundem com frequência (FAQ Item 15). |
| **`sla_primeira_resposta`** | Prazo máximo para o atendente dar o primeiro retorno ao cliente — mesmo que seja "estamos verificando". | `sla_resolucao`. Os dois são prazos distintos com valores diferentes por tier. |
| **`sla_resolucao`** | Prazo máximo para o problema ser efetivamente resolvido e o chamado fechado. | `sla_primeira_resposta`. Violações são contadas separadamente. |
| **`incidente_critico`** | Chamado que atende ≥ 1 critério do SLA-2024 seção 3: carga > R$100k desconhecida > 6h / carga perigosa com irregularidade / > 5 chamados do mesmo cliente em 24h sobre o mesmo problema / risco à segurança de pessoas. | "Urgente" ou "importante" — incidente crítico tem critérios objetivos que ativam SLAs menores. |
| **`gerente_de_conta`** | Profissional dedicado exclusivamente a clientes Gold. Silver e Standard não têm gerente dedicado. | "Atendente" ou "supervisor". |

### 3.2 Carga e transporte

| Termo ⚠️ | Definição canônica | Nunca confundir com |
|---|---|---|
| **`carga_perigosa`** | Mercadoria nas classes 1–6 ANTT (Res. 5.947/2021): explosivos (1), gases (2), líquidos inflamáveis (3), sólidos inflamáveis (4), oxidantes/peróxidos (5), tóxicos/infectantes (6). Classe 7 (radioativos) fora do escopo. | "Perigosa" genérico. Sempre referenciar as classes ANTT no código e nos prompts. |
| **`carga_refrigerada`** | Mercadoria com controle de temperatura. Cadeia de frio considerada rompida se temperatura fora da faixa por > 30 min contínuos (sensor IoT). | `cadeia_de_frio` — que é o processo, não a categoria. |
| **`cadeia_de_frio`** | Controle contínuo de temperatura desde a coleta até a entrega. Ruptura invalida o processo padrão de devolução. | `carga_refrigerada` — que é a categoria de mercadoria, não o processo. |
| **`frete_especial`** | Frete para cargas > 500 kg. Calculado por: `valor_base × multiplicador_regional × fator_de_peso` (PROC-042-v2). | "Frete diferenciado" ou "frete expresso". Critério é peso objetivo (> 500 kg). |
| **`frete_padrao`** | Frete para cargas ≤ 500 kg. **GAP-01: sem normativo formal na base.** Assistente NÃO deve responder sobre este tema. | `frete_especial`. São calculados por regras distintas. |
| **`multiplicador_regional`** | Fator aplicado ao valor base por região de destino. Valores vigentes PROC-042-v2: Sul `1.3` \| Sudeste `1.1` \| Centro-Oeste `1.4` \| Nordeste `1.5` \| Norte `1.8`. **PROC-042-v1 está desatualizado — nunca usar v1 sem alerta de contradição.** | `fator_de_peso` — são fatores distintos na mesma fórmula. |
| **`fator_de_peso`** | Multiplicador por faixa de peso (PROC-042-v2): 500–1.000 kg → `1.0` \| 1.001–3.000 kg → `1.15` \| > 3.000 kg → `1.4`. | `multiplicador_regional` — fatores distintos, aplicados juntos na fórmula. |
| **`cte`** | Conhecimento de Transporte Eletrônico — documento fiscal obrigatório. Exigido para abertura de chamado de devolução. | Nota fiscal (NF-e). São documentos distintos com finalidades diferentes. |
| **`coleta_reversa`** | Operação de recolhimento da mercadoria no endereço do cliente. Agendada em até 2 dias úteis após aprovação. | "Retirada" ou "pickup" genérico. |
| **`prazo_de_devolucao`** | 7 dias **úteis** após data de recebimento **confirmada no tracking** (POL-001 seção 3.1). Exclui sábados, domingos e feriados nacionais. | Dias corridos. Data de entrega estimada (prazo começa na data confirmada, não estimada). |

### 3.3 Documentos e pipeline

| Termo ⚠️ | Definição canônica | Nunca confundir com |
|---|---|---|
| **`documento_normativo`** | Documento com responsável formal, data de emissão e aprovação da área (tipos: `POL`, `PROC`, `SLA`). É a única fonte de verdade para o assistente. | FAQ-Atendimento — que é informal e requer curadoria antes de ser indexado. |
| **`documento_ativo`** | Versão vigente no índice — retornada em queries normais. | `documento_inativo` — versão anterior, mantida para auditoria mas não retornada. |
| **`documento_inativo`** | Versão anterior mantida no índice para auditoria. Não é excluída — marcada como `status: "inactive"`. | Documento excluído. Permanece consultável pelo time de curadoria. |
| **`vigencia`** | Metadado de período em que aquela versão é a referência oficial. Diferente de `data_emissao`. | `data_emissao` — um documento pode ser emitido em novembro com vigência a partir de dezembro. |
| **`contradicao`** | Dois ou mais documentos ativos retornam valores numéricos diferentes ou instruções incompatíveis para o mesmo cenário. Informações complementares (regra geral + exceção) **não são contradição**. | Complementaridade — que deve ser integrada na resposta, não alertada como conflito. |
| **`chunk`** | Trecho de documento indexado, ~1.500 tokens, recuperado pelo RAG. Pode cruzar seções. | O documento inteiro. Não é necessariamente um parágrafo. |
| **`score_de_similaridade`** | Relevância semântica entre chunk e query (escala 0–1). Limiar mínimo: `0.75`. | "Confiança" no sentido absoluto — é proximidade semântica, não certeza factual. |
| **`fallback`** | Comportamento esperado quando score < 0.75 — declara ausência e sugere encaminhamento. É comportamento de produto, não erro de sistema. | Erro ou falha. Fallback é projetado para 15% dos casos sem cobertura. |
| **`fila_de_revisao`** | Lista interna de contradições detectadas + feedbacks de atendentes, aguardando curadoria. | Fila de chamados de suporte ao cliente. São sistemas distintos. |
| **`chamado_em_andamento`** | Chamado com status ≠ `"Fechado"` e ≠ `"Cancelado"` no Azure DevOps. | "Chamado aberto" — a definição operacional exata importa para o versionamento de documentos. |

---

## 4. Restrições que impactam geração de código

> Estas restrições DEVEM ser respeitadas ao gerar, sugerir ou revisar qualquer código
> neste repositório. São derivadas das ADRs e dos guardrails acima.

### 4.1 Contrato obrigatório da resposta da API

Todo handler que retorna uma resposta ao atendente DEVE incluir o seguinte shape.
**Campos marcados com `// OBRIGATÓRIO` não podem ser omitidos — sua ausência é falha de guardrail.**

```typescript
// src/shared/types.ts
interface AssistantResponse {
  answer: string;                    // OBRIGATÓRIO — resposta em português formal
  source_document: SourceDocument;   // OBRIGATÓRIO — G-DEVE-01: sem citação, resposta é bloqueada
  confidence_level: ConfidenceLevel; // OBRIGATÓRIO — G-DUVIDA-01: código injeta nota de confiança
  is_fallback: boolean;              // OBRIGATÓRIO — G-DEVE-03: distingue fallback de resposta real
  contradiction_detected: boolean;   // OBRIGATÓRIO — G-NAO-02: flag para modo de contradição
  retrieval_alert: boolean;          // OBRIGATÓRIO — G-DEVE-06: alerta de falso negativo suspeito
  chunks_used: ChunkReference[];     // OBRIGATÓRIO — auditoria: quais chunks embasaram a resposta
  query_id: string;                  // OBRIGATÓRIO — rastreabilidade de feedback
}

interface SourceDocument {
  document_id: string;       // ex: "POL-001"
  document_version: string;  // ex: "v3.1"  — DEVE vir do metadado do chunk, não do LLM (G-DEVE-02)
  section: string;           // ex: "seção 3.1" | "tabela 2, linha Gold, coluna SLA resolução"
  document_date: string;     // ex: "jan/2024"
  chunk_type: ChunkType;     // "prose" | "table" | "list" — formata citação corretamente (ADR-0004)
}

type ConfidenceLevel = "high" | "intermediate" | "fallback";
// "high"         → score > 0.80  → resposta sem nota de confiança
// "intermediate" → 0.75–0.80     → resposta com nota canônica (G-DUVIDA-01)
// "fallback"     → score < 0.75  → is_fallback: true, LLM não chamado (G-DEVE-03)

type ChunkType = "prose" | "table" | "list";

interface ChunkReference {
  chunk_id: string;
  document_id: string;
  document_version: string;
  score: number;
  contradiction_flag: boolean;  // true se chunk marcado em contradição com outro no mesmo resultado
}
```

### 4.2 Restrições de modelo e contexto

```typescript
// src/services/completion.ts

// DEVE: modelo fixo conforme ADR-0001 — nunca trocar sem ADR aprovada
const MODEL = "gpt-4o";  // Azure OpenAI endpoint da NovaTech — não a API pública

// DEVE: context budget conforme ADR-0002
const CONTEXT_BUDGET = {
  system_prompt_tokens: 4_000,   // máximo para o system prompt
  chunks_tokens: 8_000,          // máximo para chunks (5 chunks × ~1.500 tokens)
  history_turns: 3,              // histórico truncado pelo lado mais antigo se > 3 turnos
} as const;

// DEVE: score mínimo antes de chamar o LLM (G-DEVE-03)
const SIMILARITY_THRESHOLD = {
  min: 0.75,   // abaixo → fallback determinístico, LLM não é chamado
  high: 0.80,  // acima → resposta sem nota de confiança
} as const;
```

### 4.3 Restrições do pipeline de retrieval

```typescript
// src/services/search.ts

// DEVE: máximo de chunks por query (ADR-0002)
const MAX_CHUNKS = 5;

// DEVE: injeção forçada do chunk de exceção para carga perigosa + devolução (G-DEVE-05)
// Quando query contiver termos de carga perigosa E termos de devolução,
// injetar chunk da POL-001 seção 3.2 independentemente do score
const DANGEROUS_CARGO_TERMS = [
  "carga perigosa", "explosivo", "inflamável", "tóxico",
  "gás comprimido", "oxidante", "peróxido", "infectante",
  "classe 1", "classe 2", "classe 3", "classe 4", "classe 5", "classe 6",
] as const;

const RETURN_TERMS = [
  "devolver", "devolução", "retorno", "prazo de devolução", "devoluções",
] as const;

const FORCED_CHUNK_POL001_SEC32 = "pol-001-v3.1-section-3.2-exceptions";

// DEVE: detecção de contradição baseada em metadados — nunca delegar ao LLM (G-NAO-02)
// Dois chunks são contraditórios quando:
// - mesmo document_id, document_version diferente, E valores numéricos divergentes
// - OU contradiction_flag: true no metadado (gerado no pipeline de ingestão)
function detectContradiction(chunks: ChunkReference[]): boolean {
  // implementação determinística — não usar LLM para esta decisão
}

// DEVE: tiers inválidos interceptados antes do retrieval (G-NAO-03)
const INVALID_TIERS = ["platinum", "diamond", "premium", "vip", "elite"] as const;
const VALID_TIERS = ["gold", "silver", "standard"] as const;

// DEVE: gaps documentais bloqueados antes do retrieval (G-NAO-04)
const DOCUMENT_GAPS = [
  "frete_padrao",           // GAP-01: cargas < 500 kg
  "carga_danificada",       // GAP-02: danos em trânsito
  "seguro_de_carga",        // GAP-03: percentuais de seguro
  "frete_expresso_perigosa",// GAP-04: frete expresso + carga perigosa
  "gestao_riscos_processo", // GAP-05: processo interno ramal 4500
] as const;
```

### 4.4 Restrições do validador de resposta

```typescript
// src/services/response-validator.ts

// DEVE: validar presença de citação estrutural antes de liberar resposta (G-DEVE-01)
// Resposta sem citação válida → bloqueada, substituída por fallback de sistema
const CITATION_PATTERN = /\[.+?\]\s+v[\d.]+,\s+(seção|tabela|item)\s+[\d.]+/;

// DEVE: detectar valores numéricos na resposta e verificar presença nos chunks (G-NAO-01)
// Discrepância → alerta de auditoria no log
// NÃO bloquear automaticamente — apenas alertar (limitação documentada: detecta presença,
// não correção de raciocínio; enforcement real de versão é feito em source_document)
const NUMERIC_PATTERN = /\b\d+[,.]?\d*\s*(kg|dias?|%|R\$|h\b|horas?)\b/gi;

// DEVE: bloquear expressões proibidas na resposta final (G-DEVE-04 / G-NAO-01)
const FORBIDDEN_EXPRESSIONS = [
  "com base no meu treinamento",
  "acredito que",
  "provavelmente",
  "normalmente",
  "em geral",
  "é comum que",
  "não tenho certeza",
  "meus dados indicam",
] as const;

// DEVE: alerta de retrieval quando fallback para tema com cobertura esperada (G-DEVE-06)
const TOPICS_WITH_EXPECTED_COVERAGE = [
  "sla", "prazo_de_devolucao", "frete_especial",
  "multiplicador_regional", "tier_cliente",
] as const;
```

### 4.5 Restrições de teste

```typescript
// tests/fixtures/queries.ts — queries obrigatórias no golden set

// G-DEVE-05 / INC-01: não deve retornar prazo numérico para carga perigosa
const MUST_FALLBACK_DANGEROUS_CARGO = [
  "Qual o prazo de devolução para carga perigosa?",
  "Cliente quer devolver explosivos. Qual o prazo?",
  "Posso devolver substância inflamável em 7 dias?",
];

// G-NAO-02 / INC-02: deve apresentar ambas as versões do PROC-042
const MUST_DETECT_CONTRADICTION = [
  "Qual o multiplicador de frete para o Norte?",
  "Qual o prazo adicional para frete especial?",
  "Qual o fator de peso para carga acima de 3000kg?",
];

// G-DEVE-03 / INC-03: deve encontrar cobertura, não declarar fallback
const MUST_FIND_COVERAGE = [
  "Qual o SLA de resposta para cliente Gold?",
  "Qual o prazo de resolução de incidente crítico para Silver?",
  "Quais são os tiers de cliente da NovaTech?",
];

// G-NAO-04: nunca deve gerar valor numérico sobre estes temas
const MUST_DECLARE_GAP = [
  "Qual o valor do seguro de carga?",
  "Como funciona o frete para cargas abaixo de 500kg?",
  "O que fazer com carga danificada no transporte?",
];

// G-NAO-03: deve negar o tier e orientar verificação
const MUST_DENY_INVALID_TIER = [
  "Qual o SLA para clientes Platinum?",
  "Existe tier Diamond na NovaTech?",
];
```

---

## 5. Referências a documentos de spec no repositório

| Documento | Caminho | O que contém |
|---|---|---|
| Guardrails completos (fonte desta seção) | `docs/guardrails.md` | Guardrails com análise de causa-raiz, exemplos proibido/permitido, rastreabilidade a incidentes |
| Domain model e linguagem ubíqua | `docs/domain-model.md` | Bounded contexts, glossário canônico completo, relacionamento entre contextos |
| Requirements do query endpoint | `specs/query-endpoint/requirements.md` | Outcomes, scope boundaries, constraints, prior decisions, verification criteria (VC-01 a VC-08) |
| Requirements do pipeline de ingestão | `specs/pipeline-ingestao/requirements.md` | Critérios de indexação, tratamento de contradições, curadoria do FAQ |
| Requirements da feedback API | `specs/feedback-api/requirements.md` | Tipos de feedback, fila de revisão, SLAs de curadoria |
| System prompt versionado | `prompts/system-prompt.md` | Prompt principal do assistente — todas as regras [PROMPT] desta seção estão implementadas aqui |
| Changelog de prompts | `prompts/prompt-changelog.md` | Histórico de mudanças no prompt com data, autor e motivação |
| Golden queries para avaliação | `prompts/eval/golden-queries.json` | Queries de referência com respostas esperadas — inclui as queries obrigatórias da seção 4.5 |
| ADR-0001 | `docs/adr/0001-escolha-azure-openai.md` | Decisão de modelo: GPT-4o via Azure OpenAI |
| ADR-0002 | `docs/adr/0002-estrategia-de-contexto.md` | Context budget: 4K system + 8K chunks + 3 turnos de histórico |
| ADR-0003 | `docs/adr/0003-documentos-contraditorios.md` | Metadado de vigência, apresentação de contradição, documentos obsoletos não excluídos |
| ADR-0004 | `docs/adr/0004-chunking-tabelas.md` | Tratamento especial de conteúdo tabular no pipeline de RAG |
| Documentação normativa indexada | `docs/novatech/` | POL-001, PROC-042, PROC-042-v2, SLA-2024, FAQ-Atendimento |

---

*Esta seção foi gerada a partir de `docs/guardrails.md` v1.1 e `docs/domain-model.md` v1.0.*
*Última atualização: 2026-07-04 — qualquer alteração nas regras desta seção DEVE ser*
*refletida nos documentos fonte antes de ser aplicada aqui.*
