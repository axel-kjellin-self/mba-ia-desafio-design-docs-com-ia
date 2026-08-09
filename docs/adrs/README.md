# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do Sistema de Webhooks de Notificação de Pedidos.

## Índice de Decisões

| ADR | Título | Status | Data Decisão |
|-----|--------|--------|--------------|
| [ADR-001](./ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL | Aceito | Reunião 09:00 |
| [ADR-002](./ADR-002-retry-backoff-dlq.md) | Política de Retry com Backoff Exponencial e DLQ | Aceito | Reunião 09:00 |
| [ADR-003](./ADR-003-hmac-sha256-por-endpoint.md) | Autenticação HMAC-SHA256 com Secret por Endpoint | Aceito | Reunião 09:00 |
| [ADR-004](./ADR-004-at-least-once-event-id.md) | Garantia At-Least-Once com X-Event-Id | Aceito | Reunião 09:00 |
| [ADR-005](./ADR-005-worker-processo-separado-polling.md) | Worker em Processo Separado com Polling de 2s | Aceito | Reunião 09:00 |
| [ADR-006](./ADR-006-reuso-padroes-projeto.md) | Reuso Máximo dos Padrões Existentes do Projeto | Aceito | Reunião 09:00 |

## Resumo das Decisões Principais

### Arquitetura
- **Outbox Pattern** (ADR-001): Eventos persistidos em tabela MySQL dentro da transação de mudança de status
- **Worker Separado** (ADR-005): Processo Node.js independente fazendo polling a cada 2 segundos

### Resiliência
- **Retry com Backoff** (ADR-002): 5 tentativas (1m/5m/30m/2h/12h), depois DLQ
- **At-Least-Once** (ADR-004): Cliente pode receber eventos duplicados, deduplica por X-Event-Id

### Segurança
- **HMAC-SHA256** (ADR-003): Secret única por endpoint, rotacionável com grace period de 24h

### Consistência
- **Reuso de Padrões** (ADR-006): Mesma estrutura de módulos, erros, logging e validação do projeto

## Formato dos ADRs

Todos os ADRs seguem o formato MADR (Markdown Any Decision Records):
- **Status**: Aceito, Proposto, Depreciado ou Supersedido
- **Contexto**: Problema e forças em jogo
- **Decisão**: O que foi decidido
- **Alternativas Consideradas**: Opções avaliadas e descartadas
- **Consequências**: Impactos positivos e negativos
- **Referências**: Timestamps da transcrição e arquivos de código

## Rastreabilidade

Todas as decisões são rastreáveis à reunião técnica do dia [data], com timestamps no formato `[hh:mm] Participante` ou a arquivos específicos do código existente.

## Relacionamento com Outros Documentos

- **RFC** (`docs/RFC.md`): Referencia estes ADRs na seção "Decisões Relacionadas"
- **FDD** (`docs/FDD.md`): Implementa as decisões descritas nestes ADRs
- **PRD** (`docs/PRD.md`): Decisões influenciam trade-offs e requisitos não-funcionais
- **TRACKER** (`docs/TRACKER.md`): Mapeia cada ADR às suas fontes na transcrição
