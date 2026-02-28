# Contributing to pragmatic-service-modeling

Thank you for your interest in contributing! / Obrigado pelo seu interesse em contribuir!

---

## 🇺🇸 English

### Ways to contribute

- **New skills** — Add an expert for a new AWS service category (e.g., storage, networking, security)
- **Improved decision logic** — Refine the interview questions or recommendation heuristics in an existing `SKILL.md`
- **Updated references** — Keep the `references/` documentation in sync with AWS service updates
- **Bug reports** — Report incorrect recommendations or confusing interview flows
- **Examples** — Add real-world conversation examples to the `examples/` folder

### Skill structure

Every skill must follow this structure:

```
skills/
└── your-skill-name/
    ├── SKILL.md              ← Required: skill prompt with frontmatter
    └── references/
        └── your-reference.md ← Required: detailed decision reference
```

#### SKILL.md format

```markdown
---
name: your-skill-name
description: One paragraph describing when this skill should be triggered. Include both English and Portuguese trigger phrases.
---

# Your Skill Title

Brief description of the skill's role and goal.

## How to conduct the interview

Instructions for how Claude should ask questions — conversationally, grouped, adaptive.

## Questions to ask (adapt and reorder as needed)

**1. Topic**
- Question 1?
- Question 2?

(4–6 topic groups)

## Decision logic

Reference to the `references/` file and key heuristics as bullet points.

## Recommendation format

Structured format Claude should use for the final recommendation.
```

#### References file

The `references/` file is the decision brain of the skill. It should include:

- **Service profiles** — Execution model, billing, scaling, limits, when it wins, when it struggles
- **Decision matrix** — A comparison table across key criteria
- **Architecture combination patterns** — Common multi-service patterns with use cases

### Development workflow

1. Fork this repository
2. Create a branch: `git checkout -b skill/your-skill-name` or `fix/description`
3. Follow the skill structure above
4. Test your skill in Claude (Cowork or Claude Code) with at least 3 different conversation scenarios
5. Add at least one example to `examples/your-category/`
6. Open a pull request with the PR template filled out

### Quality checklist

Before opening a PR, make sure:

- [ ] `SKILL.md` has valid frontmatter with `name` and `description`
- [ ] The description includes trigger phrases in both English and Portuguese
- [ ] The interview section has 4–6 question groups
- [ ] The decision logic references the `references/` file
- [ ] The recommendation format is clearly defined
- [ ] The references file covers: service profiles, a decision matrix, and combination patterns
- [ ] At least one example conversation is included in `examples/`
- [ ] All markdown files are well-formatted and free of typos

### Updating existing skills

When AWS releases a new service or updates pricing/limits:

1. Update the relevant `references/` file with accurate information
2. Adjust heuristics in `SKILL.md` if the new info changes decision logic
3. Note the update in your PR description with a link to the AWS announcement

### Opening issues

Use the issue templates:
- **Bug report** — For incorrect, misleading, or confusing skill behavior
- **Skill proposal** — To propose a new skill with a brief description of the decision space

---

## 🇧🇷 Português

### Formas de contribuir

- **Novas skills** — Adicione um expert para uma nova categoria de serviço AWS (ex: storage, networking, segurança)
- **Lógica de decisão aprimorada** — Refine as perguntas da entrevista ou as heurísticas de recomendação em um `SKILL.md` existente
- **Referências atualizadas** — Mantenha a documentação em `references/` sincronizada com atualizações dos serviços AWS
- **Bug reports** — Reporte recomendações incorretas ou fluxos de entrevista confusos
- **Exemplos** — Adicione exemplos de conversas reais na pasta `examples/`

### Estrutura de uma skill

Toda skill deve seguir esta estrutura:

```
skills/
└── nome-da-sua-skill/
    ├── SKILL.md              ← Obrigatório: prompt da skill com frontmatter
    └── references/
        └── sua-referencia.md ← Obrigatório: referência detalhada de decisão
```

#### Formato do SKILL.md

```markdown
---
name: nome-da-sua-skill
description: Um parágrafo descrevendo quando esta skill deve ser acionada. Inclua frases-gatilho em inglês e em português.
---

# Título da Sua Skill

Breve descrição do papel e objetivo da skill.

## How to conduct the interview

Instruções de como Claude deve fazer as perguntas — de forma conversacional, agrupada, adaptativa.

## Questions to ask (adapt and reorder as needed)

**1. Tópico**
- Pergunta 1?
- Pergunta 2?

(4–6 grupos de tópicos)

## Decision logic

Referência ao arquivo `references/` e heurísticas-chave como bullets.

## Recommendation format

Formato estruturado que Claude deve usar na recomendação final.
```

### Fluxo de desenvolvimento

1. Faça um fork deste repositório
2. Crie uma branch: `git checkout -b skill/nome-da-sua-skill` ou `fix/descricao`
3. Siga a estrutura de skill acima
4. Teste sua skill no Claude (Cowork ou Claude Code) com pelo menos 3 cenários de conversa diferentes
5. Adicione pelo menos um exemplo em `examples/sua-categoria/`
6. Abra um pull request com o template de PR preenchido

### Checklist de qualidade

Antes de abrir um PR, verifique:

- [ ] `SKILL.md` tem frontmatter válido com `name` e `description`
- [ ] A description inclui frases-gatilho em inglês e português
- [ ] A seção de entrevista tem 4–6 grupos de perguntas
- [ ] A lógica de decisão referencia o arquivo `references/`
- [ ] O formato de recomendação está claramente definido
- [ ] O arquivo de referências cobre: perfis de serviço, uma matriz de decisão e padrões de combinação
- [ ] Pelo menos um exemplo de conversa está incluído em `examples/`
- [ ] Todos os arquivos markdown estão bem formatados e sem erros de digitação

---

## Code of Conduct

Be respectful and constructive. This is a technical project focused on helping people make better architecture decisions — keep discussions focused on that goal.
