# Enums e Índices (Banco de Dados)

Origem: `tech-doc.md`.

## Mapeamento de Enums

Os campos abaixo utilizam representação numérica para otimização de armazenamento e performance.

### CONVERSIONS (status)

> Nota: o documento original tem IDs duplicados/inconsistentes. Mantivemos o conteúdo como referência e recomenda-se revisar/normalizar o enum antes de usar como contrato.

| ID | Valor | Descrição |
|:---|:---|:---|
| 1 | `RECEIVED` | Recebido do anunciante/importado. |
| 2 | `PENDING` | Aguardando processamento/validação. |
| 2 | `APPROVED` | Conversão validada com sucesso. |
| 3 | `READY_TO_PAY` | Pronto para ser pago. |
| 4 | `PAID` | Comissão já paga ao beneficiário. |
| 5 | `REJECTED` | Conversão negada (regras de negócio ou fraude). |
| 6 | `CANCELLED` | Cancelada pelo cliente final. |
| 7 | `CHARGEBACK` | Revertido depois de pago. |

---

## Estratégia de Indexação

Índices configurados para suportar alta carga de leitura e filtragem em relatórios.

### Tabela: `LinkShortener`

- **UNIQUE(`url`)**: garante unicidade do link encurtado e lookup rápido no redirecionamento.
- **INDEX(`link_type_id`)**: utilizado para filtros e regras por tipo de link.

### Tabela: `Link_Campaign_Channel_Creative_Associate`

- **INDEX(`link_shortener_id`)**: utilizado na resolução do redirecionamento em tempo real.
- **INDEX(`campaign_id`)**: otimização para relatórios e filtros por campanha.
- **INDEX(`creative_id`)**: utilizado para análises de performance por criativo.
- **INDEX(`link_shortener_id`, `campaign_id`, `channel_id`, `creative_id`)**: índice composto para resolução completa do vínculo link–campanha–canal–criativo.

### Tabela: `RedirectLinkShortener`

- **UNIQUE(`click_id`)**: identificação única do clique para deduplicação e rastreamento de conversão.
- **INDEX(`link_shortener_id`)**: utilizado para lookup direto de cliques via link.
- **INDEX(`clicked_at`)**: otimização para filtros de período e agrupamentos temporais.
- **INDEX(`affiliate_id`, `campaign_id`)**: índice composto para consultas de performance de afiliados dentro de campanhas específicas.
- **INDEX(`channel_id`, `campaign_id`)**: utilizado para análises de performance por canal dentro da campanha.
- **INDEX(`campaign_id`, `clicked_at`)**: otimização para séries temporais de cliques por campanha.

### Tabela: `CONVERSION`

- **UNIQUE(`click_id`)**: garante uma única conversão por clique.
- **UNIQUE(`external_order_id`)**: utilizado para deduplicação e conciliação com sistemas externos.
- **INDEX(`conversion_timestamp`)**: otimização para relatórios temporais e agrupamentos por período.
- **INDEX(`campaign_id`, `conversion_timestamp`)**: utilizado para análises de performance de campanhas ao longo do tempo.
- **INDEX(`channel_id`, `conversion_timestamp`)**: utilizado para análises de performance por canal.
- **INDEX(`affiliate_id`, `conversion_timestamp`)**: utilizado para relatórios de ganhos e performance por afiliado.

### Tabela: `CONVERSION_STATUS`

- **INDEX(`conversion_id`)**: utilizado para recuperação rápida do histórico de status de uma conversão.
- **INDEX(`status`)**: utilizado para consultas operacionais e filas de processamento por estado (pendente, aprovado, pago).
- **INDEX(`is_current`)**: utilizado para localizar rapidamente o status ativo da conversão.
- **UNIQUE(`conversion_id`, `is_current`)**: garante que exista apenas um status ativo por conversão.
- **INDEX(`status`, `changed_at`)**: utilizado para relatórios temporais e monitoramento do fluxo operacional.
- **INDEX(`changed_at`)**: utilizado para auditoria e análise de SLA de processamento.
