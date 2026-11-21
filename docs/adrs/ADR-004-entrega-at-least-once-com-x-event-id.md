# ADR-004 — Garantia at-least-once com deduplicação por `X-Event-Id`

## Status

Aceito

## Contexto

Qualquer sistema de entrega por rede com retry precisa escolher sua garantia de entrega. Com retry após timeout, o emissor não sabe se o cliente recebeu a requisição antes de falhar — então a escolha real é entre:

- **at-most-once**: nunca duplica, mas pode perder;
- **at-least-once**: nunca perde, mas pode duplicar;
- **exactly-once**: nem perde nem duplica.

O OMS hoje não tem nenhum mecanismo de idempotência de entrega. A reunião precisou escolher uma das três, considerando o custo de implementação de cada uma para a plataforma e para o cliente.

## Decisão

Adotar **at-least-once**: a plataforma pode entregar o mesmo evento mais de uma vez, e o cliente deve estar preparado para isso.

Para permitir a deduplicação do lado do cliente, cada entrega carrega o header **`X-Event-Id`** com o **UUID gerado no momento em que o evento entra na outbox**. O identificador é único por evento e estável entre retentativas daquele mesmo evento.

## Alternativas Consideradas

### Exactly-once

Descartada em `[09:25] Diego`. Exigiria coordenação dos dois lados e é muito mais complexo. A posição registrada foi a de que at-least-once com `event_id` resolve 99% dos casos, e que Stripe e GitHub operam assim.

### At-most-once (sem retry, ou retry apenas em falhas de conexão pré-envio)

Não foi formalmente proposta, mas é o contraponto lógico da decisão. Seria incompatível com o requisito de negócio: um cliente com indisponibilidade temporária simplesmente perderia a notificação, e o pedido dele teria mudado sem ele saber. A política de retry de 5 tentativas do [ADR-002](ADR-002-retry-exponencial-e-dlq.md) só faz sentido sob at-least-once.

## Consequências

### Positivas

- Nenhuma notificação é perdida por falha transitória de rede ou indisponibilidade do cliente.
- Implementação simples na plataforma: nenhum protocolo de confirmação de duas fases.
- Padrão de mercado, com bibliotecas e documentação amplamente disponíveis para o cliente.
- O `X-Event-Id` também serve como chave de correlação em logs e suporte.

### Negativas

- **Duplicatas são possíveis** e a responsabilidade de deduplicação é do cliente. Sofia registrou essa transferência explicitamente (`[09:25] Sofia`), e ela foi aceita.
- O cliente precisa persistir os `event_id` já processados — custo real de armazenamento e implementação do lado dele.
- Marcos se comprometeu a documentar isso de forma destacada no portal de desenvolvedor (`[09:26] Marcos`), então a documentação para o cliente é uma dependência de entrega.

## Nota sobre unicidade

O `X-Event-Id` identifica um evento. Quando o mesmo cliente tem dois endpoints cadastrados inscritos no mesmo status, a semântica de fan-out (um `event_id` compartilhado ou um por endpoint) **não foi definida na reunião**. Ver questão em aberto no [RFC](../RFC.md) e no [FDD](../FDD.md).

## Referências

- Transcrição: `[09:24] Diego`, `[09:25] Bruno`, `[09:25] Diego`, `[09:25] Sofia`, `[09:26] Marcos`, `[09:26] Larissa`, `[09:51] Larissa`
- Relacionados: [ADR-002](ADR-002-retry-exponencial-e-dlq.md), [ADR-007](ADR-007-snapshot-do-payload-na-outbox.md)
