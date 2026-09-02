# Troubleshooting

Problemas reais encontrados durante a montagem e cenários comuns.

| Problema | Causa | Solução |
|---|---|---|
| **LVM usando só 50% do disco** | Particionamento padrão do Ubuntu Server | `lvextend -l +100%FREE` + `resize2fs` |
| **Agent não registra no Manager** | Versão do Agent > Manager | Instalar versão fixa: `wazuh-agent=4.9.2-1` |
| **Agent desconecta após pausa da VM** | Heartbeat expira com VM suspensa | `sudo systemctl restart wazuh-agent` |
| **Firewall do Windows falha por idioma** | Grupo "Remote Desktop" em PT-BR vira outro nome | Usar `netsh` pela porta: `localport=3389` |
| **Clipboard não funciona na VM** | Falta VMware Tools | Instalar `open-vm-tools` ou VMware Tools pelo menu |
| **Dashboard inacessível após reboot** | Serviço falhou por falta de memória | `systemctl status wazuh-dashboard` — garantir 4+ GB RAM |
| **Suricata sem alertas no Wazuh** | Interface sem modo promíscuo ou eve.json não configurado | Validar `ip a` (PROMISC) e `<localfile>` no ossec.conf |
