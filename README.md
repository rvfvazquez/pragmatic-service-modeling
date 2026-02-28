# pragmatic-service-modeling

> **AI-powered AWS architecture advisor** — A collection of Claude skills that guide architects and developers through structured decision-making for AWS service selection.
>
> **Consultor de arquitetura AWS com IA** — Uma coleção de skills para Claude que orientam arquitetos e desenvolvedores em decisões estruturadas de seleção de serviços AWS.

---

## 🇺🇸 English

### Overview

`pragmatic-service-modeling` is a plugin for [Claude](https://claude.ai) (via [Cowork](https://claude.ai/download) or [Claude Code](https://claude.ai/claude-code)) that provides three expert skills for AWS architecture decisions. Instead of returning a generic answer, each skill conducts a short **conversational interview** to understand your specific context, then delivers a direct, justified recommendation.

### Skills

| Skill | What it decides | Key services covered |
|---|---|---|
| [pragmatic-compute-expert](./skills/pragmatic-compute-expert/) | Which compute model to use | Lambda, ECS, EKS, Fargate, EC2 |
| [pragmatic-db-expert](./skills/pragmatic-db-expert/) | Which database to use | DynamoDB, DocumentDB, Aurora DSQL |
| [pragmatic-messaging-expert](./skills/pragmatic-messaging-expert/) | Which messaging service to use | SQS, SNS, EventBridge, Kinesis, MSK |

### How it works

Each skill follows the same pattern:

1. **Interview** — Claude asks 4–6 focused questions about your workload, team, and constraints
2. **Analysis** — Claude applies the decision framework from the `references/` folder
3. **Recommendation** — A direct verdict with justification, tradeoffs, and next steps

### Installation

#### Via Cowork (desktop app)

1. Download `pragmatic-service-modeling.skill` from the [releases page](../../releases)
2. Open the Cowork app and go to **Plugins → Install from file**
3. Select the `.skill` file — all three experts are installed at once

#### Via Claude Code

```bash
# Clone the repository
git clone https://github.com/rvfvazquez/pragmatic-service-modeling.git

# Copy skills to your Claude skills directory
cp -r pragmatic-service-modeling/skills/* ~/.claude/skills/
```

### Usage examples

Just talk to Claude naturally:

```
"Qual compute devo usar para o meu novo microserviço?"
"Should I use Lambda or ECS for this background job?"
"Which database makes sense for a high-throughput e-commerce catalog?"
"Precisamos de SQS ou EventBridge para desacoplar nossos serviços?"
```

See the [examples/](./examples/) folder for complete conversation walkthroughs.

### Repository structure

```
pragmatic-service-modeling/
├── plugin.json                        ← Plugin manifest (bundles all skills)
├── README.md
├── CONTRIBUTING.md
├── LICENSE
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── skill_proposal.md
│   └── PULL_REQUEST_TEMPLATE.md
├── skills/                            ← All sub-skills live here
│   ├── pragmatic-compute-expert/      ← Skill: compute model selection
│   │   ├── SKILL.md                   ← Skill prompt and decision logic
│   │   └── references/
│   │       └── aws-compute.md         ← Detailed reference for Lambda, ECS, EKS
│   ├── pragmatic-db-expert/           ← Skill: database selection
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── aws-databases.md
│   └── pragmatic-messaging-expert/    ← Skill: messaging service selection
│       ├── SKILL.md
│       └── references/
│           └── aws-messaging.md
└── examples/
    ├── README.md
    ├── compute/
    │   └── startup-lambda-vs-ecs.md
    ├── database/
    │   └── ecommerce-dynamodb-vs-aurora.md
    └── messaging/
        └── microservices-sqs-vs-eventbridge.md
```

### Contributing

We welcome new skills, improved decision logic, and updated references. See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

---

## 🇧🇷 Português

### Visão Geral

`pragmatic-service-modeling` é um plugin para [Claude](https://claude.ai) (via [Cowork](https://claude.ai/download) ou [Claude Code](https://claude.ai/claude-code)) que oferece três skills especializadas em decisões de arquitetura AWS. Em vez de retornar uma resposta genérica, cada skill conduz uma **entrevista conversacional** para entender seu contexto específico e entrega uma recomendação direta e justificada.

### Skills disponíveis

| Skill | O que decide | Serviços cobertos |
|---|---|---|
| [pragmatic-compute-expert](./skills/pragmatic-compute-expert/) | Qual modelo de compute usar | Lambda, ECS, EKS, Fargate, EC2 |
| [pragmatic-db-expert](./skills/pragmatic-db-expert/) | Qual banco de dados usar | DynamoDB, DocumentDB, Aurora DSQL |
| [pragmatic-messaging-expert](./skills/pragmatic-messaging-expert/) | Qual serviço de mensageria usar | SQS, SNS, EventBridge, Kinesis, MSK |

### Como funciona

Cada skill segue o mesmo padrão:

1. **Entrevista** — Claude faz 4–6 perguntas focadas sobre sua carga de trabalho, equipe e restrições
2. **Análise** — Claude aplica o framework de decisão da pasta `references/`
3. **Recomendação** — Um veredicto direto com justificativa, tradeoffs e próximos passos

### Instalação

#### Via Cowork (app desktop)

1. Baixe `pragmatic-service-modeling.skill` na [página de releases](../../releases)
2. Abra o Cowork e vá em **Plugins → Instalar do arquivo**
3. Selecione o arquivo `.skill` — os três experts são instalados de uma vez

#### Via Claude Code

```bash
# Clone o repositório
git clone https://github.com/rvfvazquez/pragmatic-service-modeling.git

# Copie as skills para seu diretório de skills do Claude
cp -r pragmatic-service-modeling/skills/* ~/.claude/skills/
```

### Como usar

Fale com o Claude naturalmente:

```
"Qual compute devo usar para o meu novo microserviço?"
"Devo usar Lambda ou ECS para esse job em background?"
"Qual banco faz sentido para um catálogo de e-commerce com alto throughput?"
"Precisamos de SQS ou EventBridge para desacoplar nossos serviços?"
```

Veja a pasta [examples/](./examples/) para exemplos completos de conversas.

### Contribuindo

Contribuições são bem-vindas — novas skills, lógica de decisão aprimorada e referências atualizadas. Veja [CONTRIBUTING.md](./CONTRIBUTING.md) para as diretrizes.

---

## License / Licença

[MIT](./LICENSE) © Rodrigo Vazquez
