# Resultado da Avaliação — Product Specialist (Cenário 2)

> Programa: Trilha de Certificação AI First — DGS / DB1 Global Software  
> Papel avaliado: Product Specialist  
> Data da avaliação: 2026-07-04

## Escopo avaliado

Arquivos considerados na avaliação:
- `Prática 2 - V2/novatech-domain-model.md`
- `Prática 2 - V2/novatech-guardrails.md`
- `Prática 2 - V2/Interface de Resposta - Teams (standalone).html`
- `Prática 2 - V2/AGENTS-product-rules.md`
- `Prática 2 - V2/novatech-assistant/AGENTS.md`
- `Prática 2 - V2/novatech-assistant/specs/query-endpoint/requirements.md`
- `Prática 2 - V2/anexo-c-estrutura-repositorio.md`
- `Prática 2 - V2/exercicio-2-fase-estruturacao.md`

---

## Síntese do parecer

O material entregue demonstra domínio forte de recorte de domínio, linguagem ubíqua, guardrails e tradução de regras de produto para artefatos consumíveis por agentes. O destaque principal está na qualidade do `novatech-domain-model.md`, no documento de guardrails e na seção `AGENTS-product-rules.md`, que é prescritiva, detalhada e suficientemente concreta para influenciar geração de código, testes e prompts.

A principal ressalva não está na qualidade do conteúdo, mas no fechamento do artefato dentro do repositório simulado: a seção de Product Rules foi produzida em arquivo separado, enquanto o `AGENTS.md` principal ainda permanece com `TODO`. Além disso, parte das referências apontadas pela seção ainda não está materializada no scaffold local, o que reduz a aplicabilidade imediata do pacote como constituição já integrada do projeto.

---

## Exercício 2.1 — Recorte de domínio e spec SDD do query endpoint

### Avaliação

**Bounded contexts coerentes:** 3/3  
Os contextos são organizados por domínio de negócio, não por camada técnica, com fronteiras claras entre Atendimento, Gestão Documental, SLAs e Logística/Frete.

**Linguagem ubíqua útil:** 3/3  
O glossário cobre precisamente os termos que um LLM confundiria sem contexto, como `gold`, `platinum`, `carga_perigosa`, `frete_especial`, `vigencia` e `contradicao`.

**Linguagem extraída do Anexo A:** 3/3  
Os termos e exemplos estão claramente ancorados na documentação simulada da NovaTech e nos documentos normativos citados.

**Outcomes orientados a resultado:** 3/3  
Os outcomes priorizam ganho operacional do atendente, tempo de resposta, redução de risco e melhoria contínua da base, em vez de apenas descrever features técnicas.

**Scope boundaries derivados dos bounded contexts:** 3/3  
O escopo do query endpoint deriva diretamente do BC-01, com exclusões explícitas e bem justificadas.

**Verification criteria testáveis:** 3/3  
Os critérios são verificáveis por QA e suficientemente binários para orientar testes.

**Mockup coerente com requirements:** 3/3  
O mockup cobre resposta com fonte, contradição, fallback, confiança intermediária e fluxo de feedback.

**Prior decisions referenciando cenário 1:** 3/3  
As ADRs da fase anterior são incorporadas com impacto concreto sobre o componente.

**Iteração com “Tech Lead”:** 2/3  
Há sinais de iteração e coautoria, mas a evidência explícita do ciclo de revisão e incorporação de ambiguidades não está tão forte quanto o restante.

### Nota do exercício 2.1

**2.9 / 3.0**

---

## Exercício 2.2 — Guardrails formalizados

### Avaliação

**3 categorias presentes:** 3/3  
O documento está claramente estruturado em `DEVE`, `NÃO DEVE` e `QUANDO EM DÚVIDA`.

**Classificação prompt vs código:** 3/3  
A distinção entre enforcement probabilístico e determinístico está madura, com justificativas consistentes.

**Rastreabilidade aos 3 incidentes:** 2/3  
A maior parte dos guardrails está bem rastreada aos incidentes, mas alguns itens ficam como regra preventiva, por analogia, ou ancorados em BC/FAQ em vez de incidente direto.

**Específicos ao domínio NovaTech:** 3/3  
O documento é fortemente contextualizado no domínio, sem cair em guardrails genéricos de chatbot.

### Nota do exercício 2.2

**2.75 / 3.0**

---

## Exercício 2.3 — Seção “Product Rules & Guardrails” do AGENTS.md

### Avaliação

**Machine-readable:** 3/3  
A seção em `AGENTS-product-rules.md` é prescritiva, estruturada, orientada a regras e utilizável por agentes.

**Glossário conectado ao recorte de domínio:** 3/3  
O glossário mantém consistência com bounded contexts e linguagem ubíqua definidos no exercício 2.1.

**Restrições de código concretas:** 3/3  
Há contratos de resposta, thresholds, listas de termos, regras de validação e exemplos de tipos/constantes suficientemente concretos para influenciar Copilot e implementação.

**Consistência com guardrails fornecidos:** 3/3  
Os guardrails do exercício 2.2 foram corretamente refletidos e expandidos.

### Ressalva relevante

Apesar da qualidade do conteúdo, a seção ainda não foi incorporada ao `AGENTS.md` principal do repositório simulado, que segue com placeholder `TODO`. Além disso, algumas referências citadas pela seção apontam para artefatos ainda vazios ou ausentes no scaffold local. Isso não invalida o exercício, mas reduz a completude da integração no repositório.

### Nota do exercício 2.3

**2.8 / 3.0**

---

## Avaliação pelas 5 dimensões (Foundation)

### D1 — Domínio Conceitual

**3/3**  
A entrega demonstra entendimento correto, específico ao projeto e com nuance sobre bounded contexts, linguagem ubíqua, enforcement, RAG, contradições documentais e regras de produto.

### D2 — Uso de Ferramentas

**2/3**  
Há forte evidência de produção de artefatos compatíveis com Claude/Claude Design, mas a trilha de iteração e revisão não está igualmente explícita em todos os entregáveis.

### D3 — Qualidade do Entregável

**3/3**  
Os artefatos são completos, acionáveis e claramente reutilizáveis por outros membros do time.

### D4 — Pensamento Crítico

**3/3**  
A entrega vai além do básico ao distinguir enforcement por prompt e código, tratar limitações das heurísticas e separar contradição de complementaridade.

### D5 — Aplicabilidade ao Projeto

**2/3**  
A entrega está profundamente conectada ao projeto, mas perde um ponto por ainda não estar plenamente integrada ao `AGENTS.md` principal e por depender de referências ainda não materializadas no scaffold local.

---

## Score final

Média das 5 dimensões:

`(3 + 2 + 3 + 3 + 2) / 5 = 2.6`

**Classificação final: Aprovado com distinção**

---

## Conclusão do avaliador

A entrega do Product Specialist está entre as mais fortes da fase de estruturação. O participante demonstrou capacidade real de traduzir domínio de negócio em artefatos consumíveis por humanos e agentes, com boa precisão terminológica, critérios verificáveis e regras prescritivas úteis para implementação.

A única lacuna relevante é de fechamento operacional dentro do repositório simulado: o conteúdo de `AGENTS-product-rules.md` deveria estar incorporado ao `AGENTS.md` principal e alinhado com todos os paths/referências que ele declara como canônicos. Corrigido esse ponto, a entrega ficaria muito próxima de excelência plena.
