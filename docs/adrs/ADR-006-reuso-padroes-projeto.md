# ADR-006: Reuso Máximo dos Padrões Existentes do Projeto

## Status
Aceito

## Contexto

O projeto já possui padrões estabelecidos e funcionais para:
- Estrutura de módulos
- Tratamento de erros
- Logging
- Validação de entrada
- Middleware de autenticação
- Schema de banco de dados

A feature de webhooks precisa decidir entre:
1. Criar novos padrões específicos para webhooks
2. Reutilizar ao máximo os padrões existentes

## Decisão

**Reuso máximo dos padrões existentes** ([09:30] Larissa).

O módulo de webhooks seguirá exatamente os mesmos padrões do resto do projeto, sem inventar novos padrões ou trazer novas bibliotecas.

## Padrões Reutilizados

### 1. Estrutura de Módulos
- **Padrão existente**: `src/modules/{dominio}/` com `controller.ts`, `service.ts`, `repository.ts`, `routes.ts`, `schemas.ts` ([09:27] Bruno)
- **Aplicação**: Criar `src/modules/webhooks/` seguindo mesma estrutura
- **Arquivos do módulo**:
  ```
  src/modules/webhooks/
  ├── webhook.controller.ts
  ├── webhook.service.ts
  ├── webhook.repository.ts
  ├── webhook.routes.ts
  ├── webhook.schemas.ts
  └── webhook.worker.ts  (lógica de processamento)
  ```

### 2. Classes de Erro
- **Padrão existente**: Classe base `AppError` em `src/shared/errors/app-error.ts`
- **Padrão existente**: Classes específicas como `InsufficientStockError`, `InvalidStatusTransitionError` ([09:28] Bruno)
- **Aplicação**: Criar classes de erro específicas do webhook:
  - `WebhookNotFoundError`
  - `WebhookInvalidUrlError`
  - `WebhookSecretRequiredError`
  - Etc.

### 3. Códigos de Erro
- **Padrão existente**: `UPPERCASE_SNAKE_CASE` (ex: `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) em `src/shared/errors/http-errors.ts`
- **Aplicação**: Prefixo `WEBHOOK_` para todos os erros do módulo ([09:28] Bruno, [09:29] Larissa)
  - `WEBHOOK_NOT_FOUND`
  - `WEBHOOK_INVALID_URL`
  - `WEBHOOK_SECRET_REQUIRED`
  - `WEBHOOK_DELIVERY_FAILED`
  - Etc.

### 4. Logger
- **Padrão existente**: Pino em `src/shared/logger/index.ts` ([09:29] Bruno)
- **Configuração**: Redação de campos sensíveis, formato estruturado JSON
- **Aplicação**: Usar mesmo logger, sem trazer Winston, Bunyan ou outro

### 5. Middleware de Erro Centralizado
- **Padrão existente**: `src/middlewares/error.middleware.ts` trata `AppError`, `ZodError`, `Prisma.PrismaClientKnownRequestError` ([09:29] Bruno)
- **Benefício**: Erros do webhook são automaticamente formatados sem modificar o middleware

### 6. Validação com Zod
- **Padrão existente**: Schemas Zod em `*.schemas.ts` (ex: `src/modules/orders/order.schemas.ts`)
- **Padrão existente**: Middleware `validate.middleware.ts`
- **Aplicação**: Criar `webhook.schemas.ts` seguindo mesma estrutura

### 7. Middleware de Autenticação
- **Padrão existente**: `authenticate` e `requireRole(...roles)` em `src/middlewares/auth.middleware.ts` ([09:36] Larissa)
- **Aplicação**:
  - CRUD de webhooks: `authenticate` (qualquer role autenticada, [09:36] Marcos, [09:37] Sofia)
  - Replay de DLQ: `requireRole('ADMIN')` ([09:36] Sofia)

### 8. Prisma e UUIDs
- **Padrão existente**: Todas as entidades usam `@default(uuid())` em `prisma/schema.prisma`
- **Aplicação**: Tabelas `webhook_configuration`, `webhook_outbox`, `webhook_dead_letter` usam UUIDs ([09:51] Larissa)

### 9. Worker Entry-Point
- **Padrão existente**: `src/server.ts` como entry-point principal
- **Aplicação**: Criar `src/worker.ts` como entry-point separada ([09:11] Larissa)
- **Script**: `npm run worker` similar a `npm run dev` e `npm run start`

## Alternativas Consideradas

### Alternativa: Criar padrões novos específicos para webhooks
- **Não proposta formalmente**, mas implicitamente descartada
- **Razão**: Consistência é mais valiosa que otimização local
- **Trade-off**: Possíveis otimizações específicas vs. familiaridade e manutenibilidade

## Consequências

### Positivas
- **Velocidade de desenvolvimento**: Time já conhece os padrões ([09:30] Larissa)
- **Manutenibilidade**: Código de webhooks é familiar para qualquer desenvolvedor do projeto
- **Redução de bugs**: Padrões já testados e validados em produção
- **Onboarding**: Novos desenvolvedores não precisam aprender padrões diferentes por módulo
- **Code review mais fácil**: Revisores conhecem os padrões

### Negativas
- Nenhuma negativa significativa identificada. Reuso é universalmente benéfico neste contexto.

### Riscos
- **Risco teórico**: Padrões gerais podem não ser otimizados para caso de uso específico de webhooks
  - **Avaliação**: Não identificado nenhum conflito real entre padrões existentes e necessidades de webhook

## Detalhes de Implementação

### Integração no `app.ts`
Seguir mesmo padrão de inicialização dos outros módulos em `src/app.ts`:

```typescript
const webhookRepository = new WebhookRepository(prisma);
const webhookService = new WebhookService(webhookRepository);
const webhookController = new WebhookController(webhookService);
```

### Dependency Injection
- Manual via constructors (padrão do projeto)
- Sem IoC container

## Integração com Código Existente

**Arquivos de referência** para seguir os padrões:
- Estrutura de módulo: `src/modules/orders/` (controller, service, repository, routes, schemas)
- Classes de erro: `src/shared/errors/http-errors.ts`
- Logger: `src/shared/logger/index.ts`
- Middleware de auth: `src/middlewares/auth.middleware.ts`
- Error middleware: `src/middlewares/error.middleware.ts`
- Schemas Zod: `src/modules/orders/order.schemas.ts`
- Entry-point: `src/server.ts`

## Referências
- Transcrição: [09:27] Bruno, [09:28] Bruno, [09:28] Bruno, [09:29] Larissa, [09:29] Bruno, [09:30] Larissa, [09:36] Larissa, [09:36] Marcos, [09:36] Sofia, [09:37] Sofia, [09:51] Larissa
- Código:
  - `src/app.ts` (padrão de dependency injection)
  - `src/modules/orders/` (estrutura de módulo)
  - `src/shared/errors/` (classes de erro)
  - `src/middlewares/` (auth e error middleware)
  - `prisma/schema.prisma` (padrão de UUIDs)
