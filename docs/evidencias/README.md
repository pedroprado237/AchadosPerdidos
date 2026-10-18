# Evidências (A4)

Saídas reais dos comandos, geradas em 18/10/2026 contra o código deste repositório
(Docker Engine 28.5.1, Compose v2.40.3, Docker Desktop/WSL2 no Windows 11).
Cada arquivo começa com uma legenda de uma linha, a data/hora e, depois, cada comando
(`$ ...`) seguido da sua saída. As imagens usadas nos uploads estão em [`../amostras/`](../amostras/).

| # | Arquivo | Legenda |
|---|---------|---------|
| 01 | [01-compose-up-build.txt](01-compose-up-build.txt) | T4: subida do zero com `docker compose up -d --build` (build, redes, volumes e ordem db → web → nginx por healthcheck). |
| 02 | [02-compose-ps.txt](02-compose-ps.txt) | T4: `db` e `web` *healthy*; só o nginx publica porta (`0.0.0.0:8080->80`). O `5432/tcp` do db é só a porta exposta na rede interna, sem mapeamento para o host. |
| 03 | [03-health.txt](03-health.txt) | RF01: `GET /health` → `status: ok`, `banco: conectado`. |
| 04 | [04-itens.txt](04-itens.txt) | RF02: `GET /itens` lista os 3 objetos de exemplo do `init.sql`. |
| 05 | [05-itens-filtro.txt](05-itens-filtro.txt) | RF02: `GET /itens?local=biblio` filtra (ILIKE) e devolve só o item da Biblioteca. |
| 06 | [06-post-item-foto.txt](06-post-item-foto.txt) | RF03: `POST /itens` com foto → 201 + nome UUID; sem `descricao` → 400. |
| 07 | [07-estatisticas.txt](07-estatisticas.txt) | RF04: `GET /estatisticas` → 4 itens no banco e 1 foto no volume. |
| 08 | [08-get-foto.txt](08-get-foto.txt) | RF04: `GET /fotos/<nome>` devolve `image/jpeg` com SHA-256 igual ao arquivo enviado. |
| 09 | [09-logs-web.txt](09-logs-web.txt) | T4: `docker compose logs web --tail 20`, com boot do Gunicorn e access log (IP do nginx e healthcheck local). |
| 10 | [10-seguranca.txt](10-seguranca.txt) | RNF03/RNF04: `whoami` = `app` (uid 10001); db e web sem porta no host; senha real ausente dos arquivos rastreáveis; `.env` ignorado pelo Git. |
| 11 | [11-network-inspect.txt](11-network-inspect.txt) | A4: `docker network inspect` da `rede_interna` (só web e db) e da `rede_borda` (nginx e web). |
| 12 | [12-image-ls.txt](12-image-ls.txt) | A4: `docker image ls` (imagem web com 206 MB) e `docker image history` com a ordem das camadas. |
| 13 | [13-persistencia-1-down-up.txt](13-persistencia-1-down-up.txt) | T5.1: `down` + `up -d` recria os contêineres (IDs novos) e mantém 6 itens e 3 fotos. |
| 14 | [14-persistencia-2-inspecao-volumes.txt](14-persistencia-2-inspecao-volumes.txt) | T5.2: `docker volume ls/inspect`, `ls -l /app/uploads`, diretório do PostgreSQL e a mesma pasta vista do Windows. |
| 15 | [15-bonus-backup-restore.txt](15-bonus-backup-restore.txt) | Bônus: backup do volume `fotos` em `.tar.gz` com `docker run --rm` e restauração em volume novo (checksums e dono idênticos). |
| 16 | [16-bonus-adminer-profile.txt](16-bonus-adminer-profile.txt) | Bônus: adminer só sobe com `--profile debug`, ouve em `127.0.0.1:8081` e faz login no `db`. |
| 17 | [17-q7-scale-com-nginx.txt](17-q7-scale-com-nginx.txt) | Q7: com o proxy, `--scale web=3` funciona; DNS devolve 3 IPs e o nginx balanceia 4/4/4. |
| 18 | [18-q7-scale-config-base.txt](18-q7-scale-config-base.txt) | Q7: na configuração base (`web` em `8080:5000`), `--scale web=3` falha com `port is already allocated`. |
| 19 | [19-persistencia-3-down-v.txt](19-persistencia-3-down-v.txt) | T5.3: `down -v` apaga os volumes; o `init.sql` roda de novo e restam só os 3 itens de exemplo e 0 fotos. |
| 20 | [20-q3-q4-initdb-healthcheck.txt](20-q3-q4-initdb-healthcheck.txt) | Q3/Q4: servidor temporário pronto antes do `init.sql`; *Skipping initialization* na recriação; `ALTER TABLE` revertido; healthchecks. |
| 21 | [21-q5-cache-camadas.txt](21-q5-cache-camadas.txt) | Q5: build frio (`pip install` = 83,6 s), rebuild todo `CACHED`, e só `COPY app.py` refeito (0,1 s) quando apenas o código muda. |
| 22 | [22-teste-clone-limpo.txt](22-teste-clone-limpo.txt) | Checklist: clone novo → sem `.env` falha com mensagem clara → `cp .env.example .env` → `up -d --build` → health ok, `whoami` = app, db sem porta, cadastro persiste após `down` + `up`. |

**Observações sobre a coleta**

- Os testes rodaram no Git Bash com `curl` (no PowerShell, o equivalente é `curl.exe`, como mostra o README).
- A evidência 21 refez a imagem do zero (`--no-cache`). Por isso o ID em 12 (`5cd2003a861b`) é o da imagem final, com o mesmo conteúdo usado nos testes. A alteração do `app.py` em 21 foi feita numa cópia temporária, fora do repositório.
- A evidência 18 usou um override temporário (`web` publicando `8080:5000`) num projeto descartável (`-p q7-base`), destruído logo depois. O arquivo não faz parte do repositório.
- Em 14, a lista de volumes foi filtrada por `name=achados-perdidos` para omitir volumes de outros projetos da máquina.
- O progresso de download de imagens foi omitido em 16, por ser apenas ruído.
