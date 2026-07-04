# Exercício 1.3 — Especificação de requisitos de RAG do ponto de vista do produto
**Papel:** Product Specialist  
**Projeto:** Assistente de IA NovaTech (DB1)  
**Fase:** Intent + Discovery

---

## Contexto da especificação

Esta especificação define os requisitos que o pipeline de RAG deve atender para que o assistente entregue valor aos atendentes da NovaTech. Os requisitos são escritos do ponto de vista do produto, sendo precisos o suficiente para serem testados pelo QA, mas sem entrar em decisões de implementação técnica conforme enunciado da atividade.

**Insumos utilizados:**
- Documentação oficial da NovaTech (POL-001, PROC-042, PROC-042-v2, SLA-2024, FAQ-Atendimento)
- Dados do discovery: 45 atendentes, 320 chamados/dia, 60% com consulta a documentação, tempo médio de busca de 12 minutos, 15% de escala para supervisor
- Contradições e gaps mapeados no Exercício 1.1

---

## Versão inicial da especificação (v1)

### REQ-01 — Fontes de dados indexadas

**O que deve ser indexado:**
- POL-001: Política de Devolução de Mercadorias (versão vigente)
- PROC-042-v2: Procedimento de Cálculo de Frete Especial (versão revisada)
- SLA-2024: Tabela de SLA por Tipo de Cliente
- FAQ-Atendimento: somente após curadoria e validação formal por Compliance e Operações

**O que não deve ser indexado:**
- PROC-042 v1: versão anterior com multiplicadores desatualizados — deve ser marcada como obsoleta e excluída do índice
- Documentos sem responsável formal identificado
- Versões anteriores de qualquer documento normativo

**Critério de elegibilidade para indexação:**
Um documento só entra na base se atender a todos os seguintes critérios: 
(1) possui responsável formal identificado; 
(2) possui data de emissão ou última atualização; 
(3) não está explicitamente marcado como obsoleto; 
(4) foi aprovado pelo responsável da área.

---

### REQ-02 — Tratamento de documentos contraditórios

Quando o pipeline identificar dois ou mais documentos que tratam do mesmo tema com informações divergentes, o assistente deve:

1. Não escolher sozinho, e sem que o usuário saiba, uma versão.
2. Apresentar as duas versões ao atendente com identificação clara de cada fonte.
3. Emitir alerta explícito: "Encontrei informações divergentes entre dois documentos sobre este tema. Recomendo confirmar com o responsável antes de informar ao cliente."
4. Indicar, quando disponível, qual documento tem data de emissão mais recente.

**Critério de teste:** dado um chamado sobre multiplicador de frete para a região Norte, o assistente não pode retornar apenas um valor — deve apresentar os dois (1.6 da v1 e 1.8 da v2) com identificação das fontes e alerta de divergência.

---

### REQ-03 — Comportamento quando não há resposta na base

Quando a pergunta do atendente não tiver cobertura na base indexada, o assistente deve:

1. Declarar explicitamente que não encontrou a informação: "Não encontrei resposta para essa pergunta na documentação disponível."
2. Não especular, não inferir e não usar conhecimento geral para preencher a lacuna.
3. Sugerir o próximo passo: indicar a área responsável ou sugerir escalar para o supervisor.
4. Nunca responder com linguagem que sugira incerteza disfarçada de resposta (ex: "provavelmente", "normalmente", "em geral").

**Critério de teste:** perguntado sobre a política de seguro de carga (tema sem cobertura normativa na base), o assistente deve declarar que não encontrou a informação — não deve reproduzir os percentuais do FAQ Item 22 como se fossem normativos.

---

### REQ-04 — Atualização da base

Quando novos documentos forem publicados ou documentos existentes forem revisados:

1. O documento atualizado deve estar disponível no assistente em até 24 horas após a publicação no SharePoint.
2. A versão anterior deve ser automaticamente marcada como inativa no índice — não excluída, mas não recuperável em queries normais.
3. Chamados em andamento no momento da atualização continuam usando a versão vigente no momento de abertura do chamado.

**Critério de teste:** publicado um novo SLA-2024-v2 no SharePoint, após 24 horas o assistente deve retornar os novos valores de SLA — e não os anteriores.

---

### REQ-05 — Rastreabilidade das respostas

Toda resposta do assistente deve incluir:

1. **Citação de fonte obrigatória:** nome do documento e seção de origem (ex: "POL-001, seção 3.1").
2. **Trecho relevante:** o assistente deve exibir o trecho exato do documento que fundamenta a resposta, entre aspas.
3. **Data de vigência do documento:** exibida junto à citação para que o atendente avalie se a informação está atualizada.

**Critério de teste:** toda resposta gerada pelo assistente deve conter ao menos uma citação no formato especificado. Respostas sem citação são consideradas não conformes.

---

## Feedback do Claude sobre a v1

*Nesta etapa, a especificação v1 foi enviada ao Claude com o seguinte prompt:*

```
Abaixo está a versão inicial de uma especificação de requisitos de produto para um pipeline de RAG. Você é um revisor técnico experiente em sistemas de IA generativa e produto. Identifique gaps, ambiguidades e requisitos que um QA não conseguiria testar com clareza. Seja específico: aponte o problema e por que é um problema.

[especificação v1 colada acima]
```

*Output do Claude:*

**Gap 1 — REQ-01: critério de curadoria do FAQ é necessário mas não suficiente**

A spec diz que o FAQ só entra após "curadoria e validação formal", mas não define o que significa "validado": quem valida, com qual critério, e o que acontece com itens aprovados parcialmente (ex: um item correto em conteúdo mas que cita uma versão desatualizada de um PROC). Sem isso, dois revisores chegariam a decisões diferentes sobre o mesmo item.

**Gap 2 — REQ-02: o critério de "documentos contraditórios" não está definido**

O que caracteriza uma contradição para o pipeline? Dois documentos com o mesmo número de PROC são contradição óbvia, mas e dois documentos que cobrem o mesmo tema com informações complementares mas não idênticas? O pipeline precisa de um critério operacional para detectar contradição — a spec não define isso.

**Gap 3 — REQ-02: falta definir o que o atendente faz com a informação contraditória**

O assistente apresenta as duas versões — e depois? A spec não define se o atendente tem um caminho para registrar qual versão usou, ou se há algum mecanismo para que esse conflito seja tratado. Sem isso, a divergência é apresentada mas não resolvida, e o atendente fica sem ação clara.

**Gap 4 — REQ-03: "sugerir o próximo passo" é ambíguo**

"Indicar a área responsável" pressupõe que o sistema sabe qual área é responsável por cada tema. Isso precisa ser configurado — não emerge automaticamente. A spec deveria definir se essa informação vem de metadados dos documentos, de uma tabela de roteamento, ou de outra fonte.

**Gap 5 — REQ-04: "em andamento no momento da atualização" não é testável**

O que define que um chamado está "em andamento"? Status aberto no sistema? Última mensagem enviada há menos de X horas? Sem esse critério, o QA não consegue verificar se o comportamento está correto.

**Gap 6 — REQ-04: ausência de requisito para notificação de atualização**

Se um documento é atualizado e o assistente passa a responder com novos valores, os atendentes precisam saber. Sem um mecanismo de notificação, um atendente pode receber uma resposta diferente da que recebeu ontem para a mesma pergunta — sem entender por quê.

**Gap 7 — REQ-05: "trecho exato entre aspas" pode conflitar com documentos estruturados**

Tabelas e listas não têm "trechos" naturais para citar entre aspas. A spec precisa definir o comportamento de rastreabilidade para conteúdo estruturado (ex: "para tabelas, citar a linha e coluna relevantes").

**Gap 8 — Ausência de requisito sobre confiança mínima**

Nenhum requisito define o que acontece quando o retriever retorna chunks com baixa similaridade. O assistente deve responder mesmo com confiança baixa? Deve declarar incerteza? Deve tratar como "não encontrei"? Isso impacta diretamente a qualidade percebida pelo atendente.

---

## Versão refinada da especificação (v2)

*As seções abaixo incorporam as correções derivadas do feedback. Cada requisito indica o gap que motivou a mudança.*

---

### REQ-01 — Fontes de dados indexadas *(revisado: Gap 1)*

**O que deve ser indexado:**
- POL-001: Política de Devolução de Mercadorias (versão 3.1 ou mais recente)
- PROC-042-v2: Procedimento de Cálculo de Frete Especial (versão 2.0 ou mais recente)
- SLA-2024: Tabela de SLA por Tipo de Cliente (versão 2024.1 ou mais recente)
- FAQ-Atendimento: somente itens que passarem pelo processo de curadoria definido abaixo

**O que não deve ser indexado:**
- PROC-042 v1: excluído do índice por ter sido substituído pelo PROC-042-v2 (data de emissão posterior, mesma numeração, mesma cobertura temática)
- Qualquer documento sem responsável formal, sem data de emissão, ou explicitamente marcado como obsoleto

**Critério de curadoria do FAQ:**

Cada item do FAQ deve ser classificado por um revisor das áreas de Compliance e Operações em uma das três categorias:

| Categoria | Critério | Ação |
|-----------|----------|------|
| Aprovado | Conteúdo alinhado com normativo oficial vigente | Indexar com tag de fonte: "FAQ validado — [normativo de referência]" |
| Aprovado com ressalva | Conteúdo correto mas incompleto ou com nuance não documentada | Indexar com nota de limitação explícita na resposta |
| Bloqueado | Conteúdo desatualizado, sem amparo normativo, ou com risco jurídico | Não indexar; registrar como gap a ser endereçado por novo documento |

O responsável pela curadoria é definido no kickoff do projeto. A curadoria deve ser refeita sempre que um normativo referenciado pelo FAQ for atualizado.

---

### REQ-02 — Tratamento de documentos contraditórios *(revisado: Gaps 2 e 3)*

**Definição operacional de contradição:**

Dois ou mais documentos são considerados contraditórios quando, para a mesma pergunta, retornam valores numéricos diferentes (prazos, multiplicadores, percentuais) ou instruções incompatíveis (ex: "pode devolver" vs "não pode devolver") para o mesmo cenário.

Documentos que cobrem o mesmo tema com informações complementares (ex: um define a regra geral e outro define uma exceção) não são tratados como contraditórios — são apresentados de forma integrada.

**Comportamento do assistente:**

Quando identificar contradição:

1. Apresentar as duas versões com identificação de fonte e data de emissão de cada uma.
2. Emitir alerta: "Encontrei informações divergentes entre [documento A, data] e [documento B, data] sobre este tema. Recomendo confirmar com o responsável antes de informar ao cliente."
3. Sugerir o responsável pela resolução com base nos metadados do documento (campo "Responsável" do cabeçalho).
4. Registrar automaticamente o conflito na fila de curadoria para revisão pelo responsável da base.

**Critério de teste:** dado um chamado sobre multiplicador de frete para o Norte, o assistente deve apresentar 1.6 (PROC-042 v1, mar/2023) e 1.8 (PROC-042-v2, nov/2023), emitir o alerta e registrar o conflito na fila de curadoria.

---

### REQ-03 — Comportamento quando não há resposta na base *(revisado: Gap 4)*

**Limiar de confiança:**

Se o score de similaridade dos chunks recuperados for inferior ao limiar definido pela equipe técnica (a ser calibrado durante os testes de retrieval — estimativa inicial: 0.75 em escala 0-1), o assistente trata a query como "sem resposta na base".

**Comportamento do assistente:**

1. Declarar explicitamente: "Não encontrei resposta para essa pergunta na documentação disponível."
2. Não especular, não inferir, não usar conhecimento geral.
3. Sugerir próximo passo com base em tabela de roteamento configurada no sistema:

| Tema da pergunta | Sugestão de encaminhamento |
|-----------------|---------------------------|
| Frete e cálculo | Diretoria Comercial |
| Devolução e sinistros | Gestão de Riscos (ramal 4500) |
| SLA e contratos | Gerente de conta (Gold) ou Operações (Silver/Standard) |
| Seguro de carga | Comercial |
| Outros | Supervisor de atendimento |

A tabela de roteamento é configurada no sistema e mantida pelo Product Specialist. Deve ser revisada a cada atualização de estrutura organizacional da NovaTech.

**Critério de teste:** perguntado sobre seguro de carga, o assistente deve declarar que não encontrou a informação e sugerir encaminhamento ao Comercial — nunca reproduzir os percentuais do FAQ Item 22 como resposta.

---

### REQ-04 — Atualização da base *(revisado: Gaps 5 e 6)*

**Prazo de atualização:**

Documentos publicados ou revisados no SharePoint devem estar disponíveis no assistente em até 24 horas úteis após a publicação (contadas em horário comercial, 08h-18h, dias úteis).

**Versionamento:**

A versão anterior do documento é marcada como inativa no índice — não excluída. Permanece acessível para consulta de histórico pelo time de curadoria, mas não é retornada em queries normais dos atendentes.

**Chamados em andamento:**

Um chamado é considerado "em andamento" quando seu status no sistema de chamados (Azure DevOps) for diferente de "Fechado" ou "Cancelado". Chamados em andamento no momento da atualização de um documento continuam vinculados à versão vigente no momento de abertura. Essa vinculação é registrada nos metadados do chamado.

**Notificação de atualização:**

Quando um documento for atualizado, uma notificação deve ser enviada automaticamente ao canal do time de atendimento no Teams com: nome do documento atualizado, resumo das mudanças relevantes para o atendimento, e data de vigência da nova versão.

**Critério de teste:** publicado SLA-2024-v2 no SharePoint às 09h de uma segunda-feira, até as 09h da terça-feira o assistente deve retornar os novos valores de SLA. O canal do Teams deve ter recebido a notificação de atualização.

---

### REQ-05 — Rastreabilidade das respostas *(revisado: Gap 7)*

**Citação obrigatória:**

Toda resposta deve incluir ao menos uma citação no formato: [Nome do documento, versão, seção] — [data de emissão ou última atualização].

Exemplos:
- Texto corrido: "POL-001 v3.1, seção 3.1 — jan/2024"
- Tabela: "SLA-2024 v2024.1, tabela 2, linha Gold — jan/2024"
- FAQ validado: "FAQ-Atendimento, Item 41 (validado por Compliance — fev/2024)"

**Trecho de origem por tipo de conteúdo:**

- Prosa: exibir o trecho relevante entre aspas.
- Tabela: indicar linha e coluna relevantes e reproduzir o valor da célula (ex: "Tabela de SLAs, linha Gold, coluna Tempo de resolução (chamados gerais): até 24h úteis").
- Lista numerada: citar o número do item e reproduzir o texto do item.

**Critério de teste:** toda resposta deve conter (1) nome e versão do documento, (2) seção ou localização específica, (3) data de emissão. Respostas com qualquer elemento ausente são não conformes.

---

### REQ-06 — Confiança mínima para resposta *(novo: Gap 8)*

**Limiar de similaridade:**

O assistente só gera resposta quando o score de similaridade do chunk mais relevante for superior ao limiar de confiança (estimativa inicial: 0.75 — a ser calibrado durante testes de retrieval).

**Comportamento abaixo do limiar:**

Se nenhum chunk recuperado superar o limiar, o assistente segue o comportamento definido no REQ-03.

**Comportamento com confiança intermediária:**

Se o chunk mais relevante estiver entre 0.75 e 0.80, o assistente inclui nota: "Esta resposta foi gerada com base em documentação parcialmente relacionada à sua pergunta. Confirme com o responsável antes de informar ao cliente."

**Critério de teste:** para perguntas sobre temas inexistentes na base (ex: "qual o prazo para devolução de pallets vazios"), o assistente deve declarar que não encontrou a informação — nunca gerar resposta com confiança baixa apresentada como certeza.

---

## Registro de iteração

| Versão | O que mudou | Gap que motivou |
|--------|------------|-----------------|
| v1 | Especificação inicial com 5 requisitos | Base para revisão |
| v2 — REQ-01 | Adicionado processo formal de curadoria do FAQ com 3 categorias e critérios por categoria | Gap 1 |
| v2 — REQ-02 | Adicionada definição operacional de contradição; adicionado mecanismo de registro na fila de curadoria | Gaps 2 e 3 |
| v2 — REQ-03 | Adicionado conceito de limiar de confiança; substituído "área responsável" por tabela de roteamento configurável | Gap 4 |
| v2 — REQ-04 | Definido critério para "chamado em andamento"; adicionado requisito de notificação ao time | Gaps 5 e 6 |
| v2 — REQ-05 | Diferenciado comportamento de citação para prosa, tabela e lista | Gap 7 |
| v2 — REQ-06 | Novo requisito sobre limiar de confiança e comportamento com confiança intermediária | Gap 8 |
