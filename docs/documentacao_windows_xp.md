# Documentação de Implantação: Windows XP no Podman (Modo Bridge)

Esta documentação descreve como implantar, gerenciar e automatizar um container Windows XP utilizando o Podman no Fedora Linux. O ambiente é configurado com IP estático em uma rede em modo bridge e aceleração de hardware KVM.

---

## 1. Pré-requisitos do Hospedeiro (Fedora Linux)

Antes de iniciar, certifique-se de que o diretório de dados existe e possui as permissões corretas (o sufixo `:Z` no Podman cuidará das permissões do SELinux):

```bash
mkdir -p /home/wellington/winxp_dados
```

> **Nota:** Se for utilizar uma ISO customizada, coloque o arquivo `custom.iso` exatamente dentro do diretório `/home/wellington/winxp_dados/` antes de rodar o comando de criação.

---

## 2. Comando de Implantação (Deploy)

Caso precise recriar o container do zero, utilize o comando abaixo. Ele já inclui a rede em modo bridge, o IP estático desejado (`192.168.0.150`), os servidores DNS e o mapeamento da ISO customizada.

```bash
sudo podman run -d \
  --name windows_xp \
  --network rede_bridge_local \
  --ip 192.168.0.150 \
  --dns 8.8.8.8 \
  --dns 1.1.1.1 \
  --device /dev/kvm \
  --device /dev/net/tun \
  --cap-add NET_ADMIN \
  -v /home/wellington/winxp_dados:/storage:Z \
  -v /home/wellington/winxp_dados/custom.iso:/custom.iso:Z \
  -e VERSION="xp" \
  dockurr/windows
```

---

## 3. Gerenciamento Manual (Operação e Manutenção)

Utilize estes comandos para gerenciar o container manualmente quando precisar realizar manutenções no Fedora Linux ou no próprio container.

*   **Iniciar o Windows XP manualmente:**
    ```bash
    sudo podman start windows_xp
    ```
*   **Parar o Windows XP de forma limpa (Antes de atualizar/reiniciar o Fedora):**
    ```bash
    sudo podman stop windows_xp
    ```
*   **Verificar o status atual do container:**
    ```bash
    sudo podman ps -a -f name=windows_xp
    ```
*   **Acompanhar a inicialização e logs do sistema:**
    ```bash
    sudo podman logs -f windows_xp
    ```

---

## 4. Configuração como Serviço (Inicialização Automática com o Sistema)

O Podman se integra nativamente ao **systemd** do Fedora. Siga os passos abaixo para fazer com que o container do Windows XP inicie automaticamente junto com o boot do servidor e desligue corretamente caso o Fedora seja reiniciado.

### Passo 4.1: Gerar o arquivo de serviço do systemd
O Podman consegue criar o arquivo de configuração de forma automática baseado no container existente:

```bash
sudo podman generate systemd --new --name windows_xp --files --output /etc/systemd/system/
```

### Passo 4.2: Recarregar o gerenciador do systemd
Sempre que um novo arquivo de serviço é adicionado, o sistema precisa ser avisado:

```bash
sudo systemctl daemon-reload
```

### Passo 4.3: Habilitar a inicialização automática (Boot)
Ative o serviço para que ele inicie junto com o Fedora:

```bash
sudo systemctl enable container-windows_xp.service
```

---

## 5. Comandos de Controle via Serviço (Alternativa ao item 3)

Após configurar o item 4, você pode passar a gerenciar o container utilizando os comandos padrão do sistema operacional Fedora:

*   **Iniciar o serviço:** 
    ```bash
    sudo systemctl start container-windows_xp.service
    ```
*   **Parar o serviço (Manutenção):** 
    ```bash
    sudo systemctl stop container-windows_xp.service
    ```
*   **Verificar se o serviço está rodando:** 
    ```bash
    sudo systemctl status container-windows_xp.service
    ```
*   **Desativar a inicialização automática temporariamente:** 
    ```bash
    sudo systemctl disable container-windows_xp.service
    ```