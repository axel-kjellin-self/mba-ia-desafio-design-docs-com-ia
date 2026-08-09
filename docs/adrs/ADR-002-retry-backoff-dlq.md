# ADR-002: Política de Retry com Backoff Exponencial e Dead Letter Queue

## Status
Aceito

## Contexto

Webhooks podem falhar por diversos motivos: cliente temporariamente offline, manutenção programada, timeout de rede, sobrecarga. O sistema precisa de uma política de retry que balance resiliência (dar oportunidade para o cliente se recuperar) com pragmatismo (não manter eventos pendurados indefinidamente).

## Decisão

Implementar retry com **backoff exponencial** usando **5 tentativas** ([09:17] Larissa) com a seguinte progressão:
- 1ª retry: 1 minuto após falha
- 2ª retry: 5 minutos
- 3ª retry: 30 minutos
- 4ª retry: 2 horas
- 5ª retry: 12 horas

Total: ~15 horas entre primeira falha e última tentativa ([09:17] Diego, [09:17] Marcos).

Após esgotar as 5 tentativas, o evento é movido para uma **Dead Letter Queue (DLQ)** em tabela separada `webhook_dead_letter` ([09:18] Diego).

## Alternativas Consideradas

### Alternativa 1: 3 tentativas (mais agressivo)
- **Proposta** por Bruno ([09:16])
- **Descartada** por Diego ([09:16])
- **Razão**: 3 tentativas em ~30 minutos é insuficiente. Clientes reais já tiveram manutenção programada de 2 horas. Com 3 tentativas, perderíamos o evento durante janelas normais de manutenção.

### Alternativa 2: Retry indefinido com backoff
- **Mencionada** por Diego ([09:15])
- **Descartada** ([09:15] Diego)
- **Razão**: Eventos ficariam pendurados para sempre se o cliente desativasse o endpoint ou saísse do ar permanentemente. Isso geraria acúmulo sem fim na tabela de outbox.

## Consequências

### Positivas
- **Cobertura de janelas de manutenção**: 15 horas cobre manutenções programadas e incidentes prolongados ([09:17] Marcos: "Se cair por 15h, já tá com problema sério dele")
- **Backoff exponencial**: Evita bombardear cliente em recuperação
- **DLQ explícita**: Eventos que falharam definitivamente ficam isolados para análise e reprocessamento manual
- **Evidência para debug**: DLQ armazena payload, motivo da falha e timestamp ([09:18] Diego)

### Negativas
- **Perda após 15h**: Se cliente ficar offline por mais de 15h, evento é perdido (mas considerado aceitável pelo PM)
- **Complexidade de reprocessamento**: DLQ requer endpoint admin para replay manual

### Riscos e Mitigação
- **Risco**: Cliente offline por período superior a 15h perde eventos
  - **Mitigação aceita**: Considerado problema grave do cliente ([09:17] Marcos)
  - **Contingência**: Endpoint admin para replay manual via `POST /admin/webhooks/dead-letter/:id/replay` ([09:18] Diego, [09:35] Diego)

## Detalhes de Implementação

### Estrutura da DLQ
- Tabela separada `webhook_dead_letter` ([09:18] Diego)
- Campos: payload completo, motivo da falha (último erro HTTP), timestamp da última tentativa
- **Benefício**: Mantém outbox principal limpa e otimizada para leitura de pendentes

### Endpoint de Replay
- `POST /admin/webhooks/dead-letter/:id/replay` ([09:35] Diego)
- Requer role `ADMIN` ([09:36] Sofia)
- Deve logar quem executou o replay para auditoria ([09:36] Sofia)
- Recoloca evento na outbox como `pendente`

### Timeout HTTP
- Cliente que não responde em **10 segundos** é tratado como falha ([09:42] Diego)
- Marca evento para retry conforme política acima

## Integração com Código Existente

Reutiliza padrão de autorização:
- Middleware `requireRole('ADMIN')` existente em `src/middlewares/auth.middleware.ts:49` ([09:36] Larissa)

## Referências
- Transcrição: [09:15] Diego, [09:16] Bruno, [09:16] Diego, [09:17] Larissa, [09:17] Diego, [09:17] Marcos, [09:18] Diego, [09:35] Diego, [09:36] Sofia, [09:42] Diego
- Código: `src/middlewares/auth.middleware.ts` (requireRole)
