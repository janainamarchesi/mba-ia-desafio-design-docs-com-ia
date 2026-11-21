# FDD — Sistema de Webhooks de Notificação de Pedidos

> **Pergunta que este documento responde:** como construir, em detalhe?
> A proposta em nível de arquitetura está no [RFC](RFC.md). O racional de cada decisão está nos [ADRs](adrs/README.md). Este documento assume as decisões como fechadas e desce ao nível de implementação.

## 1. Contexto e motivação técnica

O OMS hoje não possui eventos, filas nem webhooks. O ciclo de vida do pedido é transacional: `OrderService.changeStatus` em `src/modules/orders/order.service.ts` roda dentro de `this.prisma.$transaction(async (tx) => …)`, valida a transição com `canTransition` de `src/modules/orders/order.status.ts`, debita ou repõe estoque e grava o histórico em `order_status_history`.

O desafio técnico é notificar clientes externos sem que essa transação dependa de terceiros. A resposta é o **padrão Outbox** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)): a intenção de notificar é gravada na mesma transação, e um worker separado ([ADR-005](adrs/ADR-005-worker-processo-separado-com-polling.md)) materializa a entrega.

O módulo de webhooks é o primeiro ponto de saída de dados para fora da infraestrutura, o que traz requisitos de integridade e autenticidade que não existiam antes.

## 2. Objetivos técnicos

- Gravar o evento de webhook **atomicamente** com a mudança de status, sem chamada de rede dentro da transação.
- Entregar com latência de ponta a ponta **abaixo de 10 segundos** em operação normal.
- Autenticar cada entrega com **HMAC-SHA256** e secret **por endpoint**, com rotação e grace period de 24 h.
- Garantir **at-least-once** com identificador estável (`X-Event-Id`) para deduplicação.
- Sobreviver a indisponibilidade do cliente com **5 tentativas** e backoff fixo, terminando em **DLQ**.
- Reutilizar integralmente os padrões existentes: `AppError`, Pino, middleware de erro, Zod, estrutura de módulo ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).
- Manter a API de pedidos isolada da indisponibilidade de clientes externos.

## 3. Escopo e exclusões

**Incluído:**

- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`.
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro de `changeStatus`.
- `src/worker.ts` com polling de 2 s, dispatcher HTTP, retry e DLQ.
- Endpoints de CRUD de configuração, histórico de entregas, rotação de secret e replay de DLQ.
- Observabilidade por logs estruturados e correlação.

**Excluído:**

- Dashboard visual para clientes (`[09:40] Larissa`).
- Fallback por e-mail quando o webhook falha (`[09:37] Larissa`).
- Rate limiting de saída (ponto em aberto, `[09:39] Larissa`).
- Arquivamento de linhas entregues após 30 dias (`[09:08] Diego`).
- Webhooks inbound (`[09:02] Marcos`).
- Escala horizontal do worker e garantia de ordenação global (`[09:13] Larissa`).
- Criptografia da secret em repouso (não discutido — ver risco R-6).

## 4. Fluxos detalhados

### 4.1 Criação do evento na outbox

```
PATCH /api/v1/orders/:id/status
      │
      ▼
OrderService.changeStatus(id, input, userId)
      │  prisma.$transaction(tx =>
      │    1. tx.order.findUnique({ include: { items: true } })
      │    2. if (!canTransition(from, to)) throw InvalidStatusTransitionError
      │    3. if (shouldDebitStock)  debitStock(tx, ...)
      │    4. if (shouldReplenishStock) replenishStock(tx, ...)
      │    5. tx.order.update({ status: to })
      │    6. tx.orderStatusHistory.create({ ... })
      │    7. publishWebhookEvent(tx, order, from, to)   ◄── NOVO
      │    8. tx.order.findUnique({ ... include relations })
      │  )
      ▼
publishWebhookEvent(tx, order, from, to):
      a. encontra webhook_endpoints WHERE customerId = order.customerId
                                       AND active = true
                                       AND to_status ∈ eventFilter
      b. se nenhum endpoint: retorna sem inserir linha
      c. para cada endpoint (fan-out):
         - monta o payload (snapshot) com event_id UUID novo
         - tx.webhookOutbox.create({ status: PENDING, payload, ... })
```

Regras:

- A função recebe o **`tx` transacional**. Não abre transação própria e não faz I/O de rede.
- O `event_id` é gerado aqui, no momento da inserção ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md)).
- O payload é **renderizado e persistido** neste ponto — snapshot, não referência.
- Se uma inserção na outbox falhar, o erro propaga e **a transação inteira sofre rollback**, incluindo a mudança de status (`[09:40] Bruno`).
- O filtro de eventos é aplicado **na inserção**, não no envio (`[09:34] Bruno`).

### 4.2 Processamento pelo worker

```
src/worker.ts  (npm run worker)
      │
      ▼ bootstrap()
      │  cria PrismaClient próprio (createPrismaClient)
      │  registra SIGINT / SIGTERM → $disconnect
      ▼ loop a cada 2 segundos:
        1. tx: SELECT ... FROM webhook_outbox
                 WHERE status = 'PENDING' AND nextAttemptAt <= now()
                 ORDER BY createdAt ASC
                 LIMIT batchSize
                 FOR UPDATE SKIP LOCKED
        2. marca status = 'PROCESSING' e grava lease/claimedAt
        3. para cada evento, para cada endpoint alvo:
             - carrega a secret vigente do endpoint
             - monta assinatura HMAC-SHA256(body, secret)
             - POST https://... com timeout de 10 s
        4. sucesso (2xx): status = 'DELIVERED'; grava webhook_deliveries
        5. falha: agenda retry ou move para DLQ
```

O worker é **single-instance**. A ordenação por `order_id` é consequência disso e não é garantida sob escala horizontal.

### 4.3 Retry

| Tentativa | Espera desde a falha anterior | `nextAttemptAt` |
| --- | --- | --- |
| 1ª | 1 min | now + 1m |
| 2ª | 5 min | now + 5m |
| 3ª | 30 min | now + 30m |
| 4ª | 2 h | now + 2h |
| 5ª | 12 h | now + 12h |
| Esgotou | — | move para DLQ |

A progressão é **fixa** e configurável em constantes do módulo, não adaptativa. O total até a última tentativa é de ~14h36min (`[09:17] Diego`). Uma resposta HTTP não-2xx, timeout de 10 s ou erro de TLS contam como falha e consomem uma tentativa. Conexão recusada também.

**Recomendação de implementação:** a progressão deve ser injetável (`backoffScheduleMs: number[]`) com os valores de produção como default, para permitir teste automatizado sem esperar horas (ver R-7).

### 4.4 DLQ e replay

```
esgotou 5 tentativas
      │
      ▼
tx.webhookDeadLetter.create({
     originalOutboxId, webhookId, eventId,
     payload, failureReason, attempts, createdAt
   })
   e remove/marca a linha da outbox como terminal
      │
      ▼
POST /api/v1/admin/webhooks/dead-letter/:id/replay   (ADMIN)
      │
      ├─ loga { replayById: req.user.id, deadLetterId, ... }   ◄── auditoria
      └─ reinsere na outbox com status = PENDING e attempt = 0
```

O replay exige `requireRole('ADMIN')` e registra quem executou, conforme `[09:36] Sofia`.

## 5. Contratos públicos

Todos os endpoints ficam sob `/api/v1` (montagem em `src/app.ts`). O envelope de erro segue o middleware existente: `{ "error": { "code", "message", "details?" } }`.

### 5.1 `POST /api/v1/webhooks`

Cria um endpoint de webhook e devolve a secret **uma única vez**.

**Autorização:** `authenticate` (qualquer role).

**Request:**

```json
{
  "customerId": "6f1c2b1e-3a4d-4c9e-9f2a-1b2c3d4e5f60",
  "url": "https://integracao.atlascomercial.com.br/webhooks/oms",
  "eventFilter": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`:**

```json
{
  "id": "0b9f4c1a-7e2d-4a11-b7f3-2c8d9e0a1b23",
  "customerId": "6f1c2b1e-3a4d-4c9e-9f2a-1b2c3d4e5f60",
  "url": "https://integracao.atlascomercial.com.br/webhooks/oms",
  "eventFilter": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f3b...",
  "createdAt": "2027-11-21T09:00:00.000Z"
}
```

`customerId` vem do **body**, nunca do JWT (`[09:32] Larissa`). URL sem `https` é rejeitada com `WEBHOOK_INVALID_URL` (`[09:23] Sofia`).

### 5.2 `GET /api/v1/webhooks?customerId=&page=&pageSize=`

Lista os endpoints de um cliente, paginado no padrão `paginated()` de `src/shared/http/response.ts`.

**Autorização:** `authenticate`.

**Request:** sem corpo. Parâmetros na query string:

```
GET /api/v1/webhooks?customerId=6f1c2b1e-3a4d-4c9e-9f2a-1b2c3d4e5f60&page=1&pageSize=20
```

**Response `200 OK`:**

```json
{
  "data": [
    {
      "id": "0b9f4c1a-7e2d-4a11-b7f3-2c8d9e0a1b23",
      "customerId": "6f1c2b1e-3a4d-4c9e-9f2a-1b2c3d4e5f60",
      "url": "https://integracao.atlascomercial.com.br/webhooks/oms",
      "eventFilter": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2027-11-21T09:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

A secret **nunca** aparece em listagem.

### 5.3 `PATCH /api/v1/webhooks/:id`

Atualiza URL, filtro de eventos e estado ativo.

**Autorização:** `authenticate`.

**Request:**

```json
{ "url": "https://integracao.atlascomercial.com.br/v2/webhooks", "active": false }
```

**Response `200 OK`:** o objeto do endpoint, sem secret.
**Response `404`:** `WEBHOOK_NOT_FOUND`.

### 5.4 `DELETE /api/v1/webhooks/:id`

Remove um endpoint.

**Autorização:** `authenticate`.

**Request:** sem corpo — o alvo é o `:id` do path.

```
DELETE /api/v1/webhooks/0b9f4c1a-7e2d-4a11-b7f3-2c8d9e0a1b23
```

**Response `204 No Content`** — sem corpo.
**Response `404`:** `WEBHOOK_NOT_FOUND`.

### 5.5 `GET /api/v1/webhooks/:id/deliveries?page=&pageSize=`

Histórico de entregas do endpoint: os **últimos 100** (`[09:34] Marcos`), expostos no padrão de paginação existente, com `pageSize` máximo 100.

**Autorização:** `authenticate`.

**Request:** sem corpo. Parâmetros na query string:

```
GET /api/v1/webhooks/0b9f4c1a-7e2d-4a11-b7f3-2c8d9e0a1b23/deliveries?page=1&pageSize=20
```

**Response `200 OK`:**

```json
{
  "data": [
    {
      "id": "c4d5e6f7-1111-4a2b-8c3d-4e5f6a7b8c9d",
      "eventId": "9a8b7c6d-2222-4e3f-a1b2-c3d4e5f6a7b8",
      "eventType": "order.status_changed",
      "status": "DELIVERED",
      "responseStatus": 200,
      "responseTimeMs": 143,
      "payload": { "event_id": "9a8b7c6d-...", "order_id": "..." },
      "responseBody": "{\"received\":true}",
      "attemptedAt": "2027-11-20T14:03:11.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

### 5.6 `POST /api/v1/webhooks/:id/secret/rotate`

Rotaciona a secret do endpoint. A antiga continua válida por **24 horas** (`[09:21] Sofia`).

> **Nota de rastreabilidade:** a *existência* da rotação está na transcrição; o *caminho* desta rota é uma decisão de engenharia deste documento, não uma fala da reunião.

**Autorização:** `authenticate`.

**Response `200 OK`:**

```json
{
  "id": "0b9f4c1a-7e2d-4a11-b7f3-2c8d9e0a1b23",
  "secret": "whsec_nova_7c1d...",
  "previousSecretValidUntil": "2027-11-22T09:00:00.000Z"
}
```

### 5.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Recoloca um evento da DLQ na outbox como pendente.

**Autorização:** `authenticate` + `requireRole('ADMIN')` (`[09:36] Larissa`).

**Request:** sem corpo.

**Response `200 OK`:**

```json
{
  "deadLetterId": "e1f2a3b4-3333-4c5d-9e6f-7a8b9c0d1e2f",
  "outboxId": "f2a3b4c5-4444-4d6e-af70-8b9c0d1e2f3a",
  "status": "PENDING",
  "replayedBy": "1a2b3c4d-5555-4e6f-b071-9c0d1e2f3a4b",
  "replayedAt": "2027-11-21T09:30:00.000Z"
}
```

**Response `403`:** `FORBIDDEN` para role não-ADMIN. **Response `404`:** `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 5.8 Entrega outbound (o contrato que o cliente recebe)

**Método e URL:** `POST <url cadastrada>`, obrigatoriamente HTTPS.

**Headers:**

| Header | Valor |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (`event_id`) |
| `X-Webhook-Id` | id do endpoint que está recebendo |
| `X-Signature` | HMAC-SHA256 hex do corpo, com a secret do endpoint |
| `X-Timestamp` | timestamp ISO 8601 do envio (para o cliente detectar replay) |

**Body:**

```json
{
  "event_id": "9a8b7c6d-2222-4e3f-a1b2-c3d4e5f6a7b8",
  "event_type": "order.status_changed",
  "timestamp": "2027-11-21T09:15:02.000Z",
  "order_id": "3c2b1a09-8888-4f7e-a6b5-c4d3e2f1a0b9",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2b1e-3a4d-4c9e-9f2a-1b2c3d4e5f60",
  "total_cents": 45990
}
```

O payload **não inclui os itens** do pedido; o cliente consulta `GET /orders/:id` se precisar de detalhe (`[09:43] Diego`).

**Sucesso:** qualquer `2xx` do cliente encerra o ciclo.
**Falha:** não-`2xx`, timeout de 10 s, TLS inválido ou erro de conexão consomem uma tentativa.

### 5.9 Ausência deliberada de `GET /api/v1/webhooks/:id`

Não existe, e não havia previsão na reunião. Quem precisa da configuração de um endpoint usa a listagem por `customerId`.

## 6. Matriz de erros

Os erros se dividem em **dois planos distintos**. Misturá-los seria um erro de modelagem: um é resposta HTTP da API, o outro é estado persistido da entrega.

### 6.1 Erros da API (retornados ao chamador)

| Código | HTTP | Situação | Origem na implementação |
| --- | --- | --- | --- |
| `WEBHOOK_INVALID_URL` | 400 | URL ausente, malformada ou sem `https` | schema Zod de `webhook.schemas.ts` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação exige secret e ela não pôde ser gerada/derivada | service |
| `WEBHOOK_INVALID_EVENT` | 400 | `eventFilter` contém valor que não é `OrderStatus` | `z.nativeEnum(OrderStatus)` |
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint inexistente | classe própria estendendo `AppError` (ver ADR-006) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Item de DLQ inexistente | classe própria |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | Item de DLQ já foi recolocado | `ConflictError` com código customizado |
| `VALIDATION_ERROR` | 400 | Falha genérica de schema | `error.middleware.ts` (existente) |
| `UNAUTHORIZED` | 401 | Token ausente ou inválido | `auth.middleware.ts` (existente) |
| `FORBIDDEN` | 403 | Role não permitida (replay, sem ADMIN) | `requireRole` (existente) |

### 6.2 Erros do plano de entrega (persistidos, não retornados por rota)

| Código | Situação | Onde é registrado |
| --- | --- | --- |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | payload > 64 KB | log + abortar a inserção/entrega, conforme decisão de R-2 |
| `WEBHOOK_DELIVERY_TIMEOUT` | cliente não respondeu em 10 s | `webhook_deliveries` + contador de retry |
| `WEBHOOK_DELIVERY_FAILED` | resposta não-2xx ou erro de conexão/TLS | `webhook_deliveries` + contador de retry |
| `WEBHOOK_ENDPOINT_INACTIVE` | endpoint inativado entre a inserção e o envio | log + encerrar o evento |
| `WEBHOOK_SECRET_MISSING` | secret ausente/corrompida no momento do envio | log + mover para DLQ |

## 7. Estratégias de resiliência

| Mecanismo | Valor | Observação |
| --- | --- | --- |
| Timeout HTTP | 10 s | por tentativa (`[09:42] Diego`) |
| Tentativas | 5 | ADR-002 |
| Backoff | 1m / 5m / 30m / 2h / 12h | fixo, não adaptativo |
| DLQ | tabela separada | replay manual ADMIN |
| Isolamento da API | nenhuma chamada de rede em `changeStatus` | outbox |
| Falha na inserção | rollback da transação | atômico |
| Worker preso em `PROCESSING` | lease com `claimedAt`; recolhe após N min | ver R-4 |
| Payload grande | erro acima de 64 KB, nunca truncar | `[09:23] Sofia` |

**Não há fallback permissivo.** Se a entrega falha, ela falha de forma rastreável; a operação de pedido já foi concluída antes de qualquer tentativa de rede.

## 8. Observabilidade

O projeto **não possui** SDK de métricas nem OpenTelemetry (`package.json` contém Express, Prisma, Pino, Zod, `jsonwebtoken`, `bcrypt`, `uuid`; não há `prom-client` nem pacotes OTel). Introduzi-los seria uma dependência nova nunca discutida. Portanto, a observabilidade usa o que já existe — Pino — e eventos estruturados com nomes fixos.

### 8.1 Logs

O logger é `src/shared/logger/index.ts` (Pino, JSON em produção). Eventos estruturados com campos fixos:

| Evento (`msg`) | Campos |
| --- | --- |
| `webhook_event_enqueued` | `eventId`, `orderId`, `fromStatus`, `toStatus`, `endpointCount` |
| `webhook_delivery_attempt` | `eventId`, `webhookId`, `attempt`, `statusCode`, `durationMs` |
| `webhook_delivery_succeeded` | `eventId`, `webhookId`, `durationMs` |
| `webhook_delivery_failed` | `eventId`, `webhookId`, `attempt`, `errorCode`, `durationMs` |
| `webhook_moved_to_dlq` | `eventId`, `webhookId`, `attempts`, `failureReason` |
| `webhook_dead_letter_replayed` | `deadLetterId`, `eventId`, `replayedById` |
| `webhook_worker_started` / `_stopped` | `pollIntervalMs`, `batchSize` |

**Segurança de log:** a secret nunca é logada. `redactPaths` em `src/shared/logger/index.ts` precisa ganhar `'*.secret'` (ver R-6).

### 8.2 Métricas

Sem SDK, as métricas são **derivadas dos logs** e de consultas ao banco. As que importam:

- `webhook_outbox_pending` — `SELECT COUNT(*) FROM webhook_outbox WHERE status = 'PENDING'`.
- `webhook_outbox_age_seconds` — idade do pendente mais antigo; é o indicador direto do alvo de 10 s.
- `webhook_dlq_size` — `SELECT COUNT(*) FROM webhook_dead_letter`.
- Taxa de sucesso por endpoint — derivada de `webhook_deliveries`.
- Latência de entrega — p50/p95/p99 a partir de `webhook_deliveries.attemptedAt` menos o `createdAt` da outbox.

Essas consultas são adequadas para healthcheck e dashboard interno, e não substituem um APM.

### 8.3 Tracing

Sem OTel, a correlação é por identificadores. `src/middlewares/request-logger.middleware.ts` já propaga `X-Request-Id` e gera UUID quando ausente. O worker estende o mesmo padrão: cada evento carrega `eventId`, `webhookId` e `deliveryId` em todos os logs da cadeia, permitindo reconstruir a trajetória de uma entrega por grep/consulta.

Isso é **elaboração de engenharia**, não uma decisão da reunião; está marcado como tal no [TRACKER](TRACKER.md).

## 9. Dependências e compatibilidade

| Dependência | Versão / estado | Observação |
| --- | --- | --- |
| Node.js + TypeScript | conforme `package.json` | sem dependência nova |
| MySQL | 8.0 (`docker-compose.yml`) | `FOR UPDATE SKIP LOCKED` disponível |
| Prisma | já no projeto | novas tabelas via migration |
| Zod | já no projeto | validação de schemas |
| Pino | já no projeto | logs |
| `node:crypto` | built-in | HMAC-SHA256, geração de secret |
| Docker Compose | já existe | adicionar serviço `worker` |

**Compatibilidade:** novas tabelas e colunas são aditivas; nenhum contrato existente de `orders`, `customers` ou `products` muda. O `OrderService` ganha uma chamada interna, mas a assinatura dos métodos públicos não muda.

**Variáveis de ambiente:** qualquer nova variável (ex.: `WEBHOOK_POLL_INTERVAL_MS`) deve entrar em `src/config/env.ts` **com default**, porque o schema chama `process.exit(1)` quando falta uma variável obrigatória — sem default, `npm test` e deploys existentes quebram.

## 10. Integração com o sistema existente

Esta seção nomeia os arquivos reais do código base e o ponto exato de acoplamento.

### 10.1 `src/modules/orders/order.service.ts`

**Hoje:** `changeStatus(id, input, userId)` roda dentro de `this.prisma.$transaction(async (tx) => { ... })`, na ordem: busca o pedido → valida `canTransition` → `debitStock`/`replenishStock` → `tx.order.update` → `tx.orderStatusHistory.create` → refetch.

**Como o módulo entra:** uma chamada a `publishWebhookEvent(tx, order, from, to)` inserida **dentro do callback**, depois do `orderStatusHistory.create` e antes do refetch. O arquivo já declara `type TxClient = Prisma.TransactionClient`, então a função nova usa o mesmo tipo. O construtor `constructor(orders: OrderRepository, prisma: PrismaClient)` **não muda**, porque a função recebe o `tx` e não uma dependência injetada (`[09:41] Diego`).

**Por que importa:** é a única alteração em lógica de negócio existente, e é o que garante a atomicidade.

### 10.2 `src/modules/orders/order.status.ts`

**Hoje:** exporta `canTransition`, `allowedTransitions`, `isTerminal`, `STOCK_DEBIT_TRANSITION`, `shouldDebitStock`, `shouldReplenishStock`, sobre o enum `OrderStatus` (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`).

**Como o módulo entra:** o `eventFilter` de cada endpoint é validado com `z.nativeEnum(OrderStatus)` em `webhook.schemas.ts`, reusando o enum — sem duplicar a lista de status. As transições elegíveis a evento são exatamente as que `canTransition` permite, porque só existe evento quando há transição válida. `DELIVERED` e `CANCELLED` são terminais, logo um evento `CANCELLED` é necessariamente o último que um pedido pode gerar.

### 10.3 `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`

**Hoje:** `AppError(message, statusCode, errorCode, details?)` é a base. `BadRequestError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError` derivam dela; `InvalidStatusTransitionError` e `InsufficientStockError` são exemplos de erro de domínio com código próprio.

**Como o módulo entra:** `WEBHOOK_*` segue o mesmo molde. `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` estende `ConflictError` com código customizado, como `InvalidStatusTransitionError` faz. **Ponto de atenção:** `NotFoundError(resource)` fixa o código em `'NOT_FOUND'`; para obter `WEBHOOK_NOT_FOUND` é preciso uma classe própria sobre `AppError` (ver ADR-006, seção 2).

### 10.4 `src/middlewares/error.middleware.ts`

**Hoje:** serializa `AppError` como `{ error: { code, message, details? } }`, trata `ZodError` (400 `VALIDATION_ERROR`), `P2002` (409) e `P2025` (404), e loga o resto como 500.

**Como o módulo entra:** **nenhuma alteração.** Os erros `WEBHOOK_*`, por derivarem de `AppError`, são serializados corretamente e com o `statusCode` certo (`[09:29] Bruno`).

### 10.5 `src/middlewares/auth.middleware.ts`

**Hoje:** `authenticate` (Bearer JWT → `req.user: { id, email, role }`, com `role: 'ADMIN' | 'OPERATOR'`) e `requireRole(...)`.

**Como o módulo entra:** `buildWebhookRouter` chama `router.use(authenticate)` como `buildOrderRouter` faz. A rota de replay encadeia `authenticate, requireRole('ADMIN')`, exatamente como `src/modules/users/user.routes.ts` faz em `GET /:id`. O `replayedById` sai de `req.user.id`.

### 10.6 `src/middlewares/validate.middleware.ts`

**Hoje:** `validate({ body?, query?, params? })` faz `parse` e reatribui.

**Como o módulo entra:** todas as rotas usam `validate` com schemas de `webhook.schemas.ts`, incluindo a regra de `https` na URL como refine de Zod, conforme `[09:23] Sofia`.

### 10.7 `src/routes/index.ts` e `src/app.ts`

**Hoje:** `buildApiRouter(controllers)` recebe `Controllers`, monta cada sub-router com `router.use`, e `buildControllers(prisma)` instancia `Repository → Service → Controller`.

**Como o módulo entra:** adicionar `webhooks` ao tipo `Controllers`, montar `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`, e instanciar os três componentes em `buildControllers`. As montagens precisam vir **antes** do catch-all `app.use((req,res,next) => next(new NotFoundError(...)))` em `src/app.ts`, senão as rotas nunca são alcançadas.

### 10.8 `src/server.ts` → novo `src/worker.ts`

**Hoje:** `bootstrap()` sobe o HTTP, registra `SIGINT`/`SIGTERM` chamando `prisma.$disconnect()` e `process.exit(0)`, e em falha faz `logger.fatal` + `exit(1)`.

**Como o módulo entra:** `src/worker.ts` repete o esqueleto (import de `createPrismaClient`, `logger`, handlers de sinal), trocando o `app.listen` pelo loop de polling. O `package.json` ganha o script `"worker"`.

### 10.9 `prisma/schema.prisma`

**Hoje:** convenções `String @id @default(uuid()) @db.Char(36)`, `@@map("snake_case")`, `@@index(...)`, `Json` para semiestruturado (`Customer.address`).

**Como o módulo entra:** quatro modelos novos seguindo essas convenções, incluindo `@@index([status, createdAt])` na outbox — exigido em `[09:08] Diego`.

### 10.10 `src/config/env.ts`

**Hoje:** schema Zod que chama `process.exit(1)` se faltar variável obrigatória.

**Como o módulo entra:** novas variáveis (`WEBHOOK_POLL_INTERVAL_MS`, `WEBHOOK_BATCH_SIZE`, etc.) entram **com default**, para não quebrar `npm test` nem deploys existentes.

### 10.11 `src/shared/logger/index.ts`

**Hoje:** `redactPaths` cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`.

**Como o módulo entra:** adicionar `'*.secret'`. Sem isso, uma secret de webhook logada por engano vaza em texto puro — o mesmo cenário que Diego descreveu de um cliente (`[09:22]`).

### 10.12 `tests/setup.ts` e `tests/helpers/factories.ts`

**Hoje:** `beforeEach` limpa, nesta ordem, `orderStatusHistory → orderItem → order → orderNumberSequence → product → customer → user`. `factories.ts` expõe `bootstrapAuthenticatedUser(role)` com `ADMIN`/`OPERATOR`.

**Como o módulo entra:** as novas tabelas com FK para `customers` precisam ser limpas **antes** de `customer`, senão a suíte existente quebra por violação de FK. Como o entregável não altera `tests/`, isso fica registrado como tarefa de implementação. Os testes de autorização do replay usam `bootstrapAuthenticatedUser('ADMIN')` e `('OPERATOR')`.

## 11. Critérios de aceite técnicos

- [ ] Um `PATCH /orders/:id/status` válido grava exatamente uma linha em `webhook_outbox` por endpoint inscrito, dentro da mesma transação.
- [ ] Forçar falha na inserção da outbox provoca rollback do status, do histórico e do estoque.
- [ ] Nenhuma linha é inserida quando nenhum endpoint do cliente se inscreveu no status de destino.
- [ ] O payload persistido é o snapshot do momento da transição e não muda se o pedido for editado depois.
- [ ] O worker entrega em menos de 10 s da mudança de status em operação normal.
- [ ] Falha de rede agenda a próxima tentativa conforme a progressão 1m/5m/30m/2h/12h.
- [ ] Esgotadas as 5 tentativas, o evento existe em `webhook_dead_letter` com payload, motivo e timestamp.
- [ ] `POST /admin/webhooks/dead-letter/:id/replay` responde 200 para ADMIN, 403 para OPERATOR e grava `replayedById`.
- [ ] A assinatura enviada em `X-Signature` é `HMAC-SHA256(body, secret)` e é verificável pelo cliente.
- [ ] Rotação devolve secret nova e mantém a anterior válida por 24 h.
- [ ] URL `http` é rejeitada com `WEBHOOK_INVALID_URL`.
- [ ] Payload acima de 64 KB não é truncado; gera erro rastreável.
- [ ] `GET /webhooks/:id/deliveries` retorna os últimos 100 registros com status, payload, resposta e tempo.
- [ ] Nenhuma secret aparece em log, resposta de listagem ou de atualização.
- [ ] Erros `WEBHOOK_*` saem no envelope `{ error: { code, message, details? } }` do middleware existente.

## 12. Riscos e mitigação

| ID | Risco | Impacto | Mitigação |
| --- | --- | --- | --- |
| R-1 | Retry quebra a ordenação: um evento em retry é entregue depois de um mais novo do mesmo pedido | Cliente pode ver `PAID` após `SHIPPED` | Documentar que a garantia vale para entregas bem-sucedidas na primeira tentativa; avaliar bloqueio por `order_id` em fase futura |
| R-2 | Ambiguidade de payload > 64 KB: o erro impede a mudança de status ou só descarta o evento? | Pode reverter uma operação legítima por causa de notificação | Decidir antes do código; recomendação: não reverter, registrar `WEBHOOK_PAYLOAD_TOO_LARGE` e seguir |
| R-3 | Dois workers acidentais quebram ordenação e geram duplicatas em massa | Entregas fora de ordem e multiplicadas | Trava/lease de instância única por `GET_LOCK` ou tabela de lease, com alarme |
| R-4 | Worker morre com eventos em `PROCESSING` | Eventos presos, nunca retomados | `claimedAt` + recolhimento após N minutos via `FOR UPDATE SKIP LOCKED` |
| R-5 | `X-Timestamp` fora da base assinada permite replay de corpo válido | Reentrega maliciosa detectável só pelo cliente | Documentar que o cliente deve validar janela de timestamp; mudar a base assinada é decisão de Segurança |
| R-6 | Secret em claro no banco e não redigida no log | Vazamento de credencial | Adicionar `'*.secret'` ao `redactPaths`; discutir criptografia em repouso com Sofia |
| R-7 | Backoff de horas é praticamente intestável | Regressões invisíveis | Injetar a tabela de backoff como parâmetro com default de produção |
| R-8 | Trabalho extra dentro da transação de status aumenta contenção | Latência de mudança de status | Uma consulta indexada por `(customerId, active)`; sem I/O de rede; fan-out pequeno |
| R-9 | Novas FKs quebram a suíte existente em `tests/setup.ts` | CI vermelho na primeira migration | Ajustar a ordem de limpeza como tarefa de implementação |
| R-10 | CRUD não tem escopo por cliente: qualquer role autenticada acessa qualquer customer | Exposição cross-tenant | Aceito nesta fase (`[09:37] Sofia`); registrar ticket de endurecimento |
| R-11 | `X-Event-Id` em fan-out ambíguo | Cliente pode descartar entrega legítima | Definir se é por evento ou por endpoint antes de congelar o contrato |

## 13. Como este documento se relaciona com os demais

| Documento | Papel |
| --- | --- |
| [RFC](RFC.md) | Proposta em nível de arquitetura e questões em aberto |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md) | Racional de cada decisão |
| [PRD](PRD.md) | Requisitos de produto, escopo e métricas |
| [TRACKER](TRACKER.md) | Origem de cada item deste documento na transcrição ou no código |
