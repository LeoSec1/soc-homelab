# Cenários de Teste

Todos os testes são executados exclusivamente na rede isolada `10.0.1.0/24` (VMnet2).

## Cenários

| ID | Cenário | Técnica MITRE | Alvo | Status |
|---|---|---|---|---|
| TST-01 | Varredura SYN (Nmap) | T1046 | VM-Vítima Linux (`10.0.1.200`) | [✅ Validado](01-varredura-nmap.md) |
| TST-02 | Força bruta SSH (Hydra) | T1110.001 | VM-Vítima Linux (`10.0.1.200`) | [✅ Validado](02-forca-bruta-ssh.md) |
| TST-03 | Shellshock via HTTP (cURL) | T1190 | VM-Vítima Linux (`10.0.1.200`) | [✅ Validado](03-exploit-shellshock.md) |
| TST-04 | Força bruta RDP (Hydra) | T1110.001 | VM-Vítima Windows (`10.0.1.201`) | [✅ Validado](04-forca-bruta-rdp.md) |
| TST-05 | Login FTP anônimo | — | VM-Vítima Linux (`10.0.1.200`) | [✅ Validado](05-login-ftp-anonimo.md) |
| TST-06 | ICMP anômalo (Hping3) | T1095 | VM-Vítima Linux (`10.0.1.200`) | [✅ Validado](06-icmp-anomalo.md) |
| TST-07 | Cadeia de ataque (Scan + Brute Force) | T1046 + T1110.001 | VM-Vítima Linux (`10.0.1.200`) | [✅ Validado](07-cadeia-de-ataque.md) |

Use o template [`modelo-relatorio.md`](modelo-relatorio.md) para documentar cada teste.
