# Exercício 1.2 — Design de jornada com componente de IA
**Papel:** Product Specialist  
**Projeto:** Assistente de IA NovaTech (DB1)  
**Fase:** Intent + Discovery

---

## 1. Contexto e decisões de design

A jornada foi desenhada a partir dos insumos do discovery:

- Os atendentes abrem em média 4 fontes diferentes por chamado.
- Em 15% dos casos, o atendente não encontra resposta e escala para o supervisor.
- As dúvidas mais comuns são: prazos de entrega (35%), regras de frete (25%), política de devolução (20%) e outros (20%).

Essas informações orientaram três decisões de design:

**1. O assistente não substitui o julgamento do atendente, ele reduz a busca.** O fluxo principal termina com o atendente decidindo se usa a resposta, não com o assistente respondendo diretamente ao cliente. Isso é intencional: o assistente é uma ferramenta de consulta, não um agente autônomo de atendimento.

**2. O fallback existe para os 15% que hoje vão para o supervisor.** O objetivo não é eliminar a escala — é qualificá-la. Quando o assistente não tem confiança, ele diz explicitamente e sugere o próximo passo, em vez de inventar uma resposta.

**3. O feedback loop é parte do produto, não um recurso adicional.** RAG precisa de manutenção contínua. Se o atendente não tem como sinalizar que uma resposta estava errada ou desatualizada, a base se degrada silenciosamente.

---

## 2. Guardrails de comportamento do assistente

Antes dos fluxos, é necessário definir os guardrails — as regras de comportamento que o assistente deve seguir em qualquer situação.

**Guardrail 1 — Nunca afirmar prazo ou valor sem citação de fonte verificável**  
O assistente só informa prazos (SLA, devolução, entrega) e valores (multiplicadores de frete, percentuais de desconto) se conseguir citar o documento e a seção de origem. Se os chunks recuperados não contiverem informação suficiente, o assistente declara explicitamente: *"Não encontrei essa informação na documentação disponível."* Isso endereça diretamente o risco de alucinação identificado no Exercício 1.1 — especialmente crítico dado o conflito entre PROC-042 e PROC-042-v2.

**Guardrail 2 — Quando houver documentos contraditórios, apresentar ambas as versões com alerta**  
Se o retriever retornar chunks de versões diferentes do mesmo procedimento (ex: os dois PROC-042), o assistente não escolhe silenciosamente uma versão. Ele apresenta as duas com a seguinte estrutura: *"Encontrei duas versões deste procedimento com informações divergentes. [versão A diz X — fonte]. [versão B diz Y — fonte]. Recomendo confirmar com o responsável pelo procedimento qual é a vigente antes de informar ao cliente."* Esse guardrail é diretamente derivado do risco 1 mapeado no Exercício 1.1.

**Guardrail 3 — Responder sempre em português formal e nunca usar jargão técnico de IA**  
As respostas do assistente são lidas pelos atendentes durante chamados ao vivo. Linguagem informal ou termos como "com base no meu treinamento" ou "não tenho certeza" criam ruído. O padrão é linguagem formal, direta e orientada à ação.

**Guardrail 4 — Declarar explicitamente quando não encontrar resposta, sem especular**  
Se a pergunta não tiver cobertura na base indexada, o assistente não infere, não extrapola e não sugere que "provavelmente" é de uma forma. A resposta padrão é: *"Não encontrei resposta para essa pergunta na documentação disponível. Recomendo consultar [área responsável]."*

---

## 3. Fluxo principal — Caminho feliz

**Contexto:** O atendente está em um chamado ativo e precisa de uma informação que normalmente buscaria em 4 fontes diferentes.

```
[1] ATENDENTE recebe dúvida do cliente durante chamado
        |
        v
[2] ATENDENTE abre o assistente no Teams e digita a pergunta
    em linguagem natural
    Exemplo: "Qual o prazo de devolução para cliente Gold 
    que quer devolver carga não-perigosa?"
        |
        v
[3] ASSISTENTE recupera chunks relevantes da base indexada
    (POL-001 + SLA-2024)
        |
        v
[4] ASSISTENTE gera resposta com:
    - Resposta direta à pergunta
    - Fonte citada (documento + seção)
    - Nível de confiança implícito na citação
    Exemplo: "Conforme POL-001, seção 3.1, o prazo padrão 
    de devolução é de 7 dias corridos. Para clientes Gold, 
    o SLA-2024 não prevê prazo diferenciado para devoluções 
    — o prazo de 7 dias se aplica igualmente."
        |
        v
[5] ATENDENTE avalia a resposta:
    - A fonte citada é conhecida e confiável?
    - A resposta faz sentido para o contexto do chamado?
        |
    [SIM, está ok]
        |
        v
[6] ATENDENTE usa a informação no atendimento ao cliente
        |
        v
[7] FIM DO FLUXO PRINCIPAL
```

**Critério de saída do fluxo principal:** o atendente conseguiu a informação necessária, com fonte, em tempo menor que a busca manual (meta: menos de 2 minutos).

---

## 4. Fluxo de fallback — Quando o assistente não resolve

**Contexto:** O assistente retorna resposta com baixa confiança, retorna documentos contraditórios, ou o atendente discorda da resposta com base no seu conhecimento.

```
[1] ATENDENTE recebe resposta do assistente
        |
        v
[2] ATENDENTE identifica um dos seguintes problemas:
    (A) O assistente declarou que não encontrou resposta
    (B) O assistente apresentou duas versões contraditórias 
        com alerta (Guardrail 2)
    (C) O atendente discorda da resposta com base em 
        conhecimento próprio ou experiência recente
        |
        v
[3] ASSISTENTE sugere próximo passo conforme o caso:
    (A) Não encontrou: "Recomendo consultar [área responsável 
        pelo tema — ex: Operações para frete, Compliance 
        para devoluções]."
    (B) Contraditório: "Recomendo confirmar com o responsável 
        pelo PROC-042 qual versão está vigente antes de 
        informar ao cliente."
    (C) Discordância do atendente: o atendente aciona o 
        fluxo de feedback (ver Seção 5) antes de escalar
        |
        v
[4] ATENDENTE decide:
    ┌─────────────────────────────────────┐
    │ Pode resolver consultando colega    │──→ [5A] Consulta colega ou área 
    │ ou área responsável?               │         responsável diretamente
    └─────────────────────────────────────┘
    ┌─────────────────────────────────────┐
    │ Precisa escalar para supervisor?   │──→ [5B] Escala com contexto:
    └─────────────────────────────────────┘         "O assistente não encontrou /
                                                     encontrou versões contraditórias
                                                     sobre [tema]. Preciso de orientação."
        |
        v
[6] ATENDENTE registra o fallback no assistente
    (ver Fluxo de Feedback, item 5.2)
        |
        v
[7] FIM DO FLUXO DE FALLBACK
```

**Critério de qualidade do fallback:** o atendente chega à escala com contexto qualificado — não apenas "não sabia", mas "o assistente não encontrou documentação sobre X" ou "há conflito entre versões de Y". Isso torna a escala mais rápida e rastreável.

---

## 5. Fluxo de feedback — Quando a resposta estava errada ou desatualizada

**Contexto:** O assistente retornou uma resposta que o atendente identificou como incorreta, desatualizada ou incompleta — seja durante o chamado, seja depois.

O feedback loop é o mecanismo que impede que a base se degrade silenciosamente ao longo do tempo. Sem ele, erros se acumulam e o assistente perde credibilidade progressivamente.

### 5.1 Tipos de feedback

| Tipo | Descrição | Exemplo |
|------|-----------|---------|
| **Incorreto** | A resposta contradiz o que o atendente sabe ser verdade | "O assistente disse prazo de 7 dias, mas o procedimento foi atualizado para 5 dias no mês passado" |
| **Desatualizado** | A resposta estava correta mas o normativo mudou | "Esse multiplicador de frete mudou na última revisão" |
| **Incompleto** | A resposta existe mas faltou informação relevante | "Faltou mencionar a exceção para clientes com contrato especial" |
| **Fonte errada** | A resposta citou o documento, mas a seção está incorreta | "A seção 3.2 não fala sobre isso — é a 3.4" |

### 5.2 Fluxo de registro

```
[1] ATENDENTE identifica problema na resposta do assistente
        |
        v
[2] ATENDENTE clica em "Reportar problema" na interface
    do assistente (botão presente em toda resposta)
        |
        v
[3] ASSISTENTE apresenta formulário simplificado:
    - Tipo do problema (incorreto / desatualizado / 
      incompleto / fonte errada)
    - Campo livre: "O que estava errado ou faltando?"
    - Campo opcional: "Qual a informação correta?" 
      (se o atendente souber)
        |
        v
[4] FEEDBACK é registrado com:
    - ID da query original
    - Resposta que gerou o feedback
    - Chunks que foram usados para gerá-la
    - Tipo e descrição do problema
    - Atendente que reportou (para follow-up se necessário)
        |
        v
[5] FILA DE REVISÃO é atualizada para o responsável 
    pela curadoria da base (definido no projeto)
        |
        v
[6] RESPONSÁVEL pela curadoria avalia:
    ┌──────────────────────────────────────┐
    │ Problema confirmado?                 │
    └──────────────────────────────────────┘
         │                    │
        [SIM]               [NÃO]
         │                    │
         v                    v
    [7A] Documento       [7B] Feedback
    é corrigido /        é arquivado com
    atualizado na        justificativa
    base                 (para rastreio)
         │
         v
    [8] ATENDENTE que reportou é notificado
        da correção (fecha o loop)
```

**Critério de SLA do feedback:** feedbacks do tipo "incorreto" devem ser avaliados em até 24h. Feedbacks "desatualizado" em até 48h. Isso garante que erros com impacto em chamados ativos sejam corrigidos rapidamente.

---

## 6. Resumo dos fluxos

| Fluxo | Gatilho | Resultado esperado | Métrica de sucesso |
|-------|---------|-------------------|-------------------|
| **Principal** | Atendente tem dúvida durante chamado | Informação com fonte em menos de 2 minutos | Tempo médio de busca < 2min |
| **Fallback** | Assistente não resolve ou há conflito | Escala qualificada ou consulta direta à área | % de escalas com contexto registrado |
| **Feedback** | Resposta incorreta, desatualizada ou incompleta | Correção na base em até 24-48h | Tempo médio de resolução de feedback |