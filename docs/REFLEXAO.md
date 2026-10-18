# Reflexão e equivalência em nuvem (A5)

As referências `[ev NN]` apontam para [`docs/evidencias/`](evidencias/README.md).

## Questões de reflexão

### Q1. Por que `web` encontra o banco pelo nome `db`?
Os dois serviços estão na `rede_interna`, uma rede *bridge* **definida pelo usuário**. Nesse tipo de rede, o Docker
mantém um servidor DNS embutido, acessível em `127.0.0.11` dentro de cada contêiner, que traduz o nome do serviço (`db`)
para o IP atual do contêiner. Quem resolve o nome, portanto, é o **Docker Engine**, e não um `/etc/hosts` fixo.
Isso importa porque os IPs mudam a cada recriação: o `db` foi `172.20.0.2` numa execução [ev 11] e os contêineres ganham
IDs novos após `down`/`up` [ev 13]. O mesmo mecanismo devolve os 3 IPs de `web` quando o serviço é escalado [ev 17].
Na **bridge padrão** (`docker0`, usada por `docker run` sem `--network`), não existe resolução por nome entre contêineres:
seria preciso descobrir e fixar o IP do banco (que muda) ou usar o `--link`, que está obsoleto.

### Q2. Volume nomeado × bind mount
Um **volume nomeado** é criado e gerenciado pelo Docker (em `/var/lib/docker/volumes/<nome>/_data`, dentro da VM no Docker Desktop [ev 14]).
Ele independe da estrutura de pastas do host, tem I/O nativo do Linux e, quando nasce vazio, herda o conteúdo e as permissões do diretório da imagem.
Um **bind mount** monta um caminho específico do host, e o conteúdo é controlado por quem edita esse arquivo no host.
O `init.sql` é **código**: versionado no Git, editado pela equipe, igual para todos os clones. Por isso é um bind mount do
repositório, montado como `:ro` para que o contêiner não o altere. Os dados do banco são **estado da aplicação**: não devem ser editados à mão,
precisam ter o dono que o Postgres exige (uid 70) e desempenho de disco nativo. No Docker Desktop, um bind de `C:\` atravessa a
fronteira Windows→VM, é mais lento e tem problemas de permissão e `fsync`. Por isso esses dados ficam num volume nomeado.

### Q3. Alterar o `init.sql` depois da primeira subida tem efeito?
**Não.** O entrypoint da imagem `postgres` só executa `/docker-entrypoint-initdb.d/*` quando `/var/lib/postgresql/data` está vazio.
Na segunda subida o log diz *"PostgreSQL Database directory appears to contain a database; Skipping initialization"* [ev 20].
Como o volume `dados_banco` persiste, o arquivo novo é ignorado. O `down -v` aplicaria a mudança, mas **apagando todos os dados** [ev 19].
O caminho correto é uma **migração**: executar o DDL no banco que já existe, por exemplo
`docker compose exec db sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "ALTER TABLE itens ADD COLUMN …"'`,
o que mantém as linhas existentes (demonstrado numa transação revertida em [ev 20]). Em seguida, atualiza-se o `init.sql` para ambientes novos.
Em produção, isso deve ser versionado em `db/migrations/` com uma ferramenta como Flyway, Liquibase ou Alembic, sempre após um `pg_dump`.

### Q4. Por que `depends_on` sem `condition: service_healthy` não basta?
Na forma curta, o `depends_on` só ordena a **criação e o início** dos contêineres: o `web` é iniciado assim que o processo do `db`
está *running*. Isso não significa que o PostgreSQL aceite conexões. Na primeira subida ele roda `initdb`, um servidor temporário só no socket Unix,
o `init.sql` e um reinício, o que leva alguns segundos [ev 20: `[41] ready` → `init.sql` → `[1] ready`]. Nesse intervalo, a primeira
conexão do `web` receberia *connection refused* (o `/health` responderia 503). Uma aplicação que conecta no boot entraria em
*crash loop*. Com `condition: service_healthy`, o Compose espera o `pg_isready` passar. No nosso healthcheck, o `-h 127.0.0.1`
garante que só o servidor definitivo, que escuta em TCP, seja considerado pronto [ev 01: `db Healthy` antes de `web Starting`].

### Q5. Ordem `COPY requirements.txt` → `RUN pip install` → `COPY app.py` e o cache de camadas
Cada instrução do Dockerfile gera uma camada. No build, o Docker reaproveita uma camada enquanto a instrução e as entradas dela
(para `COPY`, o checksum dos arquivos) forem iguais. A partir da **primeira** camada invalidada, todas as seguintes são refeitas.
Medido em [ev 21]: no build frio, o `pip install` levou **83,6 s**. Num rebuild sem mudanças, todas as camadas vieram `CACHED`.
Com **só o `app.py` alterado** (numa cópia temporária do contexto), as camadas até o `pip install` continuaram `CACHED` e apenas o
`COPY app.py` foi refeito, em **0,1 s** (uma camada de 12 KB [ev 12]). Se o código fosse copiado antes do `pip install` (um `COPY . .` no início),
qualquer alteração de uma linha reinstalaria todas as dependências: os mesmos ~80 s e uma camada nova de 21 MB para enviar ao registry a cada deploy.

### Q6. Porta 5432 fechada, LGPD e medidas adicionais
Publicar a 5432 exporia o banco a qualquer máquina que alcance o host (Wi-Fi do campus, VPN), abrindo espaço para força bruta
e exploração de falhas. No nosso ambiente, o `db` não tem `ports` [ev 10], fica só na `rede_interna`, o nginx nem tem rota até ele
e o adminer só ouve em `127.0.0.1`. Fotos de RG ou carteira de estudante são **dados pessoais** (LGPD, art. 5º, I), e o art. 46 exige
medidas técnicas de segurança. A superfície mínima, os segredos fora do Git (RNF03) e o processo sem root (RNF04) atendem a esse dever.
**Medidas adicionais:** (1) **controle de acesso e criptografia**: hoje `/fotos/<nome>` é público. Exigir autenticação ou URLs assinadas
com expiração, HTTPS e criptografia em repouso. (2) **Retenção mínima**: apagar fotos e registros após a devolução ou após N dias,
fotografar o objeto sem expor o documento e registrar quem acessou cada foto.

### Q7. `docker compose up -d --scale web=3`
**Na arquitetura base** (`web` publicando `8080:5000`), o comando falha: a 1ª réplica ocupa a porta 8080 do host e a 2ª recebe
*"port is already allocated"*. Uma porta do host só pode ter um dono [ev 18]. Para ter 3 réplicas, é preciso tirar a porta do `web`
(e não usar `container_name`) e pôr um **balanceador** na frente. Foi o que o bônus fez: o nginx publica a 8080, o DNS do Docker devolve
os 3 IPs de `web` e o tráfego fica 4/4/4 [ev 17]. Isso exigiu `zone` no `upstream` e um reinício do nginx após escalar. **Volume `fotos`
compartilhado:** no mesmo host, sim. As 3 réplicas veem o mesmo diretório [ev 17] e cada upload tem nome UUID único (sem colisão).
Mas o volume `local` existe num só nó: réplicas em várias máquinas exigiriam storage de rede (NFS/EFS/Azure Files) ou, melhor, object storage.

---

## 5.1 Do ambiente local para a nuvem

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
|------------------------|-----------------|-----|--------------|
| Contêiner `web` | Azure Container Apps (ou App Service for Containers) | Amazon ECS on Fargate (ou AWS App Runner) | Cloud Run |
| Contêiner `db` + volume `dados_banco` | Azure Database for PostgreSQL (Flexible Server) | Amazon RDS for PostgreSQL (ou Aurora PostgreSQL) | Cloud SQL for PostgreSQL |
| Volume `fotos` | Azure Blob Storage | Amazon S3 | Cloud Storage |
| `rede_interna` | Virtual Network (VNet) com sub-rede privada, NSG e Private Endpoint | VPC com sub-redes privadas e Security Groups | VPC com Private Service Access e Direct VPC egress / Serverless VPC Access |
| Arquivo `.env` (segredos) | Azure Key Vault | AWS Secrets Manager (ou SSM Parameter Store) | Secret Manager |
| Imagem construída localmente | Azure Container Registry (ACR) | Amazon ECR | Artifact Registry |
| *(bônus)* proxy `nginx` | Ingress do Container Apps / Application Gateway | Application Load Balancer | Cloud Load Balancing (já embutido no Cloud Run) |

**Qual peça muda mais na migração, e por quê.** O **volume `fotos`**. Todas as outras peças trocam um recurso local por um
serviço gerenciado **sem mudar o código**: o `app.py` já lê `DB_HOST`, `DB_USER`, `DB_PASSWORD` etc. de variáveis de ambiente.
Basta apontá-las para o endpoint do banco gerenciado, injetar os segredos a partir do Key Vault, Secrets Manager ou Secret Manager e publicar a mesma
imagem no registry. Já as fotos dependem do **sistema de arquivos local** (`foto.save(...)`, `os.listdir(UPLOAD_DIR)`, `send_from_directory`).
Nos serviços de contêiner gerenciados, o disco é efêmero e cada réplica tem o seu, e eles escalam horizontalmente (exatamente o problema da Q7).
A troca natural é o object storage (Blob, S3, GCS), que tem outra **API** (upload por SDK, URLs assinadas ou CDN para servir as imagens,
contagem por listagem do bucket). Isso exige reescrever as rotas `POST /itens`, `GET /fotos` e `/estatisticas`. É a única peça em que a migração deixa de ser só
infraestrutura e passa a envolver código, e também onde moram os dados pessoais mais sensíveis (Q6).
