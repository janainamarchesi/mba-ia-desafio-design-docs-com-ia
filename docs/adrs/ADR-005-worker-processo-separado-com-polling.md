# ADR-005 — Worker em processo separado com polling de 2 segundos

## Status

Aceito

## Contexto

Com o padrão outbox decidido ([ADR-001](ADR-001-outbox-no-mysql.md)), a pergunta seguinte é **quem** lê a tabela e **como** ele sabe que há trabalho. Duas dimensões independentes:

1. **Onde o consumidor roda**: dentro da API ou em processo próprio?
2. **Como ele descobre eventos novos**: polling ou notificação reativa?

A aplicação hoje tem um único entry point, `src/server.ts`, com `bootstrap()`, graceful shutdown em `SIGINT`/`SIGTERM` e um `PrismaClient` próprio exportado por `src/config/database.ts`.

## Decisão

1. O worker roda como **processo Node separado da API**, com entry point `src/worker.ts` e script `npm run worker`, espelhando o padrão de `src/server.ts` (`[09:11] Larissa`).
2. A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`) (`[09:28] Bruno`).
3. O worker usa **`PrismaClient` próprio**, criado no seu processo, apontando para o mesmo banco e `DATABASE_URL` (`[09:30] Bruno`). `PrismaClient` é por processo.
4. A descoberta é por **polling a cada 2 segundos**: a cada ciclo, busca os eventos pendentes mais antigos, processa e marca (`[09:09] Diego`).
5. Execução **single-worker**. A ordenação por `order_id` depende dessa escolha ([ADR-002](ADR-002-retry-exponencial-e-dlq.md) e limitação registrada abaixo).

## Alternativas Consideradas

### Worker dentro da instância da API

Descartada em `[09:11] Diego`. Se a API reiniciar — deploy, crash, autoscaling — o worker morre junto e as notificações param até a próxima subida. Processo separado desacopla o ciclo de vida.

### Trigger do banco notificando o worker

Descartada em `[09:09] Diego`. Detalhada no [ADR-001](ADR-001-outbox-no-mysql.md): MySQL não tem `LISTEN`/`NOTIFY`, e um trigger não consegue sinalizar processo externo. Polling de 2 s já atende o requisito de 10 s.

### Broker de mensagens com push (Redis Streams, RabbitMQ, Kafka)

Descartada em `[09:07] Diego` por exigir infraestrutura adicional para um time pequeno.

## Consequências

### Positivas

- O worker sobrevive a deploys e reinícios da API.
- Pode ser escalado, monitorado e reiniciado de forma independente.
- Polling é simples e sem dependência de infraestrutura extra.
- O padrão de bootstrap e graceful shutdown já existe em `src/server.ts` e serve de molde direto.

### Negativas

- **Latência intrínseca**: pior caso de 2 segundos entre a mudança de status e o início do envio, mais o tempo da chamada HTTP. Compatível com o alvo de 10 segundos, mas a margem precisa ser preservada — o timeout de 10 s do HTTP (`[09:42] Diego`) é o teto de uma tentativa, não da latência total.
- **Custo de polling constante**: consultas a cada 2 segundos, mesmo sem eventos. Mitigado por ser uma consulta indexada em status e `created_at`, em lote pequeno.
- **Worker único limita o scale-out**: com múltiplos workers, a ordenação por `created_at` se perde. A reunião registrou isso como limitação conhecida, não como garantia global (`[09:13] Larissa`). Particionar por `order_id` ou usar lock pessimista foi explicitamente adiado como "problema do futuro" (`[09:13] Diego`).
- Nada impede operacionalmente que dois workers subam por engano; isso quebraria a ordenação e criaria entregas concorrentes. A implementação precisa de uma trava ou lease (ver riscos no [FDD](../FDD.md)).

## Referências

- Transcrição: `[09:08] Larissa`, `[09:09] Diego`, `[09:09] Bruno`, `[09:10] Marcos`, `[09:10] Larissa`, `[09:11] Diego`, `[09:11] Larissa`, `[09:11] Bruno`, `[09:12] Larissa`, `[09:12] Diego`, `[09:13] Bruno`, `[09:13] Diego`, `[09:13] Larissa`, `[09:28] Bruno`
- Código: `src/server.ts`, `src/config/database.ts`, `src/config/env.ts`
- Relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-retry-exponencial-e-dlq.md)
