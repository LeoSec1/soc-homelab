# Guia de Instalação

Passos para montar o ambiente do laboratório do zero.

## Pré-requisitos

### Hardware

* RAM: Mínimo 16 GB (recomendado 32 GB)
* Armazenamento: Mínimo 200 GB SSD
* CPU: 4 núcleos (recomendado 8)

### Software

* VMware Workstation Pro
* ISOs: Ubuntu 22.04 LTS Server, Windows 10/11, Kali Linux 2024

## Rede

Criar uma rede virtual no VMware:

* **VMnet2** — Host-Only, sem DHCP, segmento `10.0.1.0/24`
* **VMnet8** — NAT (para downloads e atualizações)

Cada VM recebe duas interfaces: NAT para internet e VMnet2 para o laboratório.

## Ordem de Instalação

1. Configurar VMnet2 no VMware.
2. VM-SIEM (Ubuntu Server) — Wazuh All-in-One.
3. VM-Sensor (Ubuntu Server) — Suricata + Wazuh Agent.
4. VM-Vítima Linux (Ubuntu Server) — Apache, SSH, FTP + Wazuh Agent.
5. VM-Vítima Windows — RDP, SMB + Wazuh Agent.
6. VM-Atacante (Kali Linux).

## Expansão do LVM (Ubuntu Server)

O instalador do Ubuntu aloca apenas ~50% do disco. Expandir após a instalação:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

## Wazuh All-in-One (VM-SIEM)

Consulte a [documentação oficial do Wazuh](https://documentation.wazuh.com/) para o script de instalação All-in-One.

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.9/config.yml
# Editar config.yml com o IP 10.0.1.50
sudo bash wazuh-install.sh -a
```

Guarde as senhas geradas, especialmente a do usuário `admin`.

## Agentes Wazuh

A versão do agente não pode ser superior à do Manager. Fixar em `4.9.2-1`:

```bash
WAZUH_MANAGER="10.0.1.50" apt-get install wazuh-agent=4.9.2-1
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

## Suricata (VM-Sensor)

```bash
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt update
sudo apt install suricata jq -y
```

Configurar `HOME_NET` como `[10.0.1.0/24]` no `suricata.yaml` e a interface `ens34` em modo promíscuo com AF_PACKET.

Adicionar leitura do `eve.json` no `ossec.conf` do agente Wazuh:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```
