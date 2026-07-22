# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| Autora | Larissa (Tech Lead), com Bruno (Pedidos) e Diego (Plataforma) |
| Revisão de segurança | Sofia — mínimo 2 dias úteis antes do deploy ([09:46] Sofia) |
| Status | Em revisão |
| Documentos relacionados | [RFC](RFC.md) · [PRD](PRD.md) · [ADRs](adrs/) |

> **Nota sobre rastreabilidade**: toda decisão neste documento está ancorada na transcrição da reunião ([hh:mm] Nome, conforme `TRANSCRICAO.md`) ou no código existente do repositório. Itens sem âncora são **derivações de engenharia** dos padrões já presentes na codebase e estão marcados como tal.

---

## 1. Contexto e motivação técnica

O OMS (Node.js/TypeScript/Express/Prisma/MySQL) não possui nenhum mecanismo de eventos ou notificação externa. Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) hoje fazem polling em `GET /orders` e exigem notificação em menos de **10 segundos** após mudança de status ([09:02] Marcos).

O ponto técnico central é a transação de `changeStatus` em `src/modules/orders/order.service.ts`: dentro de um único `prisma.$transaction`, ela valida a transição (`canTransition`), debita/repõe estoque, atualiza `orders` e insere em `order_status_history`. Essa transação "já é pesada" ([09:04] Bruno) — qualquer chamada HTTP síncrona dentro dela travaria mudanças de status de outros pedidos e criaria o dilema insolúvel de rollback por cliente fora do ar ([09:04] Bruno; [09:06] Diego: "Síncrono está fora de questão").

A solução decidida é o **padrão Outbox no MySQL existente** ([09:08] Larissa; [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md)): o evento é gravado na mesma transação da mudança de status, e um worker em processo separado ([ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)) entrega por HTTP com assinatura HMAC-SHA256 ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)), retry com backoff e DLQ ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)). Este FDD detalha a implementação; a proposta em nível de arquitetura está no [RFC](RFC.md).

## 2. Objetivos técnicos

| # | Objetivo | Medida de verificação | Origem |
|---|---|---|---|
| O1 | Latência fim-a-fim ≤ 10s entre commit da mudança de status e POST no cliente | Polling do worker a cada 2s + envio; latência mínima ~2s no pior caso do ciclo | [09:02] Marcos; [09:09–09:10] Diego/Larissa |
| O2 | Atomicidade: evento na outbox se e somente se a transição comitou | Rollback do `$transaction` descarta o evento junto; falha de inserção na outbox faz rollback da transição | [09:06] Diego; [09:40] Bruno |
| O3 | Entrega at-least-once com dedup pelo cliente | `X-Event-Id` (UUID gerado na inserção na outbox) presente em toda entrega, estável entre retries | [09:24–09:26] Diego/Larissa; [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| O4 | Resiliência a indisponibilidade do cliente por até ~15h | 5 tentativas com backoff 1m/5m/30m/2h/12h; depois DLQ | [09:17] Larissa |
| O5 | Autenticidade e integridade verificáveis pelo cliente | HMAC-SHA256 do corpo em `X-Signature`; secret única por endpoint; rotação com grace de 24h | [09:22] Sofia |
| O6 | Zero infraestrutura nova | Apenas MySQL + Prisma existentes; nenhum broker/fila externa | [09:07] Diego/Larissa |
| O7 | Zero regressão na API existente | Nenhum endpoint atual muda; suíte `tests/orders.test.ts` continua verde | [09:29–09:30] Bruno/Larissa (reuso máximo) |
| O8 | Timeout de 10s por chamada HTTP do worker | Cliente que não responde em 10s conta como falha e entra no ciclo de retry | [09:42] Diego |
| O9 | Payload ≤ 64KB | Evento acima do limite gera erro (não trunca, não envia) | [09:23–09:24] Sofia/Diego/Larissa |

## 3. Escopo e exclusões

**No escopo**: novo módulo `src/modules/webhooks` (padrão controller/service/repository/routes/schemas — [09:27] Bruno); novas tabelas (endpoint, outbox, dead letter, deliveries); worker `src/worker.ts` + `npm run worker` ([09:11] Larissa); CRUD de configuração, rotação de secret, histórico de entregas, replay admin de DLQ; integração no `changeStatus`.

**Exclusões explícitas da reunião**:

| Exclusão | Decisão | Âncora |
|---|---|---|
| E-mail ao cliente quando webhook falha | "Email tá fora de escopo dessa fase. Talvez próxima fase" | [09:37] Larissa |
| Dashboard/painel visual para o cliente | "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" | [09:39–09:40] Marcos/Larissa |
| Arquivamento de linhas entregues da outbox | "Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature" | [09:08] Diego |
| Múltiplos workers em paralelo | "Aí dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro" | [09:13] Diego |
| Rate limiting de envio para o cliente | Ponto **em aberto**: "a gente observa e implementa se virar problema" | [09:38–09:39] Diego/Larissa |
| Endurecimento de roles no CRUD | "Por enquanto sim [qualquer role autenticada]. Mais pra frente a gente pode endurecer" | [09:36–09:37] Marcos/Sofia |
| Exactly-once | Exigiria coordenação dos dois lados; at-least-once "resolve 99% dos casos" | [09:25] Diego |

## 4. Arquitetura e modelos de dados

```mermaid
flowchart LR
    subgraph API["Processo API (src/server.ts)"]
        CS["OrderService.changeStatus<br/>$transaction"] -->|publishWebhookEvent(tx,...)| OB[("webhook_outbox")]
        CRUD["src/modules/webhooks<br/>CRUD + rotate + deliveries"] --> WE[("webhook_endpoints")]
        REPLAY["POST /admin/.../replay (ADMIN)"] --> DLQ
    end
    subgraph W["Processo Worker (src/worker.ts, polling 2s)"]
        P["batch pendentes mais antigos"] --> OB
        P -->|"POST HMAC, timeout 10s"| CLI["HTTPS do cliente"]
        P --> DEL[("webhook_deliveries")]
        P -->|"5ª falha"| DLQ[("webhook_dead_letter")]
    end
```

> **Importante**: os modelos abaixo são **PROPOSTAS de schema**. A entrega deste desafio é documental — `prisma/schema.prisma` **não é alterado** neste repositório. Os modelos seguem as convenções do schema existente: `@id @default(uuid()) @db.Char(36)` ([09:51] Larissa: "UUID, segue o padrão do resto do projeto"), `@@map` snake_case, `createdAt`/`updatedAt`, índices explícitos.

```prisma
enum WebhookOutboxStatus {
  PENDING     // pendente     — [09:08] Diego
  PROCESSING  // processando
  FAILED      // falhou (aguardando retry)
  DELIVERED   // entregue
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)      // https obrigatório — [09:23] Sofia
  secret                  String    @db.VarChar(128)       // gerada pela plataforma — [09:31] Marcos
  previousSecret          String?   @db.VarChar(128)       // rotação com grace — [09:21] Sofia
  previousSecretExpiresAt DateTime?                        // now() + 24h na rotação
  events                  Json                             // OrderStatus[] assinados — [09:33] Marcos
  active                  Boolean   @default(true)         // [09:21] Bruno/Sofia
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer          @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id                String              @id @default(uuid()) @db.Char(36) // = event_id — [09:25] Diego, [09:51] Larissa
  webhookEndpointId String              @db.Char(36)
  orderId           String              @db.Char(36)
  eventType         String              @db.VarChar(64)     // "order.status_changed" — [09:43] Diego
  payload           Json                                    // snapshot na inserção — [09:52] Larissa
  status            WebhookOutboxStatus @default(PENDING)
  attempts          Int                 @default(0)
  nextRetryAt       DateTime?                               // agenda do backoff — [09:17] Larissa
  lastError         String?             @db.VarChar(500)
  createdAt         DateTime            @default(now())
  updatedAt         DateTime            @updatedAt

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id])

  @@index([status])       // [09:08] Diego
  @@index([createdAt])    // [09:08] Diego
  @@index([orderId])      // derivado: consulta de ordering por pedido
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id                String   @id @default(uuid()) @db.Char(36)
  outboxEventId     String   @db.Char(36)   // referência ao evento original — [09:18] Diego
  webhookEndpointId String   @db.Char(36)
  orderId           String   @db.Char(36)
  eventType         String   @db.VarChar(64)
  payload           Json                    // payload — [09:18] Diego
  reason            String   @db.VarChar(500) // motivo da falha — [09:18] Diego
  failedAt          DateTime @default(now())  // timestamp — [09:18] Diego
  replayedAt        DateTime?               // derivado: idempotência do replay (ver §8)
  replayedById      String?  @db.Char(36)   // derivado: trilha do replay — auditoria [09:36] Sofia

  @@index([webhookEndpointId])
  @@map("webhook_dead_letter")
}

model WebhookDelivery {
  id                String   @id @default(uuid()) @db.Char(36)
  webhookEndpointId String   @db.Char(36)
  outboxEventId     String   @db.Char(36)
  attemptNumber     Int
  success           Boolean                       // sucesso/falha — [09:34] Marcos
  payload           Json                          // payload enviado — [09:34] Marcos
  responseStatus    Int?                          // response — [09:34] Marcos
  responseBody      String?  @db.VarChar(1024)    // truncado (limite derivado)
  durationMs        Int                           // tempo de resposta — [09:34] Marcos
  createdAt         DateTime @default(now())

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id])

  @@index([webhookEndpointId, createdAt])  // derivado: "últimas ~100" por endpoint — [09:34]
  @@map("webhook_deliveries")
}
```

Notas de modelagem:

- `events` como `Json` (e não relação N:N com enum) porque Prisma/MySQL não suporta array de enum nativo; o conteúdo é validado no Zod contra `z.nativeEnum(OrderStatus)` — mesmo padrão de `updateOrderStatusSchema` em `src/modules/orders/order.schemas.ts`. Os valores possíveis são exatamente os do enum `OrderStatus` existente (`PENDING/PAID/PROCESSING/SHIPPED/DELIVERED/CANCELLED`, `prisma/schema.prisma`).
- Cada endpoint assinante gera **uma linha própria** na outbox por transição (event_id distinto por endpoint) — derivação necessária de [09:34] (filtro por endpoint na inserção) + [09:25] (event_id nasce na inserção).
- `Customer` ganharia a relação inversa `webhookEndpoints WebhookEndpoint[]` (ajuste no model existente, também apenas proposto).

## 5. Fluxos detalhados

### 5.1 Publicação do evento na outbox (dentro do `$transaction` do `changeStatus`)

Ponto exato de integração em `src/modules/orders/order.service.ts` ([09:40–09:41] Bruno/Diego): **após** `tx.orderStatusHistory.create(...)` (linha ~159–167 atual) e **antes** do `findUnique` final/commit.

```ts
// src/modules/webhooks/webhook.publisher.ts — função pura recebendo o tx ([09:41] Diego)
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void> {
  // 1. Filtro NA INSERÇÃO ([09:34] Bruno: "Se nenhum webhook do customer quer aquele status, nem insere")
  const endpoints = await tx.webhookEndpoint.findMany({
    where: { customerId: order.customerId, active: true },
  });
  const listening = endpoints.filter((e) =>
    (e.events as OrderStatus[]).includes(toStatus),
  );
  if (listening.length === 0) return; // economiza linha na tabela ([09:34])

  // 2. Snapshot do payload renderizado na inserção ([09:52] Larissa: "o evento ainda
  //    reflete o estado de quando o status mudou")
  const timestamp = new Date().toISOString();
  for (const endpoint of listening) {
    const eventId = randomUUID(); // UUID = padrão do projeto ([09:51] Larissa)
    await tx.webhookOutbox.create({
      data: {
        id: eventId, // id da linha É o event_id ([09:25] Diego)
        webhookEndpointId: endpoint.id,
        orderId: order.id,
        eventType: 'order.status_changed',
        status: 'PENDING',
        payload: {
          event_id: eventId,
          event_type: 'order.status_changed',
          timestamp,
          order_id: order.id,
          order_number: order.orderNumber,
          from_status: fromStatus,
          to_status: toStatus,
          customer_id: order.customerId,
          total_cents: order.totalCents, // sem items ([09:43] Diego)
        },
      },
    });
  }
}
```

E no `changeStatus`:

```ts
await tx.order.update({ where: { id }, data: { status: to } });
await tx.orderStatusHistory.create({ /* ...existente... */ });
await publishWebhookEvent(tx, order, from, to); // ADICIONADO — mesma transação
// falha aqui => rollback de status + history + estoque ([09:40] Bruno)
```

### 5.2 Loop do worker (`src/worker.ts`)

Processo separado da API ([09:11] Diego: "não pode ser o mesmo processo"), polling de 2 segundos ([09:09] Diego), batch pequeno dos pendentes mais antigos ([09:08–09:09] Diego).

```ts
// src/worker.ts — entry-point análoga a src/server.ts ([09:11] Larissa)
const prisma = new PrismaClient(); // instância própria do processo ([09:30] Bruno)

async function tick(): Promise<void> {
  const now = new Date();
  const batch = await prisma.webhookOutbox.findMany({
    where: {
      OR: [
        { status: 'PENDING' },
        { status: 'FAILED', nextRetryAt: { lte: now } }, // retries vencidos
      ],
    },
    orderBy: { createdAt: 'asc' }, // ordering por created_at ([09:12] Diego)
    take: 20, // "batch pequeno" ([09:08] Diego); valor exato é derivado
  });
  for (const event of batch) {
    await processEvent(prisma, event); // sequencial: preserva ordering por order_id
  }
}

async function main(): Promise<void> {
  for (;;) {
    await tick();
    await sleep(2_000); // polling 2s ([09:09] Diego)
  }
}
```

Passos de `processEvent` (em `src/modules/webhooks/webhook.processor.ts` — [09:28] Bruno):

1. Marca a linha como `PROCESSING` (update guardado por `status` anterior, evitando reprocesso concorrente — derivado).
2. Carrega o `WebhookEndpoint`; se inativo/removido, falha com `WEBHOOK_ENDPOINT_INACTIVE` (vai para o ciclo de falha).
3. Serializa o payload (snapshot já pronto); se `Buffer.byteLength(body) > 64 * 1024`, erra com `WEBHOOK_PAYLOAD_TOO_LARGE` — não trunca, não envia ([09:23–09:24] Sofia/Larissa).
4. Assina: `signature = createHmac('sha256', endpoint.secret).update(body).digest('hex')` ([09:20] Sofia).
5. `POST` na `endpoint.url` com os headers do §6.8 e **timeout de 10s** ([09:42] Diego), via `fetch` + `AbortSignal.timeout(10_000)` (mecanismo derivado — Node 20+ nativo, sem dependência nova).
6. Registra a tentativa em `webhook_deliveries` (sucesso/falha, payload, status/corpo truncado da resposta, duração — [09:34] Marcos).
7. Resposta **2xx** → `status = 'DELIVERED'`. Qualquer outra resposta, erro de rede ou timeout → fluxo de falha (§5.3) ([09:42] Diego).

### 5.3 Retry com backoff exponencial

Decisão: **5 tentativas**, progressão **1m / 5m / 30m / 2h / 12h**, janela total ~15h ([09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h"; [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)).

```ts
const BACKOFF_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000] as const;
// índice = attempts - 1: delay aplicado APÓS a falha da tentativa N, agendando a N+1

async function handleFailure(tx, event, error): Promise<void> {
  const attempts = event.attempts + 1;
  if (attempts >= 5) {
    await moveToDeadLetter(tx, event, error); // §5.4
    return;
  }
  await tx.webhookOutbox.update({
    where: { id: event.id },
    data: {
      status: 'FAILED',
      attempts,
      lastError: truncate(String(error), 500),
      nextRetryAt: new Date(Date.now() + BACKOFF_MS[attempts - 1]),
    },
  });
}
```

| Tentativa | Delay após falha anterior | Tempo acumulado desde a 1ª falha |
|---|---|---|
| 1 | — (imediata, no ciclo de polling) | 0 |
| 2 | 1 min | ~1 min |
| 3 | 5 min | ~6 min |
| 4 | 30 min | ~36 min |
| 5 | 2 h | ~2h36 |
| DLQ | após 12 h de espera da janela final | ~14h36 ("quase 15 horas" — [09:17] Diego) |

### 5.4 DLQ e replay admin

Após a 5ª falha, o evento vai para tabela separada `webhook_dead_letter` com payload, motivo e timestamp ([09:18] Diego), em transação: cria a linha na DLQ e marca a linha da outbox como falha permanente (removendo-a do ciclo de leitura do worker — "mais limpa a leitura da outbox principal", [09:18] Diego).

Replay manual ([09:18] Diego; [09:35] Diego: `POST /admin/webhooks/dead-letter/:id/replay`):

1. Rota protegida por `authenticate` + `requireRole('ADMIN')` ([09:36] Sofia/Larissa — reaproveita o `requireRole` de `src/middlewares/auth.middleware.ts`).
2. Busca a dead letter; se não existe → `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404).
3. Se `replayedAt` já preenchido → `409` (idempotência do replay, derivado — ver §8).
4. Em transação: "recoloca na outbox como pendente" ([09:18] Diego) — nova linha `PENDING` com `attempts = 0`, **mesmo `event_id` no payload** (o cliente pode dedupear a re-entrega via `X-Event-Id` — [09:25] Diego); marca `replayedAt`/`replayedById`.
5. **Log de auditoria** com o usuário que executou ([09:36] Sofia: "o endpoint de admin tem que logar quem fez o replay, pra auditoria"): `logger.info({ deadLetterId, eventId, replayedBy: req.user.id }, 'webhook_dlq_replay')`.

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1` (padrão de `src/app.ts` / `src/routes/index.ts`, via `buildWebhookRouter` registrado em `buildApiRouter`). Todas exigem `Authorization: Bearer <JWT>` (`authenticate`); apenas o replay exige role ADMIN ([09:36–09:37] Sofia/Marcos). Erros seguem o formato do `error.middleware.ts` existente:

```json
{ "error": { "code": "WEBHOOK_INVALID_URL", "message": "...", "details": {} } }
```

### 6.1 `POST /api/v1/webhooks` — cadastrar webhook

`customerId` vai no **body** — o JWT é do usuário operador, não do cliente ([09:32] Larissa: "o customer_id é passado no body ou no path. Não vem do JWT"). A secret é **gerada pela plataforma e devolvida na criação** ([09:31] Marcos). Devolvê-la apenas nesta resposta (e na rotação) é prática derivada de segurança — os GETs não a expõem.

Request:

```json
{
  "customerId": "c1a2b3c4-0000-0000-0000-000000000001",
  "url": "https://integracao.atlascomercial.com.br/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:

```json
{
  "id": "9f8e7d6c-0000-0000-0000-000000000010",
  "customerId": "c1a2b3c4-0000-0000-0000-000000000001",
  "url": "https://integracao.atlascomercial.com.br/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9b1dEXAMPLEc0ffee42EXAMPLE77aa55",
  "createdAt": "2025-11-06T12:00:00.000Z",
  "updatedAt": "2025-11-06T12:00:00.000Z"
}
```

Erros: `400 VALIDATION_ERROR` (Zod: uuid inválido, `events` vazio ou fora do enum `OrderStatus`); `400 WEBHOOK_INVALID_URL` (URL não-`https` — [09:23] Sofia); `404 WEBHOOK_CUSTOMER_NOT_FOUND` (customer inexistente); `401 UNAUTHORIZED`.

### 6.2 `GET /api/v1/webhooks?customerId=<uuid>` — listar por customer

Lista os webhooks de um customer ([09:33] Bruno). Resposta paginada no padrão de `src/shared/http/response.ts` (mesmo envelope de `GET /orders`). A secret **não** aparece.

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "9f8e7d6c-0000-0000-0000-000000000010",
      "customerId": "c1a2b3c4-0000-0000-0000-000000000001",
      "url": "https://integracao.atlascomercial.com.br/webhooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2025-11-06T12:00:00.000Z",
      "updatedAt": "2025-11-06T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Erros: `400 VALIDATION_ERROR` (`customerId` ausente/inválido); `401 UNAUTHORIZED`.

### 6.3 `PATCH /api/v1/webhooks/:id` — editar

Editar url, eventos assinados ou ativação ([09:33] Bruno: "PATCH pra editar"). Request (campos opcionais):

```json
{ "url": "https://nova.atlascomercial.com.br/hooks", "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

Response `200 OK`: objeto do webhook atualizado (mesmo shape do 6.2, sem secret).
Erros: `404 WEBHOOK_NOT_FOUND`; `400 WEBHOOK_INVALID_URL`; `400 VALIDATION_ERROR`; `401`.

### 6.4 `DELETE /api/v1/webhooks/:id` — remover

([09:33] Bruno: "DELETE pra remover".) Response `204 No Content` (sem corpo).
Erros: `404 WEBHOOK_NOT_FOUND`; `401`.

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

"Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo" ([09:21] Sofia). Sem body.

Response `200 OK`:

```json
{
  "id": "9f8e7d6c-0000-0000-0000-000000000010",
  "secret": "whsec_NEWc0ffee42EXAMPLEdeadbeef1234ab",
  "previousSecretExpiresAt": "2025-11-07T12:00:00.000Z"
}
```

Semântica: a secret atual vira `previousSecret` com `previousSecretExpiresAt = now() + 24h`; até lá o worker assina com a secret **nova** e o cliente pode validar com qualquer uma das duas; expirada, "a antiga morre" ([09:21] Sofia). (Detalhe de implementação derivado: como só enviamos **uma** `X-Signature`, assinada com a nova, o grace serve para o cliente que ainda valida com a antiga aceitar ambas durante a migração — a janela dupla é do lado da validação do cliente, e a plataforma mantém a antiga disponível/documentada durante as 24h.)
Erros: `404 WEBHOOK_NOT_FOUND`; `409 WEBHOOK_ENDPOINT_INACTIVE` (rotação de endpoint desativado — derivado); `401`.

### 6.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

"Esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" ([09:34] Marcos). Paginação padrão (`page`, `pageSize` ≤ 100), ordenado por `createdAt desc`, janela consultável limitada às últimas ~100 tentativas por endpoint.

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "d0d1d2d3-0000-0000-0000-000000000042",
      "eventId": "5e6f7a8b-0000-0000-0000-000000000099",
      "attemptNumber": 2,
      "success": false,
      "payload": { "event_id": "5e6f7a8b-0000-0000-0000-000000000099", "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "responseStatus": 503,
      "responseBody": "Service Unavailable",
      "durationMs": 812,
      "createdAt": "2025-11-06T14:03:22.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 37, "totalPages": 2 }
}
```

Erros: `404 WEBHOOK_NOT_FOUND`; `400 VALIDATION_ERROR`; `401`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay de DLQ (ADMIN)

([09:35] Diego; role ADMIN + auditoria — [09:36] Sofia/Larissa.) Sem body.

Response `200 OK`:

```json
{
  "deadLetterId": "aa11bb22-0000-0000-0000-000000000077",
  "eventId": "5e6f7a8b-0000-0000-0000-000000000099",
  "status": "PENDING",
  "replayedAt": "2025-11-06T15:00:00.000Z"
}
```

Erros: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`; `409 CONFLICT` (dead letter já reprocessada — idempotência, derivado); `403 FORBIDDEN` (role ≠ ADMIN, via `requireRole`); `401 UNAUTHORIZED`.

### 6.8 Contrato do webhook enviado ao cliente

`POST` na URL cadastrada, com timeout de 10s do nosso lado ([09:42] Diego).

Headers ([09:44–09:45] Diego/Sofia):

| Header | Valor | Âncora |
|---|---|---|
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento (gerado na inserção na outbox; estável entre retries) | [09:25], [09:44] Diego |
| `X-Signature` | HMAC-SHA256 do corpo bruto, hex | [09:20] Sofia, [09:44] Diego |
| `X-Timestamp` | timestamp do envio (detecção de replay attack pelo cliente) | [09:44] Diego |
| `X-Webhook-Id` | id do endpoint webhook cadastrado | [09:44] Sofia |

Body ([09:43] Diego — sem `items`, "pra não inflar"; detalhes via `GET /orders/:id`):

```json
{
  "event_id": "5e6f7a8b-0000-0000-0000-000000000099",
  "event_type": "order.status_changed",
  "timestamp": "2025-11-06T14:01:07.000Z",
  "order_id": "0b1c2d3e-0000-0000-0000-000000000005",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "c1a2b3c4-0000-0000-0000-000000000001",
  "total_cents": 158000
}
```

Semântica de resposta ([09:42] Diego): **qualquer 2xx = entregue**; qualquer outro status, erro de conexão ou ausência de resposta em **10s** = falha, entra no ciclo de retry (§5.3). Duplicatas são possíveis (at-least-once); o cliente **deve** dedupear por `X-Event-Id` ([09:24–09:26]; [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).

Verificação da assinatura pelo cliente:

```ts
const expected = crypto.createHmac('sha256', secret).update(rawBody).digest('hex');
const valid = crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(xSignatureHeader));
// Durante rotação: validar contra a secret nova E a anterior (grace de 24h — [09:21] Sofia)
```

## 7. Matriz de erros

Prefixo `WEBHOOK_` para todos os códigos do módulo ([09:29] Larissa), no padrão de `src/shared/errors/app-error.ts` + `http-errors.ts` (mesma mecânica de `InvalidStatusTransitionError`/`InsufficientStockError` — [09:28] Bruno). Os três primeiros códigos foram citados literalmente na reunião ([09:28] Bruno: "Códigos tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc."); os demais são **derivados** do "etc." seguindo o mesmo padrão.

| Código | HTTP | Quando ocorre | Classe/origem |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` † | 404 | `:id` de webhook inexistente (GET/PATCH/DELETE/rotate/deliveries) | Nova `WebhookNotFoundError extends AppError` (404) — a `NotFoundError` existente fixa o código `NOT_FOUND`, então a subclasse define o código próprio |
| `WEBHOOK_INVALID_URL` † | 400 | URL cadastrada não-`https` ([09:23] Sofia) | Nova `WebhookInvalidUrlError extends BadRequestError` (aceita `code` custom) — reforça a validação Zod |
| `WEBHOOK_SECRET_REQUIRED` † | 400 | Invariante de secret violada (endpoint sem secret utilizável ao assinar/rotacionar — gatilho exato derivado; código literal da reunião) | Nova `WebhookSecretRequiredError extends BadRequestError` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — (interno, worker) | Payload serializado > 64KB no envio — erra, não trunca ([09:23–09:24]) | Nova `WebhookPayloadTooLargeError extends AppError` (422 se aflorar em HTTP); no worker vira falha → retry/DLQ com `reason` |
| `WEBHOOK_ENDPOINT_INACTIVE` | 409 | Operação sobre endpoint desativado (ex.: rotate-secret); no worker: evento pendente cujo endpoint foi desativado | Nova `WebhookEndpointInactiveError extends ConflictError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de `:id` inexistente na DLQ | Nova subclasse de `AppError` (404), como `WEBHOOK_NOT_FOUND` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` do body sem Customer correspondente no `POST /webhooks` | Nova subclasse de `AppError` (404) — análoga ao `NotFoundError('Customer')` usado em `OrderService.create` |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno, worker) | Cliente não respondeu em 10s ([09:42] Diego) | Erro interno do processor; nunca vira resposta HTTP da API — registrado em `lastError`/`webhook_deliveries.responseBody` e conta como tentativa falha |
| `VALIDATION_ERROR` | 400 | Zod (uuid, enum de events, body malformado) | `ValidationError` **existente** via `validate.middleware.ts` — sem código novo |
| `UNAUTHORIZED` / `FORBIDDEN` | 401/403 | JWT ausente/inválido; replay sem role ADMIN | `UnauthorizedError`/`ForbiddenError` **existentes** via `authenticate`/`requireRole` |

† Código literal de [09:28] Bruno. Todos os erros novos caem no `error.middleware.ts` existente **sem alteração** ([09:29] Bruno: "Vai pegar nossos erros sem precisar mudar nada"), produzindo `{ error: { code, message, details } }`.

## 8. Estratégias de resiliência

- **Timeout de 10s** por chamada HTTP do worker ([09:42] Diego). Falha por timeout é indistinguível de falha de rede para fins de retry.
- **Retry + backoff + DLQ**: 5 tentativas, 1m/5m/30m/2h/12h, depois `webhook_dead_letter` com replay manual — ver §5.3–5.4 e [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md). Cobre janelas reais de indisponibilidade já observadas (manutenção de 2h de cliente — [09:16] Diego).
- **Crash do worker entre o envio e a marcação `DELIVERED`**: a linha fica `PROCESSING` e o evento pode ser reenviado (ex.: reaper que devolve `PROCESSING` antigos para `PENDING` após limiar — mecanismo derivado). A duplicata resultante é **coberta por design** pela semântica at-least-once: o cliente dedupica por `X-Event-Id`, estável entre reenvios ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md); a consequência "worker morre entre envio e marcação" está registrada no próprio ADR).
- **Idempotência do replay**: `replayedAt` impede reenfileirar a mesma dead letter duas vezes (409 na segunda chamada); e mesmo que um replay gere re-entrega de evento já recebido, o `event_id` preservado permite dedup no cliente (derivado de [09:18] + [09:25]).
- **Worker single-instance e ordering**: nesta fase roda **um único worker**; a ordem de entrega segue `created_at` da outbox, o que dá ordering **por `order_id`** (não global) ([09:12–09:13] Diego/Larissa: "Documentamos como limitação conhecida"). Escalar para múltiplos workers exigirá particionamento por `order_id` ou lock pessimista — fora de escopo ([09:13] Diego).
- **Atomicidade na origem**: nenhum cenário em que status mudou sem evento registrado ou vice-versa — garantido pelo `$transaction` ([09:06] Diego; [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md)).

## 9. Observabilidade

**Logs estruturados (Pino existente — `src/shared/logger/index.ts`)** — decisão de reuso: "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo" ([09:29] Bruno). O worker cria seu logger com o mesmo `createLogger()` (ajustando `base.service` para `order-management-worker` — derivado). **A secret do endpoint NUNCA pode ser logada**: adicionar `*.secret` e `*.previousSecret` aos `redactPaths` existentes (`*.password`, `*.token`, ...) — prática derivada diretamente do mecanismo de redaction já presente no logger, motivada pelo incidente real de secret vazada em log ([09:22] Diego).

**Correlação por `event_id`** (derivado do padrão `requestId` do `request-logger.middleware.ts` existente): o `event_id` atravessa toda a cadeia — nasce na inserção na outbox (API), aparece nos logs do worker, no header `X-Event-Id` e em `webhook_deliveries.outboxEventId` — permitindo rastrear uma entrega de ponta a ponta. Eventos de log sugeridos: `webhook_event_enqueued`, `webhook_delivery_attempt`, `webhook_delivered`, `webhook_delivery_failed`, `webhook_moved_to_dlq`, `webhook_dlq_replay`.

**Log de auditoria do replay** — decisão da reunião ([09:36] Sofia): registrar `replayedBy` (id do usuário ADMIN do JWT) no log e na coluna `replayedById`.

**Métricas** (práticas derivadas dos objetivos decididos; sem stack nova de métricas — expostas via logs agregáveis/consultas SQL nesta fase):

| Métrica | Fonte | Alerta sugerido |
|---|---|---|
| Eventos pendentes na outbox (backlog/lag do worker) | `count(status IN (PENDING, FAILED))` | Crescimento sustentado → worker parado (viola o <10s de [09:02]) |
| Idade do pendente mais antigo | `min(createdAt)` dos pendentes | > 10s indica violação do SLO |
| Taxa de sucesso/falha de entrega | `webhook_deliveries.success` | Queda por endpoint → cliente com problema |
| Tamanho da DLQ | `count(webhook_dead_letter)` sem `replayedAt` | > 0 exige intervenção manual (não há e-mail — [09:37]) |
| Duração da chamada HTTP | `webhook_deliveries.durationMs` | p95 próximo de 10s → clientes lentos pressionando o ciclo |

## 10. Dependências e compatibilidade

- **Nenhuma infraestrutura nova**: MySQL + Prisma existentes; sem Redis, broker ou fila externa ([09:07] Diego: "Outbox no MySQL existente resolve"). Sem dependências npm novas (HMAC via `node:crypto`; HTTP via `fetch` nativo — derivado).
- **Novo processo**: `src/worker.ts` + script `"worker": "tsx watch --env-file=.env src/worker.ts"` (dev) e `node dist/worker.js` (produção) no `package.json`, análogo aos scripts `dev`/`start` existentes ([09:11] Larissa: "criar um src/worker.ts e um script npm run worker").
- **PrismaClient por processo**: o worker abre instância própria — "Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node" ([09:30] Bruno). Implica um segundo pool de conexões contra o mesmo MySQL (risco tratado no §13).
- **Migração de banco**: uma migration Prisma nova (aditiva: 4 tabelas + 1 enum + relação inversa em `Customer`); nenhuma tabela existente alterada estruturalmente. *(Proposta — não aplicada neste repositório.)*
- **Compatibilidade retroativa total**: nenhum endpoint atual muda de contrato; `GET /orders/:id` continua sendo a fonte de detalhes do pedido para o cliente após receber o webhook ([09:43] Diego). A única alteração em código existente é o passo adicional dentro do `changeStatus` (§11.1).

## 11. Integração com o sistema existente

### 11.1 `src/modules/orders/order.service.ts`

Única alteração em código existente: `changeStatus` ganha a chamada `await publishWebhookEvent(tx, order, from, to)` **dentro do `$transaction` existente**, no ponto exato entre o `tx.orderStatusHistory.create(...)` e o `tx.order.findUnique(...)` final que precede o commit ([09:40] Bruno: "A gente vai inserir na webhook_outbox dentro da mesma transação. Se a outbox falhar de inserir, rollback"). Assinatura em função pura recebendo o `TxClient` — sem injetar repository no `OrderService` ([09:41] Bruno/Diego: "função pura recebendo o tx. Não precisa injetar repository inteiro"). `create` (pedido novo, `PENDING` inicial) **não** publica evento — o requisito é notificação de **mudança** de status ([09:00] Marcos); registrado como comportamento explícito.

### 11.2 `src/modules/orders/order.status.ts`

Fonte da verdade dos eventos possíveis: o mapa `transitions` define quais pares `from → to` existem, logo todo evento `order.status_changed` corresponde a uma transição validada por `canTransition` antes de `publishWebhookEvent` rodar. Os **status assináveis** em `WebhookEndpoint.events` são exatamente os valores do enum `OrderStatus` (o Zod usa `z.nativeEnum(OrderStatus)`, como `updateOrderStatusSchema`). Nenhuma alteração neste arquivo.

### 11.3 `src/shared/errors/app-error.ts` + `src/shared/errors/http-errors.ts`

Novas subclasses `WEBHOOK_*` (§7) seguem o padrão exato de `InvalidStatusTransitionError extends ConflictError` e `InsufficientStockError extends UnprocessableEntityError`: classe específica, código estável, `details` estruturado. Ficam em `src/modules/webhooks/webhook.errors.ts` ou agregadas em `http-errors.ts` (padrão atual centraliza; qualquer das duas mantém o import via `shared/errors/index.js` — decisão de organização derivada). `AppError` não muda.

### 11.4 `src/middlewares/auth.middleware.ts`

`authenticate` aplicado no topo do `buildWebhookRouter` (`router.use(authenticate)`, como em `order.routes.ts`). CRUD/rotate/deliveries: qualquer role autenticada nesta fase ([09:36–09:37] Marcos/Sofia). Replay: `requireRole('ADMIN')` adicional ([09:36] Larissa: "a gente reaproveita o requireRole que já existe"). `req.user.id` alimenta o log de auditoria do replay. Nenhuma alteração no middleware.

### 11.5 `src/middlewares/error.middleware.ts`

Zero alteração: as novas subclasses de `AppError` caem no primeiro branch (`err instanceof AppError`) e saem como `{ error: { code, message, details } }`; erros Zod e Prisma dos novos fluxos já são tratados pelos branches existentes ([09:29] Bruno: "O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada"). No **worker** não há middleware Express: o processor captura erros por evento (try/catch no loop) e os converte em `lastError`/retry — derivado.

### 11.6 `src/middlewares/validate.middleware.ts` + padrão `*.schemas.ts`

Novo `src/modules/webhooks/webhook.schemas.ts` no padrão de `order.schemas.ts`, consumido pelo `validate({ body, params, query })` existente:

```ts
export const createWebhookSchema = z.object({
  customerId: z.string().uuid(),
  url: z.string().url().refine((u) => u.startsWith('https://'), {
    message: 'webhook url must use https', // [09:23] Sofia: "é só uma validação no schema Zod"
  }),
  events: z.array(z.nativeEnum(OrderStatus)).min(1),
});
export const webhookIdParamSchema = z.object({ id: z.string().uuid() });
```

### 11.7 `src/routes/index.ts` e `src/app.ts`

`buildWebhookRouter(controllers.webhooks)` entra no `buildApiRouter` (`router.use('/webhooks', ...)` e `router.use('/admin/webhooks', ...)` para o replay), e `buildControllers` em `app.ts` passa a montar `WebhookRepository → WebhookService → WebhookController` — mesma cadeia de DI manual dos módulos existentes. Tudo automaticamente sob `/api/v1` (prefixo aplicado em `app.ts`).

### 11.8 `src/server.ts` → novo `src/worker.ts`

`src/worker.ts` replica a estrutura de bootstrap de `src/server.ts`: `PrismaClient` próprio ([09:30] Bruno), logger Pino, loop do §5.2 e graceful shutdown por `SIGINT`/`SIGTERM` (termina o batch corrente, para o polling, `prisma.$disconnect()` — espelha o `shutdown` existente). A lógica de processamento vive em `src/modules/webhooks/webhook.processor.ts`, mantendo o entry-point fino ([09:28] Bruno).

### 11.9 `prisma/schema.prisma`

Recebe os 4 models + 1 enum propostos no §4 (aditivo; convenções idênticas às existentes). **Proposta documental — o arquivo não é alterado neste desafio.**

### 11.10 `src/shared/http/response.ts`

`GET /webhooks` e `GET /webhooks/:id/deliveries` usam `paginated()`/`PaginatedResponse<T>` existentes — mesmo envelope `{ data, pagination }` de `OrderService.list`, com o teto das "últimas ~100" entregas ([09:34] Marcos) imposto no service.

### 11.11 `src/shared/logger/index.ts`

Worker e módulo usam `createLogger()`/`logger` existentes; extensão dos `redactPaths` com `*.secret`/`*.previousSecret` (§9). Nenhuma lib de log nova ([09:29] Bruno).

## 12. Critérios de aceite técnicos

- [ ] **Atomicidade (testável)**: transição de status válida com endpoint assinante ⇒ exatamente 1 linha `PENDING` na outbox por endpoint assinante, no mesmo commit; transição que falha (ex.: `INSUFFICIENT_STOCK`) ⇒ zero linhas na outbox ([09:40] Bruno).
- [ ] **Filtro na inserção**: transição para status que nenhum endpoint do customer assina ⇒ zero linhas na outbox ([09:34] Bruno).
- [ ] **Snapshot**: alterar o pedido após a transição não muda o payload do evento já enfileirado ([09:52] Larissa; [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md)).
- [ ] **Latência ≤ 10s**: evento comitado é entregue a endpoint saudável em menos de 10s (polling 2s + envio) ([09:02] Marcos).
- [ ] **Assinatura verificável**: recomputar HMAC-SHA256 do corpo recebido com a secret devolvida na criação reproduz `X-Signature`; headers `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id` presentes ([09:20–09:22] Sofia; [09:44–09:45]).
- [ ] **Rotação com grace**: após rotate-secret, assinatura passa a usar a nova secret e a anterior permanece registrada com expiração `+24h`; após a expiração, a anterior é descartada ([09:21] Sofia).
- [ ] **Retry observável**: endpoint que responde 500 ⇒ linha vira `FAILED` com `attempts = 1` e `nextRetryAt ≈ now + 1min`; progressão 1m/5m/30m/2h/12h conferível em `nextRetryAt` ([09:17] Larissa).
- [ ] **Timeout**: endpoint que demora > 10s conta como falha ([09:42] Diego).
- [ ] **DLQ após 5 falhas**: quinta falha move o evento para `webhook_dead_letter` com payload, motivo e timestamp, e ele sai do ciclo do worker ([09:18] Diego).
- [ ] **Replay funcional e restrito**: `POST .../replay` com ADMIN reenfileira como `PENDING` (200) e loga quem executou; com OPERATOR retorna 403 ([09:36] Sofia/Larissa).
- [ ] **At-least-once**: reenvio (crash simulado/replay) mantém o mesmo `X-Event-Id` ([09:25] Diego).
- [ ] **Deliveries**: `GET /webhooks/:id/deliveries` expõe tentativas com sucesso/falha, payload, response e duração, janela de ~100 ([09:34] Marcos).
- [ ] **Payload > 64KB** gera erro e não é enviado ([09:23–09:24]).
- [ ] **Regressão zero**: suíte existente (`tests/orders.test.ts` etc.) permanece verde sem alterações nos testes atuais.
- [ ] **Erros no padrão**: todos os erros novos saem como `{ error: { code: "WEBHOOK_*", ... } }` pelo middleware existente ([09:28–09:29] Bruno).

## 13. Riscos e mitigação

| Risco | Origem | Mitigação |
|---|---|---|
| Crescimento sem limite da outbox (arquivamento fora de escopo) | [09:08] Diego | Índices em `status`/`createdAt` mantêm a leitura dos pendentes barata mesmo com tabela grande ([09:08]); métrica de backlog (§9) monitora; arquivamento após ~30 dias já apontado como follow-up na própria reunião ([09:08]) |
| Pool duplo de conexões (API + worker) contra o mesmo MySQL | [09:30] Bruno | Worker é single-instance com batch pequeno e conexões limitadas via `connection_limit` na `DATABASE_URL` do worker (parametrização derivada do Prisma); monitorar conexões no MySQL |
| Transação do `changeStatus` mais longa (consulta endpoints + inserts na outbox) | [09:40] Bruno | Operações adicionadas são leituras indexadas por `customerId` + inserts simples — sem I/O externo dentro da transação (o HTTP fica no worker, [09:04] Bruno); filtro na inserção reduz linhas ([09:34]); teste de integração cobre a transação estendida |
| Duplicatas de entrega (crash do worker, replay) | [09:25] Diego | Aceitas por design (at-least-once); dedup no cliente por `X-Event-Id` estável; documentação destacada no portal ([09:26] Marcos); [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Evento preso em `PROCESSING` após crash | Derivado do §8 | Reaper devolve `PROCESSING` mais antigos que um limiar para `PENDING`; duplicata resultante coberta pelo item anterior |
| Cliente indisponível > ~15h perde entrega automática | [09:17] Marcos (aceito) | Evento preservado na DLQ com replay manual; sem e-mail nesta fase ([09:37] Larissa) — DLQ > 0 exige operação humana (métrica §9) |
| Vazamento de secret | [09:22] Diego (incidente real) | Secret por endpoint (raio de dano limitado — [09:21] Sofia); rotação com grace 24h; redaction no Pino (§9); secret exposta apenas na criação/rotação (§6.1) |

## 14. Estratégia de testes

Segue o padrão existente: **Vitest + Supertest, integração E2E contra banco real**, com `tests/setup.ts` (truncamento por `deleteMany` em `beforeEach` — ganhará as novas tabelas webhook na ordem correta de FK) e `tests/helpers/factories.ts` (`getTestApp`, `bootstrapAuthenticatedUser(role)`, `createTestCustomer`, `createTestProduct`, `loginAndGetToken`).

| Suíte nova | Cobre | Como |
|---|---|---|
| `tests/webhooks.test.ts` | CRUD + rotate + deliveries + matriz de erros | `getTestApp()` + `bootstrapAuthenticatedUser()` + `createTestCustomer()`; asserts no shape `{ error: { code } }` como em `orders.test.ts` (ex.: `expect(res.body.error.code).toBe('WEBHOOK_INVALID_URL')` para URL `http://`); secret presente no 201 e ausente no GET; 403 no replay com OPERATOR vs. 200 com `bootstrapAuthenticatedUser('ADMIN')` |
| `tests/webhooks-outbox.test.ts` | Atomicidade da outbox no `changeStatus` | Cadastra webhook assinando `PAID`; `PATCH /orders/:id/status` → assert de 1 linha `PENDING` na outbox via `prisma.webhookOutbox.findMany` (mesmo estilo dos asserts diretos com `prisma.product.findUnique` de `orders.test.ts`); caso negativo: transição com estoque insuficiente (422) ⇒ outbox vazia; caso filtro: transição para status não assinado ⇒ outbox vazia; caso snapshot: payload não reflete mutações posteriores do pedido |
| `tests/webhook-worker.test.ts` | Loop/processor com endpoint fake | Servidor HTTP local efêmero (`node:http`) como endpoint do cliente; invoca `processEvent`/um `tick()` diretamente (sem esperar polling real); cenários: 200 ⇒ `DELIVERED` + linha em `webhook_deliveries`; 500 ⇒ `FAILED`, `attempts = 1`, `nextRetryAt ≈ +1min`; 5ª falha ⇒ linha na `webhook_dead_letter` e replay via endpoint admin reenfileira |
| `tests/webhook-hmac.test.ts` | Verificação de HMAC e headers | Endpoint fake captura headers/corpo; recomputa `createHmac('sha256', secret)` e compara com `X-Signature` (`timingSafeEqual`); valida `X-Event-Id` = id da linha da outbox, `X-Webhook-Id`, `X-Timestamp`; após rotate-secret, assinatura confere com a secret nova |

Observações: o teste do worker controla o tempo chamando o processor diretamente e/ou manipulando `nextRetryAt` no banco — evita esperas reais de minutos/horas do backoff; o timeout de 10s é testado com limiar reduzido injetado por parâmetro (derivado — a constante de produção segue [09:42] Diego). Integração no `order.service` reaproveita exatamente o fluxo `create → PATCH status` já exercitado em `tests/orders.test.ts`.
