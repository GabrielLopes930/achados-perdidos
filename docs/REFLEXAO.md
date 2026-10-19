# Questões de reflexão e equivalência em nuvem

As evidências citadas estão em `docs/evidencias/` e descritas em
`docs/evidencias/LEGENDAS.md`.

---

## Q1. Resolução do nome `db`

O `web` usa `DB_HOST: db`, que é o nome do serviço no `docker-compose.yml`.
Quem resolve esse nome é o servidor DNS embutido do Docker, presente em
redes definidas pelo usuário como a `rede_interna`. Ele traduz `db` para o
IP atual do contêiner: na evidência `21-network-inspect.txt`, o `db` aparece
com `172.21.0.2` e o `web` com `172.21.0.3`. Usar o nome é necessário porque
esses IPs mudam a cada recriação.
Com `docker run` na rede `bridge` padrão, não haveria resolução por nome:
seria preciso descobrir o IP com `docker inspect` e atualizá-lo a cada
recriação, ou criar uma rede própria com `docker network create`.

## Q2. Volume nomeado vs. bind mount

Um volume nomeado é criado e administrado pelo Docker, fica em uma área
própria do host (`/var/lib/docker/volumes/`, conforme as evidências `18` e
`19`) e é referenciado só pelo nome. Um bind mount expõe um caminho
específico do host dentro do contêiner. O `init.sql` é bind mount porque é
um arquivo do repositório, versionado com o código, e precisa ser lido da
pasta `db/`; foi montado como `:ro` porque o contêiner só o lê. Os dados do
banco usam volume nomeado porque são gerados em execução e nunca devem ser
versionados, e porque o Docker gerencia as permissões do diretório, evitando
conflitos de dono comuns com bind mount no Windows.

## Q3. Alterar o `init.sql` depois do primeiro `up`

Não, a mudança não é aplicada. A imagem do PostgreSQL só executa os scripts
de `/docker-entrypoint-initdb.d/` quando o diretório de dados está vazio;
como o volume `dados_banco` já guarda um banco inicializado, o script é
ignorado. Os testes mostraram os dois lados: após o `down -v`, o volume
voltou vazio e o `init.sql` rodou de novo, reinserindo os três itens com
data nova (evidência `27`); já num volume existente, até a senha nova do
`.env` foi ignorada, como registrado no relatório. Para aplicar sem perder
dados, a coluna deve ser criada no banco em execução, com
`docker compose exec db psql ... -c "ALTER TABLE itens ADD COLUMN ..."`.

## Q4. Por que `depends_on` sozinho não basta

Na forma simples, o `depends_on` só controla a ordem de início: garante que
o contêiner do banco foi iniciado, não que o PostgreSQL já aceita conexões.
Entre esses dois momentos há alguns segundos em que o banco ainda está
inicializando, e uma conexão feita nessa janela é recusada. Como o `app.py`
não tenta de novo, a aplicação responderia com erro logo após a subida. Com
`condition: service_healthy`, o `web` só inicia depois que o healthcheck
`pg_isready` do `db` passa. Isso aparece nas evidências `14` e `25`: o
Compose mostra o `db` em `Waiting`, depois `Healthy`, e só então inicia o
`web`.

## Q5. Ordem das instruções e cache de camadas

Cada instrução do Dockerfile gera uma camada guardada em cache. No build,
o Docker reaproveita as camadas até encontrar a primeira que mudou e refaz
todas a partir dela. Como o `requirements.txt` é copiado e instalado antes
do `app.py`, alterar só o código invalida apenas a última camada: o
`pip install` é reaproveitado. Isso foi observado na prática: o primeiro
build levou 19,5 s, e o seguinte, sem mudanças, levou 1,9 s, com todas as
etapas marcadas como `CACHED`. Se o código fosse copiado antes do
`pip install`, qualquer alteração em uma linha forçaria a reinstalação de
todas as dependências.

## Q6. Porta 5432 fechada, LGPD e proteções adicionais

Publicar a 5432 exporia o banco a qualquer um que alcance o host, permitindo
ataques diretos de senha sem passar pela aplicação. O único cliente
legítimo é o `web`, que já o alcança pela `rede_interna`; por isso só a 8080
do `nginx` é publicada, e nem o `web` fica exposto (evidência `02`). Como as fotos podem mostrar documentos como RG
e carteira de estudante, o volume `fotos` guarda dados pessoais, e a LGPD
exige medidas de segurança contra acesso não autorizado (art. 46). Proponho
duas medidas: (1) exigir autenticação nas rotas `/itens` e `/fotos`, hoje
públicas, restringindo o acesso ao setor de atendimento; (2) apagar
automaticamente as fotos de itens já devolvidos, mantendo os dados só pelo
tempo necessário (art. 15).

## Q7. `docker compose up -d --scale web=3`

O resultado depende da porta. Com o `web` publicando `8080:5000`, o comando
falha com `port is already allocated`: as réplicas herdam a mesma porta, e
só uma pode reservar a 8080 do host. Foi o erro observado nos testes quando
outro projeto ocupava a porta. Por isso foi adicionado o proxy reverso: o
`web` deixou de publicar porta, só o `nginx` expõe a 8080, e o comando sobe
três réplicas sem erro (evidências `28` e `29`). O nginx as encontra pelo
nome `web`, que resolve para os três IPs (`31`), e distribui as requisições
entre elas (`30`). As réplicas podem compartilhar o volume `fotos`, porque
cada foto recebe um nome único por UUID e nunca é alterada depois.

---

## 5.1 Do ambiente local para a nuvem

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
|---|---|---|---|
| Contêiner `web` | Azure Container Apps | Amazon ECS com AWS Fargate | Cloud Run |
| Contêiner `db` + volume `dados_banco` | Azure Database for PostgreSQL (Flexible Server) | Amazon RDS for PostgreSQL | Cloud SQL for PostgreSQL |
| Volume `fotos` | Azure Blob Storage | Amazon S3 | Cloud Storage |
| `rede_interna` | Virtual Network (VNet) | Amazon VPC | VPC |
| Arquivo `.env` (segredos) | Azure Key Vault | AWS Secrets Manager | Secret Manager |
| Imagem construída localmente | Azure Container Registry | Amazon ECR | Artifact Registry |

A peça que mais muda é o volume `fotos`, porque a migração exige alterar o
código da aplicação, e não apenas a configuração. Hoje o `app.py` grava as
imagens com `foto.save()` e as entrega com `send_from_directory`, tratando o
volume como uma pasta comum do sistema de arquivos. Um serviço de
armazenamento de objetos, como o S3, não é um sistema de arquivos: é
acessado por uma API HTTP com credenciais próprias, então a gravação
passaria a usar o SDK do provedor, e a leitura normalmente seria feita
direto do serviço, por uma URL assinada e temporária. As outras peças migram
com pouco atrito: o banco continua falando PostgreSQL e muda só o endereço,
os segredos passam a ser injetados por um cofre gerenciado, e a rede vira
uma VPC que mantém o mesmo princípio, com o banco isolado e a aplicação como
único ponto de entrada.
