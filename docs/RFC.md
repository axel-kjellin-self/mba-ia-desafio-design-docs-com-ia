# RFC: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0
**Data:** [Data da reunião técnica]
**Autor:** Larissa (Tech Lead)
**Revisores:** Marcos (PM), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Eng. Segurança)
**Status:** Proposto para Revisão

---

## TL;DR (Resumo Executivo)

Implementar sistema de webhooks outbound para notificar clientes B2B quando status de pedidos mudam. Solução usa **padrão Outbox no MySQL** para garantir atomicidade, **worker em polling** para processamento assíncrono, **retry com backoff exponencial** para resiliência, e **HMAC-SHA256** para autenticação. Delivery **at-least-once** com deduplicação via `X-Event-Id`. Prazo: 3 sprints.

---

## 1. Contexto e Problema

### Problema de Negócio
Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) fazem polling contínuo no `GET /orders` para detectar mudanças de status, tornando a integração lenta e cara. Atlas ameaça migração para concorrente se solução não for entregue até fim do trimestre ([09:00] Marcos).

### Requisito de Negócio
Notificação push com delay **< 10 segundos** quando status do pedido muda ([09:02] Marcos). Apenas outbound webhooks (sistema → cliente), sem inbound ([09:02] Marcos, [09:02] Sofia).

### Restrições Técnicas
- **Transação de mudança de status já é pesada**: atualiza `orders`, insere em `order_status_history`, decrementa `stock_quantity` ([09:04] Bruno)
- **Time pequeno**: evitar subir nova infraestrutura dedicada ([09:07] Diego)
- **Sistema existente**: integração deve reutilizar código, banco de dados e padrões atuais

---

## 2. Proposta Técnica

### Visão Geral da Solução

```
┌─────────────────────────────────────────────────────────────┐
│  Mudança de Status (OrderService.changeStatus)             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Transação MySQL                                     │  │
│  │  1. UPDATE orders SET status = 'PAID'              │  │
│  │  2. INSERT INTO order_status_history                │  │
│  │  3. UPDATE products SET stock_quantity -= N         │  │
│  │  4. INSERT INTO webhook_outbox (event_id, payload)  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         ↓
         ┌───────────────────────────────┐
         │  webhook_outbox (MySQL)       │
         │  - event_id (UUID)            │
         │  - order_id                   │
         │  - status (pendente/entregue) │
         │  - payload (snapshot JSON)    │
         │  - created_at                 │
         └───────────────────────────────┘
                         ↓
         ┌───────────────────────────────┐
         │  Worker (processo separado)   │
         │  - Polling a cada 2s          │
         │  - Lê eventos pendentes       │
         │  - POST HTTP para cliente     │
         │  - Headers: X-Signature,      │
         │    X-Event-Id, X-Timestamp    │
         └───────────────────────────────┘
                         ↓
         ┌───────────────────────────────┐
         │  Retry & DLQ                  │
         │  - 5 tentativas backoff       │
         │  - 1m/5m/30m/2h/12h           │
         │  - DLQ após esgota r          │
         └───────────────────────────────┘
```

### Componentes Principais

#### 1. Tabela `webhook_outbox` (Outbox Pattern)
- Eventos inseridos **dentro da transação** de mudança de status
- Garante atomicidade: evento registrado ↔ status mudou
- Índices em `(status, created_at)` para leitura eficiente
- **Ver [ADR-001: Padrão Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)**

#### 2. Worker em Processo Separado
- Entry-point `src/worker.ts` com script `npm run worker`
- Polling a cada 2 segundos, busca eventos `status = 'pendente'`
- Processa em batch pequeno (10-50 eventos)
- Prisma Client separado, mesmo banco
- **Ver [ADR-005: Worker em Processo Separado](./adrs/ADR-005-worker-processo-separado-polling.md)**

#### 3. Módulo `src/modules/webhooks`
- CRUD de configuração de webhooks (url, secret, filtro de status)
- Endpoint de histórico de entregas
- Endpoint admin para replay de DLQ
- **Ver [ADR-006: Reuso de Padrões](./adrs/ADR-006-reuso-padroes-projeto.md)**

#### 4. Sistema de Retry e DLQ
- 5 tentativas com backoff exponencial: 1m → 5m → 30m → 2h → 12h
- Timeout HTTP: 10 segundos
- Eventos que esgotam retries → tabela `webhook_dead_letter`
- **Ver [ADR-002: Retry com Backoff e DLQ](./adrs/ADR-002-retry-backoff-dlq.md)**

#### 5. Autenticação HMAC-SHA256
- Secret única por endpoint webhook
- Assinatura em header `X-Signature`
- Rotação de secret com grace period de 24h
- TLS obrigatório (apenas HTTPS)
- **Ver [ADR-003: HMAC-SHA256](./adrs/ADR-003-hmac-sha256-por-endpoint.md)**

#### 6. Semântica At-Least-Once
- Cliente pode receber evento múltiplas vezes
- Header `X-Event-Id` (UUID) para deduplicação
- Cliente deduplica do lado dele
- **Ver [ADR-004: At-Least-Once](./adrs/ADR-004-at-least-once-event-id.md)**

### Payload do Webhook (Exemplo)

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "order.status_changed",
  "timestamp": "2024-11-15T14:23:45.123Z",
  "order_id": "abc123...",
  "order_number": "ORD-000042",
  "from_status": "PENDING",
  "to_status": "PAID",
  "customer_id": "def456...",
  "total_cents": 15990
}
```
([09:43] Diego: não inclui `items` para manter payload enxuto)

### Headers HTTP

- `X-Event-Id`: UUID único (deduplicação)
- `X-Signature`: HMAC-SHA256(secret, body)
- `X-Timestamp`: timestamp do envio (detecção de replay attack)
- `X-Webhook-Id`: ID da configuração do webhook (clientes com múltiplos endpoints)
- `Content-Type`: application/json

([09:44] Diego, [09:44] Sofia)

---

## 3. Alternativas Consideradas

### Alternativa 1: Disparo Síncrono dentro da Transação
**Proposta:** Fazer HTTP call para cliente dentro do `OrderService.changeStatus`, dentro da transação MySQL.

**Descartada por:** Bruno ([09:04]) e Diego ([09:06])

**Trade-off:**
- ✅ Simplicidade: sem tabela de outbox, sem worker
- ❌ **Performance**: cliente lento trava mudança de status para todos os pedidos
- ❌ **Resiliência**: cliente offline forçaria rollback da mudança de status (semântica incorreta)
- ❌ **Acoplamento**: disponibilidade do cliente afeta operação crítica do sistema

**Razão do descarte:** Transação já é pesada; cliente lento/offline não pode impactar operação crítica.

### Alternativa 2: Redis Streams ou Mensageria Dedicada
**Proposta:** Usar Redis Streams, Kafka ou RabbitMQ como camada de eventos.

**Considerada por:** Larissa ([09:07])
**Descartada por:** Diego ([09:07])

**Trade-off:**
- ✅ Latência menor (notificação reativa)
- ✅ Throughput maior
- ❌ **Operacional**: subir e gerenciar Redis Cluster ou Kafka
- ❌ **Complexidade**: time pequeno, mais infra para manter
- ❌ **Garantias**: ainda precisaria outbox para atomicidade com transação MySQL

**Razão do descarte:** Overengineering para time pequeno. Outbox MySQL atende requisito de latência (< 10s) confortavelmente.

### Alternativa 3: Trigger MySQL para Notificação Reativa
**Proposta:** Trigger SQL que notifica worker quando evento é inserido.

**Proposta por:** Bruno ([09:09])
**Descartada por:** Diego ([09:09])

**Trade-off:**
- ✅ Latência menor que polling
- ❌ **Limitação técnica**: MySQL não tem `NOTIFY/LISTEN` como Postgres
- ❌ **Workarounds esquisitos**: escrever em arquivo ou bater em endpoint HTTP via trigger

**Razão do descarte:** Polling de 2s atende requisito; implementação seria complexa e frágil.

### Alternativa 4: 3 Tentativas de Retry (Mais Agressivo)
**Proposta:** Apenas 3 tentativas de retry em vez de 5.

**Proposta por:** Bruno ([09:16])
**Descartada por:** Diego ([09:16])

**Trade-off:**
- ✅ Falha mais rápido (30 minutos vs. 15 horas)
- ❌ **Cobertura insuficiente**: Cliente com manutenção de 2h perderia eventos

**Razão do descarte:** 5 tentativas (15h) cobre janelas realistas de manutenção programada.

### Alternativa 5: Exactly-Once Delivery
**Proposta:** Garantir que evento é entregue exatamente uma vez.

**Descartada por:** Diego ([09:25])

**Trade-off:**
- ✅ Simplicidade para o cliente (sem deduplicação)
- ❌ **Complexidade extrema**: protocolo de confirmação bidirecional, coordenação distribuída
- ❌ **Não é padrão de mercado**: Stripe, GitHub usam at-least-once

**Razão do descarte:** Benefício marginal não justifica complexidade.

---

## 4. Questões em Aberto

### QA-01: Rate Limiting de Saída por Cliente
**Pergunta:** Se cliente tiver 50 pedidos mudando status em 1 minuto, enviamos 50 webhooks sem limite?

**Discussão:** Diego ([09:38]) levanta a questão; Larissa ([09:39]) decide "observar e implementar se virar problema".

**Status:** Fora de escopo inicial. Observar em produção, decidir em fase futura se necessário.

**Critério para revisitar:** Se algum cliente relatar sobrecarga ou se métricas mostrarem picos problemáticos.

### QA-02: Escala com Múltiplos Workers
**Pergunta:** Se precisar escalar além de single worker, como garantir ordem de eventos?

**Discussão:** Bruno ([09:13]) pergunta sobre escala; Diego ([09:13]) sugere particionamento por `order_id` ou lock pessimista, mas "problema do futuro".

**Status:** Single worker no MVP. Limitação documentada: ordenação garantida por `order_id` apenas com single worker ([09:13] Larissa).

**Critério para revisitar:** Se throughput de eventos superar capacidade de single worker (~50k eventos/min estimado).

### QA-03: Notificação de Problemas ao Cliente
**Pergunta:** Enviar email para cliente quando webhook falhar repetidamente?

**Discussão:** Marcos ([09:37]) pergunta sobre email de fallback; Larissa ([09:37]) decide "não, fora de escopo dessa fase".

**Status:** Explicitamente fora de escopo inicial. Portal de desenvolvedor terá documentação; cliente monitora via endpoint de histórico de entregas.

**Critério para revisitar:** Após MVP em produção, medir quantos clientes têm webhooks em DLQ cronicamente.

---

## 5. Impacto e Riscos

### Impacto em Sistemas Existentes

#### `src/modules/orders/order.service.ts`
- **Modificação**: Método `changeStatus` (linha ~126)
- **Alteração**: Adicionar `await publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação existente
- **Risco**: Se inserção na outbox falhar, rollback completo (comportamento desejado)

#### Banco de Dados
- **Novas tabelas**: `webhook_configuration`, `webhook_outbox`, `webhook_dead_letter`
- **Índices**: `(status, created_at)` em `webhook_outbox` crítico para performance de polling
- **Crescimento**: Outbox precisa de processo de arquivamento (fora de escopo, [09:08] Diego)

### Riscos Técnicos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Worker crashar e parar | Média | Alto (eventos não processados) | Process manager (PM2, systemd); alertas de "último evento processado" |
| Cliente offline > 15h | Baixa | Médio (eventos perdidos) | Endpoint de replay manual via DLQ; documentação clara do comportamento |
| Hot keys na outbox | Baixa | Médio (contenção MySQL) | Índices otimizados; batch processing; monitoramento de locks |
| Vazamento de secret | Baixa | Alto (segurança) | Rotação via API; secret por endpoint isola impacto ([ADR-003](./adrs/ADR-003-hmac-sha256-por-endpoint.md)) |
| Payload > 64KB | Muito Baixa | Baixo (evento rejeitado) | Validação retorna erro claro; payload não inclui `items` ([09:43] Diego) |

---

## 6. Observabilidade

### Métricas Essenciais
- `webhook_events_created_total` (por order_id, status)
- `webhook_delivery_attempts_total{status="success|failure|retry"}`
- `webhook_delivery_latency_seconds` (histograma)
- `webhook_dlq_events_total`
- `webhook_worker_last_processed_timestamp`

### Logs Estruturados
- **Pino** reutilizado ([ADR-006](./adrs/ADR-006-reuso-padroes-projeto.md))
- Campos: `event_id`, `order_id`, `customer_id`, `webhook_id`, `attempt_number`, `http_status`, `latency_ms`
- Redação automática de `payload` e `secret`

### Alertas Mínimos
- Worker parado (> 5 minutos sem processar)
- Taxa de falha > 10% por cliente
- DLQ acumulando eventos (> 100 eventos)

---

## 7. Dependências

### Técnicas
- MySQL 8.0 (já existe)
- Prisma ORM (já existe)
- Node.js 20+ (já existe)
- Biblioteca `uuid` (já existe no `package.json`)

### Organizacionais
- **Aprovação de Segurança**: Sofia precisa revisar implementação de HMAC e geração de secrets (2 dias úteis reservados, [09:46] Larissa)
- **Documentação para Clientes**: Marcos responsável por atualizar portal de desenvolvedor com guias de integração ([09:40] Marcos)

### Bloqueantes
Nenhum bloqueante identificado. Feature pode iniciar imediatamente após aprovação deste RFC.

---

## 8. Decisões Relacionadas

- [ADR-001: Padrão Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Política de Retry com Backoff Exponencial e DLQ](./adrs/ADR-002-retry-backoff-dlq.md)
- [ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint](./adrs/ADR-003-hmac-sha256-por-endpoint.md)
- [ADR-004: Garantia At-Least-Once com X-Event-Id](./adrs/ADR-004-at-least-once-event-id.md)
- [ADR-005: Worker em Processo Separado com Polling](./adrs/ADR-005-worker-processo-separado-polling.md)
- [ADR-006: Reuso Máximo dos Padrões Existentes](./adrs/ADR-006-reuso-padroes-projeto.md)

---

## 9. Cronograma e Próximos Passos

### Estimativa
**3 sprints** ([09:46] Larissa):
- Sprint 1: Modelagem outbox + DLQ + schemas Prisma
- Sprint 2: Worker + retry + processamento
- Sprint 3: CRUD configuração + deliveries + integração OrderService + testes E2E

### Próximos Passos Imediatos
1. **Aprovação deste RFC** pelos revisores listados
2. **Kick-off técnico**: Bruno e Diego revisar com Larissa ([09:50] Larissa)
3. **FDD detalhado**: Contratos de API, matriz de erros, fluxos completos
4. **Revisão de segurança**: Sofia revisa implementação HMAC antes do deploy ([09:47] Sofia)

---

## Referências

- **Transcrição da Reunião Técnica**: Timestamps indicados ao longo do documento
- **Código Existente**:
  - `src/modules/orders/order.service.ts` (changeStatus method)
  - `src/shared/errors/` (classes de erro)
  - `src/middlewares/auth.middleware.ts` (requireRole)
  - `prisma/schema.prisma` (padrão de UUIDs)
- **PRD**: `docs/PRD.md` (requisitos de produto)
- **FDD**: `docs/FDD.md` (especificação de implementação)
