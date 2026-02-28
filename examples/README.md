# Examples / Exemplos

This folder contains complete conversation walkthroughs showing how each skill interviews a user and delivers a recommendation.

Esta pasta contém exemplos completos de conversas mostrando como cada skill entrevista o usuário e entrega uma recomendação.

---

## Compute

| Example | Scenario | Recommendation |
|---|---|---|
| [startup-lambda-vs-ecs.md](./compute/startup-lambda-vs-ecs.md) | Startup processing orders, team of 3, spiky traffic | Lambda |

## Database

| Example | Scenario | Recommendation |
|---|---|---|
| [ecommerce-dynamodb-vs-aurora.md](./database/ecommerce-dynamodb-vs-aurora.md) | E-commerce product catalog, 50k req/min, Lambda stack | DynamoDB + OpenSearch |

## Messaging

| Example | Scenario | Recommendation |
|---|---|---|
| [microservices-sqs-vs-eventbridge.md](./messaging/microservices-sqs-vs-eventbridge.md) | Order service fan-out to 3 downstream services, growing platform | EventBridge + SQS per consumer |

---

## How to read these examples / Como ler esses exemplos

Each example follows the same format:

- **Context** — the scenario and what the user is trying to decide
- **Conversation** — the full interview Claude conducted
- **Recommendation** — the final verdict with justification, tradeoffs, and next steps

The goal is to show the *quality of reasoning* the skill produces, not just the final answer.
