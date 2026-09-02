<div align="center">

# SOC Homelab: Detecção de Intrusão com Suricata e Wazuh

Monitoramento de segurança de rede com NIDS e SIEM em ambiente virtualizado com Linux e Windows, integrado a alertas em tempo real no Telegram.

![Suricata](https://img.shields.io/badge/Suricata-7.x-F6A821?style=flat-square&logo=suricata)
![Wazuh](https://img.shields.io/badge/Wazuh-4.9-3AABE6?style=flat-square)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![Windows](https://img.shields.io/badge/Windows-10%2F11-0078D4?style=flat-square&logo=windows&logoColor=white)
![Kali](https://img.shields.io/badge/Kali_Linux-2024-557C94?style=flat-square&logo=kalilinux&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-Alertas-229ED9?style=flat-square&logo=telegram&logoColor=white)
![MITRE](https://img.shields.io/badge/MITRE_ATT%26CK-Mapeado-ED1C24?style=flat-square)

</div>

---

## Sobre o Projeto

Laboratório desenvolvido para estudo e validação de detecção de intrusão em cenário híbrido (Linux e Windows). O ambiente utiliza o Suricata como sensor NIDS para análise de tráfego em modo promíscuo e o Wazuh como SIEM para centralização de logs, correlação de eventos e notificações automáticas no Telegram.

---

## Arquitetura do Laboratório

<div align="center">

![Arquitetura do Laboratório](imagens/arquitetura.svg)

</div>

### Especificações das Máquinas Virtuais

| Máquina | Sistema Operacional | Recursos | Papel no Laboratório |
|---|---|---|---|
| **VM-SIEM** | Ubuntu Server 22.04 LTS | 8 GB RAM · 4 vCPUs · 50 GB | Wazuh Manager, Indexer (OpenSearch) e Dashboard |
| **VM-Sensor** | Ubuntu Server 22.04 LTS | 4 GB RAM · 2 vCPUs · 30 GB | Suricata 7.x NIDS em modo promíscuo + Agente Wazuh |
| **VM-Vítima** | Ubuntu Server 22.04 LTS | 2 GB RAM · 2 vCPUs · 20 GB | Serviços-alvo: Apache HTTP (80), OpenSSH (22), vsftpd (21) |
| **VM-Vítima-Windows** | Windows 10 / 11 | 4 GB RAM · 2 vCPUs · 40 GB | Serviços-alvo: RDP (3389), SMB (445) e Event Viewer |
| **VM-Atacante** | Kali Linux 2024 | 4 GB RAM · 2 vCPUs · 25 GB | Ferramentas ofensivas: Nmap, Hydra, Hping3, cURL |

> **Ambiente Host:** AMD Ryzen 7 5700X (8C/16T), 32 GB RAM DDR4, SSD NVMe 1 TB, VMware Workstation Pro.  
> **Rede Isolada:** VMware VMnet2 (`10.0.1.0/24`).

---

## Testes Realizados e Detecção

| Cenário de Ataque | Alvo | Resultado e Severidade no SIEM |
|---|---|---|
| **Varredura SYN de Portas** (Nmap) | Linux e Windows | Alerta Nível 8 (Suricata SID 1000001) |
| **Força Bruta SSH** (Hydra) | Vítima Linux | Alerta Nível 10 (Wazuh Rule 5763) + Notificação Telegram |
| **Força Bruta RDP** (Hydra) | Vítima Windows | Alerta Nível 10 (Event ID 4625) + Notificação Telegram |
| **Exploração Web Shellshock** (cURL) | Vítima Linux | Alerta Nível 14 (Suricata SID 1000005) + Notificação Telegram |
| **Login FTP Anônimo** (cURL) | Vítima Linux | Alerta Nível 5 (Suricata SID 1000004) |
| **Pacote ICMP Anômalo** (Hping3) | Vítima Linux | Alerta Nível 6 (Suricata SID 1000003) |

---

## Tecnologias Aplicadas

| Categoria | Tecnologia | Aplicação Prática no Projeto |
|---|---|---|
| **Sistemas Operacionais** | Ubuntu Server 22.04, Windows 10/11, Kali Linux | Infraestrutura híbrida virtualizada no VMware |
| **Detecção de Rede (NIDS)** | Suricata 7.x | Inspeção profunda de pacotes em modo promíscuo (`AF_PACKET`) |
| **SIEM / XDR** | Wazuh 4.9 (Manager, Indexer, Dashboard) | Centralização de logs, regras de correlação e severidades |
| **Auditoria Windows** | Windows Event Logs (`Security.evtx`) | Monitoramento de Event IDs de autenticação e logon |
| **Notificação em Tempo Real** | Bot do Telegram (Webhook / API) | Alertas push automáticos para incidentes críticos (Nível 10+) |
| **Classificação de Ameaças** | MITRE ATT&CK | Mapeamento formal de táticas e técnicas adversárias |

---

## Estrutura do Repositório

```
soc-homelab/
├── README.md                  # Apresentação do projeto e resultados dos testes
├── .gitignore                 # Filtro de arquivos sensíveis e binários
├── docs/
│   └── arquitetura.md         # Topologia detalhada e justificativa técnica
├── regras/
│   ├── suricata-custom.rules  # Regras de assinatura do Suricata comentadas
│   └── wazuh-local-rules.xml  # Regras de correlação e severidade do Wazuh
├── testes/                    # Relatórios de execução e evidências de cada teste
├── imagens/
│   └── arquitetura.svg        # Diagrama vetorial moderno da arquitetura
└── notas.md                   # Registro técnico de problemas encontrados e soluções
```

---

## Roadmap

- [ ] Implementação de Threat Intelligence com CrowdSec e bouncers de firewall.
- [ ] Visibilidade profunda de endpoint no Windows com Microsoft Sysmon.
- [ ] Automação de playbooks de resposta a incidentes com Shuffle SOAR.

---

<div align="center">

Desenvolvido por **Leonardo Ramos**  
[GitHub](https://github.com/LeoSec1)

</div>
