# ADR-002: Worker em processo separado com polling de 2 segundos

| Campo | Valor |
|---|---|
| Status | Aceito |
| Data | Quinta-feira (conforme cabeçalho da transcrição; dia de calendário não registrado) |
| Decisores | Larissa (Tech Lead), Diego (Eng. Sênior Plataforma), Bruno (Eng. Pleno Pedidos), Marcos (PM — validou a latência) |

## Contexto

Decidido o padrão Outbox no MySQL ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)), a pergunta seguinte foi "como o worker lê isso?" ([09:08] Larissa). O requisito de produto é latência "abaixo de 10 segundos" ([09:02] Marcos). Restrições técnicas relevantes:

- MySQL não possui mecanismo de notificação a processos externos equivalente ao NOTIFY/LISTEN do Postgres ([09:09] Diego).
- Um restart da API não pode interromper o envio de webhooks ([09:11] Diego).
- O projeto já tem o padrão de entry-point em `src/server.ts` ([09:11] Larissa).

## Decisão

O consumo da outbox será feito por um **worker em processo separado**, com **polling em loop a cada 2 segundos**: "A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca" ([09:09] Diego). Registrado por Larissa em [09:10]: "Worker em polling, 2s."

Estrutura e infraestrutura:

- Novo entry-point **`src/worker.ts`**, análogo ao `src/server.ts` existente, com script **`npm run worker`** ([09:11] Larissa: "Tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um script 'npm run worker'").
- "Sim, mesmo banco, mesma stack. Só não pode ser o mesmo processo" ([09:11] Diego). A lógica de processamento fica dentro do módulo, em algo como `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
- O worker abre um **PrismaClient próprio**, com a mesma `DATABASE_URL`: "PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node" ([09:30] Bruno).
- Operação **single-worker**: com um único worker, os eventos são processados em ordem de `created_at` da outbox; a garantia de ordering é "implícita por order_id" e só vale enquanto for single-worker ([09:12] Diego, [09:13] Larissa: "Não é garantia de ordering global, só por order_id e enquanto for single-worker"). Os clientes nunca pediram ordering global ([09:14] Marcos).
- Timeout de 10 segundos por chamada HTTP; cliente que não responde é tratado como falha e marcado para retry ([09:42] Diego).

## Alternativas Consideradas

### Trigger de banco para notificação reativa

Proposta por Bruno ("Não dá pra usar trigger do banco pra ser mais reativo?", [09:09]). Rejeitada por Diego no mesmo minuto: "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres. Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL." Avisar o worker exigiria improvisos ("escrever em arquivo ou bater num endpoint, fica esquisito"), enquanto "Polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo" ([09:09] Diego).

### Rodar o worker dentro do processo da API

Executar o loop de envio na mesma instância da API. Rejeitada: "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker" ([09:11] Diego). O restart/deploy da API não pode derrubar o envio de webhooks.

## Consequências

### Positivas

- Envio de webhooks isolado do ciclo de vida da API: deploys e restarts da API não interrompem o worker ([09:11] Diego).
- Latência máxima de polling (2s) confortavelmente dentro do requisito de 10s ([09:09] Diego; [09:10] Marcos: "2 segundos serve, perfeito").
- Reuso total da stack: mesmo banco, mesmo Prisma, mesmo padrão de entry-point do projeto ([09:11] Diego e Larissa).
- Com single-worker, o cliente recebe os eventos de um mesmo pedido na ordem correta ([09:12] Diego).

### Negativas

- Latência mínima de até 2 segundos no pior caso, aceita explicitamente: "A latência mínima vai ser 2 segundos no pior caso. Aceitamos" ([09:10] Larissa).
- Escalar para múltiplos workers em paralelo quebra a garantia de ordering; a solução (particionar por order_id ou lock pessimista) foi adiada — "é problema do futuro, não agora" ([09:13] Diego). Documentado como limitação conhecida ([09:13] Larissa).
- Polling gera consultas constantes ao banco mesmo sem eventos pendentes (mitigado pelos índices e batch pequeno de [09:08], ver [ADR-001](ADR-001-padrao-outbox-no-mysql.md)).

## Referências

- Transcrição: [09:09] Diego — "Polling em loop. A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca."
- Transcrição: [09:11] Diego — "o worker tem que rodar como processo separado [...] se a API reinicia, perde o worker."
- Transcrição: [09:11] Larissa — "criar um src/worker.ts e um script 'npm run worker'."
- Transcrição: [09:30] Bruno — "Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node."
- Transcrição: [09:13] Diego — "isso é problema do futuro, não agora."
- Relacionado: [ADR-001](ADR-001-padrao-outbox-no-mysql.md) — a tabela que o worker consome.
- Relacionado: [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) — tratamento de falhas de entrega pelo worker.
- Relacionado: [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) — padrões de código reaproveitados no worker.
