# ADR-006: Reuso dos padrões existentes do projeto

| Campo | Valor |
|---|---|
| Status | Aceito |
| Data | Quinta-feira, 09:00 (conforme cabeçalho da transcrição; data calendário não registrada) |
| Decisores | Bruno (Eng. Pleno Pedidos), Larissa (Tech Lead), Diego (Eng. Sênior Plataforma), Sofia (Segurança) |

## Contexto

A codebase do OMS (Node.js/TypeScript/Express/Prisma) já tem convenções consolidadas: "Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas" ([09:27] Bruno) — como se vê em `src/modules/orders/` (order.controller.ts, order.service.ts, order.repository.ts, order.routes.ts, order.schemas.ts). Havia a escolha entre criar convenções próprias para o módulo de webhooks ou aderir integralmente ao que existe. A equipe é pequena e o prazo é de três sprints ([09:46] Larissa), o que pesa contra qualquer invenção desnecessária.

## Decisão

Registrada por Larissa em [09:30]: "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros."

Em detalhe:

1. **Módulo `src/modules/webhooks`** seguindo o padrão controller/service/repository/routes/schemas dos módulos existentes ([09:27] Bruno: "Vou propor uma pasta src/modules/webhooks com toda a estrutura"), espelhando `src/modules/orders/`.
2. **Worker**: entry-point separada em `src/worker.ts` e a lógica de processamento dentro do módulo, "tipo src/modules/webhooks/webhook.worker.ts ou webhook.processor.ts" ([09:28] Bruno). O worker abre um `PrismaClient` próprio — "PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node" ([09:30] Bruno).
3. **Erros**: reuso da classe base `AppError` (`src/shared/errors/app-error.ts`) e do padrão de erros específicos como `InvalidStatusTransitionError` e `InsufficientStockError` (`src/shared/errors/http-errors.ts`), com novos códigos prefixados `WEBHOOK_` — ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno; [09:29] Larissa: "Prefixo WEBHOOK_ pra tudo do módulo").
4. **Logger e middleware de erro**: reuso do Pino (`src/shared/logger/index.ts`) e do middleware de erro centralizado (`src/middlewares/error.middleware.ts`), que já trata `AppError`, `ZodError` e erros do Prisma — "Vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno).
5. **Validação**: schemas Zod por módulo aplicados via middleware `validate` (`src/middlewares/validate.middleware.ts`), inclusive a validação de URL HTTPS ([09:23] Sofia; ver [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
6. **Autorização**: o endpoint de replay da DLQ exige role ADMIN reaproveitando o `requireRole` existente (`src/middlewares/auth.middleware.ts`) — "a gente reaproveita o requireRole que já existe" ([09:36] Larissa).
7. **Integração com orders**: a inserção na outbox acontece dentro da transação do método `changeStatus` de `src/modules/orders/order.service.ts` ([09:40] Bruno: "a alteração crítica é dentro do service de orders, no método changeStatus [...] Se a outbox falhar de inserir, rollback"), via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o transaction client da transação atual ([09:41] Bruno).
8. **IDs**: UUID, "segue o padrão do resto do projeto. Tudo é uuid" ([09:51] Larissa) — como os `@default(uuid())` do `prisma/schema.prisma`.
9. **Rotas**: registradas junto às demais em `src/routes/index.ts`, montadas sob `/api/v1` em `src/app.ts`.

## Alternativas Consideradas

### Criar convenções novas/isoladas para o módulo de webhooks

Estrutura de pastas, tratamento de erro ou logger próprios do módulo. Descartada implicitamente durante todo o bloco [09:27]–[09:30]: cada item (estrutura, erros, logger, validação) foi resolvido apontando para o padrão existente, culminando na decisão de "reuso máximo do que já existe" ([09:30] Larissa). Convenções novas aumentariam carga cognitiva e custo de manutenção sem benefício, num time pequeno com prazo de três sprints ([09:46] Larissa).

### Injetar o repository de webhooks no OrderService

Para inserir na outbox dentro da transação de `changeStatus`, uma opção seria injetar o repository inteiro do módulo de webhooks no `OrderService` ([09:41] Bruno: "Vai me obrigar a passar um repository do webhook pro OrderService ou uma função de 'enqueue event'"). Descartada em favor da função pura: "Boa, função pura recebendo o tx. Não precisa injetar repository inteiro" ([09:41] Diego). A função pura minimiza o acoplamento entre os módulos e mantém a atomicidade, já que opera sobre o mesmo transaction client.

## Consequências

### Positivas

- Nenhuma convenção nova para aprender: quem conhece `src/modules/orders/` entende o módulo de webhooks.
- Erros de webhook (`WEBHOOK_*`) são tratados pelo `src/middlewares/error.middleware.ts` sem qualquer alteração no middleware ([09:29] Bruno).
- Atomicidade garantida na integração: status e evento na outbox commitam ou dão rollback juntos, na mesma transação de `changeStatus` ([09:40] Bruno, [09:41] Diego: "Se ficar fora da transação, perde a garantia toda").
- Acoplamento mínimo entre `orders` e `webhooks`: o `OrderService` conhece apenas a assinatura de `publishWebhookEvent(tx, order, fromStatus, toStatus)` ([09:41] Bruno/Diego).
- Autorização do replay ADMIN sem código novo de auth, via `requireRole` ([09:36] Larissa).

### Negativas

- O módulo herda as limitações dos padrões atuais: se uma convenção existente se mostrar inadequada para webhooks, mudá-la impacta todos os módulos.
- O `OrderService` passa a ter uma dependência (ainda que mínima) do domínio de webhooks — a mudança de status ganha um passo a mais na transação.
- O worker duplica a instância de `PrismaClient` por ser outro processo ([09:30] Bruno), dobrando conexões de pool contra o mesmo MySQL.

## Referências

- Transcrição: [09:27] Bruno — "Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas"
- Transcrição: [09:28] Bruno — "src/modules/webhooks/webhook.worker.ts ou webhook.processor.ts"; "Códigos tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED"
- Transcrição: [09:29] Bruno — "O middleware de erro centralizado já trata AppError, Zod e Prisma"
- Transcrição: [09:30] Larissa — "Decisão: reuso máximo do que já existe"
- Transcrição: [09:36] Larissa — "a gente reaproveita o requireRole que já existe"
- Transcrição: [09:41] Diego — "Boa, função pura recebendo o tx. Não precisa injetar repository inteiro"
- Transcrição: [09:51] Larissa — "UUID, segue o padrão do resto do projeto"
- Código: `src/modules/orders/order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.routes.ts`, `order.schemas.ts` — padrão de módulo a espelhar
- Código: `src/shared/errors/app-error.ts` (classe base `AppError`); `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError`, `InsufficientStockError`)
- Código: `src/shared/logger/index.ts` (Pino); `src/middlewares/error.middleware.ts` (trata `AppError`/`ZodError`/Prisma)
- Código: `src/middlewares/validate.middleware.ts` (`validate`); `src/middlewares/auth.middleware.ts` (`requireRole`)
- Código: `src/modules/orders/order.service.ts` (método `changeStatus`, transação Prisma); `src/routes/index.ts` e `src/app.ts` (montagem sob `/api/v1`)
- Relacionado: [ADR-001](ADR-001-padrao-outbox-no-mysql.md), [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)
