# ADR-006 — Reuso dos padrões existentes do projeto para o módulo de webhooks

## Status

Aceito

## Contexto

A codebase tem um padrão de organização consolidado e repetido em cinco domínios: `auth`, `users`, `customers`, `products` e `orders`. Introduzir um sexto domínio (`webhooks`) com convenções próprias criaria inconsistência e obrigaria o time a aprender duas formas de fazer a mesma coisa.

A reunião fechou a decisão de **reuso máximo** (`[09:30] Larissa`). O objetivo deste ADR é registrar *quais* padrões concretos são reusados e *como*, com referência direta ao código existente — porque é aqui que o módulo de webhooks se conecta ao que já existe.

## Decisão

O módulo `src/modules/webhooks` é um módulo igual aos outros, e reutiliza os seguintes artefatos já existentes:

### 1. Estrutura de módulo — controller / service / repository / routes / schemas

Reusado de `src/modules/orders/`, que é o molde mais completo:

| Arquivo de referência | Papel | Equivalente no módulo de webhooks |
| --- | --- | --- |
| `src/modules/orders/order.controller.ts` | tradução HTTP ↔ service | `webhook.controller.ts` |
| `src/modules/orders/order.service.ts` | regra de negócio | `webhook.service.ts` |
| `src/modules/orders/order.repository.ts` | acesso a dados | `webhook.repository.ts` |
| `src/modules/orders/order.routes.ts` | `buildOrderRouter(controller)` | `webhook.routes.ts` com `buildWebhookRouter` |
| `src/modules/orders/order.schemas.ts` | schemas Zod + tipos inferidos | `webhook.schemas.ts` |

O padrão de router é sempre a factory `build<X>Router(controller: <X>Controller): Router` que chama `router.use(authenticate)` e então registra rotas com `validate({ ... })` (ver `src/modules/orders/order.routes.ts`).

### 2. Classe base de erro — `AppError`

`src/shared/errors/app-error.ts` define `AppError(message, statusCode, errorCode, details?)`. Todos os erros novos do módulo derivam dela e carregam código com prefixo `WEBHOOK_` (`[09:28] Bruno`, `[09:29] Larissa`).

**Detalhe importante descoberto ao ler o código:** `NotFoundError` em `src/shared/errors/http-errors.ts` tem assinatura `constructor(resource = 'Resource')` e **fixa o código `'NOT_FOUND'`** — não é possível obter `WEBHOOK_NOT_FOUND` a partir dela. Já `BadRequestError`, `ConflictError` e `UnprocessableEntityError` aceitam `code` como parâmetro. Portanto:

- `WEBHOOK_NOT_FOUND` (404) exige uma classe própria estendendo `AppError` diretamente, ou `new AppError(msg, 404, 'WEBHOOK_NOT_FOUND')`.
- `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` podem estender `BadRequestError` com código customizado, exatamente como `InvalidStatusTransitionError` e `InsufficientStockError` fazem em `src/shared/errors/http-errors.ts`.

### 3. Middleware de erro centralizado

`src/middlewares/error.middleware.ts` já serializa qualquer `AppError` como `{ error: { code, message, details? } }` com o `statusCode` correto, mapeia `ZodError` para 400 `VALIDATION_ERROR` e trata `P2002`/`P2025` do Prisma. **Nenhuma alteração é necessária** para que erros `WEBHOOK_*` cheguem ao cliente corretamente (`[09:29] Bruno`).

### 4. Logger — Pino

`src/shared/logger/index.ts` exporta o `logger` Pino já configurado com `service`, `env`, timestamps ISO e paths de redação. O worker e o dispatcher usam esse mesmo logger, sem biblioteca nova (`[09:29] Bruno`).

**Lacuna identificada:** `redactPaths` cobre `*.password`, `*.passwordHash`, `*.token`, `*.accessToken` e headers de auth, mas **não cobre `*.secret`**. Como as secrets de webhook trafegam por esse módulo, adicionar `'*.secret'` é uma tarefa de implementação obrigatória — do contrário o cenário de vazamento em log descrito em `[09:22] Diego` se repete do nosso lado.

### 5. Autenticação e autorização — `requireRole`

`src/middlewares/auth.middleware.ts` exporta `authenticate` (Bearer JWT → `req.user: { id, email, role }`, com `role: 'ADMIN' | 'OPERATOR'`) e `requireRole(...roles)`, que lança `ForbiddenError('Insufficient permissions')`.

O CRUD de configuração usa apenas `authenticate` (`[09:37] Sofia`). O replay de DLQ usa `authenticate, requireRole('ADMIN')` — exatamente o encadeamento já usado em `src/modules/users/user.routes.ts` para `GET /:id` (`[09:36] Larissa`).

### 6. Validação — middleware Zod

`src/middlewares/validate.middleware.ts` expõe `validate({ body?, query?, params? })`, que faz `parse` e reatribui. A regra "URL precisa ser `https`" vive num schema Zod em `webhook.schemas.ts` (`[09:23] Sofia`), não numa decisão arquitetural.

### 7. Registro de rotas — `buildApiRouter`

`src/routes/index.ts` define o tipo `Controllers` e a factory `buildApiRouter(controllers)` que monta cada sub-router com `router.use`. O módulo entra como `controllers.webhooks` + `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', ...)`, ambas as montagens **antes** do catch-all de `NotFoundError` em `src/app.ts`.

`src/app.ts` (`buildControllers`) é onde as dependências são instanciadas no padrão `new XRepository(prisma) → new XService(repo) → new XController(service)`; o webhook segue o mesmo encadeamento.

### 8. Paginação

`src/shared/http/response.ts` exporta `paginated(data, page, pageSize, total)` e `PaginatedResponse<T>`, usados por `OrderService.list`. Reutilizado em `GET /webhooks` e `GET /webhooks/:id/deliveries`, com `page`/`pageSize` validados por Zod (`pageSize` máximo 100, como em `listOrdersQuerySchema`).

### 9. Prisma e modelo de dados

`prisma/schema.prisma` estabelece as convenções: `String @id @default(uuid()) @db.Char(36)`, `@@map("nome_snake_case")`, `@@index(...)`, e o uso de `Json` para colunas semiestruturadas (ex.: `Customer.address`). As novas tabelas (`webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`) seguem exatamente essas convenções.

### 10. Bootstrap e shutdown

`src/server.ts` é o molde do novo `src/worker.ts`: `bootstrap()` com try/catch, `logger.fatal` em falha e handlers de `SIGINT`/`SIGTERM` chamando `prisma.$disconnect()`.

## Alternativas Consideradas

### Criar um módulo de notificação com stack própria (biblioteca de eventos, ORM separado, logger próprio)

Descartada em `[09:29] Bruno` e `[09:30] Larissa`. Duplicaria infraestrutura e criaria dois padrões para o mesmo problema. O time é pequeno e a codebase já tem todos os componentes necessários.

### Injetar um `WebhookRepository` inteiro dentro do `OrderService`

Descartada em `[09:41] Diego`. Em vez de acoplar o `OrderService` a um repositório de outro módulo, a integração é uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação corrente. O construtor `(orders, prisma)` de `OrderService` não muda.

## Consequências

### Positivas

- Curva de aprendizado zero: quem conhece `orders` conhece `webhooks`.
- Middleware de erro, logger e autenticação funcionam sem alteração.
- Testes reaproveitam `tests/helpers/factories.ts` (`bootstrapAuthenticatedUser(role)`, `getTestApp()`).
- Menos código novo, menos superfície de bug.

### Negativas

- **Herdamos limitações do código existente.** O caso mais concreto é `NotFoundError` fixar `NOT_FOUND`: para cumprir o padrão `WEBHOOK_*` é preciso contornar, não estender. Isso cria uma pequena inconsistência que precisa ser consciente.
- `redactPaths` precisa ser estendido para `*.secret`; é uma alteração em arquivo compartilhado (`src/shared/logger/index.ts`), não só em código novo.
- Novas tabelas com FK para `customers` quebram a ordem de limpeza em `tests/setup.ts` até que as novas tabelas sejam inseridas antes de `customer` no `beforeEach`. Como o entregável não pode alterar `tests/`, isso entra como tarefa de implementação futura, não como algo feito aqui.

## Referências

- Transcrição: `[09:27] Bruno`, `[09:28] Diego`, `[09:28] Bruno`, `[09:29] Larissa`, `[09:29] Bruno`, `[09:29] Diego`, `[09:30] Bruno`, `[09:30] Larissa`, `[09:36] Larissa`, `[09:37] Sofia`, `[09:41] Bruno`, `[09:41] Diego`
- Código: `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`, `src/modules/orders/order.routes.ts`, `src/modules/orders/order.schemas.ts`, `src/modules/users/user.routes.ts`, `prisma/schema.prisma`, `tests/setup.ts`, `tests/helpers/factories.ts`
- Relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-worker-processo-separado-com-polling.md)
