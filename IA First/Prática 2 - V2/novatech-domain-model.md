# NovaTech Assistant — Domain Model
**Versão:** 1.0  
**Data:** 2026-07-03  
**Autores:** Product Specialist + Tech Lead (NovaTech / DB1)  
**Status:** Draft — aguardando revisão do time

---

## Parte 1 — Bounded Contexts

> **Princípio de corte:** os contextos são divisões do domínio de negócio, não camadas técnicas. Um mesmo componente de software pode servir a mais de um contexto. A fronteira define *linguagem* e *responsabilidade*, não deployment.

---

### BC-01 · Atendimento ao Cliente

**Propósito:** suportar o atendente durante um chamado ativo — da dúvida à resposta — reduzindo tempo de busca e qualificando escalas.

**O que está dentro:**
- Consulta ao assistente (query) durante ou fora de um chamado
- Exibição de resposta com citação de fonte
- Fluxo de fallback quando o assistente não resolve
- Registro de feedback de resposta (incorreto / desatualizado / incompleto / fonte errada)
- Sugestão de encaminhamento ao responsável quando não há cobertura na base
- Histórico de queries do atendente (para rastreabilidade dentro do chamado)

**O que está fora:**
- Abertura e gestão de chamados no sistema de atendimento (Azure DevOps) — isso é do sistema de chamados, não do assistente
- Atendimento direto ao cliente final — o assistente fala com o atendente, nunca com o cliente
- Autorização de exceções (ex: devolução de carga perigosa) — responsabilidade da Gestão de Riscos
- Negociação comercial (descontos, aditivos contratuais)

**Relaciona-se com:**
- **BC-02 (Gestão Documental):** consome documentos indexados e recebe notificações de atualização de base
- **BC-03 (SLAs e Contratos):** consulta tier do cliente para determinar encaminhamento correto
- **BC-04 (Logística e Frete):** consome regras de cálculo e prazos para responder perguntas sobre frete e devolução

---

### BC-02 · Gestão Documental

**Propósito:** garantir que a base de conhecimento do assistente seja confiável, versionada e auditável — desde a ingestão até a curadoria contínua.

**O que está dentro:**
- Critérios de elegibilidade para indexação de documentos (responsável formal, data, status de vigência)
- Pipeline de ingestão (chunking, embedding, metadados de vigência)
- Controle de versões de documentos (ativo vs. inativo no índice)
- Detecção e registro de contradições entre documentos
- Curadoria do FAQ-Atendimento (classificação: Aprovado / Aprovado com ressalva / Bloqueado)
- Fila de revisão de feedbacks e conflitos detectados
- Notificação ao time de atendimento quando um documento é atualizado
- Auditoria de quais versões de documentos foram usadas em quais respostas

**O que está fora:**
- Criação ou edição dos documentos em si — isso é responsabilidade das áreas donos (Diretoria de Operações, Diretoria Comercial, Compliance)
- Publicação de documentos no SharePoint — o assistente consome, não publica
- Aprovação formal de documentos normativos

**Relaciona-se com:**
- **BC-01 (Atendimento):** fornece os chunks e metadados que embasam cada resposta; recebe feedbacks de qualidade gerados pelos atendentes
- **BC-03 (SLAs):** documentos do tipo SLA-2024 são ingeridos e versionados aqui
- **BC-04 (Logística):** documentos do tipo PROC-042 e POL-001 são ingeridos e versionados aqui

---

### BC-03 · SLAs e Contratos

**Propósito:** representar os compromissos contratuais da NovaTech com seus clientes — tiers, prazos de atendimento, penalidades e incidentes críticos — de forma que o assistente possa responder corretamente sobre direitos e obrigações de cada cliente.

**O que está dentro:**
- Classificação de clientes em tiers (Gold / Silver / Standard)
- Tabela de SLAs por tier e por tipo de chamado (geral vs. crítico)
- Definição de incidente crítico e seus critérios
- Regras de penalidade por descumprimento de SLA
- Diferença entre SLA de primeira resposta e SLA de resolução
- Identificação de gerente de conta dedicado (Gold)

**O que está fora:**
- Medição efetiva do SLA (feita pelo Azure DevOps, não pelo assistente)
- Negociação de SLA diferenciado — responsabilidade do Comercial
- Tiers que não existem na NovaTech (ex: Platinum, Diamond) — o assistente deve negar sua existência explicitamente

**Relaciona-se com:**
- **BC-01 (Atendimento):** o tier do cliente determina o encaminhamento em caso de fallback (ex: Gold tem gerente dedicado)
- **BC-02 (Gestão Documental):** SLA-2024 é o documento normativo versionado neste contexto

---

### BC-04 · Logística e Frete

**Propósito:** cobrir as regras operacionais de transporte — cálculo de frete especial, prazos de entrega, devoluções, classificação de cargas e restrições por categoria — que são o núcleo das dúvidas mais frequentes dos atendentes.

**O que está dentro:**
- Definição e cálculo de frete especial (acima de 500 kg)
- Multiplicadores regionais e fatores de peso
- Prazos adicionais para carga pesada
- Regras de devolução (prazo, elegibilidade, custo, procedimento)
- Classificação de cargas por categoria (perigosa, refrigerada, lacrada)
- Devoluções parciais (múltiplos volumes)
- Condições especiais de frete (aprovação para >5.000 kg, PROC-043 para carga perigosa)
- Referências a documentos de suporte de outros processos (CT-e, Portal do Cliente)

**O que está fora:**
- Tabela de frete padrão (abaixo de 500 kg) — **gap documentado**: não há normativo na base atual
- Política de carga danificada em trânsito — **gap documentado**: existe apenas no FAQ informal
- Seguro de carga — **gap documentado**: sem cobertura normativa na base atual
- Processo de frete expresso para carga perigosa — **gap documentado**: mencionado no FAQ mas sem normativo formal
- Processo interno da Gestão de Riscos para devoluções especiais — responsabilidade da área, não do assistente

**Relaciona-se com:**
- **BC-01 (Atendimento):** as perguntas mais frequentes (35% prazos de entrega, 25% frete, 20% devolução) originam-se deste contexto
- **BC-02 (Gestão Documental):** POL-001 e PROC-042/PROC-042-v2 são os documentos normativos deste contexto; a contradição entre versões do PROC-042 é um problema ativo neste contexto

---

### Mapa de relacionamento entre contextos

```
┌─────────────────────────────────────────────────────────────┐
│                    BC-01 · Atendimento                      │
│         (interface primária — atendente ↔ assistente)       │
└───────────┬──────────────┬──────────────┬───────────────────┘
            │              │              │
     consome │       consulta│     consulta │
     docs    │       tier   │     regras   │
            ▼              ▼              ▼
┌───────────────┐  ┌─────────────────┐  ┌────────────────────┐
│  BC-02        │  │  BC-03          │  │  BC-04             │
│  Gestão       │  │  SLAs e         │  │  Logística e       │
│  Documental   │  │  Contratos      │  │  Frete             │
│               │  │                 │  │                    │
│  (indexação,  │  │  (tiers, SLAs,  │  │  (frete especial,  │
│  versão,      │  │  penalidades,   │  │  devolução,        │
│  curadoria,   │  │  incidentes     │  │  carga perigosa)   │
│  contradições)│  │  críticos)      │  │                    │
└───────────────┘  └─────────────────┘  └────────────────────┘
        ▲                  ▲                     ▲
        └──────────────────┴─────────────────────┘
                  BC-02 versiona documentos
                  de BC-03 e BC-04
```

---

## Parte 2 — Linguagem Ubíqua

> **Propósito:** termos que, sem definição explícita, um LLM (ou um desenvolvedor novo no time) poderia interpretar incorretamente. Esta lista governa a linguagem usada nos prompts do sistema, nos metadados de documentos, nas interfaces de usuário e no código. Termos fora desta lista usam o significado comum do português.

---

### 2.1 Termos de domínio — Clientes e Contratos

| Termo | Definição canônica | Confusão que previne |
|---|---|---|
| **Gold** | Tier de cliente NovaTech com contrato anual acima de R$ 500.000 OU mais de 200 operações/mês. Revisão semestral. | Não é o metal. Não é sinônimo de "premium genérico". Não existe variação "Gold Plus" ou "Gold Elite". |
| **Silver** | Tier de cliente NovaTech com contrato anual entre R$ 100.000 e R$ 500.000 OU entre 50 e 200 operações/mês. | Não é o metal. Intermediário entre Gold e Standard — não é "básico". |
| **Standard** | Tier base de cliente NovaTech — todos os clientes que não atingem os critérios de Gold ou Silver. | Não significa "padrão do mercado". É um tier específico com SLAs definidos. |
| **Platinum** | **Não existe na NovaTech.** Tier descontinuado em 2022. | Clientes às vezes referenciam este tier. O assistente deve negar explicitamente e orientar sobre os tiers vigentes. |
| **Tier** | Categoria de cliente (Gold / Silver / Standard). Nunca usar "nível", "categoria" ou "plano" como sinônimo em contextos técnicos. | Evita ambiguidade com "plano de serviço" ou "nível de contrato". |
| **SLA de primeira resposta** | Prazo máximo para o atendente dar o primeiro retorno ao cliente após abertura do chamado (mesmo que seja "estamos verificando"). | Diferente de SLA de resolução. Um não implica o outro. |
| **SLA de resolução** | Prazo máximo para o problema do cliente ser efetivamente resolvido e o chamado fechado. | Diferente de SLA de primeira resposta. |
| **Incidente crítico** | Chamado que atende ao menos um dos critérios da seção 3 do SLA-2024: carga >R$100k desconhecida por >6h, carga perigosa com irregularidade, >5 chamados do mesmo cliente em 24h sobre o mesmo problema, ou risco à segurança de pessoas. | Não é sinônimo de "urgente" ou "importante". Tem critérios objetivos que ativam SLAs menores. |
| **Gerente de conta** | Profissional da NovaTech dedicado exclusivamente a clientes Gold. Clientes Silver e Standard não têm gerente de conta dedicado. | Não confundir com "atendente" ou "supervisor". |

---

### 2.2 Termos de domínio — Carga e Transporte

| Termo | Definição canônica | Confusão que previne |
|---|---|---|
| **Carga perigosa** | Mercadoria classificada nas classes 1 a 6 da ANTT (Resolução ANTT nº 5.947/2021): explosivos (1), gases (2), líquidos inflamáveis (3), sólidos inflamáveis (4), oxidantes e peróxidos (5), substâncias tóxicas e infectantes (6). | "Perigosa" sem qualificação é ambígua. O assistente deve sempre referenciar as classes ANTT quando aplicável. Classe 7 (radioativos) não está no escopo da documentação atual. |
| **Carga refrigerada** | Mercadoria que requer controle de temperatura durante o transporte. Considera-se com cadeia de frio rompida quando a temperatura ficou fora da faixa especificada na nota fiscal por mais de 30 minutos contínuos, conforme sensor IoT. | O critério de "30 minutos contínuos" é preciso — não é qualquer desvio de temperatura. |
| **Frete especial** | Frete aplicável a cargas com peso acima de 500 kg, calculado com multiplicadores regionais e fatores de peso conforme PROC-042-v2 (versão vigente). | Não é sinônimo de "frete diferenciado" ou "frete expresso". Tem critério objetivo de peso (>500 kg). |
| **Frete padrão** | Frete aplicável a cargas com peso até 500 kg. **Gap documentado:** não há normativo formal na base atual — o assistente não deve responder sobre frete padrão sem confirmar com o Comercial. | Evita confusão com "frete normal" ou "frete regular". |
| **Multiplicador regional** | Fator numérico aplicado ao valor base do frete conforme a região de destino da carga. Valores vigentes (PROC-042-v2): Sul 1.3, Sudeste 1.1, Centro-Oeste 1.4, Nordeste 1.5, Norte 1.8. | Os multiplicadores da v1 (Sul 1.2, Sudeste 1.0, CO 1.3, NE 1.4, Norte 1.6) estão desatualizados. O assistente deve usar v2 e alertar sobre a contradição existente. |
| **Fator de peso** | Multiplicador adicional aplicado conforme a faixa de peso da carga (PROC-042-v2): 1.0 para 500–1.000 kg; 1.15 para 1.001–3.000 kg; 1.4 para acima de 3.000 kg. | Diferente do multiplicador regional — são dois fatores distintos na mesma fórmula. |
| **CT-e** | Conhecimento de Transporte Eletrônico — documento fiscal obrigatório para todas as operações de transporte. Número do CT-e é exigido para abertura de chamado de devolução. | Não é "nota fiscal" (NF-e). São documentos distintos com finalidades diferentes. |
| **Cadeia de frio** | Controle contínuo de temperatura de cargas refrigeradas desde a coleta até a entrega. Ruptura da cadeia de frio invalida o processo padrão de devolução. | Não é sinônimo de "carga refrigerada" — é o processo/controle, não a categoria de carga. |
| **Coleta reversa** | Operação de recolhimento da mercadoria no endereço do cliente para devolução ao centro de distribuição da NovaTech. Agendada em até 2 dias úteis após aprovação de devolução. | Não é "retirada" ou "pickup" genérico. É o processo específico de devolução física. |
| **Prazo de devolução** | 7 dias úteis após a data de recebimento confirmada no sistema de tracking (POL-001, seção 3.1). Exclui sábados, domingos e feriados nacionais. | "Dias úteis" ≠ "dias corridos". O prazo não começa na data de entrega estimada — começa na data confirmada no tracking. |

---

### 2.3 Termos de domínio — Documentos e Processos

| Termo | Definição canônica | Confusão que previne |
|---|---|---|
| **Documento normativo** | Documento com responsável formal identificado, data de emissão, e que passou por aprovação da área responsável (POL, PROC, SLA). É a fonte de verdade para o assistente. | Diferente do FAQ-Atendimento, que é informal e requer curadoria antes de ser indexado. |
| **Documento ativo** | Documento com versão vigente no índice do assistente — retornado em queries normais dos atendentes. | Diferente de documento inativo (versão anterior, mantida para histórico mas não retornada em queries). |
| **Documento inativo** | Versão anterior de um documento, mantida no índice para auditoria mas não retornada em queries normais. Não é excluído — é marcado como inativo. | "Inativo" ≠ "excluído". A versão ainda existe e pode ser consultada pelo time de curadoria. |
| **Contradição** | Situação em que dois ou mais documentos ativos retornam valores numéricos diferentes (prazos, multiplicadores, percentuais) ou instruções incompatíveis para o mesmo cenário. Documentos que cobrem o mesmo tema com informações complementares não são contraditórios. | Distingue contradição real (requer alerta) de complementaridade (pode ser integrada na resposta). |
| **Curadoria** | Processo de revisão e classificação de itens do FAQ-Atendimento por revisores de Compliance e Operações, resultando em: Aprovado / Aprovado com ressalva / Bloqueado. | Não é revisão genérica. É um processo formal com categorias e responsáveis definidos. |
| **Fila de revisão** | Lista de contradições detectadas automaticamente e feedbacks registrados pelos atendentes, aguardando avaliação pelo responsável pela curadoria da base. | Não é uma fila de chamados de suporte. É interna ao processo de gestão documental. |
| **Vigência** | Metadado de cada documento que indica o período em que aquela versão é a referência oficial. Usado pelo pipeline para priorizar versões mais recentes em caso de contradição. | Diferente de "data de emissão" — um documento pode ser emitido em novembro e ter vigência a partir de dezembro (como o PROC-042-v2). |
| **Chamado em andamento** | Chamado com status diferente de "Fechado" ou "Cancelado" no Azure DevOps. Chamados em andamento mantêm vínculo com a versão do documento vigente no momento de abertura. | Definição operacional necessária para o comportamento de versionamento do REQ-04. |

---

### 2.4 Termos do assistente (interface e comportamento)

| Termo | Definição canônica | Confusão que previne |
|---|---|---|
| **Query** | Pergunta em linguagem natural submetida pelo atendente ao assistente. Unidade básica de interação. | Não confundir com "busca textual" ou "pesquisa por palavra-chave" — o assistente interpreta intenção, não filtra por termo exato. |
| **Chunk** | Trecho de documento indexado, de aproximadamente 1.500 tokens, recuperado pelo pipeline de RAG em resposta a uma query. | Não é o documento inteiro. Não é um parágrafo necessariamente — pode cruzar seções dependendo do algoritmo de chunking. |
| **Score de similaridade** | Medida numérica (escala 0–1) de relevância entre um chunk recuperado e a query do atendente. Limiar mínimo para geração de resposta: 0,75 (a ser calibrado). | Não é "confiança" no sentido absoluto — é proximidade semântica entre o texto da query e o texto do chunk. |
| **Fallback** | Comportamento do assistente quando não consegue gerar resposta com confiança suficiente — declara a limitação e sugere encaminhamento. | Não é erro do sistema. É um comportamento esperado e projetado para os 15% de casos sem cobertura na base. |
| **Encaminhamento** | Sugestão do assistente sobre qual área ou profissional o atendente deve consultar quando o assistente não resolve. Derivado da tabela de roteamento configurada pelo Product Specialist. | Não é redirecionamento automático de chamado. É uma sugestão textual ao atendente. |
| **Feedback** | Registro pelo atendente de que uma resposta do assistente estava incorreta, desatualizada, incompleta ou com fonte errada. Alimenta a fila de revisão. | Não é avaliação de satisfação (like/dislike genérico). Tem categorias específicas e gera ação de curadoria. |

---

## Parte 3 — requirements.md · Query Endpoint

**Arquivo:** `docs/requirements/query-endpoint.md`  
**Componente:** API do Assistente — Query Handler  
**Contexto coberto:** BC-01 (Atendimento ao Cliente), com integração a BC-02 (Gestão Documental)  
**ADRs referenciadas:** ADR-0001, ADR-0002, ADR-0003, ADR-0004

---

# Query Endpoint — Requirements

**Status:** Draft  
**Versão:** 1.0  
**Data:** 2026-07-03

---

## 1. Outcomes

> O que um atendente deve conseguir fazer, e que hoje não consegue de forma satisfatória.

**OUT-01 — Obter resposta confiável em menos de 2 minutos**  
O atendente, durante um chamado ativo, submete uma pergunta sobre prazos de devolução, cálculo de frete ou SLA de cliente e recebe uma resposta com fonte citada antes de precisar abrir qualquer outra ferramenta. Hoje, o processo médio de busca leva 12 minutos e envolve 4 fontes distintas.

**OUT-02 — Saber quando a resposta não é confiável, antes de informar ao cliente**  
Quando o assistente não tem cobertura suficiente na base ou detecta documentos contraditórios, o atendente recebe esse diagnóstico explicitamente — com sugestão de encaminhamento — em vez de receber uma resposta incorreta ou inventada.

**OUT-03 — Escalar com contexto, não com vazio**  
Quando o caso precisar de escala para supervisor ou área especializada, o atendente tem um registro concreto do que o assistente encontrou (ou não encontrou) para embasar a escala — eliminando o "não sabia e escalei" sem mais informações.

**OUT-04 — Contribuir para a melhoria da base sem processo adicional**  
Quando o atendente identifica que uma resposta estava errada ou desatualizada, ele consegue registrar esse feedback no mesmo fluxo do chamado — sem abrir um sistema separado — e o problema chega ao responsável pela curadoria.

---

## 2. Scope Boundaries

### 2.1 O que este componente cobre

Este componente cobre o **BC-01 · Atendimento ao Cliente** — a interação entre o atendente e o assistente durante um chamado ou consulta avulsa.

Especificamente:

- Receber a query do atendente (via Teams Bot ou painel web)
- Executar o pipeline de retrieval (Azure AI Search) para recuperar chunks relevantes
- Avaliar o score de similaridade dos chunks recuperados
- Gerar a resposta via Azure OpenAI (GPT-4o) dentro do context budget definido (ADR-0002)
- Formatar a resposta com citação de fonte, versão e data do documento
- Detectar e sinalizar contradições entre chunks recuperados de documentos diferentes
- Responder com fallback quando o limiar de confiança não for atingido
- Sugerir encaminhamento com base na tabela de roteamento configurada
- Registrar o feedback do atendente e encaminhar à fila de revisão (BC-02)
- Persistir o log de query, chunks usados e resposta para auditoria

### 2.2 O que este componente NÃO cobre

- **Pipeline de ingestão** — indexação de documentos, chunking, embedding: responsabilidade do componente de ingestão (BC-02). Este componente consome o índice, não o constrói.
- **Curadoria de documentos** — classificação de FAQ, resolução de contradições, aprovação de novos normativos: responsabilidade do fluxo de gestão documental (BC-02).
- **Gestão de chamados** — abertura, status, fechamento de chamados no Azure DevOps: sistema externo, fora do escopo do assistente.
- **Atendimento direto ao cliente final** — o endpoint recebe queries de atendentes autenticados, nunca de clientes externos.
- **Cálculo de frete em tempo real** — o assistente cita as regras de cálculo da base documental; não executa o cálculo como serviço.
- **Autorização de exceções operacionais** — ex: aprovação de devolução de carga perigosa pelo Gestão de Riscos. O assistente orienta o encaminhamento, nunca autoriza.
- **Frete padrão (abaixo de 500 kg)** — gap documental: não há normativo na base. O assistente deve declarar ausência de cobertura.
- **Seguro de carga** — gap documental: apenas FAQ informal disponível. O assistente deve declarar ausência de cobertura normativa.

---

## 3. Constraints

**C-01 — Context budget fixo (ADR-0002)**  
Cada requisição ao GPT-4o deve respeitar o orçamento de tokens: ~4.000 tokens para system prompt, ~8.000 tokens para chunks (até 5 chunks de ~1.500 tokens), pergunta do atendente, e histórico limitado a 3 turnos anteriores. O componente é responsável por montar o payload dentro desse orçamento antes de chamar o modelo.

**C-02 — Modelo fixo: GPT-4o via Azure OpenAI (ADR-0001)**  
Nenhuma outra instância de LLM pode ser usada no query handler sem uma ADR aprovada que substitua a ADR-0001. A integração usa o endpoint Azure OpenAI da NovaTech — não a API pública da OpenAI.

**C-03 — Documentos contraditórios nunca resolvidos silenciosamente (ADR-0003)**  
Quando o retriever retornar chunks de documentos com metadado de vigência conflitante sobre o mesmo tema, o componente não pode escolher uma versão automaticamente. Deve apresentar ambas e emitir alerta.

**C-04 — Chunking de tabelas requer tratamento especial (ADR-0004)**  
O protótipo identificou degradação de qualidade no retrieval de conteúdo tabular. O componente deve esperar chunks com metadado de tipo (prosa / tabela / lista) e formatar a citação de acordo — nunca citar linha de tabela como se fosse prosa.

**C-05 — Resposta sempre em português formal**  
O sistema prompt deve instruir o modelo a responder exclusivamente em português formal. Nenhum jargão técnico de IA ("com base no meu treinamento", "não tenho certeza") é aceitável na resposta ao atendente.

**C-06 — Ausência de especulação**  
O modelo não pode usar conhecimento geral para preencher lacunas da base. Se a informação não estiver nos chunks recuperados com score acima do limiar, a resposta é fallback — não inferência.

**C-07 — Latência alvo**  
O endpoint deve retornar resposta em até 8 segundos em condições normais de carga (p95). Esse valor é orientativo para a fase de desenvolvimento — será ajustado após testes de carga.

---

## 4. Prior Decisions

| ADR | Decisão | Impacto neste componente |
|---|---|---|
| **ADR-0001** | Modelo LLM: Azure OpenAI GPT-4o (janela de 128K tokens, integração Microsoft) | O cliente Azure OpenAI é a única integração permitida para geração de resposta. Qualquer mudança de modelo requer nova ADR. |
| **ADR-0002** | Estratégia de contexto: 4K system prompt + 8K chunks (5 × ~1.500 tokens) + pergunta + 3 turnos de histórico | O componente monta o payload dentro desse orçamento. Histórico > 3 turnos deve ser truncado pelo lado mais antigo. |
| **ADR-0003** | Documentos contraditórios: metadado de vigência + prompt instrui priorizar mais recente + documentos obsoletos marcados, não excluídos | Quando dois chunks do mesmo tema tiverem metadados de vigência conflitantes, o componente emite alerta e apresenta ambas as versões — nunca escolhe silenciosamente. |
| **ADR-0004** | Chunking: Azure AI Search para produção; problema de chunking em tabelas identificado no protótipo (ChromaDB + sentence-transformers) | O componente deve usar o metadado de tipo de chunk (prosa / tabela / lista) para formatar a citação corretamente. Não assumir que todo chunk é prosa. |

---

## 5. Verification Criteria

> Critérios testáveis pelo QA. Cada critério referencia o behavior que verifica e o mecanismo de teste.

---

### VC-01 · Resposta com citação completa (cobre OUT-01, REQ-05)

**Dado** que o atendente submete uma query sobre prazo de devolução para cliente Gold  
**Quando** o assistente retorna uma resposta  
**Então:**
- A resposta contém o nome do documento (ex: "POL-001")
- A resposta contém a versão do documento (ex: "v3.1")
- A resposta contém a seção de origem (ex: "seção 3.1")
- A resposta contém a data de emissão ou última atualização (ex: "jan/2024")
- Para conteúdo tabular: a resposta indica linha e coluna relevantes e reproduz o valor da célula

**Falha se:** qualquer um dos quatro elementos estiver ausente.  
**Como testar:** suite de testes de integração com golden queries e validação estrutural da resposta (regex ou parser de formato de citação).

---

### VC-02 · Detecção e apresentação de contradição (cobre OUT-02, REQ-02, ADR-0003)

**Dado** que o atendente pergunta sobre o multiplicador de frete para a região Norte  
**Quando** o retriever retorna chunks de PROC-042 v1 e PROC-042-v2  
**Então:**
- A resposta apresenta os dois valores: 1.6 (PROC-042 v1, mar/2023) e 1.8 (PROC-042-v2, nov/2023)
- A resposta contém alerta explícito de divergência com identificação de ambas as fontes e suas datas
- A resposta sugere o responsável pela resolução com base nos metadados do documento
- O conflito é registrado automaticamente na fila de revisão (verificável via log ou API de curadoria)
- A resposta NÃO apresenta apenas um dos valores sem mencionar o outro

**Falha se:** a resposta retornar apenas um multiplicador sem alerta, ou retornar os dois sem identificar as fontes.  
**Como testar:** teste de integração com query "multiplicador Norte" + verificação da estrutura da resposta + verificação do registro na fila de curadoria.

---

### VC-03 · Fallback quando não há cobertura (cobre OUT-02, OUT-03, REQ-03, REQ-06)

**Dado** que o atendente pergunta sobre seguro de carga (tema sem normativo na base)  
**Quando** nenhum chunk recuperado supera o limiar de score de similaridade  
**Então:**
- A resposta declara explicitamente: "Não encontrei resposta para essa pergunta na documentação disponível."
- A resposta NÃO reproduz os percentuais do FAQ Item 22 (0,3% e 0,8%) como se fossem normativos
- A resposta sugere encaminhamento ao Comercial (conforme tabela de roteamento)
- A resposta NÃO usa linguagem especulativa ("provavelmente", "normalmente", "em geral", "acredito que")

**Falha se:** o assistente gerar qualquer valor numérico ou regra sobre seguro de carga.  
**Como testar:** golden query "qual o valor do seguro de carga?" + análise de resposta por presença/ausência de valores numéricos e linguagem especulativa.

---

### VC-04 · Fallback quando score abaixo do limiar com confiança intermediária (cobre REQ-06)

**Dado** que o atendente pergunta sobre devolução de pallets vazios (tema inexistente na base)  
**Quando** o chunk mais relevante retorna score entre 0,75 e 0,80  
**Então:**
- A resposta inclui nota de confiança intermediária: "Esta resposta foi gerada com base em documentação parcialmente relacionada à sua pergunta. Confirme com o responsável antes de informar ao cliente."

**Dado** que o score do chunk mais relevante é inferior a 0,75  
**Então:**
- O componente segue o comportamento de VC-03 (fallback completo, sem resposta de conteúdo)

**Como testar:** injeção de queries sobre temas inexistentes com mock do retriever retornando scores controlados.

---

### VC-05 · Ausência de especulação com conhecimento geral (cobre C-06, REQ-03)

**Dado** que o atendente pergunta sobre um tema sem cobertura na base  
**Quando** o assistente gera resposta  
**Então:**
- A resposta não contém informação que não esteja presente nos chunks recuperados
- A resposta não usa: "provavelmente", "normalmente", "em geral", "geralmente", "é comum que", "acredito que", "com base no meu treinamento"
- A resposta não cita fontes que não estejam no índice do assistente

**Como testar:** adversarial queries sobre temas conhecidos pelo LLM mas ausentes na base (ex: legislação trabalhista, INCOTERMS) + análise de resposta.

---

### VC-06 · Registro de feedback e encaminhamento à fila de revisão (cobre OUT-04, REQ-02)

**Dado** que o atendente clica em "Reportar problema" após receber uma resposta  
**Quando** o atendente seleciona o tipo de problema e submete o formulário  
**Então:**
- O registro é criado na fila de revisão com: ID da query original, resposta que gerou o feedback, chunks usados, tipo de problema, e identificador do atendente
- O feedback fica disponível para o responsável pela curadoria em até 5 minutos após o registro
- O atendente recebe confirmação visual de que o feedback foi registrado

**Como testar:** fluxo de teste de ponta a ponta via interface do Teams ou painel web + verificação do registro via API ou dashboard de curadoria.

---

### VC-07 · Context budget respeitado (cobre C-01, ADR-0002)

**Dado** que o atendente submete uma query em uma conversa com 3 turnos anteriores  
**Quando** o componente monta o payload para o Azure OpenAI  
**Então:**
- O total de tokens do payload não excede: 4.000 (system prompt) + 8.000 (chunks) + tokens da query + tokens do histórico de 3 turnos
- Se o histórico tiver mais de 3 turnos, os mais antigos são truncados

**Como testar:** teste unitário com mock do tokenizador e histórico sintético de 5+ turnos + verificação do payload montado antes do envio ao modelo.

---

### VC-08 · Tier inexistente rejeitado com orientação correta (cobre BC-03, Linguagem Ubíqua)

**Dado** que o atendente pergunta sobre o SLA de um cliente "Platinum"  
**Quando** o assistente processa a query  
**Então:**
- A resposta declara que o tier Platinum não existe na NovaTech
- A resposta informa os tiers vigentes (Gold, Silver, Standard)
- A resposta orienta a verificar o número do contrato para identificar o tier correto

**Como testar:** golden query "qual o SLA do cliente Platinum?" + verificação de ausência de qualquer SLA atribuído ao tier Platinum na resposta.

---

## 6. Gaps Documentados (para rastreabilidade)

Os itens abaixo representam perguntas que o assistente **não pode responder** com base na documentação atual. O QA deve verificar que o assistente declara ausência de cobertura para esses temas — não inventa resposta.

| Gap | Tema | Documento existente | Ação recomendada |
|---|---|---|---|
| GAP-01 | Frete padrão (abaixo de 500 kg) | Nenhum normativo | Solicitar ao Comercial a criação de PROC para frete padrão |
| GAP-02 | Política de carga danificada em trânsito | Apenas FAQ Item 38 (informal) | Solicitar ao Jurídico/Operações a criação de POL formal |
| GAP-03 | Seguro de carga (percentuais e condições) | Apenas FAQ Item 22 (informal) | Solicitar ao Comercial a criação de documento normativo |
| GAP-04 | Frete expresso para carga perigosa | Apenas FAQ Item 32 (informal, sem normativo que defina o processo) | Solicitar ao Compliance a formalização do processo |
| GAP-05 | Processo interno da Gestão de Riscos para devoluções especiais | POL-001 menciona o ramal 4500 mas não descreve o processo | Solicitar ao Gestão de Riscos a criação de PROC |

---

*Fim do documento.*
