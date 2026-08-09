# ADR-004: Garantia At-Least-Once com X-Event-Id para Deduplicação

## Status
Aceito

## Contexto

Webhooks podem ser entregues múltiplas vezes ao cliente devido a:
- Retries após timeout de rede
- Falha parcial (request enviado mas response não recebida)
- Replay manual via DLQ

O sistema precisa definir a semântica de entrega e fornecer mecanismo para o cliente lidar com duplicatas.

Duas abordagens principais existem:
1. **Exactly-once**: garantia de que evento é entregue uma única vez
2. **At-least-once**: evento pode ser entregue múltiplas vezes, cliente deduplica

## Decisão

Implementar garantia **at-least-once** ([09:24] Diego, [09:26] Larissa).

O cliente pode receber o mesmo evento duas (ou mais) vezes. Para permitir deduplicação, enviamos header `X-Event-Id` com um **UUID único** gerado quando o evento entra na outbox ([09:25] Diego).

Cliente usa o `X-Event-Id` para dedupuplicar do lado dele ([09:25] Diego).

## Alternativas Consideradas

### Alternativa: Exactly-once delivery
- **Descartada** por Diego ([09:25])
- **Razão**: Exigiria coordenação distribuída entre nosso sistema e o cliente (acks bidirecionais, estado compartilhado, etc). Complexidade muito maior para benefício marginal.
- **Trade-off rejeitado**: Simplicidade vs. garantia teórica de entrega única

## Consequências

### Positivas
- **Simplicidade arquitetural**: Não requer protocolo complexo de confirmação bidirecional
- **Padrão de mercado**: Stripe, GitHub, e outros webhooks grandes operam em at-least-once ([09:25] Diego)
- **Resiliência**: Podemos fazer retry agressivo sem medo de criar duplicatas irrecuperáveis
- **Compatibilidade com DLQ**: Replay manual funciona naturalmente

### Negativas
- **Responsabilidade delegada ao cliente**: Cliente **precisa** implementar deduplicação ([09:25] Sofia: "Isso joga responsabilidade pro cliente")
- **Risco de processamento duplo**: Se cliente não deduplica corretamente, pode processar o mesmo evento mais de uma vez

### Riscos e Mitigação
- **Risco**: Cliente implementa deduplicação incorretamente ou não implementa
  - **Mitigação**: Documentação clara e destacada no portal de desenvolvedor ([09:26] Marcos)
  - **Educação**: Exemplos de código mostrando deduplicação por `X-Event-Id`

## Detalhes de Implementação

### Geração do Event ID
- UUID v4 gerado **na inserção do evento na outbox** ([09:25] Diego)
- Armazenado na coluna `event_id` da tabela `webhook_outbox`
- **Importante**: Event ID é gerado UMA vez, no momento da mudança de status, não a cada retry

### Header X-Event-Id
- Enviado em todas as tentativas de entrega (incluindo retries)
- Formato: UUID v4 (ex: `550e8400-e29b-41d4-a716-446655440000`)
- Cliente deve indexar por esse campo para busca rápida de duplicatas

### Deduplicação Recomendada (para documentação do cliente)
```javascript
// Pseudocódigo da lógica recomendada
if (alreadyProcessed(event.headers['X-Event-Id'])) {
  return 200; // Já processado, retorna sucesso
}

processEvent(event.body);
markAsProcessed(event.headers['X-Event-Id']);
return 200;
```

### Interação com Outros Headers
- `X-Event-Id`: identificador único do evento (deduplicação)
- `X-Webhook-Id`: identificador do endpoint webhook cadastrado ([09:44] Sofia)
- `X-Signature`: assinatura HMAC (ver ADR-003)
- `X-Timestamp`: timestamp do envio (detecção de replay attack)

## Integração com Código Existente

Reutiliza:
- Biblioteca `uuid` já presente no `package.json`
- Mesmo padrão de UUIDs usado em todas as entidades (`@default(uuid())` no Prisma)

## Referências
- Transcrição: [09:24] Diego, [09:25] Diego, [09:25] Sofia, [09:25] Diego, [09:26] Marcos, [09:26] Larissa, [09:44] Sofia
- Código: Uso de UUID em `prisma/schema.prisma` (todas as entidades usam `@default(uuid())`)
