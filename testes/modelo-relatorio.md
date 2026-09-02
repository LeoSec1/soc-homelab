# Relatório de Teste: [ID] — [Nome do Cenário]

## Informações

* **ID:** [Ex: TST-01]
* **Data:** [YYYY-MM-DD]
* **Rede:** `10.0.1.0/24` (VMnet2, isolada)
* **Objetivo:** [Descrição do teste]

## MITRE ATT&CK

* **Técnica:** [ID e Nome]
* **Justificativa:** [Por que este ataque se associa a esta técnica]

## Topologia

* **Atacante:** `10.0.1.100` (Kali)
* **Alvo:** `[IP]`
* **Serviço/Porta:** `[Protocolo e Porta]`

## Regras Relacionadas

* **Suricata SID:** [Número]
* **Wazuh Rule ID:** [Número]

## Evidências

### Terminal do Atacante

[Inserir screenshot em imagens/evidencias/]

### Alerta Suricata (eve.json)

```json
[Inserir snippet JSON]
```

### Dashboard Wazuh

[Inserir screenshot em imagens/dashboard/]

## Resultados

* **Esperado:** [O que deveria alertar]
* **Observado:** [O que aconteceu]
* **Severidade:** [Nível no Wazuh]

## Conclusão

* **Análise:** [Eficácia da regra, falsos positivos, etc.]
* **Próximas ações:** [Ajustar threshold, melhorar regra, etc.]
