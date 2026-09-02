<div align="center">

# Laboratório de Detecção de Intrusão

**Suricata · Wazuh · Blue Team · Engenharia de Detecção**

[![Suricata](https://img.shields.io/badge/Suricata-7.x-F6A821?style=flat-square&logo=suricata)](https://suricata.io)
[![Wazuh](https://img.shields.io/badge/Wazuh-4.9-3AABE6?style=flat-square)](https://wazuh.com)
[![MITRE ATT&CK](https://img.shields.io/badge/MITRE_ATT%26CK-Mapeado-ED1C24?style=flat-square)](https://attack.mitre.org)
[![Licença MIT](https://img.shields.io/badge/Licença-MIT-green?style=flat-square)](LICENSE)

</div>

Este projeto foi desenvolvido para estudo prático de Blue Team e Engenharia de Detecção. A arquitetura integra o Suricata como Network Intrusion Detection System (NIDS), responsável pela inspeção de tráfego de rede em tempo real, e o Wazuh como SIEM, utilizado para centralização, normalização e correlação de logs provenientes da rede e dos endpoints monitorados.

O ambiente foi construído em uma infraestrutura virtualizada e isolada, composta por máquinas Linux e Windows monitoradas, além de uma máquina Kali Linux utilizada para executar simulações controladas. O objetivo é validar, de forma prática, o ciclo de detecção de incidentes: geração de telemetria, coleta de eventos, aplicação de regras customizadas, correlação de alertas e análise de comportamentos suspeitos.

[Arquitetura](#arquitetura-do-laboratório) · [Estado Operacional](#estado-operacional) · [Ambiente](#ambiente-monitorado) · [Da Simulação ao Alerta](#da-simulação-ao-alerta) · [Casos de Detecção](#casos-de-detecção) · [Regras](#regras-de-detecção) · [Documentação](#documentação-técnica)

> **Aviso de Uso Ético e Ambiente Autorizado**  
> Todas as simulações, regras e análises documentadas neste repositório foram executadas exclusivamente em ambiente de laboratório virtualizado, controlado e isolado (VMware VMnet2, sem conexão externa). As técnicas demonstradas têm finalidade estritamente educacional e de pesquisa defensiva.

---

## Arquitetura do Laboratório

A arquitetura foi projetada para segmentar funções operacionais entre atacante, sensor passivo de rede, SIEM central e endpoints-alvo heterogêneos, garantindo isolamento de tráfego e visibilidade independente.

<div align="center">

![Diagrama de Arquitetura do Laboratório](imagens/arquitetura.svg)

</div>

<details>
<summary><strong>Fluxo de telemetria e detecção</strong> (clique para expandir)</summary>

```text
[ 1. Kali Linux ] ─── Atividade controlada (VMnet2) ───► [ Endpoints Alvo (Linux / Win) ]
                              │                                      │
                              ▼                                      ▼
                [ 2. Suricata NIDS (ens34) ]            [ Logs Locais (auth.log / Security.evtx) ]
                Modo promíscuo (AF_PACKET)                           │
                              │                                      │
                              ▼                                      │
                [ Alertas: eve.json ]                                │
                              │                                      │
                              └──────────────┬───────────────────────┘
                                             ▼
                               [ 3. Wazuh Agent (Porta 1514) ]
                                             │
                                             ▼
                              [ 4. Wazuh Manager (Correlação) ]
                                             │
                                             ▼
                             [ 5. Wazuh Dashboard (Análise) ]
```

</details>

---

## Estado Operacional

| Componente / Área | Estado | Detalhes Técnicos |
|---|---|---|
| Virtualização e Rede | `Operacional` | Segmento isolado VMware VMnet2 (`10.0.1.0/24`) com conectividade validada |
| Wazuh SIEM (Manager/Indexer/Dashboard) | `Operacional` | Instalação All-in-One v4.9 ativa na VM-SIEM (`10.0.1.50`) |
| Suricata NIDS | `Operacional` | Motor v7.x capturando via `AF_PACKET` em interface promíscua |
| Wazuh Agents Linux | `Operacional` | Versão `4.9.2-1` ativa no Sensor e na Vítima Linux |
| Wazuh Agent Windows | `Operacional` | Versão `4.9.2-1` ativa e monitorando canal de segurança (`Security.evtx`) |
| Regras Customizadas (Suricata / Wazuh) | `Configurado` | 6 regras Suricata e 7 regras locais Wazuh carregadas |
| Casos de Teste Ofensivos | `Em validação` | Cenários mapeados e comandos definidos |
| Evidências de Dashboard e Logs | `Em coleta de evidências` | Estrutura de diretórios pronta para inclusão de capturas e JSONs |
| Integração de Notificações Telegram | `Planejado` | Implementação via Webhook/Script para alertas de alta severidade |

---

## Ambiente Monitorado

| Máquina | Sistema Operacional | Função | IP Privado |
|---|---|---|---|
| **VM-SIEM** | Ubuntu Server 22.04 LTS | Wazuh All-in-One (Manager, Indexer, Dashboard) | `10.0.1.50` |
| **VM-Sensor** | Ubuntu Server 22.04 LTS | Sensor NIDS Suricata em modo promíscuo + Agente Wazuh | `10.0.1.10` |
| **VM-Vítima Linux** | Ubuntu Server 22.04 LTS | Servidor de serviços: Apache (80), OpenSSH (22), vsftpd (21) | `10.0.1.200` |
| **VM-Vítima Windows** | Windows 10/11 Enterprise | Estação de trabalho: RDP (3389), SMB (445), Auditoria de Logon | `10.0.1.201` |
| **VM-Atacante** | Kali Linux 2024 | Plataforma para simulações controladas (Nmap, Hydra, cURL) | `10.0.1.100` |

> Os endereços apresentados pertencem exclusivamente à rede privada e isolada do laboratório (RFC 1918).

### Especificações do Ambiente Host

| Componente | Especificação |
|---|---|
| **Processador** | AMD Ryzen 7 5700X (8 Cores / 16 Threads) |
| **Memória RAM** | 32 GB DDR4 (22 GB alocados dinamicamente para o laboratório) |
| **Armazenamento** | SSD NVMe 1 TB |
| **Hipervisor** | VMware Workstation Pro |
| **Segmentação** | Switch Virtual `VMnet2` (Host-Only, sem DHCP) |

---

## Da Simulação ao Alerta

O ciclo de detecção transforma tráfego e eventos brutos em inteligência acionável por meio de uma esteira estruturada em quatro etapas consecutivas:

| Etapa | Componente | Resultado |
|---|---|---|
| **1. Simulação** | Kali Linux (`10.0.1.100`) | Geração de atividade controlada e direcionada contra os serviços dos alvos |
| **2. Coleta** | Suricata e Wazuh Agents | Registro de telemetria de rede (`eve.json`) e eventos de host (`auth.log`, `Security.evtx`) |
| **3. Correlação** | Wazuh Manager (`10.0.1.50`) | Cruzamento de eventos com regras locais, mapeamento MITRE e atribuição de severidade |
| **4. Análise** | Wazuh Dashboard | Triagem, investigação inicial e registro de evidências pelo analista |

---

## Casos de Detecção

| ID | Cenário | Fonte de Telemetria | MITRE ATT&CK | Status | Evidência |
|---:|---|---|---|---|---|
| 01 | Varredura SYN de Portas | Suricata NIDS (Tráfego de rede) | `T1046` Network Service Discovery | `Em validação` | [testes/README.md](testes/README.md) |
| 02 | Tentativas Repetidas de Autenticação SSH | Suricata NIDS + `auth.log` | `T1110.001` Password Guessing | `Em validação` | [testes/README.md](testes/README.md) |
| 03 | Tentativas Repetidas de Autenticação RDP | Windows Event Log (`Security.evtx` Event ID 4625) | `T1110.001` Password Guessing | `Em validação` | [testes/README.md](testes/README.md) |
| 04 | Exploração Web Shellshock | Suricata NIDS (Headers HTTP L7) | `T1190` Exploit Public-Facing App | `Em validação` | [testes/README.md](testes/README.md) |
| 05 | Autenticação FTP Anônima | Suricata NIDS (Comandos FTP L7) | *Em revisão (Misconfiguration)* | `Em validação` | [testes/README.md](testes/README.md) |
| 06 | Tráfego ICMP Anômalo com Carga Elevada | Suricata NIDS (Inspeção de Payload) | *Em revisão (T1095 / T1572)* | `Em validação` | [testes/README.md](testes/README.md) |
| 07 | Cadeia de Ataque: Scan seguido de Força Bruta | Wazuh Manager (Correlação Temporal 10 min) | `T1046` + `T1110` | `Em validação` | [testes/README.md](testes/README.md) |

---

## Regras de Detecção

A engenharia de detecção do projeto opera em duas camadas complementares:

1. **Inspeção de Rede (Suricata):** Assinaturas focadas em anomalias de protocolo, padrões de carga útil (payloads) e limites de frequência (thresholds). Arquivo: [`regras/suricata-custom.rules`](regras/suricata-custom.rules).
2. **Correlação de Eventos (Wazuh):** Regras baseadas em XML que enriquecem os alertas do NIDS e dos agentes, atribuem níveis de severidade (1 a 15) e implementam correlação temporal multi-estágio (Regra `100021`). Arquivo: [`regras/wazuh-local-rules.xml`](regras/wazuh-local-rules.xml).

A documentação detalhada do relacionamento entre cada regra Suricata e sua respectiva regra no Wazuh está disponível em [`regras/README.md`](regras/README.md).

> As regras devem ser relacionadas a cenários testados e acompanhadas de evidências reais antes de serem consideradas validadas.

---

## Evidências e Relatórios

Para assegurar reprodutibilidade e autenticidade técnica, cada caso de detecção executado no laboratório será documentado com:

- Captura de tela sanitizada da interface do Wazuh Dashboard.
- Evento bruto relevante (JSON do `eve.json` ou entrada de log de host).
- Identificador da regra acionada (SID / Rule ID).
- Técnica correspondente no MITRE ATT&CK.
- Comparativo entre resultado esperado e resultado observado.
- Análise técnica e recomendações do analista.

Os diretórios de evidências e relatórios foram preparados para receber validações reais à medida que os cenários forem executados:

- Índice de cenários de teste: [`testes/README.md`](testes/README.md)
- Modelo padrão para documentação: [`testes/modelo-relatorio.md`](testes/modelo-relatorio.md)
- Diretório para capturas do SIEM: [`imagens/dashboard/`](imagens/dashboard/)
- Diretório para capturas das simulações: [`imagens/evidencias/`](imagens/evidencias/)

---

## Documentação Técnica

| Documento | Descrição |
|---|---|
| [`docs/arquitetura.md`](docs/arquitetura.md) | Topologia de rede, segmentação e dimensionamento de hardware |
| [`docs/instalacao.md`](docs/instalacao.md) | Procedimento passo a passo para replicação do laboratório do zero |
| [`docs/troubleshooting.md`](docs/troubleshooting.md) | Guia estruturado de diagnóstico e resolução de falhas operacionais |
| [`regras/README.md`](regras/README.md) | Mapeamento completo entre SIDs Suricata e Rule IDs Wazuh |
| [`testes/README.md`](testes/README.md) | Escopo dos testes e registro de status de execução |
| [`notas.md`](notas.md) | Diário de bordo técnico com problemas reais e soluções aplicadas |
| [`SECURITY.md`](SECURITY.md) | Política de segurança, escopo de IPs e tratamento de credenciais |
| [`LICENSE`](LICENSE) | Termos de uso e distribuição sob Licença MIT |

---

## Aprendizados Técnicos

A implementação e sustentação do ambiente demandou resolução prática de desafios de infraestrutura e engenharia de detecção:

- **Segmentação de Redes Virtuais:** Configuração de adaptador host-only dedicado (`VMnet2`) isolado da interface NAT (`VMnet8`), garantindo que o tráfego de teste permaneça confinado.
- **Captura Promíscua com AF_PACKET:** Ajuste do Suricata para operar em modo promíscuo na interface virtual, permitindo inspeção passiva sem necessidade de inline/IPS.
- **Pipeline de Telemetria NIDS-to-SIEM:** Configuração do Wazuh Agent no sensor para ingestão e parsing estruturado do arquivo `eve.json`.
- **Alinhamento de Versões no Wazuh:** Resolução de incompatibilidade impedindo o registro de agentes mais novos que o Manager, estabelecendo o versionamento fixo no pacote `4.9.2-1`.
- **Expansão de Volumes LVM no Linux:** Resolução do esgotamento de disco no Wazuh Indexer decorrente do particionamento padrão do Ubuntu Server (uso de `lvextend` e `resize2fs`).
- **Resiliência de Conexão dos Agentes:** Identificação e tratamento de perda de handshake dos agentes após eventos de suspensão/retomada de máquinas virtuais.
- **Regras de Firewall com Localização do S.O.:** Tratamento de falhas em scripts de automação PowerShell no Windows decorrentes de nomes localizados em português ("Área de Trabalho Remota"), padronizando a liberação via `netsh` por porta.

---

## Próximas Etapas

- [ ] Executar formalmente a bateria de testes a partir da VM-Atacante.
- [ ] Coletar capturas sanitizadas dos alertas gerados no Wazuh Dashboard.
- [ ] Criar os relatórios individuais de cada cenário com base no modelo padronizado.
- [ ] Validar a precisão dos thresholds e níveis de severidade das regras locais.
- [ ] Concluir a revisão técnica das associações MITRE ATT&CK para tráfego anômalo e FTP.
- [ ] Validar script de integração para envio de alertas críticos via Telegram.

---

## Autor, Ética e Licença

Projeto desenvolvido por **Leonardo Ramos** ([@LeoSec1](https://github.com/LeoSec1)) para consolidação prática de competências em Segurança Defensiva, Operações de SOC e Engenharia de Detecção.

- **Uso Responsável:** Todo o conteúdo é destinado estritamente para pesquisa e capacitação defensiva em ambientes devidamente autorizados. Para mais diretrizes, consulte [`SECURITY.md`](SECURITY.md).
- **Licença:** Distribuído sob os termos da [Licença MIT](LICENSE).
