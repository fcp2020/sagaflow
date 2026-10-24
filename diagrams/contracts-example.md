# Exemplo de contrato de mensagem

```json
{
  "messageId": "msg-uuid",
  "messageType": "ProcessPayment",
  "schemaVersion": 1,
  "sagaId": "saga-uuid",
  "correlationId": "checkout-uuid",
  "orderId": "order-uuid",
  "idempotencyKey": "payment-order-uuid",
  "occurredAt": "2026-10-23T20:00:00Z",
  "payload": {
    "amount": 149.90,
    "currency": "BRL",
    "paymentMethodToken": "tokenizado-pelo-provedor"
  }
}
```

Não inclua número completo do cartão, CVV ou outros dados sensíveis. Os nomes e campos são uma proposta de contrato, não uma implementação vinculada a um broker específico.
