# A5 — Reflexão e equivalência em nuvem

## 5. Questões de reflexão

**Q1. Por que o serviço `web` consegue se conectar ao banco usando o nome `db` em vez de um
endereço IP? Quem resolve esse nome e o que mudaria se os contêineres fossem criados com
`docker run` na rede bridge padrão?**

Porque `web` e `db` estão ligados à mesma rede definida pelo usuário, `rede_interna`
(bloco `networks` do `docker-compose.yml`). Toda rede *bridge* criada pelo usuário no
Docker vem com um servidor DNS embutido (resolvido pelo processo `dockerd`) que resolve o
nome de cada serviço/contêiner conectado a ela para o IP atual desse contêiner — é esse DNS
que resolve `db` para `172.18.0.2` (ver
[`evidencias/23-network-inspect.txt`](evidencias/23-network-inspect.txt)). Se os contêineres
fossem criados com `docker run` simples, sem `--network`, ambos cairiam na rede *bridge*
padrão (`docker0`), que **não** tem esse DNS automático — nela, contêineres só se enxergam
por IP (que muda a cada recriação) ou exigindo `--link` (recurso legado). `DB_HOST=db`
simplesmente pararia de resolver.

**Q2. Qual a diferença entre um volume nomeado e um bind mount? Justifique por que o
`init.sql` foi montado como bind mount e os dados do banco como volume nomeado.**

Um **volume nomeado** é um recurso de armazenamento que o próprio Docker cria e gerencia
(local em `/var/lib/docker/volumes/<nome>/_data` — ver
[`evidencias/16-volume-inspect-dados_banco.txt`](evidencias/16-volume-inspect-dados_banco.txt)):
não existe previamente no projeto, sobrevive independente do contêiner e é a forma
recomendada de persistir dados gerados *pela aplicação*. Um **bind mount** aponta
diretamente para um caminho já existente no sistema de arquivos do host (`./db/init.sql`),
sem o Docker criar nem gerenciar nada — o conteúdo é o mesmo arquivo do repositório, editável
fora do Docker. `init.sql` é código-fonte versionado no Git: faz sentido como bind mount
porque ele **precisa** refletir exatamente o arquivo do repositório, montado como somente
leitura (`:ro`) para o contêiner nunca escrever nele. Os dados do Postgres, ao contrário,
são gerados em tempo de execução pela própria aplicação e não devem ir para o Git — por
isso usam um volume nomeado, que o Docker cria vazio e a aplicação popula.

**Q3. Se você alterar o `init.sql` depois que o ambiente já subiu uma vez e executar
`docker compose up -d` de novo, a mudança é aplicada? Por quê? Como aplicar a alteração sem
perder dados?**

Não é aplicada automaticamente. A imagem oficial do `postgres` só executa os scripts de
`/docker-entrypoint-initdb.d/` quando o diretório de dados (`/var/lib/postgresql/data`,
ou seja, o volume `dados_banco`) está **vazio** — isso acontece uma única vez, na primeira
inicialização daquele volume. Depois disso, o volume já contém um cluster PostgreSQL
inicializado, e o entrypoint pula a etapa de inicialização em qualquer subida seguinte,
mesmo que o conteúdo de `init.sql` tenha mudado (foi exatamente o que a Tarefa 5
demonstrou: um `down`+`up` sem `-v` manteve os 5 itens, sem re-executar o script —
[`14-estatisticas-apos-recriacao.txt`](evidencias/14-estatisticas-apos-recriacao.txt)).
Para aplicar uma mudança de schema (como adicionar uma coluna) **sem perder os dados já
cadastrados**, a alteração precisa ser aplicada como uma migração vida real contra o banco
já existente — por exemplo, conectando com `docker compose exec db psql -U
$POSTGRES_USER -d $POSTGRES_DB -c "ALTER TABLE itens ADD COLUMN ..."`, ou rodando um
arquivo de migração separado. Rodar `down -v` para forçar o `init.sql` a executar de novo
funciona, mas apaga todos os itens e fotos cadastrados — por isso não é uma opção aceitável
fora de ambiente de desenvolvimento.

**Q4. Por que `depends_on` sozinho, sem `condition: service_healthy`, não garante que a
aplicação consiga se conectar ao banco na primeira tentativa?**

Um `depends_on` sem `condition` (ou com o padrão implícito `service_started`) só garante
**ordem de inicialização dos processos** — o Compose espera o contêiner do `db` começar a
rodar antes de criar o `web`, mas "começar a rodar" só significa que o processo
`postgres` foi iniciado dentro do contêiner, não que ele já terminou de inicializar o
cluster e está pronto para aceitar conexões na porta 5432. Esse processo de boot do
Postgres leva uma fração de segundo a alguns segundos, dependendo da máquina. Sem
`condition: service_healthy`, o `web` poderia subir e tentar `psycopg.connect(...)`
exatamente nesse intervalo, recebendo uma recusa de conexão (`ConnectionRefusedError`) e
fazendo até o primeiro `GET /health` falhar. Com `condition: service_healthy`, o Compose
só cria o contêiner `web` depois que o `healthcheck` do `db` (baseado em `pg_isready`, que
pergunta ativamente "você aceita conexões?") reportar `healthy` — o que a evidência
[`evidencias/01-up-build.txt`](evidencias/01-up-build.txt) confirma: `db Healthy` aparece
antes de `web Starting`.

**Q5. A ordem das instruções `COPY requirements.txt` → `RUN pip install` → `COPY app.py`
influencia o tempo de build. Explique usando o conceito de cache de camadas e cite o que
acontece quando só o `app.py` muda.**

O Docker constrói uma imagem em camadas, uma por instrução, e cacheia cada camada. Ao
reconstruir, ele compara o conteúdo de entrada de cada instrução com o da build anterior:
se for idêntico, reaproveita a camada já existente (rápido); se mudou, reconstrói essa
camada **e todas as seguintes**, mesmo que o conteúdo delas não tenha mudado — uma vez que
uma camada é invalidada, o cache deixa de valer a partir dali. Com `COPY requirements.txt
.` antes de `RUN pip install` e **antes** de `COPY . .` (que traz `app.py`), uma alteração
em `app.py` só invalida a camada `COPY . .` em diante — o `pip install`, que normalmente é
o passo mais demorado do build (baixar e instalar Flask, psycopg, Gunicorn e dependências
transitivas), continua usando o cache da build anterior e não roda de novo. Se a ordem
fosse invertida (copiar tudo, incluindo `app.py`, antes de instalar as dependências),
qualquer mudança de uma linha em `app.py` forçaria reinstalar todas as dependências do
zero a cada build, tornando o ciclo de desenvolvimento bem mais lento.

**Q6. Por que a porta 5432 do banco não deve ser publicada no host? Considerando que as
fotos podem mostrar documentos pessoais esquecidos (carteira de estudante, RG), relacione a
sua configuração com a LGPD e proponha duas medidas adicionais de proteção.**

Publicar a porta 5432 no host exporia o PostgreSQL diretamente para qualquer processo (ou,
num servidor real, qualquer máquina) com acesso à rede do host, abrindo uma superfície de
ataque desnecessária para tentativas de força bruta de senha ou exploração de
vulnerabilidades do próprio Postgres — sem nenhum ganho funcional, já que o único cliente
legítimo do banco é o serviço `web`, que já o alcança por dentro da `rede_interna`. Manter
o banco sem porta publicada (como está no `docker-compose.yml` atual — sem bloco `ports:`
no serviço `db`) segue o princípio de menor privilégio: cada componente só é acessível por
quem realmente precisa falar com ele. Sob a ótica da LGPD, as fotos de objetos esquecidos
podem efetivamente conter dados pessoais (uma carteira de estudante ou RG esquecidos
tornam o item, e a própria foto, dado pessoal, possivelmente sensível se expuser CPF ou
outras informações). O isolamento do banco já reduz um vetor de vazamento, mas duas medidas
adicionais cabem bem aqui:

1. **Controle de acesso e retenção nas fotos** — hoje qualquer pessoa que descubra a URL de
   `/fotos/<nome>` consegue baixar a imagem, já que não há autenticação nessa rota; exigir
   autenticação para download (ou gerar URLs assinadas com expiração) e definir uma
   política de retenção (apagar a foto após o item ser retirado ou após X dias) reduz a
   janela de exposição de dados pessoais.
2. **Minimização/anonimização no cadastro** — orientar (ou tecnicamente impedir) o upload
   de fotos que mostrem documentos com informação legível, e, se for indispensável manter
   a foto para identificação do item, aplicar borrão automático em regiões com texto/rosto
   antes de salvar — isso é tratamento de dado pessoal sob a LGPD e deveria ter uma base
   legal e finalidade claras, não apenas "guardar porque foi enviado".

**Q7. Tente executar `docker compose up -d --scale web=3`. O que acontece e por quê? O que
seria necessário para rodar três réplicas da aplicação? As réplicas poderiam compartilhar o
volume `fotos` com segurança?**

O comando falha ao tentar criar a segunda réplica, porque o serviço `web` declara
`ports: ["8080:5000"]` com uma porta fixa do host — duas (ou três) réplicas tentando
publicar simultaneamente a mesma porta `8080` do host entram em conflito, já que uma porta
do host só pode ser ocupada por um processo por vez. Para rodar três réplicas de fato
seria necessário remover o mapeamento de porta fixa de dentro do serviço `web` e colocar um
balanceador de carga/proxy reverso (como o `nginx` sugerido nos desafios bônus) na frente,
publicando ele a porta `8080` e distribuindo as requisições entre as três réplicas internas
(que não precisariam de porta publicada, só estarem na `rede_interna`). O volume `fotos`
**poderia** ser compartilhado entre as réplicas com segurança para leitura e escrita
simples de arquivos (é exatamente para isso que existem drivers de volume — no caso do
driver `local` usado aqui, todas as réplicas rodando na mesma máquina Docker enxergariam o
mesmo diretório), mas duas rotinas do `app.py` fariam sentido revisar: o nome do arquivo
já é gerado com `uuid.uuid4().hex`, o que evita colisão de nomes entre réplicas gravando ao
mesmo tempo; porém `GET /estatisticas` usa `len(os.listdir(UPLOAD_DIR))` para contar fotos,
o que é seguro mesmo com múltiplas réplicas lendo o mesmo diretório (list+count não tem
condição de corrida problemática aqui). O ponto mais delicado para escalar de verdade não é
o volume de fotos, e sim o banco: com três réplicas de `web` todas gravando no mesmo
Postgres, é o Postgres (não coberto por este desafio) que precisaria suportar a carga
concorrente.

## 5.1 Do ambiente local para a nuvem

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
|---|---|---|---|
| Contêiner `web` | Azure Container Apps (ou App Service for Containers) | Amazon ECS/Fargate (ou App Runner) | Cloud Run |
| Contêiner `db` + volume `dados_banco` | Azure Database for PostgreSQL – Flexible Server | Amazon RDS for PostgreSQL | Cloud SQL for PostgreSQL |
| Volume `fotos` | Azure Blob Storage | Amazon S3 | Google Cloud Storage |
| `rede_interna` | Azure Virtual Network (VNet) + regras de NSG | Amazon VPC + Security Groups | VPC do Google Cloud + regras de firewall |
| Arquivo `.env` (segredos) | Azure Key Vault | AWS Secrets Manager (ou Parameter Store) | Secret Manager |
| Imagem construída localmente | Azure Container Registry (ACR) | Amazon ECR | Artifact Registry |

A peça que mais muda ao migrar é o par **banco de dados + volume `fotos`**. Hoje ambos são
só diretórios em disco local gerenciados pelo Docker, amarrados à máquina onde os
contêineres rodam — o que funciona para um protótipo de single-host, mas não sobrevive à
ideia de nuvem de contêineres efêmeros e escaláveis (como no cenário da Q7, com múltiplas
réplicas). Na nuvem, o volume `dados_banco` deixa de existir como tal: o Postgres passa a
ser um serviço gerenciado (RDS/Cloud SQL/Azure Database), com backup, failover e scaling
cuidados pelo provedor, e o contêiner `web` só guarda um endpoint de conexão. O volume
`fotos`, por sua vez, deixa de ser um diretório local e vira armazenamento de objetos
(S3/GCS/Blob Storage), acessível por qualquer réplica do `web` ao mesmo tempo sem depender
de todas rodarem na mesma máquina — o que é justamente o que falta hoje para o `--scale
web=3` da Q7 funcionar de verdade com múltiplas réplicas em hosts diferentes. Em
comparação, o próprio contêiner `web` muda relativamente pouco: a mesma imagem Docker
construída localmente (via `docker build`) é o artefato publicado num registry gerenciado
e executado por um serviço de contêineres gerenciado — a lógica da aplicação e o Dockerfile
não precisam mudar.
