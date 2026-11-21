# ADR-007 — Snapshot do payload renderizado na inserção da outbox

## Status

Aceito

## Contexto

A linha da outbox pode guardar o evento de duas formas: apenas uma referência (`order_id` mais os status) e renderizar o payload no momento do envio, ou o JSON já renderizado no momento em que o evento é registrado.

A diferença aparece quando o pedido muda entre a inscrição na outbox e o envio pelo worker. Se o evento guardar só a referência, e o cliente editar o pedido no meio do caminho, o payload entregue refletiria o estado novo — que não é o estado que disparou o evento.

A decisão de identificar os eventos por UUID (e não auto-incremento) foi tomada no mesmo bloco da conversa e fica registrada aqui porque afeta a chave do evento.

## Decisão

1. Cada linha da outbox guarda o **payload JSON já renderizado** no momento da inserção — **snapshot**, e não referência.
2. O evento é identificado por **UUID** (`@default(uuid())`), seguindo o padrão do projeto em que todas as chaves são UUID (`[09:51] Larissa`). Esse UUID é o valor de `X-Event-Id` e do `event_id` no corpo.
3. O `X-Event-Id` é gerado quando o evento entra na outbox e permanece estável entre retentativas do mesmo evento.

## Alternativas Consideradas

### Guardar apenas `order_id` e renderizar no envio

Descartada em `[09:52] Larissa`. Se o pedido mudar depois, o evento deixaria de refletir o estado do momento da transição, gerando "caso esquisito". O snapshot preserva a semântica imutável do evento.

### ID auto-incremental para a outbox

Descartada em `[09:51] Larissa`. Quebraria a convenção do projeto de usar UUID em todas as entidades, e exporia volume de eventos em um identificador sequencial previsível.

## Consequências

### Positivas

- O evento é imutável e auditável: ele sempre representa o estado exato do pedido no instante da transição.
- Reordenar ou editar um pedido depois não altera o que já foi notificado.
- O `event_id` estável permite deduplicação confiável no cliente ([ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md)).
- UUID evita chave sequencial previsível e mantém consistência com o resto do schema (`prisma/schema.prisma`).

### Negativas

- **Desnormalização**: o payload é duplicado no banco e ocupa mais espaço do que uma referência. Irrelevante no volume esperado (eventos pequenos, sem `items`, teto de 64 KB).
- Correções retroativas no payload (ex.: campo errado) não alcançam eventos já enfileirados — só valem para eventos futuros. É aceito como consequência da imutabilidade.
- O payload precisa ser montado dentro da transação de `changeStatus`, o que adiciona trabalho ao caminho crítico. Mitigado por ser apenas montagem de JSON em memória, sem I/O adicional.

## Referências

- Transcrição: `[09:51] Diego`, `[09:51] Larissa`, `[09:51] Bruno`, `[09:52] Larissa`, `[09:52] Diego`, `[09:52] Bruno`, `[09:43] Diego`
- Código: `prisma/schema.prisma` (convenção UUID e `@@map`), `src/modules/orders/order.service.ts` (`changeStatus`)
- Relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md)
