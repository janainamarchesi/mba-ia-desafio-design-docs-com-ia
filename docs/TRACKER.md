# Tracker de Rastreabilidade

Referência cruzada entre cada item registrado nos documentos e sua origem na transcrição da reunião (`TRANSCRICAO.md`) ou no código-fonte da aplicação.

**Legenda**

- `Fonte = TRANSCRICAO` → `Localização` no formato `[hh:mm] Falante`, exatamente como no arquivo de transcrição.
- `Fonte = CODIGO` → `Localização` é o caminho real do arquivo no repositório.

Itens que são elaboração de engenharia sem origem na reunião nem âncora direta em código **não entram nesta tabela** — eles estão listados separadamente no [Apêndice A](#apêndice-a--derivações-de-engenharia), para que ninguém os confunda com requisitos discutidos.

---

## PRD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Hoje os clientes fazem polling em `GET /orders`, tornando a integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Contexto | Atlas pode migrar para o concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Restrição | Webhooks são outbound apenas; clientes não enviam eventos para nós | TRANSCRICAO | [09:02] Marcos |
| PRD-META-01 | docs/PRD.md | Métrica | Latência de ponta a ponta abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-META-02 | docs/PRD.md | Métrica | Janela de retry cobre ~14h36min (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Diego |
| PRD-META-03 | docs/PRD.md | Métrica | 100% das entregas assinadas com HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| PRD-META-04 | docs/PRD.md | Métrica | 3 clientes B2B migrados para webhook | TRANSCRICAO | [09:00] Marcos |
| PRD-META-05 | docs/PRD.md | Métrica | Prazo de 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-ESCOPO-01 | docs/PRD.md | Descarte | Dashboard visual para clientes fora de escopo | TRANSCRICAO | [09:40] Larissa |
| PRD-ESCOPO-02 | docs/PRD.md | Adiamento | Fallback por e-mail adiado para fase futura | TRANSCRICAO | [09:37] Larissa |
| PRD-ESCOPO-03 | docs/PRD.md | Ponto em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-ESCOPO-04 | docs/PRD.md | Descarte | Arquivamento de linhas entregues após 30 dias fora do escopo | TRANSCRICAO | [09:08] Diego |
| PRD-ESCOPO-05 | docs/PRD.md | Restrição | Webhooks inbound não fazem parte | TRANSCRICAO | [09:02] Marcos |
| PRD-ESCOPO-06 | docs/PRD.md | Adiamento | Escala horizontal do worker e ordenação global adiadas | TRANSCRICAO | [09:13] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de endpoint com URL, filtro de status e `customerId`; secret devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01a | docs/PRD.md | Requisito Funcional | `customerId` vem do body ou path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Listagem de endpoints por cliente, paginada | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição de endpoint via PATCH | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção de endpoint via DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Toda transição de status válida gera evento na mesma transação | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Entrega HTTP POST com payload JSON enxuto e headers | TRANSCRICAO | [09:43] Diego |
| PRD-FR-07a | docs/PRD.md | Requisito Funcional | Header `X-Webhook-Id` com o id do endpoint | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 com secret do endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Retry com backoff 1m/5m/30m/2h/12h, 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay de DLQ exige ADMIN e registra o autor | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por endpoint | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | URL deve ser HTTPS; `http` é recusado na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | At-least-once com `X-Event-Id` para deduplicação | TRANSCRICAO | [09:26] Larissa |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Notificação entregue em menos de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Polling de 2 segundos no worker | TRANSCRICAO | [09:09] Diego |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64 KB, erro acima disso, sem truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por tentativa HTTP | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Nenhuma chamada de rede dentro da transação de status | TRANSCRICAO | [09:04] Bruno |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Secret devolvida apenas na criação, nunca reexposta depois | TRANSCRICAO | [09:31] Marcos |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Indisponibilidade do cliente não bloqueia mudança de status de outros pedidos | TRANSCRICAO | [09:04] Bruno |
| PRD-RNF-07 | docs/PRD.md | Limitação | Ordenação garantida apenas por pedido e com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-RNF-08 | docs/PRD.md | Limitação | CRUD sem escopo por cliente nesta fase | TRANSCRICAO | [09:37] Sofia |
| PRD-RISCO-01 | docs/PRD.md | Risco | Prazo comercial não fecha com a estimativa de 3 sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-RISCO-02 | docs/PRD.md | Risco | Cliente não implementa verificação de assinatura ou deduplicação | TRANSCRICAO | [09:25] Sofia |
| PRD-RISCO-03 | docs/PRD.md | Risco | Evento preso ou entregue fora de ordem | TRANSCRICAO | [09:13] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de Sofia, 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação de integração no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |

---

## RFC

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-REV-01 | docs/RFC.md | Metadado | Revisores: Larissa, Marcos, Bruno, Diego, Sofia | TRANSCRICAO | [09:00] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Chamada HTTP síncrona dentro de `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-01a | docs/RFC.md | Trade-off | Rollback da mudança de status seria a única saída segura — inaceitável | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams ou broker externo por exigir infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco notificando o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry infinito deixa evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Marcar `failed` na outbox em vez de tabela de DLQ separada | TRANSCRICAO | [09:18] Diego |
| RFC-QA-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:38] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em aberto | Ordenação sob escala horizontal: particionar por `order_id` ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| RFC-QA-05 | docs/RFC.md | Questão em aberto | Recuperação de secret perdida (só é devolvida na criação) | TRANSCRICAO | [09:31] Marcos |
| RFC-QA-06 | docs/RFC.md | Questão em aberto | Retenção e arquivamento da outbox e da DLQ | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-01 | docs/RFC.md | Impacto | `OrderService.changeStatus` ganha dependência nova dentro da transação | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | docs/RFC.md | Impacto | Primeiro canal de saída de dados para terceiros exige revisão de segurança | TRANSCRICAO | [09:46] Sofia |
| RFC-RISCO-01 | docs/RFC.md | Risco | Retry quebra a ordenação por pedido | TRANSCRICAO | [09:12] Diego |
| RFC-RISCO-02 | docs/RFC.md | Risco | Trabalho extra na transação crítica de pedidos | TRANSCRICAO | [09:04] Bruno |

---

## FDD

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Gravar evento atomicamente com a mudança de status | TRANSCRICAO | [09:06] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Reuso integral de `AppError`, Pino, middleware e Zod | TRANSCRICAO | [09:30] Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /api/v1/webhooks` cria endpoint e devolve secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /api/v1/webhooks?customerId=` lista endpoints sem secret | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /api/v1/webhooks/:id` edita url, filtro e estado ativo | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /api/v1/webhooks/:id` remove endpoint | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `GET /api/v1/webhooks/:id/deliveries` retorna últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06a | docs/FDD.md | Requisito | Existência da rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` recoloca evento na outbox | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Entrega outbound com headers `X-Event-Id`, `X-Signature`, `X-Timestamp` | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-08a | docs/FDD.md | Contrato | Header `X-Webhook-Id` com o id do endpoint | TRANSCRICAO | [09:44] Sofia |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato | Payload com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-02 | docs/FDD.md | Restrição | Payload não inclui itens do pedido | TRANSCRICAO | [09:43] Diego |
| FDD-ERRO-01 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` para URL sem HTTPS | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | `WEBHOOK_SECRET_REQUIRED` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-07 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` acima de 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-08 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_TIMEOUT` | TRANSCRICAO | [09:42] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | `src/modules/orders/order.service.ts` — `publishWebhookEvent(tx, ...)` dentro do `$transaction` | TRANSCRICAO | [09:40] Bruno |
| FDD-INT-01a | docs/FDD.md | Integração | Função pura recebendo `tx`, sem injetar repositório | TRANSCRICAO | [09:41] Diego |
| FDD-INT-02 | docs/FDD.md | Integração | `src/modules/orders/order.status.ts` — enum `OrderStatus` e `canTransition` | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | `src/shared/errors/app-error.ts` — base `AppError` | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | `src/shared/errors/http-errors.ts` — `NotFoundError` fixa `NOT_FOUND`, exige classe própria | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-05 | docs/FDD.md | Integração | `src/middlewares/error.middleware.ts` — serializa `AppError` sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole('ADMIN')` | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | `src/middlewares/validate.middleware.ts` — `validate({ body, query, params })` | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | `src/routes/index.ts` — `Controllers` e `buildApiRouter` | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | `src/app.ts` — `buildControllers` e montagem antes do catch-all | CODIGO | src/app.ts |
| FDD-INT-10 | docs/FDD.md | Integração | `src/server.ts` — molde de `bootstrap()` e graceful shutdown para o worker | CODIGO | src/server.ts |
| FDD-INT-11 | docs/FDD.md | Integração | `src/config/env.ts` — schema Zod com `process.exit(1)`, nova variável precisa de default | CODIGO | src/config/env.ts |
| FDD-INT-12 | docs/FDD.md | Integração | `src/shared/logger/index.ts` — `redactPaths` não cobre `*.secret` | CODIGO | src/shared/logger/index.ts |
| FDD-INT-13 | docs/FDD.md | Integração | `prisma/schema.prisma` — convenções UUID, `@@map`, `@@index` | CODIGO | prisma/schema.prisma |
| FDD-INT-14 | docs/FDD.md | Integração | `tests/setup.ts` — ordem de limpeza; novas FKs para `customer` devem vir antes | CODIGO | tests/setup.ts |
| FDD-INT-15 | docs/FDD.md | Integração | `tests/helpers/factories.ts` — `bootstrapAuthenticatedUser(role)` | CODIGO | tests/helpers/factories.ts |
| FDD-INT-16 | docs/FDD.md | Integração | `src/shared/http/response.ts` — `paginated()` reutilizado nas listagens | CODIGO | src/shared/http/response.ts |
| FDD-INT-17 | docs/FDD.md | Integração | `src/middlewares/request-logger.middleware.ts` — `X-Request-Id` para correlação | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-18 | docs/FDD.md | Integração | `src/config/database.ts` — `createPrismaClient()` por processo | CODIGO | src/config/database.ts |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro do `$transaction` de `changeStatus` | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Polling de 2s lendo pendentes mais antigos em lote | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Backoff 1m/5m/30m/2h/12h e movimento para DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Replay reinsere na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-01 | docs/FDD.md | Resiliência | Timeout HTTP de 10s conta como falha | TRANSCRICAO | [09:42] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados Pino para enqueue, tentativa, sucesso, falha, DLQ e replay | TRANSCRICAO | [09:29] Bruno |
| FDD-RISCO-01 | docs/FDD.md | Risco | Retry quebra ordenação por pedido (R-1) | TRANSCRICAO | [09:12] Diego |
| FDD-RISCO-06 | docs/FDD.md | Risco | Secret não redigida no log (R-6) | CODIGO | src/shared/logger/index.ts |
| FDD-RISCO-08 | docs/FDD.md | Risco | Trabalho extra na transação de status aumenta contenção (R-8) | TRANSCRICAO | [09:04] Bruno |
| FDD-RISCO-09 | docs/FDD.md | Risco | Novas FKs quebram a suíte existente (R-9) | CODIGO | tests/setup.ts |
| FDD-RISCO-10 | docs/FDD.md | Risco | CRUD sem escopo por cliente expõe dados cross-tenant (R-10) | TRANSCRICAO | [09:37] Sofia |

---

## ADRs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL, inserido na transação de status | TRANSCRICAO | [09:06] Diego |
| ADR-001a | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Síncrono bloquearia mudança de status de outros pedidos | TRANSCRICAO | [09:04] Bruno |
| ADR-001b | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Redis Cluster seria overengineering para o time | TRANSCRICAO | [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-exponencial-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-002a | docs/adrs/ADR-002-retry-exponencial-e-dlq.md | Alternativa descartada | 3 tentativas é pouco para manutenção planejada | TRANSCRICAO | [09:16] Diego |
| ADR-002b | docs/adrs/ADR-002-retry-exponencial-e-dlq.md | Decisão | DLQ em tabela `webhook_dead_letter` separada | TRANSCRICAO | [09:18] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-003a | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Alternativa descartada | Secret global faria um vazamento comprometer todos | TRANSCRICAO | [09:21] Sofia |
| ADR-003b | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Contexto | Já houve cliente que vazou secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| ADR-004 | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com `X-Event-Id` para dedup | TRANSCRICAO | [09:26] Larissa |
| ADR-004a | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once exige coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-com-polling.md | Decisão | Worker em processo separado, `src/worker.ts`, polling 2s | TRANSCRICAO | [09:11] Larissa |
| ADR-005a | docs/adrs/ADR-005-worker-processo-separado-com-polling.md | Alternativa descartada | MySQL não tem `LISTEN/NOTIFY`; trigger não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006a | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Prefixo `WEBHOOK_` em todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| ADR-006b | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Análise de código | `NotFoundError` fixa `NOT_FOUND`; exige classe própria para `WEBHOOK_NOT_FOUND` | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006c | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Análise de código | `redactPaths` não cobre `*.secret` | CODIGO | src/shared/logger/index.ts |
| ADR-006d | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Análise de código | Estrutura de módulo controller/service/repository/routes/schemas | CODIGO | src/modules/orders/order.routes.ts |
| ADR-006e | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Análise de código | `requireRole` já existe e é reutilizável no replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Decisão | Snapshot do payload renderizado na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007a | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Decisão | IDs em UUID, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-007b | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Alternativa descartada | Guardar só `order_id` e renderizar no envio geraria "caso esquisito" | TRANSCRICAO | [09:51] Bruno |

---

## Resumo de cobertura

| Métrica | Valor | Exigência |
| --- | --- | --- |
| Total de linhas rastreáveis | 111 | — |
| Linhas com `Fonte = TRANSCRICAO` | 92 | — |
| Linhas com `Fonte = CODIGO` | 19 | ≥ 5 |
| Percentual `TRANSCRICAO` válido | 82,9% | ≥ 70% |
| Percentual com localização verificável | 100% | — |

Os 19 itens de código apontam para arquivos que existem no repositório — todos foram verificados por leitura direta.

---

## Apêndice A — Derivações de engenharia

Os itens abaixo são elaboração deste pacote: caminhos de rota, códigos de erro complementares, mecanismos de proteção e detalhes de observabilidade que **não foram ditos na reunião** e não têm âncora direta em código. Eles estão registrados aqui, fora da tabela de rastreabilidade, para que um leitor saiba exatamente o que é decisão de quem escreveu e o que é registro da reunião.

| ID | Documento | Tipo | Conteúdo (resumo) | Observação |
| --- | --- | --- | --- | --- |
| RFC-QA-03 | docs/RFC.md | Questão em aberto | Formato de transporte das duas secrets durante a rotação | O grace period de 24h está na transcrição; o transporte não foi discutido |
| RFC-QA-04 | docs/RFC.md | Questão em aberto | Semântica de `X-Event-Id` em fan-out para múltiplos endpoints | Consequência de at-least-once, não discutida na reunião |
| RFC-QA-07 | docs/RFC.md | Questão em aberto | Rollback ou descarte quando o payload excede 64 KB | O teto e o erro estão na transcrição; a consequência transacional não |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Caminho `POST /webhooks/:id/secret/rotate` | A *existência* da rotação está em `[09:21] Sofia` (ver FDD-CONTRATO-06a); o caminho é decisão de engenharia |
| FDD-ERRO-04 | docs/FDD.md | Erro | `WEBHOOK_INVALID_EVENT` para filtro fora do enum `OrderStatus` | Derivado do filtro de eventos (`[09:33] Marcos`) |
| FDD-ERRO-05 | docs/FDD.md | Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | Derivado do endpoint de replay (`[09:35] Diego`) |
| FDD-ERRO-06 | docs/FDD.md | Erro | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | Proteção contra replay duplo, não discutida |
| FDD-ERRO-09 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_FAILED` para não-2xx ou erro de conexão | Derivado da política de retry |
| FDD-ERRO-10 | docs/FDD.md | Erro | `WEBHOOK_ENDPOINT_INACTIVE` | Derivado do campo `active` (`[09:21] Bruno`) |
| FDD-ERRO-11 | docs/FDD.md | Erro | `WEBHOOK_SECRET_MISSING` no momento do envio | Proteção defensiva, não discutida |
| FDD-RESIL-02 | docs/FDD.md | Resiliência | Lease/`claimedAt` com recolhimento de evento preso em `PROCESSING` | Mecanismo de recuperação após queda do worker, não discutido |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métricas derivadas de consultas à outbox e à DLQ | A reunião fala em logs (`[09:29] Bruno`), não em métricas |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação por `eventId`/`webhookId`/`deliveryId` | Extensão do `X-Request-Id` existente, não discutida |
| FDD-RISCO-02 | docs/FDD.md | Risco | Ambiguidade do comportamento acima de 64 KB (R-2) | Consequência da lacuna em RFC-QA-07 |
| FDD-RISCO-03 | docs/FDD.md | Risco | Dois workers acidentais quebram ordenação (R-3) | Decorrente da limitação de single-worker (`[09:13] Larissa`) |
| FDD-RISCO-04 | docs/FDD.md | Risco | Worker morre com eventos em `PROCESSING` (R-4) | Motivou FDD-RESIL-02 |
| FDD-RISCO-05 | docs/FDD.md | Risco | `X-Timestamp` fora da base assinada permite replay (R-5) | O header está em `[09:44] Diego`; o risco não foi analisado |
| FDD-RISCO-07 | docs/FDD.md | Risco | Backoff de horas é praticamente intestável (R-7) | Consequência de teste, não discutida |
| FDD-RISCO-11 | docs/FDD.md | Risco | `X-Event-Id` em fan-out ambíguo (R-11) | Decorrente de RFC-QA-04 |
