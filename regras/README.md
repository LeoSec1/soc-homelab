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
