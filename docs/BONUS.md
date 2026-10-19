# Desafios bônus (até +1,0 pt)

Os três desafios bônus da seção 6 foram implementados. Evidências reais de cada um estão
em [`evidencias/`](evidencias/) (arquivos 26 a 43).

## Proxy reverso (+0,4)

Um serviço `nginx` (imagem oficial `nginx:1.27-alpine`) foi adicionado na frente do `web`.
Mudanças no `docker-compose.yml`:

- O serviço `web` **perdeu** o bloco `ports: ["8080:5000"]` — agora só é alcançável de
  dentro da `rede_interna`, na porta `5000`, sem nenhuma porta publicada no host (ver
  [`evidencias/28-ps-com-nginx.txt`](evidencias/28-ps-com-nginx.txt): `web` aparece com
  `PORTS` igual a `5000/tcp`, sem mapeamento).
- O novo serviço `nginx` publica `"8080:80"` — ele é agora o **único** ponto de entrada do
  ambiente, exatamente como o desafio pede.
- Configuração em [`nginx/nginx.conf`](../nginx/nginx.conf): um `proxy_pass` simples para
  `http://web:5000` (resolvido pelo mesmo DNS interno da `rede_interna` que `web` usa para
  falar com `db`), repassando os cabeçalhos `Host`, `X-Real-IP`, `X-Forwarded-For` e
  `X-Forwarded-Proto` — prática padrão para o backend saber a origem real da requisição
  mesmo atrás do proxy. `client_max_body_size 10M` foi definido porque o padrão do nginx
  é 1 MB, o que rejeitaria uploads de fotos maiores que isso.

Testado de ponta a ponta através da nova porta: `GET /health`
([`29-health-via-nginx.txt`](evidencias/29-health-via-nginx.txt)) e um `POST /itens` com
upload de foto via `multipart/form-data`
([`30-cadastro-foto-via-nginx.txt`](evidencias/30-cadastro-foto-via-nginx.txt),
[`31-itens-via-nginx.txt`](evidencias/31-itens-via-nginx.txt)) — provando que o proxy
também repassa corretamente requisições com corpo binário, não só `GET`s simples.

Diagrama atualizado em [`arquitetura.md`](arquitetura.md).

## Backup e restauração do volume `fotos` (+0,3)

Demonstrado com um contêiner Alpine temporário e descartável (`docker run --rm`), sem
precisar parar o ambiente:

```bash
# 1. Backup: monta o volume fotos (somente leitura, de fato) e uma pasta do host,
#    e empacota tudo em .tar.gz
docker run --rm -v achados-perdidos_fotos:/data -v "<pasta-host>":/backup \
  alpine tar czf /backup/fotos-backup.tar.gz -C /data .
```

Evidência: [`39-backup-volume.txt`](evidencias/39-backup-volume.txt) (execução) e
[`40-backup-listagem.txt`](evidencias/40-backup-listagem.txt) (`tar tzf`, listando o
conteúdo do pacote gerado — confirma que o arquivo `.jpg` do volume foi capturado).

```bash
# 2. Restauração em um volume NOVO (nunca usado antes), para provar que o backup
#    é autossuficiente e não depende do volume original continuar existindo
docker volume create achados-perdidos_fotos_restore
docker run --rm -v achados-perdidos_fotos_restore:/data -v "<pasta-host>":/backup \
  alpine sh -c "cd /data && tar xzf /backup/fotos-backup.tar.gz"
```

Evidência: [`41-volume-restore-create.txt`](evidencias/41-volume-restore-create.txt),
[`42-restore-volume.txt`](evidencias/42-restore-volume.txt) e, principalmente,
[`43-fotos-apos-restore.txt`](evidencias/43-fotos-apos-restore.txt) — que lista o volume
restaurado e mostra o mesmo arquivo (`09d37684d0964c00a537688ed4c9532f.jpg`, 1305 bytes)
que estava no volume original antes do backup
([`38-fotos-antes-backup.txt`](evidencias/38-fotos-antes-backup.txt)).

O volume `achados-perdidos_fotos_restore` foi removido (`docker volume rm`) logo após a
verificação — era só para a demonstração, não faz parte do ambiente declarado no
`docker-compose.yml`.

## Painel do banco via profile `debug` (+0,3)

Serviço `adminer` (imagem oficial `adminer:4.8.1`) adicionado ao `docker-compose.yml` com
`profiles: ["debug"]` e porta `8081:8080`. Um serviço com `profiles` definido **não** sobe
num `docker compose up -d` normal — só entra na lista de serviços ativos quando o profile é
explicitamente pedido.

Evidência em duas direções, para provar o gate nos dois sentidos:

- **Sem** `--profile debug`: `docker compose up -d` sobe só `db`, `web` e `nginx`
  ([`36-up-sem-profile-debug.txt`](evidencias/36-up-sem-profile-debug.txt)); `docker compose
  ps` confirma que `adminer` não existe
  ([`37-ps-sem-adminer.txt`](evidencias/37-ps-sem-adminer.txt)).
- **Com** `docker compose --profile debug up -d`: o `adminer` é criado e sobe junto com os
  outros três ([`32-up-profile-debug.txt`](evidencias/32-up-profile-debug.txt),
  [`33-ps-com-adminer.txt`](evidencias/33-ps-com-adminer.txt)), respondendo `HTTP 200` em
  `http://localhost:8081/` ([`34-adminer-http-status.txt`](evidencias/34-adminer-http-status.txt)).

Para conectar pelo painel: abrir `http://localhost:8081`, selecionar sistema
**PostgreSQL**, servidor `db`, e usuário/senha/banco = os mesmos valores do `.env`
(`POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB`).
