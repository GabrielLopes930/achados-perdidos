# Relatório técnico — Achados & Perdidos

> Documento voltado à próxima equipe, que fará a migração do ambiente
> para a nuvem. Explica o que cada peça faz e por que foi construída
> dessa forma.

---

## (a) O Dockerfile, instrução por instrução

O arquivo está em `app/Dockerfile` e gera a imagem do serviço `web`.

### `FROM python:3.12-slim`

Define a imagem base. Partir de uma imagem oficial do Python evita montar
um sistema do zero e garante que o interpretador já venha configurado.

A variante `slim` traz apenas o necessário para executar Python, sem
compiladores e ferramentas de build. Isso reduz o tamanho final da imagem
e, com menos pacotes instalados, reduz também a superfície de ataque —
menos software significa menos vulnerabilidades potenciais a corrigir.

### `ENV PYTHONDONTWRITEBYTECODE=1` e `PYTHONUNBUFFERED=1`

A primeira impede o Python de gravar arquivos `.pyc`. Dentro de um
contêiner esses arquivos não trazem ganho prático e apenas ocupam espaço
na camada.

A segunda desliga o buffer da saída padrão. Sem ela, as mensagens do
Gunicorn e do Flask podem ficar retidas em memória e demorar a aparecer —
ou se perder caso o contêiner seja encerrado. É essa variável que faz o
`docker compose logs web` mostrar os eventos no momento em que acontecem,
o que é essencial para diagnosticar falhas.

### `WORKDIR /app`

Define o diretório de trabalho dentro da imagem. Todas as instruções
seguintes passam a ter esse caminho como referência, e é nele que o
contêiner começa a executar. Também é o caminho que aparece nas montagens
do `docker-compose.yml` (`/app/uploads`).

### `COPY requirements.txt .` e `RUN pip install --no-cache-dir -r requirements.txt`

Copia a lista de dependências e as instala: Flask, psycopg e Gunicorn.

A opção `--no-cache-dir` descarta os arquivos que o pip normalmente
guardaria para reaproveitar em instalações futuras. Dentro de uma imagem
esse cache nunca é reutilizado — a instalação acontece uma única vez, no
build — então mantê-lo apenas aumentaria o tamanho da imagem.

### `RUN useradd ... && mkdir -p /app/uploads && chown -R appuser:appuser /app`

Atende ao RNF04. Por padrão, processos dentro de um contêiner rodam como
root; se a aplicação for comprometida, o atacante começa com privilégio
máximo dentro daquele ambiente. Criar um usuário comum limita o estrago.

A mesma instrução cria o diretório de uploads e transfere a posse de
`/app` para esse usuário. **A ordem aqui é decisiva**, e vale detalhar o
porquê: quando um volume nomeado é montado pela primeira vez sobre um
diretório que já existe na imagem, o Docker copia para o volume tanto o
conteúdo quanto a propriedade daquele diretório. Como `/app/uploads` já
pertence ao `appuser` no momento da montagem, o volume `fotos` nasce
pertencendo a ele — e a aplicação, rodando sem privilégios, consegue
gravar as imagens. Se o diretório ainda fosse do root nesse instante, o
volume herdaria essa posse e todo `POST /itens` com foto falharia por
falta de permissão.

### `COPY --chown=appuser:appuser app.py .`

Copia o código da aplicação já com o dono correto, evitando uma instrução
`chown` adicional (que criaria mais uma camada na imagem).

### `USER appuser`

A partir deste ponto, tudo roda sem privilégios de root — inclusive o
processo do Gunicorn definido no `CMD`. É o que se comprova executando
`docker compose exec web whoami`, que responde `appuser`.

### `EXPOSE 5000`

Documenta que a aplicação escuta na porta 5000 dentro do contêiner. É uma
declaração informativa: não publica a porta no host nem altera o
comportamento da rede. Quem decide a publicação é a chave `ports` do
`docker-compose.yml`, e neste ambiente o `web` não a possui: a porta 5000
fica visível apenas dentro da rede interna, onde o `nginx` a alcança.

### `CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]`

Define o comando padrão do contêiner, na forma *exec* (com colchetes).
Nessa forma o Gunicorn é executado diretamente, tornando-se o processo
PID 1 do contêiner e recebendo os sinais enviados pelo Docker. Isso
permite um encerramento limpo em um `docker compose stop`. Na forma
alternativa (texto simples), o comando seria envolvido por um shell, que
ocuparia o PID 1 e poderia não repassar os sinais, fazendo o contêiner ser
encerrado à força após o tempo limite.

O argumento `app:app` indica o objeto `app` dentro do arquivo `app.py`. O
endereço `0.0.0.0` faz o Gunicorn aceitar conexões vindas de qualquer
interface do contêiner — é o que permite ao `nginx`, que está em outro
contêiner, alcançá-lo pela rede interna. Se escutasse apenas em
`127.0.0.1`, só aceitaria conexões originadas de dentro do próprio
contêiner, e o proxy receberia erro `502`.

### Sobre a ordem escolhida

As instruções seguem uma lógica de *menos volátil para mais volátil*: base
e variáveis de ambiente quase nunca mudam, dependências mudam raramente,
código muda a toda alteração. O Docker constrói a imagem em camadas e
reaproveita o cache até encontrar a primeira que mudou — com essa ordem, o
caso comum (alterar o código) reaproveita a instalação das dependências.
A análise detalhada está na questão Q5 do `REFLEXAO.md`.

### O `.dockerignore`

Impede que `.env`, `__pycache__/` e arquivos `.pyc` entrem no contexto de
build. Além de deixar o build mais rápido, tem efeito direto de segurança:
sem essa exclusão, um `COPY . .` descuidado poderia embutir o arquivo de
senhas dentro da imagem, que é distribuível e inspecionável.

---

## (b) O docker-compose.yml, bloco por bloco

### Serviço `db`

```yaml
image: postgres:16-alpine
```

Usa a imagem oficial do PostgreSQL, sem construção própria — o banco não
precisa de nenhuma customização. A variante `alpine` é a mais enxuta.

```yaml
environment:
  POSTGRES_DB: ${POSTGRES_DB}
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

A imagem oficial lê essas três variáveis na primeira inicialização para
criar o banco e o usuário. A sintaxe `${...}` busca o valor no arquivo
`.env`, que não é versionado — nenhuma senha aparece escrita no compose
(RNF03).

```yaml
volumes:
  - dados_banco:/var/lib/postgresql/data
  - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
```

A primeira linha é um **volume nomeado** montado no diretório de dados do
PostgreSQL. É o que faz os registros sobreviverem à remoção do contêiner.

A segunda é um **bind mount**: o arquivo `init.sql` do repositório é
exposto dentro do contêiner, em modo somente leitura (`:ro`). A imagem
oficial do Postgres executa automaticamente todo script colocado em
`/docker-entrypoint-initdb.d/` — mas apenas quando o diretório de dados
está vazio, ou seja, na primeira inicialização. O `:ro` garante que o
contêiner não consiga alterar um arquivo que pertence ao repositório.

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
  interval: 5s
  timeout: 5s
  retries: 10
  start_period: 10s
```

O `pg_isready` é um utilitário do próprio PostgreSQL que pergunta ao
servidor se ele já aceita conexões. O Docker executa esse teste de 5 em 5
segundos (`interval`), aguarda no máximo 5 segundos por resposta
(`timeout`) e tolera até 10 falhas seguidas (`retries`) antes de marcar o
serviço como doente. O `start_period` concede uma folga inicial, durante a
qual as falhas não são contabilizadas — reconhecendo que todo banco leva
alguns segundos para inicializar.

O resultado aparece como `(healthy)` na saída do `docker compose ps`, e é
essa informação que o serviço `web` aguarda.

```yaml
networks: [rede_interna]
restart: unless-stopped
```

Participa da rede interna e, com `unless-stopped`, é reiniciado
automaticamente se falhar ou se a máquina for reiniciada — mas **não** se
tiver sido parado explicitamente por um `docker compose stop`. Essa
diferença em relação ao `always` respeita a intenção de quem operou o
ambiente.

Não há chave `ports`: o banco não publica porta nenhuma no host.

### Serviço `web`

```yaml
build: ./app
```

Em vez de uma imagem pronta, constrói a imagem a partir do Dockerfile na
pasta `app`. É isso que caracteriza a "imagem própria" pedida no enunciado.

```yaml
environment:
  DB_HOST: db
  DB_PORT: "5432"
  DB_NAME: ${POSTGRES_DB}
  DB_USER: ${POSTGRES_USER}
  DB_PASSWORD: ${POSTGRES_PASSWORD}
  UPLOAD_DIR: /app/uploads
```

Os nomes vêm do `app.py`, que lê exatamente essas variáveis. Vale destacar
um ponto que costuma causar confusão: **os nomes são diferentes dos usados
pelo serviço `db`** (`DB_NAME` aqui, `POSTGRES_DB` lá), mas os valores são
os mesmos, vindos das mesmas variáveis do `.env`. O banco usa seus nomes
para *criar* as credenciais; a aplicação usa os dela para *se conectar*
com elas. Se esses valores divergirem, a conexão falha com
`password authentication failed`.

`DB_HOST: db` é o nome do serviço, não um endereço IP — ver Q1.

**Não há chave `ports`.** O `web` escuta na porta 5000 apenas dentro da
`rede_interna` e não pode ser alcançado a partir do host: quem recebe o
tráfego externo é o `nginx`, descrito adiante. Essa ausência também é o que
permite escalar o serviço com `--scale web=3`, já que nenhuma réplica
disputa uma porta do host (evidências `28` e `29`).

```yaml
volumes:
  - fotos:/app/uploads
```

Volume nomeado onde as imagens enviadas são gravadas. O caminho coincide
com o `UPLOAD_DIR` da aplicação e com o diretório criado no Dockerfile.

```yaml
depends_on:
  db:
    condition: service_healthy
```

Atende ao RNF05. A forma simples de `depends_on` apenas ordena a
inicialização; com `condition: service_healthy`, o `web` só começa a subir
depois que o healthcheck do banco passa. Ver Q4.

### Serviço `nginx` (bônus: proxy reverso)

```yaml
image: nginx:alpine
ports:
  - "8080:80"
volumes:
  - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
depends_on:
  - web
```

Usa a imagem oficial do nginx, sem construção própria. É o **único serviço
com porta publicada**: o formato `host:contêiner` faz o que chega na porta
8080 da máquina ser entregue na porta 80 do contêiner, onde o nginx escuta.
Ele então repassa cada requisição ao `web`, na porta 5000. A configuração
fica no repositório e é montada como bind mount somente leitura,
substituindo a configuração padrão da imagem.

O arquivo `nginx/nginx.conf` tem duas decisões que vão além do repasse
simples:

- `client_max_body_size 10m`: por padrão, o nginx recusa corpos de
  requisição maiores que 1 MB, com erro `413`. Fotos de celular costumam
  passar disso, e o limite foi elevado para que o `POST /itens` com foto
  continue funcionando.
- `resolver 127.0.0.11 valid=10s` combinado com `proxy_pass $upstream`: por
  padrão, o nginx resolve o nome `web` uma única vez, ao iniciar, e guarda o
  IP. Se o `web` fosse recriado (e mudasse de IP), o proxy responderia
  `502`, e réplicas criadas depois com `--scale` não seriam vistas. Com o
  resolver apontado para o DNS interno do Docker e o destino numa variável,
  o nome volta a ser consultado a cada 10 segundos. Nos testes, o nginx
  passou a distribuir as requisições entre três réplicas sem ser reiniciado
  (evidências `30` e `31`).

### Serviço `adminer` (bônus: painel do banco)

```yaml
image: adminer
profiles: ["debug"]
ports:
  - "127.0.0.1:8081:8080"
environment:
  ADMINER_DEFAULT_SERVER: db
```

A chave `profiles` faz o serviço ser ignorado por padrão: ele só é criado
quando o profile é ativado com `docker compose --profile debug up -d`
(evidências `38` e `40`). Isso separa uma ferramenta de manutenção, útil
para inspecionar o banco, do ambiente que roda normalmente.

A porta tem uma particularidade: o endereço `127.0.0.1` antes do mapeamento
restringe a publicação à interface local do host. O adminer permite ler e
alterar qualquer dado do banco, e por isso não pode ficar acessível a outras
máquinas da rede. Ele usa a 8081 para não disputar a 8080, que é do nginx.
`ADMINER_DEFAULT_SERVER` apenas preenche a tela de login com o nome `db`,
resolvido pelo mesmo DNS interno usado pelos outros serviços.

### Bloco `networks`

```yaml
networks:
  rede_interna:
    driver: bridge
```

Cria uma rede bridge definida pelo usuário, compartilhada pelos três
serviços. A diferença fundamental em relação à bridge padrão do Docker é
que, em uma rede definida pelo usuário, existe um servidor DNS embutido
que resolve o nome de cada serviço para o IP do contêiner correspondente.

### Bloco `volumes`

```yaml
volumes:
  dados_banco:
  fotos:
```

Declara os dois volumes nomeados. Declará-los aqui é o que os cria; montá-
los dentro de um serviço é um passo distinto. Por serem gerenciados pelo
Docker e viverem fora do ciclo de vida dos contêineres, os dados
sobrevivem a um `docker compose down` (RNF02).

---

## (c) Caminho completo de um `POST /itens` com foto

Percurso de uma requisição desde o `curl` até o disco:

```
curl -F descricao="Oculos de grau" -F local="Bloco B" -F foto=@oculos.jpg \
     http://localhost:8080/itens
```

1. **O cliente monta a requisição.** O `curl` constrói um `POST` com corpo
   `multipart/form-data`, formato que permite transportar campos de texto e
   um arquivo binário na mesma requisição. O destino é `localhost:8080`.

2. **O host entrega à porta publicada.** O pacote chega à porta 8080 da
   máquina, onde o Docker mantém um encaminhamento declarado pela chave
   `ports` do serviço `nginx`.

3. **O Docker encaminha para dentro da rede.** O tráfego atravessa a
   fronteira para a `rede_interna` e chega à porta 80 do contêiner `nginx`.

4. **O nginx repassa ao `web`.** Ele confere que o corpo da requisição está
   dentro do limite de 10 MB, consulta o DNS interno para resolver o nome
   `web` e abre uma nova conexão HTTP com a porta 5000 desse contêiner. Se
   houver várias réplicas, escolhe uma delas. Os cabeçalhos `X-Real-IP` e
   `X-Forwarded-For` levam junto o endereço do cliente original.

5. **O Gunicorn recebe e entrega a um worker.** Ele aceita a conexão,
   interpreta a requisição HTTP e a repassa à aplicação Flask pelo padrão
   WSGI, executando como o usuário `appuser`.

6. **O Flask roteia.** O método `POST` na rota `/itens` corresponde à
   função `cadastrar()` do `app.py`.

7. **A aplicação valida os campos.** Lê `descricao` e `local`. Se algum
   estiver ausente, responde `400` e o percurso termina aqui.

8. **A aplicação gera um nome único para a foto.** Extrai a extensão do
   arquivo original e monta um nome novo com `uuid4().hex`. Isso evita
   colisões entre arquivos de mesmo nome e impede que o nome enviado pelo
   usuário influencie o caminho de gravação.

9. **A foto é gravada no volume.** O `foto.save()` escreve em
   `/app/uploads/<uuid>.jpg`. Esse caminho é o ponto de montagem do volume
   nomeado `fotos`, então os bytes não vão para a camada gravável do
   contêiner: vão para o volume, administrado pelo Docker, fora do ciclo de
   vida do contêiner. É neste passo que a persistência acontece.

10. **A aplicação abre conexão com o banco.** O `psycopg.connect()` usa a
   string montada a partir das variáveis de ambiente, com host `db`. O DNS
   interno da rede traduz `db` para o IP atual do contêiner do banco.

11. **O comando SQL é executado.** Um `INSERT` grava `descricao`, `local` e
    o **nome do arquivo** na tabela `itens`, com `RETURNING id` para obter o
    identificador gerado. Note que o banco armazena apenas a referência ao
    arquivo, nunca a imagem em si.

12. **O PostgreSQL persiste no seu volume.** Os dados são gravados em
    `/var/lib/postgresql/data`, ponto de montagem do volume `dados_banco`.

13. **A resposta percorre o caminho inverso.** A aplicação devolve um JSON
    com `id` e `foto` e o código `201 Created`, que volta pelo Gunicorn,
    pela porta 5000, pelo nginx, pelo encaminhamento do Docker e pela porta
    8080 até o `curl`.

Resumindo a divisão: **o arquivo vai para o volume `fotos`, e apenas o seu
nome vai para o banco** — que por sua vez persiste no volume
`dados_banco`. Uma requisição toca os dois volumes, cada um com um papel.

---

## (d) Ciclo de vida dos volumes

> Esta seção consolida as evidências da Tarefa 5. Os arquivos citados
> estão em `docs/evidencias/` e descritos em `LEGENDAS.md`.

Antes dos testes, foram cadastrados dois objetos com foto (`06` e `07`),
somando-se aos três itens de exemplo do `init.sql`: cinco itens e duas
fotos no total (`08`).

### Situação 1 — Recriação dos contêineres (`down` + `up -d`)

| Momento | `itens_no_banco` | `fotos_no_volume` |
|---|---|---|
| Antes do `down` (`12`) | 5 | 2 |
| Depois do `up -d` (`15`) | 5 | 2 |

**Resultado:** os valores permanecem iguais. A listagem após a recriação
(`16`) traz os dois cadastros com o caminho de suas fotos. A saída do
`down` (`13`) mostra que os contêineres e a rede foram removidos, mas nenhum
volume aparece na lista de remoção.

**Por quê:** o `docker compose down` remove os contêineres e a rede, mas
não toca nos volumes nomeados. Os contêineres são descartáveis; os dados
vivem fora deles. Quando novos contêineres sobem, os mesmos volumes são
montados nos mesmos caminhos, e a aplicação encontra tudo onde esperava.

### Situação 2 — Inspeção dos volumes

Os comandos `docker volume ls` e `docker volume inspect` mostram onde os
dados ficam fisicamente: o campo `Mountpoint` aponta para um diretório sob
a área administrada pelo Docker no host — tipicamente
`/var/lib/docker/volumes/<nome>/_data`. Em Docker Desktop no Windows, esse
caminho fica dentro da máquina virtual utilizada pelo Docker, e não é
navegável diretamente pelo Explorador de Arquivos.

A listagem com `docker compose exec web ls -l /app/uploads` mostra os
arquivos de foto com os nomes gerados por UUID. Vale observar na saída que
os arquivos pertencem ao usuário `appuser`, confirmando que a gravação
ocorreu sem privilégios de root.

| Volume | Caminho no host (`Mountpoint`) | Conteúdo |
|---|---|---|
| `achados-perdidos_dados_banco` (`18`) | `/var/lib/docker/volumes/achados-perdidos_dados_banco/_data` | arquivos de dados do PostgreSQL |
| `achados-perdidos_fotos` (`19`) | `/var/lib/docker/volumes/achados-perdidos_fotos/_data` | imagens nomeadas por UUID |

O nome de cada volume recebe o prefixo `achados-perdidos_`, que é o nome do
projeto Compose (a pasta). Isso evita conflito com volumes de outros
projetos na mesma máquina: na listagem `17` aparecem também volumes de
outros projetos, cada um com seu próprio prefixo.

A listagem `20` mostra as duas fotos dentro de `/app/uploads`, ambas com
dono e grupo `appuser`. Isso confirma, na prática, a decisão do Dockerfile
descrita na seção (a): o volume herdou a propriedade do diretório criado na
imagem, e o processo sem privilégios conseguiu gravar.

### Situação 3 — Remoção dos volumes (`down -v`)

| Momento | `itens_no_banco` | `fotos_no_volume` |
|---|---|---|
| Antes do `down -v` (`23`) | 5 | 2 |
| Depois do `up -d` (`26`) | **3** | **0** |

Desta vez, a saída do `down -v` (`24`) mostra explicitamente os dois
volumes sendo removidos (`Volume ... Removed`), e a do `up` (`25`) mostra
os dois sendo criados de novo. A listagem `27` traz apenas os três itens
do `init.sql`, com data de criação nova: foram reinseridos nesta subida.

**Resultado:** os objetos cadastrados durante os testes desaparecem, as
fotos somem, e o banco volta a conter exatamente os três itens de exemplo
do `init.sql`.

**Por quê:** a opção `-v` remove também os volumes nomeados. O volume
`fotos` é recriado vazio — daí a contagem zero. O volume `dados_banco`
também é recriado vazio, e é justamente isso que faz o `init.sql` voltar a
ser executado: a imagem do PostgreSQL só roda os scripts de inicialização
quando encontra o diretório de dados vazio. Por isso os três itens de
exemplo reaparecem, enquanto os cadastrados durante o teste não.

Esse contraste entre a Situação 1 e a Situação 3 é a demonstração mais
clara do papel dos volumes: **o que distingue dado efêmero de dado
persistente não é o contêiner, é o volume.**

### Backup e restauração do volume `fotos` (bônus)

O backup usa um contêiner temporário do Alpine, criado com `docker run --rm`
e removido ao terminar. Ele monta o volume `fotos` como somente leitura em
`/origem` e a pasta `backup/` do projeto em `/backup`, e compacta o conteúdo
com `tar czvf` (evidência `33`). A listagem do arquivo (`34`) mostra a foto
com dono `1000/1000`, o UID do `appuser`.

Para a restauração, foi criado um volume novo, `fotos_restaurado`, e um
segundo contêiner temporário extraiu o arquivo dentro dele (`35`). A
listagem do volume restaurado (`36`) mostra a mesma foto, com o mesmo nome
UUID, o mesmo tamanho (27.894 bytes) e o mesmo dono. Como o `tar` preserva
a propriedade, esse volume poderia substituir o original sem nenhum ajuste
de permissão.

O ponto central é que o backup não depende da aplicação nem do banco: como
o volume é independente dos contêineres, qualquer contêiner pode montá-lo e
copiar seus dados. A pasta `backup/` foi incluída no `.gitignore`, porque o
arquivo contém as fotos, que podem mostrar documentos pessoais.

### Um caso real observado durante os testes

Durante a Situação 1, a aplicação respondeu `500` logo após o `up -d`, e o
`/health` acusou `password authentication failed for user "achados_user"`.
A senha no `.env` havia sido alterada depois da criação do volume
`dados_banco`.

A causa ilustra o mesmo mecanismo da Situação 3, pelo lado oposto. O
contêiner `web` recriado leu a senha nova do `.env`. O PostgreSQL, porém,
encontrou o diretório de dados já inicializado e **ignorou** as variáveis
`POSTGRES_*`, mantendo a senha gravada na primeira execução. As duas pontas
passaram a usar senhas diferentes.

A correção foi alinhar a senha do usuário no banco com a do `.env`, sem
apagar dados, executando dentro do contêiner do banco:

```
docker compose exec db psql -U achados_user -d postgres -c "ALTER USER achados_user PASSWORD '<senha do .env>';"
```

Os dados nunca estiveram em risco: logo após a correção, as estatísticas
voltaram a mostrar cinco itens e duas fotos. As evidências finais foram
coletadas depois, a partir de um ambiente zerado.

A lição para a operação é que **o `.env` define as credenciais apenas na
criação do banco**. Trocar a senha de um banco existente exige alterá-la no
próprio banco, e não só no arquivo.

### Implicação para a migração

Na nuvem, essa separação continua valendo, com nomes diferentes. O volume
do banco corresponde ao armazenamento gerenciado de um serviço de banco de
dados; o volume de fotos corresponde a um serviço de armazenamento de
objetos. Em ambos os casos o princípio é o mesmo: a instância de computação
é substituível, o armazenamento é que precisa de garantias de durabilidade
e backup. A tabela de equivalência está em `REFLEXAO.md`.
