# ADR-001: Padrão Outbox no MySQL

## Status
Aceito

## Contexto

O sistema precisa notificar clientes B2B quando o status de pedidos muda. A mudança de status já executa uma transação pesada que atualiza a tabela `orders`, insere em `order_status_history` e decrementa `stock_quantity` dos produtos ([09:04] Bruno).

Três alternativas foram avaliadas:
1. **Disparo síncrono**: fazer HTTP call para o webhook do cliente dentro da mesma transação
2. **Fila externa**: usar Redis Streams ou mensageria dedicada
3. **Padrão Outbox**: inserir evento em tabela MySQL dentro da transação existente

## Decisão

Implementar o **padrão Outbox no MySQL** ([09:08] Larissa).

Quando o status do pedido mudar, dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, inserimos uma linha na tabela `webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP ([09:06] Diego).

## Alternativas Consideradas

### Alternativa 1: Disparo síncrono do webhook
- **Descartada** por Bruno e Diego ([09:04], [09:06])
- **Problemas identificados**:
  - Cliente lento travaria a transação e bloquearia mudanças de status de outros pedidos ([09:04] Bruno)
  - Se cliente estiver offline, forçaria rollback da mudança de status, o que não faz sentido do ponto de vista de negócio ([09:04] Bruno)
  - Acoplamento forte entre disponibilidade do cliente e operação crítica do sistema

### Alternativa 2: Redis Streams ou fila externa
- **Considerada** por Larissa ([09:07])
- **Descartada** por Diego ([09:07])
- **Razão**: Time pequeno, subir Redis Cluster para isso seria overengineering
- **Trade-off**: Simplicidade operacional vs. possível latência ligeiramente menor

## Consequências

### Positivas
- **Garantia de atomicidade**: Se a transação commita, o evento foi registrado; se rollback, o evento some junto. Não há inconsistência possível ([09:06] Diego)
- **Sem infraestrutura adicional**: Usa o MySQL já existente, sem necessidade de Redis Cluster ou mensageria dedicada ([09:07] Diego)
- **Desacoplamento**: Mudança de status não depende da disponibilidade do cliente
- **Simplicidade operacional**: Menos componentes para gerenciar, adequado para time pequeno

### Negativas
- **Throughput limitado**: MySQL não é otimizado para filas de alta vazão como Redis Streams
- **Latência de polling**: Worker precisa fazer polling periódico em vez de ser notificado (ver ADR-005)
- **Crescimento da tabela**: Necessário processo de arquivamento periódico (fora de escopo da feature, [09:08] Diego)

### Riscos e Mitigação
- **Risco**: Acúmulo de eventos pode degradar performance
  - **Mitigação**: Índices em `status` e `created_at`; worker processa em batches; arquivamento após 30 dias ([09:08] Diego)

## Integração com Código Existente

A implementação exige modificação em `src/modules/orders/order.service.ts`, método `changeStatus` (linha ~126):

```typescript
// Dentro da transação existente em prisma.$transaction
await publishWebhookEvent(tx, order, from, to);
```

Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o transaction client e insere na `webhook_outbox` ([09:41] Bruno, [09:41] Diego).

## Referências
- Transcrição: [09:04] Bruno, [09:06] Diego, [09:07] Larissa, [09:07] Diego, [09:08] Larissa
- Código: `src/modules/orders/order.service.ts` (transação existente em changeStatus)
