# Segurança

Este repositório é um ambiente educacional de laboratório.

## IPs

Todos os endereços (`10.0.1.0/24`) são de rede privada RFC 1918, configurados em switch virtual isolado (host-only).

## Credenciais

Não submeta senhas, chaves privadas ou tokens para este repositório. O `.gitignore` bloqueia `.env`, `.pem`, `.key` e `passwords.txt`.

## Vazamentos e Reportes

Se encontrar dados sensíveis commitados, contate o autor diretamente em vez de abrir uma Issue pública. Credenciais expostas serão rotacionadas imediatamente.

## Simulações de Ataque e Artefatos de Teste

Os comandos, scripts ofensivos e payloads documentados no diretório `testes/` (ex: cURL Shellshock, Hydra, Hping3) são **estritamente didáticos**, executados exclusivamente em rede local isolada (VMnet2) para validação das regras do NIDS e SIEM. Não devem ser reportados como vulnerabilidades de segurança deste repositório.
