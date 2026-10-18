# Arquitetura: Achados & Perdidos (A2)

## Diagrama

```mermaid
flowchart TB
    cliente["Navegador / curl<br/>(na máquina host)"]

    subgraph HOST["Máquina host"]
        direction TB

        p8080(["porta publicada<br/>0.0.0.0:8080"])
        p8081(["porta publicada<br/>127.0.0.1:8081<br/>(só profile debug)"])

        subgraph BORDA["rede_borda · bridge definida pelo usuário"]
            nginx["<b>nginx</b><br/>nginx:1.27-alpine<br/>escuta :80"]
        end

        subgraph INTERNA["rede_interna · bridge definida pelo usuário (fora do alcance do host)"]
            web["<b>web</b><br/>achados-perdidos-web:1.0<br/>Gunicorn :5000 · usuário app (uid 10001)<br/>também conectado à rede_borda"]
            db[("<b>db</b><br/>postgres:16-alpine<br/>:5432 · sem porta no host")]
            adminer["<b>adminer</b> (profile debug)<br/>adminer:4.8.1 · :8080"]
        end

        vfotos[("volume nomeado <b>fotos</b><br/>montado em /app/uploads")]
        vdados[("volume nomeado <b>dados_banco</b><br/>montado em /var/lib/postgresql/data")]
        binit["bind mount (somente leitura)<br/>./db/init.sql →<br/>/docker-entrypoint-initdb.d/init.sql"]
        bconf["bind mount (somente leitura)<br/>./nginx/default.conf →<br/>/etc/nginx/conf.d/default.conf"]
    end

    cliente -->|"HTTP localhost:8080"| p8080 --> nginx
    cliente -.->|"HTTP 127.0.0.1:8081 (debug)"| p8081 -.-> adminer
    nginx -->|"proxy_pass → web:5000<br/>(DNS do Docker, via rede_borda)"| web
    web -->|"SQL · host db:5432"| db
    adminer -.->|"SQL · host db:5432"| db
    web ---|"grava/lê fotos"| vfotos
    db ---|"arquivos do banco"| vdados
    binit -.->|"executado só na 1ª subida<br/>(diretório de dados vazio)"| db
    bconf -.-> nginx
```

## Quem fica dentro e fora de cada rede

| Contêiner | rede_borda | rede_interna | Porta no host | Observação |
|-----------|:----------:|:------------:|---------------|------------|
| nginx     | sim | **não** | `0.0.0.0:8080 → 80` | Único ponto de entrada. Não enxerga o banco. |
| web       | sim | sim | nenhuma (`expose: 5000`) | Faz a ponte entre o proxy e o banco. |
| db        | **não** | sim | nenhuma | Alcançável só por `web` e `adminer`, pelo nome `db`. |
| adminer   | não | sim | `127.0.0.1:8081 → 8080` | Só existe com `--profile debug` e só aceita conexões da própria máquina. |

## Armazenamento

| Recurso | Tipo | Origem (host) | Destino (contêiner) | Ciclo de vida |
|---------|------|---------------|---------------------|---------------|
| `fotos` | volume nomeado | gerenciado pelo Docker (`/var/lib/docker/volumes/achados-perdidos_fotos/_data` na VM) | `web:/app/uploads` | Sobrevive a `down`; apagado por `down -v`. |
| `dados_banco` | volume nomeado | gerenciado pelo Docker (`/var/lib/docker/volumes/achados-perdidos_dados_banco/_data`) | `db:/var/lib/postgresql/data` | Sobrevive a `down`; apagado por `down -v`. |
| `init.sql` | bind mount `:ro` | `./db/init.sql` (repositório) | `db:/docker-entrypoint-initdb.d/init.sql` | Versionado no Git; lido só quando o diretório de dados está vazio. |
| `default.conf` | bind mount `:ro` | `./nginx/default.conf` | `nginx:/etc/nginx/conf.d/default.conf` | Versionado no Git. |

## Relação com a arquitetura base do enunciado

O enunciado descreve `web` publicando `8080:5000`. Com o **bônus de proxy reverso**, o `web` deixou
de publicar portas e o nginx passou a ocupar a 8080 do host. A `rede_interna`
continua sendo a rede compartilhada por `web` e `db`, como exigido. A `rede_borda` foi acrescentada para
que o nginx, que é o serviço exposto, não tenha nenhuma rota até o PostgreSQL (defesa em profundidade).
As justificativas detalhadas estão em [RELATORIO.md](RELATORIO.md).
