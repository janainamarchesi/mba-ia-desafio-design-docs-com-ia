# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante é registrada em um arquivo individual, no formato
MADR, nomeado como `ADR-NNN-titulo-em-kebab-case.md`.

## Índice

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para eventos de mudança de status | Aceito |
| [ADR-002](ADR-002-retry-exponencial-e-dlq.md) | Retry com backoff exponencial e DLQ em tabela separada | Aceito |
| [ADR-003](ADR-003-hmac-sha256-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period | Aceito |
| [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md) | Garantia at-least-once com deduplicação por `X-Event-Id` | Aceito |
| [ADR-005](ADR-005-worker-processo-separado-com-polling.md) | Worker em processo separado com polling de 2 segundos | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto para o módulo de webhooks | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-na-outbox.md) | Snapshot do payload renderizado na inserção da outbox | Aceito |

## Formato

Cada ADR contém as seções: **Status**, **Contexto**, **Decisão**, **Alternativas Consideradas**
e **Consequências** (positivas e negativas, com o trade-off explícito), seguido de referências
à transcrição da reunião e ao código-fonte.
