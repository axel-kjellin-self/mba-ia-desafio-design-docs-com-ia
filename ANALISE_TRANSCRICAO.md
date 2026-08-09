# Análise Estruturada da Transcrição
## Sistema de Webhooks de Notificação de Pedidos

**Data da reunião:** quinta-feira, 09:00
**Duração:** ~55 minutos
**Participantes:** Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Eng. Segurança)

---

## 1. CONTEXTO E MOTIVAÇÃO

### Problema de Negócio
- **[09:00] Marcos**: Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitam notificação em tempo real de mudanças de status de pedidos
- **[09:00] Marcos**: Clientes atualmente fazem polling no GET /orders, tornando a integração lenta e cara
- **[09:00] Marcos**: Atlas ameaça migrar para concorrente se não entregar até fim do trimestre
- **[09:02] Marcos**: "Tempo real" = delay aceitável abaixo de 10 segundos
- **[09:02] Marcos**: Apenas outbound webhooks (sistema → clientes), não inbound

### Público-alvo
- **[09:00] Marcos**: Clientes B2B específicos (Atlas Comercial, MaxDistribuição, Nova Cargo)
- Expandível para outros clientes B2B futuros

---

## 2. DECISÕES ARQUITETURAIS FECHADAS
### (Candidatas a ADRs)

### Decisão 1: Padrão Outbox no MySQL
- **[09:04] Bruno**: Descarta abordagem síncrona - transação já pesada (orders + history + stock)
- **[09:04] Bruno**: Cliente lento travaria mudança de status de outros pedidos
- **[09:04] Bruno**: Se cliente offline, não dá para fazer rollback da mudança de status
- **[09:06] Diego**: "Síncrono está fora de questão. O que a gente quer é padrão outbox"
- **[09:06] Diego**: Insere evento na tabela webhook_outbox dentro da mesma transação SQL que atualiza orders/history
- **[09:06] Diego**: Garante atomicidade - se transação commita, evento foi registrado; se rollback, evento some junto
- **[09:07] Larissa**: Alternativa seria Redis Streams, mas requer mais infra
- **[09:07] Diego**: Time pequeno, Redis Cluster seria overengineering
- **[09:08] Larissa**: "Tá decidido então: outbox em MySQL"
- **Trade-off**: Simplicidade e garantias ACID vs. possível latência maior que soluções dedicadas

### Decisão 2: Worker em Processo Separado com Polling
- **[09:09] Diego**: Polling em loop a cada 2 segundos
- **[09:09] Bruno**: Pergunta sobre trigger do banco para ser mais reativo
- **[09:09] Diego**: MySQL não tem NOTIFY/LISTEN como Postgres; trigger não notifica processo externo
- **[09:09] Diego**: Polling de 2s atende requisito de "abaixo de 10 segundos"
- **[09:10] Larissa**: "Worker em polling, 2s. Latência mínima 2s no pior caso. Aceitamos."
- **[09:11] Diego**: Worker DEVE rodar como processo separado, não dentro da API
- **[09:11] Larissa**: Entry-point separada (src/worker.ts) com script "npm run worker"
- **[09:11] Diego**: Mesmo banco, mesma stack, só não pode ser mesmo processo
- **Trade-off**: Simplicidade vs. latência ligeiramente maior que soluções reativas

### Decisão 3: Política de Retry com Backoff Exponencial e DLQ
- **[09:15] Diego**: Backoff exponencial com teto de tentativas
- **[09:15] Diego**: Sugere 5 tentativas (não indefinido)
- **[09:16] Bruno**: Questiona se 3 não seria melhor
- **[09:16] Diego**: 3 é pouco - cliente com manutenção de 2h perderia as tentativas
- **[09:17] Larissa**: "Cinco fica bom"
- **[09:17] Diego**: Progressão: 1min, 5min, 30min, 2h, 12h (total ~15h entre primeira e última)
- **[09:17] Larissa**: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h"
- **[09:18] Diego**: DLQ em tabela separada webhook_dead_letter com payload, motivo e timestamp
- **[09:18] Diego**: Mantém outbox principal limpa e facilita debug/reprocessamento
- **Trade-off**: Cobre janela de 15h de indisponibilidade vs. eventos podem ser perdidos após esse período

### Decisão 4: Autenticação HMAC-SHA256 com Secret por Endpoint
- **[09:20] Sofia**: HMAC para assinar payload com secret compartilhada
- **[09:20] Sofia**: Assinatura em header X-Signature
- **[09:20] Sofia**: HMAC-SHA256 é padrão de mercado
- **[09:21] Sofia**: **Secret única por endpoint**, não global (se vaza uma, não vaza tudo)
- **[09:21] Sofia**: Secret rotacionável via API
- **[09:21] Sofia**: Após rotação, antiga válida por 24h em paralelo (grace period)
- **[09:22] Sofia**: "Decidido: HMAC-SHA256 sobre corpo do request, secret por endpoint, suporte a rotação com grace period de 24h"
- **Trade-off**: Segurança robusta vs. complexidade de gestão de múltiplas secrets

### Decisão 5: Garantia At-Least-Once com X-Event-Id
- **[09:24] Diego**: Garantia at-least-once (cliente pode receber evento duas vezes)
- **[09:25] Diego**: X-Event-Id com UUID gerado na inserção do evento na outbox
- **[09:25] Diego**: Cliente deduplica pelo event_id
- **[09:25] Sofia**: "Isso joga responsabilidade pro cliente"
- **[09:25] Diego**: Padrão de mercado (Stripe, GitHub); exactly-once é muito complexo
- **[09:26] Larissa**: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."
- **Trade-off**: Simplicidade vs. cliente precisa implementar deduplicação

### Decisão 6: Reuso dos Padrões Existentes do Projeto
- **[09:27] Bruno**: Módulo src/modules/webhooks seguindo padrão controller/service/repository/routes/schemas
- **[09:28] Bruno**: Erros com classe AppError e códigos prefixados WEBHOOK_*
- **[09:29] Bruno**: Logger Pino mantido, middleware de erro centralizado já trata AppError/Zod/Prisma
- **[09:29] Larissa**: "Decisão: reuso máximo. AppError, Pino, error middleware, padrão de módulos, schemas Zod, códigos de erro"
- **[09:30] Bruno**: Worker abre PrismaClient separado (mesmo banco, instância nova)
- **Trade-off**: Consistência e velocidade de desenvolvimento vs. nenhum, é a melhor escolha

### Decisão 7: Ordenação Implícita por order_id (Single Worker)
- **[09:12] Larissa**: Pergunta sobre ordering de eventos sequenciais
- **[09:12] Diego**: Com single worker, processa em ordem de created_at do outbox → cliente recebe em ordem
- **[09:13] Diego**: Com múltiplos workers futuro, perde garantia (mas problema do futuro)
- **[09:13] Larissa**: "Documentamos como limitação conhecida. Não é garantia global, só por order_id enquanto single-worker"
- **Trade-off**: Simplicidade imediata vs. limitação futura de escala

### Decisão 8: Filtro de Eventos na Inserção
- **[09:33] Marcos**: Cliente escolhe quais status quer receber por endpoint
- **[09:34] Diego**: Pergunta se filtra na inserção ou no envio
- **[09:34] Bruno**: "Na inserção. Se nenhum webhook quer aquele status, nem insere"
- **[09:34] Diego**: Concorda - economiza linha na tabela

### Decisão 9: Payload como Snapshot na Inserção
- **[09:51] Bruno**: Dúvida se payload é renderizado já ou renderiza na hora do envio
- **[09:52] Larissa**: "Prefiro renderizado já, na hora da inserção. Se pedido mudar depois, evento reflete estado de quando status mudou"
- **[09:52] Diego**: Concorda - snapshot na inserção
- **[09:52] Bruno**: "Beleza, snapshot. Decidido."

### Decisão 10: UUIDs para IDs (Consistência com Projeto)
- **[09:51] Diego**: Pergunta sobre id auto-incremental ou UUID na outbox
- **[09:51] Larissa**: "UUID, segue padrão do resto do projeto. Tudo é uuid"

---

## 3. REQUISITOS FUNCIONAIS

### RF-01: Cadastro de Webhook
- **[09:31] Marcos**: Cliente cadastra webhook via POST
- **[09:31] Marcos**: Campos: url, secret gerada pelo sistema e devolvida na criação
- **[09:31] Marcos**: Lista de status que cliente quer receber
- **[09:31] Marcos**: customer_id implícito (não vem do JWT)
- **[09:32] Bruno/Larissa**: customer_id passado no body ou path, não vem do JWT
- **[09:32] Larissa**: Endpoint autenticado normal com JWT do sistema
- **Prioridade**: Alta

### RF-02: Edição de Webhook
- **[09:33] Bruno**: PATCH para editar webhook existente
- **Prioridade**: Alta

### RF-03: Remoção de Webhook
- **[09:33] Bruno**: DELETE para remover webhook
- **Prioridade**: Alta

### RF-04: Listagem de Webhooks
- **[09:33] Bruno**: GET para listar webhooks de um customer
- **Prioridade**: Alta

### RF-05: Filtro de Eventos por Status
- **[09:33] Marcos**: Por endpoint, cliente escolhe quais status quer ouvir
- **[09:33] Marcos**: Exemplo: "só quero SHIPPED e DELIVERED"
- **[09:34] Bruno**: Filtra na inserção do outbox
- **Prioridade**: Alta

### RF-06: Histórico de Entregas
- **[09:34] Marcos**: Cliente visualiza últimos 100 webhooks enviados
- **[09:34] Marcos**: GET /webhooks/:id/deliveries
- **[09:34] Marcos**: Mostra sucesso/falha, payload, response, tempo de resposta
- **Prioridade**: Média

### RF-07: Reprocessamento Manual de DLQ (Admin)
- **[09:35] Diego**: POST /admin/webhooks/dead-letter/:id/replay
- **[09:35] Diego**: Recoloca evento na outbox como pendente
- **[09:36] Sofia**: Requer role ADMIN (não OPERATOR)
- **[09:36] Sofia**: Deve logar quem fez o replay (auditoria)
- **[09:36] Larissa**: Reutiliza requireRole existente
- **Prioridade**: Média

### RF-08: Rotação de Secret
- **[09:21] Sofia**: Endpoint para cliente pedir nova secret
- **[09:21] Sofia**: Antiga válida por 24h em paralelo, depois morre
- **Prioridade**: Média

### RF-09: Integração com OrderService.changeStatus
- **[09:40] Bruno**: Alteração crítica dentro do método changeStatus
- **[09:40] Bruno**: Inserir na webhook_outbox dentro da mesma transação
- **[09:41] Bruno**: Se outbox falhar, rollback completo
- **[09:41] Bruno**: Função publishWebhookEvent(tx, order, fromStatus, toStatus)
- **[09:41] Diego**: Função pura recebendo tx client
- **Prioridade**: Alta (bloqueante)

---

## 4. REQUISITOS NÃO FUNCIONAIS

### RNF-01: Latência de Notificação
- **[09:02] Marcos**: Delay aceitável < 10 segundos
- **[09:10] Larissa**: Latência mínima de 2s (pior caso do polling)
- **Meta**: p95 < 10s, típico 2-4s

### RNF-02: Disponibilidade e Resiliência
- **[09:15] Diego**: 5 tentativas com backoff exponencial
- **[09:17] Diego**: Cobertura de ~15h de indisponibilidade do cliente
- **[09:18] Diego**: DLQ para eventos que falharam definitivamente

### RNF-03: Segurança
- **[09:20] Sofia**: HMAC-SHA256 para autenticação
- **[09:21] Sofia**: Secret única por endpoint
- **[09:21] Sofia**: Rotação de secret com grace period de 24h
- **[09:23] Sofia**: TLS obrigatório (HTTPS only)
- **[09:23] Sofia**: Rejeitar URLs http:// com erro de validação

### RNF-04: Integridade e Atomicidade
- **[09:06] Diego**: Garantia ACID via outbox pattern
- **[09:40] Bruno**: Evento não pode sair se mudança de status falhar
- **[09:41] Diego**: "Essencial. Se ficar fora da transação, perde a garantia toda"

### RNF-05: Observabilidade
- **[09:29] Bruno**: Logger Pino reutilizado
- **[09:29] Bruno**: Error middleware centralizado
- Implícito: métricas, logs estruturados, tracing

### RNF-06: Timeouts e Limites
- **[09:24] Diego**: Limite de payload 64KB
- **[09:24] Larissa**: Erro se ultrapassar (não truncar)
- **[09:42] Diego**: Timeout HTTP do worker: 10s
- **[09:42] Diego**: Cliente que não responde em 10s = falha, marca para retry

### RNF-07: Retenção e Limpeza
- **[09:08] Diego**: Linhas entregues arquivadas após 30 dias (fora de escopo da feature)
- **[09:08] Diego**: Índices em status e created_at

### RNF-08: Performance do Worker
- **[09:08] Diego**: Worker lê eventos pendentes em batch pequeno
- **[09:08] Diego**: Processa, marca como entregue
- **[09:08] Diego**: Índice em status (pendente/processando/falhou/entregue) + created_at

---

## 5. ALTERNATIVAS CONSIDERADAS E DESCARTADAS

### Alt-01: Webhook Síncrono
- **[09:04] Bruno**: "Síncrono não rola. Transação já é pesada"
- **[09:04] Bruno**: Cliente lento travaria mudança de status
- **[09:04] Bruno**: Cliente offline forçaria rollback da mudança de status
- **[09:06] Diego**: "Síncrono está fora de questão"
- **Razão do descarte**: Performance, resiliência e semântica de negócio

### Alt-02: Redis Streams ou Fila Dedicada
- **[09:07] Larissa**: Alternativa seria Redis Streams
- **[09:07] Diego**: "Acabaria precisando subir mais infra"
- **[09:07] Diego**: Time pequeno, Redis Cluster é overengineering
- **Razão do descarte**: Complexidade operacional vs. benefício marginal

### Alt-03: Trigger do MySQL para Reatividade
- **[09:09] Bruno**: Pergunta sobre trigger do banco
- **[09:09] Diego**: MySQL não tem NOTIFY/LISTEN nativo
- **[09:09] Diego**: Trigger só executa SQL, não notifica processo externo
- **Razão do descarte**: Limitação técnica do MySQL

### Alt-04: 3 Tentativas de Retry
- **[09:16] Bruno**: Sugere 3 tentativas (mais agressivo)
- **[09:16] Diego**: 3 é pouco - manutenção de 2h do cliente perderia todas
- **Razão do descarte**: Não cobre janela realista de manutenção

### Alt-05: Retry Indefinido
- **[09:15] Diego**: "Algumas pessoas defendem retry indefinido com backoff"
- **[09:15] Diego**: Problema: evento fica pendurado para sempre se cliente sumiu
- **Razão do descarte**: Eventos podem ficar pendurados indefinidamente

---

## 6. QUESTÕES EM ABERTO / ADIADAS

### QA-01: Rate Limiting de Envio
- **[09:38] Diego**: "Rate limiting de envio pra cliente?"
- **[09:38] Diego**: Cenário: 50 pedidos mudando status em 1 minuto = 50 chamadas
- **[09:39] Larissa**: "Faz parte do escopo?"
- **[09:39] Diego**: "Eu acho que não. A gente observa e implementa se virar problema"
- **[09:39] Larissa**: "Fica como 'observar e decidir depois'"
- **Status**: Observar em produção, decidir depois

### QA-02: Escala com Múltiplos Workers
- **[09:13] Bruno**: "E se algum dia a gente quiser escalar?"
- **[09:13] Diego**: "Dá pra particionar por order_id, ou usar lock pessimista"
- **[09:13] Diego**: "Mas isso é problema do futuro, não agora"
- **Status**: Futuro, fora de escopo atual

### QA-03: Ordenação Global de Eventos
- **[09:12] Larissa**: Pergunta sobre ordering de eventos sequenciais
- **[09:13] Larissa**: "Documentamos como limitação conhecida"
- **Status**: Limitação aceita, não é garantia nesta fase

---

## 7. EXPLICITAMENTE FORA DE ESCOPO

### FS-01: Email de Fallback/Notificação de Problemas
- **[09:37] Marcos**: "Tem como avisar cliente quando webhook tá com problema? Email?"
- **[09:37] Larissa**: "Não. Email tá fora de escopo dessa fase"
- **[09:37] Larissa**: "Talvez próxima fase, depois que a gente medir o impacto"
- **[09:38] Marcos**: Anotado como "futuro"

### FS-02: Dashboard Visual
- **[09:39] Marcos**: "Dashboard visual? Painel pro cliente ver webhooks dele?"
- **[09:40] Larissa**: "Não, agora não. Só endpoints"
- **[09:40] Larissa**: "Painel é projeto separado do time de frontend"

### FS-03: Inbound Webhooks
- **[09:02] Sofia**: "Os webhooks vão sair só do nosso sistema pra eles, ou eles também enviam pra gente?"
- **[09:02] Marcos**: "Só saindo da gente pra eles. Eles querem receber, não mandar"

### FS-04: Gestão Dinâmica de Políticas (UI/Console)
- Não mencionado, mas implícito: sem UI para editar regras em tempo real

### FS-05: Persistência Durável Avançada
- **[09:08] Diego**: Arquivamento de eventos entregues após 30 dias "fora do escopo dessa feature"

---

## 8. DETALHES TÉCNICOS IMPORTANTES

### Formato do Payload
- **[09:43] Diego**: JSON com:
  - event_id (UUID)
  - event_type: "order.status_changed"
  - timestamp (ISO 8601)
  - order_id
  - order_number
  - from_status
  - to_status
  - customer_id
  - total_cents (campos básicos)
- **[09:43] Diego**: NÃO manda items (para não inflar payload)
- **[09:43] Diego**: Cliente bate em GET /orders/:id se quiser detalhes

### Headers HTTP Enviados
- **[09:44] Diego**: X-Event-Id (UUID)
- **[09:44] Diego**: X-Signature (HMAC)
- **[09:44] Diego**: X-Timestamp (timestamp do envio, para detectar replay attack)
- **[09:44] Diego**: Content-Type: application/json
- **[09:44] Sofia**: X-Webhook-Id (id do endpoint webhook, para cliente com múltiplos)

### Estrutura da Tabela de Configuração
- **[09:21] Bruno**: url + secret + customer_id + estado ativo
- **[09:33] Marcos**: + lista de status que cliente quer receber

### Índices e Performance
- **[09:08] Diego**: Índice em status (pendente/processando/falhou/entregue)
- **[09:08] Diego**: Índice em created_at
- **[09:08] Diego**: Worker lê pendentes em batch pequeno

### Códigos de Erro
- **[09:28] Bruno**: Prefixo WEBHOOK_ para todos os erros
- **[09:28] Bruno**: Exemplos: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED

### Validações
- **[09:23] Sofia**: URL deve ser https (validação Zod)
- **[09:24] Diego**: Payload máximo 64KB
- **[09:24] Larissa**: Erro se ultrapassar (não truncar)

---

## 9. TRADE-OFFS IDENTIFICADOS

### T-01: Outbox MySQL vs Redis Streams
- **Escolha**: Outbox MySQL
- **Ganho**: Simplicidade, sem infra adicional, garantias ACID
- **Custo**: Possível latência maior, limite de throughput do MySQL

### T-02: Polling vs Event-Driven
- **Escolha**: Polling de 2s
- **Ganho**: Simplicidade, compatibilidade com MySQL
- **Custo**: Latência mínima de 2s, CPU de polling constante

### T-03: 5 Retries vs 3 Retries
- **Escolha**: 5 retries com backoff 1m/5m/30m/2h/12h
- **Ganho**: Cobre janela de 15h de indisponibilidade
- **Custo**: Eventos podem ficar pendentes por muito tempo

### T-04: At-Least-Once vs Exactly-Once
- **Escolha**: At-least-once
- **Ganho**: Simplicidade, padrão de mercado
- **Custo**: Cliente precisa implementar deduplicação

### T-05: Secret por Endpoint vs Secret Global
- **Escolha**: Secret por endpoint
- **Ganho**: Segurança (vazamento isolado)
- **Custo**: Complexidade de gestão

### T-06: Snapshot vs Renderização Sob Demanda
- **Escolha**: Snapshot na inserção
- **Ganho**: Evento reflete estado exato do momento da mudança
- **Custo**: Payload duplicado se pedido não mudar

---

## 10. PRAZOS E ESTIMATIVAS

### Estimativa de Implementação
- **[09:46] Larissa**: 3 sprints total
  - Modelagem outbox + DLQ: 1 sprint
  - Worker + retry: 1 sprint
  - CRUD configuração + deliveries: 0.5 sprint
  - Integração order.service + testes E2E: 0.5 sprint
  - HMAC, schemas, validações: incluído
  - Revisão de segurança Sofia: 2 dias úteis no fim

### Deadline
- **[09:45] Marcos**: Atlas quer até fim de novembro
- **[09:47] Larissa**: 3 sprints com revisão Sofia incluída

---

## 11. DEPENDÊNCIAS E INTEGRAÇÕES

### Código Existente que Será Modificado
- **[09:40] Bruno**: src/modules/orders/order.service.ts - método changeStatus
- **[09:41] Bruno**: Inserir na webhook_outbox dentro da transação existente

### Padrões Reutilizados
- **[09:28] Bruno**: Estrutura de módulos (controller/service/repository/routes/schemas)
- **[09:28] Bruno**: Classes de erro (AppError, InsufficientStockError, etc)
- **[09:28] Bruno**: Códigos de erro (UPPERCASE_SNAKE_CASE)
- **[09:29] Bruno**: Logger Pino
- **[09:29] Bruno**: Error middleware centralizado
- **[09:29] Bruno**: Schemas Zod
- **[09:30] Bruno**: Prisma com mesmo DATABASE_URL

### Middleware de Autorização
- **[09:36] Larissa/Sofia**: Reutiliza requireRole existente para endpoint admin

---

## 12. AUTORIZAÇÕES E PERMISSÕES

### CRUD de Webhooks
- **[09:36] Marcos**: Qualquer role autenticada (por enquanto)
- **[09:37] Sofia**: Futuramente pode endurecer

### Endpoint Admin (Replay DLQ)
- **[09:36] Sofia**: Requer role ADMIN obrigatório
- **[09:36] Sofia**: Logar quem fez replay (auditoria)

---

## 13. OBSERVABILIDADE E MONITORAMENTO

### Logs
- **[09:29] Bruno**: Pino reutilizado
- **[09:29] Bruno**: Error middleware centralizado já trata erros

### Auditoria
- **[09:36] Sofia**: Replay de DLQ deve logar quem executou
- Implícito: OrderStatusHistory já audita mudanças de status

---

## 14. RESUMO FINAL DA REUNIÃO

**[09:48] Larissa** - Resumo consolidado:
- ✅ Padrão outbox no MySQL, transação atômica com mudança de status
- ✅ Worker separado em polling de 2 segundos
- ✅ Retry com backoff exponencial 1m/5m/30m/2h/12h, 5 tentativas, depois DLQ
- ✅ DLQ persistida em tabela separada
- ✅ HMAC-SHA256 sobre payload, secret por endpoint, rotação com grace period 24h
- ✅ Idempotência por X-Event-Id, garantia at-least-once
- ✅ Padrões do projeto reaproveitados: AppError, Pino, error middleware, módulos, prefixo WEBHOOK_
- ✅ Endpoints CRUD autenticados normal, replay DLQ exige ADMIN
- ❌ Email fallback: próxima fase
- ❌ Rate limiting saída: observar
- ❌ Dashboard visual: fora de escopo
- ✅ Prazo: 3 sprints

**[09:49] Confirmação do time**: Todos aprovam

---

## MAPEAMENTO PARA DOCUMENTOS

### ADRs (5-8 documentos)
1. ADR-001: Padrão Outbox no MySQL
2. ADR-002: Política de Retry com Backoff e DLQ
3. ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint
4. ADR-004: Garantia At-Least-Once com X-Event-Id
5. ADR-005: Worker em Processo Separado em Polling
6. ADR-006: Reuso dos Padrões Existentes do Projeto
7. (Opcional) ADR-007: Snapshot de Payload na Inserção
8. (Opcional) ADR-008: Ordenação Implícita por order_id (Single Worker)

### PRD - Seções principais
- Problema: [09:00-09:02]
- Público: Clientes B2B
- Objetivos e métricas: Reduzir polling, melhorar integração
- Requisitos Funcionais: RF-01 a RF-09
- Requisitos Não Funcionais: RNF-01 a RNF-08
- Fora de Escopo: FS-01 a FS-05
- Riscos: Cliente offline prolongado, hot keys, etc.

### RFC - Seções principais
- Proposta: Outbox + Worker + Retry + HMAC
- Alternativas: Alt-01 a Alt-05
- Questões em Aberto: QA-01 a QA-03
- Decisões: Links para ADRs

### FDD - Seções principais
- Fluxos: Inserção outbox, worker polling, retry, DLQ
- Contratos: POST /webhooks, GET /webhooks/:id/deliveries, POST /admin/webhooks/dead-letter/:id/replay
- Payload e Headers: Seção 8
- Erros: Códigos WEBHOOK_*
- Integração com código existente: OrderService.changeStatus

### TRACKER
- Cada decisão, requisito e restrição mapeada para timestamp [hh:mm] Nome
- Integrações mapeadas para arquivos do código (src/modules/orders/order.service.ts, etc.)
