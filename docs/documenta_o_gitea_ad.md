# Documentação — Gitea + Active Directory no Alpine Linux

> **Ambiente:** IFRN · `empresa.local` · Alpine Linux 3.23.x  
> **Elaborado por:** Equipe de TI  
> **Última atualização:** Outubro/2026 | **Versão:** 2.0

---

## 📑 Sumário

1. [Visão Geral](#1-visão-geral)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Arquitetura da Solução](#3-arquitetura-da-solução)
4. [Instalação Automatizada](#4-instalação-automatizada)
5. [Verificação Pós-Instalação](#5-verificação-pós-instalação)
6. [Criação do Usuário Administrador](#6-criação-do-usuário-administrador)
7. [Configurações Recomendadas Pós-Instalação](#7-configurações-recomendadas-pós-instalação)
8. [Integração com Active Directory (AD)](#8-integração-com-active-directory-ad)
   - [8.1 Método 1 — LDAP simples (sem TLS)](#81-método-1--ldap-simples-sem-tls-funcional)
   - [8.2 Método 2 — LDAPS com certificado no AD](#82-método-2--ldaps-com-certificado-no-ad-produção)
   - [8.3 Sincronização de Usuários do AD](#83-sincronização-de-usuários-do-ad)
   - [8.4 Mapeamento de Grupos do AD para Times do Gitea](#84-mapeamento-de-grupos-do-ad-para-times-do-gitea)
   - [8.5 Diagnóstico de Erros](#85-diagnóstico-de-erros)
9. [Proxy Reverso com Nginx (Portas 80/443)](#9-proxy-reverso-com-nginx-portas-80443)
10. [Manutenção e Backup](#10-manutenção-e-backup)
11. [Solução de Problemas](#11-solução-de-problemas)
12. [Referências](#12-referências)
13. [Checklist Final de Implantação](#13-checklist-final-de-implantação)

---

## 1. Visão Geral

### 1.1 Objetivo
Este documento descreve o processo completo de instalação, configuração e integração do **Gitea** — um serviço de hospedagem de repositórios Git leve e auto-hospedado — no servidor Alpine Linux, com autenticação delegada ao **Active Directory (AD)**.

### 1.2 O que é o Gitea?
O Gitea é uma aplicação web de código aberto escrita em Go, usada para hospedar repositórios Git. É uma alternativa leve ao GitHub, GitLab e Bitbucket, ideal para ambientes com recursos limitados:

* **Leve**: binário único, baixo consumo de RAM (~100 MB).
* **Multi-banco de dados**: suporta PostgreSQL, MySQL, MariaDB, SQLite e MSSQL.
* **Multi-plataforma**: roda em Linux, Windows, macOS e ARM.
* **Integração LDAP/AD**: autenticação delegada a diretórios corporativos.
* **API REST completa** e suporte a Webhooks.

### 1.3 Escopo
* Instalação automatizada via script Bash.
* Configuração do PostgreSQL como banco de dados.
* Inicialização do serviço via OpenRC.
* Criação do usuário administrador.
* Integração com Active Directory (AD) via LDAP — **dois métodos**: LDAP simples (laboratório) e LDAPS com certificado (produção).
* Proxy reverso com Nginx (opcional).

---

## 2. Pré-requisitos

### 2.1 Ambiente

| Item | Especificação |
| :--- | :--- |
| **Sistema Operacional** | Alpine Linux 3.23.x |
| **Arquitetura** | x86_64 |
| **Acesso** | root (via `su` ou SSH) |
| **RAM mínima** | 512 MB (recomendado: 1 GB) |
| **Disco mínimo** | 2 GB (recomendado: 10 GB+) |
| **Servidor Gitea** | `192.168.100.200` |
| **Domínio AD** | `empresa.local` |
| **Controlador de Domínio** | `192.168.100.10` |

### 2.2 Pacotes Necessários
* `git`, `git-lfs`
* `bash`, `curl`, `openssh`
* `coreutils`, `grep`, `less`, `lsof`, `dialog`
* `gitea`, `gitea-openrc`
* `postgresql17`, `postgresql17-contrib`, `postgresql17-openrc`

### 2.3 Portas Utilizadas

| Porta | Protocolo | Descrição |
| :--- | :--- | :--- |
| **3000** | TCP | Interface HTTP do Gitea |
| **22** | TCP | Acesso SSH aos repositórios Git |
| **5432** | TCP | PostgreSQL (apenas localhost) |
| **389** | TCP | LDAP (Bind DN + busca de usuários) |
| **636** | TCP | LDAPS (opcional — requer certificado no AD) |

---

## 3. Arquitetura da Solução

```text
┌─────────────────────────────────────────────────────────┐
│  Servidor Alpine Linux (192.168.100.200)                │
│                                                          │
│  ┌────────────────┐        ┌──────────────────────┐     │
│  │  Gitea (Go)    │◄──────►│  PostgreSQL 17       │     │
│  │  Porta 3000    │        │  Porta 5432 (local)  │     │
│  └────────┬───────┘        └──────────────────────┘     │
│           │                                              │
│           │ OpenRC (supervise-daemon)                   │
│           │                                              │
│  ┌────────▼───────┐        ┌──────────────────────┐     │
│  │  /etc/gitea/   │        │  /var/lib/gitea/     │     │
│  │  app.ini       │        │  repositories/       │     │
│  └────────────────┘        │  custom/             │     │
│                             │  data/               │     │
│                             └──────────────────────┘     │
└─────────────────────────────────────────────────────────┘
           │
           │ LDAP (porta 389) ou LDAPS (porta 636)
           ▼
┌─────────────────────────────────────────────────────────┐
│  Controlador de Domínio (Active Directory)              │
│  192.168.100.10 · empresa.local                         │
│                                                          │
│  Estrutura:                                              │
│  DC=empresa,DC=local                                     │
│   ├── CN=Users                                           │
│   │    └── svc_gitea_ldap  (conta de serviço)           │
│   ├── OU=Usuarios                                        │
│   │    └── OU=Vendas                                     │
│   │         └── CN=Carlos Gomes  (usuário)              │
│   └── OU=Grupos                                          │
│        └── CN=Gitea_Admins  (grupo admin)               │
└─────────────────────────────────────────────────────────┘
```

**Fluxo de autenticação:**
1. Usuário acessa `http://192.168.100.200:3000`.
2. Gitea serve a interface web (embutida em Go).
3. No login, o Gitea consulta o AD via LDAP:
   1. **Bind DN** → autentica como `svc_gitea_ldap`.
   2. **Busca** → localiza o usuário pelo `sAMAccountName`.
   3. **Bind novamente** → valida a senha do usuário.
4. Se autenticado, cria sessão local e libera acesso.

---

## 4. Instalação Automatizada

### 4.1 Script Completo: `gitea-alpine-install.sh`
Salve o script abaixo como `/root/gitea-alpine-install.sh`:

```bash
#!/usr/bin/env bash
#
# gitea-alpine-install.sh
# Instalação completa do Gitea no Alpine Linux 3.23.x
# Baseado em: https://wiki.alpinelinux.org/wiki/Gitea
#
set -euo pipefail

# ==================================================================
# 1. VARIÁVEIS CONFIGURÁVEIS
# ==================================================================
DB_TYPE="${DB_TYPE:-postgres}"
DB_NAME="${DB_NAME:-gitea}"
DB_USER="${DB_USER:-gitea}"
GITEA_HTTP_PORT="${GITEA_HTTP_PORT:-3000}"
GITEA_SSH_PORT="${GITEA_SSH_PORT:-22}"
GITEA_APP_INI="/etc/gitea/app.ini"
GITEA_WORK_DIR="/var/lib/gitea"
GITEA_LOG_DIR="/var/log/gitea"
GITEA_USER="gitea"
GITEA_GROUP="www-data"

# ==================================================================
# 2. FUNÇÕES AUXILIARES
# ==================================================================
log()   { printf '\033[1;34m[INFO]\033[0m %s\n' "$*"; }
error() { printf '\033[1;31m[ERRO]\033[0m %s\n' "$*" >&2; exit 1; }

gen_random() {
    head -c 512 /dev/urandom | LC_ALL=C tr -dc 'A-Za-z0-9' | cut -c1-"${1:-24}"
}

cleanup() {
    local exit_code=$?
    if [[ $exit_code -ne 0 ]]; then
        printf '\033[1;31m[ERRO]\033[0m Instalação interrompida (código: %d).\n' "$exit_code" >&2
    fi
}
trap cleanup EXIT

# ==================================================================
# 3. VERIFICAR AMBIENTE ALPINE
# ==================================================================
[[ -f /etc/alpine-release ]] || error "Este script é destinado ao Alpine Linux."

ALPINE_VERSION=$(cat /etc/alpine-release)
log "Alpine Linux versão: ${ALPINE_VERSION}"

ALPINE_MAJOR_MINOR="$(echo "${ALPINE_VERSION}" | cut -d. -f1,2)"
if [ "$(printf '%s\n' "3.23" "${ALPINE_MAJOR_MINOR}" | sort -V | head -n1)" != "3.23" ]; then
    log "Aviso: O script foi testado no Alpine 3.23.x."
fi

# ==================================================================
# 4. GARANTIR RESOLUÇÃO DO HOSTNAME
# ==================================================================
HOSTNAME_SHORT="$(hostname)"
if ! grep -qE "^\s*127\.0\.1\.1\s+${HOSTNAME_SHORT}\b" /etc/hosts 2>/dev/null; then
    log "Adicionando ${HOSTNAME_SHORT} a /etc/hosts..."
    echo "127.0.1.1    ${HOSTNAME_SHORT}" >> /etc/hosts
fi

GITEA_DOMAIN="${GITEA_DOMAIN:-$(hostname -f 2>/dev/null || hostname)}"
GITEA_ROOT_URL="${GITEA_ROOT_URL:-http://${GITEA_DOMAIN}:${GITEA_HTTP_PORT}/}"

log "Domínio Gitea: ${GITEA_DOMAIN}"
log "URL raiz:      ${GITEA_ROOT_URL}"

# ==================================================================
# 5. INSTALAR DEPENDÊNCIAS BÁSICAS
# ==================================================================
log "Atualizando repositórios..."
apk update

log "Instalando dependências básicas..."
apk add --no-cache \
    git git-lfs bash curl openssh \
    coreutils grep less lsof dialog

# ==================================================================
# 6. INSTALAR O GITEA E O SCRIPT OPENRC
# ==================================================================
log "Instalando Gitea e script OpenRC..."
apk add --no-cache gitea gitea-openrc

if ! id -u "${GITEA_USER}" &>/dev/null; then
    log "Criando usuário ${GITEA_USER}..."
    adduser -S -D -H -h "${GITEA_WORK_DIR}" -s /sbin/nologin -G "${GITEA_GROUP}" "${GITEA_USER}" 2>/dev/null || \
        adduser -S -D -H -h "${GITEA_WORK_DIR}" -s /sbin/nologin "${GITEA_USER}"
fi

if ! id -nG "${GITEA_USER}" 2>/dev/null | grep -q "\b${GITEA_GROUP}\b"; then
    log "Adicionando ${GITEA_USER} ao grupo ${GITEA_GROUP}..."
    addgroup "${GITEA_USER}" "${GITEA_GROUP}" 2>/dev/null || true
fi

# ==================================================================
# 7. INSTALAR E CONFIGURAR O BANCO DE DADOS
# ==================================================================
if [[ "${DB_TYPE}" == "postgres" ]]; then
    log "Instalando PostgreSQL 17..."
    apk add --no-cache postgresql17 postgresql17-contrib postgresql17-openrc

    if [[ ! -f /var/lib/postgresql/data/PG_VERSION ]]; then
        log "Cluster não inicializado. Preparando para setup..."

        if rc-service postgresql status 2>/dev/null | grep -q "started"; then
            log "Parando PostgreSQL para permitir o setup..."
            rc-service postgresql stop
            sleep 2
        fi

        log "Inicializando cluster PostgreSQL..."
        /etc/init.d/postgresql setup
    else
        log "Cluster PostgreSQL já inicializado. Pulando setup."
    fi

    rc-service postgresql start
    rc-update add postgresql default

    DB_PASS="$(gen_random 24)"

    log "Criando usuário '${DB_USER}' e banco '${DB_NAME}'..."
    su postgres -c "psql -c \"CREATE USER ${DB_USER} WITH ENCRYPTED PASSWORD '${DB_PASS}';\"" 2>/dev/null || true
    su postgres -c "psql -c \"CREATE DATABASE ${DB_NAME} OWNER ${DB_USER};\"" 2>/dev/null || true
    su postgres -c "psql -c \"GRANT ALL PRIVILEGES ON DATABASE ${DB_NAME} TO ${DB_USER};\"" 2>/dev/null || true

    DB_HOST="127.0.0.1:5432"

elif [[ "${DB_TYPE}" == "mariadb" ]]; then
    log "Instalando MariaDB..."
    apk add --no-cache mariadb mariadb-client

    if [[ ! -d /var/lib/mysql/mysql ]]; then
        log "Inicializando banco MariaDB..."
        mysql_install_db --user=mysql --datadir=/var/lib/mysql
    fi

    rc-service mariadb start
    rc-update add mariadb default

    DB_PASS="$(gen_random 24)"

    log "Criando banco '${DB_NAME}' e usuário '${DB_USER}'..."
    mysql -u root <<-EOF
        CREATE DATABASE IF NOT EXISTS \`${DB_NAME}\` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
        CREATE USER IF NOT EXISTS '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PASS}';
        GRANT ALL PRIVILEGES ON \`${DB_NAME}\`.* TO '${DB_USER}'@'localhost';
        FLUSH PRIVILEGES;
EOF

    DB_HOST="127.0.0.1:3306"
else
    error "Tipo de banco inválido: '${DB_TYPE}'. Use 'postgres' ou 'mariadb'."
fi

# ==================================================================
# 8. CRIAR DIRETÓRIOS DO GITEA
# ==================================================================
log "Criando diretórios de trabalho, log e customização..."
mkdir -p "${GITEA_WORK_DIR}"/{custom,data,repositories}
mkdir -p "${GITEA_LOG_DIR}"

chown -R "${GITEA_USER}:${GITEA_GROUP}" "${GITEA_WORK_DIR}"
chown -R "${GITEA_USER}:${GITEA_GROUP}" "${GITEA_LOG_DIR}"
chmod -R 750 "${GITEA_WORK_DIR}"
chmod -R 750 "${GITEA_LOG_DIR}"

# ==================================================================
# 9. CONFIGURAR O ARQUIVO app.ini
# ==================================================================
log "Gerando ${GITEA_APP_INI}..."

if [[ -f "${GITEA_APP_INI}" ]]; then
    BACKUP="${GITEA_APP_INI}.bak.$(date +%s)"
    cp "${GITEA_APP_INI}" "${BACKUP}"
    log "Backup criado: ${BACKUP}"
fi

mkdir -p "$(dirname "${GITEA_APP_INI}")"

SECRET_KEY="$(gen_random 32)"
INTERNAL_TOKEN="$(gen_random 32)"
JWT_SECRET="$(gen_random 32)"

if [[ "${DB_TYPE}" == "mariadb" ]]; then
    GITEA_DB_TYPE="mysql"
else
    GITEA_DB_TYPE="${DB_TYPE}"
fi

cat > "${GITEA_APP_INI}" <<EOF
APP_NAME = Gitea: Git with a cup of tea
RUN_USER = ${GITEA_USER}
RUN_MODE = prod

[server]
PROTOCOL         = http
DOMAIN           = ${GITEA_DOMAIN}
HTTP_ADDR        = 0.0.0.0
HTTP_PORT        = ${GITEA_HTTP_PORT}
ROOT_URL         = ${GITEA_ROOT_URL}
DISABLE_SSH      = false
SSH_PORT         = ${GITEA_SSH_PORT}
SSH_LISTEN_PORT  = ${GITEA_SSH_PORT}
OFFLINE_MODE     = false
LFS_START_SERVER = true

[database]
DB_TYPE  = ${GITEA_DB_TYPE}
HOST     = ${DB_HOST}
NAME     = ${DB_NAME}
USER     = ${DB_USER}
PASSWD   = ${DB_PASS}
SCHEMA   = public
SSL_MODE = disable

[repository]
ROOT = ${GITEA_WORK_DIR}/repositories

[security]
INSTALL_LOCK   = true
SECRET_KEY     = ${SECRET_KEY}
INTERNAL_TOKEN = ${INTERNAL_TOKEN}

[oauth2]
JWT_SECRET = ${JWT_SECRET}

[service]
DISABLE_REGISTRATION   = false
REQUIRE_SIGNIN_VIEW    = false
REGISTER_EMAIL_CONFIRM = false

[log]
MODE      = console
LEVEL     = info
ROOT_PATH = ${GITEA_LOG_DIR}

[mailer]
ENABLED = false

[session]
PROVIDER = file

[picture]
AVATAR_UPLOAD_PATH            = ${GITEA_WORK_DIR}/data/avatars
REPOSITORY_AVATAR_UPLOAD_PATH = ${GITEA_WORK_DIR}/data/repo-avatars

[attachment]
PATH = ${GITEA_WORK_DIR}/data/attachments
EOF

chown "${GITEA_USER}:${GITEA_GROUP}" "${GITEA_APP_INI}"
chmod 640 "${GITEA_APP_INI}"

log "Arquivo ${GITEA_APP_INI} criado com sucesso."

# ==================================================================
# 10. EXECUTAR MIGRAÇÕES DO BANCO DE DADOS
# ==================================================================
log "Executando migrações do banco de dados..."
su - "${GITEA_USER}" -c "GITEA_WORK_DIR=${GITEA_WORK_DIR} gitea migrate --config ${GITEA_APP_INI}"

# ==================================================================
# 11. HABILITAR E INICIAR O SERVIÇO GITEA
# ==================================================================
log "Habilitando e iniciando o serviço Gitea..."
rc-update add gitea default
rc-service gitea start

sleep 3
if rc-service gitea status | grep -q "started"; then
    log "Serviço Gitea está em execução."
else
    log "Aviso: Verifique com: rc-service gitea status"
fi

# ==================================================================
# 12. INFORMAÇÕES FINAIS
# ==================================================================
echo ""
echo "======================================================"
echo "  Gitea instalado e configurado com sucesso!"
echo "======================================================"
echo "  URL de acesso:      ${GITEA_ROOT_URL}"
echo "  Porta HTTP:         ${GITEA_HTTP_PORT}"
echo "  Porta SSH:          ${GITEA_SSH_PORT}"
echo ""
echo "  Banco de dados:     ${DB_TYPE}"
echo "  Host do banco:      ${DB_HOST}"
echo "  Nome do banco:      ${DB_NAME}"
echo "  Usuário do banco:   ${DB_USER}"
echo "  Senha do banco:     ${DB_PASS}"
echo "======================================================"
echo ""
```

### 4.2 Como Executar

```bash
# 1. Como root
su -

# 2. Salve o script
nano /root/gitea-alpine-install.sh

# 3. Permissão de execução
chmod +x /root/gitea-alpine-install.sh

# 4. Execute
cd /root
./gitea-alpine-install.sh
```

### 4.3 Variáveis Configuráveis

| Variável | Padrão | Descrição |
| :--- | :--- | :--- |
| `DB_TYPE` | `postgres` | `postgres` ou `mariadb` |
| `DB_NAME` | `gitea` | Nome do banco de dados |
| `DB_USER` | `gitea` | Usuário do banco de dados |
| `GITEA_HTTP_PORT` | `3000` | Porta HTTP do Gitea |
| `GITEA_SSH_PORT` | `22` | Porta SSH do Gitea |

---

## 5. Verificação Pós-Instalação

### 5.1 Checar Serviços

```bash
# PostgreSQL
rc-service postgresql status

# Gitea
rc-service gitea status

# Habilitados no boot
rc-update show | grep -E 'gitea|postgresql'
```

### 5.2 Verificar Portas

```bash
ss -tlnp | grep -E ':3000|:5432'
```

### 5.3 Testar HTTP

```bash
curl -I http://localhost:3000
# Esperado: HTTP/1.1 200 OK
```

### 5.4 Verificar Estrutura de Diretórios

```bash
ls -la /etc/gitea/
ls -la /var/lib/gitea/
ls -la /var/log/gitea/

stat -c "%U:%G %a %n" /etc/gitea/app.ini
# Esperado: gitea:www-data 640 /etc/gitea/app.ini
```

---

## 6. Criação do Usuário Administrador

O primeiro usuário registrado no Gitea se torna automaticamente **administrador**.

### 6.1 Método A: Pela Interface Web
1. Acesse `http://192.168.100.200:3000`.
2. Clique em **"Need an account? Register now."**
3. Preencha:
   * **Nome de usuário:** `admin`
   * **E-mail:** `admin@empresa.local`
   * **Senha:** (forte, mínimo 8 caracteres)
4. Clique em **Registrar**.

### 6.2 Método B: Pelo Terminal (Recomendado)

```bash
su - gitea -c "gitea admin user create \
    --admin \
    --username admin \
    --password 'SuaSenhaForteAqui123!' \
    --email 'admin@empresa.local' \
    --config /etc/gitea/app.ini"
```

**Saída esperada:** `New user 'admin' has been successfully created!`

### 6.3 Listar Usuários

```bash
su - gitea -c "gitea admin user list --config /etc/gitea/app.ini"
```

### 6.4 Alterar Senha

```bash
su - gitea -c "gitea admin user change-password \
    --username admin \
    --password 'NovaSenhaForte!' \
    --config /etc/gitea/app.ini"
```

---

## 7. Configurações Recomendadas Pós-Instalação

### 7.1 Desabilitar Registro Público
Depois de integrar com AD, apenas usuários do domínio devem poder logar.

> **Caminho Web:** Administração do Site → Configurações → Desmarque "Permitir Registro de Usuários"

Ou via `app.ini`:

```ini
[service]
DISABLE_REGISTRATION = true
```

### 7.2 Configurar E-mail (Opcional)

```ini
[mailer]
ENABLED = true
HOST = smtp.empresa.local:587
FROM = gitea@empresa.local
USER = gitea@empresa.local
PASSWD = senha_do_email
```

### 7.3 Configurar Captcha (Opcional)

```ini
[service]
ENABLE_CAPTCHA = true
```

### 7.4 Reiniciar Após Alterações

```bash
rc-service gitea restart
```

---

## 8. Integração com Active Directory (AD)

O Gitea oferece suporte nativo a autenticação LDAP/AD. Este documento cobre **dois métodos**:

| Método | Protocolo | Porta | Segurança | Indicado para |
| :--- | :--- | :--- | :--- | :--- |
| **Método 1** — LDAP simples | LDAP (Unencrypted) | 389 | ⚠️ Senhas em texto claro | **LABORATÓRIO** |
| **Método 2** — LDAPS | LDAPS (SSL/TLS) | 636 | ✅ Criptografado | **PRODUÇÃO** |

---

### 8.1 Método 1 — LDAP simples (sem TLS) `FUNCIONAL`

> **✅ Método validado em laboratório**  
> Este método foi testado com sucesso no domínio `empresa.local`. É a solução mais rápida para colocar o laboratório em funcionamento.

#### Pré-requisitos no AD
Antes de configurar o Gitea, garanta que existam no AD:
* **Conta de serviço**: `svc_gitea_ldap`
  * DN: `CN=svc_gitea_ldap,CN=Users,DC=empresa,DC=local`
  * Permissão: apenas leitura nos objetos de usuário
* **Usuários** cadastrados em alguma OU (ex: `OU=Usuarios,OU=Vendas,DC=empresa,DC=local`)

#### Configuração no Gitea
1. Faça login como `admin`.
2. Vá em **Administração do Site** → **Identidade & Acesso** → **Fontes de Autenticação**.
3. Clique em **Adicionar Fonte de Autenticação** e selecione **LDAP (via BindDN)**.
4. Preencha conforme a tabela abaixo:

| Campo | Valor |
| :--- | :--- |
| **Tipo de autenticação** | LDAP (via BindDN) |
| **Nome da autenticação** | `svc_gitea_ldap` |
| **Protocolo de segurança** | **Sem criptografia** (Unencrypted) |
| **Servidor** | `192.168.100.10` |
| **Porta** | `389` |
| **Pular verificação de TLS** | Marcado (irrelevante sem TLS) |
| **Vincular DN** | `cn=svc_gitea_ldap,cn=Users,dc=empresa,dc=local` |
| **Vincular senha** | *(senha da conta de serviço)* |
| **Base de pesquisa do usuário** | `dc=empresa,dc=local` |
| **Filtro de usuário** | `(&(objectCategory=Person)(sAMAccountName=%s)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))` |
| **Filtro de administrador** | *(deixe em branco para testes)* |
| **Filtro de restrição** | *(deixe em branco)* |
| **Atributo nome de usuário** | `sAMAccountName` |
| **Atributo primeiro nome** | `givenName` |
| **Atributo sobrenome** | `sn` |
| **Atributo e-mail** | `userPrincipalName` ⬅️ **IMPORTANTE** |

> **⚠️ Por que usar `userPrincipalName` e não `mail`?**  
> Em muitos laboratórios, o atributo `mail` do AD está vazio. O Gitea **exige** um e-mail único para cada usuário. Usando `userPrincipalName` (que sempre tem o formato `usuario@dominio`), garantimos que o Gitea consiga criar a conta.  
> *Em produção, o ideal é preencher o `mail` de cada usuário no AD e trocar de volta para `mail`.*

#### Validar no terminal (antes de testar no Gitea)

```bash
# Instale o cliente LDAP
apk add openldap-clients

# Teste 1: bind com a conta de serviço
ldapsearch -x -H ldap://192.168.100.10:389 \
    -D "cn=svc_gitea_ldap,cn=Users,dc=empresa,dc=local" \
    -w "Senha.senha" \
    -b "dc=empresa,dc=local" \
    -s base "(objectclass=*)"

# Teste 2: buscar um usuário específico
ldapsearch -x -H ldap://192.168.100.10:389 \
    -D "cn=svc_gitea_ldap,cn=Users,dc=empresa,dc=local" \
    -w "Senha.senha" \
    -b "dc=empresa,dc=local" \
    "(sAMAccountName=carlosgomes)"
```

**Resultado esperado:** retorno com os atributos do usuário (`sAMAccountName`, `displayName`, etc.).

#### Testar o login
1. Faça logout do Gitea.
2. Na tela de login, use:
   * **Usuário:** `carlosgomes` (o `sAMAccountName`)
   * **Senha:** a senha do AD
3. Se funcionar, o usuário será criado automaticamente no Gitea.

---

### 8.2 Método 2 — LDAPS com certificado no AD `PRODUÇÃO`

Este método usa criptografia forte (LDAPS). Para isso, o controlador de domínio precisa ter um **certificado válido no repositório de serviço NTDS**. Sem isso, o AD rejeita qualquer tentativa de TLS.

> **🚫 Sintoma de certificado ausente**  
> Ao tentar LDAPS sem certificado, você verá no terminal:  
> `ldap_start_tls: Server is unavailable (52)`  
> `additional info: LdapErr: DSID-0C0915E3, comment: Error initializing SSL/TLS`  
>   
> Ou, no log do Gitea:  
> `LDAP Connect error: read: connection reset by peer`

#### Passo 1: Verificar o certificado no AD
No servidor Windows do AD, abra o **PowerShell como Administrador** e execute:

```powershell
certutil -store -service NTDS
```
Se retornar vazio ou apenas certificados sem **"Server Authentication"**, é preciso gerar um novo.

#### Passo 2: Gerar certificado autoassinado
No PowerShell do AD (Administrador):

```powershell
$cert = New-SelfSignedCertificate `
    -DnsName "SERVIDOR1.empresa.local", "empresa.local" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyAlgorithm RSA `
    -KeyLength 2048 `
    -NotAfter (Get-Date).AddYears(5) `
    -KeyUsage DigitalSignature, KeyEncipherment `
    -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.1")

$cert.Thumbprint
```

*Ou, se preferir interface gráfica, use `certlm.msc` e solicite um certificado usando o template **"Kerberos Authentication"**.*

#### Passo 3: Mover o certificado para o repositório NTDS
O AD **não usa** o certificado do repositório `LocalMachine\My` para LDAPS. Ele exige o repositório específico do serviço NTDS.

```powershell
# No PowerShell (como Administrador)
$thumbprint = "COLE_AQUI_O_THUMBPRINT"

$pwd = ConvertTo-SecureString -String "SenhaTemp123!" -Force -AsPlainText
Export-PfxCertificate -Cert "Cert:\LocalMachine\My\$thumbprint" `
    -FilePath "C:\temp\ldaps.pfx" -Password $pwd

# Importar no repositório NTDS
certutil -importPFX -service NTDS C:\temp\ldaps.pfx NoExport
```

#### Passo 4: Reiniciar o serviço NTDS

```powershell
Restart-Service NTDS -Force
```
Aguarde 1 minuto para o serviço subir completamente.

#### Passo 5: Testar do servidor Alpine

```bash
# Testar a conexão TLS
openssl s_client -connect 192.168.100.10:636 -showcerts

# Testar o bind com LDAPS
ldapsearch -x -H ldaps://192.168.100.10:636 \
    -D "cn=svc_gitea_ldap,cn=Users,dc=empresa,dc=local" \
    -w "Senha.senha" \
    -b "dc=empresa,dc=local" \
    -s base "(objectclass=*)"
```

Se o `openssl s_client` exibir os detalhes do certificado, o handshake TLS está funcionando.

#### Passo 6 (se necessário): Importar a CA do AD no Alpine
Se o certificado for emitido por uma CA interna, importe-a no Alpine para que o sistema confie nela:

```bash
# Copie o certificado da CA para o servidor
cp /caminho/ad-ca.crt /usr/local/share/ca-certificates/

# Atualize o repositório de certificados
update-ca-certificates

# Reinicie o Gitea
rc-service gitea restart
```

#### Passo 7: Atualizar a fonte de autenticação no Gitea
Edite a fonte `svc_gitea_ldap` e altere:

| Campo | Valor |
| :--- | :--- |
| **Protocolo de segurança** | **LDAPS** |
| **Porta** | `636` |
| **Pular verificação de TLS** | Marcado (para certificado autoassinado)<br>Desmarcado (se o certificado for de CA confiável) |

Salve e teste o login novamente.

> **💡 Sobre o certificado autoassinado**  
> Se você marcou **"Pular verificação de TLS"**, o Gitea aceitará qualquer certificado. Isso é aceitável em produção interna, mas o ideal é usar um certificado emitido por uma CA confiável (interna ou pública).

---

### 8.3 Sincronização de Usuários do AD

O Gitea pode importar periodicamente todos os usuários do AD. Na configuração da fonte de autenticação, marque **"Ativar Sincronização de Usuário"**.

Ajuste a frequência no `app.ini`:

```ini
[cron.sync_external_users]
RUN_AT_START = true
SCHEDULE = 0 0 3 * * *    # todo dia às 03:00
```

Para forçar uma sincronização imediata via CLI:

```bash
su - gitea -c "gitea admin auth sync --config /etc/gitea/app.ini"
```

---

### 8.4 Mapeamento de Grupos do AD para Times do Gitea

1. Crie uma **Organização** no Gitea (ex: `ifrn`).
2. Vá em **Organização → Equipes** e crie uma equipe (ex: `Desenvolvedores`).
3. Edite a equipe e vá em **Sincronização de Grupo Externo**.
4. Informe o DN do grupo do AD:
   ```text
   CN=Desenvolvedores,OU=Grupos,DC=empresa,DC=local
   ```
5. Associe o nível de permissão (Read/Write/Admin).
6. Ao sincronizar, os membros do grupo AD entram automaticamente na equipe.

Para definir administradores do Gitea via grupo do AD, preencha o campo **Filtro de administrador** com:

```text
(memberOf=CN=Gitea_Admins,OU=Grupos,DC=empresa,DC=local)
```

---

### 8.5 Diagnóstico de Erros

Aumente o nível de log para diagnosticar problemas no `app.ini`:

```ini
[log]
LEVEL = Trace
```

Reinicie e acompanhe em tempo real:

```bash
rc-service gitea restart
tail -f /var/log/gitea/gitea.log
```

#### Erros comuns e soluções

| Erro no log | Causa | Solução |
| :--- | :--- | :--- |
| `LDAP Result Code 200 "Network Error": read: connection reset by peer` | Falha no handshake TLS (certificado ausente/inválido no AD) | Use LDAP simples (porta 389) ou corrija o certificado no AD |
| `LDAP Result Code 200 "Network Error": dial tcp ...: i/o timeout` | Porta errada ou firewall bloqueando | Verifique a porta (389 ou 636) e o firewall |
| `LDAP Result Code 49 "Invalid Credentials"` | Senha do Bind DN incorreta | Confirme a senha da conta de serviço no AD |
| `LDAP Result Code 32 "No Such Object"` | Base de busca inválida | Corrija o campo "Base de pesquisa do usuário" |
| `Email address does not exist` | Atributo `mail` vazio no AD | Use `userPrincipalName` como atributo e-mail |
| `user does not exist` | Filtro de usuário não encontrou a conta | Teste o filtro com `ldapsearch` |
| `Error initializing SSL/TLS, data 0, v4563` | Serviço LDAP do AD sem certificado no repositório NTDS | Corrija o certificado no AD (Método 2) |

#### Ferramentas de diagnóstico

```bash
# Instalar cliente LDAP
apk add openldap-clients

# Testar bind com conta de serviço
ldapsearch -x -H ldap://192.168.100.10:389 \
    -D "cn=svc_gitea_ldap,cn=Users,dc=empresa,dc=local" \
    -w "Senha" \
    -b "dc=empresa,dc=local" \
    -s base "(objectclass=*)"

# Testar handshake TLS
openssl s_client -connect 192.168.100.10:636 -showcerts

# Testar conectividade da porta
nc -zv 192.168.100.10 636
```

---

## 9. Proxy Reverso com Nginx (Portas 80/443)

### 9.1 Por que usar Proxy Reverso?
* Acesso sem `:3000` na URL.
* HTTPS (cadeado de segurança).
* Melhor performance e cache.

### 9.2 Instalação do Nginx

```bash
apk add nginx
rc-service nginx start
rc-update add nginx default
```

### 9.3 Configuração HTTP (Porta 80)
Crie `/etc/nginx/http.d/gitea.conf`:

```nginx
server {
    listen 80;
    server_name 192.168.100.200 empresa.local;

    client_max_body_size 512M;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
nginx -t
rc-service nginx restart
```

Ajuste o `app.ini`:

```ini
[server]
ROOT_URL = http://192.168.100.200/
```

### 9.4 HTTPS com Certificado Autoassinado

```bash
mkdir -p /etc/ssl/gitea
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
    -keyout /etc/ssl/gitea/gitea.key \
    -out /etc/ssl/gitea/gitea.crt \
    -subj "/C=BR/ST=RN/L=Natal/O=IFRN/CN=empresa.local"
```

```nginx
server {
    listen 443 ssl;
    server_name empresa.local;

    ssl_certificate     /etc/ssl/gitea/gitea.crt;
    ssl_certificate_key /etc/ssl/gitea/gitea.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    client_max_body_size 512M;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 10. Manutenção e Backup

### 10.1 Backup Completo

```bash
su - gitea -c "gitea dump \
    --config /etc/gitea/app.ini \
    --file /var/lib/gitea/backup/gitea-backup-$(date +%Y%m%d).zip"
```

### 10.2 Backup do PostgreSQL

```bash
su postgres -c "pg_dump gitea > /var/lib/gitea/backup/gitea-db-$(date +%Y%m%d).sql"
```

### 10.3 Automatizar (Cron)
Crie `/etc/periodic/daily/gitea-backup`:

```bash
#!/bin/sh
BACKUP_DIR="/var/lib/gitea/backup"
mkdir -p "$BACKUP_DIR"
su - gitea -c "gitea dump --config /etc/gitea/app.ini \
    --file ${BACKUP_DIR}/gitea-$(date +%Y%m%d).zip"
ls -t ${BACKUP_DIR}/gitea-*.zip | tail -n +8 | xargs rm -f
```

```bash
chmod +x /etc/periodic/daily/gitea-backup
```

### 10.4 Restaurar Backup

```bash
cd /tmp
unzip /var/lib/gitea/backup/gitea-20251012.zip -d gitea-restore

su postgres -c "psql gitea < /tmp/gitea-restore/gitea-db.sql"

cp -r /tmp/gitea-restore/repositories/* /var/lib/gitea/repositories/
cp -r /tmp/gitea-restore/data/* /var/lib/gitea/data/
cp /tmp/gitea-restore/app.ini /etc/gitea/app.ini
chown -R gitea:www-data /var/lib/gitea

rc-service gitea restart
```

### 10.5 Atualização do Gitea

```bash
su - gitea -c "gitea dump --config /etc/gitea/app.ini --file /var/lib/gitea/backup/pre-upgrade.zip"

apk update
apk upgrade gitea gitea-openrc

rc-service gitea restart
su - gitea -c "gitea --version"
```

---

## 11. Solução de Problemas

### 11.1 O serviço não inicia

```bash
rc-service gitea status
tail -50 /var/log/gitea/gitea.log
ls -la /var/lib/gitea/
```

### 11.2 Erro 502 Bad Gateway no Nginx

```bash
rc-service gitea status
curl -I http://127.0.0.1:3000
tail -50 /var/log/nginx/error.log
```

### 11.3 Login do AD falha
1. Verifique se a fonte de autenticação está habilitada.
2. Aumente o nível de log para `Trace`.
3. Teste a consulta com `ldapsearch`.
4. Confira o firewall entre o servidor Gitea e o Active Directory.

### 11.4 Comandos úteis de diagnóstico

```bash
# Status geral dos serviços
rc-service gitea status
rc-service postgresql status
rc-service nginx status

# Logs em tempo real
tail -f /var/log/gitea/gitea.log

# Filtrar erros no log
grep -i "error\|fail\|fatal" /var/log/gitea/gitea.log

# Verificar processos
ps aux | grep gitea

# Verificar conexões ativas na porta 3000
ss -tnp | grep 3000
```

---

## 12. Referências

### 12.1 Documentação Oficial
* **Alpine Linux Wiki – Gitea:** [https://wiki.alpinelinux.org/wiki/Gitea](https://wiki.alpinelinux.org/wiki/Gitea)
* **Gitea Docs – Instalação:** [https://docs.gitea.com/installation/install-from-package](https://docs.gitea.com/installation/install-from-package)
* **Gitea Docs – app.ini:** [https://docs.gitea.com/administration/config-cheat-sheet](https://docs.gitea.com/administration/config-cheat-sheet)
* **Gitea Docs – LDAP:** [https://docs.gitea.com/administration/authentication](https://docs.gitea.com/administration/authentication)
* **PostgreSQL no Alpine:** [https://wiki.alpinelinux.org/wiki/PostgreSQL](https://wiki.alpinelinux.org/wiki/PostgreSQL)

### 12.2 Arquivos Importantes

| Arquivo | Descrição |
| :--- | :--- |
| `/etc/gitea/app.ini` | Configuração principal do Gitea |
| `/etc/init.d/gitea` | Script de serviço OpenRC |
| `/var/lib/gitea/repositories/` | Repositórios Git dos usuários |
| `/var/lib/gitea/data/` | Anexos, avatares, LFS |
| `/var/lib/gitea/custom/` | Customizações da aplicação |
| `/var/log/gitea/gitea.log` | Log do Gitea |

---

## 13. Checklist Final de Implantação

- [x] Script de instalação executado com sucesso
- [x] PostgreSQL configurado e rodando
- [x] Gitea rodando na porta 3000
- [x] Serviço habilitado no boot
- [x] Arquivo `app.ini` gerado com chaves seguras
- [x] Usuário administrador criado
- [x] Conta de serviço `svc_gitea_ldap` criada no AD
- [x] Bind com conta de serviço validado
- [x] Fonte LDAP configurada no Gitea
- [x] Teste de login com usuário do AD bem-sucedido
- [ ] Registro público desabilitado
- [ ] Sincronização periódica habilitada
- [ ] Mapeamento de grupos AD → Times do Gitea
- [ ] (Opcional) Certificado LDAPS instalado no AD
- [ ] (Opcional) Nginx configurado como proxy reverso
- [ ] (Opcional) HTTPS habilitado
- [ ] Backup automático configurado

---

*Documentação de instalação e integração do Gitea com Active Directory no Alpine Linux 3.23.x*