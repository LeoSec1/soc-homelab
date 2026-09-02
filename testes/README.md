# Cenários de Teste

Todos os testes são executados exclusivamente na rede isolada `10.0.1.0/24` (VMnet2).

## Cenários

| ID | Cenário | Técnica MITRE | Alvo | Status |
|---|---|---|---|---|
| TST-01 | Varredura SYN (Nmap) | T1046 | VM-Vítima Linux | Pendente |
| TST-02 | Força bruta SSH (Hydra) | T1110.001 | VM-Vítima Linux | Pendente |
| TST-03 | Shellshock via HTTP (cURL) | T1190 | VM-Vítima Linux | Pendente |
| TST-04 | Força bruta RDP (Hydra) | T1110.001 | VM-Vítima Windows | Pendente |
| TST-05 | Login FTP anônimo | — | VM-Vítima Linux | Pendente |
| TST-06 | ICMP anômalo (Hping3) | T1095 | VM-Vítima Linux | Pendente |

Use o template [`modelo-relatorio.md`](modelo-relatorio.md) para documentar cada teste.
