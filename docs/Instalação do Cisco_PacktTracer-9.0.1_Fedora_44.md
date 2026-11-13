# Instalação do Cisco Packet Tracer 9.0.1 no Fedora 44 (KDE Plasma)

## Sumário

1. [Visão geral](#1-visão-geral)

2. [Ambiente de execução](#2-ambiente-de-execução)

3. [Problemas encontrados no script original](#3-problemas-encontrados-no-script-original)

4. [Soluções adotadas](#4-soluções-adotadas)

5. [Pré-requisitos](#5-pré-requisitos)

6. [Script de instalação](#6-script-de-instalação)

7. [Passo a passo da instalação](#7-passo-a-passo-da-instalação)

8. [Verificação pós-instalação](#8-verificação-pós-instalação)

9. [Desinstalação](#9-desinstalação)

10. [Resolução de problemas](#10-resolução-de-problemas)

11. [Referências](#11-referências)


## 1. Visão geral

Este documento registra o processo completo de instalação do **Cisco Packet Tracer 9.1** (`CiscoPacketTracer\_901\_Ubuntu\_64bit.deb`) em um sistema **Fedora Linux 44 com KDE Plasma**, a partir do script original [`setup.sh`](https://github.com/thiagoojack/packettracer-fedora), que foi adaptado para funcionar corretamente nesta versão do Fedora.

O Packet Tracer é distribuído oficialmente pela Cisco apenas nos formatos **`.deb`** (Debian/Ubuntu) e **`.rpm`** (Red Hat/Fedora) — porém, a versão `.rpm` costuma ser desatualizada ou indisponível. O caminho mais confiável é usar o `.deb` oficial e convertê-lo manualmente, o que o script faz por meio de `ar` + `tar`, copiando os arquivos para `/opt/pt` e integrando o aplicativo ao sistema.

Este documento descreve:

- Por que o script original **não funciona** no Fedora 44 sem ajustes;

- As **modificações necessárias** (FUSE 2, `qt5-qttools` opcional, *fallback* com `--appimage-extract`, `dnf5`);

- O **script final** utilizado com sucesso;

- O **procedimento de verificação** e **desinstalação**.


## 2. Ambiente de execução

| Item | Valor |
| - | - |
| **Data da instalação** | 12/11/2026 |
| **Distribuição** | Fedora Linux 44 (KDE Plasma Desktop Edition) |
| **`NAME`** | `Fedora Linux` |
| **`VERSION`** | `44 (KDE Plasma Desktop Edition)` |
| **`RELEASE\_TYPE`** | `stable` |
| **`ID`** | `fedora` |
| **`VERSION\_ID`** | `44` |
| **`VERSION\_CODENAME`** | *(vazio)* |
| **`PRETTY\_NAME`** | `Fedora Linux 44 (KDE Plasma Desktop Edition)` |
| **Instalador usado** | `CiscoPacketTracer\_901\_Ubuntu\_64bit.deb` |
| **Versão do Packet Tracer** | 9.1 |


Para reproduzir a verificação do sistema:

```
cat /etc/os-release
```

Saída esperada no ambiente em questão:

```
NAME="Fedora Linux"  
VERSION="44 (KDE Plasma Desktop Edition)"  
RELEASE\_TYPE=stable  
ID=fedora  
VERSION\_ID=44  
VERSION\_CODENAME=""  
PRETTY\_NAME="Fedora Linux 44 (KDE Plasma Desktop Edition)"
```


## 3. Problemas encontrados no script original

O script original `setup.sh` (do repositório `thiagoojack/packettracer-fedora`) foi escrito para **Fedora 40, 41 e 42**. Ao tentar usá-lo no Fedora 44, os seguintes problemas aparecem:

### 3.1. Dependência do FUSE 2

O Packet Tracer é empacotado como **AppImage**, que depende da biblioteca **`libfuse.so.2`** (FUSE 2). A partir do Fedora 43, o FUSE 2 deixou de vir por padrão — apenas o **FUSE 3** é fornecido. Sem o FUSE 2, o AppImage falha ao ser executado com:

```
dlopen(): error loading libfuse.so.2
```

O `setup.sh` original já tenta instalar `fuse` e `fuse-libs`, mas em algumas configurações do Fedora 44 esses pacotes não estão disponíveis ou não são suficientes.

### 3.2. `qt5-qttools` pode não existir nos repositórios

O script original instala `qt5-qttools` como dependência obrigatória. No Fedora 44 esse pacote pode não estar disponível, fazendo o script abortar por causa do `set -e`.

### 3.3. `ar --output` não é portável

A opção `--output` do `ar` não é suportada em todas as versões do GNU binutils. Em alguns sistemas, é necessário executar `ar` dentro do diretório de destino.

### 3.4. Uso de `dnf` em vez de `dnf5`

O Fedora 44 usa **`dnf5`** como gerenciador de pacotes padrão. O script original chama `dnf`, que ainda funciona como *wrapper*, mas não é o ideal.

### 3.5. Criação de `~/.local/share/applications` com `sudo`

O script original cria esse diretório com `sudo`, o que pode deixá-lo com dono `root` e causar problemas futuros para o usuário.


## 4. Soluções adotadas

| Problema | Solução aplicada |
| - | - |
| FUSE 2 ausente | Instalar `fuse` + `fuse-libs` **e** `fuse3` + `fuse3-libs` |
| `qt5-qttools` opcional | Instalação com `\\|\\| true` + aviso, sem abortar |
| AppImage não monta via FUSE | *Fallback* com `./packettracer.AppImage --appimage-extract` e criação de *wrapper* em `/usr/local/bin/packettracer` |
| `ar --output` | Mantido, mas com extração no diretório `packettracer/` |
| `dnf` | Preferência por `dnf5`, com `dnf` como alternativa |
| Dono `root` em `~/.local` | Diretório criado **sem `sudo`** (o script roda como root mas usa `mkdir -p` no caminho do usuário) |
| Detecção de usuário | Uso de `$SUDO\_USER` e `getent passwd` para obter o home real |



## 5. Pré-requisitos

- **Fedora 44** (ou superior) — testado com KDE Plasma Desktop Edition.

- **Acesso a `sudo`** (root).

- **Conexão com a internet** para baixar dependências via `dnf5`.

- **Instalador oficial** `CiscoPacketTracer\_901\_Ubuntu\_64bit.deb` baixado do site da Cisco:

  - [https://www.netacad.com/portal/resources/packet-tracer](https://www.netacad.com/portal/resources/packet-tracer)

  - [https://skillsforall.com/resources/lab-downloads](https://skillsforall.com/resources/lab-downloads) (login necessário)

- **Conta Cisco NetAcad** (para ativar o Packet Tracer no primeiro uso).

Recomenda-se salvar o `.deb` em `~/Downloads` para agilizar a busca.


## 6. Script de instalação

Salve o conteúdo abaixo como **`install-pt-901-fedora44.sh`**:

```
\#!/bin/bash  
\#  
\# install-pt-901-fedora44.sh  
\# Instala o Cisco Packet Tracer 9.1 no Fedora 44 a partir do  
\# arquivo CiscoPacketTracer\_901\_Ubuntu\_64bit.deb.  
\#  
\# Uso:  
\#   sudo bash install-pt-901-fedora44.sh -d /caminho/para/o/deb  
\#  
\# Opções:  
\#   -d, --directory DIR   Pasta onde está o .deb (evita busca em todo o /home)  
\#   -h, --help            Mostra esta ajuda  
\#   --uninstall           Remove o Packet Tracer instalado  
\#  
  
set -euo pipefail  
  
\# ---------- Cores ----------  
Red="\\033\[0;31m"  
Green="\\033\[0;32m"  
Yellow="\\033\[0;33m"  
Cyan="\\033\[0;36m"  
Bold="\\033\[1m"  
Color\_Off="\\033\[0m"  
  
log()  \{ echo -e "$\{Green\}\[+\]$\{Color\_Off\} $\*"; \}  
warn() \{ echo -e "$\{Yellow\}\[!\]$\{Color\_Off\} $\*"; \}  
fail() \{ echo -e "$\{Red\}\[x\]$\{Color\_Off\} $\*"; \}  
  
\# ---------- Exige root ----------  
if \[\[ "$\{EUID\}" -ne 0 \]\]; then  
    fail "Execute com sudo: sudo bash $0 -d /caminho/do/deb"  
    exit 1  
fi  
  
\# ---------- Detecta usuário real ----------  
if \[\[ -n "$\{SUDO\_USER:-\}" && "$\{SUDO\_USER\}" != "root" \]\]; then  
    user\_name="$SUDO\_USER"  
    user\_home="$(getent passwd "$SUDO\_USER" | cut -d: -f6)"  
else  
    user\_name="$(who | awk 'NR==1\{print $1\}')"  
    user\_home="$(getent passwd "$user\_name" 2\>/dev/null | cut -d: -f6)"  
fi  
  
if \[\[ -z "$\{user\_name:-\}" || -z "$\{user\_home:-\}" \]\]; then  
    fail "Não foi possível determinar o usuário alvo."  
    exit 1  
fi  
  
\# ---------- Parse de opções ----------  
installer\_dir=""  
uninstall\_flag=0  
  
while \[\[ $\# -gt 0 \]\]; do  
    case "$1" in  
        -d|--directory)  
            installer\_dir="$\{2:?'--directory exige um caminho'\}"  
            shift 2  
            ;;  
        -h|--help)  
            cat \<\<EOF  
Uso: sudo bash $0 \[OPÇÕES\]  
  
Instala o Cisco Packet Tracer 9.1 no Fedora 44.  
  
Opções:  
  -d, --directory DIR   Pasta onde está o .deb (padrão: busca em ~/)  
  -h, --help            Mostra esta ajuda  
  --uninstall           Remove o Packet Tracer instalado  
  
Exemplo:  
  sudo bash $0 -d /home/$user\_name/Downloads  
EOF  
            exit 0  
            ;;  
        --uninstall)  
            uninstall\_flag=1  
            shift  
            ;;  
        \*)  
            fail "Opção desconhecida: $1"  
            exit 1  
            ;;  
    esac  
done  
  
\# ---------- Função de desinstalação ----------  
uninstall() \{  
    if \[\[ -e /opt/pt \]\]; then  
        log "Desinstalando Cisco Packet Tracer…"  
  
        if \[\[ -x /usr/local/bin/packettracer \]\]; then  
            runuser -u "$user\_name" -- /usr/local/bin/packettracer --pt-deactivate 2\>/dev/null || true  
        fi  
  
        rm -rf /opt/pt  
        rm -f /usr/local/share/applications/CiscoPacketTracer\*.desktop 2\>/dev/null || true  
        rm -f "/home/$\{user\_name\}/.local/share/applications/CiscoPacketTracer\*.desktop" 2\>/dev/null || true  
        rm -f "/home/$\{user\_name\}/.local/share/mime/packages/cisco-pk\*.xml" 2\>/dev/null || true  
        rm -f /usr/local/bin/packettracer  
  
        xdg-desktop-menu uninstall /usr/share/applications/cisco-pt\*.desktop 2\>/dev/null || true  
        update-mime-database /usr/share/mime 2\>/dev/null || true  
        gtk-update-icon-cache -t --force /usr/share/icons 2\>/dev/null || true  
  
        log "Cisco Packet Tracer removido."  
    else  
        warn "Packet Tracer não está instalado em /opt/pt."  
    fi  
    sleep 2  
\}  
  
\# ---------- Se pediu apenas desinstalar ----------  
if \[\[ "$uninstall\_flag" -eq 1 \]\]; then  
    uninstall  
    exit 0  
fi  
  
\# ---------- Localiza o .deb ----------  
search\_path="$\{installer\_dir:-$user\_home\}"  
selected\_installer=""  
  
log "Procurando por CiscoPacketTracer\_901\_Ubuntu\_64bit.deb em $\{search\_path\}…"  
  
while IFS= read -r -d '' f; do  
    selected\_installer="$f"  
    break  
done \< \<(find "$search\_path" -type f \\  
    \\( -name "CiscoPacketTracer\_901\_Ubuntu\_64bit.deb" \\  
       -o -name "Cisco\*Packet\*901\*.deb" \\  
       -o -name "Packet\*Tracer\*901\*.deb" \\) \\  
    -print0 2\>/dev/null)  
  
if \[\[ -z "$selected\_installer" \]\]; then  
    fail "Instalador não encontrado em $\{search\_path\}."  
    echo "Baixe o arquivo CiscoPacketTracer\_901\_Ubuntu\_64bit.deb em:"  
    echo "  https://www.netacad.com/portal/resources/packet-tracer"  
    echo "  https://skillsforall.com/resources/lab-downloads (login necessário)"  
    exit 1  
fi  
  
log "Instalador selecionado: $\{Cyan\}$\{selected\_installer\}$\{Color\_Off\}"  
sleep 1  
  
\# ---------- Instalação ----------  
install() \{  
    log "Removendo versão anterior (se existir)…"  
    uninstall  
  
    log "Instalando dependências…"  
    if command -v dnf5 \>/dev/null 2\>&1; then  
        dnf5 install -y binutils fuse fuse-libs fuse3 fuse3-libs || \\  
        dnf5 install -y binutils fuse fuse-libs fuse3 fuse3-libs  
    else  
        dnf install -y binutils fuse fuse-libs fuse3 fuse3-libs  
    fi  
  
    dnf5 install -y qt5-qttools 2\>/dev/null || \\  
        dnf install -y qt5-qttools 2\>/dev/null || \\  
        warn "qt5-qttools não encontrado — ignorando (o PT já traz seu Qt)."  
  
    local apps\_dir="/home/$\{user\_name\}/.local/share/applications"  
    \[\[ -d "$apps\_dir" \]\] || mkdir -p "$apps\_dir"  
  
    log "Extraindo o .deb…"  
    rm -rf packettracer  
    mkdir packettracer  
    ar -x "$selected\_installer" --output=packettracer  
    tar -xf packettracer/control.tar.xz --directory=packettracer  
    tar -xf packettracer/data.tar.xz --directory=packettracer  
  
    log "Copiando arquivos para /opt/pt…"  
    cp -r packettracer/opt /  
  
    log "Criando link simbólico…"  
    \[\[ -d /usr/local/bin \]\] || mkdir -p /usr/local/bin  
    ln -sf /opt/pt/packettracer.AppImage /usr/local/bin/packettracer  
  
    log "Ativando Packet Tracer…"  
    if /usr/local/bin/packettracer --pt-activate 2\>/dev/null; then  
        log "AppImage ativado via FUSE."  
    else  
        warn "FUSE indisponível — usando fallback com --appimage-extract…"  
        pushd /opt/pt \>/dev/null  
        ./packettracer.AppImage --appimage-extract \>/dev/null 2\>&1  
  
        tee /usr/local/bin/packettracer \>/dev/null \<\<'WRAPPER'  
\#!/bin/bash  
exec /opt/pt/squashfs-root/AppRun "$@"  
WRAPPER  
        chmod +x /usr/local/bin/packettracer  
        /usr/local/bin/packettracer --pt-activate 2\>/dev/null || true  
        popd \>/dev/null  
    fi  
  
    log "Executando postinst (pós-instalação oficial)…"  
    bash packettracer/postinst 2\>/dev/null || \\  
        warn "postinst terminou com avisos (normal fora do Debian/Ubuntu)."  
  
    log "Atualizando menus, MIME e ícones…"  
    xdg-desktop-menu uninstall /usr/share/applications/cisco-pt\*.desktop 2\>/dev/null || true  
    update-mime-database /usr/share/mime 2\>/dev/null || true  
    gtk-update-icon-cache -t --force /usr/share/icons 2\>/dev/null || true  
  
    rm -rf packettracer  
  
    log "Cisco Packet Tracer 9.1 instalado com sucesso!"  
\}  
  
install
```

> **Nota**: para conferir o conteúdo do script antes de executá-lo, use `less install-pt-901-fedora44.sh` ou `bash -n install-pt-901-fedora44.sh` (verifica sintaxe sem executar).


## 7. Passo a passo da instalação

### 7.1. Baixar o instalador

Acesse um dos links oficiais (é necessário login):

- [https://www.netacad.com/portal/resources/packet-tracer](https://www.netacad.com/portal/resources/packet-tracer)

- [https://skillsforall.com/resources/lab-downloads](https://skillsforall.com/resources/lab-downloads)

Baixe o arquivo **`CiscoPacketTracer\_901\_Ubuntu\_64bit.deb`** e mova-o para `~/Downloads` (ou qualquer pasta em `/home`).

```
ls -lh ~/Downloads/CiscoPacketTracer\_901\_Ubuntu\_64bit.deb
```

### 7.2. Salvar o script

Salve o script da [seção 6](#6-script-de-instalação) em um arquivo, por exemplo:

```
nano ~/install-pt-901-fedora44.sh
```

### 7.3. Dar permissão de execução

```
chmod +x ~/install-pt-901-fedora44.sh
```

### 7.4. Executar a instalação

```
sudo bash ~/install-pt-901-fedora44.sh -d /home/$USER/Downloads
```

Substitua `/home/$USER/Downloads` pelo caminho real onde está o `.deb`. A opção `-d` evita que o script varra todo o `/home`.

### 7.5. Acompanhar a saída

Durante a execução, o script exibe mensagens com prefixos coloridos:

| Prefixo | Significado |
| - | - |
| `\[+\]` | Etapa concluída com sucesso |
| `\[!\]` | Aviso (não impede a instalação) |
| `\[x\]` | Erro (interrompe a instalação) |


Um resultado esperado ao final é:

```
\[+\] Cisco Packet Tracer 9.1 instalado com sucesso!
```

### 7.6. Ativar o Packet Tracer

Na primeira execução, o Packet Tracer pedirá login na conta Cisco NetAcad para ativação. Isso pode ser feito pela interface gráfica ou pelo comando:

```
packettracer --pt-activate
```


## 8. Verificação pós-instalação

Após a instalação, execute os comandos abaixo para confirmar que tudo está no lugar:

### 8.1. Arquivos e diretórios

```
ls -l /opt/pt/packettracer.AppImage  
ls -l /usr/local/bin/packettracer
```

Saída esperada:

```
-rwxr-xr-x. 1 root root ... /opt/pt/packettracer.AppImage  
lrwxrwxrwx. 1 root root ... /usr/local/bin/packettracer -\> /opt/pt/packettracer.AppImage
```

### 8.2. Comando no PATH

```
which packettracer
```

Saída esperada:

```
/usr/local/bin/packettracer
```

### 8.3. Versão

```
packettracer --version 2\>/dev/null || echo "(sem --version; abra a GUI)"
```

### 8.4. Menu de aplicativos (KDE Plasma)

No KDE Plasma, o aplicativo deve aparecer no menu como **"Cisco Packet Tracer"**. Caso não apareça, atualize o cache:

```
kbuildsycoca6
```

### 8.5. Execução

```
packettracer
```

A janela do Packet Tracer 9.1 deve abrir.


## 9. Desinstalação

### 9.1. Pelo próprio script (recomendado)

```
sudo bash ~/install-pt-901-fedora44.sh --uninstall
```

O script remove:

- `/opt/pt` (aplicação completa);

- `/usr/local/bin/packettracer` (link simbólico);

- arquivos `.desktop` do sistema e do usuário;

- XMLs de MIME do Packet Tracer;

- atualiza `update-mime-database`, `gtk-update-icon-cache` e `xdg-desktop-menu`.

Antes de remover, ele executa `--pt-deactivate` **como o usuário real** (via `runuser`) para limpar o registro de ativação.

### 9.2. Manualmente

Caso o script não esteja disponível:

```
\# Desativar  
runuser -u "$USER" -- /usr/local/bin/packettracer --pt-deactivate 2\>/dev/null || true  
  
\# Remover arquivos  
sudo rm -rf /opt/pt  
sudo rm -f /usr/local/bin/packettracer  
sudo rm -f /usr/local/share/applications/CiscoPacketTracer\*.desktop  
sudo rm -f ~/.local/share/applications/CiscoPacketTracer\*.desktop  
sudo rm -f ~/.local/share/mime/packages/cisco-pk\*.xml  
  
\# Atualizar caches  
sudo xdg-desktop-menu uninstall /usr/share/applications/cisco-pt\*.desktop 2\>/dev/null || true  
sudo update-mime-database /usr/share/mime  
sudo gtk-update-icon-cache -t --force /usr/share/icons
```

### 9.3. Dependências

As dependências instaladas (`binutils`, `fuse`, `fuse-libs`, `fuse3`, `fuse3-libs`, `qt5-qttools`) **não são removidas** automaticamente, pois outras aplicações podem depender delas. Se quiser removê-las:

```
sudo dnf5 remove fuse fuse-libs fuse3 fuse3-libs qt5-qttools
```

> **Atenção**: `binutils` é usado por muitos programas (compiladores, empacotadores). **Não remova `binutils`**.


## 10. Resolução de problemas

### 10.1. `dlopen(): error loading libfuse.so.2`

O FUSE 2 não está instalado. Instale:

```
sudo dnf5 install -y fuse fuse-libs
```

Se o erro persistir, use o *fallback* do script (`--appimage-extract`), que já está implementado.

### 10.2. `postinst` termina com erros

Isso é **esperado** fora do Debian/Ubuntu. O `postinst` oficial da Cisco foi escrito para ambientes Debian e pode falhar em etapas específicas (ex.: `update-desktop-database`). O script trata isso como aviso e continua.

### 10.3. Aplicativo não aparece no menu KDE

Atualize o cache do KDE:

```
kbuildsycoca6
```

E verifique se o `.desktop` existe:

```
ls /usr/share/applications/cisco-pt\*.desktop  
ls ~/.local/share/applications/CiscoPacketTracer\*.desktop
```

### 10.4. Erro de ativação

Verifique a conexão com a internet e se a conta Cisco NetAcad está ativa. Tente reativar:

```
packettracer --pt-activate
```

### 10.5. `dnf5` não encontrado

O script já trata esse caso com *fallback* para `dnf`. Se nem `dnf` estiver disponível, instale-o:

```
sudo dnf install -y dnf5
```

### 10.6. Erro ao extrair o `.deb` (`ar: unrecognized option '--output'`)

Atualize o `binutils`:

```
sudo dnf5 update -y binutils
```

Se o problema persistir, execute o `ar` dentro da pasta de destino:

```
cd packettracer && ar -x ../CiscoPacketTracer\_901\_Ubuntu\_64bit.deb && cd ..
```


## 11. Referências

- **Repositório original** (arquivado em julho/2026): [https://github.com/thiagoojack/packettracer-fedora](https://github.com/thiagoojack/packettracer-fedora)

- **Cisco NetAcad — Downloads**: [https://www.netacad.com/portal/resources/packet-tracer](https://www.netacad.com/portal/resources/packet-tracer)

- **Cisco Skills for All — Lab Downloads**: [https://skillsforall.com/resources/lab-downloads](https://skillsforall.com/resources/lab-downloads)

- **Fedora 44 — Release Notes**: [https://docs.fedoraproject.org/en-US/fedora/latest/release-notes/](https://docs.fedoraproject.org/en-US/fedora/latest/release-notes/)

- **FUSE 3 no Fedora**: [https://fedoraproject.org/wiki/Changes/Fuse3](https://fedoraproject.org/wiki/Changes/Fuse3)

- **AppImage — `--appimage-extract`**: [https://docs.appimage.org/user-guide/troubleshooting/fuse.html](https://docs.appimage.org/user-guide/troubleshooting/fuse.html)


## Apêndice A — Resumo dos comandos

```
\# 1. Baixar o .deb em ~/Downloads (via site da Cisco)  
  
\# 2. Salvar o script  
nano ~/install-pt-901-fedora44.sh  
chmod +x ~/install-pt-901-fedora44.sh  
  
\# 3. Instalar  
sudo bash ~/install-pt-901-fedora44.sh -d /home/$USER/Downloads  
  
\# 4. Verificar  
which packettracer  
ls -l /opt/pt/packettracer.AppImage  
kbuildsycoca6  
  
\# 5. Executar  
packettracer  
  
\# 6. Desinstalar (quando necessário)  
sudo bash ~/install-pt-901-fedora44.sh --uninstall
```


## Apêndice B — Estrutura de arquivos após a instalação

```
/opt/pt/  
├── packettracer.AppImage        \# AppImage principal  
├── postinst                     \# script oficial de pós-instalação  
├── ...                          \# demais arquivos do .deb  
└── (squashfs-root/              \# apenas se o fallback foi usado)  
  
/usr/local/bin/  
└── packettracer -\> /opt/pt/packettracer.AppImage  
  
/usr/share/applications/  
└── cisco-pt\*.desktop            \# atalho do sistema (criado pelo postinst)  
  
~/.local/share/applications/  
└── CiscoPacketTracer\*.desktop   \# atalho do usuário  
  
~/.local/share/mime/packages/  
└── cisco-pk\*.xml                \# registro de MIME
```


## Apêndice C — Registro de execução bem-sucedida

| Campo | Valor |
| - | - |
| **Data** | 12/11/2026 |
| **Sistema** | Fedora Linux 44 (KDE Plasma Desktop Edition) |
| **Kernel** | *(conforme `uname -r` no momento da instalação)* |
| **Instalador** | `CiscoPacketTracer\_901\_Ubuntu\_64bit.deb` |
| **Versão instalada** | Cisco Packet Tracer 9.1 |
| **Script utilizado** | `install-pt-901-fedora44.sh` |
| **Resultado** | Instalação concluída com sucesso |
| **Método de ativação** | AppImage via FUSE (com fallback disponível) |
| **Local de instalação** | `/opt/pt/` |
| **Atalho** | `/usr/local/bin/packettracer` |



**Fim da documentação.**

