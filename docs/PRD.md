# PRD — Sistema de Webhooks de Notificação de Pedidos

> **Pergunta que este documento responde:** por que e o quê?
> A proposta técnica está no [RFC](RFC.md); a implementação, no [FDD](FDD.md); o racional das decisões, nos [ADRs](adrs/README.md).

| Campo | Valor |
| --- | --- |
| Versão | v1 |
| Data | Reunião técnica de `quinta-feira, 09:00` (registro em `TRANSCRICAO.md`) |
| Responsável | Equipe de Produto e Tecnologia — Pedidos |
| Status | Aprovado na reunião técnica |

---

## 1. Resumo e contexto

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram formalmente, na semana anterior à reunião, ser notificados em tempo real quando o status dos seus pedidos muda. Hoje eles fazem polling em `GET /orders` em intervalos regulares, o que torna a integração deles lenta e cara. A Atlas sinalizou risco de migração para o concorrente caso a entrega não aconteça até o fim do trimestre.

Esta feature entrega **webhooks outbound**: quando o status de um pedido muda, a plataforma envia um HTTP `POST` assinado para o endpoint HTTPS cadastrado pelo cliente. O evento é registrado de forma atômica com a mudança de status e entregue por um worker dedicado, com retry e DLQ. A entrega é at-least-once e o cliente deduplica pelo `X-Event-Id`.

A feature preenche um vácuo deliberado: o OMS não possui hoje nenhum mecanismo de eventos, filas ou notificação externa.

---

## 2. Problema e motivação

### 2.1 O problema

Os clientes B2B dependem de **polling manual** para descobrir que um pedido mudou. Isso tem três consequências:

1. **Integração lenta e cara para o cliente** — ele precisa bater repetidamente em `GET /orders` para descobrir mudanças que talvez não existam.
2. **Risco de negócio** — a Atlas condicionou a permanência ao fim do trimestre à entrega desta notificação.
3. **Experiência pobre** — o cliente não recebe a informação no momento em que ela acontece.

### 2.2 Por que agora

Os três pedidos formais são recentes, a Atlas colocou prazo e há sinal claro de risco competitivo. Do lado técnico, a plataforma já tem o que é necessário — MySQL transacional, máquina de estados de pedido controlada, auditoria de status — então a feature é aditiva, sem exigir infraestrutura nova.

### 2.3 O que muda para o cliente

Em vez de perguntar "mudou?", o cliente **recebe** a mudança, com garantia de autenticidade, em menos de 10 segundos, e com histórico consultável de entregas.

---

## 3. Público-alvo e cenários de uso

**Públicos:**

- **Clientes B2B integrados** (Atlas Comercial, MaxDistribuição, Nova Cargo) — consomem os webhooks nos seus sistemas.
- **Usuários que representam o cliente** — cadastram e administram os endpoints pela API, autenticados com o JWT da plataforma.
- **Operação e suporte internos** — consultam configuração e histórico de entregas.
- **Administradores** — únicos autorizados a reprocessar eventos da DLQ.

**Cenários-chave:**

1. **Cadastro**: um usuário representando o cliente cria um endpoint informando URL HTTPS e quais status quer receber. A plataforma devolve a secret, exibida uma única vez.
2. **Notificação**: o pedido muda de `PROCESSING` para `SHIPPED`. O cliente, que assinou apenas `SHIPPED` e `DELIVERED`, recebe o evento em menos de 10 segundos.
3. **Verificação**: o cliente valida a assinatura HMAC do corpo antes de processar.
4. **Indisponibilidade**: o endpoint do cliente está fora do ar. A plataforma tenta 5 vezes ao longo de ~14h36min; se o cliente voltar, ele recebe o evento.
5. **Falha permanente**: as 5 tentativas se esgotam. O evento vai para a DLQ, e um ADMIN pode recolocá-lo na fila manualmente, com registro de auditoria.
6. **Rotação de secret**: o cliente suspeita de vazamento e solicita uma nova secret; a antiga continua válida por 24 horas para permitir a migração.
7. **Diagnóstico**: o cliente consulta o histórico das últimas 100 entregas para investigar uma notificação que não chegou.

---

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Entregar a notificação em "tempo real" na percepção do cliente | Latência de ponta a ponta entre a mudança de status e o recebimento | **abaixo de 10 segundos** |
| Sustentar indisponibilidade razoável do cliente | Janela coberta pela política de retry | **~14h36min** (1m/5m/30m/2h/12h) |
| Garantir autenticidade verificável | Cobertura das entregas com assinatura HMAC-SHA256 válida | **100% das entregas** |
| Eliminar a dependência de polling para os clientes integrados | Clientes B2B migrados para webhook | **3 clientes** (Atlas, MaxDistribuição, Nova Cargo) |
| Entregar dentro da expectativa comercial | Prazo de implementação | **3 sprints**, incluindo 2 dias úteis de revisão de segurança |

> As metas acima são as explicitamente afirmadas na reunião. Nenhum número adicional foi inventado.

---

## 5. Escopo

### 5.1 Incluso

- Cadastro, listagem, edição e remoção de endpoints de webhook por cliente.
- Filtro de eventos por endpoint: cada endpoint escolhe quais status quer receber.
- Notificação automática em toda transição de status válida de pedido.
- Assinatura HMAC-SHA256 com secret por endpoint.
- Rotação de secret com grace period de 24 horas.
- Entrega at-least-once com `X-Event-Id` para deduplicação pelo cliente.
- Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada.
- Replay manual de DLQ restrito a `ADMIN`, com auditoria de quem executou.
- Histórico de entregas por endpoint (últimos 100 registros).
- Payload enxuto, sem itens do pedido.
- URL obrigatoriamente HTTPS.

### 5.2 Fora de escopo

Itens **explicitamente descartados ou adiados na reunião**:

1. **Dashboard visual para clientes** — descartado. "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend." (`[09:40] Larissa`)
2. **Fallback por e-mail quando o webhook falha** (ex.: após 3 falhas consecutivas) — **adiado para fase futura**, condicionado a medir o impacto primeiro. (`[09:37] Larissa`, `[09:38] Marcos`)
3. **Rate limiting de saída** por cliente — **não descartado nem escopado**: registrado como ponto em aberto, "observar e decidir depois". (`[09:38] Diego`, `[09:39] Larissa`)
4. **Arquivamento de linhas entregues da outbox após ~30 dias** — explicitamente fora do escopo desta feature. (`[09:08] Diego`)
5. **Webhooks inbound** — os clientes só recebem; não enviam eventos para nós. (`[09:02] Marcos`)
6. **Escala horizontal do worker e ordenação global** — adiado como "problema do futuro". (`[09:13] Diego`, `[09:13] Larissa`)

> Os itens 1, 2 e 4 são descartes/adiamentos explícitos. O item 3 é ponto em aberto, e o PRD não o trata como requisito.

---

## 6. Requisitos funcionais

### RF-01 Cadastro de endpoint de webhook

O cliente cadastra um endpoint informando URL, filtro de status e `customerId`. A plataforma gera a secret e a devolve **somente na criação**.

- **Fluxo principal:** usuário autenticado chama `POST /webhooks` → informa `customerId`, `url` HTTPS e `eventFilter` → recebe o endpoint criado com a secret.
- **Alternativos/exceções:** URL sem HTTPS é recusada com `WEBHOOK_INVALID_URL`; `eventFilter` inválido recusa a criação.
- **Erros previstos:** `WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_EVENT`, `VALIDATION_ERROR`.
- **Prioridade:** alta.

### RF-02 Listagem de endpoints por cliente

- **Fluxo principal:** `GET /webhooks?customerId=` retorna os endpoints do cliente, paginados, **sem** a secret.
- **Erros previstos:** `VALIDATION_ERROR` para paginação inválida.
- **Prioridade:** alta.

### RF-03 Edição de endpoint

- **Fluxo principal:** `PATCH /webhooks/:id` altera URL, filtro ou estado ativo.
- **Alternativos:** URL sem HTTPS é recusada.
- **Erros previstos:** `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`.
- **Prioridade:** alta.

### RF-04 Remoção de endpoint

- **Fluxo principal:** `DELETE /webhooks/:id` remove o endpoint; eventos futuros não são gerados para ele.
- **Erros previstos:** `WEBHOOK_NOT_FOUND`.
- **Prioridade:** alta.

### RF-05 Filtro de eventos por endpoint na inserção

- **Fluxo principal:** cada endpoint declara os status que quer ouvir (ex.: só `SHIPPED` e `DELIVERED`). O filtro é aplicado **no momento de inserir na outbox**; se nenhum endpoint do cliente quer aquele status, **nenhuma linha é inserida**.
- **Prioridade:** alta.

### RF-06 Notificação automática na mudança de status

- **Fluxo principal:** toda transição de status válida gera o evento, gravado na mesma transação da mudança (ver RNF-01).
- **Alternativos:** transições inválidas não geram evento, porque são recusadas pela máquina de estados.
- **Prioridade:** alta.

### RF-07 Entrega HTTP assinada

- **Fluxo principal:** o worker envia `POST` para o endpoint com corpo JSON contendo `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, sem os itens do pedido.
- **Headers:** `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`.
- **Erros previstos:** timeout, não-2xx, TLS inválido — ver RF-10.
- **Prioridade:** alta.

### RF-08 Assinatura HMAC-SHA256 por endpoint

- **Fluxo principal:** o corpo é assinado com HMAC-SHA256 com a secret do endpoint, enviado em `X-Signature`.
- **Prioridade:** alta.

### RF-09 Rotação de secret com grace period

- **Fluxo principal:** o cliente solicita nova secret; a antiga permanece válida por 24 horas.
- **Erros previstos:** `WEBHOOK_NOT_FOUND`.
- **Prioridade:** média.

### RF-10 Retry com backoff exponencial

- **Fluxo principal:** falha de entrega agenda nova tentativa em 1m, 5m, 30m, 2h e 12h, totalizando 5 tentativas.
- **Prioridade:** alta.

### RF-11 DLQ em tabela separada

- **Fluxo principal:** esgotadas as tentativas, o evento é persistido em `webhook_dead_letter` com payload, motivo e timestamp.
- **Prioridade:** alta.

### RF-12 Replay manual de DLQ restrito a ADMIN

- **Fluxo principal:** `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento como pendente; o usuário que executou é registrado para auditoria.
- **Erros previstos:** `FORBIDDEN` para não-ADMIN, `WEBHOOK_DEAD_LETTER_NOT_FOUND`.
- **Prioridade:** alta.

### RF-13 Histórico de entregas

- **Fluxo principal:** `GET /webhooks/:id/deliveries` retorna os últimos 100 registros com sucesso/falha, payload, resposta do cliente e tempo de resposta.
- **Prioridade:** média.

### RF-14 Validação de URL HTTPS

- **Fluxo principal:** cadastro com URL `http` é recusado na validação.
- **Prioridade:** alta.

### RF-15 Garantia at-least-once com deduplicação

- **Fluxo principal:** o mesmo evento pode ser entregue mais de uma vez; o `X-Event-Id` estável permite ao cliente deduplicar.
- **Prioridade:** alta.

### RF-16 Latência abaixo de 10 segundos

- **Fluxo principal:** em operação normal, o cliente recebe a notificação em menos de 10 segundos da mudança de status.
- **Prioridade:** alta.

---

## 7. Requisitos não funcionais

**Performance**

- Latência de ponta a ponta abaixo de 10 segundos, com polling de 2 segundos no worker como componente dominante da latência mínima.
- A transação de mudança de status não pode sofrer I/O de rede; a inserção na outbox é feita na mesma transação local.

**Segurança**

- Toda entrega assinada com HMAC-SHA256, secret por endpoint.
- URL obrigatoriamente HTTPS.
- Secret nunca exposta em listagem, edição ou log.
- Rotação de secret com grace period de 24 horas.
- Replay de DLQ exige role `ADMIN` e é auditado.

**Confiabilidade e integridade**

- Evento e mudança de status são atômicos: status nunca muda sem evento, evento nunca existe para status revertido.
- Entrega at-least-once.
- Payload é snapshot imutável do momento da transição.

**Limites**

- Payload máximo de 64 KB; acima disso gera erro, **nunca truncamento**.
- Timeout de 10 segundos por tentativa de entrega.

**Disponibilidade**

- A indisponibilidade de um cliente externo não pode bloquear a mudança de status de outros pedidos.

**Observabilidade**

- Logs estruturados de enquadramento, tentativa, sucesso, falha, DLQ e replay.
- Consulta ao estado da outbox e da DLQ disponível para operação.

**Limitações conhecidas e aceitas**

- Ordenação garantida apenas por pedido e apenas enquanto houver um único worker.
- Sem escopo por cliente no CRUD nesta fase: qualquer role autenticada opera qualquer customer (endurecimento futuro).

---

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| Outbox no MySQL em vez de broker | Consistência eventual em troca de atomicidade e zero infraestrutura nova | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| 5 tentativas em vez de 3 ou infinito | Evento pode levar ~14h36min para falhar em definitivo, em troca de resistir a manutenções de horas | [ADR-002](adrs/ADR-002-retry-exponencial-e-dlq.md) |
| Secret por endpoint com rotação de 24h | Mais complexidade para o cliente, em troca de confinar vazamentos | [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) |
| At-least-once em vez de exactly-once | Cliente precisa deduplicar, em troca de simplicidade e padronização de mercado | [ADR-004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md) |
| Worker separado com polling de 2s | Até 2s de latência intrínseca, em troca de sobreviver a reinícios da API e não exigir broker | [ADR-005](adrs/ADR-005-worker-processo-separado-com-polling.md) |
| Snapshot do payload na outbox | Payload desnormalizado e sem correção retroativa, em troca de evento imutável e fiel à transição | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md) |

---

## 9. Dependências

**Técnicas**

- MySQL 8.0 disponível (já é a base da aplicação, via `docker-compose.yml`).
- Prisma para as novas tabelas e migrations.
- Processo adicional a operar em produção (`npm run worker`), com deploy e monitoramento próprios.

**Organizacionais**

- **Revisão de segurança** de Sofia, reservados pelo menos 2 dias úteis antes do deploy, com foco em HMAC e geração de secret (`[09:46] Sofia`).
- **Documentação no portal de desenvolvedor** dos clientes, incluindo a semântica de at-least-once e deduplicação (`[09:26] Marcos`, `[09:40] Marcos`).

**Comerciais**

- Alinhamento de prazo com a Atlas: a expectativa declarada é **fim de novembro** (`[09:45] Marcos`), contra uma estimativa de **3 sprints** com revisão de segurança (`[09:46] Larissa`).

---

## 10. Riscos e mitigação

### Risco 1 — Prazo comercial não fecha com a estimativa

- **Probabilidade:** média.
- **Impacto:** alto (risco de perda da Atlas).
- **Mitigação:** dividir a entrega de forma que o caminho crítico (outbox → worker → entrega assinada) chegue primeiro; comunicar o status à Atlas desde o início.
- **Contingência:** priorizar apenas os 3 clientes nomeados, adiando features secundárias.

### Risco 2 — Cliente não implementa verificação de assinatura ou deduplicação

- **Probabilidade:** média.
- **Impacto:** médio (segurança e duplicidade de processamento do lado do cliente).
- **Mitigação:** documentar com destaque no portal, incluindo exemplo de verificação e a semântica at-least-once.
- **Contingência:** suporte de integração assistido aos 3 clientes iniciais.

### Risco 3 — Evento fica preso ou é entregue fora de ordem

- **Probabilidade:** baixa.
- **Impacto:** médio (cliente vê estado inconsistente por um tempo).
- **Mitigação:** worker único com lease; DLQ com replay auditado; cliente consulta `GET /orders/:id` como fonte de verdade.
- **Contingência:** replay manual do evento pela DLQ.

### Risco 4 — Vazamento de secret

- **Probabilidade:** baixa.
- **Impacto:** alto (uma secret exposta permite forjar entregas para aquele cliente).
- **Mitigação:** secret por endpoint (raio de explosão limitado), rotação com grace period, redação de `*.secret` nos logs, revisão de segurança dedicada.
- **Contingência:** rotação imediata da secret comprometida.

---

## 11. Critérios de aceitação

- [ ] É possível cadastrar um endpoint com URL HTTPS e filtro de status, recebendo a secret apenas na criação.
- [ ] URL `http` é recusada com erro de validação identificável.
- [ ] É possível listar, editar e remover endpoints de um cliente; a secret nunca aparece em listagem ou edição.
- [ ] Um endpoint inscrito em `SHIPPED` recebe a notificação quando um pedido vira `SHIPPED`, em menos de 10 segundos.
- [ ] Um endpoint que não se inscreveu no status não recebe evento algum para aquele status.
- [ ] A entrega traz `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
- [ ] A assinatura é verificável pelo cliente com a secret do endpoint.
- [ ] Rotacionar a secret mantém a antiga válida por 24 horas.
- [ ] Falha de entrega dispara retry conforme 1m/5m/30m/2h/12h.
- [ ] Esgotadas as tentativas, o evento está na DLQ com payload, motivo e timestamp.
- [ ] `ADMIN` consegue reprocessar a DLQ e o replay fica registrado com o autor; `OPERATOR` recebe 403.
- [ ] O cliente consulta as últimas 100 entregas com status, payload, resposta e tempo de resposta.
- [ ] Status nunca muda sem o evento correspondente ter sido registrado.
- [ ] Payload acima de 64 KB não é truncado.

---

## 12. Estratégia de testes e validação

**Testes automatizados**

- **Unitários:** regra de filtro de eventos na inserção, montagem do payload snapshot, progressão de backoff, geração e rotação de secret, serialização dos erros `WEBHOOK_*`.
- **Integração:** `changeStatus` gravando a outbox na mesma transação; rollback do status quando a inserção falha; worker consumindo a outbox e gravando `webhook_deliveries`; replay de DLQ por ADMIN.
- **Contrato:** cada endpoint da seção 5 do [FDD](FDD.md) com request e response e status codes; verificação de que a secret não vaza nas respostas.
- **Segurança:** autorização do replay (ADMIN vs OPERATOR); assinatura HMAC aceita e recusada; rejeição de URL `http`.
- **Confiabilidade:** injeção de falha de rede e de resposta não-2xx; progressão de retry com relógio/backoff injetável.

**Validação manual**

- Roteiro ponta a ponta com um receptor HTTP local simulando um cliente: cadastro, mudança de status, recebimento, verificação de assinatura, falha simulada, retry, DLQ e replay.
- Validação exploratória com os 3 clientes piloto em ambiente de homologação.
- Revisão de segurança de Sofia antes do deploy, com foco em HMAC, geração e armazenamento de secret.

**Critério de saída**

- Todos os critérios de aceitação da seção 11 atendidos; suíte automatizada verde incluindo os testes de regressão de pedido; revisão de segurança aprovada.
