# FDD: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0
**Data:** [Data da reunião técnica]
**Responsável Técnico:** Bruno (Engenheiro Pleno, Time de Pedidos)

---

## 1. Contexto e Motivação Técnica

O sistema atual não possui mecanismo de notificação externa quando status de pedidos mudam. Clientes fazem polling contínuo no endpoint `GET /orders`, gerando carga desnecessária e aumentando latência de integração.

A feature implementa webhooks outbound usando **padrão Outbox** integrado à transação existente de mudança de status (`OrderService.changeStatus` em `src/modules/orders/order.service.ts:126`). Worker assíncrono processa eventos e entrega via HTTP POST para endpoints configurados pelos clientes.

**Problema técnico resolvido:** Garantir que toda mudança de status gera evento de webhook de forma atômica, sem bloquear a transação principal e sem depender de infraestrutura externa adicional.

---

## 2. Objetivos Técnicos

- Garantir **atomicidade** entre mudança de status e registro do evento (se um commita, o outro também)
- Entregar eventos com latência **< 10 segundos** (p95) após mudança de status
- Processar eventos de forma **assíncrona** sem bloquear transação de `changeStatus`
- Suportar **retry automático** com backoff exponencial para lidar com falhas temporárias
- Prover **autenticação forte** (HMAC-SHA256) para validação de origem
- Manter **compatibilidade total** com padrões existentes do projeto (erros, logging, validação)

---

## 3. Escopo e Exclusões

### Incluído
- CRUD de configuração de webhooks (POST, GET, PATCH, DELETE)
- Inserção de eventos na outbox dentro da transação de `changeStatus`
- Worker em processo separado com polling a cada 2s
- Retry com backoff exponencial (5 tentativas)
- Dead Letter Queue (DLQ) para eventos que falharam definitivamente
- Endpoint admin para replay manual de DLQ
- Histórico de entregas (últimos 100 eventos por webhook)
- Autenticação HMAC-SHA256 com secret rotacionável
- Filtro de eventos por status (cliente escolhe quais status receber)
- Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`

### Excluído
- Email de fallback quando webhook falha ([09:37] Larissa: fora de escopo)
- Dashboard visual/UI ([09:40] Larissa: projeto separado frontend)
- Inbound webhooks (receber eventos de clientes)
- Rate limiting de envio por cliente ([09:39] Diego/Larissa: observar e decidir depois)
- Múltiplos workers (single worker no MVP, [09:13] Diego)
- Arquivamento automático de eventos entregues ([09:08] Diego: fora de escopo da feature)

---

## 4. Fluxos Detalhados

### Fluxo 1: Criação de Evento na Outbox

**Trigger:** Chamada a `OrderService.changeStatus(orderId, { toStatus, reason }, userId)`

**Passos:**
1. **Início da transação Prisma** (`prisma.$transaction`)
2. Validar transição de status (via `canTransition(from, to)` em `src/modules/orders/order.status.ts`)
3. Atualizar estoque se necessário (debit ou replenish)
4. **UPDATE** `orders` SET `status = to`
5. **INSERT** `order_status_history` (fromStatus, toStatus, changedById, reason)
6. **Buscar webhooks ativos** do customer que querem esse status:
   ```sql
   SELECT id, url, secret FROM webhook_configuration
   WHERE customer_id = :customerId
     AND active = true
     AND :toStatus IN (status_filter)
   ```
7. **Para cada webhook encontrado**:
   - Gerar `event_id` (UUID v4)
   - Renderizar payload JSON (snapshot do estado atual):
     ```json
     {
       "event_id": "<uuid>",
       "event_type": "order.status_changed",
       "timestamp": "<ISO 8601>",
       "order_id": "<id>",
       "order_number": "ORD-000042",
       "from_status": "PENDING",
       "to_status": "PAID",
       "customer_id": "<id>",
       "total_cents": 15990
     }
     ```
   - **INSERT** `webhook_outbox`:
     ```sql
     INSERT INTO webhook_outbox (
       id, event_id, webhook_id, order_id, payload, status, created_at
     ) VALUES (
       uuid(), :event_id, :webhook_id, :order_id, :payload, 'pendente', NOW()
     )
     ```
8. **COMMIT** da transação

**Exceções:**
- Se inserção na outbox falha → Rollback completo da transação
- Se nenhum webhook ativo do customer quer aquele status → Não insere evento (otimização, [09:34] Bruno)

**Invariantes:**
- Se `order.status` mudou → evento foi registrado
- Se evento foi registrado → `order.status` mudou
- **Não existe estado inconsistente**

---

### Fluxo 2: Processamento pelo Worker

**Trigger:** Loop infinito com polling a cada 2 segundos

**Passos:**
1. **SELECT** eventos pendentes:
   ```sql
   SELECT id, event_id, webhook_id, order_id, payload, attempt_count
   FROM webhook_outbox
   WHERE status = 'pendente'
     AND (next_retry_at IS NULL OR next_retry_at <= NOW())
   ORDER BY created_at ASC
   LIMIT 50
   ```
2. **Para cada evento no batch**:
   - **UPDATE** `webhook_outbox` SET `status = 'processando'` (lock otimista)
   - Buscar configuração do webhook:
     ```sql
     SELECT url, secret FROM webhook_configuration WHERE id = :webhook_id
     ```
   - Calcular `X-Signature`:
     ```javascript
     const signature = crypto
       .createHmac('sha256', secret)
       .update(JSON.stringify(payload))
       .digest('hex');
     ```
   - **POST HTTP** para `webhook.url`:
     - Headers:
       - `Content-Type: application/json`
       - `X-Event-Id: <event_id>`
       - `X-Signature: <signature>`
       - `X-Timestamp: <ISO 8601 atual>`
       - `X-Webhook-Id: <webhook_id>`
     - Body: `payload` (JSON)
     - Timeout: **10 segundos** ([09:42] Diego)

3. **Tratamento da resposta**:
   - **Sucesso** (status 2xx):
     - **UPDATE** `webhook_outbox` SET `status = 'entregue'`, `delivered_at = NOW()`
     - **INSERT** `webhook_delivery_log` (sucesso, http_status, response_time)

   - **Falha** (timeout, 4xx, 5xx, erro de rede):
     - Incrementar `attempt_count`
     - **Se `attempt_count < 5`**:
       - Calcular `next_retry_at` (backoff exponencial):
         - Attempt 1: NOW() + 1 minuto
         - Attempt 2: NOW() + 5 minutos
         - Attempt 3: NOW() + 30 minutos
         - Attempt 4: NOW() + 2 horas
         - Attempt 5: NOW() + 12 horas
       - **UPDATE** `webhook_outbox` SET `status = 'pendente'`, `next_retry_at = <calculado>`
       - **INSERT** `webhook_delivery_log` (falha, http_status, erro)

     - **Se `attempt_count >= 5`**:
       - **INSERT** `webhook_dead_letter` (event_id, payload, motivo, timestamp)
       - **UPDATE** `webhook_outbox` SET `status = 'failed'`
       - **INSERT** `webhook_delivery_log` (falha definitiva)

4. **Sleep 2 segundos**, volta ao passo 1

**Exceções:**
- Worker crash → Process manager reinicia (PM2, systemd)
- Banco indisponível → Worker loga erro e retenta no próximo ciclo

---

### Fluxo 3: Replay Manual de DLQ (Admin)

**Trigger:** `POST /admin/webhooks/dead-letter/:id/replay`

**Pré-condição:** Role ADMIN no JWT ([09:36] Sofia)

**Passos:**
1. Middleware `requireRole('ADMIN')` valida permissão
2. **SELECT** evento da DLQ: `SELECT * FROM webhook_dead_letter WHERE id = :id`
3. **INSERT** novo evento na outbox:
   ```sql
   INSERT INTO webhook_outbox (
     id, event_id, webhook_id, order_id, payload, status, created_at, attempt_count
   ) VALUES (
     uuid(), :event_id, :webhook_id, :order_id, :payload, 'pendente', NOW(), 0
   )
   ```
4. **INSERT** log de auditoria: `INSERT INTO audit_log (action, user_id, resource_type, resource_id)`
5. **Retornar 200 OK** com `{ message: "Evento reprocessado", outbox_id: "<id>" }`

**Exceções:**
- Evento DLQ não encontrado → 404 Not Found
- Usuário sem role ADMIN → 403 Forbidden

---

## 5. Contratos Públicos (Assinaturas, Endpoints, Headers, Exemplos)

### Endpoint 1: Criar Webhook

**Tipo:** HTTP REST Endpoint
**Rota:** `POST /api/v1/webhooks`
**Autenticação:** JWT (qualquer role autenticada)

**Request:**
```json
{
  "customer_id": "abc123...",
  "url": "https://cliente.com/webhooks/orders",
  "status_filter": ["PAID", "SHIPPED", "DELIVERED"]
}
```

**Response (201 Created):**
```json
{
  "id": "webhook-uuid...",
  "customer_id": "abc123...",
  "url": "https://cliente.com/webhooks/orders",
  "secret": "wh_sec_a1b2c3d4e5f6...",
  "status_filter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "created_at": "2024-11-15T10:00:00.000Z"
}
```

**Validações:**
- `url` deve ser HTTPS (Zod valida, [09:23] Sofia)
- `status_filter` deve conter apenas status válidos do enum `OrderStatus`

**Códigos de erro:**
- `400 VALIDATION_ERROR`: URL inválida ou não-HTTPS
- `401 UNAUTHORIZED`: JWT inválido
- `422 WEBHOOK_INVALID_URL`: URL malformada

---

### Endpoint 2: Listar Webhooks

**Rota:** `GET /api/v1/webhooks?customer_id=<id>`
**Autenticação:** JWT

**Response (200 OK):**
```json
{
  "items": [
    {
      "id": "webhook-uuid...",
      "customer_id": "abc123...",
      "url": "https://cliente.com/webhooks/orders",
      "status_filter": ["PAID", "SHIPPED"],
      "active": true,
      "created_at": "2024-11-15T10:00:00.000Z"
    }
  ],
  "total": 1,
  "page": 1,
  "page_size": 25
}
```

**Nota:** Secret **NÃO** é retornado em listagens (apenas na criação)

---

### Endpoint 3: Atualizar Webhook

**Rota:** `PATCH /api/v1/webhooks/:id`
**Autenticação:** JWT

**Request:**
```json
{
  "url": "https://cliente.com/novo-endpoint",
  "status_filter": ["DELIVERED"],
  "active": false
}
```

**Response (200 OK):** Objeto webhook atualizado (sem `secret`)

---

### Endpoint 4: Remover Webhook

**Rota:** `DELETE /api/v1/webhooks/:id`
**Autenticação:** JWT

**Response (204 No Content)**

---

### Endpoint 5: Histórico de Entregas

**Rota:** `GET /api/v1/webhooks/:id/deliveries?page=1&page_size=100`
**Autenticação:** JWT

**Response (200 OK):**
```json
{
  "items": [
    {
      "id": "delivery-uuid...",
      "event_id": "550e8400-e29b-41d4-a716-446655440000",
      "attempt_number": 1,
      "http_status": 200,
      "response_time_ms": 45,
      "delivered_at": "2024-11-15T10:05:23.000Z",
      "status": "success"
    },
    {
      "id": "delivery-uuid2...",
      "event_id": "650e8400-e29b-41d4-a716-446655440001",
      "attempt_number": 3,
      "http_status": 503,
      "response_time_ms": 10000,
      "delivered_at": null,
      "status": "failed",
      "error": "Timeout after 10s"
    }
  ],
  "total": 87,
  "page": 1,
  "page_size": 100
}
```

---

### Endpoint 6: Rotacionar Secret

**Rota:** `POST /api/v1/webhooks/:id/rotate-secret`
**Autenticação:** JWT

**Request:** (vazio)

**Response (200 OK):**
```json
{
  "new_secret": "wh_sec_z9y8x7w6...",
  "old_secret_expires_at": "2024-11-16T10:00:00.000Z"
}
```

**Comportamento:** Secret antiga válida por 24h ([09:21] Sofia)

---

### Endpoint 7: Replay de DLQ (Admin Only)

**Rota:** `POST /admin/webhooks/dead-letter/:id/replay`
**Autenticação:** JWT com role ADMIN

**Request:** (vazio)

**Response (200 OK):**
```json
{
  "message": "Evento reprocessado com sucesso",
  "outbox_id": "new-outbox-uuid..."
}
```

**Códigos de erro:**
- `403 FORBIDDEN`: Usuário não é ADMIN
- `404 WEBHOOK_DLQ_NOT_FOUND`: Evento DLQ não existe

---

### Endpoint 8: Webhook Recebido pelo Cliente (Outbound)

**Rota:** (configurada pelo cliente)
**Método:** `POST`
**Enviado por:** Worker do sistema

**Headers:**
- `Content-Type: application/json`
- `X-Event-Id: 550e8400-e29b-41d4-a716-446655440000`
- `X-Signature: a1b2c3d4e5f6...` (HMAC-SHA256)
- `X-Timestamp: 2024-11-15T10:05:20.000Z`
- `X-Webhook-Id: webhook-uuid...`

**Payload:**
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "order.status_changed",
  "timestamp": "2024-11-15T10:05:18.123Z",
  "order_id": "order-uuid...",
  "order_number": "ORD-000042",
  "from_status": "PENDING",
  "to_status": "PAID",
  "customer_id": "customer-uuid...",
  "total_cents": 15990
}
```

**Respostas esperadas:**
- `200-299`: Sucesso (evento marcado como entregue)
- `4xx, 5xx, timeout`: Falha (retry conforme política)

**Validação HMAC (cliente):**
```javascript
const expectedSignature = crypto
  .createHmac('sha256', secret)
  .update(rawBody) // corpo JSON exato recebido
  .digest('hex');

if (expectedSignature !== receivedSignature) {
  return 401; // Assinatura inválida
}
```

---

## 6. Erros, Exceções e Fallback

### Matriz de Erros

| Código | HTTP | Condição | Tratamento |
|--------|------|----------|------------|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook não existe | Retornar 404 |
| `WEBHOOK_INVALID_URL` | 422 | URL não é HTTPS ou malformada | Rejeitar na validação Zod |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Secret vazia ou nula | Rejeitar na validação |
| `WEBHOOK_DELIVERY_FAILED` | - | Cliente retorna 4xx/5xx | Retry conforme política |
| `WEBHOOK_DELIVERY_TIMEOUT` | - | Cliente não responde em 10s | Retry conforme política |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | Evento DLQ não existe para replay | Retornar 404 |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 413 | Payload > 64KB | Rejeitar evento, logar erro crítico |

### Estratégias de Resiliência

**Timeouts:**
- HTTP call para cliente: **10 segundos** ([09:42] Diego)
- Conexão MySQL: padrão Prisma (5s)

**Retries:**
- Política: 5 tentativas com backoff exponencial
- Progressão: 1m → 5m → 30m → 2h → 12h ([09:17] Diego)
- Total: ~15 horas de janela

**Circuit Breaker:**
- Não implementado no MVP (observar necessidade)

**Fallback:**
- Após 5 tentativas → DLQ
- Cliente pode requisitar replay manual

### Invariantes

- Nenhum evento entregue com sucesso é reprocessado (idempotência por `X-Event-Id`)
- Eventos em DLQ nunca são deletados automaticamente
- Payload renderizado no momento da inserção (snapshot, [09:52] Larissa)

---

## 7. Observabilidade

### Métricas (Prometheus)

```
webhook_events_created_total{customer_id, status_transition}
webhook_delivery_attempts_total{status="success|failure|retry", webhook_id}
webhook_delivery_latency_seconds{webhook_id, percentile="p50|p95|p99"}
webhook_dlq_events_total{customer_id}
webhook_worker_last_processed_timestamp
webhook_worker_batch_size{percentile="p50|p95"}
webhook_retry_backoff_seconds{attempt_number}
```

### Logs (Pino)

**Formato:** JSON estruturado

**Campos essenciais:**
```json
{
  "timestamp": "2024-11-15T10:05:20.123Z",
  "level": "info",
  "service": "order-management-api",
  "component": "webhook-worker",
  "event_id": "550e8400...",
  "order_id": "order-uuid...",
  "webhook_id": "webhook-uuid...",
  "customer_id": "customer-uuid...",
  "attempt_number": 2,
  "http_status": 503,
  "latency_ms": 10000,
  "action": "delivery_attempt",
  "result": "retry_scheduled",
  "next_retry_at": "2024-11-15T10:10:20.000Z"
}
```

**Redação automática:** `secret`, `payload` (se contiver PII)

### Tracing (OpenTelemetry)

**Spans principais:**
- `order.change_status` (origem)
  - `webhook.insert_outbox_event` (dentro da transação)
- `webhook.worker.poll` (ciclo do worker)
  - `webhook.worker.process_event`
    - `webhook.http_call` (chamada para cliente)

**Sampling:** 100% para eventos em retry, 10% para sucesso de primeira

### Dashboards e Alertas

**Painel mínimo:**
- Taxa de eventos criados (por minuto)
- Taxa de sucesso vs. falha de entregas
- Latência p95/p99 de entregas
- Tamanho da DLQ ao longo do tempo
- Última execução do worker

**Alertas:**
- Worker parado (> 5 min sem processar)
- Taxa de falha > 10% em 15 minutos
- DLQ > 100 eventos
- Latência p95 > 30s

---

## 8. Dependências e Compatibilidade

| Componente | Versão Mínima | Observações |
|------------|---------------|-------------|
| Node.js | 20.x | Mesma versão do projeto |
| MySQL | 8.0 | Já em uso |
| Prisma ORM | 5.22.0 | Já no package.json |
| TypeScript | 5.6.3 | Já no package.json |
| Biblioteca `uuid` | 11.0.3 | Já no package.json |
| Biblioteca `crypto` | Built-in Node | Para HMAC-SHA256 |

**Garantias de compatibilidade:**
- Mudanças no schema da outbox são backwards-compatible (apenas adicionar colunas)
- Payload JSON segue versionamento implícito (campo `event_type`)
- Headers seguem padrão HTTP standard (X-* custom headers)

---

## 9. Integração com o Sistema Existente

### 9.1. `src/modules/orders/order.service.ts`

**Linha:** ~126 (método `changeStatus`)

**Modificação:**
```typescript
async changeStatus(
  id: string,
  input: UpdateOrderStatusInput,
  userId: string
): Promise<OrderWithRelations> {
  return this.prisma.$transaction(async (tx) => {
    // ... código existente ...

    await tx.order.update({ where: { id }, data: { status: to } });
    await tx.orderStatusHistory.create({ ... });

    // 🆕 NOVA INTEGRAÇÃO
    await publishWebhookEvent(tx, {
      orderId: id,
      orderNumber: order.orderNumber,
      fromStatus: from,
      toStatus: to,
      customerId: order.customerId,
      totalCents: order.totalCents
    });

    // ... refresh e retorno ...
  });
}
```

**Função `publishWebhookEvent`:**
```typescript
async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  data: WebhookEventData
): Promise<void> {
  const webhooks = await tx.webhookConfiguration.findMany({
    where: {
      customerId: data.customerId,
      active: true,
      statusFilter: { has: data.toStatus }
    }
  });

  for (const webhook of webhooks) {
    const eventId = randomUUID();
    const payload = {
      event_id: eventId,
      event_type: 'order.status_changed',
      timestamp: new Date().toISOString(),
      order_id: data.orderId,
      order_number: data.orderNumber,
      from_status: data.fromStatus,
      to_status: data.toStatus,
      customer_id: data.customerId,
      total_cents: data.totalCents
    };

    await tx.webhookOutbox.create({
      data: {
        id: randomUUID(),
        eventId,
        webhookId: webhook.id,
        orderId: data.orderId,
        payload,
        status: 'pendente',
        attemptCount: 0
      }
    });
  }
}
```

**Impacto:** Transação aumenta em ~N queries (N = número de webhooks ativos do cliente). Tipicamente N < 5.

---

### 9.2. `src/modules/orders/order.status.ts`

**Linha:** Todo o arquivo (máquina de estados)

**Integração:** Reutiliza `canTransition(from, to)` para validar transições antes de criar evento

**Exemplo:**
```typescript
import { canTransition } from '@/modules/orders/order.status';

if (!canTransition(data.fromStatus, data.toStatus)) {
  throw new InvalidStatusTransitionError(data.fromStatus, data.toStatus);
}
```

---

### 9.3. `src/shared/errors/http-errors.ts`

**Linha:** Classes de erro

**Integração:** Criar novas classes seguindo mesmo padrão:

```typescript
export class WebhookNotFoundError extends AppError {
  constructor() {
    super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND');
  }
}

export class WebhookInvalidUrlError extends AppError {
  constructor(url: string) {
    super(
      `Webhook URL must use HTTPS: ${url}`,
      422,
      'WEBHOOK_INVALID_URL',
      { url }
    );
  }
}
```

---

### 9.4. `src/middlewares/auth.middleware.ts`

**Linha:** 49 (função `requireRole`)

**Integração:** Reutilizar para endpoint de replay DLQ:

```typescript
// src/modules/webhooks/webhook.routes.ts
router.post(
  '/admin/webhooks/dead-letter/:id/replay',
  authenticate,
  requireRole('ADMIN'), // 🔒 Reuso
  webhookController.replayDLQ
);
```

---

### 9.5. `src/middlewares/error.middleware.ts`

**Linha:** 14 (tratamento de AppError)

**Integração:** Nenhuma modificação necessária. Erros de webhook herdam de `AppError` e são automaticamente formatados:

```typescript
// Erro lançado:
throw new WebhookNotFoundError();

// Middleware transforma em:
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found"
  }
}
```

---

### 9.6. `src/shared/logger/index.ts`

**Linha:** Logger Pino configurado

**Integração:** Reutilizar logger existente no worker:

```typescript
import { logger } from '@/shared/logger';

logger.info({
  event_id: eventId,
  webhook_id: webhookId,
  attempt_number: attemptCount,
  http_status: response.status
}, 'Webhook delivered successfully');
```

**Redação automática:** já configurada para `password`, `token`, `authorization`. Adicionar `secret` se necessário.

---

### 9.7. `prisma/schema.prisma`

**Integração:** Adicionar novos models seguindo padrão UUID:

```prisma
model WebhookConfiguration {
  id           String   @id @default(uuid()) @db.Char(36)
  customerId   String   @db.Char(36)
  url          String   @db.VarChar(500)
  secret       String   @db.VarChar(255)
  statusFilter Json     // Array de OrderStatus
  active       Boolean  @default(true)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  customer Customer @relation(fields: [customerId], references: [id])
  outbox   WebhookOutbox[]

  @@index([customerId])
  @@index([active])
  @@map("webhook_configuration")
}

model WebhookOutbox {
  id            String   @id @default(uuid()) @db.Char(36)
  eventId       String   @db.Char(36)
  webhookId     String   @db.Char(36)
  orderId       String   @db.Char(36)
  payload       Json
  status        String   @db.VarChar(20) // pendente, processando, entregue, failed
  attemptCount  Int      @default(0)
  nextRetryAt   DateTime?
  deliveredAt   DateTime?
  createdAt     DateTime @default(now())

  webhook WebhookConfiguration @relation(fields: [webhookId], references: [id])
  order   Order @relation(fields: [orderId], references: [id])

  @@index([status, createdAt])
  @@index([webhookId])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id          String   @id @default(uuid()) @db.Char(36)
  eventId     String   @db.Char(36)
  webhookId   String   @db.Char(36)
  payload     Json
  failReason  String   @db.Text
  failedAt    DateTime @default(now())

  @@index([webhookId])
  @@map("webhook_dead_letter")
}
```

---

## 10. Critérios de Aceite Técnicos

### Funcional
- [ ] Toda mudança de status gera evento na outbox (se houver webhook configurado)
- [ ] Evento não é criado se transação de status falha (atomicidade)
- [ ] Worker processa eventos na ordem de `created_at`
- [ ] Assinatura HMAC-SHA256 está correta e validável
- [ ] Cliente recebe headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`
- [ ] Payload contém todos os campos especificados
- [ ] Filtro de status funciona (webhook só recebe status configurados)

### Performance
- [ ] Latência p95 de criação do evento < 50ms (adicionado à transação)
- [ ] Latência p95 de entrega < 10s (do momento da mudança de status até POST)
- [ ] Worker processa batch de 50 eventos em < 5 segundos (assumindo clientes rápidos)

### Resiliência
- [ ] Retry ocorre nos intervalos corretos (1m, 5m, 30m, 2h, 12h)
- [ ] Evento vai para DLQ após 5 tentativas falhadas
- [ ] Timeout de 10s é respeitado
- [ ] Worker recupera de crash sem perder eventos

### Segurança
- [ ] URL não-HTTPS é rejeitada com `WEBHOOK_INVALID_URL`
- [ ] Secret é gerada com entropia >= 128 bits
- [ ] Secret não aparece em logs nem em respostas de listagem
- [ ] Rotação de secret mantém antiga válida por exatamente 24h
- [ ] Endpoint de replay exige role ADMIN

### Observabilidade
- [ ] Métricas são exportadas para Prometheus
- [ ] Logs estruturados incluem `event_id`, `webhook_id`, `customer_id`
- [ ] Tracing distribu ído funciona ponta a ponta

---

## 11. Riscos e Mitigação

### Risco 1: Worker crash prolonga delay de entrega
- **Probabilidade:** Média
- **Impacto:** Delay adicional de até tempo de restart (~1-2 min)
- **Mitigação:**
  - Process manager (PM2 ou systemd) com restart automático
  - Alerta se worker parado > 5 min
- **Plano de contingência:** Restart manual se automático falhar

### Risco 2: Hot keys na outbox causam contenção MySQL
- **Probabilidade:** Baixa
- **Impacto:** Degradação de latência da transação de `changeStatus`
- **Mitigação:**
  - Índice composto `(status, created_at)` otimizado
  - Batch processing pequeno (50 eventos/ciclo)
  - Monitorar locks no MySQL
- **Plano de contingência:** Aumentar frequência de arquivamento

### Risco 3: Cliente demora > 10s consistentemente
- **Probabilidade:** Baixa
- **Impacto:** Acúmulo de retries e DLQ
- **Mitigação:**
  - Timeout agressivo de 10s
  - Métricas por webhook_id identificam problema
  - Cliente recebe notificação via endpoint de histórico
- **Plano de contingência:** Desativar webhook automaticamente após X falhas consecutivas (fase futura)

### Risco 4: Payload > 64KB por ordem muito grande
- **Probabilidade:** Muito Baixa
- **Impacto:** Evento rejeitado, não entregue
- **Mitigação:**
  - Payload não inclui `items` ([09:43] Diego)
  - Apenas campos escalares do pedido
  - Validação retorna erro claro
- **Plano de contingência:** Cliente busca detalhes via `GET /orders/:id`

---

## Referências

- **Transcrição da Reunião Técnica**: Timestamps indicados ao longo do documento
- **Código Existente**:
  - `src/modules/orders/order.service.ts` (changeStatus)
  - `src/modules/orders/order.status.ts` (máquina de estados)
  - `src/shared/errors/http-errors.ts` (classes de erro)
  - `src/middlewares/auth.middleware.ts` (requireRole)
  - `src/middlewares/error.middleware.ts` (error handling)
  - `src/shared/logger/index.ts` (Pino logger)
  - `prisma/schema.prisma` (models e padrão UUID)
- **ADRs**:
  - [ADR-001 a ADR-006](./adrs/)
- **RFC**: `docs/RFC.md`
- **PRD**: `docs/PRD.md`
