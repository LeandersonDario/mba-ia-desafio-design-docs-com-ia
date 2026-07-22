# Tracker — Rastreabilidade dos Design Docs

Este documento é a tabela de rastreabilidade do Sistema de Webhooks de Notificação de Pedidos: cada item registrado no [PRD](PRD.md), no [RFC](RFC.md), no [FDD](FDD.md) e nos [ADRs](adrs/) é mapeado à sua origem — uma fala real da reunião (`TRANSCRICAO.md`, âncora no formato `[hh:mm] Falante`) ou um arquivo real do código-fonte deste repositório. O propósito é servir de referência cruzada contra alucinação: qualquer linha abaixo permite verificar, com `grep` na transcrição ou abrindo o arquivo citado, que o item documentado tem origem legítima. Quando um item possui múltiplas âncoras na transcrição, a tabela registra a principal/decisiva.

Legenda da coluna **Fonte**: `TRANSCRICAO` = a origem é uma fala da reunião; `CODIGO` = a origem é o código existente do repositório (padrões, arquivos e pontos de integração).

## docs/PRD.md — Requisitos funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Notificar endpoint do cliente a cada mudança de status do pedido (webhook outbound) | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (POST) com secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição da configuração de webhook existente (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção de webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listagem de webhooks por customer (GET); customer_id vem do body/path, não do JWT | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro por status assinado: endpoint escolhe quais status ouvir (ex.: SHIPPED e DELIVERED) | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico das últimas ~100 entregas: sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ restrito a role ADMIN, com registro de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret via API com grace period de 24h para a secret antiga | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Requisição assinada (HMAC) com headers X-Signature, X-Event-Id, X-Timestamp e X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Registro do evento na mesma transação da mudança de status (publicação transacional) | TRANSCRICAO | [09:06] Diego |

## docs/PRD.md — Requisitos não funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação abaixo de 10 segundos (definição de "tempo real" dos clientes) | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Semântica at-least-once: cliente pode receber duplicatas e deduplica por X-Event-Id | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Atomicidade: falha ao inserir na outbox causa rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | HTTPS obrigatório: URL http é recusada com erro de validação no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB — evento acima do limite gera erro, não truncamento | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada ao endpoint do cliente; estouro conta como falha | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Envio em processo separado: restart da API não interrompe a entrega | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Auditoria: todo replay de DLQ registra em log quem executou | TRANSCRICAO | [09:36] Sofia |

## docs/PRD.md — Fora de escopo

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-ESC-01 | docs/PRD.md | Fora de Escopo | Notificação por e-mail quando o webhook do cliente falha — adiada para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-02 | docs/PRD.md | Fora de Escopo | Dashboard/painel visual para o cliente — projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-03 | docs/PRD.md | Fora de Escopo | Webhooks inbound — fluxo é exclusivamente da plataforma para o cliente | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-04 | docs/PRD.md | Fora de Escopo | Arquivamento das linhas entregues da outbox (~30 dias) — fora desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-ESC-05 | docs/PRD.md | Fora de Escopo | Múltiplos workers em paralelo — "problema do futuro"; single-worker nesta fase | TRANSCRICAO | [09:13] Diego |
| PRD-ESC-06 | docs/PRD.md | Fora de Escopo | Rate limiting de envio para o cliente — observar e implementar se virar problema | TRANSCRICAO | [09:39] Diego |
| PRD-ESC-07 | docs/PRD.md | Fora de Escopo | Endurecimento de roles do CRUD de configuração — qualquer role autenticada por enquanto | TRANSCRICAO | [09:37] Sofia |

## docs/PRD.md — Objetivos e métricas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência p95 entre mudança de status e recebimento pelo cliente < 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Entrega automática para indisponibilidades de até ~15h (janela de retry de 5 tentativas) | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Adoção pelos 3 clientes solicitantes (Atlas, MaxDistribuição, Nova Cargo) antes do fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Redução do volume de polling no GET /orders pelos clientes integrados (meta qualitativa) | TRANSCRICAO | [09:00] Marcos |

## docs/PRD.md — Riscos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-RISCO-01 | docs/PRD.md | Risco | Cliente indisponível por mais de ~15h perde a entrega automática (aceito; mitigado por DLQ + replay) | TRANSCRICAO | [09:17] Marcos |
| PRD-RISCO-02 | docs/PRD.md | Risco | Atraso frente ao compromisso comercial de fim de novembro (3 sprints estimados) | TRANSCRICAO | [09:45] Marcos |
| PRD-RISCO-03 | docs/PRD.md | Risco | Crescimento da tabela de outbox sem arquivamento nesta fase (mitigado por índices) | TRANSCRICAO | [09:08] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | Vazamento de secret de webhook — já houve cliente que vazou secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Rajadas de chamadas para cliente com alto volume (50 mudanças em 1 min = 50 POSTs) | TRANSCRICAO | [09:38] Diego |

## docs/RFC.md — Alternativas descartadas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo síncrono no changeStatus — transação já pesada; cliente lento travaria mudanças de status | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Fila externa (Redis Streams/broker) — subir Redis Cluster é overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger de banco para reatividade — MySQL não notifica processo externo; polling de 2s atende | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Exactly-once — exigiria coordenação dos dois lados; at-least-once resolve 99% dos casos | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Secret global da plataforma — "se vaza uma, vaza tudo"; secret por endpoint contém o dano | TRANSCRICAO | [09:21] Sofia |

## docs/RFC.md — Questões em aberto

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-QA-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída — decisão adiada: observar e implementar se virar problema | TRANSCRICAO | [09:39] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em Aberto | Endurecimento de roles no CRUD de configuração — pode ser endurecido mais pra frente | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-03 | docs/RFC.md | Questão em Aberto | Escala do worker — múltiplos workers exigiriam particionamento por order_id ou lock | TRANSCRICAO | [09:13] Diego |

## docs/adrs/ — Decisões arquiteturais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL existente: evento inserido na mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo separado (src/worker.ts) com polling da outbox a cada 2 segundos | TRANSCRICAO | [09:10] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Retry com backoff exponencial (5 tentativas, 1m/5m/30m/2h/12h) e DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret única por endpoint, rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once com deduplicação pelo cliente via header X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes: AppError, Pino, error middleware, módulos, Zod | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-na-insercao.md | Decisão | Payload renderizado como snapshot no momento da inserção na outbox | TRANSCRICAO | [09:52] Larissa |

## docs/FDD.md — Contratos públicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks — cadastrar webhook; secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks?customerId= — listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id — editar url, eventos assinados ou ativação | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id — remover webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret — rotacionar secret com grace de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries — histórico das últimas ~100 entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay — replay de DLQ (ADMIN) | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Webhook enviado ao cliente: POST JSON com event_id, event_type, from/to_status, sem items | TRANSCRICAO | [09:43] Diego |

## docs/FDD.md — Matriz de erros

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND (404) — webhook inexistente; código citado literalmente na reunião | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (400) — URL não-https; código citado literalmente na reunião | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED (400) — invariante de secret; código citado literalmente na reunião | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE — payload > 64KB gera erro no worker, não trunca nem envia | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_ENDPOINT_INACTIVE (409) — derivado do padrão de subclasses ConflictError existente | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) — replay de id inexistente na DLQ | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND (404) — análogo ao NotFoundError('Customer') do OrderService.create | CODIGO | src/modules/orders/order.service.ts |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT (interno) — cliente não respondeu em 10s; conta como falha | TRANSCRICAO | [09:42] Diego |

## docs/FDD.md — Fluxos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Publicação do evento na outbox dentro do $transaction do changeStatus, com rollback conjunto | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Loop do worker: polling a cada 2s, batch pequeno dos pendentes mais antigos, envio e marcação | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com backoff exponencial: 5 tentativas, progressão 1m/5m/30m/2h/12h (~15h de janela) | TRANSCRICAO | [09:17] Larissa |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | DLQ e replay admin: 5ª falha move para webhook_dead_letter; replay recoloca como pendente | TRANSCRICAO | [09:18] Diego |

## docs/FDD.md — Modelos de dados

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-MODELO-01 | docs/FDD.md | Modelo de Dados | WebhookEndpoint — configuração com url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-MODELO-02 | docs/FDD.md | Modelo de Dados | WebhookOutbox — tabela webhook_outbox com o evento, inserida na mesma transação SQL | TRANSCRICAO | [09:06] Diego |
| FDD-MODELO-03 | docs/FDD.md | Modelo de Dados | WebhookDeadLetter — tabela separada com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-MODELO-04 | docs/FDD.md | Modelo de Dados | WebhookDelivery — registro de tentativas: sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | [09:34] Marcos |

## docs/FDD.md — Pontos de integração com o código existente

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus ganha publishWebhookEvent(tx, ...) dentro do $transaction existente | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Mapa transitions/canTransition é a fonte da verdade dos status assináveis (sem alteração) | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novas subclasses WEBHOOK_* seguem o padrão AppError/códigos estáveis existente | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | authenticate no topo do router de webhooks; requireRole('ADMIN') no replay (sem alteração) | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Middleware de erro centralizado já trata AppError/Zod/Prisma — zero alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Novo webhook.schemas.ts consumido pelo validate({ body, params, query }) existente | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | buildWebhookRouter registrado no buildApiRouter sob /api/v1 (rotas /webhooks e /admin/webhooks) | CODIGO | src/routes/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | src/worker.ts replica o bootstrap de src/server.ts (PrismaClient próprio, graceful shutdown) | CODIGO | src/server.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Proposta aditiva de 4 models + 1 enum nas convenções do schema existente (não aplicada) | CODIGO | prisma/schema.prisma |
| FDD-INT-10 | docs/FDD.md | Integração | Listagens usam paginated()/PaginatedResponse existentes — mesmo envelope { data, pagination } | CODIGO | src/shared/http/response.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Worker e módulo usam createLogger()/Pino existentes; redactPaths ganham *.secret | CODIGO | src/shared/logger/index.ts |

## Estatísticas de cobertura

| Métrica | Valor |
|---|---|
| Total de linhas rastreadas | **85** |
| Fonte = TRANSCRICAO | 72 (84,7%) |
| Fonte = CODIGO | 13 (15,3%) |

Distribuição por documento: PRD 35 linhas (11 RF, 8 RNF, 7 fora de escopo, 4 objetivos, 5 riscos) · RFC 8 linhas (5 alternativas, 3 questões em aberto) · ADRs 7 linhas (uma por decisão) · FDD 35 linhas (8 contratos, 8 erros, 4 fluxos, 4 modelos, 11 integrações). Todos os timestamps foram validados contra a `TRANSCRICAO.md` (timestamp + falante conferem com uma fala real que suporta o item) e todos os caminhos de código foram validados contra o filesystem do repositório.
