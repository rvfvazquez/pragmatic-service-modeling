# Example: Startup choosing between Lambda and ECS

**Skill:** `pragmatic-compute-expert`
**Scenario:** A small startup building a new order-processing microservice. The team is uncertain whether to go serverless with Lambda or containerized with ECS.

---

## Conversation

**User:**
> Qual compute devo usar para o meu novo microserviço de processamento de pedidos?

---

**Claude (pragmatic-compute-expert):**

Boa pergunta — vamos descobrir o que faz mais sentido para o seu contexto. Me conta um pouco sobre o tráfego.

**O tráfego do seu serviço de pedidos é previsível e estável, ou ele vem em picos (ex: campanhas, horários de pico)?** E o processamento é disparado por um evento (novo pedido chega) ou o serviço precisa rodar continuamente esperando requisições?

---

**User:**
> O tráfego é bem irregular — temos picos no almoço e à noite, mas à madrugada é quase zero. O serviço é disparado quando um novo pedido chega via API.

---

**Claude:**

Perfeito, tráfego spike-driven com evento claro (chegada de pedido) já aponta numa direção. Agora me conta sobre o processamento em si:

**Quanto tempo leva para processar um pedido, em média?** E o trabalho é stateless (cada pedido é independente dos outros) ou você precisa manter estado entre chamadas?

---

**User:**
> Cada pedido leva entre 2 e 8 segundos para processar — valida estoque, chama o gateway de pagamento e grava no banco. É stateless, cada pedido é independente. Somos um time de 3 devs, startup early-stage.

---

**Claude:**

Ótimo contexto. Uma última pergunta sobre infraestrutura:

**Seu time tem experiência com containers Docker?** E minimizar custo é a prioridade agora, ou vocês preferem uma arquitetura mais simples de operar?

---

**User:**
> Usamos Docker no dev, mas nunca deployamos nada com ECS. Custo importa bastante, mas simplicidade também — a gente não tem DevOps dedicado.

---

**Claude:**

Com base no que você me contou, tenho o suficiente para uma recomendação clara.

---

### Recommendation: AWS Lambda

**Por que Lambda ganha para o seu caso:**

- **Tráfego irregular com picos** — Lambda escala do zero ao pico instantaneamente e você paga apenas pelo que usa. Com tráfego quase zero à madrugada, você economiza 100% do custo ocioso que teria com ECS rodando 24/7.
- **Processamento de 2–8 segundos, stateless, disparado por evento** — esse é o caso de uso perfeito para Lambda. Bem dentro do limite de 15 minutos, sem necessidade de estado persistente.
- **Time de 3 pessoas sem DevOps** — Lambda elimina toda a gestão de infraestrutura. Sem clusters, sem task definitions, sem auto-scaling configurado manualmente. Você deployta código e pronto.

**Tradeoff principal a considerar:**

Cold starts podem adicionar latência na primeira invocação após um período inativo (geralmente 200ms–1s dependendo do runtime e tamanho do pacote). Para um serviço de pedidos, isso raramente é problema — se for, Provisioned Concurrency resolve sem mudar a arquitetura.

**Sobre ECS/Fargate:** faz sentido considerar quando vocês tiverem processos mais longos (> 15 min), precisarem de workers em background contínuos, ou o time crescer e quiser mais controle. Não é o caso agora.

---

**Próximos passos:**

1. **Defina os limites de concorrência reservada** para proteger o gateway de pagamento e o banco de picos de Lambda demais invocações simultâneas (ex: `ReservedConcurrency: 50`)
2. **Configure um Dead Letter Queue (DLQ) via SQS** para capturar pedidos que falharem no processamento e garantir que nenhum pedido seja perdido silenciosamente

---

*Skill: pragmatic-compute-expert | Services evaluated: Lambda, ECS Fargate*
