# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint

## Status
Aceito

## Contexto

O sistema enviará eventos com dados de pedidos (order_id, customer_id, total_cents) para endpoints fora da infraestrutura controlada. O cliente precisa:
1. Validar que a requisição veio realmente do nosso sistema
2. Detectar se o payload foi adulterado no meio do caminho

Isso é um requisito de segurança fundamental para webhooks ([09:19] Sofia).

## Decisão

Implementar autenticação via **HMAC-SHA256** ([09:20] Sofia, [09:22] Sofia):

1. **Algoritmo**: HMAC-SHA256 sobre o corpo completo do request
2. **Secret por endpoint**: Cada endpoint webhook tem sua própria secret única ([09:21] Sofia)
3. **Header**: Assinatura enviada em `X-Signature` ([09:20] Sofia)
4. **Rotação**: Secret é rotacionável via API; após rotação, secret antiga permanece válida por **24 horas** em paralelo ([09:21] Sofia)

## Alternativas Consideradas

### Alternativa 1: Secret global da plataforma
- **Descartada** por Sofia ([09:21])
- **Razão**: Se uma secret vaza (ex: cliente expõe em log, [09:22] Diego), todos os endpoints ficam comprometidos
- **Trade-off rejeitado**: Simplicidade vs. risco de segurança em escala

### Alternativa 2: JWT ou OAuth
- **Não mencionada explicitamente**, mas implicitamente descartada
- **Razão provável**: Webhooks são push unidirecional; HMAC é padrão de mercado mais simples para esse caso de uso

## Consequências

### Positivas
- **Isolamento de vazamento**: Se secret de um cliente vaza, apenas aquele endpoint é afetado ([09:21] Sofia)
- **Padrão de mercado**: HMAC-SHA256 é amplamente adotado (Stripe, GitHub, etc.), todo cliente sério tem biblioteca ([09:20] Sofia)
- **Rotação segura**: Grace period de 24h permite migração sem downtime ([09:21] Sofia)
- **Integridade**: Cliente detecta adulteração de payload

### Negativas
- **Complexidade de gestão**: Sistema precisa armazenar e gerenciar múltiplas secrets
- **Responsabilidade do cliente**: Cliente precisa implementar verificação HMAC do lado dele
- **Grace period requer lógica dupla**: Durante rotação, sistema valida contra duas secrets por 24h

### Riscos e Mitigação
- **Risco**: Cliente expõe secret em log ou repositório
  - **Mitigação**: Suporte a rotação de secret via API ([09:21] Sofia)
  - **Histórico real**: Time já teve cliente que vazou secret em log de aplicação ([09:22] Diego)
- **Risco**: Replay attack
  - **Mitigação adicional**: Header `X-Timestamp` permite cliente detectar requisições antigas sendo reprocessadas ([09:44] Diego)

## Detalhes de Implementação

### Armazenamento de Secret
- Tabela de configuração de webhook armazena: `url`, `secret`, `customer_id`, `estado ativo` ([09:21] Bruno)
- Secret gerada pelo sistema e devolvida apenas na criação do webhook ([09:31] Marcos)
- **Importante**: Secret nunca é retornada em endpoints de listagem (apenas na criação)

### Endpoint de Rotação
- Cliente pode requisitar nova secret via API ([09:21] Sofia)
- Sistema gera nova secret, mantém antiga ativa por 24h
- Após 24h, antiga secret é invalidada automaticamente

### Cálculo do HMAC
```
HMAC-SHA256(secret, request_body)
```
- Input: corpo JSON completo do request (antes de qualquer parsing)
- Output: assinatura hexadecimal em `X-Signature`

### Headers de Segurança
- `X-Signature`: assinatura HMAC-SHA256 ([09:20] Sofia)
- `X-Timestamp`: timestamp do envio, para detecção de replay attack ([09:44] Diego)
- `X-Event-Id`: UUID único do evento, para deduplicação ([09:25] Diego, ver ADR-004)

### Validação de URL
- TLS obrigatório: URL deve ser `https://` ([09:23] Sofia)
- Schema Zod rejeita `http://` com erro de validação ([09:23] Sofia)

## Integração com Código Existente

Reutiliza padrões de:
- Validação com Zod: `src/modules/*/schemas.ts`
- Geração de UUID: biblioteca `uuid` já presente no projeto

## Referências
- Transcrição: [09:19] Sofia, [09:20] Sofia, [09:21] Sofia, [09:21] Bruno, [09:22] Sofia, [09:22] Diego, [09:23] Sofia, [09:31] Marcos, [09:44] Diego
- Código: Padrão de schemas Zod em `src/modules/*/schemas.ts`
