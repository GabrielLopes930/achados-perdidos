# Achados & Perdidos — ambiente conteinerizado

API em Flask para registro de objetos perdidos no campus, empacotada em
contêineres junto com seu banco PostgreSQL e um proxy reverso nginx. Todo
o ambiente sobe com um único comando, e os dados (registros e fotos)
sobrevivem à recriação dos contêineres.

| Serviço | Papel | Porta publicada no host |
|---|---|---|
| `nginx` | proxy reverso, único ponto de entrada | `8080` |
| `web` | API Flask servida pelo Gunicorn (imagem própria) | nenhuma |
| `db` | PostgreSQL | nenhuma |

O diagrama completo está em `docs/arquitetura.md`.

---

## Pré-requisitos

- Docker Engine 20.10+ com Docker Compose v2 (`docker compose`, sem hífen)
- Nenhuma outra instalação: não é preciso ter Python, Flask ou PostgreSQL
  na máquina

Para conferir:

```
docker --version
docker compose version
```

---

## Subindo o ambiente

**1. Crie o arquivo de segredos a partir do modelo.**

```
cp .env.example .env
```

No Windows (PowerShell): `Copy-Item .env.example .env`

Abra o `.env` e troque a senha. O arquivo não é versionado (está no
`.gitignore`), por isso precisa ser criado na primeira execução.

| Variável | Para que serve |
|---|---|
| `POSTGRES_DB` | Nome do banco criado na primeira execução |
| `POSTGRES_USER` | Usuário dono do banco |
| `POSTGRES_PASSWORD` | Senha desse usuário |

As mesmas três variáveis alimentam os dois serviços: o `db` as usa para
**criar** o banco, o `web` as usa para **conectar** nele.

**2. Suba os serviços.**

```
docker compose up -d --build
```

**3. Confirme que está no ar.**

```
docker compose ps
```

Os três serviços devem aparecer como `running`, e o `db` como `healthy`.
Só o `nginx` mostra uma porta publicada (`0.0.0.0:8080->80/tcp`). O `web`
só inicia depois que o banco aceita conexões.

**4. Teste.**

```
curl http://localhost:8080/health
```

Deve responder `"status": "ok"` e `"banco": "conectado"`.

No PowerShell do Windows use `curl.exe` em vez de `curl` — o `curl` do
PowerShell é um apelido para outro comando e não aceita as mesmas opções.

---

## Rotas da API

Todas disponíveis em `http://localhost:8080`.

| Método | Rota | O que faz |
|---|---|---|
| GET | `/health` | Estado da aplicação e da conexão com o banco |
| GET | `/itens` | Lista todos os objetos cadastrados |
| GET | `/itens?local=texto` | Filtra os objetos por local (busca parcial) |
| POST | `/itens` | Cadastra um objeto (`descricao`, `local` e, opcional, `foto`) |
| GET | `/fotos/<nome>` | Devolve o arquivo de foto armazenado |
| GET | `/estatisticas` | Total de itens no banco e de fotos no volume |

### Exemplos

Listar tudo:

```
curl http://localhost:8080/itens
```

Filtrar por local:

```
curl "http://localhost:8080/itens?local=biblio"
```

Cadastrar sem foto:

```
curl -F descricao="Guarda-chuva preto" -F local="Bloco A" http://localhost:8080/itens
```

Cadastrar com foto:

```
curl -F descricao="Oculos de grau" -F local="Bloco B" -F foto=@oculos.jpg http://localhost:8080/itens
```

A resposta traz o nome gerado para o arquivo. A foto fica acessível em
`http://localhost:8080/fotos/<nome>`, e o caminho também aparece no
`GET /itens`.

---

## Operação do dia a dia

| Objetivo | Comando |
|---|---|
| Ver o estado dos serviços | `docker compose ps` |
| Ver os logs da aplicação | `docker compose logs web --tail 20` |
| Acompanhar logs ao vivo | `docker compose logs -f web` |
| Reiniciar um serviço | `docker compose restart web` |
| Parar sem apagar nada | `docker compose stop` |
| Retomar o que foi parado | `docker compose start` |
| Reconstruir após mudar o código | `docker compose up -d --build` |

---

## Parando e destruindo

**Parar e remover os contêineres, preservando os dados:**

```
docker compose down
```

Os volumes `dados_banco` e `fotos` continuam existindo. Ao subir de novo,
os itens e as fotos estarão lá.

**Destruir tudo, inclusive os dados:**

```
docker compose down -v
```

O `-v` remove também os volumes. Os objetos cadastrados e as fotos são
perdidos de forma definitiva, e na próxima subida o `init.sql` roda de
novo, recriando a tabela com os três itens de exemplo.

---

## Estrutura do repositório

```
achados-perdidos/
├── app/
│   ├── app.py              aplicação Flask (fornecida)
│   ├── requirements.txt    dependências (fornecida)
│   ├── Dockerfile          receita da imagem do serviço web
│   └── .dockerignore       o que não entra no contexto de build
├── db/
│   └── init.sql            cria a tabela itens (fornecida)
├── docs/
│   ├── arquitetura.md      diagrama do ambiente
│   ├── RELATORIO.md        relatório técnico
│   ├── REFLEXAO.md         questões de reflexão e equivalência em nuvem
│   └── evidencias/         saídas dos testes e LEGENDAS.md
├── nginx/
│   └── nginx.conf          configuração do proxy reverso
├── docker-compose.yml      orquestração dos serviços
├── .env.example            modelo de configuração (versionado)
├── .env                    segredos reais (NÃO versionado)
├── .gitignore
└── README.md
```

---

## Problemas comuns

**`port is already allocated`** — a porta 8080 já está em uso por outro
processo. Descubra qual com `docker ps` ou troque o mapeamento do serviço
`nginx` no `docker-compose.yml` para `"8081:80"`.

**`502 Bad Gateway`** — o `nginx` está no ar, mas não conseguiu falar com o
`web`. Verifique se o `web` está rodando com `docker compose ps` e veja o
motivo em `docker compose logs web --tail 20`.

**`413 Request Entity Too Large`** — a foto enviada passa do limite de 10 MB
configurado em `client_max_body_size`, no `nginx/nginx.conf`.

**`password authentication failed`** — a senha do `.env` não bate com a
gravada no volume do banco. As variáveis `POSTGRES_*` só são aplicadas
quando o volume é criado vazio; um volume já existente mantém a senha
original. Para aplicar a senha nova, recrie o volume com
`docker compose down -v` (isso apaga os dados).

**`/health` responde 503** — o banco ainda está subindo ou as variáveis de
ambiente não chegaram ao contêiner. Verifique com
`docker compose logs web --tail 20` e `docker compose ps`.

**Erro ao gravar foto (permissão negada)** — o volume `fotos` foi criado
antes de o Dockerfile definir o dono correto do diretório. Recrie com
`docker compose down -v` seguido de `docker compose up -d --build`.

---

## Bônus: proxy reverso e escala horizontal

O serviço `nginx` fica na frente do `web`, que não publica porta nenhuma.
Isso permite rodar várias réplicas da aplicação:

```
docker compose up -d --scale web=3
```

O nome `web` passa a resolver para o IP das três réplicas no DNS interno, e
o nginx distribui as requisições entre elas. Para ver o balanceamento, o
campo `container` do `/health` muda conforme a réplica que atendeu:

```
1..6 | ForEach-Object { curl.exe -s http://localhost:8080/health; "" }
```

Para voltar a uma réplica:

```
docker compose up -d --scale web=1
```

O `nginx.conf` consulta o DNS interno a cada 10 segundos, então réplicas
novas podem levar alguns segundos para começar a receber tráfego.

---

## Bônus: backup e restauração do volume de fotos

O backup é feito por um contêiner temporário (`docker run --rm`), que monta
o volume `fotos` em modo somente leitura e a pasta `backup/` do projeto, e
compacta as fotos em `backup/fotos-backup.tar.gz`. O contêiner é removido
assim que termina.

```
mkdir backup
docker run --rm -v achados-perdidos_fotos:/origem:ro -v "${PWD}/backup:/backup" alpine tar czvf /backup/fotos-backup.tar.gz -C /origem .
```

Para conferir o conteúdo do arquivo sem extraí-lo:

```
docker run --rm -v "${PWD}/backup:/backup:ro" alpine tar tzvf /backup/fotos-backup.tar.gz
```

Para restaurar em um volume novo:

```
docker volume create fotos_restaurado
docker run --rm -v fotos_restaurado:/destino -v "${PWD}/backup:/backup:ro" alpine tar xzvf /backup/fotos-backup.tar.gz -C /destino
```

O `tar` preserva o dono dos arquivos (UID 1000, o `appuser`), então o volume
restaurado pode ser usado pela aplicação sem ajustes de permissão.

A pasta `backup/` está no `.gitignore`: o arquivo contém as fotos, que podem
mostrar documentos pessoais, e não deve ser versionado.

---

## Bônus: painel do banco (adminer)

O serviço `adminer` está no profile `debug` e **não sobe** com o
`docker compose up -d` comum. Para subi-lo junto com o resto:

```
docker compose --profile debug up -d
```

O painel fica em <http://localhost:8081>. Na tela de login, use o sistema
PostgreSQL, o servidor `db` e as credenciais do `.env`.

A porta é publicada apenas em `127.0.0.1`: o painel dá acesso total ao
banco, então só pode ser aberto na própria máquina, nunca por outro
computador da rede.

Para desligar só o painel, mantendo o resto do ambiente no ar:

```
docker compose rm -sf adminer
```
