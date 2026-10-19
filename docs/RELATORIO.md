# A3 — Relatório técnico

## (a) Dockerfile — instrução a instrução

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd --system --no-create-home appuser \
    && mkdir -p /app/uploads \
    && chown -R appuser:appuser /app/uploads

USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

1. **`FROM python:3.12-slim`** — parte de uma imagem oficial do Python 3.12 na variante
   `slim`, que contém só o essencial para rodar Python (sem compiladores, headers e
   ferramentas de build da imagem completa). Isso mantém a imagem final menor e reduz a
   superfície de ataque, sem abrir mão de pacotes `manylinux` pré-compilados (o
   `psycopg[binary]` já traz o driver do PostgreSQL compilado, então não é necessário
   `gcc`/`libpq-dev` na imagem).

2. **`WORKDIR /app`** — define `/app` como diretório de trabalho para todas as instruções
   seguintes (`COPY`, `RUN`, `CMD`). Evita caminhos absolutos repetidos e é o mesmo
   diretório que `UPLOAD_DIR=/app/uploads` (padrão do `app.py`) pressupõe como base.

3. **`COPY requirements.txt .`** seguido de **`RUN pip install --no-cache-dir -r requirements.txt`**
   — copia **só** o arquivo de dependências antes do resto do código, de propósito. O
   Docker cacheia cada instrução como uma camada e só invalida uma camada (e todas as
   seguintes) quando o conteúdo copiado muda. Como `requirements.txt` muda raramente
   (bem menos que `app.py`), isso significa que, no dia a dia, alterar a lógica da
   aplicação **não força reinstalar as dependências do zero** — só a camada do `pip
   install` é reconstruída quando o `requirements.txt` muda de fato. Veja a questão Q5 em
   [`REFLEXAO.md`](REFLEXAO.md) para o cenário concreto. O `--no-cache-dir` evita que o
   pip guarde os pacotes baixados em `~/.cache/pip` dentro da imagem — sem isso, cada
   pacote baixado ficaria duplicado na imagem final (uma cópia instalada, outra em cache),
   inflando o tamanho sem benefício nenhum em um ambiente descartável como um contêiner.

4. **`COPY . .`** — só depois de instalar as dependências é que o código da aplicação
   (`app.py`) entra na imagem. Isso é o que torna o truque do passo anterior possível: se
   essa instrução viesse antes do `pip install`, qualquer mudança em `app.py` invalidaria
   também a camada de instalação de dependências, mesmo sem o `requirements.txt` ter
   mudado.

5. **`RUN useradd --system --no-create-home appuser && mkdir -p /app/uploads && chown -R appuser:appuser /app/uploads`**
   — três ações em uma única camada (via `&&`), para não gerar camadas intermediárias
   desnecessárias:
   - `useradd --system --no-create-home appuser` cria um usuário de sistema (sem senha de
     login, sem diretório home) dedicado a rodar a aplicação — nunca o root.
   - `mkdir -p /app/uploads` garante que o diretório de uploads exista **antes** do
     `chown`, já que ele é o alvo do volume nomeado `fotos` no `docker-compose.yml`.
   - `chown -R appuser:appuser /app/uploads` dá a esse usuário permissão de escrita no
     diretório onde as fotos são salvas — sem isso, o processo rodando como `appuser`
     (ver próximo passo) não conseguiria gravar arquivos ali.

6. **`USER appuser`** — a partir daqui, toda instrução seguinte (e o processo principal do
   contêiner, via `CMD`) roda como `appuser`, não como root. Atende diretamente ao RNF04 e
   é verificável com `docker compose exec web whoami` (evidência em
   [`evidencias/25-whoami.txt`](evidencias/25-whoami.txt), que retorna `appuser`).

7. **`EXPOSE 5000`** — instrução documental: declara que o contêiner escuta na porta 5000.
   Não publica a porta sozinha (quem faz isso é o `ports:` do `docker-compose.yml`) — serve
   para quem lê o Dockerfile (ou ferramentas como `docker inspect`) saber qual porta a
   aplicação usa internamente, sem precisar ler o código.

8. **`CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]`** — inicia o servidor de
   produção Gunicorn (não o servidor de desenvolvimento do Flask) na forma **exec** (lista
   JSON, não string). Na forma exec, o processo do Gunicorn vira o **PID 1** do contêiner
   diretamente, recebendo sinais do Docker (como `SIGTERM` em um `docker compose stop`)
   sem um shell intermediário — o que permite um desligamento gracioso. `--bind
   0.0.0.0:5000` faz o Gunicorn escutar em todas as interfaces do contêiner (não só
   `127.0.0.1`), o que é obrigatório para que o tráfego vindo da rede Docker (encaminhado
   pelo `ports:` do host) consiga alcançá-lo. `app:app` aponta para o módulo `app.py` e o
   objeto Flask `app` dentro dele.

### `.dockerignore`

```
.env
__pycache__/
*.pyc
```

Impede que o contexto de build enviado ao daemon do Docker inclua o `.env` (credenciais
reais nunca devem entrar em uma imagem — RNF03), caches de bytecode Python (`__pycache__/`,
`*.pyc`), que não servem de nada na imagem e ainda poderiam mascarar código desatualizado.

## (b) docker-compose.yml — bloco a bloco

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - dados_banco:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - rede_interna
```

- **`image: postgres:16-alpine`** — usa a imagem oficial, sem build próprio: não há
  necessidade de customizar o PostgreSQL, então não vale a pena manter um Dockerfile extra
  para ele.
- **`environment`** — as três variáveis que a imagem oficial do Postgres exige para se
  autoconfigurar na primeira subida (criar o banco, o usuário e definir a senha). Os
  valores vêm de `${...}`, que o Compose resolve lendo o arquivo `.env` na raiz do
  projeto — nenhuma senha literal aparece neste arquivo (RNF03).
- **`volumes`** — duas entradas bem diferentes:
  - `dados_banco:/var/lib/postgresql/data` é um **volume nomeado**: o Docker gerencia o
    armazenamento (fica em `/var/lib/docker/volumes/.../_data` dentro da VM do Docker
    Desktop — ver evidência em
    [`evidencias/16-volume-inspect-dados_banco.txt`](evidencias/16-volume-inspect-dados_banco.txt)),
    e o conteúdo sobrevive a `docker compose down` e a recriações do contêiner.
  - `./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro` é um **bind mount**: aponta
    diretamente para um arquivo do repositório, montado como somente leitura. Ver Q2 em
    [`REFLEXAO.md`](REFLEXAO.md) para a justificativa de usar bind mount aqui e volume
    nomeado para os dados.
- **`healthcheck`** — `pg_isready` é a ferramenta oficial do Postgres para perguntar "você
  está pronto para aceitar conexões?" (diferente de só "o processo está rodando"). O `$$`
  duplicado escapa o `$` para o Compose não tentar substituir `$POSTGRES_USER` como se
  fosse uma variável *dele*; o comando real, dentro do contêiner, é `pg_isready -U
  $POSTGRES_USER -d $POSTGRES_DB`, usando as variáveis de ambiente que a própria imagem já
  define a partir do bloco `environment` acima. `interval: 5s` / `timeout: 5s` / `retries:
  5` controlam a frequência e tolerância da checagem — até 5 tentativas com 5s de intervalo
  (25s) antes de considerar o serviço não saudável.
- **`restart: unless-stopped`** — reinicia o contêiner automaticamente se ele cair (crash,
  reboot do host), exceto se alguém o parou manualmente (`docker compose stop`) — nesse
  caso ele não volta sozinho.
- **`networks: [rede_interna]`** — conecta o serviço à rede definida no final do arquivo.

```yaml
  web:
    build: ./app
    environment:
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: ${POSTGRES_DB}
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      UPLOAD_DIR: /app/uploads
    ports:
      - "8080:5000"
    volumes:
      - fotos:/app/uploads
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - rede_interna
```

- **`build: ./app`** — ao contrário do `db`, este serviço não usa uma imagem pronta: o
  Compose constrói a imagem a partir do `Dockerfile` dentro de `./app` (analisado na seção
  (a)).
- **`environment`** — valores que o `app.py` lê via `os.getenv(...)`: `DB_HOST=db` é o
  nome do serviço `db` (resolvido pelo DNS interno do Docker — ver Q1 em
  [`REFLEXAO.md`](REFLEXAO.md)); `DB_NAME`/`DB_USER`/`DB_PASSWORD` reaproveitam as mesmas
  variáveis do `.env` usadas pelo `db`, garantindo que os dois lados usem as mesmas
  credenciais sem repeti-las em dois lugares diferentes do `.env`; `UPLOAD_DIR=/app/uploads`
  bate exatamente com o diretório criado e liberado para o `appuser` no Dockerfile e com o
  ponto de montagem do volume `fotos` logo abaixo.
- **`ports: ["8080:5000"]`** — publica a porta `5000` do contêiner (onde o Gunicorn escuta)
  como `8080` no host. É a única porta exposta para fora do Docker em todo o ambiente.
- **`volumes: [fotos:/app/uploads]`** — volume nomeado que guarda as fotos enviadas,
  independente do ciclo de vida do contêiner `web`.
- **`depends_on: db: condition: service_healthy`** — faz o Compose só criar/iniciar o
  contêiner `web` depois que o `healthcheck` do `db` reportar saudável, não apenas "o
  processo do Postgres iniciou" (que é o que um `depends_on` simples, sem `condition`,
  garantiria). Ver Q4 em [`REFLEXAO.md`](REFLEXAO.md) para por que essa diferença importa
  na prática. A evidência em
  [`evidencias/01-up-build.txt`](evidencias/01-up-build.txt) mostra exatamente essa ordem:
  `Container achados-perdidos-db-1 Healthy` aparece *antes* de
  `Container achados-perdidos-web-1 Starting`.
- **`restart` / `networks`** — mesmo raciocínio do serviço `db`.

```yaml
networks:
  rede_interna:
    driver: bridge

volumes:
  dados_banco:
  fotos:
```

- **`networks.rede_interna`** — rede *bridge* definida pelo usuário (diferente da rede
  bridge padrão do Docker). Redes definidas pelo usuário ganham um servidor DNS interno
  automático que resolve o nome de cada serviço para o IP do seu contêiner — é isso que
  permite `web` falar com `db` pelo nome, em vez de IP fixo.
- **`volumes.dados_banco` / `volumes.fotos`** — declaração dos dois volumes nomeados
  referenciados acima. Sem entradas extras (como `driver` ou `external`), o Compose usa o
  driver `local` padrão e cria os volumes automaticamente na primeira subida, prefixados
  com o nome do projeto (`achados-perdidos_dados_banco`, `achados-perdidos_fotos` — ver
  [`evidencias/15-volume-ls.txt`](evidencias/15-volume-ls.txt)).

## (c) Caminho completo de um `POST /itens` com foto

Requisição de exemplo (a mesma usada na evidência
[`evidencias/06-cadastro-foto.txt`](evidencias/06-cadastro-foto.txt)):

```bash
curl -F descricao="Oculos de grau" -F local="Bloco B" -F foto=@oculos.jpg \
     http://localhost:8080/itens
```

1. O `curl` monta uma requisição HTTP `POST` `multipart/form-data` para
   `localhost:8080/itens` e abre uma conexão TCP para a porta `8080` do **host**.
2. O Docker Desktop encaminha essa conexão para a porta `5000` do contêiner `web`, conforme
   o mapeamento `"8080:5000"` do `docker-compose.yml`.
3. Dentro do contêiner, o **Gunicorn** (processo PID 1, rodando como `appuser`) aceita a
   conexão e repassa a requisição para o objeto Flask `app`.
4. O Flask faz o roteamento por método + path e chama `cadastrar()` (decorada com
   `@app.post("/itens")` em `app.py`).
5. `cadastrar()` lê `descricao` e `local` de `request.form` e valida que ambos existem;
   se faltar algum, retorna `400` imediatamente (sem tocar banco ou disco).
6. Lê o arquivo enviado de `request.files.get("foto")`. Se existe e tem nome, gera um nome
   novo e único com `uuid.uuid4().hex` + a extensão original (ex.:
   `9330c549c05b413b950ab8902e6ee83d.jpg`) — evitando colisões de nome e *path traversal*
   via nome de arquivo controlado pelo cliente.
7. `foto.save(os.path.join(UPLOAD_DIR, nome_foto))` grava o arquivo em
   `/app/uploads/<nome_foto>` **dentro do contêiner**. Como `/app/uploads` é o ponto de
   montagem do volume nomeado `fotos`, o dado é escrito fisicamente no armazenamento
   gerenciado desse volume (fora do *writable layer* do contêiner), e sobrevive à remoção
   do contêiner.
8. Com o nome da foto definido (ou `None`, se nenhuma foi enviada), `conectar()` abre uma
   conexão `psycopg` para `DB_URL`, que resolve `host=db` pelo DNS interno da
   `rede_interna` até o contêiner `db`.
9. Executa `INSERT INTO itens (descricao, local, foto) VALUES (...) RETURNING id`, que o
   PostgreSQL grava no arquivo de dados dentro de `/var/lib/postgresql/data` — ponto de
   montagem do volume nomeado `dados_banco`, novamente fora do contêiner `db` em si.
10. O `id` retornado pelo `RETURNING id` volta para o Flask, que monta a resposta
    `{"id": novo_id, "foto": nome_foto}` com status `201 Created`.
11. A resposta volta pelo Gunicorn → porta `5000` do contêiner → mapeamento de porta do
    Docker → porta `8080` do host → `curl`, que a imprime no terminal.

A partir desse ponto, o arquivo pode ser lido de volta por qualquer cliente em
`GET /fotos/<nome_foto>` (evidência de cabeçalhos HTTP em
[`evidencias/08-foto-headers.txt`](evidencias/08-foto-headers.txt)), e o item aparece em
`GET /itens` e é contado em `GET /estatisticas`.

## (d) Ciclo de vida dos volumes (com base na Tarefa 5)

Resumo do que foi efetivamente observado e registrado em `docs/evidencias/`:

| Situação | Comando | Itens no banco | Fotos no volume | Evidência |
|---|---|---|---|---|
| Estado inicial (seed do `init.sql`) | `up -d --build` | 3 | 0 | [`03-health.txt`](evidencias/03-health.txt), [`04-itens.txt`](evidencias/04-itens.txt) |
| Após cadastrar 2 itens com foto | `POST /itens` ×2 | 5 | 2 | [`11-estatisticas-antes-down.txt`](evidencias/11-estatisticas-antes-down.txt) |
| Após `down` (sem `-v`) + `up -d` | recriação dos contêineres | 5 | 2 | [`14-estatisticas-apos-recriacao.txt`](evidencias/14-estatisticas-apos-recriacao.txt) |
| Após `down -v` + `up -d` | remoção dos volumes | 3 | 0 | [`22-estatisticas-apos-down-v.txt`](evidencias/22-estatisticas-apos-down-v.txt) |

**Por que o 1º `down`+`up` preserva os dados:** `docker compose down` (sem `-v`) remove
apenas os **contêineres** e a **rede** — os volumes nomeados (`dados_banco`, `fotos`) não
são removidos por padrão, exatamente para proteger dados contra recriação acidental de
contêineres. O comando `up -d` seguinte cria contêineres novos (PIDs, IPs e nomes internos
diferentes — note o `container` retornado por `/health` muda), mas o Compose os religa aos
**mesmos** volumes nomeados que já existiam, pelo nome. Como o diretório de dados do
Postgres já continha as tabelas e linhas gravadas, o entrypoint oficial da imagem
**não** executa `init.sql` de novo (ele só roda scripts de inicialização quando o diretório
de dados está vazio) — por isso os 3 itens de seed não duplicam, e os 2 itens cadastrados
manualmente continuam lá.

**Por que o `down -v` apaga tudo:** a flag `-v` instrui o Compose a remover também os
volumes nomeados declarados no `docker-compose.yml`. Isso apaga fisicamente o conteúdo de
`/var/lib/docker/volumes/achados-perdidos_dados_banco` e
`achados-perdidos_fotos`. Na subida seguinte, o Compose recria os volumes **vazios**; o
contêiner `db` encontra um diretório de dados vazio e, seguindo o comportamento padrão da
imagem oficial do Postgres, executa `init.sql` de novo — recriando a tabela `itens` e
reinserindo só os três itens de exemplo originais. O volume `fotos`, por sua vez, não tem
nenhum mecanismo de "seed": ele simplesmente volta vazio, e as duas fotos enviadas durante
os testes são perdidas de forma permanente (não havia mais nenhuma cópia delas fora do
volume).
