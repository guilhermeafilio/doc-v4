# Contrato de Evento: `postback_received`

Origem: `contrato-conversion-lambda.json`.

## Quando acontece

Emitido quando a plataforma recebe um postback de conversão do anunciante.

## Quem produz / quem consome

- **Produtor**: Receiver (AWS Lambda)
- **Consumidor**: Worker (via SQS) para validação, criação/atualização de conversão e cálculo de comissões

## Payload (exemplo)

```json
{
  "event": "postback_received",
  "click_id": "id_vindo_do_anunciante",
  "external_order_id": "ordem_12345",
  "gross_value": 150.0,
  "currency": "BRL",
  "raw_payload": {
    "full_query_string": "click_id=abc&amount=150...",
    "source_ip": "ip-do-anunciante"
  },
  "received_at": "2023-10-27T10:05:00Z"
}
```
