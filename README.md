# Achados & Perdidos — Comp Nuvem

Empacotamento em contêineres do sistema de Achados & Perdidos do Setor de Atendimento ao
Estudante: uma API Flask + banco PostgreSQL que permite cadastrar e consultar objetos
encontrados no campus, com foto, sem depender de instalação manual ou planilha compartilhada.

## Objetivo

Entregar um ambiente que sobe por completo com um único comando
(`docker compose up -d --build`), mantendo os dados do banco e as fotos persistidos em
volumes nomeados mesmo quando os contêineres são recriados — e documentar como e por que
cada peça funciona, para que a próxima equipe consiga migrar isso para a nuvem.

## Arquitetura (resumo)

- **web**: API Flask servida pelo Gunicorn, porta `8080` no host → `5000` no contêiner.
- **db**: PostgreSQL 16 (imagem oficial `postgres:16-alpine`), sem porta publicada no host.
- **rede_interna**: rede bridge que permite `web` encontrar `db` pelo nome.
- **dados_banco**: volume nomeado com os dados do PostgreSQL.
- **fotos**: volume nomeado com as fotos enviadas pelos usuários.

Diagrama completo em [`docs/arquitetura.md`](docs/arquitetura.md). Explicação linha a linha
do Dockerfile e do docker-compose.yml em [`docs/RELATORIO.md`](docs/RELATORIO.md).

## Pré-requisitos

- Docker e Docker Compose instalados (Docker Desktop no Windows/Mac, ou Docker Engine +
  plugin Compose no Linux).
- Nenhuma instalação manual de Python, PostgreSQL ou dependências — tudo roda em contêiner.

## Configuração do `.env`

As credenciais do banco não ficam no código nem no `docker-compose.yml`. Copie o modelo e
preencha com valores reais:

```bash
cp .env.example .env
```

Edite `.env` e defina `POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD`. Esse arquivo
**não é versionado** (está no `.gitignore`) — cada ambiente (dev, produção) tem o seu.

## Subir o ambiente

```bash
docker compose up -d --build
docker compose ps        # os dois serviços "running"; db "healthy"
```

A primeira subida cria os volumes `dados_banco` e `fotos` e inicializa o banco a partir de
`db/init.sql` (três itens de exemplo).

## Testar

```bash
curl http://localhost:8080/health
curl http://localhost:8080/itens
curl "http://localhost:8080/itens?local=biblio"

# cadastro com foto
curl -F descricao="Oculos de grau" -F local="Bloco B" -F foto=@caminho/para/foto.jpg \
     http://localhost:8080/itens

curl http://localhost:8080/estatisticas
```

No PowerShell do Windows, use `curl.exe` no lugar de `curl` (o `curl` do PowerShell é um
alias de `Invoke-WebRequest` e não aceita os mesmos parâmetros).

## Parar o ambiente (mantendo os dados)

```bash
docker compose down
```

Os volumes `dados_banco` e `fotos` continuam no disco. Ao rodar `docker compose up -d`
novamente, os itens e fotos cadastrados continuam lá.

## Destruir o ambiente (apagando os dados)

```bash
docker compose down -v
```

Remove também os volumes nomeados. Na próxima subida, o banco é reinicializado do zero a
partir de `db/init.sql` (volta a ter só os três itens de exemplo) e a pasta de fotos volta
vazia. Evidência disso em [`docs/evidencias/`](docs/evidencias/).

## Rotas da API

| Rota | Método | Descrição |
|---|---|---|
| `/health` | GET | Retorna `status: ok` e `banco: conectado` quando o ambiente está no ar. |
| `/itens` | GET | Lista os objetos cadastrados. Aceita `?local=texto` para filtrar por local (case-insensitive, substring). |
| `/itens` | POST | Cadastra um objeto. Campos (`multipart/form-data`): `descricao` e `local` (obrigatórios), `foto` (opcional). |
| `/fotos/<nome>` | GET | Devolve o arquivo de foto armazenado no volume `fotos`. |
| `/estatisticas` | GET | Retorna `itens_no_banco` (total de linhas na tabela) e `fotos_no_volume` (total de arquivos em `/app/uploads`). |

## Estrutura do repositório

```
achados-perdidos/
|-- app/
|   |-- app.py             (fornecido pelo professor)
|   |-- requirements.txt   (fornecido pelo professor)
|   |-- Dockerfile
|   `-- .dockerignore
|-- db/
|   `-- init.sql           (fornecido pelo professor)
|-- docs/
|   |-- arquitetura.md
|   |-- RELATORIO.md
|   |-- REFLEXAO.md
|   `-- evidencias/
|-- docker-compose.yml
|-- .env                   (não versionado)
|-- .env.example
|-- .gitignore
`-- README.md
```

## Documentação complementar

- [`docs/arquitetura.md`](docs/arquitetura.md) — diagrama de arquitetura.
- [`docs/RELATORIO.md`](docs/RELATORIO.md) — explicação do Dockerfile, do docker-compose.yml,
  o caminho de uma requisição `POST /itens` com foto e o ciclo de vida dos volumes.
- [`docs/REFLEXAO.md`](docs/REFLEXAO.md) — respostas às questões de reflexão e tabela de
  equivalência com serviços gerenciados em nuvem.
- [`docs/evidencias/`](docs/evidencias/) — saídas reais dos testes funcionais e da prova de
  persistência.
