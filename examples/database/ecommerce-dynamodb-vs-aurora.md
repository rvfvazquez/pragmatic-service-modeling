# Example: E-commerce product catalog — DynamoDB vs Aurora DSQL

**Skill:** `pragmatic-db-expert`
**Scenario:** A mid-size e-commerce platform is building a new product catalog service. The team is debating between DynamoDB and Aurora DSQL. They need to handle high read traffic and occasional complex queries for search and filtering.

---

## Conversation

**User:**
> Estou modelando o serviço de catálogo de produtos do nosso e-commerce. Preciso decidir entre DynamoDB e Aurora DSQL. Temos alto volume de leitura e às vezes precisamos fazer queries mais complexas por categoria, faixa de preço e disponibilidade.

---

**Claude (pragmatic-db-expert):**

Boa questão — catálogo de produtos é um dos casos mais interessantes de modelagem. Vamos explorar o seu contexto.

**Seus dados são estruturados e uniformes** (todos os produtos têm os mesmos campos) **ou semi-estruturados** (produtos de categorias diferentes têm atributos muito distintos — eletrônicos têm voltagem, roupas têm tamanho/cor)?

---

**User:**
> Semi-estruturado. Temos eletrônicos, moda, alimentos e cada categoria tem atributos muito diferentes. Um produto de eletrônico tem specs técnicas que um vestido não tem.

---

**Claude:**

Faz sentido. Agora sobre os padrões de acesso — você mencionou queries por categoria, faixa de preço e disponibilidade:

**Essas queries são sempre por combinações conhecidas** (ex: categoria + filtro de preço) **ou vocês precisam de queries ad-hoc, dinâmicas**, como um usuário montando filtros livremente na interface? E com que frequência esse catálogo é atualizado — muitas vezes por dia ou principalmente lido?

---

**User:**
> As queries são relativamente previsíveis — a gente filtra por categoria, subcategoria, faixa de preço e às vezes por atributos específicos da categoria. Mas o usuário pode combinar esses filtros livremente no frontend. O catálogo é read-heavy, atualizações acontecem algumas vezes por hora no máximo.

---

**Claude:**

Entendido. Uma questão sobre escala e arquitetura:

**Qual o volume esperado de leituras no pico?** E sua stack é serverless (Lambda, Fargate) ou vocês têm servidores/containers de longa duração? Pergunto porque conexões persistentes de banco importam muito nessa decisão.

---

**User:**
> No pico (Black Friday) esperamos até 50 mil requisições por minuto no catálogo. Nossa stack é toda Lambda + Fargate. Temos aversão a gerenciar conexões de banco.

---

**Claude:**

Com 50k req/min de Lambda/Fargate, esse detalhe muda bastante a equação. Última pergunta:

**ACID e consistência forte são necessários para leitura do catálogo?** Ou leitura eventual — onde um produto recém-atualizado pode demorar alguns millisegundos para aparecer para todos os usuários — é aceitável?

---

**User:**
> Eventual consistency é aceitável para leitura do catálogo. Não precisamos de transações no serviço de catálogo.

---

**Claude:**

Tenho contexto suficiente para uma recomendação sólida.

---

### Recommendation: DynamoDB (primary) + OpenSearch (for complex filtering)

**Recomendação primária: DynamoDB**

DynamoDB é o banco certo para o seu catálogo pelas razões conectadas ao que você me disse:

- **50k req/min de ambientes serverless** — DynamoDB foi projetado para isso. Sem pool de conexões para gerenciar, sem RDS Proxy necessário. Cada invocação Lambda faz uma chamada HTTP para a API do DynamoDB e pronto. Aurora DSQL, apesar de serverless, ainda usa conexões TCP e começa a ter problemas de connection pooling nessa escala sem RDS Proxy.
- **Dados semi-estruturados com atributos por categoria** — DynamoDB armazena itens como documentos JSON, então eletrônicos podem ter specs técnicas que roupas não têm, sem precisar de colunas nulas em uma tabela relacional.
- **Read-heavy + eventual consistency aceitável** — DynamoDB com leituras eventually consistent custa metade do que strongly consistent e tem latência de single-digit milliseconds. Para catálogo de produtos, isso é exatamente o que você quer.

**Tradeoff principal:**

Filtros combinatórios livres (categoria + faixa de preço + atributos específicos) **não são o ponto forte do DynamoDB**. Queries ad-hoc multi-atributo exigem GSIs (Global Secondary Indexes) bem planejados ou scan completo da tabela — o que não escala bem para 50k req/min de leituras de catálogo filtradas.

**Por isso, a arquitetura recomendada é uma combinação:**

---

### Architecture pattern: DynamoDB + OpenSearch

```
[Product Updates] → DynamoDB (source of truth)
                        ↓
               DynamoDB Streams
                        ↓
                AWS Lambda (indexer)
                        ↓
              Amazon OpenSearch Service
                        ↑
              [Search/Filter queries from frontend]
```

- **DynamoDB** é o source of truth do catálogo — leituras por `productId` (página de detalhe do produto, carrinho) são instantâneas e infinitamente escaláveis
- **OpenSearch** recebe os produtos via DynamoDB Streams e Lambda indexer, e responde às queries filtradas (categoria + faixa de preço + atributos livres) com baixa latência e suporte a full-text search
- Dessa forma, você não força DynamoDB a fazer o que ele não é bom, e não perde o que ele faz melhor

**Sobre Aurora DSQL:** seria válido se você precisasse de ACID, JOINs entre entidades (produto + estoque + promoção na mesma query), ou se a stack não fosse serverless. Para o seu caso, a latência e o modelo de conexão do Aurora são desnecessariamente complexos.

---

**Próximos passos:**

1. **Defina sua estratégia de partition key** para DynamoDB — recomendo `categoryId` como PK e `productId` como SK para queries por categoria, com GSI em `productId` para leituras diretas por ID
2. **Configure DynamoDB Streams + Lambda indexer** para manter o OpenSearch sincronizado — evite chamadas diretas ao OpenSearch no path de escrita para não acoplar os sistemas

---

*Skill: pragmatic-db-expert | Services evaluated: DynamoDB, Aurora DSQL, DocumentDB, OpenSearch*
