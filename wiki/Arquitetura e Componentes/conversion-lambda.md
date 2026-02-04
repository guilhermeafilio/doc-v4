# Postback Receiver (AWS Lambda)

## Visão Geral

O **Postback Receiver Lambda** é o ponto de entrada externo para eventos de conversão.

Ele atua como uma **camada de ingestão desacoplada**, responsável por:

- Receber eventos de conversão dos anunciantes
- Validar e normalizar dados
- Aplicar regras de segurança e rate limit
- Enfileirar eventos no SQS
- Retornar resposta rápida ao anunciante

> O Lambda **não executa lógica de negócio pesada**. Seu papel é ingestão, validação e entrega confiável para o core.

---

## Responsabilidade Arquitetural

### O Lambda DEVE:

- Receber postbacks HTTP externos
- Validar assinatura e parâmetros obrigatórios
- Normalizar payload para o contrato interno
- Enviar evento bruto para SQS
- Retornar HTTP status adequado
- Garantir baixa latência (<100ms)

---

### O Lambda NÃO DEVE:

- Acessar banco de dados
- Calcular comissão
- Resolver atribuição
- Alterar status de conversão
- Executar regras de negócio

Essas responsabilidades pertencem ao **Worker**.
