# Da Reunião ao Documento — Design Docs de Webhooks de Notificação de Pedidos

## Sobre o desafio

Este repositório entrega o pacote de documentação técnica de uma feature nova do OMS: um **Sistema de Webhooks de Notificação de Pedidos**. A feature já foi decidida numa reunião técnica de ~55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança — mas nada foi registrado além da transcrição da call. O trabalho foi transformar essa transcrição, mais o código real da aplicação, num conjunto de documentos acionável o suficiente para o time começar a implementar.

A entrega é **puramente documental**. Nenhuma linha de `src/`, `prisma/`, `tests/` ou de configuração foi alterada. O código serviu como contexto: cada afirmação sobre a aplicação existente foi verificada lendo o arquivo correspondente, e cada afirmação sobre a feature foi ancorada num timestamp da transcrição. Onde uma afirmação não tinha origem identificável, ela foi marcada como derivação de engenharia ou removida.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
| --- | --- |
| **Claude Code (VS Code)** | Ambiente principal. Leitura de `TRANSCRICAO.md` e de ~25 arquivos de código, extração de evidências, geração e revisão crítica de todos os documentos. |
| **Subagentes de exploração/planejamento** | Varredura do workspace e leitura integral da transcrição para produzir o mapa de evidências usado como insumo do tracker. |
| **WebFetch** | Primeira tentativa de acesso à transcrição remota, antes do clone local. Substituído pela leitura local quando o truncamento ficou evidente. |
| **Script de validação mecânica (Node)** | Cross-check de caminhos de arquivo e de pares `[hh:mm] Falante` contra o repositório e a transcrição. Escrito fora do repositório e apontado para ele por argumento. |

A IA foi usada como produtora, não como oráculo: cada documento passou por pelo menos uma rodada de leitura crítica contra a fonte antes de ser considerado pronto.

---

## Workflow adotado

A ordem seguida foi a sugerida no enunciado, com uma etapa anterior a tudo:

```
0. Setup              fork, clone, baseline intacto
1. Base de evidências transcrição completa + código mapeado → tabela de fatos
2. ADRs               as 7 decisões fechadas, uma por arquivo
3. RFC                consolida proposta, alternativas descartadas e questões em aberto
4. FDD                como construir, sobre as decisões
5. PRD                consolidação de produto, por último entre os grandes
6. TRACKER            varredura dos documentos prontos
7. README             este arquivo
8. Validação          checklist de aceite + cross-checks mecânicos
9. Commit + push      repositório público
```

**Por que ADRs primeiro.** As decisões são o esqueleto do "como implementar". Escrevendo os ADRs antes, o RFC virou consolidação de decisões já fechadas em vez de prosa solta, e o FDD pôde referenciar decisões concretas em vez de reabri-las.

**O que a Fase 1 mudou.** Antes de escrever qualquer documento, li a transcrição inteira localmente e mapeei os símbolos reais do código (`changeStatus`, `canTransition`, `AppError`, `requireRole`, `redactPaths`…). Isso evitou o modo de falha mais comum: documento genérico que poderia descrever qualquer sistema de webhooks.

**Fronteira entre documentos.** Cada documento abre declarando sua pergunta — PRD "por que e o quê", RFC "o que propomos e o que está em aberto", ADR "por que decidimos exatamente assim", FDD "como construir em detalhe". Usei isso como guarda-corpo contra duplicação.

---

## Prompts customizados

Os dois prompts abaixo foram os que mais mudaram a qualidade do resultado.

**1. Extração dirigida de evidências (o que entra e o que NÃO entra)**

```
Leia TRANSCRICAO.md na íntegra e monte uma tabela com as colunas:
| timestamp + falante | tipo | fato em uma linha | destino |

Tipos permitidos: Decisão, Requisito, Restrição, Descarte, Adiamento,
Hook de código, Ruído.

Regras:
- Todo item precisa do timestamp exato no formato [hh:mm] Nome.
- Separe explicitamente o que foi DESCARTADO do que foi ADIADO do que
  ficou PONTO EM ABERTO. São três categorias diferentes.
- Nada de Descarte/Adiamento pode virar Requisito depois.
- Se um item não tem timestamp, ele não entra.
```

Esse prompt foi escrito porque um pedido genérico ("resuma a transcrição") produz um resumo que perde exatamente a distinção mais importante do desafio: a reunião descarta coisas de propósito. Dashboard visual, fallback por e-mail e rate limiting ocupam lugares diferentes — descartado, adiado e ponto em aberto, respectivamente — e o tracker precisa refletir isso.

**2. Auditoria de rastreabilidade antes de fechar um documento**

```
Para cada afirmação deste documento, responda apenas com:
- TRANSCRICAO: [hh:mm] Falante — cite a fala
- CODIGO: caminho/do/arquivo.ts — cite o símbolo
- DERIVADO: sem origem na reunião

Se você não conseguir preencher a localização de um item, marque-o
como suspeito e proponha removê-lo. Não invente justificativa.
```

Esse é o prompt que transforma o tracker de tabela decorativa em mecanismo de controle: ele força a IA a admitir o que derivou em vez de apresentar uma inferência como se fosse fala da reunião.

---

## Iterações e ajustes

O processo não saiu de primeira. Os ajustes concretos:

**1. Descoberta de que não existe data de calendário na transcrição.**
A primeira versão dos documentos colocou uma data inventada nos metadados do RFC e do PRD. Ao reler o cabeçalho de `TRANSCRICAO.md`, vi que ele traz apenas `**Data:** quinta-feira, 09:00` — sem dia, mês ou ano. Corrigi os metadados para datar os documentos pela *reunião registrada*, declarando explicitamente a ausência da data de calendário, em vez de preencher a lacuna com um número plausível.

**2. Confusão entre "adiado" e "descartado" no "Fora de escopo".**
A primeira redação do PRD tratava fallback por e-mail e rate limiting como itens do mesmo tipo. Na transcrição são coisas diferentes: o e-mail foi **adiado** (`[09:37] Larissa`), o rate limiting ficou como **ponto em aberto** para "observar e decidir depois" (`[09:39] Larissa`), e o dashboard foi **descartado** (`[09:40] Larissa`). Reescrevi a seção separando as três categorias e anotei que o rate limiting não pode ser lido como requisito.

**3. Citações de timestamp imprecisas no tracker.**
A primeira varredura do tracker atribuiu algumas linhas ao falante errado — por exemplo, ligar o requisito de "secret nunca reexposta" a `[09:22] Diego` quando a fonte real é `[09:31] Marcos`, e atribuir o risco de não-deduplicação a Marcos quando a preocupação é de Sofia (`[09:25]`). Rodei um cross-check que compara cada par `[hh:mm] Nome` do tracker com os turnos reais da transcrição e corrigi as divergências. Um item que descreve um risco *derivado* (o `X-Timestamp` fora da base assinada) foi reclassificado de `TRANSCRICAO` para `DERIVADO`, porque a reunião menciona o header mas não analisa esse risco.

**4. Um erro de digitação em ADR-001.**
Sobrou um trecho corrompido no meio de uma frase (`[ográfica a transcrição]`). Corrigido para referenciar as falas reais de `[09:40] Bruno` e `[09:41] Diego`.

**5. FDD com contratos incompletos.**
A primeira versão de cinco endpoints mostrava apenas o *response* de alguns deles. A rubrica exige request **e** response por endpoint. Voltei à seção 5 e completei request de listagem, remoção, entregas e replay, além de explicitar o contrato outbound que o cliente recebe.

Foram, no total, cinco ciclos principais de geração → revisão crítica → ajuste.

---

## Como navegar a entrega

**Ordem sugerida de leitura** — do negócio para a implementação:

1. **[docs/PRD.md](docs/PRD.md)** — o problema, o público, o escopo (inclusive o que ficou de fora) e as métricas.
2. **[docs/RFC.md](docs/RFC.md)** — a proposta técnica em nível de arquitetura, as alternativas descartadas e as 7 questões em aberto.
3. **[docs/adrs/](docs/adrs/)** — as 7 decisões fechadas, uma por arquivo, com contexto e consequências. Comece pelo [índice](docs/adrs/README.md).
4. **[docs/FDD.md](docs/FDD.md)** — o "como construir": fluxos, contratos, matriz de erros, observabilidade e a seção de integração com o código existente.
5. **[docs/TRACKER.md](docs/TRACKER.md)** — de onde veio cada afirmação.

**Estrutura dos arquivos entregues:**

```
.
├── README.md                    este relato de processo
├── TRANSCRICAO.md               fonte original (não alterada)
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── README.md            índice dos ADRs
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-retry-exponencial-e-dlq.md
│       ├── ADR-003-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-004-entrega-at-least-once-com-x-event-id.md
│       ├── ADR-005-worker-processo-separado-com-polling.md
│       ├── ADR-006-reuso-dos-padroes-existentes.md
│       └── ADR-007-snapshot-do-payload-na-outbox.md
├── src/                         não alterado
├── prisma/                      não alterado
└── tests/                       não alterado
```

**Se você só tem cinco minutos:** leia o TL;DR do RFC e a tabela de decisões do PRD. Isso dá o desenho completo e o mapa dos trade-offs sem precisar do resto.

---

## Referência

Enunciado original do desafio: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).
