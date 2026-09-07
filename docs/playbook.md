# Playbook Operacional Completo: SOC Homelab
**Guia Passo a Passo de Replicação, Configuração, Simulação e Troubleshooting**

---

## 1. Topologia, Matriz de Endereçamento e Credenciais

### 1.1 Matriz de Rede (`VMnet2` - Host-Only `10.0.1.0/24`)

| Hostname | Função | IP Estático | Sistema Operacional | RAM | vCPU | Disco |
|---|---|---|---|:---:|:---:|:---:|
| `vm-siem` | Wazuh All-in-One (Manager/Indexer/Dashboard) | `10.0.1.50` | Ubuntu Server 22.04 LTS | 8 GB | 4 | 50 GB |
| `vm-sensor` | Suricata 7.x NIDS (AF_PACKET Promíscuo) | `10.0.1.10` | Ubuntu Server 22.04 LTS | 4 GB | 2 | 30 GB |
| `vm-vitima` | Vítima Linux (Apache, SSH, vsftpd) | `10.0.1.200` | Ubuntu Server 22.04 LTS | 2 GB | 2 | 20 GB |
| `vm-vitima-windows` | Vítima Windows (RDP, SMB, EventLog) | `10.0.1.201` | Windows 10/11 Enterprise | 4 GB | 2 | 40 GB |
| `vm-atacante` | Simulação de Ameaças (Kali Linux) | `10.0.1.100` | Kali Linux 2024 | 4 GB | 2 | 25 GB |

### 1.2 Credenciais e Usuários Padrão do Laboratório

| Máquina | Usuário | Senha Padrão de Lab | Finalidade |
|---|---|---|---|
| `vm-siem` | `socadmin` | `<DEFINIDA_NA_INSTALACAO>` | Acesso SSH ao servidor SIEM |
| `vm-siem` (Web) | `admin` | *(Gerada no `wazuh-passwords.txt`)* | Acesso ao Wazuh Dashboard HTTPS |
| `vm-sensor` | `sensoradmin` | `<DEFINIDA_NA_INSTALACAO>` | Acesso SSH ao NIDS |
| `vm-vitima` | `adminlab` | `LabAdmin123!` | Alvo de testes de força bruta SSH |
| `vm-vitima` (FTP) | `anonymous` | *(qualquer e-mail / em branco)* | Alvo de teste de login anônimo |
| `vm-vitima-windows` | `Administrator` | `LabWindows2026!` | Alvo de testes de força bruta RDP |
| `vm-atacante` | `kali` | `kali` | Acesso à estação ofensiva |

---

## 2. Fase 0: Configuração do Hipervisor (VMware Workstation Pro)

### 2.1 Configuração dos Switches Virtuais

1. Abra o VMware Workstation Pro como Administrador.
2. Acesse: **Edit** -> **Virtual Network Editor...**
3. Clique em **Change Settings** (requer privilégios elevados).
4. Configure a rede **VMnet2**:
   - Tipo: **Host-only (connect VMs internally in a private network)**
   - **Desmarque:** *Connect a host virtual adapter to this network* (opcional: mantenha apenas se desejar acessar a interface web diretamente do host físico).
   - **Desmarque obrigatoriamente:** *Use local DHCP service to distribute IP addresses to VMs*.
   - **Subnet IP:** `10.0.1.0`
   - **Subnet mask:** `255.255.255.0`
5. Confirme que a rede **VMnet8** (NAT) está ativa para acesso temporário à internet durante a fase de instalação dos pacotes.

### 2.2 Criação dos Adaptadores nas VMs

* **`vm-siem`**: Adaptador 1 na `VMnet8` (NAT temporário) + Adaptador 2 na `VMnet2` (IP `10.0.1.50`).
* **`vm-sensor`**: Adaptador 1 na `VMnet8` (NAT temporário) + Adaptador 2 na `VMnet2` (Gerência `10.0.1.10`) + Adaptador 3 na `VMnet2` (Sniffer Promíscuo `ens34` sem IP).
* **`vm-vitima`**: Adaptador 1 na `VMnet8` (NAT temporário) + Adaptador 2 na `VMnet2` (IP `10.0.1.200`).
* **`vm-vitima-windows`**: Adaptador 1 na `VMnet8` (NAT temporário) + Adaptador 2 na `VMnet2` (IP `10.0.1.201`).
* **`vm-atacante`**: Adaptador 1 na `VMnet8` (NAT temporário) + Adaptador 2 na `VMnet2` (IP `10.0.1.100`).

> **Nota de Segurança:** Após instalar os pacotes e atualizar os sistemas, **desconecte o Adaptador NAT (VMnet8)** de todas as máquinas antes de iniciar as simulações de ataque.

---

## 3. Fase 1: Provisionamento Base e Expansão de Disco (Ubuntu 22.04 LTS)

Execute este procedimento em **todas as 3 VMs Ubuntu** (`vm-siem`, `vm-sensor`, `vm-vitima`):

### 3.1 Expansão do Volume Lógico LVM (Evita colapso por disco cheio)

O instalador oficial do Ubuntu Server aloca por padrão apenas 50% do espaço provisionado:

```bash
# Verificar espaço alocado atual
df -h /

# Expandir o volume lógico para 100% do espaço livre do volume group
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

# Redimensionar o sistema de arquivos ext4
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv

# Validar novo tamanho alocado
df -h /
```

### 3.2 Configuração de Rede Estática (Netplan)

Substitua o arquivo de configuração de rede em `/etc/netplan/00-installer-config.yaml` de acordo com a máquina:

#### Na `vm-siem` (`10.0.1.50`):
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
      optional: true
    ens34:
      dhcp4: no
      addresses:
        - 10.0.1.50/24
```

#### Na `vm-sensor` (`10.0.1.10`):
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
      optional: true
    ens34:
      dhcp4: no
      addresses:
        - 10.0.1.10/24
    ens35:
      dhcp4: no
      optional: true
```

#### Na `vm-vitima` (`10.0.1.200`):
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
      optional: true
    ens34:
      dhcp4: no
      addresses:
        - 10.0.1.200/24
```

Aplique as configurações em cada uma:
```bash
sudo netplan apply
ip addr show
```

---

## 4. Fase 2: Instalação do Wazuh SIEM All-in-One (`vm-siem` - `10.0.1.50`)

### 4.1 Download e Instalação Automatizada

```bash
# 1. Atualizar repositórios
sudo apt update && sudo apt upgrade -y
sudo apt install curl apt-transport-https gnupg tar -y

# 2. Baixar o assistente de instalação oficial Wazuh 4.9
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.9/config.yml

# 3. Editar o arquivo config.yml para definir o IP 10.0.1.50
cat << 'EOF' > config.yml
nodes:
  indexer:
    - name: node-1
      ip: "10.0.1.50"
  server:
    - name: wazuh-1
      ip: "10.0.1.50"
  dashboard:
    - name: dashboard
      ip: "10.0.1.50"
EOF

# 4. Executar a instalação All-in-One desassistida
sudo bash wazuh-install.sh -a

# 5. Salvar o arquivo de senhas gerado em local seguro
sudo cp wazuh-install-files.tar ~/wazuh-install-files.tar
sudo tar -tf ~/wazuh-install-files.tar
sudo tar -xvf ~/wazuh-install-files.tar wazuh-passwords.txt
cat wazuh-passwords.txt
```

### 4.2 Verificação dos Serviços e Portas

```bash
# Confirmar status ativo de todos os componentes
sudo systemctl status wazuh-manager --no-pager
sudo systemctl status wazuh-indexer --no-pager
sudo systemctl status wazuh-dashboard --no-pager

# Validar portas essenciais em escuta
# 1514 (Agent connection), 1515 (Agent registration), 443 (Dashboard), 9200 (Indexer)
sudo ss -tulpn | grep -E '1514|1515|443|9200'
```

### 4.3 Ajuste Preventivo de Espaço (Prevenção do Bug de Vulnerability Detector)

Em redes isoladas sem conexão externa contínua, o updater de vulnerabilidades pode encher o disco com downloads parciais:

```bash
# Criar tarefa de expurgo do cache temporário se necessário
sudo rm -rf /var/ossec/queue/vd_updater/tmp/*
sudo rm -rf /var/ossec/queue/vd/feed/*
```

---

## 5. Fase 3: Implantação do Sensor NIDS Suricata 7.x (`vm-sensor` - `10.0.1.10`)

### 5.1 Instalação do Suricata e Dependências

```bash
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt update
sudo apt install suricata jq net-tools -y
```

### 5.2 Persistência do Modo Promíscuo na Interface de Captura

Identifique a interface conectada à `VMnet2` dedicada ao monitoramento (exemplo: `ens34` ou `ens35`):

```bash
# Criar serviço systemd para manter PROMISC ativo após reboot
sudo tee /etc/systemd/system/promisc-ens34.service << 'EOF'
[Unit]
Description=Garante Modo Promiscuo Persistente para o Suricata na interface ens34
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ip link set ens34 promisc on
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# Habilitar e iniciar o serviço
sudo systemctl daemon-reload
sudo systemctl enable --now promisc-ens34.service

# Validar se a flag PROMISC está ativa
ip link show ens34 | grep -i promisc
```

### 5.3 Configuração do `suricata.yaml`

Edite `/etc/suricata/suricata.yaml` ajustando os seguintes blocos principais:

```yaml
vars:
  address-groups:
    HOME_NET: "[10.0.1.0/24]"
    EXTERNAL_NET: "!$HOME_NET"

af-packet:
  - interface: ens34
    threads: auto
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
    use-mmap: yes
    mmsg-size: 16

default-rule-path: /var/lib/suricata/rules
rule-files:
  - suricata.rules
  - suricata-custom.rules

outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - alert:
            payload: yes
            payload-printable: yes
            packet: yes
            metadata: yes
            http-body: yes
        - http
        - dns
        - tls
        - files
        - drop
```

### 5.4 Implantação das Regras Customizadas do Suricata

Crie o arquivo `/var/lib/suricata/rules/suricata-custom.rules`:

```bash
sudo tee /var/lib/suricata/rules/suricata-custom.rules << 'EOF'
# ============================================================
# Regras customizadas do Suricata para o SOC Homelab
# Rede monitorada: 10.0.1.0/24 (VMnet2)
# ============================================================

# 1. Detecta varredura SYN (20 pacotes SYN do mesmo IP em 5 segundos)
alert tcp any any -> $HOME_NET any (msg:"SOC-HOMELAB - Varredura SYN Detectada"; flags:S,12; threshold:type both, track by_src, count 20, seconds 5; classtype:attempted-recon; sid:1000001; rev:1;)

# 2. Detecta tentativa de forca bruta SSH (5 conexoes na porta 22 em 30 segundos)
alert tcp any any -> $HOME_NET 22 (msg:"SOC-HOMELAB - Tentativa de Brute Force SSH"; flags:S; threshold:type both, track by_src, count 5, seconds 30; classtype:attempted-admin; sid:1000002; rev:1;)

# 3. Detecta pacote ICMP com payload anomalo (acima de 1000 bytes)
alert icmp any any -> $HOME_NET any (msg:"SOC-HOMELAB - ICMP com Payload Anomalo"; dsize:>1000; classtype:bad-unknown; sid:1000003; rev:1;)

# 4. Detecta login anonimo no FTP
alert tcp any any -> $HOME_NET 21 (msg:"SOC-HOMELAB - Login Anonimo FTP"; flow:to_server,established; content:"USER anonymous"; nocase; classtype:suspicious-login; sid:1000004; rev:1;)

# 5. Detecta tentativa de Shellshock via HTTP
alert http any any -> $HOME_NET any (msg:"SOC-HOMELAB - Tentativa de Shellshock"; content:"() {"; http_header; classtype:web-application-attack; sid:1000005; rev:1;)

# 6. Detecta varredura de versao de servico (Nmap -sV)
alert tcp any any -> $HOME_NET any (msg:"SOC-HOMELAB - Deteccao de Servico (Service Scan)"; flags:S; threshold:type both, track by_src, count 50, seconds 10; classtype:attempted-recon; sid:1000008; rev:1;)
EOF
```

### 5.5 Validação e Inicialização do Suricata

```bash
# Atualizar bases de assinaturas
sudo suricata-update

# Validar a sintaxe sem erros
sudo suricata -T -c /etc/suricata/suricata.yaml -v

# Reiniciar o serviço do Suricata
sudo systemctl restart suricata
sudo systemctl enable suricata
sudo systemctl status suricata --no-pager
```

---

## 6. Fase 4: Configuração das Máquinas Vítimas

### 6.1 VM-Vítima Linux (`vm-vitima` - `10.0.1.200`)

#### Instalação dos Serviços Alvo:
```bash
sudo apt update
sudo apt install apache2 vsftpd openssh-server -y

# 1. Configurar Usuário para Teste de Brute Force SSH
sudo id -u adminlab &>/dev/null || sudo useradd -m -s /bin/bash adminlab
echo "adminlab:LabAdmin123!" | sudo chpasswd
sudo systemctl restart ssh

# 2. Configurar FTP com Login Anônimo Permitido
sudo sed -i 's/anonymous_enable=NO/anonymous_enable=YES/g' /etc/vsftpd.conf
echo "anon_root=/srv/ftp" | sudo tee -a /etc/vsftpd.conf
sudo mkdir -p /srv/ftp
sudo touch /srv/ftp/aviso_confidencial.txt
sudo systemctl restart vsftpd

# 3. Configurar Script Vulnerável Shellshock no Apache
sudo a2enmod cgid
sudo systemctl restart apache2

sudo tee /usr/lib/cgi-bin/test.cgi << 'EOF'
#!/bin/bash
echo "Content-type: text/plain"
echo ""
echo "Servico CGI Ativo - Ambiente de Teste SOC Homelab"
EOF

sudo chmod +x /usr/lib/cgi-bin/test.cgi
sudo systemctl restart apache2
```

### 6.2 VM-Vítima Windows (`vm-vitima-windows` - `10.0.1.201`)

Abra o PowerShell como Administrador na VM Windows:

```powershell
# 1. Habilitar RDP (Area de Trabalho Remota) no Registro
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0

# 2. Liberar porta RDP 3389 no Firewall do Windows (sem dependencia de idioma)
netsh advfirewall firewall add rule name="Allow RDP Lab" dir=in action=allow protocol=TCP localport=3389

# 3. Habilitar Auditoria de Logins com Sucesso e Falha (Gera Event IDs 4624 e 4625)
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Logoff" /success:enable /failure:enable

# 4. Confirmar se a porta 3389 está aberta
Test-NetConnection -ComputerName 127.0.0.1 -Port 3389
```

---

## 7. Fase 5: Instalação e Integração dos Agentes Wazuh

> **Atenção:** A versão do Agente **nunca pode ser maior** que a versão do Manager (`4.9.x`). Portanto, instale fixando explicitamente a versão `4.9.2-1`.

### 7.1 Instalação nas VMs Linux (`vm-sensor` e `vm-vitima`)

Execute no **Sensor (`10.0.1.10`)** e na **Vítima Linux (`10.0.1.200`)**:

```bash
# 1. Importar chave GPG oficial do repositório Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg

# 2. Adicionar repositório APT do Wazuh 4.x
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list
sudo apt-get update

# 3. Instalar o agente fixando a versão 4.9.2-1 apontando para o Manager 10.0.1.50
sudo WAZUH_MANAGER="10.0.1.50" apt-get install -y wazuh-agent=4.9.2-1
```

### 7.2 Integração Específica do Suricata no Agente do Sensor (`vm-sensor`)

Na `vm-sensor`, adicione a leitura do `eve.json` dentro do bloco `<ossec_config>` no arquivo `/var/ossec/etc/ossec.conf`:

```xml
  <!-- Monitoramento do EVE JSON do Suricata -->
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>
```

Inicie o agente no sensor:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent --no-pager
```

Na `vm-vitima`, inicie o agente:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent --no-pager
```

### 7.3 Instalação do Agente na Vítima Windows (`vm-vitima-windows`)

No PowerShell Administrativo:

```powershell
# 1. Download do instalador oficial do Wazuh Agent 4.9.2
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi -OutFile ${env:TEMP}\wazuh-agent-4.9.2-1.msi

# 2. Instalação silenciosa apontando para o Manager 10.0.1.50
Start-Process msiexec.exe -Wait -ArgumentList "/i ${env:TEMP}\wazuh-agent-4.9.2-1.msi /q WAZUH_MANAGER='10.0.1.50' WAZUH_REGISTRATION_SERVER='10.0.1.50'"

# 3. Iniciar o serviço do agente
Start-Service -Name WazuhSvc
Get-Service -Name WazuhSvc
```

### 7.4 Validação dos Agentes Conectados no Manager (`vm-siem`)

Na `vm-siem`, verifique a lista de agentes registrados:

```bash
sudo /var/ossec/bin/manage_agents -l
```
*Saída esperada:*
```text
Available agents: 
   ID: 001, Name: vm-sensor, IP: 10.0.1.10, Active
   ID: 002, Name: vm-vitima, IP: 10.0.1.200, Active
   ID: 003, Name: vm-vitima-windows, IP: 10.0.1.201, Active
```

---

## 8. Fase 5: Regras de Correlação Customizadas no Wazuh (`vm-siem`)

### 8.1 Implantação no `local_rules.xml`

Edite `/var/ossec/etc/rules/local_rules.xml` no Manager e adicione o bloco abaixo dentro da tag `<group name="local,syslog,sshd,">`:

```xml
<!-- ============================================================ -->
<!-- Regras de correlacao customizadas do Wazuh para o SOC Homelab -->
<!-- ============================================================ -->

<group name="local,suricata,soc-homelab,">

  <!-- Alerta generico do Suricata capturado pelo Wazuh -->
  <rule id="100001" level="3">
    <if_sid>86601</if_sid>
    <description>SOC-HOMELAB: Alerta do Suricata capturado</description>
    <group>ids,suricata,</group>
  </rule>

  <!-- Varredura de rede detectada pelo Suricata (SID 1000001) -->
  <rule id="100010" level="8">
    <if_sid>86601</if_sid>
    <field name="alert.signature_id">1000001</field>
    <description>SOC-HOMELAB: Varredura SYN detectada na rede</description>
    <mitre>
      <id>T1046</id>
    </mitre>
    <group>ids,recon,</group>
  </rule>

  <!-- Tentativa de brute force SSH detectada pelo Suricata (SID 1000002) -->
  <rule id="100020" level="10">
    <if_sid>86601</if_sid>
    <field name="alert.signature_id">1000002</field>
    <description>SOC-HOMELAB: Tentativa de brute force SSH detectada</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>ids,brute_force,</group>
  </rule>

  <!-- ICMP com payload anomalo (SID 1000003) -->
  <rule id="100030" level="6">
    <if_sid>86601</if_sid>
    <field name="alert.signature_id">1000003</field>
    <description>SOC-HOMELAB: Pacote ICMP com payload anomalo</description>
    <mitre>
      <id>T1095</id>
    </mitre>
    <group>ids,anomaly,</group>
  </rule>

  <!-- Login anonimo FTP (SID 1000004) -->
  <rule id="100040" level="5">
    <if_sid>86601</if_sid>
    <field name="alert.signature_id">1000004</field>
    <description>SOC-HOMELAB: Login anonimo FTP detectado</description>
    <mitre>
      <id>T1078</id>
    </mitre>
    <group>ids,suspicious_login,</group>
  </rule>

  <!-- Tentativa de Shellshock (SID 1000005) -->
  <rule id="100050" level="14">
    <if_sid>86601</if_sid>
    <field name="alert.signature_id">1000005</field>
    <description>SOC-HOMELAB: Tentativa de exploit Shellshock via HTTP</description>
    <mitre>
      <id>T1190</id>
    </mitre>
    <group>ids,exploit,web,</group>
  </rule>

  <!-- Correlacao: Scan + Brute Force do mesmo IP em 10 minutos -->
  <rule id="100021" level="14" timeframe="600">
    <if_sid>100020</if_sid>
    <if_matched_sid>100010</if_matched_sid>
    <same_source_ip />
    <description>SOC-HOMELAB: Cadeia de ataque detectada (Scan seguido de Brute Force SSH do mesmo IP)</description>
    <mitre>
      <id>T1046</id>
      <id>T1110</id>
    </mitre>
    <group>ids,attack_chain,</group>
  </rule>

</group>
```

### 8.2 Teste com `wazuh-logtest` e Reinicialização

```bash
# Validar se as regras compilam corretamente
sudo /var/ossec/bin/wazuh-logtest << 'EOF'
{"timestamp":"2026-09-07T00:00:01.000000+0000","event_type":"alert","src_ip":"10.0.1.100","src_port":44444,"dest_ip":"10.0.1.200","dest_port":22,"proto":"TCP","alert":{"action":"allowed","gid":1,"signature_id":1000001,"rev":1,"signature":"SOC-HOMELAB - Varredura SYN Detectada","category":"Attempted Information Leak","severity":3}}
EOF

# Reiniciar o serviço do Wazuh Manager
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

---

## 9. Fase 6: Configuração da VM-Atacante (Kali Linux - `10.0.1.100`)

Na `vm-atacante`:

```bash
# 1. Configurar IP fixo na interface conectada a VMnet2
sudo ip addr add 10.0.1.100/24 dev eth1
sudo ip link set eth1 up

# 2. Testar conectividade estrita com as VMs do laboratório
ping -c 2 10.0.1.10
ping -c 2 10.0.1.50
ping -c 2 10.0.1.200
ping -c 2 10.0.1.201

# 3. Desconectar o adaptador NAT (VMnet8) nas configurações da VM no VMware
```

---

## 10. Fase 7: Execução dos Cenários de Teste e Validação de Detecção

Execute os comandos a partir da **`vm-atacante` (`10.0.1.100`)**:

### Teste 01: Varredura de Portas SYN (MITRE ATT&CK: T1046)
```bash
# Execução no Kali:
nmap -sS -p 1-1000 10.0.1.200

# Validação no Sensor (Suricata):
sudo grep -i "1000001" /var/log/suricata/eve.json | tail -n 1 | jq .

# Validação no SIEM (Wazuh):
# Alerta gerado: Rule ID 100010 (Nível 8) - "SOC-HOMELAB: Varredura SYN detectada na rede"
```

### Teste 02: Força Bruta SSH (MITRE ATT&CK: T1110)
```bash
# Execução no Kali:
hydra -l adminlab -P /usr/share/wordlists/rockyou.txt ssh://10.0.1.200 -t 4 -s 22

# Validação no Sensor (Suricata):
sudo grep -i "1000002" /var/log/suricata/eve.json | tail -n 1 | jq .

# Validação no SIEM (Wazuh):
# Alerta gerado: Rule ID 100020 (Nível 10) - "SOC-HOMELAB: Tentativa de brute force SSH detectada"
```

### Teste 03: Exploração Web Shellshock (MITRE ATT&CK: T1190)
```bash
# Execução no Kali:
curl -H "User-Agent: () { :; }; /bin/bash -c 'cat /etc/passwd'" http://10.0.1.200/cgi-bin/test.cgi

# Validação no Sensor e Vítima:
sudo grep -i "1000005" /var/log/suricata/eve.json | tail -n 1 | jq .
sudo tail -n 5 /var/log/apache2/error.log

# Validação no SIEM (Wazuh):
# Alerta gerado: Rule ID 100050 (Nível 14) + Rule ID 31105 (Nível 15) no Apache
```

### Teste 04: Força Bruta RDP no Windows (MITRE ATT&CK: T1110)
```bash
# Criar wordlist reduzida no Kali:
cat << 'EOF' > /tmp/rdp_passwords.txt
123456
password
admin123
LabWindows2026!
EOF

# Execução no Kali:
hydra -l Administrator -P /tmp/rdp_passwords.txt rdp://10.0.1.201 -t 1

# Validação no SIEM (Wazuh):
# Alerta gerado no endpoint Windows: Rule ID 60204 (Event ID 4625 - Falha de logon no Windows)
```

### Teste 05: Login Anônimo no FTP (MITRE ATT&CK: T1078.001)
```bash
# Execução no Kali:
python3 -c "
import ftplib
ftp = ftplib.FTP('10.0.1.200')
ftp.login('anonymous', 'anonymous@lab.local')
print(ftp.retrlines('LIST'))
ftp.quit()
"

# Validação no Sensor (Suricata):
sudo grep -i "1000004" /var/log/suricata/eve.json | tail -n 1 | jq .

# Validação no SIEM (Wazuh):
# Alerta gerado: Rule ID 100040 (Nível 5) - "SOC-HOMELAB: Login anonimo FTP detectado"
```

### Teste 06: Tráfego Anômalo ICMP com Payload Excessivo (MITRE ATT&CK: T1095)
```bash
# Execução no Kali:
sudo hping3 --icmp -d 1200 10.0.1.200 -c 5

# Validação no Sensor (Suricata):
sudo grep -i "1000003" /var/log/suricata/eve.json | tail -n 1 | jq .

# Validação no SIEM (Wazuh):
# Alerta gerado: Rule ID 100030 (Nível 6) - "SOC-HOMELAB: Pacote ICMP com payload anomalo"
```

### Teste 07: Cadeia de Ataque Correlacionada (MITRE ATT&CK: T1046 -> T1110)
```bash
# Execução encadeada imediata no Kali pelo mesmo IP 10.0.1.100:
nmap -sS -p 1-1000 10.0.1.200 && sleep 5 && hydra -l adminlab -P /tmp/rdp_passwords.txt ssh://10.0.1.200 -t 4

# Validação no SIEM (Wazuh):
# Disparo correlacionado: Rule ID 100021 (Nível 14)
# "SOC-HOMELAB: Cadeia de ataque detectada (Scan seguido de Brute Force SSH do mesmo IP)"
```

---

## 11. Fase 8: Importação do Dashboard Customizado no OpenSearch / Wazuh

### 11.1 Acesso Seguro ao Wazuh Dashboard

1. Abra o navegador em **Modo Anônimo / Privado** (`Ctrl + Shift + N`):
   - *Motivo:* Evita extensões de tradução automática do navegador que corrompem o código JavaScript do plugin Wazuh e extensões de antivírus (como Kaspersky) que bloqueiam alertas contendo assinaturas de ataque.
2. Acesse: `https://10.0.1.50`
3. Aceite o certificado SSL autoassinado.
4. Usuário: `admin` | Senha: `<SENHA_WAZUH_ADMIN_DE_WAZUH_PASSWORDS_TXT>`

### 11.2 Importação do Pacote de Visualizações

O arquivo de exportação está localizado na raiz do repositório: `soc-homelab-dashboard.ndjson`.

1. No menu lateral do Wazuh Dashboard, acesse:
   **Management** -> **Stack Management** -> **Saved Objects**
2. Clique no botão **Import** no canto superior direito.
3. Arraste e solte o arquivo `soc-homelab-dashboard.ndjson`.
4. Marque a opção: **Automatically overwrite all conflicts**.
5. Clique em **Import**.
6. Acesse o menu **Dashboard** -> Selecione **Threat Monitoring Center - SOC Homelab**.

O painel carregará instantaneamente as 12 visualizações configuradas (KPIs de severidade, linha do tempo, matriz MITRE ATT&CK, ranking de IPs atacantes e incidentes por agente).

---

## 12. Fase 9: Troubleshooting e Resolução de Problemas Críticos

| Sintoma | Causa Raiz | Procedimento de Correção |
|---|---|---|
| **Wazuh Indexer travado em Read-Only** | Disco da VM-SIEM atingiu mais de 85% de uso (LVM alocou 50% na instalação). | `sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`<br>`sudo resize2fs /dev/ubuntu-vg/ubuntu-lv` |
| **Agente rejeitado: `Agent version must be lower or equal to manager`** | Tentativa de instalar pacote padrão do Wazuh sem fixar versão compatível. | `sudo apt install -y wazuh-agent=4.9.2-1` |
| **Agente Disconnected após suspensão de VM** | Timeouts de rede e perda do handshake UDP/TCP após resume. | `sudo systemctl restart wazuh-agent` |
| **Erro ao habilitar regra RDP no Windows** | Sistema operacional em português (falha com nome em inglês de grupo). | `netsh advfirewall firewall add rule name="Allow RDP" dir=in action=allow protocol=TCP localport=3389` |
| **Suricata não alerta tráfego das outras VMs** | Interface `ens34` perdeu a flag `PROMISC` após reinício. | `sudo systemctl restart promisc-ens34.service`<br>Verificar com: `ip link show ens34` |
| **Dashboard exibe `Error Pattern Handler (getPatternList)`** | Extensão de tradução automática do Google no navegador corrompe strings JS. | Desativar a tradução para o site ou usar janela em modo anônimo (`Ctrl+Shift+N`). |
| **Dashboard bloqueado com `AxiosError: Network Error`** | Antivírus host (Kaspersky Web Protection) interceptando payloads nos alertas. | Adicionar `10.0.1.50` às exclusões de escaneamento web do antivírus. |
| **Consumo excessivo de espaço por vulnerabilidades** | Cache corrompido em `/var/ossec/queue/vd_updater/`. | `sudo systemctl stop wazuh-manager`<br>`sudo rm -rf /var/ossec/queue/vd_updater/tmp/*`<br>`sudo systemctl start wazuh-manager` |

---

## 13. Fase 10: Ordem Operacional de Inicialização e Desligamento

Para evitar corrupção de índices OpenSearch e perda de telemetria durante o início dos testes, siga rigorosamente a ordem:

### Ordem de Inicialização (Boot Sequence):
1. **1º Iniciar `vm-siem`:** Aguarde 2 minutos até que o Indexer (9200) e o Manager (1514) estejam totalmente operacionais.
2. **2º Iniciar `vm-sensor`:** Garante que o serviço de promisc e o Suricata subam e conectem o agente ao Manager.
3. **3º Iniciar Vítimas (`vm-vitima` e `vm-vitima-windows`):** Os serviços de endpoint sobem e registram no SIEM.
4. **4º Iniciar `vm-atacante`:** Pronta para executar os scripts ofensivos.

### Ordem de Desligamento (Graceful Shutdown):
```bash
# Na vm-atacante, vm-vitima, vm-vitima-windows, vm-sensor:
sudo poweroff

# Na vm-siem (aguardar o encerramento correto do OpenSearch):
sudo systemctl stop wazuh-dashboard
sudo systemctl stop wazuh-manager
sudo systemctl stop wazuh-indexer
sudo poweroff
```
