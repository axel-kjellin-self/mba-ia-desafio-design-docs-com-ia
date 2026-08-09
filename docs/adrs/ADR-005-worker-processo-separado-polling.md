# ADR-005: Worker em Processo Separado com Polling de 2 Segundos

## Status
Aceito

## Contexto

Eventos persistidos na tabela `webhook_outbox` precisam ser processados e enviados para os endpoints dos clientes. Três decisões inter-relacionadas precisam ser tomadas:
1. **Onde o worker roda**: dentro da API ou processo separado?
2. **Como o worker detecta novos eventos**: polling ou notificação reativa?
3. **Qual a frequência de leitura**: intervalo do polling?

## Decisão

Implementar worker como **processo Node.js separado** ([09:11] Diego, [09:11] Larissa) que faz **polling a cada 2 segundos** ([09:09] Diego, [09:10] Larissa).

### Estrutura
- Entry-point separada: `src/worker.ts` ([09:11] Larissa)
- Script npm: `npm run worker` ([09:11] Larissa)
- Worker conecta no **mesmo banco** MySQL usando **Prisma Client separado** (mesma DATABASE_URL, instância nova do PrismaClient) ([09:11] Diego, [09:30] Bruno)

### Lógica de Polling
- Loop infinito com `await sleep(2000)` entre iterações
- Busca eventos com `status = 'pendente'` ordenados por `created_at ASC`
- Processa em **batch pequeno** (ex: 10-50 eventos por ciclo) ([09:08] Diego)
- Após processar, marca como `entregue` ou incrementa retry counter

## Alternativas Consideradas

### Alternativa 1: Worker dentro da mesma instância da API
- **Descartada** por Diego ([09:11])
- **Razão**: Se a API reinicia (deploy, crash, scaling), o worker para junto. Eventos ficam sem processamento durante restart.
- **Trade-off rejeitado**: Simplicidade de deploy vs. resiliência do processamento

### Alternativa 2: Trigger do MySQL para notificação reativa
- **Proposta** por Bruno ([09:09])
- **Descartada** por Diego ([09:09])
- **Razão técnica**: MySQL não tem `NOTIFY/LISTEN` nativo como PostgreSQL. Triggers MySQL executam apenas SQL, não notificam processos externos.
- **Workarounds considerados e rejeitados**: Escrever em arquivo ou bater em endpoint HTTP via trigger "fica esquisito" ([09:09] Diego)

### Alternativa 3: Intervalo de polling menor (< 2s)
- **Não proposta explicitamente**
- **Razão implícita**: Requisito de negócio aceita delay de até 10s ([09:02] Marcos). Polling de 2s atende confortavelmente (pior caso = 2s de latência) ([09:10] Larissa)

## Consequências

### Positivas
- **Resiliência**: API pode reiniciar sem afetar processamento de webhooks
- **Isolamento de recursos**: CPU do worker não compete com CPU da API
- **Simplicidade**: Polling é trivial de implementar e debugar
- **Latência aceitável**: 2s atende requisito de "< 10s" com margem confortável ([09:10] Marcos: "2 segundos serve, perfeito")

### Negativas
- **Latência mínima de 2s**: Evento pode esperar até 2s na outbox antes de ser processado (pior caso)
- **CPU de polling**: Worker consome CPU mesmo quando não há eventos (mitigado por sleep)
- **Não é tempo real**: Comparado com soluções event-driven (Kafka, Redis Streams), tem latência maior

### Riscos e Mitigação
- **Risco**: Worker crashar ou parar
  - **Mitigação futura**: Process manager (PM2, systemd) ou orquestrador (Kubernetes) para restart automático
  - **Observabilidade**: Métricas de "último evento processado" e alertas se worker parar
- **Risco**: Acúmulo de eventos se worker não acompanhar a taxa de produção
  - **Mitigação**: Índices em `status` e `created_at` para leitura rápida ([09:08] Diego)
  - **Escalabilidade futura**: Múltiplos workers com particionamento por `order_id` ([09:13] Diego, fora de escopo atual)

## Detalhes de Implementação

### Índices na Tabela
- `(status, created_at)`: para query eficiente de `WHERE status = 'pendente' ORDER BY created_at ASC LIMIT N`
- Permite worker encontrar próximo batch rapidamente mesmo com milhões de eventos entregues

### Batch Processing
- Busca N eventos (ex: 50)
- Processa sequencialmente (garante ordem por `order_id` implicitamente, ver limitação conhecida)
- Marca como processado
- Próximo ciclo busca o próximo batch

### Limitação Conhecida: Ordenação
- **Single worker atual**: eventos de um mesmo `order_id` são entregues em ordem (garantia implícita) ([09:12] Diego)
- **Múltiplos workers futuro**: perde garantia de ordem global ([09:13] Diego)
- **Documentado como limitação aceitável** ([09:13] Larissa)
- **Solução futura se necessário**: particionamento por `order_id` ou lock pessimista ([09:13] Diego)

## Integração com Código Existente

### PrismaClient Separado
- `src/worker.ts` instancia novo `PrismaClient` ([09:30] Bruno)
- Mesmo `DATABASE_URL` que a API
- Pool de conexões independente
- **Razão**: PrismaClient é por processo ([09:30] Bruno)

### Reutilização
- Logger Pino: mesma configuração
- Error handling: mesmas classes de erro
- Ambiente: mesmas variáveis de ambiente

## Referências
- Transcrição: [09:02] Marcos, [09:08] Diego, [09:09] Diego, [09:09] Bruno, [09:09] Diego, [09:10] Larissa, [09:10] Marcos, [09:11] Diego, [09:11] Larissa, [09:11] Bruno, [09:11] Diego, [09:12] Diego, [09:13] Diego, [09:13] Larissa, [09:30] Bruno
- Código: Entry-point existente em `src/server.ts` (padrão a seguir)
