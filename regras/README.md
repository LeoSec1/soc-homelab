# Regras Customizadas

Regras escritas para o ambiente do laboratório.

* **Suricata:** 6 regras de assinatura de rede. Arquivo: `suricata-custom.rules`.
* **Wazuh:** 7 regras de correlação. Arquivo: `wazuh-local-rules.xml`.

Os SIDs `1000006` e `1000007` estão reservados e não ativos na versão atual.

## Mapeamento Suricata → Wazuh

| Suricata SID | Wazuh ID | Descrição | Severidade | MITRE |
|---|---|---|---|---|
| 1000001 | 100010 | Varredura SYN detectada | Nível 8 | T1046 |
| 1000002 | 100020 | Tentativa de brute force SSH | Nível 10 | T1110 |
| 1000003 | 100030 | ICMP com payload anômalo | Nível 6 | T1095 |
| 1000004 | 100040 | Login anônimo FTP | Nível 5 | — |
| 1000005 | 100050 | Tentativa de Shellshock | Nível 14 | T1190 |
| 1000008 | — | Varredura de serviço (Nmap -sV) | — | T1046 |
| — | 100001 | Alerta genérico do Suricata | Nível 3 | — |
| — | 100021 | Correlação: Scan + Brute Force (mesmo IP, 10 min) | Nível 14 | T1046 + T1110 |

---

## Validação e Teste de Sintaxe

Antes de reiniciar os serviços em produção ou laboratório, sempre valide a sintaxe das regras:

### 1. Testar Regras do Suricata
Na **VM-Sensor** (`10.0.1.10`), execute o modo de teste para verificar se o arquivo não possui erros de parsing:
```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```
> O retorno deve conter: `Suricata Configuration Successfully Validated!`.

### 2. Testar Regras do Wazuh
Na **VM-SIEM** (`10.0.1.50`), use o utilitário interativo `wazuh-logtest` para simular a ingestão de um log e verificar se a regra esperada dispara:
```bash
/var/ossec/bin/wazuh-logtest
```
Exemplo de log para testar a regra `100010` (Varredura SYN):
```json
{"timestamp":"2026-09-07T00:00:00.000Z","event_type":"alert","src_ip":"10.0.1.135","dest_ip":"10.0.1.200","alert":{"signature_id":1000001,"signature":"SOC-HOMELAB - Varredura SYN Detectada"}}
```
