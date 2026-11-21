# RFC — Sistema de Webhooks de Notificação de Pedidos

> **Pergunta que este documento responde:** como pretendemos resolver, e o que ainda está em aberto?
> O detalhamento de implementação está no [FDD](FDD.md). As decisões fechadas estão nos [ADRs](adrs/README.md).

## 1. Metadados

| Campo | Valor |
| --- | --- |
| Título | Sistema de Webhooks de Notificação de Pedidos |
| Autor | Equipe de Pedidos (documento redigido a partir da reunião técnica de `quinta-feira, 09:00`) |
| Status | Em revisão |
| Data | Reunião técnica registrada em `TRANSCRICAO.md`, sem data de calendário |
| Revisores | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma), Sofia (Engenheira de Segurança) |
| ADRs relacionados | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md) |

## 2. TL;DR

Vamos entregar webhooks **outbound** de mudança de status de pedido para clientes B2B, com **alvo de latência abaixo de 10 segundos**. A solução usa o **padrão Outbox dentro do MySQL já existente**: o evento é gravado numa tabela `webhook_outbox` na mesma transação SQL que muda o status do pedido. Um **worker em processo separado**, com **polling de 2 segundos**, lê a outbox e dispara chamadas HTTP assinadas com **HMAC-SHA256** por endpoint. Falhas seguem **5 tentativas com backoff de 1m/5m/30m/2h/12h** e então vão para uma **DLQ** com replay manual restrito a `ADMIN`. A entrega é **at-least-once**, com `X-Event-Id` permitindo deduplicação no cliente.

## 3. Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente notificação em tempo real de mudança de status. Hoje eles fazem polling em `GET /orders`, o que torna a integração lenta e cara para eles. A Atlas sinalizou risco de migrar para o concorrente caso a entrega não aconteça até o fim do trimestre.

Para os clientes, "tempo real" significa qualquer coisa **abaixo de 10 segundos**; o problema real é não precisar mais atualizar manualmente.

A aplicação hoje não tem eventos, filas nem webhooks. O ciclo de vida do pedido é controlado por máquina de estados em `src/modules/orders/order.status.ts`, com controle transacional de estoque e auditoria de mudanças de status em `order_status_history`. Qualquer solução precisa se acoplar a esse ciclo sem comprometer suas garantias.

## 4. Proposta técnica (visão geral)

A arquitetura tem quatro partes e uma regra de ouro.

**Regra de ouro:** o evento e a mudança de status são atômicos. Se a transação de status commita, o evento existe; se dá rollback, o evento desaparece. Status nunca muda sem evento.

1. **Inserção na outbox dentro da transação de status.** `OrderService.changeStatus` passa a chamar uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro do `prisma.$transaction` existente. A função recebe o client transacional, consulta quais endpoints do cliente se inscreveram naquele status, e insere o payload já renderizado em `webhook_outbox`. Se nenhum endpoint quer aquele status, nada é inserido.

2. **Worker separado em polling.** Um processo Node próprio (`src/worker.ts`, script `npm run worker`), com `PrismaClient` próprio apontando para o mesmo banco, varre a outbox a cada 2 segundos em lotes pequenos, do mais antigo para o mais novo, e dispara as chamadas HTTP com timeout de 10 segundos.

3. **Assinatura e entrega.** Cada entrega é um `POST` HTTPS com corpo JSON enxuto (sem itens do pedido) e os headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`. A assinatura é HMAC-SHA256 sobre o corpo, com a secret do endpoint.

4. **Ciclo de vida da falha.** Timeout ou resposta não-2xx agenda retry conforme a progressão fixa. Esgotadas as 5 tentativas, o evento vai para `webhook_dead_letter` com payload, motivo e timestamp. O replay é manual, via endpoint administrativo restrito a `ADMIN`, e fica registrado para auditoria.

A leitura e a escrita ficam separadas por natureza: a API escreve na outbox, o worker é o único consumidor. A API nunca faz HTTP para o cliente.

### Fronteiras do desenho

- **Consistência eventual aceita**: até 2 s de polling mais o tempo da chamada, dentro do alvo de 10 s.
- **Worker único**: a ordenação por pedido depende disso e é uma limitação registrada, não uma garantia global.
- **Outbound apenas**: os clientes recebem; não enviam eventos para nós.

## 5. Alternativas consideradas e descartadas

### 5.1 Chamada HTTP síncrona dentro de `changeStatus`

**O que era:** disparar o webhook no mesmo fluxo da mudança de status.

**Por que foi descartada:** a transação de status já é pesada (atualiza `orders`, insere histórico, mexe em estoque). Um cliente lento travaria a mudança de status de outros pedidos, e a única saída para cliente offline seria rollback da mudança de status — inaceitável (`[09:04] Bruno`, `[09:06] Diego`).

**Trade-off:** ganharíamos entrega imediata, mas ao custo de acoplar a disponibilidade de um cliente externo à operação interna de pedidos.

### 5.2 Redis Streams ou broker externo

**O que era:** publicar eventos num broker (Redis Streams ou equivalente) e consumir de lá.

**Por que foi descartada:** exigiria subir infraestrutura adicional para um time pequeno, caracterizado como *overengineering*. O MySQL existente já resolve (`[09:07] Diego`).

**Trade-off:** ganharíamos push reativo e escala horizontal, mas ao custo de um novo componente operacional, novas credenciais, novo monitoramento e mais superfície de falha.

### 5.3 Trigger de banco notificando o worker

**O que era:** usar um trigger MySQL para reagir à inserção, em vez de polling.

**Por que foi descartada:** MySQL não tem `LISTEN`/`NOTIFY`; trigger só executa SQL e não sinaliza processo externo. Qualquer improviso seria frágil, e polling de 2 s já atende ao requisito (`[09:09] Diego`).

**Trade-off:** ganharíamos latência menor e menos consultas ociosas, mas ao custo de um mecanismo sem suporte nativo na engine.

### 5.4 Retry infinito

**O que era:** nunca desistir, apenas espaçar as tentativas.

**Por que foi descartada:** evento ficaria pendurado para sempre se o cliente desaparecesse, sem estado terminal nem sinal operacional (`[09:15] Diego`).

**Trade-off:** ganharíamos cobertura máxima de indisponibilidade, mas ao custo de um sistema sem fim de vida explícito para o evento.

### 5.5 DLQ marcada como `failed` na própria outbox

**O que era:** usar um status `falhou` na tabela principal em vez de uma tabela separada.

**Por que foi descartada:** poluiria o caminho de leitura da outbox; a tabela separada preserva o payload e o motivo como evidência para depuração e replay (`[09:18] Diego`).

**Trade-off:** ganharíamos uma tabela a menos, mas ao custo de degradar a varredura do caminho quente.

## 6. Questões em aberto

1. **Rate limiting de saída.** Se um cliente tiver 50 pedidos mudando de status em um minuto, devemos bombardeá-lo com 50 chamadas? Foi registrado como ponto em aberto — "observar e decidir depois" (`[09:38] Diego`, `[09:39] Larissa`). Nada é implementado nesta fase.

2. **Ordenação sob escala horizontal.** A ordenação por pedido só existe enquanto houver um único worker. Particionar por `order_id` ou usar lock pessimista foi explicitamente adiado como "problema do futuro" (`[09:13] Diego`).

3. **Formato de transporte das duas secrets durante a rotação.** O grace period de 24 h foi decidido (`[09:21] Sofia`), mas não se definiu como o cliente recebe/valida duas assinaturas ao mesmo tempo — header repetido, segundo header, ou assinaturas concatenadas. Precisa ser fechado com Segurança antes do contrato público ser congelado.

4. **Semântica de `X-Event-Id` em fan-out.** Quando um cliente tem dois endpoints inscritos no mesmo status, não foi definido se o `event_id` é compartilhado entre as entregas ou distinto por endpoint. Isso muda a regra de deduplicação do cliente.

5. **Recuperação de secret perdida.** A secret só é devolvida na criação (`[09:31] Marcos`). Não existe endpoint para reexibir a secret atual; se o cliente a perder, o caminho previsto é rotacionar.

6. **Retenção e arquivamento da outbox e da DLQ.** Arquivar linhas entregues após ~30 dias foi explicitamente deixado fora do escopo desta feature (`[09:08] Diego`). Não há política de retenção definida.

7. **Comportamento quando o payload excede 64 KB.** O teto e a decisão de errar foram fixados (`[09:24] Larissa`), mas não se definiu se o erro impede a mudança de status (rollback) ou apenas descarta o evento com log/métrica. Precisa ser fechado antes de escrever o critério de aceite.

## 7. Impacto e riscos

| Área | Impacto |
| --- | --- |
| `OrderService.changeStatus` | Passa a ter uma dependência nova dentro da transação (`publishWebhookEvent`). É a única alteração em lógica de negócio existente. |
| Banco de dados | Quatro tabelas novas (`webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`) com índices em status e `created_at`. |
| Operação | Um novo processo a operar (worker), com deploy, healthcheck e métricas próprios. |
| Segurança | Primeiro canal de saída de dados para terceiros; exige revisão de Sofia (2 dias úteis reservados, `[09:46] Sofia`). |
| Testes | Novas tabelas com FK para `customers` afetam a ordem de limpeza de `tests/setup.ts`. |

**Riscos principais:**

- **Acoplamento ao caminho crítico de pedidos.** A consulta de inscrições e a montagem do payload passam a rodar dentro da transação de status. Mitigação: consulta única indexada, sem I/O de rede dentro da transação.
- **Ordem quebrada por retry.** Um evento em retry pode ser entregue depois de um evento mais novo do mesmo pedido. A garantia de ordenação vale apenas para entregas bem-sucedidas na primeira tentativa.
- **Duplicatas.** At-least-once significa que o cliente pode receber o mesmo evento mais de uma vez e precisa deduplicar.
- **Prazo.** Três sprints incluindo a revisão de segurança, contra a expectativa da Atlas para fim de novembro (`[09:45] Marcos`, `[09:46] Larissa`).

## 8. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry com backoff exponencial e DLQ](adrs/ADR-002-retry-exponencial-e-dlq.md)
- [ADR-003 — HMAC-SHA256 com secret por endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-least-once com `X-Event-Id`](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)
- [ADR-005 — Worker em processo separado com polling](adrs/ADR-005-worker-processo-separado-com-polling.md)
- [ADR-006 — Reuso dos padrões existentes](adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Snapshot do payload na outbox](adrs/ADR-007-snapshot-do-payload-na-outbox.md)

## 9. Como este documento se relaciona com os demais

| Documento | Papel | Onde olhar |
| --- | --- | --- |
| Este RFC | O que propomos e o que está em aberto | Aqui |
| [FDD](FDD.md) | Como construir em detalhe: fluxos, contratos, erros, integração com o código | Implementação |
| [ADRs](adrs/README.md) | Por que decidimos exatamente assim, decisão por decisão | Racional |
| [PRD](PRD.md) | Por que e o quê, do ponto de vista de produto | Negócio |
| [TRACKER](TRACKER.md) | De onde veio cada afirmação | Rastreabilidade |
