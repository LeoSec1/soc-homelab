# Arquitetura do Laboratorio

## Visao Geral

O laboratorio e composto por 5 maquinas virtuais conectadas em uma rede
isolada (VMnet2 - 10.0.1.0/24) dentro do VMware Workstation Pro.

A segmentacao em maquinas separadas foi uma decisao de projeto para simular
um cenario real onde o IDS fica isolado dos servidores de producao,
reduzindo a superficie de ataque.

## Topologia

```
ATACANTE (10.0.1.100) ---- ataque -----> VITIMA LINUX (10.0.1.200)
         |                                        |
         +---- ataque -----> VITIMA WINDOWS (10.0.1.201)
                                                  |
    SENSOR (10.0.1.10)                            |
    Suricata em modo promiscuo                    |
    Captura TODO o trafego da rede                |
         |                                        |
         | eve.json (alertas)        auth.log / Event IDs
         |                                        |
         +----------------+  +--------------------+
                           v  v
                     SIEM (10.0.1.50)
                     Wazuh All-in-One
                     Manager + Indexer + Dashboard
```

## Por que cada maquina e separada

**VM-Sensor separada das Vitimas:**
O Suricata precisa ficar em uma maquina dedicada por dois motivos. Primeiro,
ele usa modo promiscuo para capturar o trafego de toda a rede, nao apenas
o que chega nele. Segundo, se um atacante comprometer a Vitima, o Sensor
continua monitorando normalmente.

**VM-SIEM separada do Sensor:**
O Wazuh Manager, Indexer e Dashboard consomem bastante memoria e CPU.
Colocar junto com o Suricata causaria perda de pacotes em momentos de
alto trafego.

**Duas Vitimas (Linux + Windows):**
Demonstra que o SIEM e capaz de correlacionar eventos de ambientes
heterogeneos. No mundo real, a maioria das empresas opera com servidores
Linux e estacoes Windows simultaneamente.

## Rede

| Interface VMware | Tipo | Enderecamento | Funcao |
|---|---|---|---|
| VMnet8 | NAT | DHCP | Internet (downloads e atualizacoes) |
| VMnet2 | Host-Only | 10.0.1.0/24 | Rede isolada do laboratorio |

Cada VM possui duas placas de rede. A primeira (NAT) serve apenas para
baixar pacotes da internet. A segunda (VMnet2) e onde todo o trafego
de ataque e defesa acontece, sem afetar a rede real.

## Maquinas Virtuais

| Maquina | Sistema | RAM | vCPUs | Disco | Funcao |
|---|---|:---:|:---:|:---:|---|
| VM-SIEM | Ubuntu Server 22.04 | 8 GB | 4 | 50 GB | Wazuh Manager + Indexer + Dashboard |
| VM-Sensor | Ubuntu Server 22.04 | 4 GB | 2 | 30 GB | Suricata 7.x + Wazuh Agent |
| VM-Vitima | Ubuntu Server 22.04 | 2 GB | 2 | 20 GB | Apache, OpenSSH, vsftpd |
| VM-Vitima-Windows | Windows 10/11 | 4 GB | 2 | 40 GB | RDP, SMB, Event Viewer |
| VM-Atacante | Kali Linux 2024 | 4 GB | 2 | 25 GB | Nmap, Hydra, Hping3 |

## Hardware do Host

- AMD Ryzen 7 5700X (8 cores / 16 threads)
- 32 GB RAM DDR4
- SSD NVMe 1 TB
- VMware Workstation Pro

**Consumo total das VMs:** 22 GB de RAM (sobram 10 GB para o host).
