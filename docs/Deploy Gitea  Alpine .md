# ***Documentação Completa: Instalação e Configuração do Gitea no Alpine Linux 3.23.x**

***Instituição: IFRN  
Servidor: `alpinista-xy.ifrn.local` (192.168.0.9)  
Sistema Operacional: Alpine Linux 3.23.6  
Data: Outubro/2025**


## 📑 ***Sumário**

1. ***Visão Geral**

2. ***Pré-requisitos**

3. ***Arquitetura da Solução**

4. ***Instalação Automatizada**

5. ***Verificação Pós-Instalação**

6. ***Criação do Usuário Administrador**

7. ***Configurações Recomendadas Pós-Instalação**

8. ***Integração com Active Directory (AD)**

9. ***Proxy Reverso com Nginx (Portas 80/443)**

10. ***Manutenção e Backup**

11. ***Solução de Problemas**

12. ***Referências**


## ***1. Visão Geral**

### ***1.1 Objetivo**

***Este documento descreve o processo completo de instalação, configuração e integração do Gitea — um serviço de hospedagem de repositórios Git leve e auto-hospedado — no servidor Alpine Linux do IFRN.**

### ***1.2 O que é o Gitea?**

***O Gitea é uma aplicação web de código aberto escrita em Go, usada para hospedar repositórios Git. É uma alternativa leve ao GitHub, GitLab e Bitbucket, ideal para ambientes com recursos limitados. Suas principais características são:**

- ***Leve: binário único, baixo consumo de RAM (~100 MB).**

- ***Multi-banco de dados: suporta PostgreSQL, MySQL, MariaDB, SQLite e MSSQL.**

- ***Multi-plataforma: roda em Linux, Windows, macOS e ARM.**

- ***Integração LDAP/AD: autenticação delegada a diretórios corporativos.**

- ***API REST completa e suporte a Webhooks.**

### ***1.3 Escopo**

***Este documento cobre:**

- ***Instalação automatizada via script Bash.**

- ***Configuração do PostgreSQL como banco de dados.**

- ***Inicialização do serviço via OpenRC.**

- ***Criação do usuário administrador.**

- ***Integração com Active Directory (AD) via LDAP.**

- ***Proxy reverso com Nginx (opcional).**


## ***2. Pré-requisitos**

### ***2.1 Ambiente**

| ***Item** | ***Especificação** |
| - | - |
| ***Sistema Operacional** | ***Alpine Linux 3.23.x** |
| ***Arquitetura** | ***x86\_64** |
| ***Acesso** | ***root (via `su` ou SSH)** |
| ***RAM mínima** | ***512 MB (recomendado: 1 GB)** |
| ***Disco mínimo** | ***2 GB (recomendado: 10 GB+)** |
| ***Rede** | ***IP fixo: `192.168.0.9`** |
| ***Hostname** | ***`alpinista-xy.ifrn.local`** |

### ***2.2 Pacotes Necessários (instalados automaticamente)**

- ***`git`, `git-lfs`**

- ***`bash`, `curl`, `openssh`**

- ***`coreutils`, `grep`, `less`, `lsof`, `dialog`**

- ***`gitea`, `gitea-openrc`**

- ***`postgresql17`, `postgresql17-contrib`, `postgresql17-openrc`**

### ***2.3 Portas Utilizadas**

| ***Porta** | ***Protocolo** | ***Descrição** |
| - | - | - |
| ***3000** | ***TCP** | ***Interface HTTP do Gitea** |
| ***22** | ***TCP** | ***Acesso SSH aos repositórios Git** |
| ***5432** | ***TCP** | ***PostgreSQL (apenas localhost)** |


## ***3. Arquitetura da Solução**

***text**

```
┌─────────────────────────────────────────────────────────┐

│  ***Servidor Alpine Linux (alpinista-xy.ifrn.local)        │**

│                                                          │

│  ┌────────────────┐        ┌──────────────────────┐     │

│  │  ***Gitea (Go)    │◄──────►│  PostgreSQL 17       │     │**

│  │  ***Porta 3000    │        │  Porta 5432 (local)  │     │**

│  └────────┬───────┘        └──────────────────────┘     │

│           │                                              │

│           │ ***OpenRC (supervise-daemon)                   │**

│           │                                              │

│  ┌────────▼───────┐        ┌──────────────────────┐     │

│  │  ***/etc/gitea/   │        │  /var/lib/gitea/     │     │**

│  │  ***app.ini       │        │  repositories/       │     │**

│  └────────────────┘        │  ***custom/             │     │**

│                             │  ***data/               │     │**

│                             └──────────────────────┘     │

└─────────────────────────────────────────────────────────┘

           ▲

           │ ***HTTP (porta 3000)**

           │

    ┌──────┴───────┐

    │  ***Cliente Web │**

    │  ***(Navegador) │**

    └──────────────┘
```

***Fluxo de autenticação:**

1. ***Usuário acessa `http://192.168.0.9:3000`.**

2. ***Gitea serve a interface web (embutida em Go).**

3. ***No login, o Gitea consulta o AD via LDAP (Bind DN → busca usuário → valida senha).**

4. ***Se autenticado, cria sessão local e libera acesso.**


## ***4. Instalação Automatizada**

### ***4.1 Script Completo: `gitea-alpine-install.sh`**

***Salve o script abaixo como `/root/gitea-alpine-install.sh`:**

***bash**

```
***#!/usr/bin/env bash**

***#***

***# gitea-alpine-install.sh***

***# Instalação completa do Gitea no Alpine Linux 3.23.x***

***# Baseado em: https://wiki.alpinelinux.org/wiki/Gitea***

***#***

***set -euo pipefail**


***# ==================================================================***

***# 1. VARIÁVEIS CONFIGURÁVEIS***

***# ==================================================================***

***DB_TYPE="${DB_TYPE:-postgres}"**

***DB_NAME="${DB_NAME:-gitea}"**

***DB_USER="${DB_USER:-gitea}"**

***GITEA_HTTP_PORT="${GITEA_HTTP_PORT:-3000}"**

***GITEA_SSH_PORT="${GITEA_SSH_PORT:-22}"**

***GITEA_APP_INI="/etc/gitea/app.ini"**

***GITEA_WORK_DIR="/var/lib/gitea"**

***GITEA_LOG_DIR="/var/log/gitea"**

***GITEA_USER="gitea"**

***GITEA_GROUP="www-data"**


***# ==================================================================***

***# 2. FUNÇÕES AUXILIARES***

***# ==================================================================***

***log()   { printf '\033[1;34m[INFO]\033[0m %s\n' "$*"; }**

***error() { printf '\033[1;31m[ERRO]\033[0m %s\n' "$*" >&2; exit 1; }**


***gen_random() {**

    ***head -c 512 /dev/urandom | LC_ALL=C tr -dc 'A-Za-z0-9' | cut -c1-"${1:-24}"**

***}**


***cleanup() {**

    ***local exit_code=$?**

    ***if [[ $exit_code -ne 0 ]]; then**

        ***printf '\033[1;31m[ERRO]\033[0m Instalação interrompida (código: %d).\n' "$exit_code" >&2**

    ***fi**

***}**

***trap cleanup EXIT**


***# ==================================================================***

***# 3. VERIFICAR AMBIENTE ALPINE***

***# ==================================================================***

***[[ -f /etc/alpine-release ]] || error "Este script é destinado ao Alpine Linux."**


***ALPINE_VERSION=$(cat /etc/alpine-release)**

***log "Alpine Linux versão: ${ALPINE_VERSION}"**


***ALPINE_MAJOR_MINOR="$(echo "${ALPINE_VERSION}" | cut -d. -f1,2)"**

***if [ "$(printf '%s\n' "3.23" "${ALPINE_MAJOR_MINOR}" | sort -V | head -n1)" != "3.23" ]; then**

    ***log "Aviso: O script foi testado no Alpine 3.23.x."**

***fi**


***# ==================================================================***

***# 4. GARANTIR RESOLUÇÃO DO HOSTNAME***

***# ==================================================================***

***HOSTNAME_SHORT="$(hostname)"**

***if ! grep -qE "^\s*127\.0\.1\.1\s+${HOSTNAME_SHORT}\b" /etc/hosts 2>/dev/null; then**

    ***log "Adicionando ${HOSTNAME_SHORT} a /etc/hosts..."**

    ***echo "127.0.1.1    ${HOSTNAME_SHORT}" >> /etc/hosts**

***fi**


***GITEA_DOMAIN="${GITEA_DOMAIN:-$(hostname -f 2>/dev/null || hostname)}"**

***GITEA_ROOT_URL="${GITEA_ROOT_URL:-http://${GITEA_DOMAIN}:${GITEA_HTTP_PORT}/}"**


***log "Domínio Gitea: ${GITEA_DOMAIN}"**

***log "URL raiz:      ${GITEA_ROOT_URL}"**


***# ==================================================================***

***# 5. INSTALAR DEPENDÊNCIAS BÁSICAS***

***# ==================================================================***

***log "Atualizando repositórios..."**

***apk update**


***log "Instalando dependências básicas..."**

***apk add --no-cache \**

    ***git git-lfs bash curl openssh \**

    ***coreutils grep less lsof dialog**


***# ==================================================================***

***# 6. INSTALAR O GITEA E O SCRIPT OPENRC***

***# ==================================================================***

***log "Instalando Gitea e script OpenRC..."**

***apk add --no-cache gitea gitea-openrc**


***# Garantir usuário/grupo***

***if ! id -u "${GITEA_USER}" &>/dev/null; then**

    ***log "Criando usuário ${GITEA_USER}..."**

    ***adduser -S -D -H -h "${GITEA_WORK_DIR}" -s /sbin/nologin -G "${GITEA_GROUP}" "${GITEA_USER}" 2>/dev/null || \**

        ***adduser -S -D -H -h "${GITEA_WORK_DIR}" -s /sbin/nologin "${GITEA_USER}"**

***fi**


***if ! id -nG "${GITEA_USER}" 2>/dev/null | grep -q "\b${GITEA_GROUP}\b"; then**

    ***log "Adicionando ${GITEA_USER} ao grupo ${GITEA_GROUP}..."**

    ***addgroup "${GITEA_USER}" "${GITEA_GROUP}" 2>/dev/null || true**

***fi**


***# ==================================================================***

***# 7. INSTALAR E CONFIGURAR O BANCO DE DADOS***

***# ==================================================================***

***if [[ "${DB_TYPE}" == "postgres" ]]; then**

    ***log "Instalando PostgreSQL 17..."**

    ***apk add --no-cache postgresql17 postgresql17-contrib postgresql17-openrc**


    ***if [[ ! -f /var/lib/postgresql/data/PG_VERSION ]]; then**

        ***log "Cluster não inicializado. Preparando para setup..."**


        ***if rc-service postgresql status 2>/dev/null | grep -q "started"; then**

            ***log "Parando PostgreSQL para permitir o setup..."**

            ***rc-service postgresql stop**

            ***sleep 2**

        ***fi**


        ***log "Inicializando cluster PostgreSQL..."**

        ***/etc/init.d/postgresql setup**

    ***else**

        ***log "Cluster PostgreSQL já inicializado. Pulando setup."**

    ***fi**


    ***rc-service postgresql start**

    ***rc-update add postgresql default**


    ***DB_PASS="$(gen_random 24)"**


    ***log "Criando usuário '${DB_USER}' e banco '${DB_NAME}'..."**

    ***su postgres -c "psql -c \"CREATE USER ${DB_USER} WITH ENCRYPTED PASSWORD '${DB_PASS}';\"" 2>/dev/null || true**

    ***su postgres -c "psql -c \"CREATE DATABASE ${DB_NAME} OWNER ${DB_USER};\"" 2>/dev/null || true**

    ***su postgres -c "psql -c \"GRANT ALL PRIVILEGES ON DATABASE ${DB_NAME} TO ${DB_USER};\"" 2>/dev/null || true**


    ***DB_HOST="127.0.0.1:5432"**


***elif [[ "${DB_TYPE}" == "mariadb" ]]; then**

    ***log "Instalando MariaDB..."**

    ***apk add --no-cache mariadb mariadb-client**


    ***if [[ ! -d /var/lib/mysql/mysql ]]; then**

        ***log "Inicializando banco MariaDB..."**

        ***mysql_install_db --user=mysql --datadir=/var/lib/mysql**

    ***fi**


    ***rc-service mariadb start**

    ***rc-update add mariadb default**


    ***DB_PASS="$(gen_random 24)"**


    ***log "Criando banco '${DB_NAME}' e usuário '${DB_USER}'..."**

    ***mysql -u root <<-EOF**

        ***CREATE DATABASE IF NOT EXISTS \`${DB_NAME}\` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;**

        ***CREATE USER IF NOT EXISTS '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASS}';**

        ***GRANT ALL PRIVILEGES ON \`${DB_NAME}\`.* TO '${DB_USER}'@'localhost';**

        ***FLUSH PRIVILEGES;**

***EOF**


    ***DB_HOST="127.0.0.1:3306"**

***else**

    ***error "Tipo de banco inválido: '${DB_TYPE}'. Use 'postgres' ou 'mariadb'."**

***fi**


***# ==================================================================***

***# 8. CRIAR DIRETÓRIOS DO GITEA***

***# ==================================================================***

***log "Criando diretórios de trabalho, log e customização..."**

***mkdir -p "${GITEA_WORK_DIR}"/{custom,data,repositories}**

***mkdir -p "${GITEA_LOG_DIR}"**


***chown -R "${GITEA_USER}:${GITEA_GROUP}" "${GITEA_WORK_DIR}"**

***chown -R "${GITEA_USER}:${GITEA_GROUP}" "${GITEA_LOG_DIR}"**

***chmod -R 750 "${GITEA_WORK_DIR}"**

***chmod -R 750 "${GITEA_LOG_DIR}"**


***# ==================================================================***

***# 9. CONFIGURAR O ARQUIVO app.ini***

***# ==================================================================***

***log "Gerando ${GITEA_APP_INI}..."**


***if [[ -f "${GITEA_APP_INI}" ]]; then**

    ***BACKUP="${GITEA_APP_INI}.bak.$(date +%s)"**

    ***cp "${GITEA_APP_INI}" "${BACKUP}"**

    ***log "Backup criado: ${BACKUP}"**

***fi**


***mkdir -p "$(dirname "${GITEA_APP_INI}")"**


***SECRET_KEY="$(gen_random 32)"**

***INTERNAL_TOKEN="$(gen_random 32)"**

***JWT_SECRET="$(gen_random 32)"**


***if [[ "${DB_TYPE}" == "mariadb" ]]; then**

    ***GITEA_DB_TYPE="mysql"**

***else**

    ***GITEA_DB_TYPE="${DB_TYPE}"**

***fi**


***cat > "${GITEA_APP_INI}" <<EOF**

***APP_NAME = Gitea: Git with a cup of tea**

***RUN_USER = ${GITEA_USER}**

***RUN_MODE = prod**


***[server]**

***PROTOCOL         = http**

***DOMAIN           = ${GITEA_DOMAIN}**

***HTTP_ADDR        = 0.0.0.0**

***HTTP_PORT        = ${GITEA_HTTP_PORT}**

***ROOT_URL         = ${GITEA_ROOT_URL}**

***DISABLE_SSH      = false**

***SSH_PORT         = ${GITEA_SSH_PORT}**

***SSH_LISTEN_PORT  = ${GITEA_SSH_PORT}**

***OFFLINE_MODE     = false**

***LFS_START_SERVER = true**


***[database]**

***DB_TYPE  = ${GITEA_DB_TYPE}**

***HOST     = ${DB_HOST}**

***NAME     = ${DB_NAME}**

***USER     = ${DB_USER}**

***PASSWD   = ${DB_PASS}**

***SCHEMA   = public**

***SSL_MODE = disable**


***[repository]**

***ROOT = ${GITEA_WORK_DIR}/repositories**


***[security]**

***INSTALL_LOCK   = true**

***SECRET_KEY     = ${SECRET_KEY}**

***INTERNAL_TOKEN = ${INTERNAL_TOKEN}**


***[oauth2]**

***JWT_SECRET = ${JWT_SECRET}**


***[service]**

***DISABLE_REGISTRATION   = false**

***REQUIRE_SIGNIN_VIEW    = false**

***REGISTER_EMAIL_CONFIRM = false**


***[log]**

***MODE      = console**

***LEVEL     = info**

***ROOT_PATH = ${GITEA_LOG_DIR}**


***[mailer]**

***ENABLED = false**


***[session]**

***PROVIDER = file**


***[picture]**

***AVATAR_UPLOAD_PATH            = ${GITEA_WORK_DIR}/data/avatars**

***REPOSITORY_AVATAR_UPLOAD_PATH = ${GITEA_WORK_DIR}/data/repo-avatars**


***[attachment]**

***PATH = ${GITEA_WORK_DIR}/data/attachments**

***EOF**


***chown "${GITEA_USER}:${GITEA_GROUP}" "${GITEA_APP_INI}"**

***chmod 640 "${GITEA_APP_INI}"**


***log "Arquivo ${GITEA_APP_INI} criado com sucesso."**


***# ==================================================================***

***# 10. EXECUTAR MIGRAÇÕES DO BANCO DE DADOS***

***# ==================================================================***

***log "Executando migrações do banco de dados..."**

***su - "${GITEA_USER}" -c "GITEA_WORK_DIR=${GITEA_WORK_DIR} gitea migrate --config ${GITEA_APP_INI}"**


***# ==================================================================***

***# 11. HABILITAR E INICIAR O SERVIÇO GITEA***

***# ==================================================================***

***log "Habilitando e iniciando o serviço Gitea..."**

***rc-update add gitea default**

***rc-service gitea start**


***sleep 3**

***if rc-service gitea status | grep -q "started"; then**

    ***log "Serviço Gitea está em execução."**

***else**

    ***log "Aviso: Verifique com: rc-service gitea status"**

***fi**


***# ==================================================================***

***# 12. CONFIGURAR FIREWALL (se disponível)***

***# ==================================================================***

***if command -v nft &>/dev/null && nft list ruleset &>/dev/null; then**

    ***log "Adicionando regras de firewall (nftables)..."**

    ***nft add rule inet filter input tcp dport "${GITEA_HTTP_PORT}" accept 2>/dev/null || true**

    ***nft add rule inet filter input tcp dport "${GITEA_SSH_PORT}" accept 2>/dev/null || true**

    ***nft list ruleset > /etc/nftables.nft 2>/dev/null || true**

    ***rc-service nftables restart 2>/dev/null || true**

***elif command -v iptables &>/dev/null; then**

    ***log "Adicionando regras de firewall (iptables)..."**

    ***iptables -A INPUT -p tcp --dport "${GITEA_HTTP_PORT}" -j ACCEPT 2>/dev/null || true**

    ***iptables -A INPUT -p tcp --dport "${GITEA_SSH_PORT}" -j ACCEPT 2>/dev/null || true**

    ***rc-service iptables save 2>/dev/null || true**

***fi**


***# ==================================================================***

***# 13. INFORMAÇÕES FINAIS***

***# ==================================================================***

***echo ""**

***echo "======================================================"**

***echo "  Gitea instalado e configurado com sucesso!"**

***echo "======================================================"**

***echo "  URL de acesso:      ${GITEA_ROOT_URL}"**

***echo "  Porta HTTP:         ${GITEA_HTTP_PORT}"**

***echo "  Porta SSH:          ${GITEA_SSH_PORT}"**

***echo ""**

***echo "  Banco de dados:     ${DB_TYPE}"**

***echo "  Host do banco:      ${DB_HOST}"**

***echo "  Nome do banco:      ${DB_NAME}"**

***echo "  Usuário do banco:   ${DB_USER}"**

***echo "  Senha do banco:     ${DB_PASS}"**

***echo ""**

***echo "  Arquivo de config:  ${GITEA_APP_INI}"**

***echo "  Diretório de dados: ${GITEA_WORK_DIR}"**

***echo "  Diretório de logs:  ${GITEA_LOG_DIR}"**

***echo "  Diretório custom:   ${GITEA_WORK_DIR}/custom"**

***echo "======================================================"**

***echo ""**
```

### ***4.2 Como Executar**

***bash**

```
***# 1. Como root***

***su -**


***# 2. Salve o script***

***nano /root/gitea-alpine-install.sh**

***# (cole o conteúdo acima, salve com Ctrl+O, Enter, Ctrl+X)***


***# 3. Permissão de execução***

***chmod +x /root/gitea-alpine-install.sh**


***# 4. Execute***

***cd /root**

***./gitea-alpine-install.sh**
```

### ***4.3 Variáveis Configuráveis**

| ***Variável** | ***Padrão** | ***Descrição** |
| - | - | - |
| ***`DB_TYPE`** | ***`postgres`** | ***`postgres` ou `mariadb`** |
| ***`DB_NAME`** | ***`gitea`** | ***Nome do banco** |
| ***`DB_USER`** | ***`gitea`** | ***Usuário do banco** |
| ***`GITEA_HTTP_PORT`** | ***`3000`** | ***Porta HTTP** |
| ***`GITEA_SSH_PORT`** | ***`22`** | ***Porta SSH** |
| ***`GITEA_DOMAIN`** | ***FQDN** | ***Domínio do servidor** |

***Exemplo com MariaDB:**

***bash**

```
***DB_TYPE=mariadb GITEA_HTTP_PORT=8080 ./gitea-alpine-install.sh**
```


## ***5. Verificação Pós-Instalação**

### ***5.1 Checar Serviços**

***bash**

```
***# PostgreSQL***

***rc-service postgresql status**

***# Saída: * status: started***


***# Gitea***

***rc-service gitea status**

***# Saída: * status: started***


***# Habilitados no boot***

***rc-update show | grep -E 'gitea|postgresql'**
```

### ***5.2 Verificar Portas**

***bash**

```
***# Porta 3000 escutando?***

***ss -tlnp | grep :3000**


***# Porta 5432 escutando (local)?***

***ss -tlnp | grep :5432**
```

### ***5.3 Testar HTTP**

***bash**

```
***curl -I http://localhost:3000**

***# Esperado: HTTP/1.1 200 OK***
```

### ***5.4 Verificar Estrutura de Diretórios**

***bash**

```
***ls -la /etc/gitea/**

***ls -la /var/lib/gitea/**

***ls -la /var/log/gitea/**


***# Permissões corretas***

***stat -c "%U:%G %a %n" /etc/gitea/app.ini**

***# Esperado: gitea:www-data 640 /etc/gitea/app.ini***
```

### ***5.5 Consultar o Banco de Dados**

***bash**

```
***su postgres -c "psql -d gitea -c '\dt'"**

***# Deve listar as tabelas criadas pelo Gitea (user, repository, etc.)***
```


## ***6. Criação do Usuário Administrador**

***O primeiro usuário registrado no Gitea se torna automaticamente administrador. Há duas formas de criar:**

### ***6.1 Método A: Pela Interface Web (Mais Simples)**

1. ***Acesse `http://192.168.0.9:3000` no navegador.**

2. ***Clique em "Need an account? Register now."**

3. ***Preencha:**

   - ***Nome de usuário: `admin`**

   - ***E-mail: `admin@ifrn.local`**

   - ***Senha: (forte, mínimo 8 caracteres)**

4. ***Clique em Registrar.**

5. ***Pronto! Você será logado como administrador.**

### ***6.2 Método B: Pelo Terminal (Recomendado para Produção)**

***Execute como `root`:**

***bash**

```
***su - gitea -c "gitea admin user create \**

    ***--admin \**

    ***--username admin \**

    ***--password 'SuaSenhaForteAqui123!' \**

    ***--email 'admin@ifrn.local' \**

    ***--config /etc/gitea/app.ini"**
```

***Saída esperada:**

***text**

```
***New user 'admin' has been successfully created!**
```

***Parâmetros:**

| ***Parâmetro** | ***Descrição** |
| - | - |
| ***`--admin`** | ***Concede privilégios de administrador** |
| ***`--username`** | ***Nome de login (ex: `admin`)** |
| ***`--password`** | ***Senha entre aspas simples** |
| ***`--email`** | ***E-mail do admin** |
| ***`--config`** | ***Caminho do `app.ini`** |

### ***6.3 Listar Usuários Existentes**

***bash**

```
***su - gitea -c "gitea admin user list --config /etc/gitea/app.ini"**
```

### ***6.4 Alterar Senha de um Usuário**

***bash**

```
***su - gitea -c "gitea admin user change-password \**

    ***--username admin \**

    ***--password 'NovaSenhaForte!' \**

    ***--config /etc/gitea/app.ini"**
```

### ***6.5 Primeiro Acesso**

1. ***Acesse `http://192.168.0.9:3000/user/login`.**

2. ***Faça login com o usuário criado.**

3. ***No canto superior direito, clique no ícone de martelo (Administração do Site).**

4. ***Explore:**

   - ***Usuários: gerenciar contas.**

   - ***Organizações: criar grupos.**

   - ***Repositórios: gerenciar projetos.**

   - ***Configurações: ajustes globais.**


## ***7. Configurações Recomendadas Pós-Instalação**

***Após criar o admin, faça login e ajuste:**

### ***7.1 Desabilitar Registro Público**

***Administração do Site → Configurações → Desmarque "Permitir Registro de Usuários"**

***Ou via `app.ini`:**

***ini**

```
***[service]**

***DISABLE_REGISTRATION = true**
```

***Isso é essencial quando se integra com AD, pois apenas usuários do domínio devem poder logar.**

### ***7.2 Configurar E-mail (Opcional)**

***Para notificações e recuperação de senha:**

***ini**

```
***[mailer]**

***ENABLED = true**

***HOST = smtp.ifrn.local:587**

***FROM = gitea@ifrn.local**

***USER = gitea@ifrn.local**

***PASSWD = senha_do_email**
```

### ***7.3 Limitar Criação de Organizações**

***ini**

```
***[service]**

***DEFAULT_ALLOW_CREATE_ORGANIZATION = false**
```

### ***7.4 Configurar Captcha (Opcional)**

***ini**

```
***[service]**

***ENABLE_CAPTCHA = true**
```

### ***7.5 Reiniciar Após Alterações**

***bash**

```
***rc-service gitea restart**
```


## ***8. Integração com Active Directory (AD)**

### ***8.1 Visão Geral**

***O Gitea integra-se ao AD via LDAP. Utilizaremos a modalidade LDAP (via BindDN), que funciona em duas etapas:**

1. ***O Gitea se autentica no AD usando uma conta de serviço (Bind DN).**

2. ***Com essa conexão, ele busca o usuário que está tentando logar e valida a senha fornecida.**

### ***8.2 Pré-requisitos no Active Directory**

***Solicite ao administrador do AD:**

#### ***a) Conta de Serviço (Bind DN)**

***Criar uma conta com permissão de leitura nos objetos de usuário (não precisa ser admin de domínio):**

- ***Nome: `svc_gitea_ldap`**

- ***DN: `CN=svc_gitea_ldap,OU=ServiceAccounts,DC=ifrn,DC=local`**

- ***Senha: (anote, será usada no Gitea)**

#### ***b) Base de Busca de Usuários**

***A "raiz" da árvore onde os usuários estão:**

***text**

```
***OU=Usuarios,DC=ifrn,DC=local**
```

#### ***c) Grupo de Administradores (Opcional)**

***Crie o grupo `Gitea_Admins` no AD e adicione os usuários que serão admins no Gitea:**

***text**

```
***CN=Gitea_Admins,OU=Grupos,DC=ifrn,DC=local**
```

#### ***d) Informações do Controlador de Domínio**

- ***Host: `192.168.0.10` (ou o IP do DC)**

- ***Porta: `389` (LDAP) ou `636` (LDAPS)**

### ***8.3 Configuração pela Interface Web (Recomendado)**

1. ***Faça login como admin no Gitea.**

2. ***Vá em Administração do Site (ícone do martelo) → Identidade & Acesso → Fontes de Autenticação.**

3. ***Clique em Adicionar Fonte de Autenticação.**

4. ***Selecione LDAP (via BindDN).**

5. ***Preencha os campos conforme a tabela abaixo (exemplos para o domínio `ifrn.local`):**

| ***Campo** | ***Valor** |
| - | - |
| ***Nome da Autorização** | ***`AD IFRN`** |
| ***Host** | ***`192.168.0.10`** |
| ***Porta** | ***`389`** |
| ***Ativar Criptografia TLS** | ***Desmarcado (para porta 389) / Marcado (para 636)** |
| ***Filtro de Administrador** | ***`(memberOf=CN=Gitea_Admins,OU=Grupos,DC=ifrn,DC=local)`** |
| ***Atributo de Nome de Usuário** | ***`sAMAccountName`** |
| ***Atributo de Primeiro Nome** | ***`givenName`** |
| ***Atributo de Sobrenome** | ***`sn`** |
| ***Atributo de E-mail** | ***`mail`** |
| ***Bind DN** | ***`CN=svc_gitea_ldap,OU=ServiceAccounts,DC=ifrn,DC=local`** |
| ***Senha do Bind** | **(senha da conta svc\_gitea\_ldap)** |
| ***Base de Busca de Usuários** | ***`OU=Usuarios,DC=ifrn,DC=local`** |
| ***Filtro de Usuário** | ***`(&(objectCategory=Person)(sAMAccountName=%s)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))`** |
| ***Atributo de Nome Completo** | ***`displayName`** |

***Explicação do Filtro de Usuário:**

- ***`objectCategory=Person`: apenas pessoas (não grupos/computadores).**

- ***`sAMAccountName=%s`: o `%s` é substituído pelo login digitado.**

- ***`!(UserAccountControl:...=2)`: exclui contas desabilitadas no AD (bit 2 = ACCOUNTDISABLE).**

6. ***Clique em Adicionar Fonte de Autenticação.**

7. ***Teste imediatamente: faça logout e tente logar com um usuário do AD.**

### ***8.4 Configuração via `app.ini`**

***Se preferir automatizar via arquivo, adicione ao final de `/etc/gitea/app.ini`:**

***ini**

```
***[ldap]**

***ENABLED = true**

***NAME = AD IFRN**

***HOST = 192.168.0.10**

***PORT = 389**

***SECURITY_PROTOCOL = 0**

***BIND_DN = CN=svc_gitea_ldap,OU=ServiceAccounts,DC=ifrn,DC=local**

***BIND_PASSWORD = senha_da_conta_de_servico**

***USER_BASE = OU=Usuarios,DC=ifrn,DC=local**

***USER_FILTER = (&(objectCategory=Person)(sAMAccountName=%s)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))**

***ADMIN_FILTER = (memberOf=CN=Gitea_Admins,OU=Grupos,DC=ifrn,DC=local)**

***USERNAME_ATTRIBUTE = sAMAccountName**

***FIRSTNAME_ATTRIBUTE = givenName**

***SURNAME_ATTRIBUTE = sn**

***EMAIL_ATTRIBUTE = mail**
```

***Valores de `SECURITY_PROTOCOL`:**

| ***Valor** | ***Significado** |
| - | - |
| ***`0`** | ***Sem TLS** |
| ***`1`** | ***StartTLS (porta 389)** |
| ***`2`** | ***LDAPS/SSL (porta 636)** |

***Reinicie após alterar:**

***bash**

```
***rc-service gitea restart**
```

### ***8.5 Sincronização de Usuários do AD**

***O Gitea pode importar periodicamente todos os usuários do AD:**

***Na configuração da fonte de autenticação, marque "Ativar Sincronização de Usuário".**

***Ajuste a frequência no `app.ini`:**

***ini**

```
***[cron.sync_external_users]**

***RUN_AT_START = true**

***SCHEDULE = 0 0 3 * * *    # todo dia às 03:00**
```

### ***8.6 Mapeamento de Grupos do AD para Times do Gitea**

1. ***Crie uma Organização no Gitea (ex: `ifrn`).**

2. ***Vá em Organização → Equipes e crie uma equipe (ex: `Desenvolvedores`).**

3. ***Edite a equipe e vá em Sincronização de Grupo Externo.**

4. ***Informe o DN do grupo do AD:**

***text**

```
***CN=Desenvolvedores,OU=Grupos,DC=ifrn,DC=local**
```

5. ***Associe o nível de permissão (Read/Write/Admin).**

6. ***Ao sincronizar, os membros do grupo AD entram automaticamente na equipe.**

### ***8.7 Teste de Integração**

1. ***Pegue um usuário real do AD (ex: `joao.silva`).**

2. ***Faça logout do Gitea.**

3. ***Acesse `http://192.168.0.9:3000/user/login`.**

4. ***Digite `joao.silva` e a senha do AD.**

5. ***Se o login funcionar, o usuário será criado automaticamente no Gitea e vinculado ao AD.**

***Verificação via terminal:**

***bash**

```
***su - gitea -c "gitea admin user list --config /etc/gitea/app.ini"**

***# O usuário joao.silva deve aparecer na lista***
```

### ***8.8 Diagnóstico com Logs**

***Se o login falhar, aumente o nível de log:**

***ini**

```
***[log]**

***LEVEL = Trace**
```

***Reinicie e tente novamente:**

***bash**

```
***rc-service gitea restart**

***tail -f /var/log/gitea/gitea.log**
```

***Erros comuns:**

| ***Erro no log** | ***Causa** | ***Solução** |
| - | - | - |
| ***`LDAP Result Code 49 "Invalid Credentials"`** | ***Bind DN ou senha erradas** | ***Revisar credenciais da conta de serviço** |
| ***`LDAP Result Code 32 "No Such Object"`** | ***Base de busca inválida** | ***Corrigir `USER_BASE`** |
| ***`no such user`** | ***Filtro não encontrou o usuário** | ***Testar filtro com `ldp.exe`** |
| ***`connection refused`** | ***Firewall ou IP errado** | ***Liberar porta 389/636 no firewall** |

***Ferramenta de teste:**

***bash**

```
***# Instalar ldapsearch***

***apk add openldap-clients**


***# Testar conexão***

***ldapsearch -x -H ldap://192.168.0.10:389 \**

    ***-D "CN=svc_gitea_ldap,OU=ServiceAccounts,DC=ifrn,DC=local" \**

    ***-W \**

    ***-b "OU=Usuarios,DC=ifrn,DC=local" \**

    ***"(sAMAccountName=joao.silva)"**
```


## ***9. Proxy Reverso com Nginx (Portas 80/443)**

### ***9.1 Por que usar Proxy Reverso?**

- ***Acesso sem `:3000` na URL.**

- ***HTTPS (cadeado de segurança).**

- ***Melhor performance e cache.**

- ***Facilita balanceamento futuro.**

### ***9.2 Instalação do Nginx**

***bash**

```
***apk add nginx**

***rc-service nginx start**

***rc-update add nginx default**
```

### ***9.3 Configuração para HTTP (Porta 80)**

***Crie `/etc/nginx/http.d/gitea.conf`:**

***nginx**

```
***server {**

    ***listen 80;**

    ***server_name 192.168.0.9 alpinista-xy.ifrn.local;**


    ***client_max_body_size 512M;**


    ***location / {**

        ***proxy_pass http://127.0.0.1:3000;**

        ***proxy_set_header Host $host;**

        ***proxy_set_header X-Real-IP $remote_addr;**

        ***proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;**

        ***proxy_set_header X-Forwarded-Proto $scheme;**

    ***}**

***}**
```

***Recarregue:**

***bash**

```
***nginx -t**

***rc-service nginx restart**
```

***Agora acesse `http://192.168.0.9` (sem `:3000`).**

### ***9.4 Ajustar `app.ini` para o Proxy**

***Edite `/etc/gitea/app.ini`:**

***ini**

```
***[server]**

***ROOT_URL = http://192.168.0.9/**
```

***Reinicie:**

***bash**

```
***rc-service gitea restart**
```

### ***9.5 HTTPS com Certificado Autoassinado (Ambiente Interno)**

***Gere o certificado:**

***bash**

```
***mkdir -p /etc/ssl/gitea**

***openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \**

    ***-keyout /etc/ssl/gitea/gitea.key \**

    ***-out /etc/ssl/gitea/gitea.crt \**

    ***-subj "/C=BR/ST=RN/L=Natal/O=IFRN/CN=alpinista-xy.ifrn.local"**
```

***Adicione o bloco HTTPS em `/etc/nginx/http.d/gitea.conf`:**

***nginx**

```
***server {**

    ***listen 443 ssl;**

    ***server_name alpinista-xy.ifrn.local;**


    ***ssl_certificate     /etc/ssl/gitea/gitea.crt;**

    ***ssl_certificate_key /etc/ssl/gitea/gitea.key;**

    ***ssl_protocols TLSv1.2 TLSv1.3;**


    ***client_max_body_size 512M;**


    ***location / {**

        ***proxy_pass http://127.0.0.1:3000;**

        ***proxy_set_header Host $host;**

        ***proxy_set_header X-Real-IP $remote_addr;**

        ***proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;**

        ***proxy_set_header X-Forwarded-Proto $scheme;**

    ***}**

***}**
```

***Recarregue o Nginx e ajuste o `ROOT_URL`:**

***ini**

```
***[server]**

***ROOT_URL = https://alpinista-xy.ifrn.local/**
```

### ***9.6 HTTPS com Let's Encrypt (Domínio Público)**

***Só funciona com domínio público apontando para o servidor. Use `certbot`:**

***bash**

```
***apk add certbot certbot-nginx**

***certbot --nginx -d gitea.ifrn.edu.br**
```


## ***10. Manutenção e Backup**

### ***10.1 Backup Completo**

***O Gitea oferece um comando nativo de dump:**

***bash**

```
***su - gitea -c "gitea dump \**

    ***--config /etc/gitea/app.ini \**

    ***--file /var/lib/gitea/backup/gitea-backup-$(date +%Y%m%d).zip"**
```

***Isso gera um `.zip` contendo:**

- ***Banco de dados (dump SQL).**

- ***Repositórios Git.**

- ***Anexos, avatares, LFS.**

- ***Arquivo `app.ini`.**

### ***10.2 Backup do PostgreSQL Separadamente**

***bash**

```
***su postgres -c "pg_dump gitea > /var/lib/gitea/backup/gitea-db-$(date +%Y%m%d).sql"**
```

### ***10.3 Automatizar Backup (Cron)**

***Crie `/etc/periodic/daily/gitea-backup`:**

***bash**

```
***#!/bin/sh**

***BACKUP_DIR="/var/lib/gitea/backup"**

***mkdir -p "$BACKUP_DIR"**

***su - gitea -c "gitea dump --config /etc/gitea/app.ini \**

    ***--file ${BACKUP_DIR}/gitea-$(date +%Y%m%d).zip"**

***# Manter apenas os 7 mais recentes***

***ls -t ${BACKUP_DIR}/gitea-*.zip | tail -n +8 | xargs rm -f**
```

***Torne executável:**

***bash**

```
***chmod +x /etc/periodic/daily/gitea-backup**
```

### ***10.4 Restaurar Backup**

***bash**

```
***# Extrair***

***cd /tmp**

***unzip /var/lib/gitea/backup/gitea-20251010.zip -d gitea-restore**


***# Restaurar banco***

***su postgres -c "psql gitea < /tmp/gitea-restore/gitea-db.sql"**


***# Restaurar arquivos***

***cp -r /tmp/gitea-restore/repositories/* /var/lib/gitea/repositories/**

***cp -r /tmp/gitea-restore/data/* /var/lib/gitea/data/**

***cp /tmp/gitea-restore/app.ini /etc/gitea/app.ini**

***chown -R gitea:www-data /var/lib/gitea**


***rc-service gitea restart**
```

### ***10.5 Atualização do Gitea**

***bash**

```
***# 1. Fazer backup antes***

***su - gitea -c "gitea dump --config /etc/gitea/app.ini --file /var/lib/gitea/backup/pre-upgrade.zip"**


***# 2. Atualizar pacote***

***apk update**

***apk upgrade gitea gitea-openrc**


***# 3. Reiniciar serviço***

***rc-service gitea restart**


***# 4. Verificar versão***

***su - gitea -c "gitea --version"**
```


## ***11. Solução de Problemas**

### ***11.1 O serviço não inicia**

***bash**

```
***rc-service gitea status**

***tail -50 /var/log/gitea/gitea.log**


***# Verificar se o usuário tem permissão nos diretórios***

***ls -la /var/lib/gitea/**
```

***Causas comuns:**

- ***Permissões erradas em `/var/lib/gitea`.**

- ***`app.ini` com sintaxe inválida.**

- ***Banco de dados inacessível.**

### ***11.2 Erro `502 Bad Gateway` no Nginx**

***bash**

```
***# Gitea está rodando?***

***rc-service gitea status**


***# Porta 3000 responde?***

***curl -I http://127.0.0.1:3000**


***# Log do Nginx***

***tail -50 /var/log/nginx/error.log**
```

### ***11.3 Não consigo logar com usuário do AD**

1. ***Verifique se a fonte de autenticação está habilitada.**

2. ***Aumente o log para `Trace` e reinicie.**

3. ***Teste o filtro LDAP com `ldapsearch`.**

4. ***Verifique o firewall entre o Gitea e o DC.**

### ***11.4 `FATAL: password authentication failed for user "gitea"`**

***A senha no `app.ini` está diferente da senha do banco. Verifique:**

***bash**

```
***su postgres -c "psql -c \"ALTER USER gitea WITH PASSWORD 'nova_senha';\""**

***# Depois edite PASSWD em /etc/gitea/app.ini***

***rc-service gitea restart**
```

### ***11.5 Porta 3000 não acessível externamente**

***bash**

```
***# Verificar firewall***

***nft list ruleset | grep 3000**

***# ou***

***iptables -L -n | grep 3000**


***# Verificar se o Gitea está escutando em 0.0.0.0***

***ss -tlnp | grep 3000**

***# Deve mostrar: 0.0.0.0:3000***
```

***Se estiver escutando em `127.0.0.1:3000`, ajuste `HTTP_ADDR = 0.0.0.0` no `app.ini`.**

### ***11.6 Comandos Úteis de Diagnóstico**

***bash**

```
***# Status geral***

***rc-service gitea status**

***rc-service postgresql status**

***rc-service nginx status**


***# Logs em tempo real***

***tail -f /var/log/gitea/gitea.log**


***# Últimas 100 linhas***

***tail -100 /var/log/gitea/gitea.log**


***# Filtrar por erro***

***grep -i "error\|fail\|fatal" /var/log/gitea/gitea.log**


***# Verificar processos***

***ps aux | grep gitea**


***# Verificar conexões***

***ss -tnp | grep 3000**
```


## ***12. Referências**

### ***12.1 Documentação Oficial**

- ***Alpine Linux Wiki – Gitea: ~~[https://wiki.alpinelinux.org/wiki/Gitea**](https://wiki.alpinelinux.org/wiki/Gitea)

- ***Gitea Docs – Instalação: ~~[https://docs.gitea.com/installation/install-from-package**](https://docs.gitea.com/installation/install-from-package)

- ***Gitea Docs – Configuração (app.ini): ~~[https://docs.gitea.com/administration/config-cheat-sheet**](https://docs.gitea.com/administration/config-cheat-sheet)

- ***Gitea Docs – Autenticação LDAP: ~~[https://docs.gitea.com/administration/authentication**](https://docs.gitea.com/administration/authentication)

- ***PostgreSQL no Alpine: ~~[https://wiki.alpinelinux.org/wiki/PostgreSQL**](https://wiki.alpinelinux.org/wiki/PostgreSQL)

### ***12.2 Documentação Complementar**

- ***Gitea – Backup e Restore: ~~[https://docs.gitea.com/administration/backup-and-restore**](https://docs.gitea.com/administration/backup-and-restore)

- ***Nginx Proxy Reverso: ~~[https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/**](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

- ***Let's Encrypt / Certbot: ~~[https://certbot.eff.org/**](https://certbot.eff.org/)

### ***12.3 Arquivos Importantes no Servidor**

| ***Arquivo** | ***Descrição** |
| - | - |
| ***`/etc/gitea/app.ini`** | ***Configuração principal do Gitea** |
| ***`/etc/init.d/gitea`** | ***Script de serviço OpenRC** |
| ***`/var/lib/gitea/repositories/`** | ***Repositórios Git dos usuários** |
| ***`/var/lib/gitea/data/`** | ***Anexos, avatares, LFS** |
| ***`/var/lib/gitea/custom/`** | ***Customizações (templates, CSS)** |
| ***`/var/log/gitea/gitea.log`** | ***Log do Gitea** |
| ***`/var/lib/postgresql/data/`** | ***Dados do PostgreSQL** |

### ***12.4 Contatos e Suporte**

- ***Equipe de TI do IFRN: (inserir contato interno)**

- ***Comunidade Gitea: ~~[https://discourse.gitea.io/**](https://discourse.gitea.io/)

- ***Issues no GitHub: ~~[https://github.com/go-gitea/gitea/issues**](https://github.com/go-gitea/gitea/issues)


## ✅ ***Checklist Final de Implantação**

- ☑ 

- ***Script de instalação executado com sucesso**

- ☑ 

- ***PostgreSQL configurado e rodando**

- ☑ 

- ***Gitea rodando na porta 3000**

- ☑ 

- ***Serviço habilitado no boot (`rc-update`)**

- ☑ 

- ***Arquivo `app.ini` gerado com chaves seguras**

- ☑ 

- ***Usuário administrador criado**

- ☑ 

- ***Registro público desabilitado (se integrado ao AD)**

- ☑ 

- ***Fonte de autenticação LDAP configurada**

- ☑ 

- ***Sincronização de usuários do AD habilitada**

- ☑ 

- ***Mapeamento de grupos AD → Times do Gitea**

- ☑ 

- ***(Opcional) Nginx configurado como proxy reverso**

- ☑ 

- ***(Opcional) HTTPS habilitado**

- ☑ 

- ***Backup automático configurado**

- ☑ 

- ***Testes de login realizados com usuários do AD**


***Documento elaborado por: Equipe de TI – IFRN  
Última atualização: Outubro/2025  
Versão: 1.0**


***Esta documentação cobre todo o ciclo de vida da implantação do Gitea no Alpine Linux: instalação automatizada, criação do admin, integração com AD, proxy reverso, backup e troubleshooting. **

