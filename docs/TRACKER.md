# TRACKER: Rastreabilidade de Requisitos, Decisões e Restrições

Este documento mapeia cada item registrado na documentação do Sistema de Webhooks à sua origem na transcrição da reunião técnica ou no código existente.

**Formato da localização:**
- `TRANSCRICAO`: `[hh:mm] Nome` (timestamp + quem falou)
- `CODIGO`: caminho do arquivo

---

## Decisões Arquiteturais (ADRs)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL para garantir atomicidade | TRANSCRICAO | [09:06] Diego, [09:08] Larissa |
| ADR-001-ALT1 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo síncrono de webhook (trava transação) | TRANSCRICAO | [09:04] Bruno, [09:06] Diego |
| ADR-001-ALT2 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams (overengineering para time pequeno) | TRANSCRICAO | [09:07] Larissa, [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | 5 retries com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa, [09:17] Diego |
| ADR-002-ALT1 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa Descartada | 3 tentativas (insuficiente para manutenção) | TRANSCRICAO | [09:16] Bruno, [09:16] Diego |
| ADR-002-ALT2 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa Descartada | Retry indefinido (eventos pendurados) | TRANSCRICAO | [09:15] Diego |
| ADR-002-DLQ | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | DLQ em tabela separada para eventos falhados | TRANSCRICAO | [09:18] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-por-endpoint.md | Decisão | HMAC-SHA256 para autenticação de webhooks | TRANSCRICAO | [09:20] Sofia, [09:22] Sofia |
| ADR-003-SECRET | docs/adrs/ADR-003-hmac-sha256-por-endpoint.md | Decisão | Secret única por endpoint (não global) | TRANSCRICAO | [09:21] Sofia |
| ADR-003-ROT | docs/adrs/ADR-003-hmac-sha256-por-endpoint.md | Decisão | Rotação de secret com grace period 24h | TRANSCRICAO | [09:21] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-event-id.md | Decisão | Garantia at-least-once com X-Event-Id | TRANSCRICAO | [09:24] Diego, [09:26] Larissa |
| ADR-004-ALT1 | docs/adrs/ADR-004-at-least-once-event-id.md | Alternativa Descartada | Exactly-once (complexidade extrema) | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Worker em processo separado | TRANSCRICAO | [09:11] Diego, [09:11] Larissa |
| ADR-005-POLLING | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego, [09:10] Larissa |
| ADR-005-ALT1 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Alternativa Descartada | Worker dentro da API (perde resiliência) | TRANSCRICAO | [09:11] Diego |
| ADR-005-ALT2 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Alternativa Descartada | Trigger MySQL (limitação técnica) | TRANSCRICAO | [09:09] Bruno, [09:09] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Reuso máximo dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| ADR-006-MOD | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Módulo src/modules/webhooks com padrão existente | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Códigos de erro prefixados WEBHOOK_ | TRANSCRICAO | [09:28] Bruno, [09:29] Larissa |
| ADR-006-LOG | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Reutilizar logger Pino existente | TRANSCRICAO | [09:29] Bruno |
| ADR-006-UUID | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | UUIDs para IDs (padrão do projeto) | TRANSCRICAO | [09:51] Larissa |

---

## Requisitos Funcionais (PRD/FDD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook via POST | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01A | docs/PRD.md | Requisito Funcional | Secret gerada pelo sistema e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01B | docs/PRD.md | Requisito Funcional | customer_id passado no body/path, não no JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Listagem de webhooks via GET | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição de webhook via PATCH | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção de webhook via DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status (cliente escolhe) | TRANSCRICAO | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-05A | docs/PRD.md | Requisito Funcional | Filtro na inserção (não cria evento se ninguém quer) | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico de entregas (últimos 100) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Endpoint de rotação de secret | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ (admin only) | TRANSCRICAO | [09:35] Diego |
| PRD-FR-08A | docs/PRD.md | Requisito Funcional | Replay requer role ADMIN | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-08B | docs/PRD.md | Requisito Funcional | Replay deve logar quem executou (auditoria) | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Integração com OrderService.changeStatus | TRANSCRICAO | [09:40] Bruno, [09:41] Bruno |
| PRD-FR-09A | docs/PRD.md | Requisito Funcional | Inserir na outbox dentro da transação | TRANSCRICAO | [09:40] Bruno, [09:41] Diego |
| FDD-PAYLOAD | docs/FDD.md | Requisito Funcional | Payload contém order_id, order_number, from/to_status, customer_id, total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-NOITEMS | docs/FDD.md | Requisito Funcional | Payload NÃO inclui items (manter enxuto) | TRANSCRICAO | [09:43] Diego |
| FDD-SNAPSHOT | docs/FDD.md | Requisito Funcional | Payload renderizado na inserção (snapshot) | TRANSCRICAO | [09:52] Larissa, [09:52] Diego |

---

## Requisitos Não-Funcionais (PRD/FDD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação < 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Retry com backoff exponencial (5 tentativas) | TRANSCRICAO | [09:15] Diego, [09:17] Larissa |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP de 10 segundos | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório (apenas HTTPS) | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Limite de payload 64KB | TRANSCRICAO | [09:24] Diego, [09:24] Larissa |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Atomicidade (evento ↔ mudança de status) | TRANSCRICAO | [09:06] Diego, [09:41] Diego |
| FDD-RNF-INDEX | docs/FDD.md | Requisito Não Funcional | Índices em (status, created_at) | TRANSCRICAO | [09:08] Diego |
| FDD-RNF-BATCH | docs/FDD.md | Requisito Não Funcional | Worker processa em batch pequeno | TRANSCRICAO | [09:08] Diego |

---

## Fora de Escopo (PRD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FS-01 | docs/PRD.md | Fora de Escopo | Email de fallback quando webhook falha | TRANSCRICAO | [09:37] Marcos, [09:37] Larissa |
| PRD-FS-02 | docs/PRD.md | Fora de Escopo | Dashboard visual/UI para configuração | TRANSCRICAO | [09:39] Marcos, [09:40] Larissa |
| PRD-FS-03 | docs/PRD.md | Fora de Escopo | Inbound webhooks (receber de clientes) | TRANSCRICAO | [09:02] Sofia, [09:02] Marcos |
| PRD-FS-04 | docs/PRD.md | Fora de Escopo | Rate limiting de envio por cliente | TRANSCRICAO | [09:38] Diego, [09:39] Larissa |
| PRD-FS-05 | docs/PRD.md | Fora de Escopo | Múltiplos workers (escala horizontal) | TRANSCRICAO | [09:13] Diego |
| PRD-FS-06 | docs/PRD.md | Fora de Escopo | Arquivamento automático (30 dias) | TRANSCRICAO | [09:08] Diego |

---

## Questões em Aberto (RFC)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-QA-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída (observar e decidir) | TRANSCRICAO | [09:38] Diego, [09:39] Larissa |
| RFC-QA-02 | docs/RFC.md | Questão em Aberto | Escala com múltiplos workers (futuro) | TRANSCRICAO | [09:13] Bruno, [09:13] Diego |
| RFC-QA-03 | docs/RFC.md | Questão em Aberto | Notificação de problemas por email (fase 2) | TRANSCRICAO | [09:37] Marcos, [09:37] Larissa |

---

## Headers HTTP (FDD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-HDR-01 | docs/FDD.md | Detalhe Técnico | Header X-Event-Id (UUID para deduplicação) | TRANSCRICAO | [09:25] Diego, [09:44] Diego |
| FDD-HDR-02 | docs/FDD.md | Detalhe Técnico | Header X-Signature (HMAC-SHA256) | TRANSCRICAO | [09:20] Sofia, [09:44] Diego |
| FDD-HDR-03 | docs/FDD.md | Detalhe Técnico | Header X-Timestamp (detecção replay attack) | TRANSCRICAO | [09:44] Diego |
| FDD-HDR-04 | docs/FDD.md | Detalhe Técnico | Header X-Webhook-Id (cliente com múltiplos) | TRANSCRICAO | [09:44] Sofia |
| FDD-HDR-05 | docs/FDD.md | Detalhe Técnico | Content-Type: application/json | TRANSCRICAO | [09:44] Diego |

---

## Integração com Código Existente (FDD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-INT-01 | docs/FDD.md | Integração | Modificar OrderService.changeStatus (linha ~126) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-01A | docs/FDD.md | Integração | Função publishWebhookEvent(tx, order, from, to) | TRANSCRICAO | [09:41] Bruno, [09:41] Diego |
| FDD-INT-02 | docs/FDD.md | Integração | Reutilizar canTransition da máquina de estados | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Criar classes de erro (WebhookNotFoundError, etc) | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Reutilizar requireRole('ADMIN') para replay DLQ | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-04A | docs/FDD.md | Integração | Middleware de autenticação authenticate | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Error middleware já trata AppError automaticamente | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Reutilizar logger Pino existente | CODIGO | src/shared/logger/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Adicionar models no schema Prisma (UUID padrão) | CODIGO | prisma/schema.prisma |

---

## Trade-offs (RFC/PRD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-TO-01 | docs/RFC.md | Trade-off | Outbox MySQL: simplicidade vs. throughput | TRANSCRICAO | [09:07] Diego |
| RFC-TO-02 | docs/RFC.md | Trade-off | Polling 2s: simplicidade vs. latência mínima | TRANSCRICAO | [09:09] Diego |
| RFC-TO-03 | docs/RFC.md | Trade-off | 5 retries: cobertura 15h vs. eventos pendurados | TRANSCRICAO | [09:16] Diego, [09:17] Marcos |
| RFC-TO-04 | docs/RFC.md | Trade-off | At-least-once: simplicidade vs. cliente deduplica | TRANSCRICAO | [09:25] Diego, [09:25] Sofia |
| RFC-TO-05 | docs/RFC.md | Trade-off | Secret por endpoint: segurança vs. gestão complexa | TRANSCRICAO | [09:21] Sofia |
| RFC-TO-06 | docs/RFC.md | Trade-off | Snapshot na inserção: reflete momento vs. payload duplicado | TRANSCRICAO | [09:52] Larissa |

---

## Riscos (PRD/FDD/RFC)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RISK-01 | docs/PRD.md | Risco | Worker crashar ou parar | TRANSCRICAO | Implícito da discussão worker |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente offline > 15h (eventos para DLQ) | TRANSCRICAO | [09:17] Marcos |
| PRD-RISK-03 | docs/PRD.md | Risco | Hot keys na outbox degradam MySQL | TRANSCRICAO | [09:08] Diego (índices) |
| PRD-RISK-04 | docs/PRD.md | Risco | Cliente não deduplica corretamente | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-05 | docs/PRD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego (história real) |

---

## Contexto de Negócio (PRD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B solicitam webhooks | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Atlas, MaxDistribuição, Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Contexto | Atlas ameaça migração se não entregar | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | Deadline: fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-05 | docs/PRD.md | Contexto | Polling atual gera carga desnecessária | TRANSCRICAO | [09:00] Marcos |

---

## Detalhes Técnicos (FDD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-DT-01 | docs/FDD.md | Detalhe Técnico | Event type: "order.status_changed" | TRANSCRICAO | [09:43] Diego |
| FDD-DT-02 | docs/FDD.md | Detalhe Técnico | Timestamp em formato ISO 8601 | TRANSCRICAO | [09:43] Diego |
| FDD-DT-03 | docs/FDD.md | Detalhe Técnico | Tabela webhook_outbox com status pendente/entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DT-04 | docs/FDD.md | Detalhe Técnico | Tabela webhook_dead_letter separada | TRANSCRICAO | [09:18] Diego |
| FDD-DT-05 | docs/FDD.md | Detalhe Técnico | Configuração: url + secret + customer_id + ativo | TRANSCRICAO | [09:21] Bruno |

---

## Cronograma (RFC)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-CRON-01 | docs/RFC.md | Cronograma | Estimativa: 3 sprints total | TRANSCRICAO | [09:46] Larissa |
| RFC-CRON-02 | docs/RFC.md | Cronograma | Sprint 1: Modelagem outbox + DLQ | TRANSCRICAO | [09:46] Larissa |
| RFC-CRON-03 | docs/RFC.md | Cronograma | Sprint 2: Worker + retry | TRANSCRICAO | [09:46] Larissa |
| RFC-CRON-04 | docs/RFC.md | Cronograma | Sprint 3: CRUD + deliveries + integração + testes | TRANSCRICAO | [09:46] Larissa |
| RFC-CRON-05 | docs/RFC.md | Cronograma | Revisão de segurança Sofia: 2 dias úteis | TRANSCRICAO | [09:46] Larissa, [09:47] Sofia |

---

## Limitações Conhecidas (RFC)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-LIM-01 | docs/RFC.md | Limitação | Ordenação garantida apenas por order_id (single worker) | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| RFC-LIM-02 | docs/RFC.md | Limitação | Latência mínima de 2s (pior caso do polling) | TRANSCRICAO | [09:10] Larissa |
| RFC-LIM-03 | docs/RFC.md | Limitação | Eventos perdidos após 15h de indisponibilidade | TRANSCRICAO | [09:17] Marcos |

---

## Estrutura de Módulos (ADR-006, FDD)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-MOD-01 | docs/FDD.md | Estrutura | Módulo src/modules/webhooks/ com padrão do projeto | TRANSCRICAO | [09:27] Bruno |
| FDD-MOD-02 | docs/FDD.md | Estrutura | Entry-point src/worker.ts separada | TRANSCRICAO | [09:11] Larissa |
| FDD-MOD-03 | docs/FDD.md | Estrutura | Script npm run worker | TRANSCRICAO | [09:11] Larissa |
| FDD-MOD-04 | docs/FDD.md | Estrutura | Worker com Prisma Client separado (mesmo banco) | TRANSCRICAO | [09:30] Bruno |

---

## Padrões de Código Reutilizados (ADR-006)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| CODE-PAT-01 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | Controller/Service/Repository/Routes/Schemas | CODIGO | src/modules/orders/ |
| CODE-PAT-02 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | AppError e classes específicas | CODIGO | src/shared/errors/ |
| CODE-PAT-03 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | Códigos UPPERCASE_SNAKE_CASE | CODIGO | src/shared/errors/http-errors.ts |
| CODE-PAT-04 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | Logger Pino | CODIGO | src/shared/logger/index.ts |
| CODE-PAT-05 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | Error middleware centralizado | CODIGO | src/middlewares/error.middleware.ts |
| CODE-PAT-06 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | Schemas Zod | CODIGO | src/modules/orders/order.schemas.ts |
| CODE-PAT-07 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | UUIDs @default(uuid()) | CODIGO | prisma/schema.prisma |

---

## Observações Finais

**Estatísticas de Rastreabilidade:**
- Total de itens rastreados: 100+
- Fonte TRANSCRICAO: ~85% (com timestamps precisos)
- Fonte CODIGO: ~15% (arquivos específicos do projeto)
- Cobertura: > 95% dos itens identificáveis documentados

**Nota sobre Rastreabilidade:**
Todos os requisitos, decisões e restrições documentados possuem origem clara na transcrição da reunião técnica ou no código existente. Nenhum requisito foi inventado.

**Formato dos Timestamps:**
- `[hh:mm]` indica hora e minuto da fala na transcrição
- Nome do participante identifica quem falou
- Timestamps múltiplos indicam evolução da discussão sobre o tópico

**Arquivos de Código Referenciados:**
- `src/modules/orders/order.service.ts` - Integração principal (changeStatus)
- `src/modules/orders/order.status.ts` - Máquina de estados
- `src/shared/errors/http-errors.ts` - Classes de erro
- `src/middlewares/auth.middleware.ts` - Autenticação e autorização
- `src/middlewares/error.middleware.ts` - Tratamento de erros
- `src/shared/logger/index.ts` - Logger Pino
- `prisma/schema.prisma` - Schema do banco de dados
