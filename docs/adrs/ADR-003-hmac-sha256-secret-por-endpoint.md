# ADR-003 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito

## Contexto

Os eventos carregam dados de pedidos e são enviados para um endpoint HTTPS fora da infraestrutura da plataforma. O cliente precisa conseguir verificar duas coisas: que a requisição veio realmente de nós, e que ninguém adulterou o payload no meio do caminho (`[09:19] Sofia`).

A plataforma não possui hoje nenhum mecanismo de assinatura de mensagens. Este é o primeiro canal de saída para terceiros.

Há um antecedente relevante citado na reunião: já houve cliente que vazou secret em log da própria aplicação (`[09:22] Diego`). A escolha do escopo da secret precisa levar esse risco em conta.

## Decisão

1. **HMAC-SHA256** sobre o **corpo do request**, com uma **secret compartilhada** entre a plataforma e o cliente.
2. A assinatura é enviada no header **`X-Signature`**.
3. A secret é **única por endpoint de webhook**. Não existe secret global da plataforma.
4. A secret é **rotacionável** por um endpoint de API. Durante a rotação, a secret antiga permanece válida **por 24 horas em paralelo**, para o cliente migrar seus sistemas; depois disso, a antiga é invalidada.
5. A URL do endpoint deve ser **HTTPS**. Cadastro com `http` é recusado na validação — porém, conforme `[09:23] Sofia`, isso é uma validação de schema Zod, não uma decisão arquitetural.

## Alternativas Consideradas

### Secret global única da plataforma

Descartada em `[09:21] Sofia`. Se vazasse uma vez, todos os clientes estariam comprometidos. Secret por endpoint confina o raio de explosão a um único cliente — exatamente o cenário do vazamento em log citado em `[09:22] Diego`.

### Não assinar, confiar apenas em HTTPS

Descartada implicitamente em `[09:19] Sofia`. TLS protege o transporte, mas não permite ao cliente provar a origem nem detectar adulteração por um intermediário com acesso ao TLS. A assinatura é o padrão de mercado.

### Algoritmo alternativo (ex.: SHA-1, MD5)

Não chegou a ser colocado na mesa como opção: Sofia definiu SHA-256 como padrão de mercado, e todo cliente sério tem biblioteca disponível (`[09:20] Sofia`).

## Consequências

### Positivas

- Integridade e autenticidade verificáveis pelo cliente sem segredo compartilhado de transporte.
- Vazamento de uma secret compromete um único endpoint, não a plataforma inteira.
- A rotação com grace period de 24 h permite migração sem downtime do lado do cliente.
- HMAC-SHA256 é suportado nativamente por praticamente toda stack cliente, reduzindo atrito de integração.

### Negativas

- **Custo no cliente**: ele precisa implementar verificação de assinatura. Sofia explicitou essa transferência de responsabilidade, e ela foi aceita como padrão de mercado.
- **Complexidade de rotação**: dois segredos válidos simultaneamente exigem que a verificação aceite ambos durante a janela. O mecanismo exato de transporte das duas assinaturas não foi definido na reunião (ver questão em aberto no [RFC](../RFC.md)).
- **Armazenamento da secret em repouso** não foi discutido. A secret fica na tabela de configuração de webhook, e o logger do projeto não redige `*.secret` hoje (ver `src/shared/logger/index.ts` e a seção de integração do [FDD](../FDD.md)).
- A base assinada é o corpo do request apenas. O `X-Timestamp` existe para o cliente detectar replay por conta própria, mas não faz parte da string assinada — o que permite replay de um corpo válido com timestamp novo. Isso está registrado como risco no [FDD](../FDD.md).

## Referências

- Transcrição: `[09:19] Sofia`, `[09:20] Sofia`, `[09:20] Bruno`, `[09:21] Sofia`, `[09:21] Bruno`, `[09:22] Diego`, `[09:22] Sofia`, `[09:23] Sofia`, `[09:44] Diego`, `[09:44] Sofia`
- Código: `src/shared/logger/index.ts` (paths de redação que não cobrem `*.secret`)
- Relacionados: [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)
