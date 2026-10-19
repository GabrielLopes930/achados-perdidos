# Diagrama de arquitetura

Ambiente do **Achados & Perdidos**: três serviços, uma rede bridge
definida pelo usuário e dois volumes nomeados. Apenas o proxy reverso
`nginx` é alcançável a partir do host; o `web` e o `db` ficam
exclusivamente na rede interna.

```mermaid
flowchart TB
    subgraph HOST["💻 HOST — máquina do usuário"]
        NAV["Navegador / curl<br/>http://localhost:8080"]
        INIT["db/init.sql<br/><i>arquivo do repositório</i>"]
        CONF["nginx/nginx.conf<br/><i>arquivo do repositório</i>"]
    end

    subgraph REDE["🔒 rede_interna — bridge definida pelo usuário"]
        NGX["<b>nginx</b><br/>nginx:alpine<br/>proxy reverso<br/>escuta na porta 80<br/><b>única porta publicada</b>"]
        WEB["<b>web</b><br/>imagem própria (build ./app)<br/>Flask + Gunicorn<br/>escuta na porta 5000<br/>usuário: appuser (não-root)<br/><b>sem porta publicada</b>"]
        DB["<b>db</b><br/>postgres:16-alpine<br/>escuta na porta 5432<br/><b>sem porta publicada</b>"]
        ADM["<b>adminer</b> · profile debug<br/>painel do banco<br/>escuta na porta 8080<br/>publicada só em 127.0.0.1"]
    end

    VFOTOS[("volume nomeado<br/><b>fotos</b><br/>→ /app/uploads")]
    VDADOS[("volume nomeado<br/><b>dados_banco</b><br/>→ /var/lib/postgresql/data")]

    NAV -->|"publicada<br/>8080 : 80"| NGX
    NGX -->|"HTTP · host \"web\" porta 5000<br/>resolvido pelo DNS interno"| WEB
    WEB -->|"SQL · host \"db\" porta 5432<br/>resolvido pelo DNS interno"| DB
    WEB -.->|"grava e lê as fotos"| VFOTOS
    DB -.->|"grava os dados"| VDADOS
    INIT -.->|"bind mount :ro<br/>/docker-entrypoint-initdb.d/"| DB
    NAV -.->|"só com --profile debug<br/>127.0.0.1:8081 : 8080"| ADM
    ADM -.->|"SQL · host \"db\""| DB
    CONF -.->|"bind mount :ro<br/>/etc/nginx/conf.d/default.conf"| NGX

    classDef publico fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f
    classDef privado fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#7f1d1d
    classDef volume fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#78350f
    classDef opcional fill:#ede9fe,stroke:#7c3aed,stroke-width:2px,stroke-dasharray:5 5,color:#4c1d95
    classDef host fill:#f3f4f6,stroke:#6b7280,stroke-width:1px,color:#374151

    class NGX publico
    class WEB,DB privado
    class ADM opcional
    class VFOTOS,VDADOS volume
    class NAV,INIT,CONF host
```

## Legenda

| Elemento | Dentro da rede interna? | Acessível do host? |
|---|---|---|
| `nginx` | sim | **sim**, pela porta 8080 do host → 80 do contêiner |
| `web` | sim | **não** — nenhuma porta publicada; só o `nginx` o alcança |
| `db` | sim | **não** — nenhuma porta publicada; só o `web` e o `adminer` o alcançam |
| `adminer` | sim | só com `--profile debug`, e apenas da própria máquina (`127.0.0.1:8081`) |
| volume `fotos` | — | não diretamente; só através da aplicação |
| volume `dados_banco` | — | não diretamente; só através do banco |
| `db/init.sql` | — | arquivo do host, montado como bind mount somente leitura |
| `nginx/nginx.conf` | — | arquivo do host, montado como bind mount somente leitura |

Endereços observados na rede interna durante os testes (evidência
`21-network-inspect.txt`): `db` em `172.21.0.2`, `web` em `172.21.0.3` e
`nginx` em `172.21.0.4`. Esses endereços mudam a cada recriação, e por isso
nenhum serviço é configurado com IP: todos se encontram pelo nome.

## Pontos que o diagrama destaca

**Único ponto de entrada.** A única seta que atravessa a fronteira do host
para dentro da rede chega ao `nginx`. Nem o `web` nem o `db` têm porta
publicada, então não existe caminho de rede do host direto até eles. Cada
camada só fala com a seguinte: `nginx` → `web` → `db`.

**Resolução por nome.** As setas internas usam nomes de serviço, não IPs.
Isso só funciona porque os três estão na mesma rede definida pelo usuário,
onde o DNS interno do Docker resolve `web` e `db`. Quando o `web` é
escalado para várias réplicas, o mesmo nome `web` passa a resolver para
todos os IPs, e o `nginx` distribui as requisições entre eles (evidências
`30` e `31`).

**Dois tipos de montagem.** Os volumes nomeados (`fotos`, `dados_banco`)
são gerenciados pelo Docker e guardam dados que precisam sobreviver aos
contêineres. Os bind mounts (`init.sql`, `nginx.conf`) são arquivos de
configuração que vivem no repositório e são apenas expostos para dentro dos
contêineres, em modo somente leitura.

**Fronteira de privilégio.** Dentro do contêiner `web`, o processo roda
como `appuser`, não como root. O volume `fotos` pertence a esse mesmo
usuário, o que permite gravar as imagens sem elevar privilégios.

**Ferramenta opcional.** O `adminer`, em tracejado, só existe quando o
profile `debug` é ativado. Mesmo assim, sua porta é publicada apenas na
interface local do host, para que o acesso total ao banco nunca fique
exposto à rede.
