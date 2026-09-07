# Anotações e Problemas Encontrados

Registro dos problemas reais que enfrentei durante a montagem do laboratório e como resolvi cada um.

---

## 29/08/2026 — Montagem do SIEM e Sensor

**1. Disco cheio no Wazuh Indexer:**
* **Problema:** O instalador do Ubuntu Server usa LVM por padrão, mas aloca apenas metade do disco virtual (~24 GB de 50 GB). O Wazuh Indexer (OpenSearch) encheu o espaço e travou.
* **Solução:** Expandir o volume lógico LVM para usar 100% do disco livre:
```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

**2. Agente Wazuh não registrava no Manager:**
* **Problema:** Ao tentar instalar a versão mais recente do agente (4.14), o Manager 4.9 rejeitou o registro (`Agent version must be lower or equal to manager version`).
* **Solução:** Fixar a instalação do agente na versão 4.9.2:
```bash
sudo apt install -y wazuh-agent=4.9.2-1
```

**3. Agente aparecendo como Disconnected após pausa:**
* **Problema:** Ao suspender/desligar as VMs e retornar, o agente perdeu o heartbeat e foi marcado como desconectado pelo Manager.
* **Solução:** Reiniciar o serviço do agente para forçar novo handshake imediato:
```bash
sudo systemctl restart wazuh-agent
```

---

## 30/08/2026 — Configuração das Vítimas

**4. Firewall do Windows em português:**
* **Problema:** O comando PowerShell `Enable-NetFirewallRule -DisplayGroup "Remote Desktop"` retornou erro porque a interface do Windows está em português (onde o grupo se chama "Área de Trabalho Remota").
* **Solução:** Criar a regra de firewall diretamente pela porta TCP 3389 via `netsh`:
```powershell
netsh advfirewall firewall add rule name="Allow RDP" dir=in action=allow protocol=TCP localport=3389
```

**5. Copiar e Colar desabilitado na VM Windows:**
* **Problema:** O clipboard compartilhado entre o host Windows e a VM Windows não funcionava após a instalação.
* **Solução:** Instalar o VMware Tools na VM (Menu `VM` -> `Install VMware Tools...`) e reiniciar o guest.

---

## 06/09/2026 — Validação do Cenário Multi-Estágio (TST-07)

**6. Perda da flag PROMISC no reboot/pausa da VM-Sensor:**
* **Problema:** A interface de captura `ens34` do Suricata na VM-Sensor perdia a flag `PROMISC` após suspensão ou reboot da máquina virtual no VMware, fazendo o NIDS ignorar o tráfego que não fosse destinado ao seu próprio IP.
* **Solução:** Criar e habilitar um serviço systemd (`promisc-ens34.service`) para aplicar `ip link set ens34 promisc on` automaticamente a cada inicialização do sistema operacional.

**7. Esgotamento de disco por cache do Vulnerability Detector no Wazuh:**
* **Problema:** O diretório `/var/ossec/queue/vd_updater/` acumulou ~37 GB de downloads temporários de bases de vulnerabilidades devido à falta de conexão direta à internet na rede isolada, levando o disco a 100% de uso e ativando a proteção `read-only` do Wazuh Indexer (OpenSearch).
* **Solução:** Parar o serviço `wazuh-manager`, purgar os diretórios temporários `/var/ossec/queue/vd_updater/tmp` e `/var/ossec/queue/vd/feed`, e reiniciar os serviços `wazuh-indexer` e `wazuh-manager`, liberando mais de 34 GB imediatamente.

**8. Antivírus e tradução automática quebrando o Wazuh Dashboard:**
* **Problema:** Duas causas simultâneas impediam o uso normal do dashboard no navegador:
  1. **Kaspersky Web Protection** — intercepta conexões HTTPS ao IP `10.0.1.50` e bloqueia requisições cujas respostas contêm assinaturas de ataque (ex: payloads Suricata nos alertas), gerando erros `Request has been forbidden by antivirus` e `AxiosError: Network Error`.
  2. **Google Translate (tradução automática)** — ao traduzir a página do Wazuh Dashboard, corrompe strings JavaScript internas, causando `Error: Error Pattern Handler (getPatternList)` no plugin Wazuh.
* **Solução:** Usar o navegador em modo anônimo/incógnito (`Ctrl+Shift+N`) para desabilitar extensões do antivírus e a tradução automática. Alternativamente, adicionar `10.0.1.50` às exclusões do Kaspersky Web Protection e desativar a tradução automática para o site.
