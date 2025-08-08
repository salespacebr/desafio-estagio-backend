# Desafio de Estágio Backend Salespace

O desafio consiste em criar uma **API Rest** que calcula o **valor total** de um pedido a partir de uma lista de itens (produto + quantidade) e aplica **regras de desconto progressivas**. A API deve retornar o **total final** do pedido e o **detalhamento dos descontos aplicados**.

## Requisitos funcionais

1. A **entrada** deverá ser uma lista de itens com `productId` e `quantity`.

2. O **cálculo base** do subtotal do pedido será `subtotal = Σ(preço_unitário * quantidade)`.

3. A aplicação deverá aplicar os seguintes descontos de forma progressiva:

   - **Regra 1 – Volume de itens**:

     - ≥ **10** itens no pedido (soma das quantidades) ⇒ **10%**
     - ≥ **20** itens ⇒ **15%**
     - ≥ **50** itens ⇒ **20%**
     - **Atenção**: faixas **mutuamente exclusivas**; aplica-se a **maior**.

   - **Regra 2 – Valor do carrinho**:

     - subtotal ≥ **1.000** ⇒ **R\$ 50,00** de desconto **fixo**
     - subtotal ≥ **2.000** ⇒ **R\$ 150,00** fixo
     - **Atenção**: faixas **mutuamente exclusivas**; aplica-se a **maior**.

   - **Regra 3 – Categoria** (opcional):
     - Se **> 5** unidades de produtos da categoria `"acessorios"` ⇒ **5%** **apenas** sobre os itens dessa categoria (desconto por linha).
     - Esta regra é considerada uma implementação bônus e pode ser desconsiderada dos demais enunciados caso você decida por não implementá-la.

4. A **composição dos descontos** deverá ocorrer da seguinte forma:

   - Os descontos devem ser aplicados na seguinte ordem, quando cabíveis: **Regra 3 (por item) → Regra 1 (percentual sobre o total remanescente) → Regra 2 (valor fixo)**.

   - O retorno deve listar **todas** as regras aplicadas, base de cálculo, valor do desconto e justificativa (ex: “20 itens ⇒ faixa 15%”).

5. A **saída** deverá conter:

   - `items`: com preço unitário, subtotal por item, descontos por item (quando houver), e total por item.

   - `discounts`: array com objetos `{code, name, basis, amount, metadata}`.

     > `basis` representa o valor original sobre o qual o desconto será aplicado. `amount` representa o valor final, em reais, do desconto aplicado. `metadata` inclui uma justificativa da aplicação do desconto, no formato de sua escolha.

   - `total`: total final do pedido após **todos** os descontos aplicados.

## Requisitos não-funcionais

- **Linguagem/Stack**: Deve ser utilizado Typescript + NodeJS

- **Entrega**: Repositório com o projeto + instruções para execução ou `Dockerfile + docker-compose`

- **Documentação**: `README.md` com instruções de como rodar o projeto e exemplos de requests/responses.

- **Observabilidade**: logs ao longo do cálculo, permitindo um mínimo nível de _debug_.

## Domínio e dados

Você pode gerar uma lista de produtos e mantê-los como JSON em memória, sem a necessidade de estruturar um banco de dados. Lembre-se de definir um preço base para cada produto, já que a entrada não espera receber o preço.

## Endpoints

### `POST /v1/orders`

Calcula o preço final de um pedido. O _request_ deverá seguir **exatamente** o formato abaixo, enquanto a _response_ pode ser modificada de acordo com sua preferência, mas preservando os campos `items`, `discounts` e `total`.

#### Request

```json
{
  "items": [
    { "productId": "sku-001", "quantity": 12 },
    { "productId": "sku-002", "quantity": 6 },
    { "productId": "sku-003", "quantity": 1 }
  ]
}
```

#### Response

```json
{
  "currency": "BRL",
  "items": [
    {
      "productId": "sku-001",
      "unitPrice": 79.9,
      "quantity": 12,
      "subtotal": 958.8,
      "itemDiscounts": [],
      "total": 958.8
    },
    {
      "productId": "sku-002",
      "unitPrice": 39.9,
      "quantity": 6,
      "subtotal": 239.4,
      "itemDiscounts": [
        {
          "code": "CAT_ACC_5PCT",
          "name": "Categoria acessórios 5%",
          "basis": 239.4,
          "amount": 11.97,
          "metadata": {
            "category": "acessorios",
            "threshold": 5
          }
        }
      ],
      "total": 227.43
    },
    {
      "productId": "sku-003",
      "unitPrice": 19.9,
      "quantity": 1,
      "subtotal": 19.9,
      "itemDiscounts": [
        {
          "code": "CAT_ACC_5PCT",
          "name": "Categoria acessórios 5%",
          "basis": 19.9,
          "amount": 99,
          "metadata": {
            "category": "acessorios",
            "threshold": 5
          }
        }
      ],
      "total": 18.91
    }
  ],
  "discounts": [
    {
      "code": "QTY_TIER_15PCT",
      "name": "Desconto por volume 15%",
      "basis": 1205.14,
      "amount": 180.77,
      "metadata": {
        "totalItems": 19,
        "tier": ">=20? false, >=10? true"
      }
    },
    {
      "code": "CART_VALUE_FIXED_50",
      "name": "Desconto por valor do carrinho",
      "basis": 1024.37,
      "amount": 5000,
      "metadata": {
        "threshold": 100000
      }
    }
  ],
  "total": 974.37
}
```

> Exemplo ilustrativo – os números apresentados não necessariamente são consistentes e servem apenas para mostrar o formato da resposta.

**Importante**: Lembre-se de garantir consistência e tratar/explicar arredondamentos.

## Regras de negócio detalhadas

- Os descontos devem ser aplicados na seguinte ordem, quando cabíveis: **Regra 3 (por item) → Regra 1 (percentual sobre o total remanescente) → Regra 2 (valor fixo)**.

- Os descontos devem ser **progressivo**: dentro de cada grupo (quantidade/valor), apenas a **maior faixa** da regra entra.

- Deve ser possível compor descontos (_stackability_): regras de **tipos diferentes** se acumulam; dentro de um **mesmo tipo** não.

- Cada desconto deve trazer **explicitamente** a base (valor original) sobre a qual foi calculado.

- Os totais devem ser fiéis à moeda brasileira, atentando-se sempre às casas decimais.

## Casos de testes

Você deverá testar sua aplicação para diferentes casos, como por exemplo (mas não somente):

1. Os produtos enviados não geram direito a desconto.

2. Os produtos enviados geram direito a somente um desconto.

3. Os produtos enviados geram direito a vários descontos combinados.

A avaliação irá levar em conta casos mais específicos para entender como a aplicação se comporta em cenários mais complexos quando há diferentes descontos sendo combinados.

## Tratativa de erros (bônus)

Você pode realizar as seguintes implementações e retornar os _status code_ adequados para casos em que a solicitação é inválida:

- **422 (Unprocessable entity)**: payload inválido (schema + regras).
- **404 (Not found)**: produto não encontrado (se buscar no banco).
- **500 (Internal server errror)**: erro inesperado (não vazar `stack trace` no body).

## Arquitetura esperada

Você é livre para escolher a metolodia de desenvolvimento que preferir (arquitetura limpa, hexagonal, microsserviço), desde que o código fique organizado e possibilite uma clara leitura.

Para o cálculo de descontos, é fortemente encorajado que se implemente um `DiscountEngine`, que pode ser uma classe ou função que tem como objetivo centralizar o processamento dos descontos e é chamada pela rota.

## Critérios de avaliação

Serão utilizados os seguintes critérios para analisar a aplicação submetida (com diferentes pesos, mas já listados em ordem de importância):

1. Assertividade do cálculo e suas regras

2. Qualidade do código (coesão, legibilidade, organização)

3. Modularidade (facilidade de inclusão de novas regras, por exemplo)

4. Robustez (validação, logs, consistência, tratativa de erros)

5. Desempenho (número de linhas, complexidade da lógica)

6. Documentação (README, exemplos)

## Entregáveis

Deverá ser entregue um link para um repositório público do GitHub ou GitLab, bem como as instruições para execução da aplicação (`docker compose up`, `make run` ou outros).

Você pode optar por criar um `Dockerfile` ou simplesmente entregar um projeto que seja executável no terminal via `node`.

## Desafio (Bônus)

> **Importante**: A implementação a seguir **não é obrigatória** e representará um bônus de 5% na avaliação.

É comum num ecossistema de vendas B2B, o cliente primeiro realizar uma **cotação**, que apresenta uma validade, para posteriormente finalizar o pedido.

Assim, pensando que pode haver variação de preços de um ou mais produtos em determinado período, o sistema poderia trabalhar com duas rotas:

1. Uma rota de cotação (`POST v1/orders/quote`), que dado os itens e respectivas quantidades, apresenta o valor final (com os descontos) ao cliente.

2. Uma rota de finalizar o pedido (`POST v1/orders`), que efetiva determinada cotação.

Para isso, utilizamos o conceito de uma **chave de idempotência** que é gerada junto à cotação, de modo que o cliente utiliza esta chave para finalizar o pedido – garantindo assim que ele pague o preço apresentado no momento da cotação, não um eventual preço atualizado.

Para isso, será necessário estabelecer alguma forma de **persistência de dados**, visto que a aplicação precisa levar em conta as chaves de idempotência geradas para resgatar preços e cálculos numa eventual finalização – desde que dentro da validade da cotação.

Como exemplo, imagine o seguinte cenário:

1. Cliente faz a cotação de X produtos e Y quantidades (`POST v1/orders/quote`).

2. Sistema devolve os totais, descontos e a chave de idempotência.

3. Ocorre uma atualização dos preços que o cliente consultou.

4. Cliente posteriormente decide finalizar o pedido, enviando a chave de idempotência na requisição (`POST v1/orders`):

   a. Se a cotação realizada está dentro da validade, o sistema permite a finalização, levando em conta o preço original (antes da atualização).

   b. Se a cotação realizada está fora da validade, o sistema retorna um erro e inclui uma nova cotação (incluindo na resposta os totais, descontos e nova chave de idempotência).
