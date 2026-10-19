# A4 — Evidências

Saídas reais, capturadas durante a execução do roteiro das Tarefas 4 e 5
(2026-10-19). Cada item abaixo é o nome do arquivo seguido de uma legenda de uma linha.

## Tarefa 4 — subida e testes funcionais

- `00-reset-ambiente.txt` — `docker compose down -v` para começar de um ambiente limpo, sem containers/volumes de testes manuais anteriores.
- `01-up-build.txt` — `docker compose up -d --build`; note `db Healthy` antes de `web Starting` (RNF05).
- `02-compose-ps.txt` — `docker compose ps`: os dois serviços `running`, `db` em `(healthy)`.
- `03-health.txt` — `GET /health` → `status: ok`, `banco: conectado` (RF01).
- `04-itens.txt` — `GET /itens` logo após o build: os 3 itens de exemplo do `init.sql`.
- `05-itens-filtro.txt` — `GET /itens?local=biblio`: filtro por local funcionando (RF02).
- `06-cadastro-foto.txt` — `POST /itens` com foto (`oculos.jpg`): cadastro retorna `id` e `foto` (RF03).
- `07-estatisticas.txt` — `GET /estatisticas` após o primeiro cadastro: 4 itens, 1 foto.
- `08-foto-headers.txt` — `curl -I` na URL da foto cadastrada: `200 OK`, `Content-Type: image/jpeg` (RF04).
- `09-logs-web.txt` — `docker compose logs web --tail 20`: Gunicorn de pé, sem erros.

## Tarefa 5 — prova de persistência

- `10-cadastro-foto2.txt` — segundo cadastro com foto (`cachecol.jpg`), para ter ≥2 itens com foto antes da recriação.
- `11-estatisticas-antes-down.txt` — baseline antes do `down`: 5 itens, 2 fotos.
- `12-down.txt` — `docker compose down` (sem `-v`): contêineres e rede removidos, **volumes mantidos**.
- `13-up-recriado.txt` — `docker compose up -d`: contêineres recriados do zero, religados aos mesmos volumes.
- `14-estatisticas-apos-recriacao.txt` — `GET /estatisticas` após recriar: ainda 5 itens, 2 fotos — nada se perdeu.
- `14b-itens-apos-recriacao.txt` — `GET /itens` após recriar: os mesmos 5 itens, incluindo os 2 com foto.
- `15-volume-ls.txt` — `docker volume ls`: aparecem `achados-perdidos_dados_banco` e `achados-perdidos_fotos` entre os volumes do host.
- `16-volume-inspect-dados_banco.txt` — `docker volume inspect` do volume do banco: mostra o `Mountpoint` físico no host.
- `17-volume-inspect-fotos.txt` — `docker volume inspect` do volume de fotos: mesma lógica.
- `18-ls-uploads.txt` — `docker compose exec web ls -l /app/uploads`: as 2 fotos cadastradas, dono `appuser` (não-root).
- `19-estatisticas-antes-down-v.txt` — baseline antes do `down -v`: ainda 5 itens, 2 fotos.
- `20-down-v.txt` — `docker compose down -v`: contêineres, rede **e volumes** removidos.
- `21-up-apos-down-v.txt` — `docker compose up -d`: tudo recriado do zero, volumes vazios.
- `22-estatisticas-apos-down-v.txt` — `GET /estatisticas` após `down -v`: voltou para 3 itens, 0 fotos — prova que os volumes foram de fato apagados e o `init.sql` rodou de novo.
- `22b-itens-apos-down-v.txt` — `GET /itens` após `down -v`: só os 3 itens de exemplo originais do `init.sql`.

## Extras pedidos no A4

- `23-network-inspect.txt` — `docker network inspect achados-perdidos_rede_interna`: mostra os dois contêineres conectados e seus IPs internos.
- `24-image-ls.txt` — `docker image ls achados-perdidos-web`: tamanho da imagem final (≈219 MB em disco / 53,8 MB de conteúdo).

## Checklist de entrega

- `25-whoami.txt` — `docker compose exec web whoami`: retorna `appuser`, confirmando que o contêiner não roda como root (RNF04).

## Desafios bônus (ver [`../BONUS.md`](../BONUS.md))

- `26-down-para-bonus.txt` / `27-up-com-nginx.txt` — ambiente recriado já com o serviço `nginx` no `docker-compose.yml`.
- `28-ps-com-nginx.txt` — `docker compose ps`: `web` aparece sem porta publicada (`5000/tcp`), só `nginx` publica `8080`.
- `29-health-via-nginx.txt` — `GET /health` através do `nginx` na porta `8080`: continua `ok`/`conectado`.
- `30-cadastro-foto-via-nginx.txt` / `31-itens-via-nginx.txt` — `POST /itens` com upload de foto atravessando o proxy reverso, provando que `multipart/form-data` passa corretamente.
- `32-up-profile-debug.txt` / `33-ps-com-adminer.txt` — `docker compose --profile debug up -d`: o `adminer` é criado junto com os outros serviços.
- `34-adminer-http-status.txt` — `adminer` respondendo `HTTP 200` em `localhost:8081`.
- `35-remove-adminer.txt` / `36-up-sem-profile-debug.txt` / `37-ps-sem-adminer.txt` — sem `--profile debug`, o `adminer` não é criado (prova do gate no sentido contrário).
- `38-fotos-antes-backup.txt` — conteúdo do volume `fotos` antes do backup.
- `39-backup-volume.txt` — `docker run --rm ... tar czf`: geração do `fotos-backup.tar.gz` com um contêiner Alpine temporário.
- `40-backup-listagem.txt` — `tar tzf` no backup gerado, confirmando o arquivo empacotado.
- `41-volume-restore-create.txt` / `42-restore-volume.txt` — criação de um volume novo e restauração do backup nele.
- `43-fotos-apos-restore.txt` — conteúdo do volume restaurado: mesmo arquivo, mesmo tamanho do volume original.
