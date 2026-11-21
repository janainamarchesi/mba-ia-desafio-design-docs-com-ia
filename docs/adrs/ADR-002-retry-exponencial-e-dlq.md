# ADR-002 — Retry com backoff exponencial e DLQ em tabela separada

## Status

Aceito

## Contexto

Um webhook pode falhar porque o cliente está fora do ar, reiniciando, ou sob manutenção planejada. A plataforma precisa de uma política explícita de nova tentativa e de um estado terminal para eventos que nunca serão entregues — caso contrário o evento fica pendurado indefinidamente e ninguém tem visibilidade.

A reunião discutiu três variáveis independentes: quantas tentativas, qual progressão de backoff, e onde persistir o evento que esgotou as tentativas.

## Decisão

1. **Cinco tentativas** com backoff exponencial de intervalos crescentes: **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas**. A soma dos intervalos é de aproximadamente 14h36min entre a primeira falha e a última tentativa.
2. Após esgotar as tentativas, o evento é considerado **falha permanente** e movido para uma **tabela separada `webhook_dead_letter`**, contendo o payload, o motivo da falha e o timestamp.
3. A reprocessagem é **manual**, por um endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Esse endpoint exige a role `ADMIN` (ver [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) para o mecanismo de autorização).

## Alternativas Consideradas

### Retry infinito com backoff

Descartada em `[09:15] Diego`: um evento ficaria pendurado para sempre se o cliente desaparecesse, sem estado terminal, sem DLQ e sem sinal operacional.

### Três tentativas

Descartada em `[09:16] Diego`. Três tentativas com backoff agressivo matariam o evento dentro de ~30 minutos. Já houve cliente com indisponibilidade de duas horas em manutenção planejada, e uma indisponibilidade matinal consumiria todas as tentativas antes do cliente voltar. Cinco tentativas cobrem uma janela de 12 a 24 horas.

### Marcar `failed` na própria tabela de outbox em vez de DLQ separada

Descartada em `[09:18] Diego`. Poluiria o caminho de leitura da outbox principal. Uma tabela separada mantém a varredura limpa e preserva o payload e o motivo da falha como evidência para depuração e reprocessamento.

## Consequências

### Positivas

- Um cliente com indisponibilidade de horas (manutenção planejada, incidente) ainda recebe o evento quando volta.
- O estado terminal é explícito: nenhum evento fica pendurado para sempre.
- A DLQ é evidência auditável e permite reprocessamento controlado.
- A leitura da outbox principal permanece eficiente, porque só contém eventos ainda em ciclo de vida.

### Negativas

- **Latência longa até o desfecho**: um evento genuinamente perdido só é declarado falho após ~14h36min. Aceito explicitamente por Marcos: "se um cliente meu cair por 15 horas, ele já tá com problema sério dele" (`[09:17]`).
- **Reprocessamento é manual**: exige um operador ADMIN agir. Não há recuperação automática depois da DLQ. A notificação ao cliente sobre a falha (e-mail) foi adiada para fase futura (`[09:37] Larissa`).
- A DLQ cresce sem política de retenção definida nesta fase.
- O replay recoloca o evento como pendente, mas não redefine a contagem de tentativas de forma automática — o comportamento precisa ser definido em implementação (ver issue em aberto no [RFC](../RFC.md)).

## Tabela de referência

| Tentativa | Intervalo desde a falha anterior | Acumulado |
| --- | --- | --- |
| 1ª retry | 1 min | 0h01 |
| 2ª retry | 5 min | 0h06 |
| 3ª retry | 30 min | 0h36 |
| 4ª retry | 2h | 2h36 |
| 5ª retry | 12h | 14h36 |
| Esgotou | — | DLQ |

## Referências

- Transcrição: `[09:14] Larissa`, `[09:15] Diego`, `[09:15] Bruno`, `[09:16] Bruno`, `[09:16] Diego`, `[09:17] Diego`, `[09:17] Marcos`, `[09:17] Larissa`, `[09:18] Diego`, `[09:18] Bruno`, `[09:19] Larissa`, `[09:35] Diego`, `[09:36] Sofia`, `[09:36] Larissa`
- Relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)
