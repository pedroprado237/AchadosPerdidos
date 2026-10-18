# Achados & Perdidos: ambiente em contêineres

Sistema do Setor de Atendimento ao Estudante para cadastrar objetos encontrados no campus, com foto, e permitir
consultá-los pela web. Este repositório empacota a aplicação (API Flask + PostgreSQL) em contêineres, de modo que
**qualquer pessoa sobe o ambiente completo com um único comando**, sem perder itens nem fotos quando os contêineres
são recriados. É a primeira etapa da migração do sistema para a nuvem.

| Documento | Conteúdo |
|-----------|----------|
| [docs/arquitetura.md](docs/arquitetura.md) | Diagrama (host, portas, contêineres, redes, volumes e bind mounts) |
| [docs/RELATORIO.md](docs/RELATORIO.md) | Explicação do Dockerfile e do Compose, caminho de uma requisição e ciclo de vida dos volumes |
| [docs/REFLEXAO.md](docs/REFLEXAO.md) | Questões de reflexão e equivalência com serviços de nuvem (Azure, AWS, GCP) |
| [docs/evidencias/](docs/evidencias/README.md) | Saídas reais dos testes, com legenda |

## Visão geral

```
Navegador / curl ──HTTP localhost:8080──► nginx ──rede_borda──► web (Flask/Gunicorn :5000) ──rede_interna──► db (PostgreSQL 16)
                                                                   │                                           │
                                                          volume fotos (/app/uploads)        volume dados_banco (/var/lib/postgresql/data)
```

- **nginx**: proxy reverso, único serviço com porta publicada no host (`8080`).
- **web**: API em Python 3.12/Flask servida pelo Gunicorn. Roda como usuário sem privilégios (`app`).
- **db**: `postgres:16-alpine`. Tabela `itens` criada automaticamente na primeira execução pelo `db/init.sql`. **Sem porta no host.**
- **adminer** *(opcional, profile `debug`)*: painel web do banco em `127.0.0.1:8081`.

## Pré-requisitos

- **Docker** com o plugin **Compose v2.20 ou mais recente**: Docker Desktop (Windows/macOS) ou Docker Engine (Linux). Confira com `docker compose version`.
- **Git**, para clonar.
- Porta **8080** livre no host (veja [Solução de problemas](#solução-de-problemas)).

Nada mais precisa ser instalado: Python, PostgreSQL e nginx rodam dentro dos contêineres.

## 1. Configurar o `.env`

As credenciais **não** estão no repositório. Crie o seu `.env` a partir do modelo:

```bash
# Linux / macOS / Git Bash
cp .env.example .env
```
```powershell
# Windows PowerShell
Copy-Item .env.example .env
```

Abra o `.env` e troque `POSTGRES_PASSWORD` por uma senha forte (sem espaços nem aspas). Para gerar uma:

```bash
openssl rand -hex 24
```

| Variável | Uso |
|----------|-----|
| `POSTGRES_DB` | Nome do banco criado na primeira subida (o `web` o recebe como `DB_NAME`) |
| `POSTGRES_USER` | Usuário do banco (o `web` o recebe como `DB_USER`) |
| `POSTGRES_PASSWORD` | Senha do banco (o `web` a recebe como `DB_PASSWORD`) |

> Sem o `.env`, o Compose para com a mensagem `required variable POSTGRES_DB is missing a value: defina POSTGRES_DB no .env`.
> O `.env` está no `.gitignore` e **nunca deve ser commitado**.
>
> As credenciais valem a partir da **primeira** subida. Se você mudar a senha no `.env` depois, o banco
> que já existe no volume continua com a senha antiga (ver [Solução de problemas](#solução-de-problemas)).

## 2. Subir o ambiente

```bash
docker compose up -d --build
```

O Compose constrói a imagem do `web`, cria as redes e os volumes e sobe os serviços na ordem
**db (healthy) → web (healthy) → nginx**. Para acompanhar:

```bash
docker compose ps          # db e web devem aparecer como "(healthy)"
docker compose logs -f web # Ctrl+C para sair
```

## 3. Testar

> No **PowerShell** do Windows, use `curl.exe` em vez de `curl` (no PowerShell 5.1, `curl` é um alias de `Invoke-WebRequest`).

```bash
# Saúde: deve responder "status": "ok" e "banco": "conectado"
curl http://localhost:8080/health

# Listar todos / filtrar por local
curl http://localhost:8080/itens
curl "http://localhost:8080/itens?local=biblio"

# Cadastrar com foto (há imagens de exemplo em docs/amostras/)
curl -F descricao="Oculos de grau" -F local="Bloco B" -F foto=@docs/amostras/oculos.jpg http://localhost:8080/itens

# Totais no banco e no volume
curl http://localhost:8080/estatisticas
```

Para ver a foto, abra no navegador a URL do campo `foto` retornado por `GET /itens`, por exemplo
`http://localhost:8080/fotos/<nome>.jpg`.

Verificações de segurança rápidas:

```bash
docker compose exec web whoami   # app (não root)
docker compose port db 5432      # vazio: banco sem porta no host
```

## 4. Rotas da API

| Método | Rota | Parâmetros | Resposta |
|--------|------|------------|----------|
| `GET` | `/health` | — | `200 {"status":"ok","banco":"conectado","container":"<hostname>"}`; `503` se o banco estiver inacessível |
| `GET` | `/itens` | `?local=<texto>` (opcional, busca parcial sem diferenciar maiúsculas) | `200 [{"id","descricao","local","foto","criado_em"}, …]`, mais recentes primeiro |
| `POST` | `/itens` | `multipart/form-data`: `descricao`* , `local`* , `foto` (arquivo, opcional) | `201 {"id":N,"foto":"<uuid>.<ext>"}`; `400` se faltar campo obrigatório |
| `GET` | `/fotos/<nome>` | nome do arquivo retornado no cadastro | o arquivo de imagem; `404` se não existir |
| `GET` | `/estatisticas` | — | `200 {"itens_no_banco":N,"fotos_no_volume":M}` |

\* obrigatório. O limite de upload é **10 MB** por requisição (definido no nginx).

## 5. Parar e destruir

| Objetivo | Comando | Itens e fotos |
|----------|---------|---------------|
| Pausar (mantém os contêineres) | `docker compose stop` → depois `docker compose start` | preservados |
| Remover contêineres e redes | `docker compose down` → depois `docker compose up -d` | **preservados** (os volumes ficam) |
| Aplicar mudança de código ou Dockerfile | `docker compose up -d --build` | preservados |
| **Destruir tudo, inclusive os dados** | `docker compose down -v` | **apagados**: no próximo `up`, o banco volta só com os 3 itens de exemplo |
| Remover também a imagem construída | `docker compose down -v` e depois `docker image rm achados-perdidos-web:1.0` | apagados |

> ⚠️ `docker compose down -v` e `docker volume prune` apagam o banco e as fotos de forma **irreversível**. Faça backup antes (seção 6).

## 6. Recursos extras

### Painel do banco (adminer)

Só sobe quando o profile `debug` é ativado e fica acessível apenas a partir da própria máquina:

```bash
docker compose --profile debug up -d
# abra http://127.0.0.1:8081 → Sistema: PostgreSQL, Servidor: db, usuário/senha/base do .env
docker compose --profile debug rm -sf adminer   # desliga o painel
```

### Backup e restauração do volume de fotos

Usa um contêiner `alpine` temporário (`--rm`) com o volume montado. O arquivo vai para `backups/`, que não é versionado.

```bash
# Backup (Git Bash/Linux/macOS)
docker run --rm -v achados-perdidos_fotos:/origem:ro -v "$(pwd)/backups:/backup" \
  alpine:3.20 tar czf /backup/fotos-backup.tar.gz -C /origem .

# Restauração em um volume novo
docker volume create achados-perdidos_fotos_restaurado
docker run --rm -v achados-perdidos_fotos_restaurado:/destino -v "$(pwd)/backups:/backup:ro" \
  alpine:3.20 tar xzf /backup/fotos-backup.tar.gz -C /destino
```
```powershell
# Mesmos comandos no PowerShell
docker run --rm -v achados-perdidos_fotos:/origem:ro -v "${PWD}/backups:/backup" alpine:3.20 tar czf /backup/fotos-backup.tar.gz -C /origem .
docker volume create achados-perdidos_fotos_restaurado
docker run --rm -v achados-perdidos_fotos_restaurado:/destino -v "${PWD}/backups:/backup:ro" alpine:3.20 tar xzf /backup/fotos-backup.tar.gz -C /destino
```

> No Git Bash do Windows, use `$(pwd -W)` no lugar de `$(pwd)` e rode antes `export MSYS_NO_PATHCONV=1`,
> para que caminhos como `/origem` não sejam convertidos para `C:/Program Files/Git/...`.

Para restaurar **no volume em uso**: `docker compose down`, depois rode o `tar xzf` com o destino em `achados-perdidos_fotos` e, por fim, `docker compose up -d`.
As fotos referenciam linhas do banco, então um backup completo também precisa de um dump do PostgreSQL:

```bash
# -T desliga o pseudo-TTY, para que a saída redirecionada fique limpa
docker compose exec -T db sh -c 'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB"' > backups/banco.sql
```

> No PowerShell 5.1, o `>` grava em UTF-16. Use `... | Out-File -Encoding utf8 backups/banco.sql`.

### Escalar a API

```bash
docker compose up -d --scale web=3
docker compose restart nginx     # o nginx resolve "web" na inicialização; reinicie para incluir as novas réplicas
```

## Estrutura do repositório

```
achados-perdidos/
├── app/
│   ├── app.py              # aplicação fornecida (não alterada)
│   ├── requirements.txt    # dependências fornecidas
│   ├── Dockerfile          # imagem do serviço web
│   └── .dockerignore
├── db/
│   └── init.sql            # cria a tabela itens + 3 exemplos (só na 1ª subida)
├── nginx/
│   └── default.conf        # proxy reverso (bônus)
├── docs/
│   ├── arquitetura.md      # A2 – diagrama
│   ├── RELATORIO.md        # A3 – relatório técnico
│   ├── REFLEXAO.md         # A5 – reflexão + tabela de nuvem
│   ├── evidencias/         # A4 – saídas dos testes
│   └── amostras/           # imagens usadas nos testes de upload
├── backups/                # destino dos backups (conteúdo não versionado)
├── docker-compose.yml
├── .env.example            # modelo sem senha real
├── .gitignore
├── .gitattributes
└── README.md
```

## Solução de problemas

| Sintoma | Causa provável | Solução |
|---------|----------------|---------|
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Outro processo ou contêiner usa a 8080 | `docker ps --format "{{.Names}} {{.Ports}}"` para achar e parar o contêiner (`docker stop <nome>`) ou o programa que ocupa a porta |
| `required variable POSTGRES_… is missing a value` | `.env` ausente ou incompleto | Seção 1 |
| `/health` retorna `503` com `password authentication failed` | A senha do `.env` foi trocada **depois** da criação do volume | Voltar à senha original ou, se os dados puderem ser descartados, `docker compose down -v` e subir de novo |
| `POST /itens` retorna `413 Request Entity Too Large` | Foto maior que 10 MB | Reduza a imagem ou ajuste `client_max_body_size` em `nginx/default.conf` |
| `PermissionError` ao gravar foto | Volume `fotos` criado por uma versão antiga da imagem, com dono root | `docker compose run --rm --user root web chown -R app:app /app/uploads` |
| `502 Bad Gateway` depois de recriar o `web` manualmente | O nginx ainda aponta para o IP antigo | `docker compose restart nginx` |
