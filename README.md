# Challenge – Oracle APEX

Ambiente completo para subir um **Oracle Database Free (23c/26ai)** e duas APIs de aplicação:

- **.NET API** (Sistema Contábil) – porta externa **8082**  
- **Java API** (Spring Boot) – porta externa **8081**  
- **Oracle FreePDB1** – portas **1521** (SQL*Net) e **5500** (EM Express)

> Testado em Oracle Cloud (VM pública **140.238.179.84**) e Linux x86_64 com Docker/Compose.

---

## 📦 Arquitetura

- `oracledb`  
  - Imagem: `container-registry.oracle.com/database/free:latest`  
  - PDB: `FREEPDB1`  
  - Usuário de app: **APPUSER / AppPass#2025** (criado no startup)  
  - Scripts de inicialização em `./db_init` (criação de usuário, schema e dados mínimos)

- `dotnet-api`  
  - Exposta em **http://<HOST>:8082** → roteia para 8080 no container  
  - Health: `/health` | Swagger: `/swagger` | Root: `/`

- `java-api`  
  - Exposta em **http://<HOST>:8081** → roteia para 8080 no container  
  - Swagger: `/swagger-ui/` (ou `/swagger-ui.html`, dependendo da versão)  
  - Por padrão, perfil **local** (H2 em memória). Você pode apontar para Oracle via env vars.

---

## ✅ Pré-requisitos

- Docker 24+ e Docker Compose Plugin
- Portas abertas no firewall (se acesso externo): **8081, 8082, 1521, 5500**
- (Opcional) SQL Developer, DBeaver ou outra IDE SQL

---

## 🚀 Subida rápida (Quick Start)

```bash
# 1) Clone
git clone git@github.com:bmvck/challenge-sprint2-cloud.git
cd challenge-sprint2-cloud

# 2) (Opcional recomendado) Crie um .env com overrides
cp .env.example .env   # edite se quiser alterar senhas/portas

# 3) Suba o banco primeiro (executa scripts de startup)
docker compose up -d oracledb

# 4) Acompanhe os logs até ver "DONE: Executing user defined scripts"
docker compose logs -f oracledb
# Ctrl+C para sair do follow

# 5) Suba as APIs
docker compose up -d

# 6) Verifique
docker compose ps
```

### Endpoints de teste

- **.NET API**:  
  - Local: `http://localhost:8082/`  
  - Remoto (OCI): `http://140.238.179.84:8082/`  
  - Swagger: `http://<HOST>:8082/swagger`  
  - Health: `http://<HOST>:8082/health`

- **Java API**:  
  - Local: `http://localhost:8081/`  
  - Remoto (OCI): `http://140.238.179.84:8081/`  
  - Swagger: `http://<HOST>:8081/swagger-ui/`

- **Oracle**:  
  - SQL*Net: `<HOST>:1521` (SERVICE NAME: `FREEPDB1`)  
  - EM Express: `https://<HOST>:5500/em` (se habilitado)

---

## 🗄️ Banco de Dados

| Item                | Valor                 |
|---------------------|-----------------------|
| CDB                 | `FREE`                |
| PDB                 | `FREEPDB1`            |
| Usuário aplicação   | `APPUSER`             |
| Senha (dev)         | `AppPass#2025`        |
| Porta               | `1521`                |
| Service             | `FREEPDB1`            |

**SQL Developer (exemplo)**  
- Tipo: Oracle  
- Host: `140.238.179.84`  
- Porta: `1521`  
- **Service name**: `FREEPDB1`  
- Usuário: `APPUSER`  
- Senha: `AppPass#2025`

### Scripts de inicialização

- `db_init/20_create_appuser.sql` – cria/garante `APPUSER` com grants essenciais  
- `db_init/40_load_schema.sh` – carrega o schema a partir de `scripts/startup/challenge_oracle2_fixed.sql`  
- *Hardening de startup*: scripts foram ajustados para **não** tentar criar triggers no schema `SYS`, evitando ruído de log.

### Popular dados padrão (se necessário)

A procedure `PR_SETUP_DEFAULTS` cria alguns registros de base:

```sql
BEGIN
  pr_setup_defaults;
END;
/
```

---

## 🔧 Variáveis de ambiente (exemplo)

Crie um `.env` (ou use `docker-compose.override.yml`) para manter secretos fora do git.

```dotenv
# .NET
DOTNET_HTTP_PORT=8080

# Java
JAVA_HTTP_PORT=8080
SPRING_PROFILES_ACTIVE=local         # ou "oracle"
SPRING_DATASOURCE_URL=jdbc:oracle:thin:@oracledb:1521/FREEPDB1
SPRING_DATASOURCE_USERNAME=APPUSER
SPRING_DATASOURCE_PASSWORD=AppPass#2025

# Oracle
ORACLE_PWD=System#2025
ORACLE_PDB=FREEPDB1
```

> **Dica:** para usar Oracle na **Java API**, troque o profile `local` (H2) para um profile que use Oracle e forneça as variáveis de datasource (como no snippet acima).

---

## 🔁 Atualizar containers

### Java

```bash
# dentro do repo da API Java (separado), gere o JAR:
./mvnw -DskipTests clean package

# copie o jar para a pasta mapeada (seu compose deve montar /app/app.jar)
# OU ajuste a imagem Docker e:
docker compose build java-api
docker compose up -d java-api

# conferir logs
docker compose logs -f java-api
```

### .NET

```bash
# dentro do repo .NET (notNetContabil), publique:
dotnet publish -c Release -o out

# ajuste o Dockerfile se necessário e:
docker compose build dotnet-api
docker compose up -d dotnet-api

# conferir logs
docker compose logs -f dotnet-api
```

### Banco

```bash
# reiniciar banco mantendo dados
docker compose restart oracledb

# reset total (APAGA dados!)
docker compose down -v
docker compose up -d oracledb
docker compose logs -f oracledb
```

---

## 🧪 Testes rápidos (cURL)

```bash
# .NET root
curl -i http://localhost:8082/

# .NET health
curl -s http://localhost:8082/health

# Java actuator (se exposto)
curl -i http://localhost:8081/actuator/health
```

---

## 📝 Estrutura de pastas (sugerida)

```
challenge-sprint2-cloud/
├─ docker-compose.yml
├─ .env.example
├─ db_init/
│  ├─ 20_create_appuser.sql
│  ├─ 40_load_schema.sh
│  └─ 50_fix_triggers.sql              # (opcional) correções de schema/trigger
├─ scripts/
│  └─ startup/
│     └─ challenge_oracle2_fixed.sql   # DDL/DML principal do schema APPUSER
├─ dotnet-api/                          # (se build local)
│  └─ Dockerfile
└─ java-api/                            # (se build local)
   └─ Dockerfile
```

---

## 🛡️ Boas práticas & segurança

- **Não** mantenha senhas reais no repositório. Use `.env` (não commitado) ou secrets.  
- Restrinja portas públicas (1521/5500) quando não precisar de acesso externo.  
- Trate `AppPass#2025` como **senha de desenvolvimento** apenas. Em produção, gere outra.

---

## 🐞 Troubleshooting

**`Internal Server Error` (Java)**  
- Verifique se a API está no profile correto. Se usando Oracle, confira as env vars de datasource e driver.

**`ORA-01017 invalid credential`**  
- Usuário/senha errados ou conectando no CDB (`FREE`) em vez do PDB (`FREEPDB1`). Use `SERVICE_NAME=FREEPDB1`.

**Triggers com erro no log (`ORA-04089 ... owned by SYS`)**  
- Já mitigado: os scripts de startup criam objetos sob `APPUSER`. Se houver algum resto, rode o `50_fix_triggers.sql` apontando `CURRENT_SCHEMA=APPUSER`.

**`permission denied while trying to connect to the Docker daemon socket`**  
- Rode com `sudo` ou adicione seu usuário ao grupo `docker` e relogue:  
  `sudo usermod -aG docker $USER && newgrp docker`

**Porta 1521 não conecta externamente**  
- Abra no firewall da VM/VCN (OCI) e no `firewalld` local:  
  `sudo firewall-cmd --add-port=1521/tcp --permanent && sudo firewall-cmd --reload`

**Reset total do banco**  
```bash
docker compose down -v
docker compose up -d oracledb
docker compose logs -f oracledb
```

