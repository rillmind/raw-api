# raw-api

API REST de produtos em Go, feita com a biblioteca padrão (`net/http`) e PostgreSQL via pgx.

## O que faz

CRUD de produtos (`id`, `name`, `price`, `stock`, `created_at`) servido em `http://localhost:2306`:

| Método | Rota             | Ação                     |
| ------ | ---------------- | ------------------------ |
| POST   | `/products`      | cria um produto          |
| GET    | `/products`      | lista todos              |
| GET    | `/products/{id}` | busca por id             |
| PUT    | `/products/{id}` | atualiza nome/preço/estoque |
| DELETE | `/products/{id}` | remove                   |

O roteamento usa os padrões com método e curinga do `http.ServeMux` (Go 1.22+), sem framework.
O banco sobe num contêiner com `podman-compose`.

## Por que existe

Para estudar como se monta uma API "na mão" em Go, usando primariamente a biblioteca padrão:
roteamento, handlers, repositório com `database/sql` e desligamento gracioso do servidor.

## Instalação

Pré-requisitos: Go 1.26+, Podman e podman-compose.

```sh
git clone https://github.com/rillmind/raw-api.git
cd raw-api
touch .env            # o servidor exige o arquivo, mesmo vazio
podman-compose up -d  # PostgreSQL 16 em localhost:5432
```

Se o Podman reclamar de "short-name resolution", baixe a imagem com o nome completo antes:
`podman pull docker.io/library/postgres:16`.

O repositório ainda não cria a tabela. Crie uma compatível com as consultas de
`src/product/repository.go`:

```sh
podman exec raw_api_db psql -U user -d raw_api_db -c "create table products (
  id bigserial primary key, name text not null, price numeric(10,2) not null,
  stock int not null default 0, created_at timestamptz not null default now());"
```

Por padrão a conexão é `postgres://user:123456@localhost:5432/raw_api_db?sslmode=disable`; para
outra, defina `DATA_SOURCE_NAME` no `.env`. Então suba a API:

```sh
go run ./cmd/server   # ou: make run (sobe o banco, compila e roda em segundo plano)
```

## Como testar

`go vet ./... && go test ./...` compila e verifica o código. Ainda não há casos de teste (o
`store_handler_test.go` só tem o repositório falso), então o teste é feito à mão, com a API rodando:

```sh
curl -X POST localhost:2306/products -d '{"name":"Teclado","price":199.9,"stock":5}'
curl localhost:2306/products
curl -X PUT localhost:2306/products/1 -d '{"name":"Teclado","price":149.9,"stock":3}'
curl localhost:2306/products/1
curl -i -X DELETE localhost:2306/products/1   # 204 No Content
curl localhost:2306/products/1                # {"error":"Produto não encontrado!"}
```

`podman-compose down` para o banco.
