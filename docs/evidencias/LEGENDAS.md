# Legendas das evidências

Todas as saídas foram geradas em 19/10/2026, no PowerShell do Windows com
Docker Desktop, a partir de um ambiente zerado (`docker compose down -v`) e
já com o proxy reverso `nginx` na frente do `web`. Cada arquivo `.txt`
contém a saída integral do comando indicado.

## Tarefa 4 — Subida e testes funcionais

| Arquivo | Legenda |
|---|---|
| `01-up-build.txt` | `docker compose up -d --build`: build da imagem própria (camadas em cache) e subida dos três serviços. |
| `02-compose-ps.txt` | `docker compose ps`: `db` healthy; só o `nginx` publica porta (`8080->80`); o `web` mostra apenas `5000/tcp`. |
| `03-health.txt` | `curl /health`: status ok e banco conectado (RF01). |
| `04-itens.txt` | `curl /itens`: os três itens de exemplo inseridos pelo `init.sql` (RF02). |
| `05-itens-filtro.txt` | `curl /itens?local=biblio`: filtro parcial por local retorna só o item da Biblioteca (RF02). |
| `06-cadastro-foto.txt` | `curl -F ... POST /itens`: cadastro com foto, retorna id 4 e o nome UUID do arquivo (RF03). |
| `07-cadastro-foto2.txt` | Segundo cadastro com foto (id 5), preparando a prova de persistência. |
| `08-estatisticas.txt` | `curl /estatisticas`: 5 itens no banco e 2 fotos no volume (RF04). |
| `09-logs-web.txt` | `docker compose logs web --tail 20`: Gunicorn escutando em `0.0.0.0:5000`. |
| `10-whoami.txt` | `docker compose exec web whoami`: retorna `appuser`, não root (RNF04). |
| `11-foto-navegador.png` | Foto do id 4 aberta no navegador pela URL `/fotos/264ead17...jpg`, servida através do nginx. |

## Tarefa 5 — Situação 1: recriação dos contêineres

| Arquivo | Legenda |
|---|---|
| `12-estat-antes-down.txt` | Estatísticas antes do `down`: 5 itens, 2 fotos. |
| `13-down.txt` | `docker compose down`: remove contêineres e rede; nenhum volume é removido. |
| `14-up.txt` | `docker compose up -d`: o `db` passa por `Waiting` e `Healthy` antes de o `web` iniciar. |
| `15-estat-depois-up.txt` | Estatísticas depois do `up`: 5 itens, 2 fotos, iguais ao `12` (RNF02). |
| `16-itens-depois-up.txt` | Listagem após a recriação: os dois cadastros continuam, com as fotos. |

## Tarefa 5 — Situação 2: inspeção dos volumes

| Arquivo | Legenda |
|---|---|
| `17-volume-ls.txt` | `docker volume ls`: os volumes `dados_banco` e `fotos` com o prefixo do projeto. |
| `18-inspect-dados-banco.txt` | `docker volume inspect` do volume do banco: `Mountpoint` em `/var/lib/docker/volumes/.../_data`. |
| `19-inspect-fotos.txt` | `docker volume inspect` do volume de fotos: `Mountpoint` em `/var/lib/docker/volumes/.../_data`. |
| `20-ls-uploads.txt` | `ls -l /app/uploads` no `web`: as duas fotos, com dono `appuser`. |

## Evidências extras (artefato A4)

| Arquivo | Legenda |
|---|---|
| `21-network-inspect.txt` | `docker network inspect`: `db` (172.21.0.2), `web` (172.21.0.3) e `nginx` (172.21.0.4) na `rede_interna`, driver bridge. |
| `22-image-ls.txt` | `docker image ls`: a imagem `achados-perdidos-web` ocupa 206 MB em disco (50,4 MB comprimida). |

## Tarefa 5 — Situação 3: remoção dos volumes

| Arquivo | Legenda |
|---|---|
| `23-estat-antes-down-v.txt` | Estatísticas antes do `down -v`: 5 itens, 2 fotos. |
| `24-down-v.txt` | `docker compose down -v`: remove contêineres, rede **e os dois volumes**. |
| `25-up-depois-down-v.txt` | `docker compose up -d`: volumes recriados vazios. |
| `26-estat-depois-down-v.txt` | Estatísticas após `down -v`: 3 itens (os do `init.sql`) e 0 fotos. |
| `27-itens-depois-down-v.txt` | Listagem após `down -v`: apenas os três itens de exemplo, reinseridos pelo `init.sql`. |

## Bônus — proxy reverso e escala horizontal

| Arquivo | Legenda |
|---|---|
| `28-scale-web-3.txt` | `docker compose up -d --scale web=3`: duas réplicas novas criadas, sem conflito de porta. |
| `29-ps-scale.txt` | `docker compose ps`: três réplicas do `web` só com `5000/tcp`; a 8080 continua apenas no `nginx`. |
| `30-balanceamento.txt` | Seis chamadas ao `/health` atendidas por três contêineres diferentes: o nginx distribui entre as réplicas. |
| `31-nslookup-web.txt` | `nslookup web` no DNS interno (127.0.0.11): o nome `web` resolve para os três IPs das réplicas. |

## Bônus — backup e restauração do volume `fotos`

| Arquivo | Legenda |
|---|---|
| `32-cadastro-para-backup.txt` | Cadastro de um item com foto, para que o volume tenha conteúdo a ser copiado. |
| `33-backup.txt` | `docker run --rm` com o volume montado em `:ro`: o `tar` compacta a foto em `backup/fotos-backup.tar.gz`. |
| `34-conteudo-backup.txt` | Listagem do arquivo de backup: a foto, com dono `1000/1000` (o `appuser`). |
| `35-restauracao.txt` | Criação do volume `fotos_restaurado` e extração do backup dentro dele, com outro contêiner temporário. |
| `36-volume-restaurado.txt` | Conteúdo do volume restaurado: a mesma foto, com o mesmo nome, tamanho e dono. |

## Bônus — painel do banco (adminer)

| Arquivo | Legenda |
|---|---|
| `37-up-sem-profile.txt` | `docker compose up -d` comum: o `adminer` não é iniciado. |
| `38-ps-sem-profile.txt` | `docker compose ps`: apenas `db`, `web` e `nginx`. |
| `39-up-profile-debug.txt` | `docker compose --profile debug up -d`: baixa a imagem e inicia o `adminer`. |
| `40-ps-profile-debug.txt` | `docker compose --profile debug ps`: o `adminer` aparece, publicado só em `127.0.0.1:8081`. |
| `41-adminer.png` | Painel do adminer conectado ao banco, exibindo a tabela `itens`. |

