# PRD: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0
**Data:** [Data da reunião técnica]
**Responsável:** Marcos (Product Manager)

---

## Resumo

Sistema de notificação push para clientes B2B quando status de pedidos mudam no Order Management System. Substitui polling contínuo no `GET /orders` por webhooks outbound com latência < 10 segundos, autenticação HMAC-SHA256 e retry automático.

---

## Contexto e Problema

### Público-alvo
- **Primário:** Clientes B2B com integração técnica (Atlas Comercial, MaxDistribuição, Nova Cargo)
- **Secundário:** Futuros clientes B2B que necessitam notificação de mudanças de status em tempo real
- **Personas:** Desenvolvedores de integração, engenheiros de sistemas dos clientes

### Cenários de Uso Chave

**Cenário 1: Cliente processa pagamento e aguarda confirmação**
- Cliente B2B cria pedido e efetua pagamento via gateway externo
- Sistema confirma pagamento e muda status para `PAID`
- Cliente recebe webhook notificando mudança
- Cliente atualiza dashboard interno instantaneamente

**Cenário 2: Rastreamento de envio em tempo real**
- Pedido muda de `PROCESSING` para `SHIPPED`
- Cliente recebe webhook com novo status
- Cliente envia notificação automática ao comprador final via SMS/email

**Cenário 3: Cliente offline durante janela de manutenção**
- Webhook falha 3 vezes por indisponibilidade do cliente
- Sistema continua tentando com backoff exponencial
- Cliente volta online após 3 horas
- Webhook é entregue com sucesso na 4ª tentativa

### Onde a Feature Será Implantada
- **Sistema existente:** Order Management System (OMS) Node.js + TypeScript + MySQL
- **Infraestrutura:** Mesma arquitetura e banco de dados, sem adicionar serviços externos

### Problemas Priorizados

**Problema 1: Polling contínuo sobrecarrega API e aumenta custos**
- **Impacto:** Clientes fazem GET /orders a cada 10-30 segundos; 80% dos requests retornam sem mudanças
- **Prioridade:** Alta
- **Evidência:** Atlas Comercial fazendo ~3000 requests/dia sendo que apenas 150 pedidos mudam de status ([09:00] Marcos)

**Problema 2: Latência de integração prejudica experiência do cliente**
- **Impacto:** Delay de até 30 segundos entre mudança real e detecção pelo cliente
- **Prioridade:** Alta
- **Evidência:** MaxDistribuição relata "integração lenta e cara" ([09:00] Marcos)

**Problema 3: Risco de churn de cliente estratégico**
- **Impacto:** Atlas ameaça migrar para concorrente que oferece webhooks
- **Prioridade:** Crítica
- **Deadline:** Fim do trimestre ([09:00] Marcos)

---

## Objetivos e Métricas

| Objetivo | Métrica | Meta |
|----------|---------|------|
| Reduzir polling desnecessário | % redução de requests GET /orders dos clientes B2B com webhooks | ≥ 80% |
| Notificação em tempo real | Latência p95 entre mudança de status e entrega do webhook | < 10 segundos |
| Alta confiabilidade de entrega | Taxa de sucesso de entrega de webhooks (após retries) | ≥ 99% |
| Retenção de cliente estratégico | Atlas Comercial renova contrato | 100% (sim/não) |
| Adoção por clientes B2B | Número de clientes B2B ativos usando webhooks | ≥ 5 clientes até fim do Q1 |

---

## Escopo

### Incluso

- CRUD completo de configuração de webhooks (criar, listar, editar, remover)
- Notificação automática via POST HTTP quando status de pedido muda
- Autenticação HMAC-SHA256 com secret única por endpoint
- Rotação de secret via API com grace period de 24 horas
- Retry automático com backoff exponencial (5 tentativas: 1m/5m/30m/2h/12h)
- Dead Letter Queue (DLQ) para eventos que falharam definitivamente
- Endpoint admin para reprocessar eventos da DLQ manualmente
- Histórico de entregas (últimos 100 webhooks por configuração)
- Filtro de eventos por status (cliente escolhe quais status receber)
- Headers HTTP: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`
- Garantia at-least-once (cliente pode receber evento múltiplas vezes)
- Documentação de integração no portal de desenvolvedor

### Fora de Escopo

**Item 1: Notificação por email quando webhook falha**
- **Razão:** Priorizar MVP funcional; email é fallback complexo que adiciona nova dependência ([09:37] Larissa)
- **Reavaliação:** Fase 2, após medição de impacto em produção

**Item 2: Dashboard visual/UI para configuração**
- **Razão:** Projeto separado do time de frontend; API-first permite clientes técnicos integrarem imediatamente ([09:40] Larissa)
- **Reavaliação:** Backlog de frontend, Q2

**Item 3: Inbound webhooks (receber eventos de clientes)**
- **Razão:** Requisito não solicitado pelos clientes; apenas outbound atende necessidade atual ([09:02] Marcos, [09:02] Sofia)
- **Reavaliação:** Só se cliente solicitar explicitamente

**Item 4: Rate limiting de envio por cliente**
- **Razão:** Incerteza sobre necessidade real; evitar otimização prematura ([09:39] Diego/Larissa)
- **Reavaliação:** Observar métricas em produção; implementar se padrão de uso indicar necessidade

**Item 5: Suporte a múltiplos workers (escala horizontal)**
- **Razão:** Single worker atende throughput estimado; adicionar complexidade de coordenação é prematuro ([09:13] Diego)
- **Reavaliação:** Quando throughput superar 50k eventos/minuto

---

## Requisitos Funcionais

### FR-001: Cadastro de Webhook
Cliente cadastra endpoint webhook via API POST.

**Fluxo principal:**
1. Cliente autentica com JWT
2. Cliente envia POST /api/v1/webhooks com `customer_id`, `url`, `status_filter`
3. Sistema valida URL (deve ser HTTPS)
4. Sistema gera secret criptograficamente segura
5. Sistema persiste configuração como ativa
6. Sistema retorna configuração incluindo secret (única vez)

**Fluxos alternativos e exceções:**
- Se URL não é HTTPS → retorna erro `WEBHOOK_INVALID_URL` (422)
- Se `status_filter` contém status inválido → retorna erro `VALIDATION_ERROR` (400)

**Erros previstos:**
- `WEBHOOK_INVALID_URL`: URL usa HTTP em vez de HTTPS
- `VALIDATION_ERROR`: Campos obrigatórios faltando ou inválidos

**Prioridade:** Alta

**Fonte:** [09:31] Marcos, [09:23] Sofia

---

### FR-002: Listagem de Webhooks
Cliente lista webhooks configurados de um customer.

**Fluxo principal:**
1. Cliente autentica com JWT
2. Cliente envia GET /api/v1/webhooks?customer_id=<id>
3. Sistema retorna lista paginada de webhooks (sem secret)

**Prioridade:** Alta

**Fonte:** [09:33] Bruno

---

### FR-003: Edição de Webhook
Cliente atualiza URL, filtro de status ou estado (ativo/inativo).

**Fluxo principal:**
1. Cliente autentica com JWT
2. Cliente envia PATCH /api/v1/webhooks/:id
3. Sistema valida mudanças
4. Sistema atualiza configuração
5. Sistema retorna configuração atualizada (sem secret)

**Prioridade:** Alta

**Fonte:** [09:33] Bruno

---

### FR-004: Remoção de Webhook
Cliente remove configuração de webhook.

**Fluxo principal:**
1. Cliente autentica com JWT
2. Cliente envia DELETE /api/v1/webhooks/:id
3. Sistema marca webhook como deletado (soft delete) ou remove fisicamente
4. Sistema retorna 204 No Content

**Prioridade:** Alta

**Fonte:** [09:33] Bruno

---

### FR-005: Notificação Automática de Mudança de Status
Sistema envia POST HTTP para endpoint do cliente quando status do pedido muda.

**Fluxo principal:**
1. Pedido muda de status (ex: PENDING → PAID) via OrderService.changeStatus
2. Sistema identifica webhooks ativos do customer que querem esse status
3. Sistema insere evento na outbox dentro da transação de mudança de status
4. Worker assíncrono lê evento da outbox
5. Worker calcula HMAC-SHA256 sobre payload
6. Worker envia POST para URL do webhook com headers e payload
7. Cliente retorna 2xx → Sistema marca evento como entregue

**Fluxos alternativos e exceções:**
- Se cliente retorna 4xx/5xx ou timeout → Sistema agenda retry com backoff
- Se nenhum webhook ativo quer aquele status → Não cria evento (otimização)

**Erros previstos:**
- `WEBHOOK_DELIVERY_FAILED`: Cliente retorna erro ou timeout
- `WEBHOOK_PAYLOAD_TOO_LARGE`: Payload excede 64KB

**Prioridade:** Crítica (bloqueante)

**Fonte:** [09:04] Bruno, [09:06] Diego, [09:40] Bruno, [09:41] Bruno

---

### FR-006: Filtro de Eventos por Status
Cliente escolhe quais status de pedido deseja receber.

**Fluxo principal:**
1. Cliente configura `status_filter: ["PAID", "SHIPPED", "DELIVERED"]` ao criar webhook
2. Sistema só envia eventos quando pedido muda para um desses status
3. Mudanças para outros status (ex: PROCESSING) não geram evento para esse webhook

**Prioridade:** Média

**Fonte:** [09:33] Marcos, [09:34] Bruno

---

### FR-007: Histórico de Entregas
Cliente visualiza histórico das últimas 100 tentativas de entrega de um webhook.

**Fluxo principal:**
1. Cliente autentica com JWT
2. Cliente envia GET /api/v1/webhooks/:id/deliveries
3. Sistema retorna lista paginada contendo:
   - event_id, attempt_number, http_status, response_time_ms, delivered_at
   - Status: success/failed
   - Erro (se houver)

**Prioridade:** Média

**Fonte:** [09:34] Marcos

---

### FR-008: Rotação de Secret
Cliente gera nova secret mantendo antiga válida por período de transição.

**Fluxo principal:**
1. Cliente autentica com JWT
2. Cliente envia POST /api/v1/webhooks/:id/rotate-secret
3. Sistema gera nova secret
4. Sistema retorna nova secret e timestamp de expiração da antiga
5. Sistema mantém secret antiga válida por 24 horas
6. Após 24h, secret antiga é invalidada automaticamente

**Prioridade:** Média

**Fonte:** [09:21] Sofia

---

### FR-009: Reprocessamento Manual de DLQ (Admin)
Administrador reprocessa evento que falhou definitivamente.

**Fluxo principal:**
1. Admin autentica com JWT (role ADMIN obrigatório)
2. Admin envia POST /admin/webhooks/dead-letter/:id/replay
3. Sistema cria novo evento na outbox com `attempt_count = 0`
4. Sistema registra audit log (quem fez replay)
5. Worker processa evento normalmente

**Fluxos alternativos e exceções:**
- Se usuário não é ADMIN → retorna 403 Forbidden
- Se evento DLQ não existe → retorna 404

**Erros previstos:**
- `FORBIDDEN`: Usuário sem role ADMIN
- `WEBHOOK_DLQ_NOT_FOUND`: Evento não encontrado

**Prioridade:** Média

**Fonte:** [09:35] Diego, [09:36] Sofia

---

## Requisitos Não Funcionais

### Performance
- Latência p95 de notificação (mudança de status → entrega) < 10 segundos ([09:02] Marcos)
- Latência adicional na transação de changeStatus < 50ms (inserção na outbox)
- Worker processa batch de 50 eventos em < 5 segundos (assumindo clientes com latência normal)

### Disponibilidade
- Sistema continua aceitando mudanças de status mesmo se worker estiver offline (eventos acumulam)
- Worker tem restart automático em caso de crash (PM2 ou systemd)
- Uptime do worker ≥ 99.5%

### Segurança e Autorização
- **Autenticação webhook:** HMAC-SHA256 sobre payload com secret única ([09:20] Sofia, [09:22] Sofia)
- **TLS obrigatório:** URLs devem usar HTTPS ([09:23] Sofia)
- **Secret por endpoint:** Isolamento de vazamento ([09:21] Sofia)
- **Rotação:** Suporte a rotação com grace period de 24h ([09:21] Sofia)
- **Controle de acesso:** CRUD de webhooks requer autenticação; replay DLQ requer role ADMIN ([09:36] Sofia)

### Observabilidade
- **Logs estruturados:** Pino com campos `event_id`, `webhook_id`, `customer_id`, `order_id`, `attempt_number`
- **Métricas:** Taxa de criação, taxa de sucesso/falha, latência de entrega, tamanho da DLQ
- **Tracing:** Span de ponta a ponta da mudança de status até entrega do webhook
- **Alertas:** Worker parado > 5min, taxa de falha > 10%, DLQ > 100 eventos

### Confiabilidade e Integridade de Dados
- **Atomicidade:** Mudança de status e criação de evento ocorrem na mesma transação MySQL ([09:06] Diego, [09:40] Bruno)
- **Retry automático:** 5 tentativas com backoff exponencial: 1m, 5m, 30m, 2h, 12h ([09:17] Diego)
- **DLQ:** Eventos que esgotam retries são persistidos para análise e replay ([09:18] Diego)
- **At-least-once:** Garantia de entrega (cliente pode receber duplicata, deduplica por `X-Event-Id`) ([09:24] Diego, [09:26] Larissa)

### Compatibilidade e Portabilidade
- **APIs REST JSON:** Endpoints HTTP seguem padrão RESTful com JSON
- **Versionamento:** `/api/v1/webhooks` indica versão da API
- **Headers padrão:** `X-*` para custom headers seguindo convenção HTTP
- **Payload estável:** Campos do payload seguem naming snake_case e são backwards-compatible

### Compliance
- **Auditoria:** Replay de DLQ registra quem executou ([09:36] Sofia)
- **Retenção:** Eventos entregues arquivados após 30 dias (fora de escopo MVP, [09:08] Diego)

### Acessibilidade no Frontend Consumidor
- N/A (feature é API-only no MVP)

---

## Arquitetura e Abordagem

### Abordagem
- **Padrão Outbox:** Eventos persistidos em tabela MySQL dentro da transação de mudança de status
- **Worker assíncrono:** Processo Node.js separado com polling a cada 2 segundos
- **Retry com backoff:** Exponencial 1m → 5m → 30m → 2h → 12h, depois DLQ

### Componentes
- **Módulo `src/modules/webhooks`:** Controller, Service, Repository, Routes, Schemas (segue padrão do projeto)
- **Tabela `webhook_outbox`:** Armazena eventos pendentes/entregues
- **Tabela `webhook_dead_letter`:** Armazena eventos que falharam após 5 tentativas
- **Worker (`src/worker.ts`):** Entry-point separada para processamento assíncrono

### Integrações
- **OrderService.changeStatus:** Inserção na outbox dentro da transação existente
- **Clientes externos:** POST HTTP para endpoints configurados

---

## Decisões e Trade-offs

### Decisão 1: Padrão Outbox no MySQL vs. Redis Streams
**Decisão:** Outbox no MySQL ([09:08] Larissa)

- **Justificativa:** Time pequeno; evitar subir e gerenciar Redis Cluster; garantias ACID nativas; reutilização de infraestrutura existente ([09:07] Diego)
- **Trade-off:** Simplicidade e garantias ACID vs. possível latência maior e throughput limitado comparado a soluções dedicadas

### Decisão 2: Worker em Polling vs. Event-Driven Reativo
**Decisão:** Polling a cada 2 segundos ([09:09] Diego, [09:10] Larissa)

- **Justificativa:** MySQL não tem NOTIFY/LISTEN nativo; polling atende requisito de < 10s com margem confortável; implementação simples e confiável
- **Trade-off:** Latência mínima de 2s vs. complexidade de workarounds para notificação reativa

### Decisão 3: 5 Retries vs. 3 Retries
**Decisão:** 5 retries com backoff 1m/5m/30m/2h/12h ([09:17] Larissa)

- **Justificativa:** Clientes reais têm janelas de manutenção de 2-3 horas; 3 tentativas em 30min é insuficiente; 5 tentativas cobrem ~15h ([09:16] Diego)
- **Trade-off:** Cobertura de 15h de indisponibilidade vs. eventos podem ficar pendentes por longo período

### Decisão 4: At-Least-Once vs. Exactly-Once
**Decisão:** At-least-once com deduplicação via `X-Event-Id` ([09:24] Diego, [09:26] Larissa)

- **Justificativa:** Exactly-once exige coordenação distribuída complexa; at-least-once é padrão de mercado (Stripe, GitHub); simplicidade vs. benefício marginal ([09:25] Diego)
- **Trade-off:** Cliente precisa implementar deduplicação vs. sistema permanece simples e confiável

### Decisão 5: Secret por Endpoint vs. Secret Global
**Decisão:** Secret única por endpoint ([09:21] Sofia)

- **Justificativa:** Isolamento de vazamento; se uma secret vaza (ex: log de cliente), apenas aquele endpoint é afetado, não todos ([09:21] Sofia, [09:22] Diego)
- **Trade-off:** Complexidade de gestão de múltiplas secrets vs. segurança robusta

---

## Dependências

### Técnica: Nenhuma Biblioteca Nova
- **Descrição:** Feature reutiliza 100% das dependências existentes (Prisma, Pino, Zod, uuid, crypto built-in)
- **Razão:** Simplicidade; manter surface area pequena

### Organizacional: Aprovação de Segurança
- **Descrição:** Sofia (Eng. Segurança) precisa revisar implementação de HMAC e geração de secrets antes do deploy
- **Prazo:** 2 dias úteis reservados no cronograma ([09:46] Larissa)
- **Bloqueante:** Sim (deploy bloqueado sem aprovação)

### Organizacional: Documentação para Clientes
- **Descrição:** Marcos responsável por atualizar portal de desenvolvedor com guias de integração
- **Prazo:** Antes do lançamento para clientes
- **Bloqueante:** Não (feature funciona sem docs, mas adoção será baixa)

---

## Riscos e Mitigação

### Risco 1: Worker crashar ou parar
- **Probabilidade:** Média
- **Impacto:** Eventos não são processados enquanto worker está offline; delay acumula
- **Mitigação:**
  - Process manager (PM2 ou systemd) com restart automático
  - Alerta se worker parado > 5 minutos
  - Métricas de `webhook_worker_last_processed_timestamp`
- **Plano de contingência:** Restart manual do worker; eventos eventualmente são processados (não há perda)

### Risco 2: Cliente offline por mais de 15 horas
- **Probabilidade:** Baixa
- **Impacto:** Eventos vão para DLQ após esgotar 5 retries; cliente perde notificação
- **Mitigação:**
  - Documentação clara da política de retry (15h de cobertura)
  - Cliente pode monitorar histórico de entregas via API
  - Endpoint de replay manual para DLQ
- **Plano de contingência:** Admin faz replay manual dos eventos via POST /admin/webhooks/dead-letter/:id/replay

### Risco 3: Hot keys na outbox degradam performance do MySQL
- **Probabilidade:** Baixa
- **Impacto:** Latência da transação de changeStatus aumenta; possível bloqueio
- **Mitigação:**
  - Índice composto `(status, created_at)` otimizado para queries do worker
  - Batch processing limitado (50 eventos/ciclo)
  - Monitoramento de locks do MySQL
- **Plano de contingência:** Aumentar frequência de arquivamento de eventos entregues

### Risco 4: Cliente não implementa deduplicação corretamente
- **Probabilidade:** Média
- **Impacto:** Cliente processa mesmo evento múltiplas vezes (ex: envia email duplicado ao comprador final)
- **Mitigação:**
  - Documentação detalhada e destacada sobre at-least-once
  - Exemplos de código para deduplicação por `X-Event-Id`
  - Marcos inclui seção específica no portal de desenvolvedor
- **Plano de contingência:** Suporte direto aos clientes que reportarem problema

### Risco 5: Vazamento de secret
- **Probabilidade:** Baixa
- **Impacto:** Atacante pode forjar webhooks para cliente específico
- **Mitigação:**
  - Secret por endpoint (isola impacto)
  - Endpoint de rotação via API
  - Grace period de 24h permite cliente migrar sem downtime
  - Cliente histórico de vazamento em log ([09:22] Diego) reforça importância da rotação
- **Plano de contingência:** Cliente rotaciona secret imediatamente; secret antiga expirada manualmente se necessário

---

## Critérios de Aceitação

### Funcional
- [ ] Cliente B2B pode cadastrar webhook via POST /api/v1/webhooks
- [ ] Cliente B2B recebe secret apenas na criação (não em listagens)
- [ ] Sistema envia POST HTTP quando status de pedido muda
- [ ] Payload contém `event_id`, `event_type`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`
- [ ] Headers incluem `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`
- [ ] Assinatura HMAC-SHA256 pode ser validada pelo cliente
- [ ] Sistema retenta automaticamente 5 vezes com backoff exponencial
- [ ] Eventos que esgotam retries vão para DLQ
- [ ] Admin pode reprocessar evento da DLQ via POST /admin/webhooks/dead-letter/:id/replay
- [ ] Cliente pode rotacionar secret via API mantendo antiga válida por 24h
- [ ] Cliente pode listar histórico de entregas via GET /api/v1/webhooks/:id/deliveries

### Performance
- [ ] Latência p95 de notificação < 10 segundos
- [ ] Worker processa batch de 50 eventos em < 5 segundos (clientes rápidos)

### Segurança
- [ ] URL não-HTTPS é rejeitada com erro de validação
- [ ] Secret tem >= 128 bits de entropia
- [ ] Replay de DLQ exige role ADMIN
- [ ] Audit log registra quem fez replay

### Negócio
- [ ] Atlas Comercial confirma que webhook atende necessidade (reduz polling)
- [ ] Taxa de sucesso de entregas (após retries) >= 99%

---

## Testes e Validação

### Tipos de Teste Obrigatórios

**Testes Unitários:**
- Lógica de cálculo HMAC-SHA256
- Lógica de backoff exponencial (próximo retry_at)
- Validação de URL (rejeitar HTTP)
- Filtro de status (evento criado apenas para status configurados)

**Testes de Integração:**
- Fluxo completo: changeStatus → outbox → worker → entrega
- Retry após falha (mockar cliente retornando 503)
- DLQ após 5 falhas
- Rotação de secret (validar ambas secrets por 24h)
- Deduplicação (mesmo event_id entregue duas vezes)

**Testes de Segurança:**
- Assinatura HMAC válida vs. inválida
- Replay de DLQ sem role ADMIN (deve falhar)
- Secret não aparece em logs nem em GET /webhooks

**Testes de Performance:**
- Latência de inserção na outbox dentro da transação (< 50ms)
- Latência de entrega ponta a ponta (< 10s p95)
- Worker processa 50 eventos em < 5s

**Testes End-to-End:**
- Cliente real (mock server) recebe webhook e valida assinatura
- Cliente offline, sistema retenta, cliente volta online, recebe evento

### Estratégia de Validação

- **TDD para lógica crítica:** HMAC, backoff, retry
- **QA manual guiado por roteiro:** Fluxo completo com servidor mock
- **Validação exploratória:** Testar edge cases (URL malformada, payload gigante, cliente ultra-lento)
- **Homologação com Atlas Comercial:** Beta test com cliente real antes do GA

---

## Referências

- **Transcrição da Reunião Técnica:** Timestamps indicados ao longo do documento
- **RFC:** `docs/RFC.md`
- **FDD:** `docs/FDD.md`
- **ADRs:** `docs/adrs/ADR-001.md` a `docs/adrs/ADR-006.md`
