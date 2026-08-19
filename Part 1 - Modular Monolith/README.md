# Parte 1 — Construção da aplicação monolítica

<br>

### 1. Objetivo

Pretende-se desenvolver uma aplicação de **comércio eletrónico simplificada**, utilizando **Java e Spring Boot**.

Nesta primeira fase, a aplicação deverá ser construída como um **monólito modular**: existirá uma única aplicação, um único processo e uma única base de dados, mas o código deverá ser organizado de forma a manter claramente separadas as diferentes áreas funcionais.

O objetivo não é construir uma loja online completa, mas criar um domínio suficientemente realista para que, numa fase posterior, seja possível transformar progressivamente a aplicação numa arquitetura de microserviços.

Essa transformação permitirá comparar diretamente conceitos como:

**módulo → microserviço**
**chamada interna → chamada remota**
**transação local → transação distribuída**
**base de dados única → database per service**
**eventos internos → mensageria**

<br>

### 2. Tecnologias

Nesta primeira fase deverão ser utilizadas:

* Java
* Spring Boot
* Spring Web
* Spring Data JPA
* Hibernate
* Maven
* uma base de dados relacional, preferencialmente PostgreSQL
* Git

Poderá ser utilizado Postman, Bruno ou ferramenta equivalente para testar a API.

Nesta fase **não deverão ainda ser utilizados** Kafka, RabbitMQ, Docker, Kubernetes, API Gateway, Service Discovery ou outras tecnologias específicas de arquiteturas distribuídas.

A ideia é começar deliberadamente simples.

<br>

### 3. Domínio da aplicação

A aplicação será constituída inicialmente por quatro áreas principais:

#### Customers

Representa os clientes da loja.

Cada cliente deverá possuir, pelo menos:

* identificador
* nome
* email
* data de registo

O email deverá ser único.

<br>

#### Products

Representa os produtos disponíveis para venda.

Cada produto deverá possuir:

* identificador
* nome
* descrição
* preço
* quantidade disponível em stock
* estado (ativo/inativo)

Um produto inativo não poderá ser utilizado em novas encomendas.

<br>

#### Orders

Representa as encomendas realizadas pelos clientes.

Cada encomenda deverá possuir:

* identificador
* cliente
* data de criação
* estado
* conjunto de itens
* valor total

Cada item deverá indicar:

* produto
* quantidade
* preço unitário no momento da compra

O preço deverá ficar guardado na própria encomenda. Uma alteração posterior do preço do produto não deverá alterar encomendas antigas.

Estados possíveis da encomenda:

`CREATED`
`PAID`
`CANCELLED`

<br>

#### Payments

Representa o pagamento de uma encomenda.

Um pagamento deverá possuir:

* identificador
* encomenda
* valor
* data
* estado

Estados possíveis:

`PENDING`
`APPROVED`
`REJECTED`

Nesta fase não será utilizado nenhum sistema de pagamentos real. O processamento deverá ser simulado pela própria aplicação.

<br>

### 4. Funcionalidades

A aplicação deverá disponibilizar uma **REST API**.

#### Clientes

Deverá ser possível:

* criar um cliente;
* consultar um cliente;
* listar clientes;
* atualizar os seus dados;
* eliminar um cliente quando as regras do domínio o permitirem.

<br>

#### Produtos

Deverá ser possível:

* criar produtos;
* consultar produtos;
* listar produtos;
* alterar produtos;
* ativar/desativar produtos;
* atualizar o stock.

<br>

#### Encomendas

Deverá ser possível criar uma encomenda para determinado cliente.

Ao criar uma encomenda, a aplicação deverá verificar:

1. se o cliente existe;
2. se cada produto existe;
3. se o produto está ativo;
4. se existe stock suficiente;
5. qual é o preço atual de cada produto.

Se todas as condições forem satisfeitas, deverá ser criada a encomenda e atualizado o stock.

O valor total deverá ser calculado pela aplicação.

Por exemplo:

```text
Produto A
Preço: 10 €
Quantidade: 3

Produto B
Preço: 25 €
Quantidade: 2

Total:
3 × 10 + 2 × 25 = 80 €
```

<br>

### 5. Pagamentos

Deverá existir uma operação para pagar uma encomenda.

Por exemplo:

```text
POST /orders/{orderId}/payment
```

O sistema deverá simular o processamento do pagamento.

Um pagamento aprovado deverá provocar a alteração:

```text
Order

CREATED → PAID
```

Um pagamento rejeitado deverá manter a encomenda por pagar.

Não é necessário nesta fase criar regras complexas para decidir se um pagamento é aprovado ou rejeitado. Basta implementar uma solução simples que permita testar os dois cenários.

<br>

### 6. Cancelamento

Uma encomenda ainda não concluída deverá poder ser cancelada.

O cancelamento deverá provocar a reposição no stock das quantidades anteriormente reservadas pela encomenda.

Por exemplo:

```text
Stock inicial: 10

Compra:
3 unidades

Stock:
7

Cancelamento da encomenda

Stock:
10
```

Esta funcionalidade será particularmente interessante mais tarde, quando `Orders` e `Products` deixarem de estar dentro da mesma aplicação.

No monólito, esta operação poderá ser executada através de **uma única transação de base de dados**.

Quando transformarmos a aplicação em microserviços, isso deixará de ser possível da mesma forma.

<br>

### 7. Organização do código

Apesar de ser um monólito, **não organizar a aplicação apenas por camadas globais**, desta forma:

```text
controller/
service/
repository/
entity/
```

Preferir uma estrutura orientada pelo domínio:

```text
com.example.store

├── customer
│   ├── controller
│   ├── service
│   ├── repository
│   ├── model
│   └── dto
│
├── product
│   ├── controller
│   ├── service
│   ├── repository
│   ├── model
│   └── dto
│
├── order
│   ├── controller
│   ├── service
│   ├── repository
│   ├── model
│   └── dto
│
└── payment
    ├── controller
    ├── service
    ├── repository
    ├── model
    └── dto
```

Isto é importante porque estas fronteiras serão aproveitadas posteriormente.

Por exemplo, aquilo que inicialmente é:

```text
store.order
```

poderá acabar transformado em:

```text
order-service
```

<br>

### 8. Regras técnicas

Os controllers não deverão conter lógica de negócio.

A estrutura deverá seguir aproximadamente:

```text
Controller
     ↓
Service
     ↓
Repository
     ↓
Database
```

Deverão ainda existir DTOs próprios para entrada e saída da REST API, evitando expor diretamente as entidades JPA.

A aplicação deverá possuir tratamento adequado de erros, incluindo situações como:

```text
404 Not Found
```

Cliente, produto ou encomenda inexistentes.

```text
400 Bad Request
```

Dados inválidos.

```text
409 Conflict
```

Operações incompatíveis com o estado atual do recurso, como tentar comprar uma quantidade superior ao stock disponível.

<br>

### 9. Transações

Deverá ser utilizada gestão transacional através do Spring sempre que uma operação alterar vários elementos do domínio.

Por exemplo, criar uma encomenda envolve:

```text
criar Order
+
criar OrderItems
+
reduzir stock
```

Estas operações deverão constituir uma unidade lógica.

Ou todas são realizadas ou nenhuma é realizada.

Esta parte é particularmente importante para o projeto porque posteriormente veremos **o que acontece a esta garantia quando Products e Orders passam a pertencer a microserviços diferentes**.

<br>

### 10. Testes

O projeto deverá possuir testes automatizados.

Não é necessário procurar cobertura de 100%.

O objetivo é testar sobretudo:

* regras de negócio;
* criação de encomendas;
* stock insuficiente;
* produtos inativos;
* cálculo do total;
* cancelamento;
* pagamentos;
* alterações de estado inválidas.

Deverão existir alguns testes de integração que exercitem a aplicação com a base de dados.

<br>

### 11. Resultado esperado

No final desta primeira parte deverás possuir algo conceptualmente semelhante a:

```text
                    ┌──────────────────────┐
                    │     REST Clients     │
                    └──────────┬───────────┘
                               │
                               ▼
                  ┌────────────────────────┐
                  │   Spring Boot Store    │
                  │                        │
                  │  Customer              │
                  │  Product               │
                  │  Order                 │
                  │  Payment               │
                  │                        │
                  └───────────┬────────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │   PostgreSQL    │
                     └─────────────────┘
```

É deliberadamente uma arquitetura simples.

Mas o código deverá possuir fronteiras suficientemente claras para permitir que, na fase seguinte, comecemos a desmontar este monólito.

<br>

### Objetivo pedagógico da Parte 1

Esta primeira fase não aborda ainda a implementação de uma arquitetura de microserviços. O seu objetivo consiste em estabelecer uma aplicação monolítica de referência, que servirá de base às fases seguintes do projeto, nas quais a arquitetura será progressivamente decomposta em serviços independentes.

Nas fases seguintes, a aplicação será progressivamente transformada:

```text
MONÓLITO
   ↓
Monólito modular
   ↓
Primeiro serviço extraído
   ↓
Comunicação entre serviços
   ↓
Bases de dados independentes
   ↓
Service Discovery
   ↓
API Gateway
   ↓
Resiliência
   ↓
Mensageria
   ↓
Consistência distribuída
   ↓
Observabilidade
   ↓
Segurança
   ↓
Docker
   ↓
Kubernetes
   ↓
CI/CD
```

O mais interessante será não acrescentar estas tecnologias simplesmente porque “fazem parte de microserviços”. Cada uma deverá surgir **à medida que surjam requisitos arquiteturais que justifiquem a sua introdução**.

Tecnologias como Kafka, Circuit Breakers, API Gateway, Saga, tracing distribuído e Kubernetes serão introduzidas progressivamente, de acordo com as necessidades arquiteturais identificadas em cada fase do projeto.

<br>
