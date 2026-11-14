# Guia de Implantação do GLPI com Podman Desktop no Linux Fedora

Este documento apresenta o passo a passo completo para implantar o **GLPI** integrado a um banco de dados **MariaDB** utilizando o **Podman Desktop** e a CLI nativa no **Linux Fedora**.

---

## 🚀 Pré-requisitos

Antes de iniciar, certifique-se de ter o Podman e o plugin de compose instalados no seu Fedora. Caso não tenha, instale-os abrindo o terminal e executando:

```bash
sudo dnf update -y
sudo dnf install podman podman-compose -y
```

---

## 📂 Passo 1: Estruturar o Ambiente

1. Abra o terminal do Fedora e crie uma pasta dedicada para o projeto do GLPI para manter a persistência dos dados organizada:
   ```bash
   mkdir -p ~/glpi && cd ~/glpi
   ```

2. Crie e abra o arquivo de configuração declarativa:
   ```bash
   nano compose.yaml
   ```

3. Cole o conteúdo abaixo dentro do arquivo:
   ```yaml
   version: '3.8'

   services:
     glpi-db:
       image: docker.io/library/mariadb:10.11
       container_name: glpi-db
       environment:
         - MARIADB_ROOT_PASSWORD=senha_mestra_root
         - MARIADB_DATABASE=glpi_db
         - MARIADB_USER=glpi_user
         - MARIADB_PASSWORD=glpi_senha_secreta
       volumes:
         - glpi_db_data:/var/lib/mysql:Z

     glpi-app:
       image: docker.io/diouxx/glpi:latest
       container_name: glpi-app
       ports:
         - "8080:80"
       depends_on:
         - glpi-db
       volumes:
         - glpi_files:/var/www/html/glpi/files:Z
         - glpi_plugins:/var/www/html/glpi/plugins:Z

   volumes:
     glpi_db_data:
     glpi_files:
     glpi_plugins:
   ```
   > ⚠️ **Nota de Segurança sobre o Fedora:** O sufixo `:Z` adicionado ao final dos volumes é **obrigatório**. Ele instrui o Podman a re rotular os volumes no sistema de arquivos para que o **SELinux** do Fedora permita que os contêineres gravem dados nas pastas locais sem bloqueios de segurança.

4. Salve e feche o arquivo (no Nano, pressione `Ctrl + O`, `Enter` para salvar e `Ctrl + X` para sair).

---

## 🛠️ Passo 2: Inicializar os Contêineres

1. Na mesma pasta onde o arquivo `compose.yaml` foi criado, execute o comando abaixo para baixar as imagens e iniciar o ecossistema em segundo plano:
   ```bash
   podman-compose up -d
   ```

2. Os contêineres serão criados na mesma rede virtual compartilhada. Você poderá acompanhar o status e o consumo de memória diretamente pela interface gráfica do **Podman Desktop**, navegando até a aba **Containers**.

---

## 🌐 Passo 3: Configuração Assistida no Navegador

1. Abra o navegador web do seu Fedora e acesse o endereço:
   ```text
   http://localhost:8080
   ```

2. Selecione o idioma de sua preferência (**Português do Brasil**), aceite os termos de licença e clique em **Instalar** (Nova Instalação).

3. Na tela **Configuração do GLPI - Etapa 1 (Instalação da conexão com o banco de dados)**, preencha rigorosamente com os seguintes parâmetros:
   * **Endereço do servidor SQL (MariaDB ou MySQL):** `glpi-db` *(Usamos o nome do serviço configurado no compose, pois no Podman os contêineres se resolvem pelo nome)*
   * **Usuário SQL:** `glpi_user`
   * **Senha SQL:** `glpi_senha_secreta`
   * Clique em **Continuar >**.

4. Na **Etapa 2 (Teste de conexão com o banco de dados)**:
   * Marque a opção sob a seção *OU USAR UM EXISTENTE*: **`glpi_db`**
   * Deixe o campo de criação de novos bancos em branco.
   * Clique em **Continuar >**.

5. O instalador vai estruturar as tabelas internas. Avance pelas telas informativas seguintes até que o sistema confirme a conclusão e exiba o redirecionamento para o painel de acesso.

---

## 🔐 Passo 4: Primeiro Acesso e Pós-Instalação

1. Na tela de login do GLPI, utilize as credenciais padrão de fábrica:
   * **Usuário:** `glpi`
   * **Senha:** `glpi`
   * Clique em **Entrar**.

2. **Ações Obrigatórias de Segurança (Imediato):**
   * No canto superior direito, clique no menu do usuário administrador (`glpi`), vá em configurações e **altere a senha padrão** para uma credencial forte.
   * Navegue no menu esquerdo em **Administração > Usuários** e altere as senhas ou desative as outras contas padrões de teste criadas pelo sistema (`tech`, `normal`, e `post-only`) para mitigar vulnerabilidades no ambiente.
