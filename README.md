# Normalização de Banco de Dados — PostgreSQL

Projeto acadêmico que apresenta os conceitos de normalização relacional por meio de um cenário de vendas. O objetivo é demonstrar como organizar dados de clientes, pedidos e produtos para reduzir redundâncias, evitar anomalias e preservar a integridade das informações.

## Objetivos

- Compreender as formas normais 1FN, 2FN e 3FN.
- Conhecer, em nível introdutório, a BCNF, a 4FN e a 5FN.
- Identificar anomalias de inserção, atualização e exclusão.
- Aplicar chaves primárias, chaves estrangeiras e restrições de integridade.
- Executar um exemplo prático em PostgreSQL.

## Conteúdo

1. [O problema da tabela única](#1-o-problema-da-tabela-única)
2. [Primeira Forma Normal (1FN)](#2-primeira-forma-normal-1fn)
3. [Segunda Forma Normal (2FN)](#3-segunda-forma-normal-2fn)
4. [Terceira Forma Normal (3FN)](#4-terceira-forma-normal-3fn)
5. [Formas normais avançadas](#5-formas-normais-avançadas)
6. [Modelo relacional proposto](#6-modelo-relacional-proposto)
7. [Como executar o SQL](#7-como-executar-o-sql)
8. [Consultas para validar os dados](#8-consultas-para-validar-os-dados)
9. [Limitações e decisões de modelagem](#9-limitações-e-decisões-de-modelagem)
10. [Referências](#10-referências)

## 1. O problema da tabela única

Armazenar clientes, pedidos e produtos em uma única tabela pode repetir os dados do cliente e as informações dos produtos. Essa redundância favorece três tipos de anomalia:

- **Atualização:** um telefone precisa ser alterado em várias linhas e alguma ocorrência pode ficar desatualizada.
- **Inserção:** pode ser difícil cadastrar um produto antes de ele participar de um pedido.
- **Exclusão:** apagar o último pedido de um produto pode eliminar, por acidente, a única cópia dos dados desse produto.

A normalização organiza os dados em relações coerentes, com dependências e restrições explicitadas.

## 2. Primeira Forma Normal (1FN)

A 1FN requer que os atributos contenham valores atômicos de acordo com o domínio definido e que não existam grupos repetitivos representados em uma mesma célula. Por exemplo, se o sistema precisa tratar dois telefones de um cliente separadamente, eles não devem ser armazenados como uma lista de telefones em um único campo.

Atomicidade é relativa ao modelo e às operações necessárias: um telefone pode ser tratado como um valor único em um sistema e ser dividido em código de área e número em outro.

## 3. Segunda Forma Normal (2FN)

Uma relação está na 2FN quando está na 1FN e nenhum atributo não primo depende parcialmente de uma chave candidata. A situação é especialmente relevante quando existe uma chave composta.

Considere uma tabela de itens identificada pela combinação `(pedido_id, produto_id)`. A quantidade comprada depende da combinação do pedido com o produto, enquanto o nome do produto depende somente de `produto_id`. Manter o nome do produto nessa tabela causa uma dependência parcial; por isso, os dados do produto ficam na tabela `produtos`.

O preço atual do catálogo pertence a `produtos`. Já o preço efetivamente cobrado em uma compra pode ser armazenado em `itens_pedido`, para preservar o histórico da venda.

## 4. Terceira Forma Normal (3FN)

Em termos intuitivos, a 3FN procura eliminar dependências transitivas de atributos não-chave em relação às chaves. Formalmente, para cada dependência funcional não trivial `X → A`, pelo menos uma das condições deve valer: `X` é uma superchave ou `A` é um atributo primo (pertence a alguma chave candidata).

Por exemplo, se a regra de negócio garante que um CEP determina uma localidade, guardar a cidade repetidamente em uma tabela de clientes pode introduzir uma dependência transitiva. Uma tabela separada de localidades pode representar essa regra. A decomposição deve refletir as dependências reais do domínio; não se deve presumir que todo CEP, em qualquer modelo, identifica universalmente uma única cidade.

## 5. Formas normais avançadas

- **BCNF (Boyce–Codd):** para toda dependência funcional não trivial `X → Y`, `X` deve ser uma superchave. É mais restritiva que a 3FN.
- **4FN:** trata dependências multivaloradas não triviais que não são determinadas por uma superchave. É útil quando dois conjuntos de valores independentes estão sendo combinados indevidamente.
- **5FN:** trata dependências de junção não implicadas pelas chaves candidatas, quando uma decomposição em relações menores é necessária para evitar redundância e pode ser recomposta sem perda.

Esses tópicos são apresentados como visão geral. Aplicações formais devem ser acompanhadas de exemplos específicos de dependências e decomposição.

## 6. Modelo relacional proposto

O modelo de exemplo utiliza quatro tabelas:

- `clientes`: dados cadastrais do cliente.
- `pedidos`: cabeçalho do pedido e referência ao cliente.
- `produtos`: cadastro e preço atual de catálogo.
- `itens_pedido`: entidade associativa entre pedidos e produtos, com quantidade e preço praticado na venda.

Relacionamentos:

- Um cliente pode realizar vários pedidos; cada pedido pertence a um cliente (1:N).
- Um pedido pode conter vários itens; cada item pertence a um pedido (1:N).
- Um produto pode aparecer em vários itens; cada item referencia um produto (N:1).
- Assim, pedidos e produtos possuem relacionamento N:N resolvido por `itens_pedido`.

### Script SQL (PostgreSQL)

```sql
CREATE TABLE clientes (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    telefone VARCHAR(20)
);

CREATE TABLE produtos (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome_produto VARCHAR(100) NOT NULL,
    valor_unitario NUMERIC(10, 2) NOT NULL,
    CONSTRAINT ck_produto_valor_nao_negativo CHECK (valor_unitario >= 0)
);

CREATE TABLE pedidos (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cliente_id BIGINT NOT NULL,
    data_pedido TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_pedido_cliente
        FOREIGN KEY (cliente_id) REFERENCES clientes(id) ON DELETE RESTRICT
);

CREATE TABLE itens_pedido (
    pedido_id BIGINT NOT NULL,
    produto_id BIGINT NOT NULL,
    quantidade INTEGER NOT NULL,
    preco_unitario_venda NUMERIC(10, 2) NOT NULL,
    CONSTRAINT pk_itens_pedido PRIMARY KEY (pedido_id, produto_id),
    CONSTRAINT fk_item_pedido
        FOREIGN KEY (pedido_id) REFERENCES pedidos(id) ON DELETE RESTRICT,
    CONSTRAINT fk_item_produto
        FOREIGN KEY (produto_id) REFERENCES produtos(id) ON DELETE RESTRICT,
    CONSTRAINT ck_item_quantidade_positiva CHECK (quantidade > 0),
    CONSTRAINT ck_item_preco_nao_negativo CHECK (preco_unitario_venda >= 0)
);

INSERT INTO clientes (nome, telefone)
VALUES ('Carlos Silva', '(11) 98888-1111');

INSERT INTO produtos (nome_produto, valor_unitario)
VALUES ('Borracha', 6.00), ('Caneta', 3.00);

INSERT INTO pedidos (cliente_id) VALUES (1);

INSERT INTO itens_pedido (pedido_id, produto_id, quantidade, preco_unitario_venda)
VALUES (1, 1, 2, 6.00), (1, 2, 3, 3.00);
```

**Observação:** o exemplo pressupõe um único registro de cada produto por pedido. Se o sistema precisar permitir linhas separadas para o mesmo produto no mesmo pedido, adicione um `id_item` como chave primária e mantenha `pedido_id` e `produto_id` como chaves estrangeiras, removendo a chave primária composta.

## 7. Como executar o SQL

### Requisitos

- PostgreSQL instalado (versão 15 ou superior é adequada para este exemplo).
- Um cliente SQL, como pgAdmin 4 ou `psql`.

### Passos

1. Crie um banco de dados para o projeto, por exemplo `normalizacao_vendas`.
2. Abra o Query Tool no pgAdmin ou conecte-se ao banco pelo `psql`.
3. Execute o script da seção anterior na ordem apresentada.
4. Se executar novamente, use um banco vazio ou remova as tabelas na ordem inversa das dependências. Os comandos de criação não são idempotentes.
5. Execute as consultas de validação a seguir.

## 8. Consultas para validar os dados

### Listar pedidos, clientes e itens

```sql
SELECT
    p.id AS pedido_id,
    c.nome AS cliente,
    pr.nome_produto AS produto,
    ip.quantidade,
    ip.preco_unitario_venda,
    ip.quantidade * ip.preco_unitario_venda AS subtotal
FROM pedidos AS p
JOIN clientes AS c ON c.id = p.cliente_id
JOIN itens_pedido AS ip ON ip.pedido_id = p.id
JOIN produtos AS pr ON pr.id = ip.produto_id
ORDER BY p.id, pr.nome_produto;
```

### Calcular o total de cada pedido

```sql
SELECT
    pedido_id,
    SUM(quantidade * preco_unitario_venda) AS total_pedido
FROM itens_pedido
GROUP BY pedido_id
ORDER BY pedido_id;
```

### Testar restrições

Os comandos abaixo devem falhar devido às restrições `CHECK`:

```sql
-- Quantidade inválida: deve ser maior que zero.
INSERT INTO itens_pedido (pedido_id, produto_id, quantidade, preco_unitario_venda)
VALUES (1, 1, 0, 6.00);

-- Preço de venda negativo: não permitido.
INSERT INTO itens_pedido (pedido_id, produto_id, quantidade, preco_unitario_venda)
VALUES (1, 1, 1, -1.00);
```

Execute cada teste separadamente. Dependendo do cliente SQL, uma instrução com erro pode interromper a transação atual; faça `ROLLBACK` se necessário antes de continuar.

## 9. Limitações e decisões de modelagem

- **Normalização e desempenho:** normalizar reduz redundâncias e anomalias, mas consultas podem exigir `JOIN`s. Em sistemas analíticos, desnormalização controlada pode ser apropriada após medir o desempenho e documentar os trade-offs.
- **Exclusão de registros:** o exemplo usa `ON DELETE RESTRICT` para preservar a integridade do histórico. Sistemas reais podem precisar de cancelamento lógico ou regras adicionais de auditoria.
- **Telefone:** é opcional no exemplo e aceita apenas um telefone por cliente. Se houver vários telefones, crie uma tabela `telefones_cliente`.
- **Preço histórico:** `preco_unitario_venda` registra o preço cobrado no momento da compra; não deve ser substituído automaticamente quando o preço atual do catálogo mudar.
- **Identificação dos exemplos:** os IDs inseridos no exemplo são previsíveis apenas em um banco recém-criado. Em dados preexistentes, consulte os IDs gerados em vez de presumir que serão 1.
- **Escopo acadêmico:** o modelo é didático e não inclui pagamentos, estoque, impostos, entregas, devoluções ou controle de usuários.

## 10. Referências

Consulte e registre a edição utilizada na entrega acadêmica. Referências recomendadas para aprofundar os conceitos:

- E. F. Codd. “A Relational Model of Data for Large Shared Data Banks.” *Communications of the ACM*, 1970. DOI: https://doi.org/10.1145/362384.362685
- Abraham Silberschatz, Henry F. Korth e S. Sudarshan. *Database System Concepts*. McGraw-Hill. Consulte a edição disponível na biblioteca.
- Ramez Elmasri e Shamkant B. Navathe. *Fundamentals of Database Systems*. Pearson. Consulte a edição disponível na biblioteca.
- PostgreSQL Documentation — Constraints: https://www.postgresql.org/docs/current/ddl-constraints.html
- PostgreSQL Documentation — Identity Columns: https://www.postgresql.org/docs/current/ddl-identity-columns.html

As referências de livros devem ser completadas com a edição, o ano e os demais dados bibliográficos da versão efetivamente consultada. Não inclua fontes que não tenha verificado.

---

## Autoria e contribuição

Projeto desenvolvido para fins educacionais. Adicione aqui os nomes dos autores, a disciplina, a instituição e a data de entrega, conforme as orientações do professor.
