# Simple Authentication API

Este projeto implementa uma **API RESTful** em Python (FastAPI) capaz de cadastrar e autenticar usuários, além de disponibilizar um endpoint de consulta a dados de terceiros via *web scraping*.

---

## Sumário

1. [Visão Geral](#visão-geral)
2. [Endpoints](#endpoints)
   - [Registro de Usuário](#registro-de-usuário)
   - [Autenticação de Usuário](#autenticação-de-usuário)
   - [Consulta de Dados](#consulta-de-dados)
3. [Como Executar (Etapa 1)](#como-executar-etapa 1)
   - [Estrutura de Diretórios](#estrutura-de-diretórios)
   - [Docker Hub](#docker-hub)
   - [compose.yaml Final](#composeyaml-final)
   - [Rodando Localmente](#rodando-localmente)
4. [Variáveis de Ambiente](#variáveis-de-ambiente)
5. [Link Docker Hub](#link-docker-hub)
6. [Screenshots](#screenshots)
7. [Vídeo de Demonstração](#vídeo-de-demonstração)

---

## Visão Geral

A **Simple Authentication API** fornece:

- **Registro** de usuários com armazenamento seguro de senhas (usando `bcrypt`).
- **Login** de usuários, retornando um **JWT** para autenticação.
- **Consulta protegida** (`GET /consultar`), que executa scraping em uma fonte externa para trazer dados atualizados (por exemplo, cotações de moedas).

A aplicação segue boas práticas de segurança:

- **Nunca** armazena senhas em texto claro.
- **JWT** para proteger endpoints.
- Conexão ao PostgreSQL via variáveis de ambiente.

---

## Endpoints

### Registro de Usuário

```http
POST /registrar
Content-Type: application/json
```

**Request**

```json
{
  "nome": "Samuel Jabes",
  "email": "samueljcc@al.insper.edu.br",
  "senha": "sam1234"
}
```

**Response (201)**

```json
{
  "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6InNhbXVlbGpjY0BhbC5pbnNwZXIuZWR1LmJyIiwicGFzc3dvcmQiOiJzYW0xMjM0In0.H3awAOszz5pQReXIFQtC6wxk0kfwlOjtXFh9QAzut8c"
}
```

```mermaid
sequenceDiagram
    autonumber
    actor Samuel
    Samuel->>+App: POST /registrar
    App->>+Postgres: SELECT * FROM users WHERE email = Samuel.email
    alt email já existe
      Postgres-->>App: 409 Conflict
      App-->>Samuel: 409 Conflict
    else email não existe
      App->>Postgres: INSERT user(hash_password)
      App->>App: gera JWT
      App-->>-Samuel: 201 Created + { jwt }
    end
```

> **Nota:** Armazenamos apenas o hash da senha, nunca o texto claro.

---

### Autenticação de Usuário

```http
POST /login
Content-Type: application/json
```

**Request**

```json
{
  "email": "samueljcc@al.insper.edu.br",
  "senha": "sam1234"
}
```

**Response (200)**

```json
{
  "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6InNhbXVlbGpjY0BhbC5pbnNwZXIuZWR1LmJyIiwicGFzc3dvcmQiOiJzYW0xMjM0In0.H3awAOszz5pQReXIFQtC6wxk0kfwlOjtXFh9QAzut8c"
}
```

```mermaid
sequenceDiagram
    autonumber
    actor Samuel
    Samuel->>+App: POST /login
    App->>+Postgres: SELECT hash_password FROM users WHERE email = Samuel.email
    alt usuário não encontrado ou senha inválida
      App-->>-Samuel: 401 Unauthorized
    else credenciais válidas
      App->>App: gera JWT
      App-->>-Samuel: 200 OK + { jwt }
    end
```

---

### Consulta de Dados

> **GET /consultar** (protegido)

**Header**

```
Authorization: Bearer <JWT>
```

**Response (200)** – exemplo de cotações USD→BRL e EUR→BRL via scraping:

```json
{
  "USD_TO_BRL": 5.632,
  "EUR_TO_BRL": 6.417
}
```

```mermaid
sequenceDiagram
    autonumber
    actor Samuel
    Samuel->>+App: GET /consultar (JWT)
    App->>App: valida JWT
    alt JWT inválido
      App-->>-Samuel: 403 Forbidden
    else JWT válido
      App->>Internet: scrap Wise USD→BRL
      App->>Internet: scrap Wise EUR→BRL
      App-->>-Samuel: 200 OK + { usd_brl, eur_brl }
    end
```

#### Detalhes do Web Scraping

Para obter os valores de câmbio na Wise, seguimos este processo:

1. Acesse a página de conversão desejada, por exemplo:

   - Dólar → Real: `https://wise.com/us/currency-converter/usd-to-brl-rate`
   - Euro → Real:  `https://wise.com/br/currency-converter/eur-to-brl`

2. Abra as ferramentas de desenvolvedor do navegador (F12) e utilize o **Inspecionar** para localizar o elemento que exibe o valor convertido.

3. Ao passar o cursor sobre a caixa de valor no HTML, verificou-se que o elemento é um `<span>` com a classe `text-success`.

4. No código Python, usamos **Requests** para obter o HTML estático da página e **BeautifulSoup** para fazer o *parse*.

5. Com `soup.select_one("span.text-success")`, capturamos esse elemento. Em seguida:

   ```python
   span = soup.select_one("span.text-success")
   rate_text = span.get_text(strip=True).replace(",", ".")
   valor = float(rate_text)
   ```

6. Adicionamos `valor` ao dicionário de respostas sob as chaves `USD_TO_BRL` e `EUR_TO_BRL`.

Este método se aplica a qualquer moeda suportada pelo conversor da Wise, bastando ajustar a URL e reutilizar a mesma lógica de seleção da classe `text-success`.

---

## Como Executar (Etapa 1)

### Estrutura de Diretórios

```
./
├── app.py
├── requirements.txt
├── sql/
│   └── users.sql
├── models/
│   └── login_schema.py
│   └── register_user_schema.py
│   └── user_schema.py
├── Dockerfile        # para a aplicação FastAPI
├── Dockerfile-db     # para o Postgres (executa users.sql)
├── compose.yaml      # orquestra app + database
├── README.md
└── .env
```

### Docker Hub

A imagem da API está publicada como:

```
samueljabes/simple_authentication_api:v1.0.1
```

### compose.yaml Final

O arquivo `compose.yaml`, na raiz do projeto, **usa apenas imagens do Docker Hub**:

```yaml
services:
  database:
    build:
      context: .
      dockerfile: Dockerfile-db
    container_name: database
    restart: always
    env_file:
      - .env
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}

  app:
    image: samueljabes/simple_authentication_api:v1.0.1
    ports:
      - "5000:5000"
    env_file:
      - .env
    depends_on:
      - database

```

### Rodando Localmente

1. Configure (ou crie) o arquivo `.env` na raiz:
   ```dotenv
   POSTGRES_USER=meuusuario
   POSTGRES_PASSWORD=minhasenha
   POSTGRES_DB=meubanco
   SECRET_KEY=SUA_CHAVE_JWT
   ```
2. Execute:
   ```bash
   docker compose down -v       # limpa containers + volume
   docker compose pull          # puxa a imagem da API (v1.0.1)
   docker compose up -d         # sobe app + database
   ```
3. Acesse `http://localhost:5000/docs` para testar.

---

## Variáveis de Ambiente

- `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`: credenciais do banco.
- `POSTGRES_HOST` e `POSTGRES_PORT` não são necessários no `compose.yaml` final, pois usamos a mesma stack.
- `SECRET_KEY`: chave para assinar JWT.

> **Nota de Segurança:** Em produção, não exponha portas de banco e não inclua credenciais no repositório.

---

## Link Docker Hub

https://hub.docker.com/r/samueljabes/simple_authentication_api

## Publicação no Docker Hub

Para publicar a imagem com a tag `latest`:
```bash
# Autentica no Docker Hub
docker login

# Build e push da tag latest
docker build -t samueljabes/simple_authentication_api:latest .
docker push samueljabes/simple_authentication_api:latest
```

Para criar e publicar uma nova tag `v1.0.1` a partir da imagem `latest`:
```bash
# Retag da imagem latest para v1.0.1
docker tag samueljabes/simple_authentication_api:latest samueljabes/simple_authentication_api:v1.0.1

# Push da nova tag
docker push samueljabes/simple_authentication_api:v1.0.1
```

Para executar localmente usando a tag `v1.0.1`:
```bash
# Puxa a imagem v1.0.1 do Docker Hub
docker pull samueljabes/simple_authentication_api:v1.0.1

docker compose pull

docker compose up -d
```

---

## Screenshots

![Swagger - DOCS](imgs/docs_swagger.png)

### Tela de Registro de Usuário

![Swagger - Registrar Usuário 1](imgs/registrar_endp1.png)
![Swagger - Registrar Usuário 2](imgs/registrar_endp2.png)
*Formulario de registro com payload e resposta de JWT.*

### Tela de Login de Usuário

![Swagger - Login Usuário 1](imgs/login_endp1.png)
![Swagger - Login Usuário 2](imgs/login_endp2.png)
*Formulario de login exibindo o campo de email, senha e resposta de JWT.*

### Tela de Consulta Protegida

![Swagger - Consulta de Dados](imgs/consultar_endp.png)
![Swagger - Consulta de Dados - Auth](imgs/consultar_auth.png)
*Endpoint `/consultar` solicitando o token JWT e retornando as cotações.*

---

---

## Vídeo de Demonstração

[![Demonstração do Projeto](https://img.youtube.com/vi/tYiMCcTRJI8/maxresdefault.jpg)](https://www.youtube.com/watch?v=tYiMCcTRJI8)

*Clique na imagem acima para assistir ao vídeo demonstrando o fluxo de registro, login e consulta.*

