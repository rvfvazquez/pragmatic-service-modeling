# Example: Microservices decoupling — SQS vs EventBridge

**Skill:** `pragmatic-messaging-expert`
**Scenario:** A platform team is redesigning how their order service communicates with downstream services (inventory, billing, notifications). They need to choose between SQS and EventBridge.

---

## Conversation

**User:**
> Estamos redesenhando a comunicação entre nosso serviço de pedidos e os downstream services: estoque, faturamento e notificações. Hoje tudo é síncrono via REST e está virando um problema. Estou em dúvida entre SQS e EventBridge.

---

**Claude (pragmatic-messaging-expert):**

Faz todo sentido querer desacoplar isso — REST síncrono entre serviços de domínio diferentes cria coupling frágil. Vamos entender o seu padrão de comunicação.

**Quando um pedido é confirmado, todos os três serviços** (estoque, faturamento e notificações) **precisam ser notificados ao mesmo tempo, ou cada um é acionado por passos diferentes do fluxo?**

---

**User:**
> Quando um pedido é confirmado, os três precisam saber ao mesmo tempo. O evento "pedido confirmado" deve chegar para estoque (reservar itens), faturamento (emitir NF) e notificações (enviar e-mail para o cliente) em paralelo.

---

**Claude:**

Entendido — fan-out claro. Agora sobre os consumidores:

**Cada serviço (estoque, faturamento, notificações) processa o evento de forma independente, no seu próprio ritmo?** Ou existe alguma orquestração — por exemplo, "aguardar a reserva de estoque antes de emitir a NF"?

---

**User:**
> Independentes mesmo. Cada um processa no seu próprio ritmo. Não tem dependência entre eles para esse evento.

---

**Claude:**

Ótimo. Uma pergunta sobre volume e confiabilidade:

**Qual o volume esperado de pedidos por dia/hora no pico?** E o que acontece se um dos serviços downstream (ex: faturamento) estiver fora do ar — vocês precisam garantir que ele processe o evento quando voltar?

---

**User:**
> No pico, 2 a 3 mil pedidos por hora. Se faturamento cair, precisa processar tudo quando voltar — não pode perder nenhum evento. Notificações têm mais tolerância a falha.

---

**Claude:**

Perfeito. Mais uma questão sobre o futuro da plataforma:

**Vocês têm outros serviços que podem querer consumir eventos de pedido no futuro?** Por exemplo, um serviço de analytics, detecção de fraude ou loyalty. E esses eventos são todos da mesma fonte (o serviço de pedidos) ou vocês têm múltiplas fontes de eventos que precisam ser roteadas?

---

**User:**
> Sim, provavelmente vamos adicionar mais consumidores — analytics e loyalty já estão no roadmap. E hoje os eventos são só do serviço de pedidos, mas em 6 meses vamos ter eventos de devolução e eventos do serviço de entrega também.

---

**Claude:**

Com esse contexto — múltiplos consumidores atuais e futuros, múltiplas fontes de eventos no roadmap — tenho uma recomendação clara.

---

### Recommendation: EventBridge (event bus) + SQS (per consumer)

**Recomendação primária: Amazon EventBridge**

EventBridge é o serviço certo para o seu caso por três razões conectadas ao seu contexto:

- **Fan-out para múltiplos consumidores independentes** — EventBridge foi construído exatamente para isso. Você publica `order.confirmed` uma vez no event bus, e os três serviços (estoque, faturamento, notificações) recebem via rules independentes. Adicionar analytics e loyalty no futuro é uma nova rule — zero mudança no producer.
- **Múltiplas fontes de eventos no roadmap** — devolução e entrega publicam seus próprios eventos no mesmo bus. EventBridge roteia automaticamente para os consumidores corretos por tipo de evento, source ou qualquer atributo do payload. Com SQS puro, você precisaria de uma fila por combinação de producer → consumer.
- **Schema Registry** — EventBridge tem um registry nativo onde você documenta e versiona o contrato dos seus eventos. Importante quando múltiplos times publicam no mesmo bus.

---

### Architecture pattern: EventBridge + SQS Fan-out

```
[Order Service]
      |
      | publishes: order.confirmed
      ↓
[EventBridge Event Bus]
      |
      ├── Rule: order.confirmed → [SQS: inventory-queue]    → Inventory Service
      ├── Rule: order.confirmed → [SQS: billing-queue]      → Billing Service
      └── Rule: order.confirmed → [SQS: notification-queue] → Notification Service
```

**Por que SQS por consumidor, não entrega direta do EventBridge?**

EventBridge pode entregar direto para Lambda, ECS, etc. — mas adicionar uma SQS por consumer é o padrão recomendado porque:

- **Durabilidade garantida**: se faturamento cair, a mensagem fica na fila até 14 dias esperando o serviço voltar. Você mencionou que não pode perder eventos de faturamento.
- **Rate control**: cada consumidor processa no seu ritmo sem sobrecarregar um serviço lento
- **Dead Letter Queue (DLQ)**: mensagens que falham repetidamente vão para DLQ por consumer — você monitora e reprocessa seletivamente
- **Retry logic**: SQS retry + DLQ é mais simples e confiável do que implementar retry no próprio consumidor

Para notificações (mais tolerante a falha), você pode optar por entrega direta do EventBridge para Lambda se preferir simplicidade — mas a SQS por consistência de arquitetura vale a pena.

**Tradeoff principal:**

EventBridge tem latência levemente maior que SQS puro (geralmente ainda abaixo de 1 segundo). Para o seu volume de 3k pedidos/hora e processamento assíncrono, isso é irrelevante. Se você precisasse de sub-100ms garantido, revisitaríamos.

**Sobre SQS puro sem EventBridge:** funcionaria para o estado atual (3 consumidores, 1 producer), mas te prenderia em uma topologia manual conforme a plataforma cresce. Você acabaria implementando sua própria lógica de fan-out.

---

**Próximos passos:**

1. **Defina o schema do evento `order.confirmed`** no EventBridge Schema Registry — inclua `orderId`, `customerId`, `items`, `totalAmount`, `timestamp`. Versione desde o início.
2. **Configure DLQ por fila SQS** com alertas no CloudWatch — quando uma mensagem vai para DLQ de faturamento, você quer saber imediatamente.

---

*Skill: pragmatic-messaging-expert | Services evaluated: SQS, SNS, EventBridge*
