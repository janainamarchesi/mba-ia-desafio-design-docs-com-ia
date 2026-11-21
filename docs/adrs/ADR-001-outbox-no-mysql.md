# ADR-001 — Padrão Outbox no MySQL para eventos de mudança de status

## Status

Aceito

## Contexto

O OMS precisa notificar clientes B2B quando o status de um pedido muda. A mudança de status é hoje uma transação pesada em `OrderService.changeStatus`: atualiza `orders`, insere em `order_status_history`, e decrementa ou repõe `stock_quantity` dos produtos do pedido (`src/modules/orders/order.service.ts`).

A pergunta colocada na reunião foi como acoplar o disparo da notificação a essa transação sem comprometer a consistência dos dados. O requisito de produto é que o cliente receba o evento em menos de 10 segundos, e que nunca exista um status alterado sem o evento correspondente registrado.

A aplicação não tem hoje eventos, filas nem brokers. O MySQL já é o sistema de registro e já é usado com transações Prisma.

## Decisão

Adotar o **padrão Outbox dentro do MySQL existente**. Quando o status de um pedido muda, uma linha é inserida na tabela `webhook_outbox` **dentro da mesma transação SQL** que atualiza `orders`, grava `order_status_history` e ajusta o estoque.

Se a transação principal commita, o evento foi registrado. Se ela dá rollback, o evento desaparece junto. Um worker separado lê a tabela e dispara as chamadas HTTP.

O ponto exato de inserção é dentro do callback de `this.prisma.$transaction(async (tx) => { ... })` já existente em `OrderService.changeStatus`, conforme `[09:40] Bruno` e `[09:41] Diego`.

## Alternativas Consideradas

### Chamada HTTP síncrona dentro de `changeStatus`

Descartada em `[09:04] Bruno` e `[09:06] Diego`. Um cliente lento bloquearia a mudança de status de outros pedidos, e a única saída segura para um cliente fora do ar seria dar rollback na mudança de status — o que é inaceitável do ponto de vista de negócio ("se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá", `[09:04] Bruno`).

### Redis Streams ou broker externo

Descartada em `[09:07] Diego`. Exigiria subir infraestrutura adicional (Redis Cluster) para um time pequeno — caracterizado como *overengineering*. O MySQL existente já resolve o problema sem novo componente operacional.

### Trigger de banco notificando o worker

Descartada em `[09:09] Diego`. O MySQL não tem equivalente ao `NOTIFY`/`LISTEN` do Postgres; um trigger só executa SQL e não consegue sinalizar um processo externo. Qualquer improviso (escrever em arquivo, bater num endpoint) seria frágil, e o polling de 2 segundos já atende o requisito de 10 segundos.

## Consequências

### Positivas

- **Atomicidade real** entre mudança de status e registro do evento: não existe janela em que o status mude sem evento gravado, nem evento gravado para um status que sofreu rollback.
- Nenhuma infraestrutura nova. O time continua operando apenas MySQL.
- O padrão é verificável por teste: um teste de integração que força falha na inserção da outbox pode provar o rollback do status.
- O worker pode ser reiniciado sem perder eventos, porque o estado durável está no banco.

### Negativas

- **Consistência eventual**: existe uma latência entre a mudança de status e a entrega, limitada pelo intervalo de polling (ver ADR-005). O requisito de 10 segundos é atendido com folga, mas não há entrega instantânea.
- **Crescimento da tabela**: `webhook_outbox` acumula linhas. Mitigação decidida é índice em status e em `created_at`, leitura em lotes pequenos apenas dos pendentes, e arquivamento de linhas entregues após ~30 dias — este último explicitamente fora do escopo desta feature (`[09:08] Diego`).
- Acopla a lógica de notificação ao processo de pedidos: a inserção na outbox passa a fazer parte do caminho crítico de `changeStatus`. Mitigado por ser uma única inserção indexada, sem I/O de rede dentro da transação.

## Referências

- Transcrição: `[09:03] Larissa`, `[09:04] Bruno`, `[09:06] Diego`, `[09:07] Larissa`, `[09:07] Diego`, `[09:08] Diego`, `[09:08] Larissa`, `[09:40] Bruno`, `[09:41] Diego`
- Código: `src/modules/orders/order.service.ts` (`changeStatus`, `prisma.$transaction`)
- Relacionados: [ADR-005](ADR-005-worker-processo-separado-com-polling.md), [ADR-007](ADR-007-snapshot-do-payload-na-outbox.md)
