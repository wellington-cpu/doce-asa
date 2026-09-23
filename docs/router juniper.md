# Guia de Configuração Juniper J2320

Este documento apresenta o passo a passo para a configuração inicial, serviços de rede (DHCP), zonas de segurança (Firewall) e tradução de endereços (NAT) no roteador **Juniper J2320** utilizando o **Junos OS**.

---

## 1. Topologia de Referência e Premissas

Para este guia, adotamos o seguinte cenário prático:
* **Interface WAN (ge-0/0/0.0):** Conectada ao provedor de Internet. IP Fixo: `172.16.1.2/24` (Gateway: `172.16.1.1`).
* **Interface LAN (ge-0/0/1.0):** Rede local interna. IP do Roteador: `192.168.1.1/24`.
* **Servidor DHCP:** Ativo na LAN distribuindo a faixa de IPs `192.168.1.10` a `192.168.1.250`.
* **Modo de Operação:** *Flow Mode* (necessário para os recursos de segurança baseados em zonas).

---

## 2. Acesso Inicial via Console

1. Conecte o cabo de console na porta **CON** do equipamento e na porta USB/Serial do seu computador.
2. Abra um emulador de terminal (ex: PuTTY) utilizando os parâmetros:
   * **Baud rate:** 9600
   * **Data bits:** 8
   * **Stop bits:** 1
   * **Parity:** None
3. No prompt de login, digite `root` (sem senha por padrão de fábrica).
4. Acesse a CLI do Junos e entre no modo de configuração estruturada:
   ```junos
   root@% cli
   root@> configure
   root@# 
   ```

---

## 3. Configurações Básicas do Sistema

O Junos OS exige a definição de uma senha para o usuário administrador (`root`) antes de permitir a aplicação (*commit*) de qualquer configuração.

```junos
# 3.1 Definir a senha do root (obrigatório)
set system root-authentication plain-text-password

# 3.2 Definir o nome do equipamento e domínio
set system host-name Roteador-J2320
set system domain-name suaempresa.com.br

# 3.3 Configurar servidores de DNS e Sincronização de Hora (NTP)
set system name-server 8.8.8.8
set system name-server 1.1.1.1
set system ntp server 200.160.7.186

# 3.4 Habilitar o acesso remoto seguro (SSH)
set system services ssh
```

---

## 4. Configuração das Interfaces de Rede

Definição dos parâmetros de Camada 3 (IPs) e descrições das interfaces físicas.

```junos
# Interface WAN - Link de Internet
set interfaces ge-0/0/0 unit 0 description "Link Internet - WAN"
set interfaces ge-0/0/0 unit 0 family inet address 172.16.1.2/24

# Interface LAN - Rede Local
set interfaces ge-0/0/1 unit 0 description "Rede Local - LAN"
set interfaces ge-0/0/1 unit 0 family inet address 192.168.1.1/24

# Configuração da Rota Padrão (Gateway da Internet)
set routing-options static route 0.0.0.0/0 next-hop 172.16.1.1
```

---

## 5. Configuração do Servidor DHCP

Provisionamento automático de endereços de rede para as estações de trabalho da rede interna através do roteador.

```junos
# Criar o pool de IPs e definir os parâmetros de rede entregues aos clientes
set system services dhcp pool 192.168.1.0/24 address-range low 192.168.1.10 high 192.168.1.250
set system services dhcp pool 192.168.1.0/24 router 192.168.1.1
set system services dhcp pool 192.168.1.0/24 name-server 8.8.8.8
set system services dhcp pool 192.168.1.0/24 name-server 1.1.1.1

# Associar o processo DHCP para escutar e responder na interface interna
set system services dhcp router-services interface ge-0/0/1.0
```

---

## 6. Firewall Baseado em Zonas (Security Zones & Policies)

Por padrão no *Flow Mode*, o Junos adota uma postura de segurança restritiva (*Default Drop*). É necessário segregar as interfaces em zonas e criar políticas explícitas de tráfego.

```junos
# 6.1 Criar a Zona de Confiança (Trust) e permitir tráfego direcionado ao roteador (Inbound)
set security zones security-zone trust interfaces ge-0/0/1.0 host-inbound-traffic system-services dhcp
set security zones security-zone trust interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone trust interfaces ge-0/0/1.0 host-inbound-traffic system-services ssh

# 6.2 Criar a Zona Não Confiável (Untrust - Internet) e limitar o tráfego de entrada apenas a testes de Ping
set security zones security-zone untrust interfaces ge-0/0/0.0 host-inbound-traffic system-services ping

# 6.3 Criar a política que autoriza a rede interna a iniciar conexões com a Internet
set security policies from-zone trust to-zone untrust policy permitir-lan-internet match source-address any
set security policies from-zone trust to-zone untrust policy permitir-lan-internet match destination-address any
set security policies from-zone trust to-zone untrust policy permitir-lan-internet match application any
set security policies from-zone trust to-zone untrust policy permitir-lan-internet then permit
```

---

## 7. Configuração de NAT de Origem (Source NAT)

Necessário para traduzir os endereços IP privados internos (`192.168.1.0/24`) para o endereço IP público associado à interface física externa conectado ao link.

```junos
# Criar regras de mascaramento (Masquerade) utilizando o IP da própria interface WAN
set security nat source rule-set nat-lan-internet from zone trust
set security nat source rule-set nat-lan-internet to zone untrust
set security nat source rule-set nat-lan-internet rule mascarar-lan match source-address 192.168.1.0/24
set security nat source rule-set nat-lan-internet rule mascarar-lan match destination-address any
set security nat source rule-set nat-lan-internet rule mascarar-lan then source-nat interface
```

---

## 8. Verificação e Salvamento da Configuração

No Junos OS, as configurações alteradas ficam em modo candidato (*candidate configuration*) e precisam ser checadas e explicitamente aplicadas.

```junos
# Verificar se existem erros de sintaxe ou lógica na configuração atual
commit check

# Aplicar as configurações de maneira segura (Dica: desfaz se perder o acesso em 10 minutos)
commit confirmed 10

# Se tudo funcionar perfeitamente e o acesso persistir, confirme permanentemente:
commit
```

---
### 📋 Ficha Técnica do Documento

| Atribuição | IFRN / Diego Pereira  |
| :--- | :--- |
| **IFRN** | Curso Técnico em Redes de Computadores [Campus Parnamirim] |
| **Orientador** | Prof. [Diego Pereira ] |
| **Contexto** | Laboratório de Redes e Conectividade |

> ### 📝 Referências e Agradecimentos
> Este guia de configuração foi desenvolvido como material de apoio técnico sob a orientação do **Professor [Diego Pereira]** no **Centro de Estudo e Tecnologia [IFRN PARNAMIRIM]**. Os dados e comandos foram baseados na [Documentação Oficial de Configuração do Junos OS](https://juniper.net).


*Fim do documento técnico de referência.*
