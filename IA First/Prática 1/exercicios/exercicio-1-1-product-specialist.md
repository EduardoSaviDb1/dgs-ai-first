# Exercício 1.1 — Mapeamento de intent com engenharia de contexto
**Papel:** Product Specialist  
**Projeto:** Assistente de IA NovaTech (DB1)  
**Fase:** Intent + Discovery

---

## 1. Etapa 1 — Visão geral

### Decisão de contexto
Nesta etapa, foram fornecidos ao Claude **apenas títulos, metadados e descrições resumidas** dos 5 documentos, sem nenhum conteúdo completo. Isso foi feito com o objetivo de não sobrecarregar o contexto com detalhes que podem não ser relevantes no momento. Fazendo com o que o chat tenha mais objetividade e desperdice menos tokens no proccesso.

### Prompt executado

```
Sou analista de negócios na fase de Intent + Discovery de um projeto que visa construir 
um assistente de IA para atendentes da NovaTech, empresa de logística. O assistente 
permitirá perguntas em linguagem natural com respostas fundamentadas na documentação 
oficial, indicando a fonte. Será integrado ao Microsoft Teams + SharePoint.

Abaixo estão os metadados dos 5 documentos disponíveis. Não tenho o conteúdo completo 
agora — apenas os títulos e descrições.

1. POL-001: Política de Devolução de Mercadorias — Define regras para devolução em até 
   7 dias, com exceções para carga perigosa.
2. PROC-042: Procedimento de Cálculo de Frete Especial — Fórmula para fretes acima de 
   500kg com tabela de multiplicadores por região.
3. SLA-2024: Tabela de SLA por Tipo de Cliente — Prazos de resposta diferenciados para 
   clientes Gold, Silver e Standard.
4. PROC-042-v2: Procedimento de Cálculo de Frete (Revisado) — Mesma numeração do item 2, 
   mas com multiplicadores diferentes. Sem indicação de qual é o vigente.
5. FAQ-Atendimento: Perguntas Frequentes do Time de Suporte — Documento informal com 
   47 perguntas e respostas escritas por atendentes experientes, sem validação formal.

Com base nesses metadados, gere:
- Um mapa de temas cobertos pela documentação atual.
- Hipóteses de gaps (o que provavelmente está faltando).
- Alertas de risco por documento.
```

### Output obtido

**Temas cobertos:**

| Tema | Documento(s) |
|------|-------------|
| Política de devolução | POL-001 |
| Cálculo de frete especial (>500kg) | PROC-042 e PROC-042-v2 (conflito) |
| SLA por tipo de cliente | SLA-2024 |
| Práticas informais de atendimento | FAQ-Atendimento |

**Hipóteses de gaps identificados:**

- Ausência de documento sobre seguro de carga e tratamento de sinistros
- Nenhuma cobertura de rastreamento de pedidos
- Frete para carga perigosa não está explicitamente coberto
- Sem normativo sobre escalonamento de chamados
- Processo de atualização de documentação sem cobertura formal

**Alertas de risco por documento:**

- **PROC-042 / PROC-042-v2 (CRÍTICO):** Dois documentos com a mesma numeração e multiplicadores diferentes, sem indicação de qual é o vigente. Um assistente que indexar ambos pode gerar respostas inconsistentes de forma silenciosa.
- **FAQ-Atendimento (MÉDIO):** Conteúdo não validado formalmente. Pode conter práticas desatualizadas ou que contradizem normativos oficiais.
- **SLA-2024 (BAIXO):** Cobre apenas três tiers (Gold, Silver, Standard). Se existirem outros tiers na operação real, o assistente dirá que não encontrou resposta para clientes válidos.
- **POL-001 (BAIXO):** Exceções para carga perigosa estão mencionadas mas provavelmente pouco detalhadas — risco de o atendente precisar de mais granularidade do que o documento oferece.

### Análise da etapa
A qualidade do output foi boa para o objetivo solicitado. Foi produzido um mapa estruturado e apontado o conflito entre os dois PROC-042 como risco prioritário, mesmo sem ver o conteúdo. Isso confirma que metadados bem descritos podem ser suficientes para a fase de mapeamento dependendo do contexto. A limitação esperada é que as hipóteses de gaps são apenas deduções que ainda precisam ser confirmadas nas etapas seguintes e no discovery.

---

## 2. Etapa 2 — Análise profunda

### Decisão de contexto
Com base no mapa da Etapa 1, os dois documentos selecionados para análise profunda foram **PROC-042 e PROC-042-v2**. A escolha é justificada pelo risco operacional direto, documentação ambigua e a prioridade elencada pelo mapeamento inicial da primeira etapa com um alto risco.

O FAQ não foi incluído nesta etapa deliberadamente: seu volume (47 itens) consumiria orçamento de atenção antes que as inconsistências normativas estivessem claramente mapeadas.

### Prompt executado

```
Na etapa anterior, o mapa de temas sinalizou conflito crítico entre dois documentos de cálculo de frete. Segue o conteúdo completo em anexo de ambos os documentos. Analise as inconsistências entre os dois documentos. 
```

### Output obtido

**Inconsistências identificadas:**

| Campo | PROC-042 (v1) | PROC-042-v2 | Impacto |
|-------|--------------|-------------|---------|
| Multiplicador Norte | 1.6 | 1.8 | Diferença de 12.5% no valor final do frete |
| Multiplicador Nordeste | 1.4 | 1.5 | Diferença de 7.1% |
| Multiplicador Centro-Oeste | 1.2 | 1.3 | Diferença de 8.3% |
| Multiplicador Sudeste | 1.0 | 1.1 | Diferença de 10% |
| Multiplicador Sul | 1.1 | 1.2 | Diferença de 9.1% |
| Limiar de desconto | 10 fretes/mês → 5% | 8 fretes/mês → 8% ou 12% | Critério de elegibilidade diferente |
| Estrutura de desconto | Única faixa | Duas faixas progressivas | Lógica de cálculo diferente |

**Risco específico para o assistente de IA:**

Se ambos os documentos forem indexados sem metadado de vigência, o retriever pode retornar chunks de versões diferentes para a mesma pergunta, dependendo de qual trecho tem maior similaridade semântica com a query. O resultado é uma resposta que mistura multiplicadores das duas versões sem que o atendente — ou o sistema — perceba. Não há sinal de erro: a resposta parece correta e fundamentada.

O segundo risco é de inconsistência por query: um atendente que pergunta sobre frete para o Norte em um chamado e sobre desconto por volume em outro pode receber respostas baseadas em versões diferentes, sem relação entre elas.

### Análise crítica da etapa
O resultado melhorou significativamente em relação à Etapa 1. Com o conteúdo completo dos dois documentos, foi possível quantificar as divergências e descrever o mecanismo específico de falha no contexto de RAG. Isso não seria possível apenas com metadados. Ainda assim deixamos os outros documentos de fora para ter mais foco no resultado.

---

## 3. Etapa 3 — Cruzamento

### Decisão de contexto
Nesta etapa, o FAQ-Atendimento foi adicionado junto com os resultados das Etapas 1 e 2. A ordem importa: o FAQ só faz sentido ser analisado depois que as inconsistências normativas já estão mapeadas, porque o objetivo é verificar se as práticas dos atendentes refletem a v1, a v2, ou algo diferente de ambas. Adicionar o FAQ antes poderia ter apenas acrescido o contexto e deixado a mercê da IA decidir o que faria com os arquivos e como chegaria no resultado sozinha, podendo ocasionar um resultado menos satisfatório.

### Prompt executado

```
Dadas as inconsistencias entre os arquivos PROC-042, cruze as inconsistencias identificadas com o que o FAQ diz sobre os mesmos temas em busca de contradições, gaps, informações divergentes da documentação.
```

### Output obtido

**Itens em conflito direto com normativos:**

- **Item 45 × PROC-042-v2:** O FAQ ainda usa a regra da v1 (limiar de 10 fretes, desconto de 5%). A v2 estabelece limiar em 8 fretes com duas faixas (8% e 12%). Um atendente guiado pelo FAQ deixará de aplicar descontos que a v2 já prevê — ou aplicará desconto incorreto.
- **Item 38 (risco jurídico):** A promessa de "reembolso integral em até 48h" para carga danificada não tem amparo em nenhum documento normativo disponível. Se reproduzida pelo assistente, pode criar expectativa contratual sem base formal.
- **Item 3 × POL-001:** O FAQ institucionaliza uma exceção informal (devolução de carga perigosa mediante termo) que a política oficial não prevê formalmente. A prática pode existir, mas sem normativo que a sustente, o assistente não tem base para afirmá-la.

**Itens que cobrem gaps não endereçados por normativos:**

- **Item 22:** Seguro de carga e prazo de indenização — nenhum documento oficial cobre.
- **Item 27:** Rastreamento por rota e região — nenhum documento oficial cobre.
- **Item 32:** Frete expresso para carga perigosa — nenhum documento oficial cobre. Adicionalmente, o item referencia "PROC-042" sem especificar qual versão.

**Itens aproveitáveis com ressalva:**

- **Item 41:** Alinhado com SLA-2024 para clientes Gold, mas menciona "incidentes críticos" — categoria não definida no SLA. Pode ser indexado após validação do gestor de SLA sobre o que se enquadra nessa categoria.

### Análise crítica da etapa
O cruzamento trouxe a tona um resultado bem relevante:
- Os itens do FAQ que contradizem normativos e precisam ser bloqueados antes da indexação. 
- Os itens que preenchem lacunas reais da documentação com conhecimento não validado. 
- O terceiro é conteúdo aproveitável com ressalvas pontuais. 

Entendo que esse resultado foi possível pois realizamos o passo a passo de raciocínio juntamente com a IA, direcionando o processo como seria executado em um processo humano. Apesar da IA ter conhecimento e estruturar suas execuções previamente, dado a grande quantidade de contexto, poderiamos chegar a um resultado mais genérico e impreciso. Ter adicionado o FAQ antes só aumentaria esse risco.

---

## 4. Mapa de riscos e proposta de tratamento no discovery humano

### Risco 1 — Documentos contraditórios indexados sem metadado de vigência

**Descrição:** PROC-042 e PROC-042-v2 têm a mesma numeração e multiplicadores diferentes para todas as regiões. Sem vigência explícita, o pipeline de RAG não tem critério para priorizar uma versão.

**Impacto:** Cálculos de frete incorretos para fretes especiais acima de 500kg — sem sinal de erro para o atendente ou para o sistema.

**Proposta para o discovery humano:**
- Entrevistar o responsável pela área de Operações para verificar qual versão é vigente, se os dados estão corretos na versão definida, ou ainda se existe a necessidade de manter ambas as versões por algum "corte temporal" dos pedidos e verificações.
- Verificar se existe processo formalizado para descontinuação de versões de arquivos de definições como este e demais arquivos que serão usados pelo RAG.
- Levantar quantos outros documentos na base podem ter o mesmo versionamento informal.
- Definir, como requisito para a construção do produto, que todo documento deve ter a informação de vigência obrigatório antes da indexação.

### Risco 2 — Conteúdo do FAQ sem amparo normativo reproduzido como verdade pelo assistente

**Descrição:** Múltiplos itens do FAQ (especialmente Item 38 e Item 3) descrevem práticas que não têm respaldo em nenhum documento oficial disponível. Se indexados, o assistente os reproduzirá com a mesma confiança que reproduz normativos formais — sem distinção de fonte.

**Impacto:** Risco jurídico (promessas sem base contratual), inconsistência de atendimento e redução de confiança quando o assistente der respostas que o supervisor ou o normativo contradizem.

**Proposta para o discovery humano:**
- Conduzir sessão de validação do FAQ com Operações, Compliance e Comercial, item por item, para classificar cada entrada como: (a) válido e alinhado ao normativo, (b) válido mas sem normativo — exige criação de documento, ou (c) prática incorreta — deve ser corrigida.
- Identificar os 3 itens que cobrem gaps reais (seguro, rastreamento, frete expresso para carga perigosa) e levantar com as áreas responsáveis se existe normativo não compartilhado ou se precisam ser criados.
- Definir como requisito do produto que o assistente trate fontes formais e FAQ com pesos diferentes — ou que o FAQ não seja indexado até validação completa.

---

## 5. Reflexão: Progressive disclosure versus tudo de uma vez

### O que teria acontecido com os 5 documentos completos no primeiro prompt

Se os 5 documentos completos tivessem sido fornecidos de uma vez, o modelo teria enfrentado dois problemas simultâneos.

O primeiro é orçamento de atenção esgotado logo no início. Um contexto com 5 documentos completos concentra a maior parte dos tokens em conteúdo que só é relevante nas etapas posteriores. O resultado esperado é que o modelo produza um resumo genérico de cada documento sem identificar as relações entre eles, especialmente as contradições mais detalhadas como a diferença nos limites de desconto dos arquivos PROC-042.

O segundo é context rot na análise de inconsistências. Ao analisar os dois PROC-042 no meio de um contexto que também inclui os demais arquivos, o modelo tende a perder o fio dispersar mais a atenção, levando a uma análise mais superficial e podendo apenas apontar que existem versões com multiplicadores diferentes, mas sem definir ou trazer detalhes sobre.

### O que a abordagem progressiva produziu de diferente

A abordagem em 3 etapas gerou resultados qualitativamente superiores porque cada camada recebeu exatamente o contexto necessário. O resultado dessa progressão é a diferença entre um resumo genérico com apenas algumas notas de atenção, para um mapa de risco mais completo e utilizável que auxilia no direcionamento e alimenta o roteiro de discovery.
