# Contrato de Evento: `click_logged`

Origem: `contrato-track-octan.json`.

## Quando acontece

Emitido quando um clique é registrado (Tracker).

## Quem produz / quem consome

- **Produtor**: Tracker (Laravel/Octane)
- **Consumidor**: processamento assíncrono (fila/job) e persistência/analytics (conforme arquitetura)

## Payload (contrato)

```json
{
  "event": "click_logged",
  "click_id": "uuid-v4-gerado-pelo-laravel",
  "enterprise_id": "uuid-da-empresa-dona-do-link",
  "link_id": "uuid-do-link-shortener",
  "campaign_id": "uuid-da-campanha",
  "metadata": {
    "ip": "192.168.1.1",
    "user_agent": "Mozilla/5.0...",
    "referer": "https://facebook.com",
    "utm_source": "social",
    "custom_params": {
      "subid1": "afiliado_x",
      "source": "banner_topo"
    }
  },
  "timestamp": "2026-01-27T10:00:00Z"
}
```
