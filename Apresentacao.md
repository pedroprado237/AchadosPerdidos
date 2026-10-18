# Apresentação: Achados & Perdidos em contêineres

**Missão:** subir a aplicação web e o banco com **um único comando** (`docker compose up -d --build`), sem perder
itens nem fotos quando os contêineres forem recriados, e documentar tudo para a equipe que fará a migração para a nuvem.

## 1. Arquitetura em 30 segundos

```
Navegador ──:8080──► nginx ──rede_borda──► web (Flask + Gunicorn :5000, usuário app) ──rede_interna──► db (postgres:16-alpine)
                                               │                                                      │
                                      volume fotos (/app/uploads)                 volume dados_banco (/var/lib/postgresql/data)
                                                                                  + bind ./db/init.sql (somente leitura)
```

- **Só o nginx** publica porta no host (8080). O `web` e o `db` não publicam nenhuma.
- **db** fica só na `rede_interna`, então o nginx (que é exposto) não consegue alcançá-lo.
- **Dois volumes nomeados** guardam o estado. O `init.sql` é bind mount, porque é código versionado.
- Diagrama completo: [docs/arquitetura.md](docs/arquitetura.md)

## 2. Requisitos: como cada um foi atendido

| Req. | Como | Prova |
|------|------|-------|
| RF01 `/health` | Rota do app, acessada via nginx: `status ok`, `banco conectado` | ev 03 |
| RF02 `/itens` + filtro | `?local=biblio` retorna só o item da Biblioteca | ev 04, 05 |
| RF03 `POST /itens` | Multipart com foto → 201 e nome UUID; sem campo obrigatório → 400 | ev 06 |
| RF04 `/fotos`, `/estatisticas` | Foto baixada com SHA-256 idêntico ao original; contadores corretos | ev 07, 08 |
| RNF01 um comando | Testado num **clone limpo**: `cp .env.example .env` + `up -d --build` | ev 22 |
| RNF02 persistência | `down` + `up`: 6 itens e 3 fotos antes e depois | ev 13 |
| RNF03 sem senhas | `${VAR:?}` lê do `.env` (ignorado pelo Git); sem `.env`, o Compose aborta com mensagem | ev 10, 22 |
| RNF04 sem root | Usuário `app` (uid 10001), dono de `/app/uploads`; `whoami` → `app` | ev 10 |
| RNF05 banco pronto | `depends_on: condition: service_healthy` + `pg_isready` | ev 01, 20 |

## 3. Tarefas entregues

| Tarefa (pts) | Entregável | Destaque para comentar |
|---|---|---|
| T1 Estrutura | Árvore idêntica à pedida (+ `nginx/`) | Código fornecido **não foi alterado** |
| T2 Dockerfile (1,5) | [app/Dockerfile](app/Dockerfile) + `.dockerignore` | slim, requirements antes do código, `--no-cache-dir`, `USER app`, CMD em forma exec |
| T3 Compose (2,5) | [docker-compose.yml](docker-compose.yml) | healthcheck, `service_healthy`, `rede_interna`, `restart: unless-stopped`, db sem porta |
| T4 Testes (1,5) | ev 01–12 | Todos os comandos do roteiro, com a saída real |
| T5 Persistência (1,5) | ev 13, 14, 19 | down/up preserva; dados ficam em `/var/lib/docker/volumes` (VM WSL2); `down -v` apaga |
| T6 Documentação (3,0) | README, arquitetura, RELATÓRIO (a–d), 23 evidências, REFLEXÃO | Q1–Q7 com 6–7 linhas cada + tabela Azure/AWS/GCP |
| Bônus (+1,0) | nginx (+0,4), backup/restore (+0,3), adminer em profile (+0,3) | ev 15, 16, 17 |

## 4. Decisões técnicas para defender

1. **`pg_isready -h 127.0.0.1`.** Na 1ª subida, o Postgres fica "pronto" no socket Unix *antes* de rodar o `init.sql` (o log mostra isso, ev 20). Testar via TCP evita liberar o `web` cedo demais.
2. **Ordem das camadas.** O build frio levou 83,6 s no `pip install`. Quando só o `app.py` muda, o rebuild leva **0,1 s** (ev 21).
3. **Duas redes.** `rede_interna` (web e db) é a exigida. A `rede_borda` (nginx e web) isola o banco do componente exposto.
4. **`zone` no upstream do nginx.** Sem ela, com `--scale web=3`, todas as requisições iam para uma réplica, porque cada worker fazia o próprio round-robin. Com ela, a divisão fica 4/4/4 (ev 17).
5. **adminer em `127.0.0.1:8081`.** O painel do banco nunca fica exposto para a rede local.
6. **`GET /` listando as rotas.** Quem responde é o nginx, já que não se podia alterar o `app.py` (ev 23).

## 5. Roteiro de demonstração (~5 min, PowerShell, na pasta do projeto)

```powershell
docker compose up -d --build                     # 1. sobe tudo; mostrar a ordem db → web → nginx
docker compose ps                                # 2. db e web "healthy"; só o nginx tem porta
curl.exe http://localhost:8080/                  # 3. índice das rotas
curl.exe http://localhost:8080/health
curl.exe -F descricao="Oculos de grau" -F local="Bloco B" -F foto=@docs/amostras/oculos.jpg http://localhost:8080/itens
curl.exe http://localhost:8080/itens             # 4. abrir a URL da foto no navegador
curl.exe http://localhost:8080/estatisticas
docker compose exec web whoami                   # 5. "app" (não root)
docker compose port db 5432                      #    vazio: banco sem porta no host
docker compose down; docker compose up -d        # 6. persistência
curl.exe http://localhost:8080/estatisticas      #    mesmos números
```

> Antes da apresentação, verifique se a porta 8080 está livre (`docker ps`) e se o `.env` existe.

## 6. Perguntas prováveis e respostas curtas

| Pergunta | Resposta |
|---|---|
| Como o `web` acha o `db`? (Q1) | DNS embutido do Docker (127.0.0.11) nas redes definidas pelo usuário. Na bridge padrão não há resolução por nome. |
| Volume × bind mount? (Q2) | Volume é gerenciado pelo Docker (estado). Bind é um caminho do host (código). O `init.sql` é código, por isso é bind `:ro`. |
| Alterei o `init.sql`, vale? (Q3) | Não: ele só roda com o diretório de dados vazio (*Skipping initialization*). Aplica-se com `ALTER TABLE`, como migração. |
| Por que `service_healthy`? (Q4) | `depends_on` simples só espera o contêiner iniciar, não o Postgres aceitar conexões. |
| Ordem do Dockerfile? (Q5) | Cache de camadas: mudar o código só refaz a última camada (83,6 s → 0,1 s). |
| Porta 5432 e LGPD? (Q6) | Banco sem porta = menor superfície. Fotos de documentos são dado pessoal (art. 5º e 46). Propostas: acesso autenticado + criptografia e retenção mínima. |
| `--scale web=3`? (Q7) | Na configuração base falha (porta 8080 já alocada). Com o nginx funciona. O volume pode ser compartilhado no mesmo host; entre máquinas, só com object storage. |
| Por que slim e não alpine? | As wheels do `psycopg[binary]` são para glibc; a slim dispensa compilador. |
| O que muda mais na nuvem? | O volume `fotos`: vira Blob/S3/GCS e **exige mudar o código**. O resto é troca 1:1 via variáveis de ambiente. |

## 7. Números para citar

Imagem `web` com **206 MB** · `pip install` 83,6 s → rebuild **0,1 s** · **23 evidências** reais · 3 réplicas balanceadas **4/4/4** ·
backup e restauração com **SHA-256 idênticos** · **0** senhas no histórico do Git.

## 8. Limitações conhecidas (código fornecido, fora do escopo)

API sem autenticação · upload sem validação de tipo · foto órfã se o `INSERT` falhar · sem HTTPS. Estão detalhadas em
[docs/RELATORIO.md](docs/RELATORIO.md#limitações-conhecidas-e-recomendações), com recomendação para a próxima etapa.
